---
title: "CodeCommit/CodeBuildからECRへPushする際に、Privateな環境になっているか確認するポイント"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "codecommit", "codebuild", "ECR"]
published: true
---

# インターネットを経由せずにパイプラインを作れるの？
まず、そもそもとしてインターネットを経由せずにパイプラインを作ることができるのか、という疑問があります。下記資料のp.58にもありますが、これに関しては**YES**です。
https://www2.slideshare.net/AmazonWebServicesJapan/20201111-aws-black-belt-online-seminar-aws-codestar-aws-codepipeline

> VPCエンドポイントを経由するとインターネットにアクセスすることなくサービスと連携が可能。
> AWS CodeCommit、AWS CodeBuild、AWS CodeDeployはVPCエンドポイントに対応している
> クライアント端末からVPCエンドポイント経由でAWS CodeCommitにコミットする。
> AWS CodePipeline内はAWSサービス間のため閉域網で連携する。

# 今回の記事の目的
今回はこの構成とまったく同じではないですが、同じような構成を利用していきたいと思います。かつ、インターネット**へ**アクセスしないことと同時に、インターネット**から**アクセスされないようになっているか、という観点から**CodeCommit, CoduBuild, ECR**についてまとめていきたいと思います。

# どういう処理・構成になるの？
全体構成について検討します。処理の流れとしては、EC2 InstanceからCodeCommitへPushするとCodeBuildが起動し、ビルドを行います。このビルド処理の中でECRへDockerイメージをpushする想定とします。

## 使用するサービス
AWSで`CodeCommit`, `CodeBuild`, `CodeDeploy`, `CodePipeline`, `Elastic Container Registry(ECR)`を利用して、インターネットに出ないCI/CDパイプラインを構築して行きたいと思います。CodeCommitにPushするための`EC2`, デプロイ先となる`ECS`も利用します。

:::message
ただ、今回の記事ではタイトルにあるように**CodeCommit + CodeBuild + ECR にフォーカスしています**。
:::

