---
title: "Kiro CLI の現在地 -- 全機能マップと他AIエージェントとの差分を整理する"
emoji: "🗺️"
type: "tech"
topics: ["kiro", "aws", "ai", "cli", "developer-experience"]
published: false
publication_name: "genda_jp"
---

# はじめに

株式会社GENDAでSRE/インフラエンジニアをしている布田です。

AIコーディングツールが増えすぎて、正直「結局どれ使えばいいの？」と思っている方は多いのではないかなと思います。自分も普段は Claude Code を使っていますが、AWSが出している Kiro も気になっていました。

Kiro には IDE版（VS Code フォーク）と CLI版がありますが、IDE版の紹介記事は見かけるものの、**CLI版にフォーカスした情報**はまだ少ない印象です。なので今回は Kiro CLI に絞って、2026年4月時点（v1.28.0）でどんな機能があるのか、他のAIエージェントと何が違うのかを整理していきたいと思います。

:::message
この記事は主に公式ドキュメントベースの機能整理です。各機能の実際の使用感については、検証でき次第追記していく予定です。
:::

# Kiro CLI とは

Kiro は AWS（Amazon）が開発したエージェント型AIコーディングツールです。IDE版とCLI版の2つの形態で提供されており、内部的には Anthropic の Claude Sonnet をベースモデルとして使用しています。

CLI版は `kiro-cli` コマンドでターミナルから利用でき、Claude Code や Amazon Q Developer CLI と同じ「ターミナルエージェント」のカテゴリに入ります。

## 料金プラン（2026年3月改定）

| プラン | 月額 | クレジット |
|--------|------|-----------|
| Free | $0 | 50 |
| Pro | $20 | 1,000 |
| Pro+ | $40 | 2,000 |
| Power | $200 | 10,000 |

新規ユーザーには30日間有効な500ボーナスクレジットが付与されます。超過分は $0.04/クレジットです。

認証方式は Builder ID、IAM Identity Center、ソーシャルログインに対応しています。v1.25.1 からは Okta や Microsoft Entra ID によるエンタープライズ SSO にも対応しました。

https://kiro.dev/pricing/

# Kiro CLI の全機能マップ

ここからが本題です。Kiro CLI の機能を「コア機能（GA）」「Experimental 機能」「IDE 専用（CLI 非対応）」の3つに分けて整理していきます。

## コア機能（GA）

### Interactive Chat

ターミナルでの対話型チャットです。`kiro-cli chat` で開始します。セッションの保存・復元（`--resume` / `--resume-picker`）にも対応しているので、中断した作業の再開ができます。v1.28.0 では `/chat new` でCLI再起動なしに新しい会話を開始できるようになりました。

<!-- TODO: 実際にchat使ってみた感想を追記 -->

### Custom Agents

特定のユースケースに特化したエージェント設定を定義できます。`kiro-cli agent` コマンドでエージェントの作成・管理を行います。

定義できる項目は以下の通りです。

- ツールアクセス権限（使えるツールを制限）
- 許可レベル（承認なしで実行できるツールの設定）
- コンテキスト情報（自動で読み込むファイルやリソース）
- MCPサーバー接続

v1.27 では `/agent create` がAI支援モードに統合され、対話形式でエージェントを作れるようになっています。

<!-- TODO: カスタムエージェント作成の具体例を追記 -->

### MCP Integration

Model Context Protocol（MCP）サーバーとの連携です。`kiro-cli mcp add/remove/list/import/status` で管理します。

設定は `<project-root>/.kiro/settings/mcp.json` またはユーザーレベルの `~/.kiro/settings/mcp.json` に記載します。

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

ここで注目なのは、**HTTP ベースの MCP サーバーにも対応している**点です。`type: "http"` と URL 指定で接続でき、OAuth スコープの設定もできます。

:::message
ツール名は64文字以内、`^[a-zA-Z][a-zA-Z0-9_]*$` の形式が必要です。ツールの説明が10,000文字を超えるとパフォーマンス低下の警告が出ます。
:::

### Hooks

エージェントのライフサイクルの特定のポイントでカスタムコマンドを実行できる機能です。5種類のフックが用意されています。

