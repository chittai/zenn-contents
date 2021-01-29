---
title: "VPCエンドポイント(PrivateLink)を利用して、CodeCommitにクロスアカウントアクセスする際の注意点"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","codecommit"]
published: true
---

# 別アカウントのCodeCommitにVCPエンドポイント経由でアクセスしたい
今回、CodeCommitにクロスアカウントアクセスする際に、VCPエンドポイントを利用してクローズドな環境でのアクセスを目指していきたいと思います。

難しいことはないのですが、1点だけハマったポイントがありましたのでその共有用の記事となります。最初に書いておくと、**VPCエンドポイント経由で別アカウントのCodeCommitリポジトリにクロスアカウントアクセスしようとすると、応答がなくなる** という事象です。これは、`AssumeRole`する際のstsコマンドを実行による問題となります。ですが、この記事では、**あくまでもCodeCommitへクロスアカウントアクセスするという操作**をベースに説明していきます。

:::message
上記事象についてだけ確認したい場合は、環境構築の情報などは飛ばしてしまって、「CodeCommitのリポジトリをクローンする」から読んでいただければ問題ありません。
:::

# 構成図
参考リンクにエンドポイントを追加しただけですが、この構成で検証していきます。
https://dev.classmethod.jp/articles/codecommit-cross-access/
![](https://storage.googleapis.com/zenn-user-upload/2zejhkami0wzuvm6w1j1e1kcf4hk)

# 環境構築
まず、前提としての環境構築になります。この辺は記事が沢山あるので改めて説明はしませんが、下記リンクを確認しながら対応していただければ問題なく環境準備はできるかと思います。

## リポジトリとかロールの作成
https://docs.aws.amazon.com/ja_jp/codecommit/latest/userguide/cross-account.html

1. アカウントAのCodeCommitにリポジトリを作成する
1. アカウントAにスイッチロールのためのロールを作成・設定する。CodeCommitの操作権限を忘れないようにしてください。
1. アカウントBにスイッチロールためのロールを作成・設定する。アクセス用のIAMユーザを用意してユーザにロールをアタッチします。

## EC2に必要なソフトウェアのインストール
アカウントBのEC2に下記をインストールします。
* git(バージョン1.7.9以降)
* Python(バージョン3以降)
* pip(バージョン9.03以降)

特に、Pythonとpipをインストールした後は、`pip install git-remote-codecommit`で`git-remote-codecommit`というユーティリティをインストールします。これは、通常はスイッチロールのときに`credential.helper`を利用して認証情報を登録するのですが、git-remote-codecommitを利用するとその必要はなくなり認証に関連する操作が少し楽になります。使い方は後ろで書きます。

## VPCエンドポイントの作成
CodeCommitにクロスアカウントアクセスするためにエンドポイントを作成します。ここで作成するエンドポイントは下記2つになります。
* git-codecommit
* sts

CodeCommitのエンドポイントは2つあるのですが、git操作のエンドポイントになるのは`git-codecommit`の方になります。`codecommit`はAPI操作などを受けるエンドポイントになります。

## EC2でスイッチロールの準備
アカウントBのEC2で`.aws/credentials,.aws/config`を修正します。

### .aws/credentials
```shell
[default]
aws_access_key_id=XXXXXXXXXXXXXXXXXXXX
aws_secret_access_key=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
### .aws/config
```shell
[default]
region=ap-northeast-1
output=json

[profile switchrole]
role_arn = arn:aws:iam::111111111111:role/SwitchRole
region=ap-northeast-1
source_profile=default
output=json
```

# CodeCommitのリポジトリをクローンする
ここまでで準備は完了です。リポジトリのクローン作業へと入ります。

## credential.helperの利用
credential.helperを利用して認証情報を登録します。
```shell
git config --global credential.helper '!aws --profile switchrole codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

`--profile`でスイッチロール先の情報が書かれたProfileを指定してるので、このまま`git clone`を実行すれば問題なくアカウントAのリポジトリにクロスアカウントアクセスできるはずです。が、ここでハマりポイントがあります。

```shell
git clone https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/<test_repo>
```

### (ポイント)git cloneを実行しても応答がない
今までの設定だと、git cloneを実行しても応答がない可能性があります。というのも、必ずというわけではなく**AWS CLI のバージョンが1.xの時に発生します**。これの原因は下記ドキュメントにあるのですが、STSがエンドポイントを利用する時、本来はリージョンのエンドポイントを見に行ってほしいのですが、AWS CLI v1の場合はデフォルでグローバルエンドポイントを見に行きます。そのため、リージョンのエンドポイントを見るように明示的に指定する必要があります。

https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-configure-files.html

下記の★の箇所を`.aws/config`内に追記します。

```shell
[profile switchrole]
role_arn = arn:aws:iam::111111111111:role/SwitchRole
region=ap-northeast-1
source_profile=default
output=json
sts_regional_endpoints=regional　★
```
ですが、これはAWS CLIv1の問題のため、AWS CLI v2を利用することでも解消できます。

## git-remote-codecommitの利用
今まではcredential.helperを利用した方式について説明しましたが、AWSとしてはgit-remote-codecommitの利用を推奨しています。というのも、credential.helperだとCodeCommitの接続に問題が発生する可能性があるからです。

https://docs.aws.amazon.com/ja_jp/codecommit/latest/userguide/setting-up.html
> 一部のオペレーティングシステムと Git バージョンには、独自の認証情報ヘルパーがあり、AWS CLI に含まれる認証情報ヘルパーと競合します。そのため、CodeCommit の接続に問題が発生する可能性があります。


git-remote-codecommitを利用すると、下記のコマンドでリポジトリをcloneすることができます。このgit-remote-codecommitでは、credential.helperのときの問題は発生しません。

```shell
git clone codecommit://switchrole@<test_repo>
```

https://github.com/aws/git-remote-codecommit/blob/master/git_remote_codecommit/__init__.py

git-remote-codecommitのコードがGithubで公開されているので確認してみるとわかるのですが、このユーティリティでは下記コードでリージョンのエンドポイントを指定するようになっています。
```python
hostname = os.environ.get('CODE_COMMIT_ENDPOINT', 'git-codecommit.{}.{}'.format(region, website_domain_mapping(region)))
```

# まとめ
VPCエンドポイントを利用してCodeCommitにアクセスするときの問題としていますが、実際はVPCエンドポイントを利用したときのSTS利用におけるハマりポイントの話でした。

AWS CLIv1を利用している環境でエンドポイント経由でクロスアカウントアクセスしたい場合、リージョンのエンドポイントを明示する必要があります。CLIでstsコマンドを実行する場合は`--endpoint-url`でエンドポイントを指定することもできます。

