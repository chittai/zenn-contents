---
title: "ファーストタッチレイテンシーの現象を試してみた"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","EBS"]
published: true
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

:::message
空のボリュームだとファーストタッチレイテンシーが発生しないので、空である場合は何か適当にファイルを配置してください。
:::

## DドライブをEC2インスタンスからデタッチ
```
aws ec2 detach-volume --volume-id <vol_id>
```
https://docs.aws.amazon.com/cli/latest/reference/ec2/detach-volume.html

## DドライブをSnapshotからリストア
```
aws ec2 create-volume --snapshot-id <snapshot_id> --volume-type gp2 --availability-zone ap-northeast-1d 
```
https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html

## DドライブをEC2にアタッチ
```
aws ec2 attach-volume --device <device> --instance-id <instance_id> --volume-id <vol_id>
```
https://docs.aws.amazon.com/cli/latest/reference/ec2/attach-volume.html

## CドライブからDドライブへ転送
```
$watch = New-Object System.Diagnostics.StopWatch
$watch.Start()
Copy-Item  C:\Users\Administrator\Desktop\dummy.data D:\
$watch.Stop()
$t = $watch.Elapsed
"{0} min {1}.{2} sec" -f $t.Minutes,$t.Seconds,$t.Milliseconds
```

## 計測まとめ
| 回数 | 経過時間|
| :--- | :--- |
| 1回目 | 22 min 56.65 sec |
| 2回目以降 | 1 min 19.341 sec |

## まとめ
初回とそれ以降ではかなり差が開きました。ただ、本来であればデータを作成したボリュームのSnapshotを作成し、そのボリュームをリストアしてデータ転送を確認したかったのですが、それだとファーストタッチレイテンシーが発生せず、何か理解を間違っているのか原因がわかっていません。。

