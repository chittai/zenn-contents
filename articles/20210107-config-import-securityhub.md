---
title: "「AWS Config ルールの評価結果を Security Hub にインポートする方法」を試す"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws"]
published: true
---

# AWS Config ルールの評価結果を Security Hub にインポートする
`自分で作成したConfig ルールも合わせてSecurityHubで管理する`場合、標準機能ではないため(SecurityHubと統合していないので)、仕組みを構築する必要があります。下記の記事ではその方法について書かれています。今回はこれを試していきます。

https://aws.amazon.com/jp/blogs/news/how-to-import-aws-config-rules-evaluations-findings-security-hub/

ここでは、GithubでCloudFormationのテンプレートと、Lambdaに展開するスクリプトが公開されています。

https://github.com/aws-samples/aws-securityhub-config-integration

このテンプレートを展開するだけだとエラーになってしまうので、対応方法について説明します。

## 手順
最初は、`cloudformation_template.yaml`を利用してスタックを構築します。ここでは、S3に配置されているスクリプトからLambda関数を作成します。

### CloudFormationで環境構築
1. そのために、事前にS3バケットを作成しておくテンプレートの中に記述しておく必要があります。バケットを作成したら、そのバケット名を`Propaties`->`Code`->`S3bucket`に記述します。
1. 次に、そのバケットにスクリプトをzipで圧縮して配置します。圧縮後のファイル名を`Propaties`->`Code`->`S3Key`に記述します。

```yaml
ConfigSecHubFunction:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        S3Bucket: '<S3BucketName>' #★要修正
        S3Key: '<ScriptFileName>.zip' #★要修正
      FunctionName : 'Config-SecHub-Lambda'
      Handler: 'lambda_function.lambda_handler'
      Role:
        Fn::GetAtt:
          - LambdaServiceRole
          - Arn
      Runtime: python3.7
      Timeout: 300  
```

参考までに、この対応をせずにスタックを作成した時に発生したエラーメッセージを載せておきます。
```
Error occurred while GetObject. S3 Error Code: PermanentRedirect. S3 Error Message: The bucket is in this region: eu-west-1. Please use this region to retry the request (Service: AWSLambdaInternal; Status Code: 400; Error Code:
```

### スクリプトの修正
これはzipで圧縮する前にやっておいても問題ないのですが、現状のスクリプトだとエラーが発生するため少し修正します。ここの`print(lambda_context)`はlambda_contextの定義がないためエラーとなります。出力しているだけなのでコメントアウトしてしまって問題ありません。

```python
def lambda_handler(event, context):
    """Begin Lambda execution."""
    print("Event Before Parsing: ", event)
    print(lambda_context)
    parse_message(event)
```
こちらも、参考までにこの対応をせずにLambdaで発生したエラーメッセージを載せておきます。
```
[ERROR] NameError: name 'lambda_context' is not definedTraceback (most recent call last):  File "/var/task/lambda_function.py", line 109, in lambda_handler    print(lambda_context)
```
これで準備完了です。あとは、CloudFormationからスタックを生成します。それが正常終了すれば、AWS Configからルールを追加するとConfigの評価結果がSecurityHubにインポートされるようになります。

# インポートした検出結果の表示
Configでルールを作成し、SecurityHubにインポートした結果はこのように表示されます。`会社はPersonal`、`製品はDefault`となっています。
![](https://storage.googleapis.com/zenn-user-upload/6uqlnyos67ui71t1fd9x5cy0pwpn)

# 検出結果の「会社」と「製品」を変えたい
この方法だと、製品名でのフィルタリングができません。そうなると管理がやり辛くなってしまいます。そのため、検出結果の`製品`の表示を変更したくなると思います。ただ、先に結論を書いておくと**このフィールドを変更することはできません**。

## 調査
まず、会社名・製品の情報がどのフィールドで管理されているのか確認します。

### get-fidingsの結果
`aws securityhub get-findings`の結果を確認します。

```
"ProductFields": {
    "aws/securityhub/ProductName": "Default",
    "aws/securityhub/CompanyName": "Personal",
    "aws/securityhub/FindingId": "arn:aws:securityhub:<region>:<account_id>:product/<account_id>/default/10160efd-xxxx-xxxx-xxxx-167c411901d4"
},
```
結果から抜粋すると、`ProductFields`の中に設定があることがわかります。ただ、Keyが`"aws/securityhub/CompanyName"`になっています。このようにプレフィックス`aws/`で始まるものは、AWSサービスで予約された名前空間のため上書きすることができません。

> プレフィックス "aws/" は、AWS 製品およびサービス専用の予約された名前空間を表します。このプレフィックスを、パートナー製品からの検出結果とともに送信しないでください。
https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/securityhub-findings-format-attributes.html#asff-required-attributes

# では、どのように管理するか
このままだと、インポートした結果の製品がすべてDefaultになってしまいます。Configだけなら管理の必要もないかもしれないですが、今後他のサービスをインポートするとなった時に管理がやり辛くなります。その場合、`ProductFields`に製品名を表すような情報を追記して、SecurityHubのコンソール画面からフィルタリングできるようにします。今回は、`ProviderName`に`Config`を設定してみます。

> Security Hub に送信する結果を生成する製品の名前を定義するには、ProductFields 属性を使用することをお勧めします。
https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/securityhub-custom-providers.html

```
"ProductFields": {
    "ProviderName": "<name of the product>",
}
```
そうしたら、下記のようにフィルタリング出来るようになります。

![](https://storage.googleapis.com/zenn-user-upload/peotpkrj4ojoukuo1i4pu0nnl53c)

# 結論
AWS ConfigはSecurityHubと統合されていないため、その仕組を構築する必要がありました。そして、会社・製品の表示をかえることはできないですが、他の情報を加えてフィルタリングできるようにすることで管理することが推奨されています。そのうち統合されるのではないかと思うので、気長に待ちましょう。
