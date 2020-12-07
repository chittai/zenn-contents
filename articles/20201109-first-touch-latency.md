---
title: "ファーストタッチレイテンシーの現象を試してみた"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws"]
published: false
---

# Snapshotからリストアしたボリュームへの初回アクセスはパフォーマンスが落ちる
Snapshotからボリュームをリストアした場合、ブロックに初めてアクセスする時にS3にデータを取りに行きます。そのため、初回アクセスのパフォーマンスが落ちます。これをがファーストタッチレイテンシー(ファーストタッチペナルティ)と呼びます。

[Amazon EBS ボリュームの初期化](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/ebs-initialize.html)

今回はこの現象を確認していきます。

# 検証環境とシナリオ
1. Windowsサーバを一台たて、EBSボリュームでCドライブ(120GB:gp2)とDドライブ(120GB:gp2)の準備をします。
1. Cドライブに10GBのデータを作成します。
1. DドライブのSnapshotを作成します
1. 既存のDドライブをデタッチし、取得したSnapshotから作成したボリュームを新しくDドライブとしてアタッチします
1. ここで、CドライブからDドライブへデータを転送して時間を計測します

# 手順
環境構築のコマンドです。

## Cドライべへダミーファイルを作成
```
fsutil file createnew dummy.data (10737418240)
```
## DドライブのEBSSnapshotを作成
```
aws ec2 describe-volumes
aws ec2 create-snapshot --volume-id <voi_id>--tag-specification 'ResourceType=snapshot, Tags=[{Key=Name,Value=testsnapshot}]' --description "for test"
```
https://docs.aws.amazon.com/cli/latest/reference/ec2/create-snapshot.html

## リストア


## D->C

## D->S3

## 計測まとめ



