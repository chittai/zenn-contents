---
title: "OpenSearch ServerlessにEC2からcurlでIndexを作成する方法"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","OpenSearch"]
published: true
---

# やりたいこと
OpenSearchServerlessにEC2からIndexを作成します。OpenSearchServerlessへの接続はPublicとPrivateから選ぶことができるのですが、今回はVPCの中にVPCエンドポイントを作成したPrivate接続で検証したいと思います。Postmanなどを利用する方法もありますが、今回はcurlコマンドで実行する方法をまとめます。

このIndexは Amazon Bedrock Knowledge Base のベクトルデータベースとして使用するため、Index作成時に必要な設定も加えます。

![](/images/20240814-opensearchserverless-createindex/image-architecture.jpg)

# やってみた結果

### 事前リソースの作成
まずはcurlを実行するEC2の準備をします。OpenSearch Serverlessのリソースは、VPCエンドポイント、Collection、データアクセスポリシー、ネットワークポリシー、暗号化ポリシーといった必要なものを作成しておきます。

データアクセスポリシーは「EC2に設定したIAMロールをPrincipalとして、Collection/Indexの操作を許可する」ように設定してください。以下は一例ですが、
コレクション `test-collection` への操作と `test-collection` に作成されるIndexすべてに対して操作をPrincipal `"arn:aws:iam::123456789012:role/<EC2のロール>"` に許可しています。今回は、Indexを作成するためResourceType indexに対して `aoss:CreateIndex` が許可されていればOKです。


```
[
   {
      "Description": "Rule 1",
      "Rules":[
         {
            "ResourceType":"collection",
            "Resource":[
               "collection/test-collection"
            ],
            "Permission":[
               "aoss:CreateCollectionItems",
               "aoss:UpdateCollectionItems",
               "aoss:DescribeCollectionItems"
            ]
         },
         {
            "ResourceType":"index",
            "Resource":[
               "index/test-collection/*"
            ],
            "Permission":[
               "aoss:*"
            ]
         }
      ],
      "Principal":[
         "arn:aws:iam::123456789012:role/<EC2のロール>"
      ]
   }
]
```
https://docs.aws.amazon.com/ja_jp/opensearch-service/latest/developerguide/serverless-data-access.html#serverless-data-access-syntax


EC2に設定したIAMロールには `aoss:APIAccessAll` を設定してください。データアクセスポリシーで許可されているPrincipalに `aoss:APIAccessAll`を設定することでOpenSearch Serverlessの中身へ操作が可能になります。以下は一例ですが、 `<collection-id>` には作成したCollectionのIDをいれてください。

```
{
    "Version": "2012-10-17",
    "Statement": [
         {
            "Effect": "Allow",
            "Action": "aoss:APIAccessAll",
            "Resource": "arn:aws:aoss:region:account-id:collection/<collection-id>"
        }
    ]
}
```

https://docs.aws.amazon.com/ja_jp/opensearch-service/latest/developerguide/security-iam-serverless.html#security_iam_id-based-policy-examples-data-plane


### EC2からcurlでOpenSearch Serverlessへコマンドを実行
まずは、EC2からcurlコマンドで操作を実行するときに、AWS SigV4でリクエストに署名をする必要があります。IMDSv2を使用してインスタンスメタデータにアクセスしクレデンシャル情報をもってきて、そこから各キーの情報を変数に格納します。SERVICEには `aoss` を格納します。

```
token=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
role=$(curl -s -H "X-aws-ec2-metadata-token: $token" http://169.254.169.254/latest/meta-data/iam/security-credentials/)
credential=$(curl -s -H "X-aws-ec2-metadata-token: $token" http://169.254.169.254/latest/meta-data/iam/security-credentials/${role})
AWS_ACCESS_KEY_ID=`echo $credential | jq -r ".AccessKeyId"`
AWS_SECRET_ACCESS_KEY=`echo $credential | jq -r ".SecretAccessKey"`
AWS_SESSION_TOKEN=`echo $credential | jq -r ".Token"`
REGION="ap-northeast-1"
SERVICE="aoss"
```

次に、OpenSearch Serverlessのインデックスを作成するのですが、これはコンソールからcurlで実行するパラメータを取ってくることができます。

![](/images/20240814-opensearchserverless-createindex/image-aoss-console-1.png)
![](/images/20240814-opensearchserverless-createindex/image-aoss-console-2.png)


curlコマンドについては、上記をそのまま貼り付けてもうまくいかないため、以下のように修正します。

```
curl -v -k -X PUT 'https://<endpointID>.ap-northeast-1.aoss.amazonaws.com/test-index' \
-H "Content-Type: application/json" \
-H "X-Amz-Content-Sha256: 6dc7b8891721a27cc7c03447c696fd991d591b609152bf20b564a4fa2c6b52d6" \
-H "X-Amz-Security-Token: ${AWS_SESSION_TOKEN}" \
--aws-sigv4 "aws:amz:${REGION}:${SERVICE}" \
--user "${AWS_ACCESS_KEY_ID}:${AWS_SECRET_ACCESS_KEY}" \
-d '{"settings":{"index":{"knn":true,"knn.algo_param.ef_search":512}},"mappings":{"properties":{"test-index":{"type":"knn_vector","dimension":1536,"method":{"name":"hnsw","engine":"faiss","parameters":{"m":16,"ef_construction":512},"space_type":"l2"}},"AMAZON_BEDROCK_METADATA":{"type":"text","index":"false"},"AMAZON_BEDROCK_TEXT_CHUNK":{"type":"text","index":"true"}}}}'
```

ハマったポイントとしては、AWS SigV4で署名をするときにリクエストペイロードを指定する場合に `X-Amz-Content-Sha256` がないと、403で怒られてしまいます。 `-d` でリクエストボディーを渡すときは必ず入れるようにしてください。

https://docs.aws.amazon.com/ja_jp/opensearch-service/latest/developerguide/serverless-clients.html#serverless-signing
