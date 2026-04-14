---
title: "久しぶりにドキュメントを読んだのでKiro CLIの機能を整理してみた"
emoji: "🗺️"
type: "tech"
topics: ["kiro", "aws", "ai"]
published: true
publication_name: "genda_jp"
---

# はじめに

株式会社GENDA SRE/インフラの布田です。

SRE/インフラでは以前から Amazon Q Developer CLI を AWS 環境の調査・運用に使っていたのですが、普段の開発ワークフローは Claude Code に寄せていたこともあり、あまり最近の動向を追えていませんでした。そこで自分のためにも改めて整理してみることにしました。

あと、この時点でどのように Kiro CLI を活用しているかも合わせて書いていくので誰かの参考になれば幸いです。


:::message
2026-04-13
kiro-cli 1.29.8 時点の情報です
:::


# Kiro CLI の料金
まずは改めて料金の確認です。

料金は Free（$0 / 50クレジット）から Power（$200 / 10,000クレジット）まで4段階あります。新規ユーザーには30日間有効な500ボーナスクレジットが付くので、とりあえず試すハードルは低いです。認証は Builder ID、IAM Identity Center、ソーシャルログインに加えて、v1.25.1 から Okta や Microsoft Entra ID の SSO にも対応しました。業務で使う時は IAM Identity Center で認証しています。

https://kiro.dev/pricing/

# Kiro CLI の機能について（自分選び）

改めて公式ドキュメントを読み直してみたら、自分が把握していなかった機能がけっこうあったので、整理していきます。ポイントになりそうなところを選んで紹介します。

https://kiro.dev/docs/cli/



## コア機能

### aws ビルトインツール

自分が Kiro CLI を使う一番の理由は、ビルトインツールである `aws`（旧 `use_aws`）の存在です。Q Developer CLI 時代からある機能で、AWS CLI を介さずに AWS API を直接呼び出せます。
何が便利かというと、Read 系の操作は確認なしで自動実行されるので、調査がサクサク進みます。また、パラメータを間違えた時にエージェントが正しい形式に補正してリトライしてくれる挙動があり、地味に助かっています。

:::message
Kiro CLIになった時にツール名が簡略化されています（`use_aws` → `aws`、`fs_read` → `read`、`execute_bash` → `shell` 等）
:::

### CustomAgent と SubAgent

Kiro CLI には「エージェント」の概念が2つあって、最初は紛らわしかったので整理しておきます。CustomAgent は **"定義"**、SubAgent は **"実行モデル"** という関係です。

- **CustomAgent**: ツール権限、自動承認、自動読み込みコンテキスト、MCP 接続などを JSON で定義する仕組み。`kiro-cli chat --agent <name>` で起動する対象そのもの。`/agent create` が AI 支援モードになって対話形式で作れるようになったのは最近知りました
- **SubAgent**: メインエージェントがタスクを委譲するためのランタイム。**独立したコンテキスト**で並列実行され、結果だけが親に返る。多段タスクの分割やコンテキスト肥大の回避に使います

SubAgent 起動時にどの CustomAgent 設定で動かすかを選べるので、**CustomAgent は SubAgent の素材にもなる**という関係です。

つまり、**同じ CustomAgent でもメインとして起動すれば通常のエージェント、別エージェントから起動されれば SubAgent として振る舞う**というのがポイントです。SubAgent として動作する時は次に述べるようにツールが制限されるので、同じ定義でも呼び出し方によってできることが変わる点は意識しておく必要があります。

| 観点 | CustomAgent | SubAgent |
|---|---|---|
| 起動方法 | `--agent <name>` で手動 | メインエージェントが会話中に起動 |
| コンテキスト | メインセッションと共有 | 独立コンテキスト、結果だけ返る |
| 目的 | 用途特化、権限制御、MCP統合 | 並列調査、コンテキスト節約 |

