---
title: "PrivateLinkについて調べてまとめてみた"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","PrivateLink"]
published: true
---

# やりたかったこと
AWSでは`PrivateLink`を用いることで、VPC内のサービスからインターネットを経由せずに、VPC外のAWSのサービスへアクセスすることができます。今回は、PrivateLink利用時の仕組みの理解が少し曖昧だったので調べがてら整理してみました。

## PrivateLinkとは何か
サービスのページに記述があります。

> AWS PrivateLink は、トラフィックをパブリックインターネットに公開することなく、VPC、AWS のサービス、およびオンプレミスネットワーク間のプライベート接続を提供します。

https://aws.amazon.com/jp/privatelink/?privatelink-blogs.sort-by=item.additionalFields.createdDate&privatelink-blogs.sort-order=desc

PrivateLinkというのはサービスを指すのですが、このプライベートなアクセスを実現する機能として**インタフェースVPCエンドポイント**というものを作成します。

## VPCエンドポイントとは
ここからはVPCエンドポイントとは何かについて説明していきます。VPCエンドポイントは、VPC内からVPC外のサービスへアクセスするときのエントリポイントになります。ここで、大事なのはVPCエンドポイントは下記に記載した3種類があるということです。ちなみに、PrivateLink=VPCエンドポイントと解釈してしまうのはNGです。PrivateLinkで利用できるVPCエンドポイントは、`2.InterfaceEndpoint`, `3.Gateway Load Balancer Endpoint`になります。

1. Gateway Endpoint
1. Interface Endpoint
1. Gateway Load Balancer Endpoint

今回は`2.Interface Endpoint`に絞って整理していきます。

## Interface Endpoint の仕組み
![](https://storage.googleapis.com/zenn-user-upload/1ezynh5nsr1ecqdf5cpedd3k6kfv)
https://docs.aws.amazon.com/ja_jp/vpc/latest/privatelink/vpce-interface.html#vpce-private-dns

公式のドキュメントから参照している図ですが、この記載の通りInterface EndpointはENI(Elastic Network Interface)として作成され、サブネットに紐付きます。別のサブネットからでもこのENIにアクセスすることができればInterface Endpointを介してVPC外のサービスにアクセスすることができます。基本的に同じVPC内であればSecurityGroupで制御していると思うのでそこで通して上げれば良いと思います。

VPCエンドポイント(以降ではVPCエンドポイントはInterface Endpointを指す)を作成すると、サービスとの通信に利用できるVPCエンドポイント固有のDNSホスト名が生成されます。これらの名前には、VPCエンドポイントID、アベイラビリティー・ゾーン名、リージョン名が含まれます。例えば、vpce-1234-abcdev-us-east-1.vpce-svc-123345.us-east-1.vpce.amazonaws.comのようになります。

ただ、PrivateDNSという機能があり、これを有効化すると、VPCにPHZ(PrivateHostedZone)が関連付けられます。このPHZに、作成したVPCエンドポイントのサービスのデフォルトDNS名（例：ec2.us-east-1.amazonaws.com）のレコードセットが含まれます。これにより、VPC内のエンドポイントのプライベートIPアドレスに解決することができるようになります。VPCエンドポイント固有のDNSホスト名ではなく、デフォルトのDNSホスト名を使ってサービスへのリクエストを行うことができるようになると、既存のアプリケーションがAWSサービスへのリクエストを行っている場合、構成の変更を必要とせずに、VPCエンドポイントを介して引き続きリクエストを行うことができます。ちなみに、S3のInterfae EndpointはPrivateDNSに対応していないため固有のDNSホスト名を利用する必要があります。

## クライアントVPN接続時
クライアントVPNで接続したときも、デフォルトDNSホスト名の名前解決ができればVPCエンドポイントにアクセスすることができます。たとえば、Route53のInboundEndpointを利用してDNSクエリをVPC内のResolverに転送するようにしたり、クライアントVPNエンドポイントを作成するときに指定できるDNSサーバにエンドポイントのレコードを登録したりすることで対応が可能です。名前解決ができればよいので、hostsファイルに記載しても問題ありません。

## コマンド実行
AWS CLIのコマンド実行時に通常はデフォルトDNSホスト名が利用されるのですが、PrivateDNSが対応していないサービスもあるといいました。そういう場合は`--endpoint-url`オプションを利用してhttps://`固有のDNSホスト名`を指定したコマンドを実行することでサービスが利用できます。

