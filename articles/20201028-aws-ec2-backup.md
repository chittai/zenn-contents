---
title: "AWS BakcupとSystemsManager AutomationでEC2バックアップを運用する方法"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","EC2"]
published: true
---

# EC2バックアップ前に停止して、バックアップ取得後に起動したい
AWSではバックアップを取得する時に何かしらの方法で静止点を取得することを推奨されています。アプリケーションの機能で静止点を取る方法もありますが、やはりインスタンスを停止することは割と一般的だと思います。今までオンプレ使っていたりバックアップジョブを自前で実装していた人たちから見ると特に。

そこで、バックアップ前のEC2インスタンス停止からバックアップ取得、そしてEC2インスタンス起動までの処理をAWSのサービスを使用して実装していきます。

# 検証構成
EC2インスタンスを立てて、SystemManager AutomationとAWS Backupを利用します。処理の順番は、下記の通りです。

1. SystemManager Automation の機能でEC2インスタンスを停止
2. SystemManager Automation の機能でBakcupAPIを呼び出し、バックアップを取得
3. Backupジョブが正常に投げられたらEC2インスタンスを起動

![](https://storage.googleapis.com/zenn-user-upload/62oqy1o6vao6hq45c67jccrqmy3u)

# 使用するサービスについて少し説明します

## AWS Backup 
下記にAWSドキュメントのリンクを張っておきます。

[AWS Backup とは](https://docs.aws.amazon.com/ja_jp/aws-backup/latest/devguide/whatisbackup.html)

AWS Backupでは既存のサービスで提供されているバックアップ機能を利用して各サービスのバックアップを取得し、それを一元管理してくれるサービスです。取得時間を設定することもできるので、バックアップを取得するだけであればこのサービスを使用すれば問題ありません。

例えば、EC2インスタンスであれば`AMI`と`EBS Snapshot`の取得・管理をしてくれます。

:::message
EFSのバックアップを取得することもできます。対応サービスはリンクから確認できます。
:::

### バックアップを取得する
AWS Backupでは下記の方法を使用してバックアップを取得します。

* バックアッププランによる取得
* オンデマンドバックアップによる取得

![](https://storage.googleapis.com/zenn-user-upload/i3l0389qyppo89ausudtd3lyxptx)

既に書きましたが、今回はAutomationでBakcupAPIを実行します。そうすると、オンデマンドバックアップとしてジョブが投入されバックアップを取得します。

## AWS System Manager Automation
Automationの機能をドキュメントから引用します。

> Systems Manager オートメーションは、EC2 インスタンスおよび他の AWS リソースの一般的なメンテナンスとデプロイのタスクを簡素化します。

[AWS Systems Manager オートメーション](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/systems-manager-automation.html)

これだけだとわかりにくいのでもう少し見ていきます。Automationでは、ドキュメントという単位でタスクをまとめることができます。下記は`AWSによって管理されているドキュメント`です。

![](https://storage.googleapis.com/zenn-user-upload/vpmb1a0ae5uthh71669suy1n71w0)


これの`AWS-StartEC2Instance`を見てみます。

![](https://storage.googleapis.com/zenn-user-upload/tkz79c2irtszuisnrqgp2yji4lec)

このように、ドキュメントではStepという単位で実行されるアクションを指定します。このアクションはいくつか種類がありますのでリファレンスのURLを書いておきます。

[Systems Manager オートメーションアクションのリファレンス](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/automation-actions.html#automation-action-changestate)

今回使用するのは下の2種類です。

* aws:changeInstanceState
* aws:executeAwsApi 

では、実際に実装するところに入っていきます。

# ドキュメント作成前に何をしておく必要があるか
BackupAPIを実行する時に、Roleを指定します。なので、BackupからEC2を操作できるように事前にロールを作成しておいてください。

私は、`ユースケースの選択`で`AWS Backup`を指定し、ポリシーとして`AmazonEC2FullAccess`を付けました。

そしたらロールARNをコピーしておいてください。

あと、対象となる`インスタンスIDのメモ`をしておいてください。`StringList`で指定するため複数でも問題ありません。

# ドキュメントの作成しよう
AWS マネジメントコンソール画面で、`AWS SystemsManager`を選択します。左側のサービス一覧の一番したにある`共有リソース`の`ドキュメント`を選択します。
![](https://storage.googleapis.com/zenn-user-upload/2bnzzdf4pkyt02hhgb8qb59fnfpc)

選択した画面で上の方に`オートメーションを作成する`があるので選択してください。作成したドキュメントは`自己所有タブ`の画面に表示されます。

![](https://storage.googleapis.com/zenn-user-upload/x2q6nogihprugh59z8ezyt018uez)

ドキュメント名などを入れたらステップの作成に入ります。今回は3つのステップを作成します。

1. StopInstance(指定したEC2インスタンスを停止する)
2. Run_API(指定したAPIを実行する)
3. StartInstance(指定したEC2インスタンスを開始する)

まずは、①から見ていきます。アクションタイプにある`Change or assert instance State`が`aws:changeInstanceState`に該当します。指定したインスタンスのステータスを`Desired state`で指定したステータスに変更します。今回は停止のため`stopped`を指定してます。

対象のインスタンスは`StringList`で指定します。これは`- i-xxx`という記述になります。インスタンスIDを書くだけだとエラーで作成できません。

![](https://storage.googleapis.com/zenn-user-upload/8rjtaymk0hoj2my38tphow84gp1e)

次に②について確認します。ここではアクションに`Call and run AWS API actions`を指定します。これが`aws:executeAwsApi `に該当します。`Service`にはAPIを叩くサービスを指定します。今回はAWS Bakupなので`backup`になります。APIは実行するAPIコマンドを指定します。`StartBakcupJob`となります。

[StartBackupJob](https://docs.aws.amazon.com/ja_jp/aws-backup/latest/devguide/API_StartBackupJob.html)

ここで追加の入力に引数となる値をいれます。上記リンクを読むと必須の引数は3つあります。

* BackupVaultName
* IamRoleArn
* ResourceArn

`BackupVaultName`はバックアップ保存先の名前です。ここでは最初からある`Default`を指定してます。次の`IamRoleArn`はBackupからバックアップを取るサービスへのアクセス件げ付与されているロールARNを指定します。ここでさっき作成したロールARNを貼り付けます。3つ目は`ResourceArn`です。対象となるリソースのARNです。今回Backup対象となるEC2のIDのARNを指定してます。

![](https://storage.googleapis.com/zenn-user-upload/rmqeoyuvy389ym5qajrjnqg07ybf)

![](https://storage.googleapis.com/zenn-user-upload/cadsvrk1792o1veqviisnkfhi79o)


③は①の逆で、インスタンスの起動です。`Desired state`が変わるぐらいです。
![](https://storage.googleapis.com/zenn-user-upload/ulfxo07c0nm2d6liqm6imhkg2qke)

これでドキュメントの作成は完了です。

# ドキュメントの実行しよう
最後に実行する方法です。

SystemsManagerの画面の左側の真ん中あたりに`自動化`という項目があるので選択してください。
![](https://storage.googleapis.com/zenn-user-upload/28qp1af9heljm4solm096718eyai)

オートメーションの実行を選択し、その先で作成したドキュメントを選択してください。画面の一番下にいくと`次へ`というボタンがあるので選択し実行します。
![](https://storage.googleapis.com/zenn-user-upload/93y3iv8jh864uwugwilcdozqq8iq)

以上で実行まで完了です。終わったらEC2からAMIとSnapshotを確認してもいいですし、AWS Backupにジョブが投入されていることを確認してもいいと思います。

# 最後に
今回Automationを使用して見ましたが、他にもLambdaを使用したりすることもできると思います。ただ、AWSのサービスかつコーディングなしで運用できるのはやはり便利なのではないかなと思います。