| フック | タイミング | 用途 |
|--------|------------|------|
| **AgentSpawn** | エージェント起動時 | 初期化処理、環境情報の注入 |
| **UserPromptSubmit** | プロンプト送信時 | コンテキスト追加、バリデーション |
| **PreToolUse** | ツール実行前 | セキュリティ検証、実行ブロック |
| **PostToolUse** | ツール実行後 | ログ記録、後処理 |
| **Stop** | 応答完了時 | テスト実行、フォーマット、クリーンアップ |

ここで大事なのは **PreToolUse で終了コード 2 を返すとツール実行をブロックできる**点です。STDERR の内容が LLM にフィードバックされるので、「なぜブロックしたか」をエージェントに伝えられます。

ツールマッチャーも柔軟で、`"fs_write"` や `"@git"` のような指定だけでなく、`"@git/status"` のように特定ツールだけにマッチさせることもできます。

<!-- TODO: 実際にHooksを設定してみた例を追記 -->

### Steering

`.kiro/steering/` 配下のマークダウンファイルでAIの振る舞いを制御する機能です。Claude Code の `CLAUDE.md` に相当しますが、構造化のアプローチが異なります。

Kiro では以下の3つの基礎ファイルが用意されています。

| ファイル | 内容 |
|----------|------|
| `product.md` | 製品の目的、対象ユーザー、主要機能 |
| `tech.md` | フレームワーク、ライブラリ、技術制約 |
| `structure.md` | ファイル構成、命名規約、アーキテクチャ |

これに加えて `api-standards.md`、`security-policies.md` など任意のファイルを追加できます。

スコープは3階層あります。

| スコープ | 場所 | 適用範囲 |
|---------|------|---------|
| Workspace | `.kiro/steering/` | 特定プロジェクト |
| Global | `~/.kiro/steering/` | 全プロジェクト |
| Team | `~/.kiro/steering/`（MDM配布） | 組織全体 |

優先順位は Workspace > Global で、競合時は Workspace が勝ちます。`AGENTS.md` もサポートされていて、ワークスペースルートに置くと自動で読み込まれます。

### Skills

「ポータブルな指示パッケージ」です。`.kiro/skills/` または `~/.kiro/skills/` に配置し、SKILL.md にフロントマターと指示を書きます。

```yaml
---
name: pr-security-review
description: "Pull request のセキュリティレビュー実施時に使用"
---

## レビュー手順
...
```

ここがポイントとなりますが、Kiro の Skills は **Agent Skills というオープン標準仕様に準拠**しています。スラッシュコマンドで明示的に呼び出す必要はなく、ユーザーのリクエストがスキルの description と一致すると自動的にロードされます。

### Sub-agents

複雑なタスクをサブエージェントに委譲し、ライブで進捗を追跡できる機能です。v1.23.0 で導入されました。

<!-- TODO: Sub-agentsの使用感を追記 -->

### ACP（Agent Client Protocol）

**これは Kiro CLI の中でも特に注目すべき機能**です。

ACP は AI エージェントとエディタを連携させるためのオープン標準で、LSP（Language Server Protocol）のエージェント版と考えるとわかりやすいです。`kiro-cli acp` で起動すると JSON-RPC 2.0 over stdin/stdout で通信するサーバーになります。

```bash
# ACPモードで起動
kiro-cli acp

# 特定エージェント設定で起動
kiro-cli acp --agent my-agent
```

対応エディタは以下の通りです。

- **JetBrains IDE**（IntelliJ IDEA / WebStorm / PyCharm 等）
- **Zed**（ネイティブ対応）
- **ACP仕様をサポートするその他のエディタ**

:::message
IDE はシェルの PATH を継承しないため、設定ファイルには `~/.local/bin/kiro-cli` のような**絶対パス**を指定する必要があります。ここはハマりポイントになりそうです。
:::

サポートされる機能は、セッション管理、モデル/モード切り替え、ストリーミングレスポンス、画像コンテンツサポートです。`_kiro.dev/` プレフィックスの拡張メソッドで、スラッシュコマンドやMCPサーバー管理などの Kiro 固有機能も利用できます（ただし試験的）。

### Autocomplete

ターミナルコマンドのAI補完機能です。2つの形式があります。

- **ドロップダウンメニュー**: カーソル右に候補を表示。矢印キーで選択
- **インライン提案**: 灰色のゴーストテキストで候補を表示。Tab で受け入れ

Git、Docker、npm、kubectl、Terraform、AWS CLI など幅広いツールに対応しています。`kiro-cli inline enable/disable/status` で制御します。

### Translate

