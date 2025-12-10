---
title: "GitHub Organizationを Terraform + HCP Terraformで管理する際の実装方法"
emoji: "🔐"
type: "tech"
topics: ["github", "terraform", "hcp", "iac", "devops"]
published: false
---

# はじめに
弊社ではSRE/インフラがTerraformによるGitHub管理を導入しました。GitHubのメンバーをTerraform で管理するのはよくあることかもしれません。この記事では以下をお伝えしたいと思います。

- なぜGitHubで管理するのか
- マルチOrganization環境でのHCP Terraform との連携
- その他Tips

弊社には多くのプロダクトがあります。また、複数のOrgも持っています。ワークフローとしてはSlackから申請してSREがそれを確認して設定するという流れを撮っていました。台帳で管理していたので名前とアカウントが一致しないということもありません。ただ、「変更を追いにくい」という点は課題だと感じていました。背景としてまずM&Aをする会社であり、今後スケール指定国あたりSREが作業するだけでは限界がくることを想定し、利用者から申請ができるようにするという点もあります。これにより、スケールがしやすいという点があります。これが狙いです。このあたりの話をしてきます。


入していくことがベストだと考えました。

# ということで
本記事では、これらの課題をTerraform + HCP Terraformで解決し、**PRベースの権限管理フロー**を実現した事例を紹介します。ちなみに私は株式会社GENDAでSRE/インフラエンジニアをしています。入社6ヶ月目の布田と申します。

ちなみにこの記事は、GENDA Advent Calendar 2025 シリーズ2 Day12 の記事です。他の記事も是非呼んで頂ければと思います！
https://qiita.com/advent-calendar/2025/genda

技術的な実装だけでなく、**運用してみて分かった効果**についても詳しく解説します。


