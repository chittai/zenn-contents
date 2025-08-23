---
title: "Amazon Q Developer CLI をインストールして、さぁ、使うぞ！となった時に読む記事"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","amazonq"]
published: true
publication_name: "genda_jp"
---

# はじめに
弊社ではAmazon Q Developer(Pro Tier) を導入し検証しています。ツールのアップデートが多く陳腐化するスピードも早いと思いますが、インストールした後に次はどうしよう？となったときに何をすると良さそうかを書きたいと思います。

:::message
基本的に**CLI**で活用する前提です。
:::

# まとめ
- **やること① CLIの最新化**: インストール後も定期的にアップデートし、最新機能と修正を取り込む。
- **やること②-1 コンテキスト設定**: `AmazonQ.md`、`README.md`、`.amazonq/rules/**/*.md` を用意する。
- **やること②-2 カスタムエージェントの作成**: ユースケースごとにエージェントを作成し、`resources` や利用ツールを調整。切り替えは `q chat --agent <name>`。
- **やること③ MCPサーバ設定**: `~/.aws/amazonq/mcp.json` を編集してMCPサーバを登録する。
- **やること④ AWSアクセス用Profileの作成**: スクリプト等で `~/.aws/config` に全AWSアカウントのプロファイルを作成する。

# やるべきことその① Amazon Q CLI の最新化
これはインストールしたばかりの人は問題ないかもしれないですが、一度インストールしてからアップデートしていない人は多いのではないでしょうか。導入方法にもよりますが、とりあえずUpdateをして最新の状態にしておきましょう。私はHomebrewを使ってインストールしているため、以下コマンドを実行します。

```
brew update
brew upgrade amazon-q
```
https://docs.aws.amazon.com/ja_jp/amazonq/latest/qdeveloper-ug/command-line-installing.html

# やるべきことその② コンテキストの設定
Amazon Q Developerでは、コンテキストファイルを作成して読み込んでもらうことが出来ます。コンテキストファイルには Amazon Q Developerに考慮してもらいたいルールや考え方、情報を含めます。たとえば、プロジェクトの要件やコーディング規約、開発ルール（PR作成時のルールやフォーマットなどなど）です。つまり、Amazon Q Developerがより利用目的に対して関連性の高いレスポンスを提供するのに役立つ情報が含まれる想定です。

方法としては2つあります。

### 1. デフォルトで読み込まれる以下の3つのファイルを作成する
以下の3つはデフォルトエージェントで読み込まれます。後述する特定のユースケースに特化させたカスタムエージェントを使用しない場合は、このファイルを作成してください。`AmazonQ.md`と`READM.md`はプロジェクトディレクトリ直下に作成しすることで読み込まれます。私は、CLIからviコマンドで作成しています。Amazon Q Developerに作成を依頼することもありだと思います。
```
  - AmazonQ.md 
  - README.md 
  - .amazonq/rules/**/*.md 
```

たとえば、`zenn-contents` ディレクトリ直下で `q chat` を実行し、`/context show` を実行すると以下のように、読み込み対象となるファイル一覧(👤)とセッションで読み込まれているコンテキスト(💬)が表示されます。そして、「1 matched file in use:」には今回のセッションで読み込まれているファイルが表示されています。
```
> /context show

👤 Agent (q_cli_default):
    AmazonQ.md 
    README.md (1 match)
    .amazonq/rules/**/*.md 

💬 Session (temporary):
    <none>

1 matched file in use:
👤 /Users/xxx/Repositories/xxx/zenn-contents/README.md (~40 tkns)

Total: ~40 tokens
```

### 2. 特定のユースケースに対応するすため、カスタムエージェントを利用する
上記の3ファイルは基本的に汎用的なものになると思います。汎用的ではなく、Terraformの開発やドキュメント作成など特定のユースケースに特化してほしいときは、カスタムエージェントを作成します。この「エージェント」は読み込むファイルやMCPサーバをエージェント単位で設定することができます。自分の業務に合わせて作成してきましょう。

https://docs.aws.amazon.com/ja_jp/amazonq/latest/qdeveloper-ug/command-line-custom-agents-configuration.html

