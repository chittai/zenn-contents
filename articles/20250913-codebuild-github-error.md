---
title: "GitHubのリポジトリでユーザー権限を変更したらCodeBuildが失敗した話"
emoji: "🤔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codebuild", "github", "githubactions", "ci"]
published: false
publication_name: "genda_jp"
---

# はじめに
ここでは、タイトルの通り「GitHubのリポジトリでユーザー権限を変更したらCodeBuildが失敗した話」について記載します。

# 現象
今回、次の問題に遭遇しました。

リポジトリの権限を整理する中で、**特定ユーザーの権限を admin から write に変更したところ、CodeBuild のビルドが失敗するようになりました。** 仕組みがわからなかったため、原因の切り分けと対応策の検討が必要でした。

![](https://storage.googleapis.com/zenn-user-upload/1c678e5ea239-20250915.png)
*Roleの admin を write に変えると失敗する*

# 発生していたエラー
特定ユーザーの権限を admin から write に変えると、以下のメッセージが表示され、ビルドが失敗するようになりました。

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
GitHub Actions の **Workflow jobs** の queued アクションをトリガーに、CodeBuild のセルフホスト型 GitHub Actions ランナーでビルドしています。構成は[公式チュートリアル](https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/action-runner-overview.html)に準拠しています。
https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/action-runner-overview.html

まずは、理解のためにCodeBuild を使ったセルフホスト型ランナーによる処理の流れを整理します。

## CodeBuildがホストするセルフホスト型のGitHub Actionsランナーとは
GitHub は **Workflow jobs** の queued イベントを受け取ると Webhook からビルドプロジェクトを起動します。CodeBuild はビルド内で GitHub Actions ランナーを起動して該当ジョブを実行し、完了後にランナー/ビルドを終了します。

処理の流れを整理すると以下のようになります。

1. GitHub のリポジトリで Push などの操作を行い、GitHub Actions の Workflow jobs が実行待ちキューに入る
1. Webhook で CodeBuild が起動され、ビルド内で GitHub API を使ってランナー登録用の registration token を取得する
1. CodeBuild のビルド環境内で、GitHub Actions の セルフホスト型ランナー用バイナリをダウンロード
1. GitHub にランナーを登録し、ランナーが GitHub からジョブを受け取り実行する
1. 終了時にランナーを削除し、CodeBuild のビルドも終了する

![](https://storage.googleapis.com/zenn-user-upload/afbc8a23726b-20250920.jpg)
*処理の流れのイメージ図*

## エラーの再確認
ここで、もう一度エラーを見ます。
> [Container] 2025/09/05 06:49:10.480905 Error while fetching runner token: error code 400: GitHub runner JIT configuration unavailable: ResourceNotFoundException: Unexpected error from GitHub while creating runner JIT configuration, please try again later

この失敗は CodeBuildの DOWNLOAD_SOURCE フェーズで発生しています。内容は「ランナー登録用（JIT）のトークン取得に失敗した」というもの。つまり、上記フローの「CodeBuild 内で registration token を取得」の段階で 400 エラーになっていると読み取れます。

ここで今回の疑問を整理します。特定ユーザーが admin のときは上記エラーは発生せず、write に変更したらこのエラーが出ます。状況から、**処理を実行する主体が特定ユーザーの権限に依存している**ことはわかります。実際、以下の Audit log にもそれが表れていました。さらに registration token を発行するには、リポジトリに対する[admin 権限が必要](https://docs.github.com/ja/rest/actions/self-hosted-runners#create-a-registration-token-for-a-repository)です。

そのため、実行の主体になっている特定ユーザの権限をadminからwriteに変更することでadmin権限がなくなり ResourceNotFound などのエラーが発生したのだと考えることができます。
つまり、admin から write に変更すると失敗するのは仕様上ごくまっとうな結果でありそうです。

![](https://storage.googleapis.com/zenn-user-upload/d245079d4375-20250920.png)
*Audit log*

ここまでで、仕組みが整理できて何がおきているのかは把握できました。では、本来どうあるべきかを考えます。今回、そもそもとしてGitHubアプリが実行の主体だと考えていました。しかし実際には個人の権限で動いていたため、なぜ？となっていたところがあります。
私としては、人に付与された権限は今回の作業よりも範囲が広いことや人が居なくなったときの影響を考えると、人に依存しないように GitHubアプリが実行の主体になっていてほしいと考えています。

## どうして特定ユーザが実行の主体になっていたのか
さて、ここからAWSを含めた実装の話に入ります。なぜ個人の権限で動いていたのか調べると、直接の原因は **CodeBuild の接続設定を空欄で作成していたこと** でした。

![](https://storage.googleapis.com/zenn-user-upload/5449fe08b4f6-20250920.png)
*アプリインストールで「 GitHub ユーザーとして接続し、AWS CodeBuild プロジェクトで使用する」と記載があります*

CodeBuild で GitHub との接続を設定する際、接続名を空欄にして作成すると、GitHubアプリがユーザーの代行として実行されるようになります。つまり、GitHubアプリによる接続・認証は行われるものの、実際の操作はそのアプリをインストールしたユーザーの権限で実行されてしまいます。

## 対応方法
そこで、GitHubアプリを適切にインストールし、GitHubアプリ経由での接続に変更します。以下にその手順を示します。ポイントとして接続は一度作成すると編集できないため新たに作成し、プロジェクトから利用する接続を変更します。これは一度切断をする必要があるためその影響には気をつけてください。

手順：
1. CodeBuildの接続設定時に必ずアプリをインストールする
1. CodeBuild のソース設定で、適切な接続名を指定する（切断→再接続）

この対応を行うことで、GitHubアプリ経由で実行されるようになります。

![](https://storage.googleapis.com/zenn-user-upload/560f9f0b97e4-20250920.png)
*GitHubアプリが実行の主体に変わっている*

# 今回学んだこと
自分は始めて触った構成だったのですが、管理の観点からもGitHubアプリ経由でのアクセスにできたほうが良いと思うので、CodeBuild と GitHub の接続設定は、空欄にせず明示的に設定するとよいと考えます。
こうした設定ミスは見落としやすいため、インフラ構築時のチェックリストに含めておくと安心そうです！