ここで注意したいのが、SubAgent として動くと**使えるツールが制限される**点です。公式ドキュメント（[SubAgents](https://kiro.dev/docs/cli/chat/subagents/)）によると、SubAgent から利用できるのは `read` / `write` / `shell` / `code` / MCP ツールのみで、**`aws`、`grep`、`glob`、`web_search`、`web_fetch` などは呼べません**。

これが自分の用途では地味に効いていて、 `aws` ビルトインが SubAgent からは使えないため、AWS 調査・運用系タスクを SubAgent に並列で投げる使い方ができません。そのため、AWS 系の作業はメインエージェント（CustomAgent 起動）で逐次実行する前提で組んでいます。

### MCP Integration

今の時代にあえて説明する必要もないような気もしますが、MCP サーバーとの連携です。設定は `.kiro/settings/mcp.json` に書きます。

```json
{
  "mcpServers": {
    "aws-docs": {
      "command": "uvx",
      "args": ["awslabs.aws-documentation-mcp-server@latest"]
    }
  }
}
```

### Hooks

エージェントのライフサイクルの特定ポイントでカスタムコマンドを実行する機能です。以下の5種類があります。
- AgentSpawn（起動時）
- UserPromptSubmit（プロンプト送信時）
- PreToolUse（ツール実行前）
- PostToolUse（ツール実行後）
- Stop（応答完了時）

https://kiro.dev/docs/cli/hooks/#hook-types


ここでポイントをあげるとしたら **PreToolUse で終了コード 2 を返すとツール実行をブロックできる**点です。STDERR の内容が LLM にフィードバックされるので、「なぜブロックしたか」をエージェントに伝えられます。セキュリティ検証用のガードレールとして使えそうな気がします。

### Steering

`.kiro/steering/` 配下のマークダウンファイルで AI の振る舞いを制御する機能で、Claude Code の `CLAUDE.md` に相当するのかなと思います。

自分は `aws-access-rules.md`（AWS アクセス時の必須手順）を Global に置いて使っています。このあたりは名前が違うだけでどのエージェントにでも再利用できると思います。

### Skills

Skills に関しては Claude Code を使っていると馴染みがあると思います。Skills は「ポータブルな指示パッケージ」で、`.kiro/skills/` に `SKILL.md` を置くと自動検出されます。Agent Skills というオープン標準仕様に準拠しているのが特徴で、将来的に他のツールでもスキル定義を使い回せる可能性があります。以下は例ですが、`references/` に必要なファイルを置いておけば CustomAgent ではなく Skills で解決できることも多そうです。

```
pr-review/
├── SKILL.md           # Required
└── references/        # Optional
    └── checklist.md

```

### ACP（Agent Client Protocol）

ACP は AI エージェントとエディタを連携させるオープン標準で、LSP（Language Server Protocol）のエージェント版と考えるとわかりやすいです。`kiro-cli acp` で起動すると JSON-RPC over stdin/stdout で通信するサーバーになって、**JetBrains IDE や Zed から Kiro エージェントを使えるようになる**というものです。（これは現時点では私は使っていないです）

```bash
kiro-cli acp --agent my-agent
```

### Autocomplete

ターミナルコマンドの AI 補完機能と、自然言語をシェルコマンドに変換する機能です。Git、Docker、kubectl、Terraform、AWS CLI あたりのコマンドを AI が補完してくれます。

## IDE 専用機能（CLI には来ていないもの）

**Specs（仕様駆動開発）** は `requirements.md` → `design.md` → `tasks.md` の3フェーズで仕様を構造化してからコードを書くアプローチで、Kiro の最大の差別化ポイントだと思います。現時点では CLI では使えません。

**Powers** は MCP + Steering + Hooks を1パッケージにバンドルして動的にロードする機能です。AWS CDK / SAM の Powers が公式提供されていますが、これも IDE 版限定。公式ブログでは CLI 対応予定に言及していますが、時期は未定です。

**Autopilot** という自律的なタスク遂行機能も IDE のみです。CLI では毎回ツール実行の確認が入ってしまうので、早めに来てほしいなと思っています。

# 自分の使い方

自分の環境では **Claude Code で日常業務のワークフロー自動化**、**Kiro CLI で AWS 環境の調査・運用操作**という棲み分けになっています。日常業務はほぼ Claude Code で回しつつ、Kiro CLI 側はエージェントやスキルを活用して AWS 運用特化で使っています。以下はその中でも使い回しができるAgentの使い方です。

## 例：idc-agent（IAM Identity Center の権限管理）

一番よく使っているエージェントです。`kiro-cli chat --agent idc-agent` で起動して、Jira の MCP サーバー経由でチケットの申請内容を読み取り、`aws` ツールで Identity Center の権限付与・剥奪を実行します。
権限付与は Write 系操作ですが、そこまで複雑ではないので Write 系操作もこちらの確認なく進めてもらうようにしています。手作業の手間は大幅に減らせています。

## 例：deactivate-user-agent（退職者アカウントの無効化）

退職者が出るたびに、複数の AWS Organizations の Identity Center を横断して対象ユーザーを検索し、レポートにまとめてくれるエージェントです。
ただし無効化操作自体は API から実行できないためエージェントにやらせていません。「調査・レポートはエージェント、実行は人間」という分担ですが、それでも全 Organizations を横断的に見てくれるのは大分助かっています。


# まとめ

Kiro CLI について整理してみて、以前よりも機能が充実してきたという感想です。Q Developer CLI 時代から使ってきた身としては、リブランディング後も `aws` ツールが健在なのは安心しました。また新しい機能 が CLI に来たら改めて評価したいです。
今回の記事が同じように「Kiro CLI ってどうなの？」と思っている SRE やインフラエンジニアの方の参考になれば幸いです。

