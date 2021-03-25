---
title: "AMI取得時にパラメータストアに最新のAMI_IDを格納する方法(+DRサイトへの展開)"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","SystemsManager","AWSBackup","Lambda","EventBridge"]
published: true
---

# 常に最新のAMI IDを保持しておきたい
障害発生時にCloudFormationでEC2を構築する場合、最新のAMIを指定して構築したいケースは多いと思います。色々やり方はあると思いますが、今回は最新のAMI IDの情報を**常にParameterStoreに格納**してそれを参照する方法を検討していきます。

:::message
もっと良い別の方法や、後続で出てくる処理の最適化方法はあるかもしれませんが、あくまでも参考として見ただければと思います。
:::

# やりたいこと
まずは、AWS Backupを使用してAMIを取得し、それを契機にParameterStoreにAMI IDの情報を格納します。最初はメインサイトで実行し、これをDRサイトに展開していくことを目標とします。

# 構成図(メインサイトのみ)
![](https://storage.googleapis.com/zenn-user-upload/sb5bmq2mbfimk4r4s3e98txq3n7c)

処理番号の説明をします。
1. AWS BackupでEC2のバックアップを取得
1. AMI作成時の`CreateImage`のアクションをEventBridgeのルールで検知
1. 検知後にLambdaの関数を実行
1. Lambda関数でParameterStoreに、新しく作成されたAMIのIDを格納
1. CloudFormationのテンプレート内で対象のパラメータを参照する

# メインサイトにおける利用サービス毎の実装の説明
ここでは、実際にどのように実装するのかやポイントについてまとめていきます。

## AWS Backup
AWS Backupはバックアッププランを作成し、その中にバックアップルール(`dr-test`)を作成します。そして、そのプランに紐づくリソースを指定します。今回はAMIを使用した検証なので、EC2インスタンスを割り当てます(`prilinux`)。

![](https://storage.googleapis.com/zenn-user-upload/rplfpv86bx1typxoeu8o2vlyj2uh)

## Lambda
ここでは、`boto3`を使用してParameterStoreにAMI IDを格納します。ここで一つポイントになるのが**パラメータを削除してから再度作成している**ことです。

これは、CloudFromationのテンプレートでParameterStoreの値を参照するときは、パラメータのバージョンを指定する必要があるためです。パラメータを更新するだけだとバージョンがどんどんインクリメントされてしまうので、このままではCloudFormationのテンプレートで指定しているバージョンも修正しなければならなくなります。そのため、CloudFormationのテンプレートの値を変えなくて良いように常に最新パラメータが同じバージョン(**今回だと1**)を指すようにしています。

```python
import json
import boto3

# Region
REGION = 'ap-northeast-1'

def lambda_handler(event, context):
    instance_id = event['detail']['requestParameters']['name'].split("_")[1]
    client = boto3.client('ssm', region_name=REGION)
    response_delete = client.delete_parameter(
    Name = instance_id
    )
    response_put =  client.put_parameter(
    Name = instance_id,
    Value = event['detail']['responseElements']['imageId'],
    Type = 'String',
    Overwrite = True
    )
```

ちなみに下記の処理は、AWS BackupでAMIを取得するとCloudTrailにCreateImageを実行した時のログが出力されます。その中にある`name`の値が`AWSBackup_<instanceId>_xxxxxx`という構成になるため、この文字列の中からインスタンスIDだけを抜き出しています。
```python
instance_id = event['detail']['requestParameters']['name'].split("_")[1]
```

このインスタンスIDを利用して、パラメータを作成します。こうすることでインスタンス毎にパラメータを持つことが可能です。

## EventBridge
EventBridgeでやりたいのは、**AMIが作成されたらメインの処理をするLambda関数を実行する**ことです。`EventBridge -> イベント -> ルール` へ移動し、ルールを作成します。イベントバスは今回は`default`を使用します(AWSサービス間の連携ではdefaultを利用するのが基本のためです。イベントバスの使い分けの詳細はBlackBeltなどで確認してください。)。

下記に、イベントパターンとターゲット指定の設定例を載せます。イベントパターンの定義では、`eventName`を`CreateImage`にすることでAMIが作成されたときにEventBridgeのルールで検知できるようになります(対象インスタンスの管理はLambda側で実装済み)。

ターゲットは作成したLambda関数の名前を指定してください。これにより、AMIが作成されたことを契機にLambdaによるパラメータ再作成処理を実行することができます。

ちなみに、EventBridgeではAWS管理のイベントがあるのですがAMI作成のイベントはなかったのでCloudTrailから持ってきています。

![](https://storage.googleapis.com/zenn-user-upload/ih9kyi01uu28osvzy606eeiar8pc)

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": ["CreateImage"]
  }
}
```

![](https://storage.googleapis.com/zenn-user-upload/z7f7jly3iykikyyue5ma1ujhaqh3)

## Parameter Store
パラメータストアには、事前に、Lambdaで再作成するパラメータと同じ名前でパラメータを作成しておいてください。これは、Lambdaの作成の前に削除するため、パラメータがないとLamdaがエラーになるための対応です。この値がAMI作成のたびに削除 -> 作成されて常に最新のAMI IDが入るようになります。

![](https://storage.googleapis.com/zenn-user-upload/29s6kah9fhkwyf364o27jiqybjao)

## CloudFormation
ものすごく簡単なテンプレートですが、EC2作成用のテンプレートです。`{{resolve:ssm:<instanceId>:1}}`がParameterStoreの値を参照している部分です。Lambdaのところでも書きましたが、最後の数字はパラメータのバージョンを指定してます。この書き方では、インスタンスIDを直書きしているため、インスタンス毎にパラメータが必要になってしまいます。こういうケースではパラメータを外だしにしたほうが良いと思います。(今回はそこまで扱わない)

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: A simple EC2 instance
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: "{{resolve:ssm:<instanceId>:1}}"
      InstanceType: t2.small
```

メインサイトだけの場合は、上記で設定完了です。あとは、毎時バックアップが取得されるため、そのタイミングで取得されているAMI IDを確認し、ParameterStoreにその値が入っているか確認してください。

# DRサイトへの展開方法
今まではメインサイト側での対応を前提としてきました。次は、DRに展開する場合について検討してみます。DRに展開するには下記の対応が必要です。

1. 基本はメインサイトと同じ構成の環境を構築する
1. DRサイトにAMIを転送する
1. DRサイトにAMIが転送されたら、DRサイトのパラメータストアに最新のAMI IDの情報を格納する

ここで変更が必要なのは、**メインサイトのAWS Backup**と、EventBridge、Lambdaです。BackupはDRサイトに転送するように設定を追加します。EventBridgeはどのようなイベントが発生しているのか見直すところから必要です。今回はメインサイトはCreateImageですが、DRサイトでは`CopyImage`が対象イベントとなります。Lambdaはコードの中にあるリージョン指定を変更します。

## AWS Backup(メインサイト)
バックアップルールにて、リージョンコピーを行う設定を追加します。これで、別リージョンの指定ボールとにデータがコピーされます。
![](https://storage.googleapis.com/zenn-user-upload/avmogbtffzlwo7zgcji3vpfrzc2x)

## EventBridge
EventBridgeはイベントが　`CopyImage`に変わるためその`eventName`を指定します。これで、メインサイトからDRサイトにAMIがコピーされてきたタイミングでイベントを検知することができます。それ以外の設定はメインサイトと同じで問題ありません。
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": ["CopyImage"]
  }
}
```

## Lambda
Lambdaでは、下記コードでリージョンを指定しているため、DRサイトに変更します。今回はシンガポールをDRサイトにしたので`ap-southeast-1`と変更しています。

```python
# Region
REGION = 'ap-southheast-1'
```

これでDR側も設定が完了です。あとは同じ様にAMIのコピーが走ったあとにDRサイトのパラメータストアにあるAMI IDの値が最新になっているか確認します。

# まとめ
既存のサービスの組み合わせでできますが、自分で実装するべき場所がいくつかあります。今回はエラーハンドリングとかしていないのですごく単純な構成になっていますが、実際はもう少し考慮しなければならない箇所があるかなとは思います。他にもRDSだったりを検討す必要がでてくるかもしれないですが、同じような構成でいけるとは思います。