## 目次
1. [なぜTerraform化したのか](#なぜterraform化したのか)
2. [アーキテクチャ概要](#アーキテクチャ概要)
3. [設計のポイント](#設計のポイント)
4. [実装](#実装)
   - [HCP Terraformのセットアップ](#hcp-terraformのセットアップ)
   - [GitHub App認証の設定](#github-app認証の設定)
   - [Workspace構成](#workspace構成)
   - [チーム設計と実装](#チーム設計と実装)
5. [運用フロー](#運用フロー)
6. [実装時のハマりポイント](#実装時のハマりポイント)
7. [運用してみて分かったこと](#運用してみて分かったこと)
8. [組織にもたらした変化](#組織にもたらした変化)
9. [新規Organizationの追加手順](#新規organizationの追加手順)
10. [まとめ](#まとめ)


# なぜ導入したのか - トレーサビリティの向上
私たちの組織では、複数のGitHub Organizationを運用しており、メンバーの追加・削除、チームの作成、リポジトリへのアクセス権限付与を日常的に行っています。しかし、これらを手作業で管理していた頃は、「誰が、いつ、なぜ権限を変更したのか追跡しずらい(Slackのやり取りなどから確認は可能)」、「この人、なぜこのリポジトリにアクセスできるんだっけ？」がたくさんのチケットから掘り出さないといけない。監査対応時に履歴をたどりづらい。という課題がありました。ただ、今はGitHubでPRベースでメンバーを管理することでこれらの課題が解決されています。
実際、運用としては誰がレビューして誰がMergeしたのか。エンジニアである我々にとっては非常に見やすい形で管理されており過去の作業も参照しやすい状況となっています。

# なぜ導入したのか2 - スケーラビリティの確保
SREが依頼を受けて設定するのも、頻度によっては全然対応することは可能です。ただ、今後M&Aを繰り返す中で組織がスケールしていくと管理の手間というのは増加します。弊社の性質上、今のうちにスケールしていくことを意識して仕組みを導


# アーキテクチャについて
## なぜHCP Terraformなのか - 管理や運用の容易さ
https://developer.hashicorp.com/terraform/cloud-docs
今回、HCP Terraformにしたのは「環境が整っていた」という点もありますが、GiHub Actionsなど別のCICDではなく、HCP Terraformを選んだ理由です。まず、HCP Terraformを利用する環境が整っていたということがあります。そして、細かい管理を別にアウトソースできることは非常に理想的だと感じました。今後変更していく可能性がないとは言いませんがまずは試していくというでも作り込みが少ない方を選びました。ただ、一つ今後課題になる点があるとすれば「コスト」です。リソース数によって課金されるため、ここはできる限りペイできるのかを意識する必要があります。スケールさせるという点でコストが増大する場合自動適用の仕組みは考え直す可能性があります。


## ディレクトリの構成
一つのリポジトリで複数のOrg用のディレクトリを作成します。そして、その下にそれぞれのメンバー管理用のコードを準備します。Module化は共通で利用するところはしています。
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

## HCP Terraform とGitHubの連携
ざっくりとしたシステム構成図は以下のようになっています。管理用のリポジトリにPRを設定すると、その更新した箇所に応じて特定のWorkspaceで処理が動きます。これは、以下のディレクトリ構成にあるのですが `github-user-management` というリポジトリの下に、OrgA/OrgB/・・・という分け方としています。このOrgディレクトリごとに適応するWorkspaceを作成しています。

![](https://storage.googleapis.com/zenn-user-upload/e7be15539597-20251210.jpg)



## 設計のポイント - 1. Workspace分離について
実装にあたって、重視した設計ポイントを紹介します。まず、**Organization毎にWorkspaceを分ける**という設計にしました。理由としては以下になります。

- **影響範囲の分離**: 1つのOrgへの変更が他のOrgに影響しない
- **明確性**: どのOrgに変更が入るか一目瞭然

「影響範囲の分離」と「明確姓」は大きな理由です。見やすさというのもありますし論理的に分離されていることの安心感もあります。これは、各Workspaceで`Working Directory`を設定することで、該当ディレクトリ配下のTerraformコードのみが実行されます。これにより、各Orgへの更新は各Org内で完結させることができます。

```
OrgA → Working Directory: OrgA/
OrgB → Working Directory: OrgB/
```
これにより、1つのリポジトリで複数のOrganizationを管理しつつ、安全に運用できます。

##  設計のポイント - 2. VCS連携の認証方法 GitHub App

https://developer.hashicorp.com/terraform/cloud-docs/vcs/github-app

特に変わった点ではないのですが、 **GitHub App**を利用して認証しています。PATやOAuth Appでもできるっぽいのですが、基本的にチームとして管理していくためOrg単位でGitHub Appを作成しています。下の方で作成や設定方法について説明しています。

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

![](https://storage.googleapis.com/zenn-user-upload/f0dc2b90b00a-20251211.png)

**ポイント**: `Working Directory`を各Organization用ディレクトリに設定することで、該当ディレクトリ配下のTerraformコードのみが実行されます。

### 3. VCS連携の設定

GitHub Repositoryと連携します：

1. HCP Terraform → Settings → Version Control
2. Version Control Workflow → GitHub App
3. OrgとRepositoryを選択


以上で設定は完了です。過去にとってキャプチャやメモを参考にしてみましたが自分でもハマったポイントなどあったので参考になれば幸いです。



# 運用してみて分かったこと

実際に運用してみて分かった効果と気づきを紹介します。対応二関しては、時間が明確にへったかというとそうではないです。正直手順的になれているかどうかなのであまり変わらないかもしれません。でも、PRにすることでやっていることが記録されたり、慣れた手順でできることは非常に良かったです。あと、コード化されることで今の状態を確認しやすくなります。生成AIの登場でそれはより顕著担ったと思います。ただ、エンジニアだけではなくデザイナーが使いたいケースもあったため通常の申請ワークフローは残して置く必要があると感じています。このときは我々が更新すれば良いのでそこも問題ありません。




# 運用フロー
参考として運用フローも紹介します。

### 1. メンバー追加の依頼

開発者がPRを作成：

```OrgA/members.tf
 locals {
   all_members = {
     # 既存メンバー...
+    
+    "new-developer" = {
+      role            = "member"
+      email           = "new-developer@genda.jp"
+      name_kanji      = "新人 太郎"
+      name_alphabet   = "Taro Shinjin"
+      employment_type = "正社員"
+    }
   }
 }
```
このように更新します。実際はGitHubのTeamも利用しているため、「どこに所属するか」も更新します。

### 2. レビュー

**レビュアー要件**:
- SREチーム（CODEOWNERS自動指定）
- マネージャーまたはリード（メンバーがPR作成時）

レビュー観点：
- メンバー情報の正確性
- 適切なチームへの所属
- 権限レベルの妥当性

プロダクトチームはレビュアーにプロダクトチームの承認者を追加するようにしています。これで、チーム内として合意が取れているかを確認し、SRE/インフラとしては全体を通して権限が強すぎないかなどを確認します。

### 3. HCP TerraformでのPlan確認

PRマージ前に、HCP Terraform上でPlan結果を確認：


### 4. マージとApply

1. PRをマージ
2. HCP TerraformでPlan/Applyが自動実行

### 5. 完了
これで完了です。


