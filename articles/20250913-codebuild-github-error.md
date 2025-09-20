---
title: "GitHubのユーザー権限変更でCodeBuildのJIT Runnerが失敗した話"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codebuild", "github", "githubactions", "ci"]
published: false
---

# 現象
今回、次の問題に遭遇しました。

リポジトリの権限を整理する中で、**特定ユーザーの権限を Admin から Write に変更したところ、CodeBuild のビルドが失敗するようになりました。** 仕組みがわからなかったため、原因の切り分けと対応策の検討が必要でした。

![](https://storage.googleapis.com/zenn-user-upload/1c678e5ea239-20250915.png)
*この Admin を Write に変えると失敗する*

# 発生していたエラー
特定ユーザーの権限を Admin から Write に変えると、以下のメッセージが表示され、ビルドが失敗するようになりました。

```
[Container] 2025/09/05 06:49:02.098320 Running on CodeBuild On-demand
[Container] 2025/09/05 06:49:02.098330 Waiting for agent ping
[Container] 2025/09/05 06:49:02.199392 Waiting for DOWNLOAD_SOURCE
[Container] 2025/09/05 06:49:02.706838 Phase is DOWNLOAD_SOURCE
[Container] 2025/09/05 06:49:02.707850 CODEBUILD_SRC_DIR=/codebuild/output/src48989638/src
[Container] 2025/09/05 06:49:02.707969 YAML location is /codebuild/readonly/buildspec.yml
[Container] 2025/09/05 06:49:02.710408 Processing environment variables
[Container] 2025/09/05 06:49:02.846944 No runtime version selected in buildspec.
[Container] 2025/09/05 06:49:10.480905 Error while fetching runner token: error code 400: GitHub runner JIT configuration unavailable: ResourceNotFoundException: Unexpected error from GitHub while creating runner JIT configuration, please try again later
[Container] 2025/09/05 06:49:10.480928 Phase complete: DOWNLOAD_SOURCE State: FAILED
[Container] 2025/09/05 06:49:10.480942 Phase context status code: CLIENT_ERROR Message: Error while fetching runner token: error code 400: GitHub runner JIT configuration unavailable: ResourceNotFoundException: Unexpected error from GitHub while creating runner JIT configuration, please try again later
```

## 環境について
GitHub Actions の WORKFLOW_JOB_QUEUED をトリガーに、CodeBuild のセルフホスト型 GitHub Actions ランナーでビルドしています。構成は[公式チュートリアル](https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/action-runner-overview.html)に準拠しています。

まず、CodeBuild を使ったセルフホスト型ランナーの流れを簡単に整理します。

## CodeBuildがホストするセルフホスト型のGitHub Actionsランナーとは
GitHub は WORKFLOW_JOB_QUEUED イベントを受け取ると Webhook からビルドプロジェクトを起動します。CodeBuild はビルド内で GitHub Actions ランナーを起動して該当ジョブを実行し、完了後にランナー/ビルドを終了します。

処理の流れを整理すると以下のようになります。

1. GitHub のリポジトリで Push などの操作を行い、GitHub Actions のジョブが実行待ちキューに入る（workflow_job イベントの queued アクションを検知）
1. Webhook で CodeBuild が起動され、ビルド内で GitHub API を使ってランナー登録用の registration token を発行する
1. CodeBuild のビルド環境内で、GitHub Actions の self-hosted runner バイナリをダウンロード
1. GitHub にランナーを登録し、ランナーが GitHub からジョブを受け取り実行する
1. 終了時にランナーを削除し、CodeBuild のビルドも終了する

![](https://storage.googleapis.com/zenn-user-upload/4c40e2a4d388-20250920.jpg)
*処理の流れのイメージ図*

ここで、もう一度エラーを見ます。
> [Container] 2025/09/05 06:49:10.480905 Error while fetching runner token: error code 400: GitHub runner JIT configuration unavailable: ResourceNotFoundException: Unexpected error from GitHub while creating runner JIT configuration, please try again later

この失敗は DOWNLOAD_SOURCE フェーズで発生しています。内容は「ランナー登録用（JIT）のトークン取得に失敗した」というもの。つまり、上記フローの「CodeBuild 内で registration token を発行（GitHub API）」の段階で 400 エラーになっていると読み取れます。

## 課題のおさらい
ここで課題を整理します。特定ユーザーが admin のときは問題なく、write に変更した瞬間にこのエラーが出ます。なぜこうなるのかを確認します。

状況から、実行の主体が特定ユーザーの権限に依存していることがわかります。実際、Audit log にもそれが表れていました。さらに registration token を発行するには、リポジトリに対する[admin 権限が必要](https://docs.github.com/ja/rest/actions/self-hosted-runners#create-a-registration-token-for-a-repository)です。writeに変更することで権限がなくなった場合は ResourceNotFound などのエラーになります。つまり、admin から write に変更すると失敗するのは仕様上ごくまっとうな結果です。

![](https://storage.googleapis.com/zenn-user-upload/3a35f7267333-20250920.png)
*Audit logでchittaiというユーザがJITランナーの設定を読もうとしている*

ここで、本来どうあるべきかを考えます。今回、そもそもとしてGitHubアプリが実行の主体だと考えていました。しかし実際には個人の権限で動いていたため、なぜ？となっていました。本来であれば GitHubアプリが実行の主体になっていてほしいと考えています。

## どうして特定ユーザが実行の主体になっていたのか
さて、ここからAWSを含めた実装の話に入ります。なぜ個人の権限で動いていたのか調べると、直接の原因は **CodeBuild の接続設定を空欄で作成していたこと** でした。

![](https://storage.googleapis.com/zenn-user-upload/5449fe08b4f6-20250920.png)
*アプリインストールで「 GitHub ユーザーとして接続し、AWS CodeBuild プロジェクトで使用する」と記載があります*

CodeBuild で GitHub との接続を設定する際、接続名を空欄にして作成すると、GitHubアプリがユーザーの代行として実行されるようになります。つまり、GitHubアプリによる接続・認証は行われるものの、実際の操作はそのアプリをインストールしたユーザーの権限で実行されてしまいます。

## 対応方法
そこで、GitHubアプリを適切にインストールし、Bot経由での接続に変更します。

具体的な手順：
1. CodeBuildの接続設定時にかならずアプリをインストールする
1. CodeBuild のソース設定で、適切な接続名を指定する
1. GitHub アプリの権限設定を見直し、必要な権限（Actions、Contents など）が付与されていることを確認する
1. アプリが Bot として動作するよう設定を見直す

です。このとき接続は一度作成すると編集できないため新たに作成し、プロジェクトから利用する接続を変更します。これは一度切断をする必要があるためその影響には気をつけてください。

# 今回学んだこと
自分は始めて触った構成だったのですが、管理の観点からもBot経由でのアクセス似できたほうが良いと思うので、CodeBuild と GitHub の接続設定は、空欄にせず明示的に設定するとよいと考えます。
こうした設定ミスは見落としやすいため、インフラ構築時のチェックリストに含めておくと安心です！



