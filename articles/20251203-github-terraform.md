---
title: "GitHub Organizationのメンバー管理をHCP Terraformで管理する際の実装方法"
emoji: "👥"
type: "tech"
topics: ["github", "terraform", "iac", "devops"]
published: false
---

# はじめに
株式会社GENDAでSRE/インフラエンジニアをしている布田です。弊社ではSRE/インフラチームがTerraformによるGitHubメンバー管理を導入しました。この記事では以下を紹介します。

- なぜGitHubをTerraform管理にしたのか
- マルチOrganization環境でのHCP Terraformとの連携方法
- 実装時のTips

弊社には複数のプロダクトがあり、それぞれ別のGitHub Organizationを持っています。これまではSlackで申請を受けてSREが設定するという流れだったのですが、変更履歴を追いにくいという課題がありました。
また、M&Aを積極的に行う会社なので、今後組織がスケールしていくとSREだけで対応するのは限界がきます。そこで、利用者自身がPRで申請できる仕組みを作る必要があるのではと考えました。

本記事では、 HCP Terraformで **PRベースの権限管理フロー** を実現した事例を紹介します。

この記事は、GENDA Advent Calendar 2025 シリーズ1 Day12 の記事です。他の記事もぜひ読んでください！
https://qiita.com/advent-calendar/2025/genda


# なぜ導入したのか 

## トレーサビリティの向上

複数のGitHub Organizationを運用していると、メンバーの追加・削除、チームの作成、リポジトリへのアクセス権限付与が日常的に発生します。手作業で管理していた頃は、Slackのやり取りから確認はできるものの、「誰が、いつ、なぜ権限を変更したのか」を追うのが大変でした。

今はGitHubでPRベースで管理しているので、これらの課題が解決されています。誰がレビューして誰がマージしたのか、エンジニアにとって見慣れた形で管理されているので、過去の作業も参照しやすくなりました。

## スケーラビリティの確保

SREが依頼を受けて設定するのも、頻度によっては問題なく対応できます。ただ、今後M&Aを繰り返して組織がスケールしていくと、管理の手間は確実に増えます。弊社の性質上、今のうちにスケールを見据えた仕組みを作っておくのがベストだと判断しました。


# アーキテクチャについて

## なぜHCP Terraformなのか

https://developer.hashicorp.com/terraform/cloud-docs

まず、弊社ではすでにHCP Terraformを利用する環境が整っていました。そして、tfstate の管理やCICDパイプラインの構築・管理をサービス側に渡すことができるのはメンバーに対して組織が拡張していくうえで理想的だと感じました。また、今後変更する可能性はゼロではないですが、まずは試すという意味でも作り込みが少ない方を選んでいます。

ただ、一つ今後課題になりそうなのが「コスト」です。リソース数によって課金されるため、スケールさせる際にコストが増大する場合は、自動適用の仕組みを考え直す可能性があります。


## ディレクトリの構成
一つのリポジトリで複数のOrg用のディレクトリを作成します。そして、その下にそれぞれのメンバー管理用のコードを準備します。moduleは各Orgで共通利用するものをまとめています。

```
github-user-management/
├── shared/
│   └── modules/
│       └── team/              # チーム管理モジュール
├── OrgA/                      # Organization毎にディレクトリ分離
│   ├── members.tf             # メンバー一覧
│   ├── developer_teams.tf     # 正社員チーム
│   ├── product_teams.tf       # プロダクト別で作成するチーム
│   └── versions.tf
│
└── OrgB/
    ├── members.tf
    └── ...
```

## HCP TerraformとGitHubの連携

システム構成図は以下のようになっています。管理用のリポジトリにPRを出すと、更新した箇所に応じて特定のWorkspaceで処理が動きます。

`github-user-management`というリポジトリの下に、OrgA/OrgB/...というディレクトリを作り、このOrgディレクトリごとに適用するWorkspaceを作成しています。