自然言語をシェルコマンドに変換する機能です。`kiro-cli translate` で使えます。

<!-- TODO: translate の使用例を追記 -->

### aws ツール（旧 use_aws）

Kiro CLI は Amazon Q Developer CLI の後継であり、**AWS 環境を直接操作するビルトインツール `aws`** を搭載しています。これは旧名 `use_aws` からリネームされたもので、後方互換性のため旧名もエイリアスとして動作します。

主な特徴は以下の通りです。

- **Read 系操作は確認不要**: describe / list / get 等の読み取り系は自動許可。Write 系（create / delete / update）のみ確認プロンプトが出る
- **パラメータスキーマの自動提案**: 間違ったパラメータを渡すと、botocore のスキーマから正しい形式を自動で提示してくれる
- **プロファイル自動切り替え**: マルチアカウント環境での横断調査が劇的に楽になる
- **ストリーミングレスポンス**: 大きなデータの取得にも対応

エージェント設定で `allowedServices` / `deniedServices` を指定すれば、操作対象の AWS サービスを制限できます。`autoAllowReadonly` を有効にすると、読み取り系の操作は全て自動承認されます。

:::message
`aws` ツールは Q Developer CLI 時代の `use_aws` と同一の機能です。リブランディング時にツール名が簡略化されました（`fs_read` → `read`、`execute_bash` → `shell` なども同様）。既存のカスタムエージェント設定は修正不要です。
:::

### Code Intelligence

18言語向けのLSP統合です。v1.24.0 で導入されました。エージェントがコードの構造を理解した上で操作できるようになります。

## Experimental 機能

`kiro-cli settings` で有効化できる実験的な機能群です。変更・削除の可能性がある点は注意が必要です。

| 機能 | コマンド | 有効化設定 | 概要 |
|------|---------|-----------|------|
| **Knowledge** | `/knowledge` | `chat.enableKnowledge` | セマンティック検索で情報を永続保存・検索 |
| **Tangent Mode** | `/tangent`（Ctrl+T） | `chat.enableTangentMode` | メイン会話を保持したまま横道探索 |
| **TODO List** | `/todo` | `chat.enableTodoList` | AI自動生成のタスクリスト。セッション間で永続化 |
| **Thinking Tool** | -- | `chat.enableThinking` | AI推論プロセスのステップバイステップ表示 |
| **Checkpoint** | `/checkpoint` | `chat.enableCheckpoint` | ファイル変更のGit風スナップショット |
| **Delegate** | -- | `chat.enableDelegate` | 非同期タスク実行 |
| **TUI** | `--tui` フラグ | -- | リッチターミナルUI（v1.28.0~） |

この中で個人的に面白いと思ったのは **Tangent Mode** です。メイン会話のコンテキストを壊さずに「ちょっとこれ調べたい」ができるのは、実務で地味に便利なのではないかなと思います。Claude Code だと横道にそれた時点で会話の流れが変わってしまうので。

**Knowledge** も興味深くて、エージェント固有のナレッジを隔離して保存できるとのことです。セッション間で永続するので、プロジェクト固有の知識をセマンティック検索で引き出せるのは実用性が高そうです。

<!-- TODO: Experimental 機能を実際に有効化して試した結果を追記 -->

## IDE 専用機能（CLI 非対応）

以下の機能は 2026年4月時点で **IDE版のみ**で利用可能です。

### Specs（仕様駆動開発）

Kiro の最大の特徴とも言える機能ですが、CLI には来ていません。`requirements.md`（EARS記法）→ `design.md` → `tasks.md` の3フェーズで仕様を構造化してからコードを書くアプローチです。

### Autopilot / Supervised

複数のコード変更を承認なしで実行する Autopilot モードと、ステップごとに確認する Supervised モードの切り替えです。

### Powers

MCP ツール + Steering + Hooks を1パッケージにバンドルし、会話コンテキストに応じて動的にロード/アンロードする機能です。AWS CDK / SAM / IAM Policy Autopilot などの公式 Powers が提供されています。

公式ブログでは「将来的に Kiro CLI、Cline、Cursor、Claude Code などにも展開する」と言及されていますが、時期は未定です。

:::message
Specs が CLI に来ていないのは個人的に残念なポイントです。ターミナルで仕様駆動開発ができたら面白いと思うのですが、今のところは IDE 版でしか体験できません。
:::

# 他のAIエージェントにないもの

