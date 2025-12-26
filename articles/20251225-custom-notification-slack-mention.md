---
title: "Amazon Q Developer in chat applications でユーザ/ユーザーグループへのSlackメンションを実現する"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Slack", "CloudWatch", "ChatBot", "amazonq"]
published: true
publication_name: "genda_jp"
---

# はじめに

AWSの監視アラートをSlackに通知する際、「特定のユーザーやグループにメンションを飛ばしたい」という要件はよくあるかなと思います。しかし、CloudWatch AlarmからSNS経由でSlackに通知する従来の方法では、メンションの実現が難しいという課題がありました。
**カスタム通知機能**を使ってCloudWatch AlarmからSlackへメンション付きの通知を送ることができるのでその方法をまとめました。

↓の画像が結果のイメージです。

![](https://storage.googleapis.com/zenn-user-upload/32ad4bc09583-20251226.png)

# Amazon Q Developer in chat applications （旧:AWS Chatbot）のカスタム通知

通常、Amazon Q Developer in chat applications はSNSトピックから通知を受け取り、定型フォーマットでSlackに投稿します。しかし、**カスタム通知（Custom Notifications）** を使うことで、SNSメッセージの内容を基にカスタマイズされた通知を送ることができます。この機能を使ってメンションをつけていきます。

https://docs.aws.amazon.com/chatbot/latest/adminguide/custom-notifs.html

## 実装方法

基本的なアーキテクチャは、CloudWatch Alarmから通知を飛ばすのではなく以下のようにAlarmのイベントをトリガーにEventBridge経由で通知をだします。

![](https://storage.googleapis.com/zenn-user-upload/ab78d801c2ca-20251226.png)


### 1. Amazon Q Developer in chat applications の設定

まず、Amazon Q Developer in chat applications でSlackワークスペースとの連携を設定します。実装のときはCDKを使うのですが、この手順は手作業でやる必要があります。

1. Amazon Q Developer in chat applications コンソールを開く
2. 「Configure new client」からSlackワークスペースを連携
3. 通知先のSlackチャンネルを設定
4. SNSトピックをサブスクライブ

### 2. SNSメッセージのフォーマットの設定

ここがポイントになります。CloudWatch AlarmからSNSに送信するメッセージに、以下のように入力トランスフォーマを利用します。 その入力テンプレートに **カスタム通知用のフィールド** を含めることでカスタム通知が利用できます。公式ドキュメントを確認してみると `version`,  `source`, `description` が必須となっています。 

![](https://storage.googleapis.com/zenn-user-upload/85e671012f06-20251226.png)


入力テンプレートには以下のような内容になります。

```json
{
  "version": "1.0",
  "source": "custom",
  "content": {
    "title": "🚨 アラート: xxが閾値を超えました",
    "description": "<@!subteam^(ユーザーグループのID)> <@ユーザID> 至急確認してください\n\nアラーム名: Highxx\n状態: ALARM\n詳細: xx",
    "nextSteps": [
      - "CloudWatchのメトリクスを確認",
      - "アプリケーションログを確認",
      - "必要に応じてロールバックを検討"
    ]
  }
}
```

### 3. Slackメンションの記法

今回の記事はここがメインなのですが、上位の入力テンプレート例にあるように `desctioption` にSlackメンションを追加しました。カスタム通知では、以下のメンション記法が使用できます。`@here` や `@channel` はそのまま利用できるようです。

| メンション対象 | 記法 |
|--------------|------|
| ユーザーグループ | `<@!subteam^グループID>` |
| ユーザー | `<@ユーザID>` |

## メンションの追加が完了

このようにカスタム通知機能を使うことで、CloudWatch AlarmからSlackへのメンション付き通知が実現できました。メンションいれたいという人がどのような作業が必要か理解する手助けになると嬉しいです！



#### Tips:ユーザーグループIDの確認方法

参考までに、Slackのユーザーグループ（例：@dev-team）のIDは、以下の手順で確認できます。

1. Slackでユーザーグループをメンションする（`@dev-team`）
2. 送信したメッセージを右クリック → 「リンクをコピー」
3. URLに含まれる`archives`の後ろの文字列がグループID


