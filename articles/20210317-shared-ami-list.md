---
title: "許可していないアカウントからリソースが共有がされている時の対応(AMI/EBS Snapshot)"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","EC2",""]
published: true
---

# 許可されていないアカウントから共有されたリソースとは？
AWSでリソースを共有するとなると[ResourceAccessManager](https://docs.aws.amazon.com/ja_jp/ram/latest/userguide/shareable.html#shareable-ec2)思い浮かべる人が多いと思います。ですが、AMIやEBS Snapshot、RDS Snapshotはこのサービス自体で共有する機能を持ちます。
https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/sharingamis-explicit.html

これらのリソースは、**共有元アカウント**が**共有先アカウント**を指定することで共有されます。もし、自分が全く知らないアカウントから共有されてしまった場合に、悪意をもって作られたイメージが勝手に共有されることも考えられます。これはセキュリティリスクとなりかねません。これを、**意図しない共有されたリソース**とこの記事では言うことにします。(RAMの場合は共有に承認が必要なので意図しない共有にはなりません)

# 意図しない共有の防ぎ方は？
この共有ですが、Publicなものを含めPriateなものも共有自体を防ぐことができなさそう(色々調べたけど見つからなかったの意)。なので、今回は共有されてしまった場合にどう対応するかについて検討していきます。

# 共有されてしまったらどう対応すればよいのか
主に、下記2つが対応策として考えられるかなと思います。

1. 共有されたリソースを削除する
1. 権限的にそのリソースを利用できないようにする

## 共有されたリソースを削除するのためには
まずはAMIを対象に確認してみます。削除する前に、その削除対象を確認する必要があります。削除対象のリソース一覧を確認していきます。下記コマンドを実行してください。ここで **<除外するアカウントID>** というのは **共有して問題ないアカウントです**。要は、リソース一覧から共有しても問題ないアカウントのリソースをgrepで除外することで、意図しない共有がされているリソースだけを表示します。

### 確認方法
```shell
aws ec2 describe-images --executable-users self --query 'Images[*].[ImageId,Name,OwnerId]' --output text | grep -v <除外するアカウントID>
```

出力例
```
ami-062d41cxxxxxxxxxx   account2-ami        001234567890 ★このアカウントを除外したい場合はこのアカウントIDをgrepで除外する
ami-0d2f1edxxxxxxxxxx   account3-ami        111234567890
```

同様に、EBS SnapshotとRDS Snapshotは下記のコマンドで実行することができます。
* EBS Snapshot
```
aws ec2 describe-snapshots --restorable-by-user-ids self --query 'Snapshots[*].[SnapshotId,OwnerId]' --output text | grep -v <除外するアカウントID>
```

* RDS Snapshot
```
aws rds describe-db-cluster-snapshots --include-shared --snapshot-type shared --query 'DBClusterSnapshots[*].[DBClusterSnapshotArn]' --output text | grep -v  <除外するアカウントID>
```

### 削除方法
あとは各リソースのコンソールから通常通りに削除すれば問題ありません。量が多い場合はコマンドで実行してもOKです。

## 権限的にそのリソースを利用できないようにする
ここでは、IAMを利用してユーザからAMIからEC2を作成したり、EBS SnapshotからVolume作成する事ができないようにします。ここではAMIとEBS Snapshotについて確認していきます。

### AMI
IAMは下記公式ドキュメントに記載されているとおりです。`Owner`の部分に許可するアカウントを追加してもらえれば問題ありません。

> 最初のステートメントの Condition エレメントは、ec2:Owner が amazon であるかどうかをテストします。(別のステートメントでユーザーに起動するアクセス許可が付与されない限り) ユーザーはその他の AMI を使用してインスタンスを起動することはできません。


```json
"Statement": [
      {
   "Effect": "Allow",
   "Action": "ec2:RunInstances",
   "Resource": [ 
      "arn:aws:ec2:region::image/ami-*"
   ],
   "Condition": {
      "StringEquals": {
         "ec2:Owner": "amazon"
      }
   }
}
```
https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/ExamplePolicies_EC2.html#iam-example-runinstances-ami

## EBS Snapshot
ここが少しハマったところなのですが、EBS Snapshotは下記のようにConditionを指定することで**全てのSnapshot**からのVolume作成を制限することができます。アカウントを指定したいのですが、SnapshotのArnが`arn:aws:ec2:*::snapshot/*`となっておりアカウントIDは省略されています。つまり、ここにアカウントIDを書いても動作しないためアカウントでの制御ができません。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Deny",
            "Action": "ec2:CreateVolume",
            "Resource": "*",
            "Condition": {
                "ForAnyValue:ArnEquals": {
                    "ec2:ParentSnapshot": "arn:aws:ec2:*::snapshot/*"
                }
            }
        }
    ]
}
```

そのため、もしEBS Snapshotの制限をしたい場合は別の方法を検討しなければなりません。(ConditionでOwnerIdを指定する方法やResourceTagを利用する方法を試してみたのですが、それも制御できなかったです)

# 結論
意図していないアカウントからAMIやEBS Snapshotなどでリソースが共有されてしまうことを防ぐ事はできません。そのため、定期的に確認して削除したり、そのリソースを権限的に利用できないようにしたり工夫が必要です。