全機能を整理した上で、「Kiro CLI にしかないもの」を5つピックアップします。

## 1. ACP -- エディタを選ばない接続方式

最も大きな差別化ポイントだと思います。

Claude Code はターミナル専用（IDE拡張は別途提供）、Cursor は Cursor IDE に閉じています。Kiro CLI は ACP 対応により、**JetBrains でも Zed でも同じエージェントを使える**のが強みです。LSP がエディタ間の言語サポートを標準化したように、ACP はエージェントのエディタ非依存を実現しようとしています。

「エディタは好きなものを使いたい、でもAIエージェントも使いたい」という人にとっては、これだけで選ぶ理由になり得ます。

## 2. Autocomplete + Translate -- ターミナル補完とコマンド変換

Claude Code にはないターミナル補完機能です。`git`、`docker`、`kubectl`、`terraform`、`aws` などのコマンドをAIが補完してくれます。Chat モードに入らなくても、普段のターミナル操作が補助されるのは使い勝手が良さそうです。

Translate（自然言語 → シェルコマンド変換）も同様で、「このファイルの最新3件のログを見たい」のような指示をコマンドに変換してくれます。

## 3. Tangent Mode -- 会話分岐による横道探索

Experimental ではありますが、メイン会話を保持したまま別のトピックを探索できる機能は他ツールにはありません。Claude Code で長い作業をしていて「ちょっとこれ調べたいけど、今の会話壊したくない」と思った経験がある人には刺さるのではないかなと思います。

## 4. Steering の構造化アプローチ

Claude Code の `CLAUDE.md` が1ファイル（+ サブディレクトリ）に全てを書くのに対し、Kiro は `product.md` / `tech.md` / `structure.md` の3ファイルに分離する設計です。さらに Workspace > Global の階層的優先順位や、Team スコープ（MDM配布）があるのも組織利用を意識した設計になっています。

ただ、Claude Code でも `.claude/` 配下にファイルを分けて管理はできるので、ここは「デフォルトの設計思想の違い」と捉えた方が正確です。

## 5. aws ビルトインツール -- AWS 環境操作のネイティブ統合

Q Developer CLI から引き継いだ `aws` ツールの存在は大きいです。Claude Code で AWS 環境を操作する場合は MCP サーバー（aws-mcp 等）経由になりますが、Kiro CLI はビルトインで AWS API を叩けます。

Read 系は自動承認、Write 系のみ確認プロンプト、パラメータ間違い時のスキーマ自動提案、プロファイル自動切り替えなど、**AWS 運用に特化した使い勝手の良さ**はMCP経由では得られないものです。SRE やインフラエンジニアにとってはこれだけで Kiro CLI を選ぶ理由になります。

## 6. Skills のオープン標準準拠

Kiro の Skills は Agent Skills 仕様というオープン標準に準拠しています。Claude Code のスキルがClaude Code固有の仕組みであるのに対し、Kiro のスキルは将来的に他のツールでも使い回せる可能性があります。

# Claude Code との比較

自分が普段使っている Claude Code と比較してみます。

| 機能カテゴリ | Kiro CLI | Claude Code |
|---|---|---|
| **対話型チャット** | `kiro-cli chat` | `claude` |
| **カスタムエージェント** | `.kiro/agents/` | `.claude/agents/` |
| **MCP** | 対応（HTTP/OAuth含む） | 対応 |
| **Hooks** | 5種（AgentSpawn / UserPromptSubmit / PreToolUse / PostToolUse / Stop） | 同等の構造 |
| **プロジェクトルール** | Steering（3ファイル分離 + 階層） | CLAUDE.md |
| **Skills** | Agent Skills 仕様準拠 / 自動検出 | 独自仕様 / スラッシュコマンド |
| **サブエージェント** | 対応 | 対応 |
| **AWS操作ビルトイン** | **`aws` ツール**（Read自動承認 / スキーマ提案） | なし（MCP経由） |
| **ACP** | **対応**（JetBrains / Zed 等） | 非対応（IDE拡張で対応） |
| **ターミナル補完** | **Autocomplete + Inline** | なし |
| **コマンド変換** | **Translate** | なし |
| **会話分岐** | **Tangent Mode**（Experimental） | なし |
| **ナレッジ管理** | **Knowledge**（Experimental） | Memory（ファイルベース） |
| **仕様駆動開発** | IDE版のみ（Specs） | なし |
| **動的ツール管理** | IDE版のみ（Powers） | なし |
| **ベースモデル** | Claude Sonnet | Claude Opus / Sonnet / Haiku |
| **モデル選択** | Sonnet 4.5 or Auto | 複数モデル切り替え可 |
| **認証** | Builder ID / IAM IdC / SSO | Anthropic アカウント / API キー |
| **料金** | $0~$200 | $20~$200（Max プラン） |

