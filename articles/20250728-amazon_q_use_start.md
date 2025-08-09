---
title: "Amazon_q_use_start"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


# はじめに
Amazon Q Developer CLI を導入し、検証しています。アップデートが多く陳腐化することも多いともいますが、使い始める際に必要なことを書き記しておきます。



# やるべきこと
基本的に設定のところは省きます。まずは、ルールの作成です。

## Amazon Q の最新化
```
brew update
brew upgrade amazon-q
```

## AmazonQ.mdの作成
まずはAmazonQ.mdの作成です。`/context show`を実行すると下記が表示されます。デフォルトでは特定のファイルが記載されています。

```
👤 Agent (q_cli_default):
    AmazonQ.md 
    README.md 
    .amazonq/rules/**/*.md 
```

グローバルコンテキストは--globalで指定することも可能です。

```
project folder 
  └AmazonQ.md
  └README.md
```

```
q chat
> /context add --global .amazonq/rules/security-standards.md
Added 1 path(s) to global context.
```




## Profile(CLI)の作成
## agentの作成
## Contextの作成
## MCPサーバの設定
## コンテキストの継続について会話の保存について
## 事例（練習）





