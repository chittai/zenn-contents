---
title: "AWS CloudFormation StackSetsでOrganizationsの機能を利用する方法"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","CloudFormation"]
published: true
---
# この記事について
仕事でStackSetsについて色々と調べる事があり、その時に`Organizations`の機能と連携させる方法について調べたのでまとめます。
Organizationsに限定せず、その時に調べた管理方法も載せています。
　
# AWS CloudFormation StackSetsとは？
CloudFormationはわかっている前提ですが、1つのAWS CloudFormationテンプレートを利用して、複数のリージョンにおけるAWSアカウントにスタックを作成できます。

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html

# Organizations で CloudFormation StackSets を使用する方法
StackSetsでは下記2種類の`アクセス許可`を選択することができます。

* サービスマネージドアクセス許可
* セルフサービスアクセス許可

ここで`サービスマネージドアクセス許可`の選択をすることで、スタックの展開先(スタックインスタンス)を`Organization`で管理することができます。

![](https://storage.googleapis.com/zenn-user-upload/uzw139edmgo4eru7h9gy5nqfbav5)

![](https://storage.googleapis.com/zenn-user-upload/zfqls3h9i95ui86rynrtj30rs05b)


## (ポイント)組織へのデプロイを選択して作成しても、編集時は組織単位(OU)へのデプロイしか選択できない
一度作成したStackSetを編集すると、`組織へのデプロイ`がなくなっています。

![](https://storage.googleapis.com/zenn-user-upload/f1xy6jeakygw3ubw3ju63zch5zhy)

OU単位やアカウント単位では指定することができますが、組織全体に反映したいケースもあると思います。そういった時は`OUID`に`RootID`を入れれば、OrganizationsのRoot配下のアカウント全てに適用されます。RootIDはOrganizationsの画面のでRootのARNにある`r-xxxx`の文字列です。

# StackSet更新するときってどうやってテストするべき？
StackSetsを更新して各スタックインスタンスに展開する場合、テンプレートの更新バージョンをテストすることを考える必要があるかと思います。そういうときのAWSの推奨のベストプラクティスについて記述します。

ベストプラクティスとしては単純で、すべてのスタックインスタンスを一気に更新するのではなく、事前にいくつかのテストアカウントを作成しておきそのテストアカウントに対して更新します。その後問題がなければ全てに展開指定気ます。

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/stacksets-bestpractices.html#w2aac29c21c10

# StackSetsのドリフト機能について
これは、特に注意することもないかもしれないですが、`StackSetsのドリフト機能`は各スタックインスタンスのドリフトを一括で実行したり、その結果を一元管理する機能です。

https://dev.classmethod.jp/articles/cloudformation-stacksets-drift-detection/

## 個別アカウントの修正は、StackSetsのテンプレートを更新しなければ再実行時に上書きされる
これはどういう意味かというと、StackSetsでスタックを展開した後、個別アカウントでリソースの変更があったとします。それを各アカウントで修正してドリフト状態を解消しても、後日StackSetsを再実行したらその変更は上書きされてしまいます。これは単純に、親であるStackSetsのテンプレートが正となるからです。そのため、StackSetsのテンプレートを更新すればよいのですが、他のアカウントへの影響も考える必要があります。なので、StackSetsで展開したリソースは個別で修正するのではなく、ちゃんと各アカウント同じ状態にしましょう。

# まとめ
Organizationsを利用するなら、RootIDは抑えておきましょう。運用によってはStackSets更新時に必要になります。それと、StackSetsで展開したリソースは各アカウント同じ状態で保つ必要がありますので、何をStackSetsで展開するのかはよく考えましょう。最後に、既存リソースの取り込みも書きたかったのですがそれは別の機会があれば書こうと思います。
