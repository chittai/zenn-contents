---
title: "Systems Manager AutomationでリソースID以外でターゲットを指定する方法"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","Systems Manager"]
published: true
---
# SystemsManager Automationのドキュメントでリソース指定にIDを使いたくない
AWSではSystemsManager Automation(以下、SSM Automation)という機能があります。その機能を利用したEC2バックアップ方法について記事を書きました。
実際はこのドキュメントをメンテナンスウィンドウを利用して定期的に実行しています。

https://zenn.dev/chittai/articles/20201028-aws-ec2-backup

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/systems-manager-maintenance.html

ここでの運用はバックアップ対象を直接ドキュメントに記述しています。ですが、これだと困ることがあります。例えば、**EC2/Volumeのリストアを行い、リソースIDが変わるケース**です。
リストアする毎にリソースIDを修正しなければなりません。それは耐えられないので、直接リソースIDを記入する運用を変えたいと思います。

ここで、ポイントとして上げられるのは下記になります。
1. 操作対象のリソースにTagをつける
1. Tagベースでリソースグループを作成する
1. メンテナンスウィンドウのターゲットでリソースグループを指定する
1. メンテナンスウィンドウのタスクで疑似パラメータを使用する
1. 1タスクで操作する対象は1種類のリソースとする(EC2,EBSなどちゃんと分けて管理する)

# 構成について
まず、構成について紐解きます。`Automationのタスクを定期的に実行する`には、`メンテナンスウィンドウを利用`する必要があります。このメンテナンスウィンドウでは
`タスクという単位で操作を登録`することができます。そして、操作対象を`ターゲットという単位で登録`することができます。このターゲットは`リソースグループ単位でもEC2インスタンス単位でも登録できます`。

そして、リソースグループはTagベースでの登録が可能です。なので、操作したい対象に同じタグをつけ、リソースグループでまとめることでリソースIDを直書きせずに
対象を指定することができます。

## 対象リソースにTagをつける
今回は、`linux`, `windows2019`というインスタンスに`Automation:Yes`というタグを追加しました。このタグベースでリソースグループを作成します。

![](https://storage.googleapis.com/zenn-user-upload/f7mfz09c4f4lg0kr95tgwt9v4kl0)

![](https://storage.googleapis.com/zenn-user-upload/cur7ddeewetauh611la3ewuream6)



## Tagベースでリソースグループを作成する
リソースグループ作成画面からTagベースでリソースグループを作成します。

![](https://storage.googleapis.com/zenn-user-upload/car6o4q2lku6zava9x9hr0q151sf)

![](https://storage.googleapis.com/zenn-user-upload/9gg99qxe9g9qwrkp6h3oqau626kf)


## メンテナンスウィンドウのターゲットでリソースグループを指定する
メンテナンスウィンドウのターゲットで先ほど作成したリソースグループを登録します。

![](https://storage.googleapis.com/zenn-user-upload/692r7spt6qige5bmhbvoutnh6ewd)

![](https://storage.googleapis.com/zenn-user-upload/vju4bft7aoj7wz1uk938nkaowmvh)


## メンテナンスウィンドウのタスクで疑似パラメータを使用する
ここからも大事なポイントとなります。次に、タスクを登録します。今回は`AWS-StopEC2Instance`を実行します。想定される動作は、先程作成したリソースグループに登録されているEC2インスタンスすべてが停止されることです。

![](https://storage.googleapis.com/zenn-user-upload/7upf2n6w629t1kqabaldfdejob8c)

先程登録したリソースグループのターゲットを登録します。

![](https://storage.googleapis.com/zenn-user-upload/o2hmdkh4ptbqjs9p23h43e8688e9)


リソースグループを操作するのに必要な権限をロールに付与します。
それと、EC2インスタンスを停止できる権限があるロールを選択します。(今回はAdministrator権限がついたロールを使用しています)

```json
}
"resource-groups":"GetGroupQuery",
"resource-groups":"GetGroup",
"resource-groups":"GetTags"
}
```

![](https://storage.googleapis.com/zenn-user-upload/w8eg4e04jcmimzlx9qqm6me8qsav)

ここで一番大事なのは、`InstanceId`のボックスに`{{RESOURCE_ID}}`を記述することです。これは疑似パラメータといいます。この疑似パラメータを
記述することで、リソースグループに登録したリソースのID情報が`AWS-StopEC2Instance`に自動で渡されて実行されます。動作として、各リソースに対して
順番に子タスクが作成されそれぞれ実行されます。

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/mw-cli-register-tasks-parameters.html

![](https://storage.googleapis.com/zenn-user-upload/ov14nils071qeznjxrri8kludes5)

以上で完了です。

# 感想
`{{RESOURCE_ID}}`が登録されたリソースを順に置き換わるので、複数のリソースタイプが登録されていると、ドキュメントの引数と型があわずに基本失敗します。
なので、リソースごとにタスクとリソースグループを登録するのが良いかと思います。