## 構成図
構成図はこんな感じです。**CodeBuildがVPC内にある**のが一つポイントとなっています。**今回の説明範囲は赤線の部分です**。
![](https://storage.googleapis.com/zenn-user-upload/9kfs0m19yvhjk1v80phzywuguj2e)

# 本当にPrivateな環境になっている？
`CodeCommit, CodeBuild, ECR`において、確認・考慮すべき項目について挙げてみます。

## CodeCommit
まず、CodeCommitへのアクセスについてです。CodeCommitは下記について考慮する必要があります。

1. AWSリソースからのアクセス
1. AWSリソース外からのアクセス

①については、`VPCエンドポイントを利用すること`でインターネットに出ないクローズドな通信をすることができます。構成図のEC2 Instanceからgit-codecommitが該当します。ですが、CodeCommitでリポジトリを作成すると下記のようなコマンドが表示されます。CodeCommitはリポジトリポリシーやSecurityGroupなどアクセスを制限する機構がないため、②のケースにおいて外部から通信ができてしまいます。

```shell
git clone https://git-codecommit.<Region>.amazonaws.com/v1/repos/<RepogitoryName>
```

ですが、CodeCommitのリポジトリへアクセスするには認証情報が必要となります。そのため、`https`の通信はできますが、実際にリポジトリへのアクセスは認証段階で防ぐことができます。
> CodeCommit リポジトリは Git ベースで Git 認証情報を含む Git の基本機能をサポートしているので、CodeCommit で作業する場合は、IAM ユーザーを使用することをお勧めします。他のアイデンティティタイプで CodeCommit にアクセスすることができますが、その他のアイデンティティタイプは、以下に説明するように制限されることがあります。

https://docs.aws.amazon.com/ja_jp/codecommit/latest/userguide/auth-and-access-control.html

結果、認証の設定さえきちんとしていれば、プライベートな環境で言えると考えます。

## CodeBuild
CodeBuildについては、下記について考慮する必要があります。

1. VPC内のサービスとするか
1. VPC外のサービスとするか

まず、CodeBuildに関して簡単に確認したいと思います。
> AWS CodeBuild は、クラウドで動作する完全マネージド型のビルドサービスです。CodeBuild はソースコードをコンパイルし、ユニットテストを実行して、すぐにデプロイできるアーティファクトを生成します。

https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/welcome.html

CodeBuildで行う操作は`buildspec.yml`ファイルで指定することができます。そして、その結果できたファイル(アーティファクト)をS3に格納したり、ECRにプッシュしたりします。
https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/build-spec-ref.html

基本、AWSサービス同士の連携であり、VPC内のリソースへアクセスしない限りは、VPC外のリソースとして利用しても特に問題ないように思えます。ただ、実際はビルド時にどのような操作をするかによって変わります。

一例を示します。下記は、Docker イメージをビルド出力として生成し、そのDocker イメージを ECRのリポジトリにプッシュしています。

`buildspec.yml`
```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...          
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG      
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
```

`Dockerfile`
```docker
FROM golang:1.12-alpine AS build
#Install git
RUN apk add --no-cache git
#Get the hello world package from a GitHub repository
RUN go get github.com/golang/example/hello
WORKDIR /go/src/github.com/golang/example/hello
# Build the project and send the output to /bin/HelloWorld 
RUN go build -o /bin/HelloWorld

FROM golang:1.12-alpine
#Copy the build's output binary from the previous build container
COPY --from=build /bin/HelloWorld /bin/HelloWorld
ENTRYPOINT ["/bin/HelloWorld"]
```

このDockerfileですが、golangのイメージファイルを指定してます。buildspec.ymlでは`docker build`を実行していますが、このコマンド実行時にローカルにイメージファイルがない場合はDockerHubへ取得しに行きます。

ここが少しやっかいなところで、VPC外のリソースとしていた場合、気づかない内にインターネットへアクセスしてしまいます。ですが、VPC内のリソースにしておけば(インターネットの口がなければ)想定外のインターネットへの通信を防ぐことができます。そのため、VPC内のリソースにしてVPC外へのアクセスはエンドポイントで制御することをオススメします。

## Elastic Container Registry(ECR)
ECRでは、下記が考慮ポイントとなります。

* レジストリ・リポジトリがPublicなっていないか

2020のre:Inventで発表がありましたが、ECRをパブリック・レジストリとし、Public Galleryへコンテナイメージを公開できるようになりました。
https://aws.amazon.com/jp/blogs/aws/amazon-ecr-public-a-new-public-container-registry/

かつ、リポジトリでもパブリックの設定をすることができます。パブリックリポジトリとプライベートリポジトリの違いは公式FAQがわかりやすいかと思います。

> プライベートリポジトリはコンテンツ検索機能を提供せず、イメージのプルを許可する前に AWS アカウント認証情報を使用した Amazon IAM ベースの認証を必要とします。パブリックリポジトリには説明的なコンテンツがあり、AWS アカウントを必要とせず、IAM 認証情報を使用しなくても、誰でもどこからでもイメージをプルできます。パブリックリポジトリのイメージは、ECR パブリックギャラリーでも入手できます。

https://aws.amazon.com/jp/ecr/faqs/

このように、公開範囲をコントロールする必要があるのですが、プライベートにしておけば外部からの通信は防げるので、今回の趣旨としてはプライベートに設定しておくで良いと思います。

# まとめ
一旦、3つのサービスについて、**プライベートな環境**という観点で確認してみました。結論としては、CodeCommitやECRはIAMベースの認証があるので、外部からのアクセスコントロールが可能です。ただ、ECRはレジストリ・リポジトリをパブリックにしないように気をつけましょう。

CodeBuildはビルド作業時に気づかずに外部へ通信をしてしまう可能性があるため、VPC内のリソースとしたほうが安全だと思います。

他のサービス含めた構成についても説明しようかと思いましたが、長くなりそうだったので今回は少しスコープを切って記事にまとめました。

# Appendix:構築時の注意点
CodeBuildからECRへアクセスする時は下記のエンドポイントを作成する必要があります。

* ecr.dkr
* ecr.api
* s3
* logs

https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/vpc-endpoints.html
https://dev.classmethod.jp/articles/privatesubnet_ecs/

s3やlogsがなくてコケる場合があるので気をつけましょう。

