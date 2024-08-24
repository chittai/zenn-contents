---
title: "AWS NetworkFirewall(集約型の展開モデル)における送信元アカウントの確認"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["networkfirewall"]
published: true
---

# はじめに
AWS NetworkFirewallはデプロイモデルを紹介する[記事](https://aws.amazon.com/jp/blogs/news/networking-and-content-delivery-deployment-models-for-aws-network-firewall/)があります。社内における通信を検査したい場合は**集約型の展開モデル**を利用することになると思います。ただ、送信元の情報を知りたい場合はNetworkFirewallで検知するイベントにはアカウントIDなどが含まれていないため、送信元IPから判断するしかありません。

共通システムがあったり、オンプレとの通信が必要な場合はCIDRが重複することがないように設計されていると思いますので、CIDRとアカウントの管理情報があればIPから送信元のアカウントを特定できます。

# やったこと
## 前提
最初にDynamoDBなどにCIDRとアカウントの紐づけ情報を格納しておきます。今回は、DynamoDBを利用してCIDRとVPCID、アカウントIDの情報を事前に整理してあります。
EventBridgeでルールを作成しVPCで`CreateVpc`や`AssociateVpcCidrBlock`アクションを検知してVPCにCIDRが設定されたらDynamoDBに情報を格納する関数を用意して実装しました。以下、ルールの例です。

```
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": ["AssociateVpcCidrBlock", "CreateVpc"]
  }
}
```

イベントを検知したらLambda関数を呼び出します。関数内では検知したイベントから必要な情報を抜き出し、以下のようにDynamoDBに書き込みます。
```
response = table.put_item(
    Item={
        'vpc_id': vpc_id,
        'cidr': cidr_block,
        'account_id': account_id
    }
)
```

## CIDRの範囲か確認する方法
NetworkFirewallでイベントを検知した際に、CloudWatchLogsに書き出してLambdaを呼び出します。その呼び出された関数で以下を用いてIPがCIDRの範囲であるかを確認しています。IPとCIDRを渡して範囲内でれば`true`範囲外であれば`false`を返します。

CIDRはDynamoDBに格納されている前提ですので、CIDRとアカウントIDの情報を全て取得した上で各CIDRに対して同様の判定をしています。

```
def check_ip_in_cidr(ip, cidr):
    return ipaddress.ip_address(ip) in ipaddress.ip_network(cidr)
```

trueとなるCIDRに紐づくアカウントIDを返すようにすることで、IPから送信元アカウントを導き出します。

# もうちょっと補足

NetworkFirewallのログについて確認します。

今回は、Suricataでルールを作成してそのルールに引っかかったものを検知したいと考えていました。そこで、簡単に以下のルールを作成しています。

```
drop http any any -> any any (msg:"Suricata Test Rule Detected"; content:"X-Test-Header: SuricataTest"; http_header; sid:1000001;)
```

以下のコマンドで検知するようにしています。

```
curl -v -H "X-Test-Header: SuricataTest" <送信先IP>
```

そうすると、以下のようなログが出力されます。アカウント情報などはありません。

### NetworkFirewallログのサンプル
```
{
    "firewall_name": "NetworkFirewall",
    "availability_zone": "ap-northeast-1a",
    "event_timestamp": "1724379571",
    "event": {
        "tx_id": 0,
        "app_proto": "http",
        "src_ip": "<送信元IP>",
        "src_port": 55300,
        "event_type": "alert",
        "alert": {
            "severity": 3,
            "signature_id": 1000001,
            "rev": 0,
            "signature": "Suricata Test Rule Detected",
            "action": "blocked",
            "category": ""
        },
        "flow_id": 553432066023341,
        "dest_ip": "<送信先IP>",
        "proto": "TCP",
        "http": {
            "hostname": "<送信先IP>",
            "url": "/",
            "http_user_agent": "curl/8.5.0",
            "http_method": "GET",
            "protocol": "HTTP/1.1",
            "length": 0
        },
        "dest_port": 80,
        "timestamp": "2024-08-23T02:19:31.848185+0000"
    }
}
```
