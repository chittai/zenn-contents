---
title: "StackSetを利用して、別アカウントのAWS Config Ruleを有効化する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","CloudFormation"]
published: true
---

# 別アカウントのAWS Config Ruleを有効化したい
CloudFormationを利用して、別アカウントの設定を変更するのであればStackSetsを利用します。

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html

StackSetsのサンプルをみると、Config Ruleを設定するサンプルテンプレートがあります。これを参考に、他のルールを有効化する時にどうやってテンプレートを作成するのかをまとめて置こうと思います。

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/stacksets-sampletemplates.html

# 環境の前提
今回の環境は、`Organization`で同じ`OU`に存在しているアカウントを対象としています。管理アカウントAから別アカウントBのConfig Ruleを有効化します。有効化するルールはマネージドのものを対象としています。

# StackSetsのテンプレート作成方法
StackSetsのテンプレートはYAMLファイルで宣言的に行うのですが、順に作成方法ついて見ていきます。

## Configの設定値
まず、テンプレートに記述する内容ですが、基本はConfig RuleをGUIから有効化しようとした時に出てくる画面の設定値になります。今回は例として`ec2-stopped-instance`を選択してみます。ここでポイントになるのは、`トリガー`と`ルールのパラメータ`です。これは、トリガーが`設定変更`の時と`定期的`の2パターンのケースがあるためです。そしてルールのパラメータもある場合と無い場合があります。

![](https://storage.googleapis.com/zenn-user-upload/gzauzu8eao823jido5psllijpj2p)


つまり、パターンとしては下記4つに分けることができます。今回は2と3について見ていきたいと思います。
1. 設定変更 + パラメータ有り
1. 設定変更 + パラメータ無し
1. 定期的 + パラメータ有り
1. 定期的 + パラメータ無し


## YAMLの全体
先程説明した番号の3に該当する`ec2-stopped-instance`を有効化するYAMLを見ていきましょう。ここでするのは、ルールを有効化することに対する説明なので全体の説明はせず対象に絞って説明します。

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Enables an AWS Config rules

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label: 
          default: Common Configuration
        Parameters:
          - Frequency
      - Label:
          default: Rule ec2-stopped-instance Configuration
        Parameters:
          - AllowedDays

    ParameterLabels:
      Frequency:
        default: Frequency
      AllowedDays:
        default: AllowedDays

Parameters:
  AllowedDays:
    Type: String
    Description: "[Option] The number of days an ec2 instance can be stopped before it is NON_COMPLIANT."
    Default: 30

  Frequency:
    Type: String
    Default: 24hours
    Description: Maximum rule execution frequency
    AllowedValues:
      - 1hour
      - 3hours
      - 6hours
      - 12hours
      - 24hours

Conditions:
  HasAllowedDays: !Not
    - !Equals
      - !Ref AllowedDays
      - ""

Mappings:
  Settings:
    FrequencyMap:
      1hour   : One_Hour
      3hours  : Three_Hours
      6hours  : Six_Hours
      12hours : Twelve_Hours
      24hours : TwentyFour_Hours

Resources:
  CheckForStoppedInstances:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: CheckForStoppedInstances
      Description: Checks whether there are instances stopped for more than the allowed number of days.
      MaximumExecutionFrequency: !FindInMap
          - Settings
          - FrequencyMap
          - !Ref Frequency
      Source:
        Owner: AWS
        SourceIdentifier: EC2_STOPPED_INSTANCE
      InputParameters:
        AllowedDays: !If
          - HasAllowedDays
          - !Ref AllowedDays
          - !Ref AWS::NoValue
```

ここでルールを定義しているのは`Resource句`になります。これは`定期的`なのでどれぐらいの間隔でルールを確認するか決める必要があります。それを設定しているのが`MaximumExecutionFrequency`です。上の方を見るとわかりますが5つの選択肢から選びます。`SourceのSourceIdentifier`で実際にどのマネージドルールを対象とするかを記述します。そして、`InputParameters`でパラメータを指定します。これも上の`Parameters句`に記述があります。

つまり、まとめると`定期的＋パラメータ有り`の場合、定期的に必要なパラメータは`MaximumExecutionFrequency`でパラメータの設定に必要なのは`InputParameters`であることがわかります。それ以外は`設定変更＋パラメータ無し`でも同じです。

```yaml
Resources:
  CheckForStoppedInstances:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: CheckForStoppedInstances
      Description: Checks whether there are instances stopped for more than the allowed number of days.
      MaximumExecutionFrequency: !FindInMap
          - Settings
          - FrequencyMap
          - !Ref Frequency
      Source:
        Owner: AWS
        SourceIdentifier: EC2_STOPPED_INSTANCE
      InputParameters:
        AllowedDays: !If
          - HasAllowedDays
          - !Ref AllowedDays
          - !Ref AWS::NoValue
```

ただ、毎回ルールのGUIを確認するのもめんどくさいので、必要なパラメータなどは下記URLで確認することも可能です。
https://docs.aws.amazon.com/ja_jp/config/latest/developerguide/managed-rules-by-aws-config.html

次に、`設定変更＋パラメータ無し`について確認します。これは`eip-attached`を例として見ていきます。今回はGUIではなく上記ドキュメントを確認していきます。
https://docs.aws.amazon.com/ja_jp/config/latest/developerguide/eip-attached.html

* 識別子：EIP_ATTACHED
* トリガータイプ：設定変更
* パラメータ：なし

このあたりを参照すればテンプレートは作成できます。いくつかわかりやすいポイントとして`MaximumExecutionFrequency`と`InputParameters`がなくなりました。このルールでは不要のため記述しません。ただ、新しくでてきたのが`Scope`です。

```yaml
Resources:
  CheckForEIPAttached:
    Type: AWS::Config::ConfigRule
    Properties:
      Description: Checks whether all EIP addresses allocated to a VPC are attached to EC2 instances or in-use ENIs.
      Source:
        Owner: AWS
        SourceIdentifier: EIP_ATTACHED
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::EIP
```

実際にルールをGUIで確認すると下記のようになっています。

![](https://storage.googleapis.com/zenn-user-upload/s83412n9bti2zwduc95juygvsm3f)

`設定変更`の場合、対象となるリソースを指定する必要があります。下記から確認していただければと思いますが、GUI画面で見たものを転記したほうが確実だと思います。今回は`AWS::EC2::EIP`です。
https://docs.aws.amazon.com/ja_jp/config/latest/developerguide/resource-config-reference.html#amazonsimplestorageservice

以上から、`設定変更＋パラメータ無し`をまとめると。設定変更では`Scope`で対象リソースを指定する必要があり`InputParameters`の記述は不要である、となります。

あとは、これを`StackSets`で展開すれば完了です。別アカウントでルールが有効化されているはずです。

# まとめ
これに関連する修復オプションのテンプレート記述方法についてまとめたいと思います。

