---
title: "CloudFormationでNetworkFirewallを作成する時に見直す記事"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "CloudFormation", "NetworkFirewall"]
published: true
---

# はじめに
AWS NetworkFirewallをCloudFormationで作成する際に、以下の要素を取り込んだテンプレートを作成しました。忘れた時に見返すように残しておきます。
- VPCルートテーブルの宛先にNetworkFirewallを設定する
- "厳密な順序"を指定したNetworkFirewallポリシーを作成する
- AWS Managed Threat Signatures を有効化する/アラートモードを有効化する
- PolicyVariables に複数のCIDRを設定する

# やったこと

## VPCルートテーブルの宛先にNetworkFirewallを設定する
ルートテーブルでNetworkFirewallを指定する、ということは「NetworkFirewallのVPCエンドポイントを指定する」ことです。そのため、`VpcEndpointId` で指定するのですが `!REF` を利用してしまうと、[ARNが返ってくる](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-networkfirewall-firewall.html#aws-resource-networkfirewall-firewall-return-values)ためVPCエンドポイントを指定することができません。そこで、`!GetAtt` で取ってきたデータを分割して取り出しています。先程のリンクからGetAttで取得できるデータが確認できます。

```yaml
# ファイアウォールエンドポイントへのルート設定
   TGWRouteToFirewall1:
     Type: AWS::EC2::Route
     Properties:
       RouteTableId: !Ref xxx
       DestinationCidrBlock: 0.0.0.0/0
       VpcEndpointId: !Select [1, !Split [':', !Select [0, !Split [',', !Select [1, !GetAtt NetworkFirewall.EndpointIds]]]]]
```

## ファイアウォールで”厳密な順序(STRICT_ORDER)”を有効化する方法
CloudFormationでNetworkFirewallを作成すると、”厳密な順序”が推奨であるにも関わらずデフォルトでは”アクションの順序”で作成されます。そのため、明確に”厳密な順序”を有効化するためには、`StatefulEngineOptions` に `RuleOrder: STRICT_ORDER` を設定する必要があります。

```yaml
  FirewallPolicy:
    Type: AWS::NetworkFirewall::FirewallPolicy
    Properties:
      FirewallPolicy:
        StatelessDefaultActions: 
          - aws:forward_to_sfe
        StatelessFragmentDefaultActions:
          - aws:forward_to_sfe
        StatefulRuleGroupReferences: 
          - ResourceArn: !Sub arn:aws:network-firewall:${AWS::Region}:aws-managed:stateful-rulegroup/ThreatSignaturesIOCStrictOrder
        StatefulEngineOptions:
          RuleOrder: STRICT_ORDER ### ここで厳密な順序を有効化する
        StatefulDefaultActions:
          - aws:drop_strict
          - aws:alert_strict
      FirewallPolicyName: network-firewall-policy
```

## AWS Managed Threat Signatures を有効化する/アラートモードを有効化する
ここでは、マネージドルールの指定方法とアラートモードの有効化の例を記載します。

```yaml
StatefulRuleGroupReferences: 
  - ResourceArn: arn:aws:network-firewall:ap-northeast-1:aws-managed:stateful-rulegroup/ThreatSignaturesIOCStrictOrder
    Priority: 1
    Override: 
      Action: DROP_TO_ALERT
  - ResourceArn: arn:aws:network-firewall:ap-northeast-1:aws-managed:stateful-rulegroup/ThreatSignaturesPhishingStrictOrder
    Priority: 2
    Override:
      Action: DROP_TO_ALERT
```

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-networkfirewall-firewallpolicy-statefulrulegroupreference.html


## PolicyVariables に複数のCIDRを設定する
PolicyVariables では、複数のCIDRを設定することができます。しかし、ドキュメントには、`PolicyVariables` に複数のCIDRを設定する方法が記載されていません。そこで、以下のように記載することで、複数のCIDRを設定することができます。Type: AWS::NetworkFirewall::FirewallPolicy に `PolicyVariables` を追加するのですが、ドキュメントには `Definition:` についての記述がなく少しハマりました。

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-networkfirewall-firewallpolicy-policyvariables.html


```yaml
        PolicyVariables:
          RuleVariables:
            HOME_NET:
              Definition:
                - 10.0.0.0/8
                - 172.16.0.0/12
                - 192.168.0.0/16
```

# おわりに
NetworkFirewallのテンプレートを作成する際に、調べたことを整理しました。テンプレート作成時の参考になれば幸いです。