## 設計思想の違い

結論からいうと、**Claude Code は「推論の自律性と速度」を重視**し、**Kiro は「構造と標準化」を重視**している印象です。

Claude Code は強力なモデル（Opus）を使って自律的にタスクを進めるのが得意で、あまり構造を強制しません。一方、Kiro は Steering の3ファイル分離や Skills のオープン標準準拠、ACP によるエディタ非依存など、「エコシステムとしての拡張性」に設計リソースを振っている感じがします。

どちらが良いかはケースバイケースで、探索的な開発やプロトタイピングなら Claude Code、チームでの標準化や複数エディタでの利用なら Kiro、という使い分けになるのではないかなと思います。

# 自分の使い方 -- Claude Code と Kiro CLI の併用

ここからは自分が実際にどう使い分けているかを紹介します。

## 全体像

自分の環境では Claude Code と Kiro CLI を**併用**しています。ざっくり言うと、**Claude Code は日常業務のワークフロー自動化**、**Kiro CLI は AWS 環境の調査・運用エージェント**という棲み分けです。

| 観点 | Claude Code | Kiro CLI |
|---|---|---|
| カスタムエージェント | 10個（レビュー系・設計系・調査系） | 4個（AWS運用特化） |
| スキル | 23個（朝会・Jira・スプリント・ブログ等） | 0個（未使用） |
| Hooks | macOS通知、CodeXレビュー催促等 | 設定なし |
| MCP設定 | **シンボリックリンクで共有** | **同上** |
| 主な用途 | 開発ワークフロー全体 | AWS環境の調査・運用操作 |

MCP の設定ファイルはシンボリックリンクで一元管理しています。Kiro の `~/.kiro/settings/mcp.json` から Claude Code と同じ設定ファイルを参照しているので、MCP サーバーの追加・変更は1箇所で済みます。

## Kiro CLI のカスタムエージェント

自分が定義している Kiro エージェントは4つです。どれも AWS 環境の運用操作に特化しています。

### idc-agent -- IAM Identity Center の権限管理

一番よく使っているエージェントです。Jira のチケットから申請内容を読み取り、IAM Identity Center の権限付与・剥奪を実行します。

`kiro-cli chat --agent idc-agent` で起動すると、Jira の MCP サーバー経由でチケット内容を取得し、`aws` ビルトインツールで Identity Center の API を叩いて権限を操作します。Steering で AWS アクセスルールを定義しているので、プロファイルの選択や SSO 認証のフローも含めてエージェントが自律的に進めてくれます。

ここで大事なのは **「完全自動化」ではなく「半自動化」にしている**点です。権限付与は `aws` ツールの Write 系操作に該当するため、毎回確認プロンプトが出ます。これがちょうどいい安全弁になっていて、「エージェントが勝手にやらかす」リスクを抑えつつ、手作業の手間は大幅に減らせています。

以前 Amazon Q Developer CLI のカスタムエージェントで同じことをやっていた時期もありますが、Kiro CLI にリブランディングされた際にそのまま移行しました。ただ、`~/.aws/amazonq/rules/` のパスから `~/.kiro/steering/` への移行漏れが最初ハマりポイントでした。

### deactivate-user-agent -- 退職者アカウントの無効化

退職者の AWS アカウントを無効化するためのエージェントです。自分のところでは複数の AWS Organization を管理しているので、退職者が出るたびに全組織を横断して対象ユーザーを検索し、レポートにまとめる必要があります。

このエージェントは6つの Organization の Identity Center を順に検索して、対象ユーザーの存在有無と権限状態を一覧にしてくれます。ただし、**無効化操作自体はエージェントにやらせていません**。API の制約もありますが、退職者対応は影響が大きいので最終操作は手動にしています。

「調査・レポートはエージェント、実行は人間」という分担は、SRE の運用ではちょうどいいバランスだと感じています。

### aws-investigator / infra-guideline-checker