![](https://storage.googleapis.com/zenn-user-upload/e7be15539597-20251210.jpg)



## 設計のポイント - Workspace分離について

**Organization毎にWorkspaceを分ける**という設計は一つのポイントです。まとめるか分離するかを考えたのですが、以下の通り分離する方針としました。

- **影響範囲の分離**: 1つのOrgへの変更が他のOrgに影響しない
- **明確性**: どのOrgに変更が入るか一目瞭然

見やすさもありますし、論理的に分離されていることの安心感もあります。各Workspaceで`Working Directory`を設定することで、該当ディレクトリ配下のTerraformコードのみが実行されます。これで各Orgへの更新は各Org内で完結します。

```
OrgAのWorkSpace → Working Directory: OrgA/
OrgBのWorkSpace → Working Directory: OrgB/
```

1つのリポジトリで複数のOrganizationを管理しつつ、安全に運用できます。

## 設計のポイント - VCS連携の認証方法

https://developer.hashicorp.com/terraform/cloud-docs/vcs/github-app

特に変わった点ではないですが、**GitHub App**を使って認証しています。PATやOAuth Appでもできますが、チームとして管理していくためOrg単位でGitHub Appを作成しています。作成や設定方法は後述します。

# 実装について
ここからは具体的な実装方法を解説します。

## GitHub App認証の設定
まず、Workspace でVCS連携を設定するために、GitHub Appを作成します。

### GitHub Appの作成手順

#### 1. GitHub Appの作成
Organization Settings → Developer settings → GitHub Apps → New GitHub App

**必要な権限**:
- Repository permissions:
  - Administration: Read & Write
  - Metadata: Read-only
- Organization permissions:
  - Members: Read & Write

#### 2. Private Keyの生成

GitHub App設定画面で「Generate a private key」をクリックし、`.pem`ファイルをダウンロード。

#### 3. HCP Terraform環境変数の設定

各Workspaceで以下の環境変数を設定：

| 変数名 | 値 |
|--------|-----|
| `GITHUB_APP_ID` | GitHub AppのID |
| `GITHUB_APP_INSTALLATION_ID` | InstallationのID |
| `GITHUB_APP_PEM_FILE` | Private Keyの内容 |

:::message
地味なTipsですが、GitHub AppのPrivate Keyは複数行のPEMファイルですが、HCP Terraformの環境変数には1行で設定する必要があります。なので、ファイルを開いて各行末に`\n`を追加して改行を削除して一行にして登録します。
:::

## HCP Terraformのセットアップ

### 1. Workspaceの作成

Organization毎にWorkspaceを作成します。

#### Workspace設定例

| 項目 | 設定値 |
|------|--------|
| Workspace名 | `OrgA` |
| Working Directory | `OrgA` |
| VCS Branch | `main` |
| Auto Apply | `Auto-apply API, UI & VCS runs`|

具体的には、以下のキャプチャのような設定にしています。**ポイント**は`Working Directory`を各Organization用ディレクトリに設定することで、該当ディレクトリ配下のTerraformコードのみが実行されます。


![](https://storage.googleapis.com/zenn-user-upload/f0dc2b90b00a-20251211.png)


### 2. VCS連携の設定

次にGitHub Repositoryと連携します。WorkSpaceの設定から、どのリポジトリと連携するかを設定します。GitHub Appによる認証をするため先にGitHubの設定を終わらせてから設定します。

1. HCP Terraform → Settings → Version Control
2. Version Control Workflow → GitHub App
3. OrgとRepositoryを選択


以上で設定は完了です。過去にとってキャプチャやメモを参考にしてみましたが自分でもハマったポイントなどあったので参考になれば幸いです。



# 運用してみて分かったこと

実際に運用してみて分かった効果と気づきを紹介します。対応時間が明確に減ったかというと、そうでもないです。正直、手順に慣れているかどうかなのであまり変わらないかもしれません。

ただ、PRにすることでやったことが記録されたり、開発と同じような手順でできるのは体験として良かったです。あと、コード化されることで今の状態を確認しやすくなります。生成AIの登場でそれはより顕著になったなと思いました。

また、一つ気づいたのはエンジニアだけではなくデザイナーが使いたい可能性があるので、通常の申請ワークフローは残しておく必要があるということです。このときは我々が更新すれば良いので問題ありません。




# 運用フロー

最後に、参考として運用フローも紹介します。

## 1. メンバー追加の依頼

開発者がPRを作成します。これは、メンバー用のファイルを更新します。これでOrgへのメンバー追加は完了です。権限の管理自体はTeamを利用しているため「どこに所属するか」も更新します。（が、今回のテーマではないので書いていません）


```diff:OrgA/members.tf
 locals {
   all_members = {
     # 既存メンバー...
+    
+    "new-developer" = {
+      role            = "member"
+      email           = "new-developer@xxx.jp"
+      name_kanji      = "新人 太郎"
+      name_alphabet   = "Taro Shinjin"
+      employment_type = "正社員"
+    }
   }
 }
```


## 2. レビュー
ここでは、レビューのルールを書いています。PRが作成されたときにかならずSREメンバーがビュアーとして設定されるようにしています。また、プロダクトチームはレビュアーにプロダクトチームの承認者を追加するようにしています。これでチーム内として合意が取れているかを確認し、SRE/インフラとしては全体を通して権限が強すぎないかなどを確認します。

**レビュアー要件**:
- SREチーム（CODEOWNERS自動指定）
- マネージャーまたはリード（メンバーがPR作成時）

レビュー観点：
- メンバー情報の正確性
- 適切なチームへの所属
- 権限レベルの妥当性



## 3. HCP TerraformでのPlan確認

PRマージ前に、HCP Terraform上でPlan結果を確認します。

## 4. マージとApply
マージしたら自動でApplyされるので問題なければマージします。

1. PRをマージ
2. HCP TerraformでPlan/Applyが自動実行

## 5. 完了

これで完了です。


以上で、TerraformによるGitHubのメンバー管理の事例紹介を終わります。これをやっている人は多いですが、HCP TerraformやマルチOrgといった環境で利用されている方(利用しようとしている方)の参考になれば幸いです！

