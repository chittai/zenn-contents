---
title: "Privated Hosted Zoneを他のアカウントと関連付ける方法"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws"]
published: true
---

# Private Hosted Zone(PHZ)とは
PHZはDNSのレコードを格納するコンテナです。PrivateとPublicがありますが、Privateだと名前解決をVPC内のみで行うことができます。作成時にドメインを指定しますが、VPC内でそのドメインとサブドメインのDNSクエリに対応することができます。

# アカウントAのPrivate Hosted Zoneに登録されたレコードをアカウントBから参照したい
まず、PHZはドメインとVPCを1:1で関連付けます。VPC内で完結するため、アカウントAのVPCからアカウントBのVPCの名前解決をすることはデフォルトではできません。方法として、PHZに別アカウントのVPCを関連付けるという操作を行います。これは、CLIから対応する必要があります。

# 今回の検証環境について
2つのアカウントを用意して、VPCAとVPCBを作成します。ちょっとわかりにくくなりましたが、黒点線での囲いはアカウント単位です(つまり、2つのアカウントが存在して、その中に環境を構築していることを表しています)

各VPCに紐づくPHZAとPHZBを作成し、各VPCに配置されたEC2インスタンスから別アカウントのEC2に対して、互いに名前解決ができるかを確認していきます。

![](https://storage.googleapis.com/zenn-user-upload/zm95qkz19ys8w8fcd2c6xul7a8og)

# 検証を構築する
## PHZを作成してレコード登録
アカウントAとBの両方でEC2とPHZを構築し、PHZにEC2のレコードを登録します。

### アカウントAのレコードと現時点での名前解決の状況
EC2インスタンのIPとFQDNです。
![](https://storage.googleapis.com/zenn-user-upload/p08m3822yybzpsedgakjupv1ccx2)

同じVPC内の名前解決はできますが、別アカウントのVPCの名前解決はできません。
![](https://storage.googleapis.com/zenn-user-upload/jvxxmwfmkn9wt8zjvv53fiagtpvd)

### アカウントBのレコードと現時点での名前解決の状況
ここの説明は上記と同じため割愛します。
![](https://storage.googleapis.com/zenn-user-upload/ark1oavikpuaf57j0vql9pedkzpx)

![](https://storage.googleapis.com/zenn-user-upload/oap0gsh7odjc4cmnmku3x68kso9q)

## PHZと別アカウントのVPCの関連付け
まずは、アカウントAのPHZにアカウントBのVPCを関連付けます。

最初はこんな感じです。このVPCはアカウントAのVPCです。
![](https://storage.googleapis.com/zenn-user-upload/wh1fr8ec8wbh8z5pbersumrvfiw9)

1. アカウントAで、実行IAMユーザの作成します。ユーザに`Route53FullAccess`を付与します。
2. 1.で作成したユーザで下記コマンドを実行する。つまり、アカウントA側で実行します。
```
aws route53 create-vpc-association-authorization --hosted-zone-id <AのPHZID> --vpc VPCRegion=ap-northeast-1,VPCId=<BのVPCID>
```
3.  アカウントBで、実行IAMユーザの作成し、ユーザに`Route53FullAccess`、`VPCFullAccess`を付与します。

4. 3.で作成したアカウントで下記コマンドを実行します。つまり、ここではアカウントB側で実行します。
```
aws route53 associate-vpc-with-hosted-zone --hosted-zone-id <AのPHZID> --vpc VPCRegion=ap-northeast-1,VPCId=<BのVPCID>
```

そうすると、こうなります。アカウントAのPHZにアカウントBのVPCを関連付けることができました。
![](https://storage.googleapis.com/zenn-user-upload/vbqvdtw5m6v2p8v2a3723d141uct)

次は、BのPHZにAのVPCを関連付けます。手順は同じです。

アカウントBで実行
```
aws route53 create-vpc-association-authorization --hosted-zone-id <BのPHZID> --vpc VPCRegion=ap-northeast-1,VPCId=<AのVPCID>
```

アカウントAで実行
```
aws route53 associate-vpc-with-hosted-zone --hosted-zone-id <BのPHZID> --vpc VPCRegion=ap-northeast-1,VPCId=<AのVPCID>
```

# 検証の結果をみてみる
検証結果を見てみます。先程はできなかった別アカウントの名前解決ができています。

A->B
![](https://storage.googleapis.com/zenn-user-upload/gsh8jrttnmxk7hwntrsbrprqqbuu)

B->A
![](https://storage.googleapis.com/zenn-user-upload/343mvljwmx3zwiultggtl5vhuyk7)