`aws-investigator` は AWS 環境の汎用調査エージェントで、MCP サーバーを4つ接続しています（Core / Billing / Pricing / AWS MCP Proxy）。`infra-guideline-checker` は Confluence に定義したインフラガイドラインに基づいて AWS 環境の準拠状況をチェックし、表形式でレポートを出力します。

## Claude Code 側の使い方

一方の Claude Code は、日常業務のワークフロー自動化に振り切っています。23個のスキルで朝会の Jira 確認、タイムブロック作成、スプリントセレモニーの準備、Capability 評価ドラフト生成、ブログ執筆支援まで幅広くカバーしています。

Hooks も活用していて、`AskUserQuestion` が呼ばれたら macOS 通知を飛ばしたり、`gh pr create` の前に「CodeX レビュー実施済みですか？」と聞いたりしています。

## なぜ棲み分けが生まれたか

結局のところ、**Kiro CLI の `aws` ビルトインツールは AWS 操作に最適化されている**ので、AWS 運用系のタスクは Kiro でやった方が体験が良いです。Read 系の自動承認やプロファイル自動切り替えは、MCP 経由ではなかなか再現できません。

一方、Claude Code は Opus モデルが使えることと、スキル・Hooks・エージェントの定義が柔軟なので、開発ワークフロー全体の自動化には向いています。

「どちらか一方」ではなく「得意分野で使い分ける」のが、今のところの自分の結論です。

# 現時点での制約・課題

正直に書いておくと、現時点の Kiro CLI にはいくつか制約があります。

**Specs / Powers / Autopilot が CLI 非対応**

Kiro の最大のウリである仕様駆動開発（Specs）が CLI では使えません。CLI版だけを使う場合、Kiro の最も差別化された機能を体験できないことになります。Powers も同様で、AWS CDK / SAM の Powers は IDE 版限定です。

**モデル選択の柔軟性**

Claude Code は Opus / Sonnet / Haiku を切り替えられますが、Kiro は Claude Sonnet 4.5 または Auto（複数モデルの自動選択）に限定されています。重いタスクに Opus を指定する、といった使い方ができません。

**コミュニティの声**

肯定的な意見がある一方で、「Cursor と比べてオートコンプリートが弱い」「複雑な要件を完全には理解しない」といった声も見られます。また、「仕様駆動開発を実践したら、結果バイブコーディングになった」という報告もあり、Specs の効果はプロジェクトや使い方に依存するようです。

# まとめ

Kiro CLI の「現在地」をまとめると、以下のような状況です。

- **コア機能は一通り揃っている**: Chat / Agents / MCP / Hooks / Steering / Skills / Sub-agents / ACP / `aws` ツールと、ターミナルエージェントとしての基本は充実
- **他ツールにない独自機能がある**: `aws` ビルトインツール / ACP / Autocomplete / Translate / Tangent Mode は Claude Code にはない
- **ただし Kiro 最大の武器は CLI に来ていない**: Specs / Powers / Autopilot は IDE 専用。CLI だけでは Kiro の本領を体験しきれない
- **Experimental 機能に将来性がある**: Knowledge / Tangent Mode / TUI など、面白い実験が進んでいる
- **他ツールとの併用が現実的**: 自分は Claude Code で日常業務ワークフロー、Kiro CLI で AWS 運用エージェントという使い分けに落ち着いた

個人的な結論としては、**AWS 環境の運用操作には `aws` ビルトインツールの体験が圧倒的に良く、SRE/インフラエンジニアなら触る価値がある**と感じています。一方、開発ワークフロー全体の自動化やモデル選択の柔軟性では Claude Code に分があるので、「どちらか一方」ではなく得意分野で使い分けるのが現時点でのベストだと思います。

Specs や Powers が CLI に来たタイミングで改めて評価したいところです。各機能の詳細な検証結果は、触り次第追記していきます。

# 参考

https://kiro.dev/docs/cli/

https://kiro.dev/docs/cli/reference/cli-commands/

https://kiro.dev/docs/cli/hooks/

https://kiro.dev/docs/cli/steering/

https://kiro.dev/docs/cli/acp/

https://kiro.dev/docs/cli/skills/

https://kiro.dev/docs/cli/experimental/

https://kiro.dev/docs/cli/reference/built-in-tools/

https://kiro.dev/docs/cli/migrating-from-q/

https://kiro.dev/changelog/cli/

https://kiro.dev/pricing/