`q chat` をしてから `/agent create --name testagent(エージェント名)` とコマンドを実行すると以下のような表示が出てきます。`resources` にドキュメントのPATHを記載したり色々カスタマイズすることが出来ます。（ファイルの命名は適当です。ちゃんと目的にそってつけましょう）

```
{
  "$schema": "https://raw.githubusercontent.com/aws/amazon-q-developer-cli/refs/heads/main/schemas/agent-v1.json",
  "name": "testagent",
  "description": "",
  "prompt": null,
  "mcpServers": {},
  "tools": [
    "*"
  ],
  "toolAliases": {},
  "allowedTools": [
    "fs_read"
  ],
  "resources": [
    "file://AmazonQ-for-testagent.md" ★ 追加
  ],
  "hooks": {},
  "toolsSettings": {},
  "useLegacyMcpJson": true
```

作成すると一覧に表示されます。ですが、2025/8時点ではセッション内からエージェントを変えることができないため、一度終了させてから `q chat --agent testagent` で入りなおします。
```
> /agent list

* q_cli_default
  testagent
  authentification-automationPrj
```

エージェントを選択して`q chat`にログインすると`>` の前にエージェント名が表示されています。`/context show` の結果も指定した通りになっています。（ファイルを作成していないので、読み込みはされていない）
```
[testagent] > /context show

👤 Agent (testagent):
    AmazonQ-for-testagent.md 

💬 Session (temporary):
    <none>

No files in the current directory matched the rules above.
```

このように、コンテキストを設定することで「より快適に」Amazon Q Developerを使用できるようになります。



# やるべきことその③ MCPサーバの設定
Amazon Q DeveloperでMCPサーバを使えるように設定します。`qchat mcp add` で[追加することもできる](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/command-line-mcp-config-CLI.html)のですが、Cursorなど他で利用している設定があれば、以下のファイルを更新するで問題ありません。私はそうしています。
```
 ~/.aws/amazonq/mcp.json
```

`q chat` で読み込まれたMCPサーバが表示されます。セッション内で`/mcp`を実行してもOKです。
```
・・・
✓ awslabs.core-mcp-server loaded in 2.98 s
✓ atlassian loaded in 3.25 s
✓ awslabs.cdk-mcp-server loaded in 3.41 s
・・・
```


# やるべきことその④ AWS環境にアクセスするためのProfileの作成
ここは人によりますが、Amazon Q Developerを活用しようとなっているからにはAWSを利用している方が多いと思います。Amazon Q Developerにはビルドインツールである`use_aws`があります。これは、AWS環境へのアクセス体験が良くなるため([参考](https://zenn.dev/genda_jp/articles/20250809-amazonq-use-aws))、ぜひAWS環境へアクセスするためのProfileを準備しましょう。以下にスクリプトを用意しています。

https://gist.github.com/chittai/62e0c19fdcf6f494cb30eb604b1ae161

前提として、IAM Identity Centerを利用してアカウントを管理していて、SSOのセッション情報があること、アカウント一覧が取得できる管理アカウントのProfileが存在していることです。もちろん、権限もアカウント一覧が取得できるものを用意してください。

```
[sso-session aws-xxx]
sso_region = ap-northeast-1
sso_start_url = https://xxx.awsapps.com/start/

[profile xxx-payer-admin]
sso_session = aws-xxx
sso_account_id = xxxxxxxxxxxx
sso_role_name = AWSAdministratorAccess
region = ap-northeast-1
output = json
```

私は、これもAmazon Q Developerに実行してもらいました。いい感じに出来上がったので、これでAWSへアクセスするための条件は整いました。use_awsではいい感じにProfileを切り替えながら作業を進めてくれるのでとても助かります。



# 最後に
ツールがどんどん増えていくなかで、学習コストが高まっていると思います。インストールしたのは良いが、どんな設定をすればよいのか、本当に活用できるのか？という不安もでてきます。そういった方の学習コストを少しでも下げられれば嬉しいです。

Amazon Q DeveloperはAWS環境へのアクセスを簡略化し、環境の調査や分析など人手だとコストが高すぎて難しい作業も簡単に実現してくれます。今後も活用を進めて、事例などを共有できれば良いなと思います。

