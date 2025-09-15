---
title: "GitHubのユーザー権限変更でCodeBuildのJIT Runnerが失敗した話"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codebuild", "github", "githubactions", "ci"]
published: false
---

# 現象
今回、以下のような現象に悩まされていました。

リポジトリのadmin権限を整理しているときに**特定の人物の権限をadminからwriteに変更したらCodeBuildのビルドが失敗するようになった**のです。仕組みもわからな状態だっため、仕組みを理解して原因を把握し対応策を考える必要がありました。

![](https://storage.googleapis.com/zenn-user-upload/1c678e5ea239-20250915.png)
*このadminをwriteに変えると失敗する*

## 環境の前提
今回の事象が発生したのはどのような環境かを説明します。

### 環境について
WORKFLOW_JOB_QUEUED イベントをトリガーとしてCodeBuildのセルフホスト型 GitHub Actions ランナーを利用してビルドを行っています。
https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/action-runner-overview.html

# 発生していたエラー
特定の人物の権限をadminからwriteに変えたら以下のようなメッセージが表示されビルドが失敗するようになりました。

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

まずはCodeBuildを利用したセルフホスト型 GitHub Actions ランナーの処理の仕組みについて整理をして行きたいと思います。

## CodeBuildがホストするセルフホスト型のGitHub Actionsランナーとは
GitHub は WORKFLOW_JOB_QUEUED イベントを受け取ると Webhookからビルドプロジェクトを動かします。CodeBuild がビルド内で GitHub Actions ランナーを起動して該当ジョブを実行し、完了後にランナー/ビルドを終了します（＝ジョブごとにエフェメラルに動く）。

処理を整理すると以下になります。

1. GitHubのリポジトリにPushなどを行い、GitHub Actions のジョブが実行待ちキューに入る(workflow_job イベントの queued アクションを検知)
1. CodeBuild が起動され、CodeBuild のビルド内で registration token 発行（GitHub API）
1. CodeBuild が起動したビルド環境内で、GitHub Actions の self-hosted runner バイナリをダウンロード(runner tarball ダウンロード → 展開 → config.sh で登録 → run.sh 起動)
1. 一時的に GitHub にランナー登録 
1. runner が GitHub からジョブを受ける → 実行 → 結果を返す
1. 終了時にランナーを削除する
1. ジョブ完了 → config.sh remove（または API で削除）→ CodeBuild 終了

ここで、もう一度エラーをみてみましょう。
>[Container] 2025/09/05 06:49:10.480905 Error while fetching runner token: error code 400: GitHub runner JIT configuration unavailable: ResourceNotFoundException: Unexpected error from GitHub while creating runner JIT configuration, please try again later

このようなエラーが発生していました。ログの失敗は DOWNLOAD_SOURCE フェーズ で発生しています。エラー内容は「ランナー登録用の（JIT）トークン取得に失敗した」というもので、CodeBuild 側が GitHub に対して registration token（＝ランナーを一時登録するためのトークン）を取りに行ったときに 400 エラーが返ってきたことを示します。つまり、上記処理の「CodeBuild 内で registration token 発行（GitHub API）」で失敗していたことがわかります。

図

このrunnerを登録するためのtoken取得の処理が失敗していますregistration-token を発行するには該当スコープ（例：repo 管理権限 / org 管理権限、または GitHub App の適切な権限）が必要。GitHub App を使っている場合、対象 repo/org にインストールされていないと ResourceNotFound 的なエラーが出ることがある。なつまり、この処理をするにはadminが必要であるということです。なので、adminからwriteに変えると失敗します。


## 原因
そもそもとして、今回の事象を整理するとこの実行が個人んおアカウントの権限に依存していることが問題です。GitHubのAudit logをみても個人が実行主体担っています。では、なぜ個人のアカウントになっているのか調査を進めると、直接的な原因は**CodeBuildの接続設定を空欄で作成したこと**でした。

CodeBuildでGitHubとの接続を設定する際、接続名を空欄にして作成すると、GitHubアプリがユーザーの代行として実行されるようになります。つまり、GitHubアプリによる接続・認証は行われているものの、実際の操作はそのアプリをインストールしたユーザーの権限で実行されることになります。

この状態では、以下のような問題が発生します：
- ユーザーの権限変更が直接CodeBuildの実行に影響する
- ユーザーがリポジトリから削除されると、CodeBuildが動作しなくなる可能性がある
- セキュリティ上、個人のアカウントに依存した運用となってしまう

## 対応方法
この問題を解決するには、GitHubアプリを適切にインストールし、Bot経由での接続に変更する必要があります。

具体的な手順：
1. CodeBuildのソース設定で、適切な接続名を指定する
2. GitHubアプリの権限設定を確認し、必要な権限（Actions、Contents等）が付与されていることを確認する
3. アプリがBot として動作するよう設定を見直す

これにより、個人のユーザー権限に依存しない、安定したCI/CD環境を構築できます。

# 学んだこと
- CodeBuildとGitHubの接続設定は、空欄にせず明示的に設定することが重要
- GitHubアプリの動作モード（ユーザー代行 vs Bot）を理解して適切に設定する
- CI/CDパイプラインは個人のアカウントに依存しない設計にすべき
- 権限変更時は、関連するサービスへの影響を事前に確認する

このような設定ミスは見落としやすいため、インフラ構築時のチェックリストに含めることをお勧めします。




----

特定の人物の権限をwriteにしたら失敗して戻したら成功するという状況から「何かしらの処理」が特定のユーザ権限に紐づいてしまっているように見えるのですが、それが何故なのか、そしてそもそもとして何がおきているのかもわからなかったので

# まとめ
GitHubリポジトリのユーザー権限を整理した際、特定のユーザーをAdminからWriteに変更したところ、CodeBuildでホストしているGitHub Actionsのセルフホストランナーが失敗するようになりました。

**原因**: CodeBuildの接続設定を空欄で作成したため、GitHubアプリがユーザーの代行として実行されており、そのユーザーの権限変更が直接影響していた。

**解決策**: GitHubアプリを適切にインストールし、Bot経由での接続に変更することで問題を解決。



## 仕組み
AWS CodeBuild が起動したビルド環境内で、GitHub Actions の self-hosted runner バイナリをダウンロード → 一時的に GitHub にランナー登録 → ジョブを受け取って実行 → 終了時にランナーを削除するフローです。ログはまさにこの一連のライフサイクル（起動 → ダウンロード → 接続 → ジョブ実行 → クリーンアップ）を順に表しています。

GitHub Actions ワークフロー：runs-on に CodeBuild 用ラベルを指定し、ジョブをキューに置く。
GitHub：WORKFLOW_JOB_QUEUED を受けた仕組み（Webhook / GitHub がトリガー）により CodeBuild を呼ぶ構成が入ることが多い。
AWS CodeBuild（オンデマンド実行環境）：ビルドコンテナ／ホストで runner を起動する実行基盤。buildspec.yml に沿ってフェーズを進める。
Runner バイナリ（actions/runner）：ダウンロードして config.sh / run.sh を使い GitHub に登録してジョブを実行する。
Secrets / Token 発行機構：registration token を GitHub API 経由で発行するための PAT または GitHub App（CodeBuild は SecretsManager 等から取得）。

1. GitHub ワークフロー → ジョブキュー
1. CodeBuild 起動（On-demand）→ buildspec 実行開始
1. CodeBuild 内で registration token 発行（GitHub API）
1. runner tarball ダウンロード → 展開 → config.sh で登録 → run.sh 起動
1. runner が GitHub からジョブを受ける → 実行 → 結果を返す
1. ジョブ完了 → config.sh remove（または API で削除）→ CodeBuild 終了

# 今回のエラーについて

## 事象
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
このようなエラーが発生していました。
ログの失敗は DOWNLOAD_SOURCE フェーズ で発生しています。エラー内容は「ランナー登録用の（JIT）トークン取得に失敗した」というもので、CodeBuild 側が GitHub に対して registration token（＝ランナーを一時登録するためのトークン）を取りに行ったときに 400 エラーが返ってきたことを示します。

## 調査
まずエラーログだけでは原因が特定できなかったため、CodeBuildの設定を確認しました。しかし、設定に明らかな問題は見つからず、試しに以前変更したAdmin→Writeの権限を元に戻したところ、エラーが解消されました。

これにより、特定のユーザーの権限に依存していることは判明しましたが、本来であればBot経由でのアクセスとなっているはずなので、なぜユーザー権限が影響するのかが理解できませんでした。

そこで、GitHubのAudit logを確認したところ、予想に反してBot経由ではなく、実際にユーザーの権限で実行されていることが判明しました。

## 原因
調査を進めると、直接的な原因は**CodeBuildの接続設定を空欄で作成したこと**でした。

CodeBuildでGitHubとの接続を設定する際、接続名を空欄にして作成すると、GitHubアプリがユーザーの代行として実行されるようになります。つまり、GitHubアプリによる接続・認証は行われているものの、実際の操作はそのアプリをインストールしたユーザーの権限で実行されることになります。

この状態では、以下のような問題が発生します：
- ユーザーの権限変更が直接CodeBuildの実行に影響する
- ユーザーがリポジトリから削除されると、CodeBuildが動作しなくなる可能性がある
- セキュリティ上、個人のアカウントに依存した運用となってしまう

## 対応方法
この問題を解決するには、GitHubアプリを適切にインストールし、Bot経由での接続に変更する必要があります。

具体的な手順：
1. CodeBuildのソース設定で、適切な接続名を指定する
2. GitHubアプリの権限設定を確認し、必要な権限（Actions、Contents等）が付与されていることを確認する
3. アプリがBot として動作するよう設定を見直す

これにより、個人のユーザー権限に依存しない、安定したCI/CD環境を構築できます。

# 学んだこと
- CodeBuildとGitHubの接続設定は、空欄にせず明示的に設定することが重要
- GitHubアプリの動作モード（ユーザー代行 vs Bot）を理解して適切に設定する
- CI/CDパイプラインは個人のアカウントに依存しない設計にすべき
- 権限変更時は、関連するサービスへの影響を事前に確認する

このような設定ミスは見落としやすいため、インフラ構築時のチェックリストに含めることをお勧めします。

## 調査
まずエラーログだけでは原因が特定できなかったため、CodeBuildの設定を確認しました。しかし、設定に明らかな問題は見つからず、試しに以前変更したAdmin→Writeの権限を元に戻したところ、エラーが解消されました。

これにより、特定のユーザーの権限に依存していることは判明しましたが、本来であればBot経由でのアクセスとなっているはずなので、なぜユーザー権限が影響するのかが理解できませんでした。

そこで、GitHubのAudit logを確認したところ、予想に反してBot経由ではなく、実際にユーザーの権限で実行されていることが判明しました。
