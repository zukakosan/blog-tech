---
title: "Change Analysis を使って Azure Firewall のルールの変更を検知して通知する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure","network","microsoft","security"]
published: true
publication_name: "microsoft"
published_at: 2024-12-20 07:00
---
この記事は、[Microsoft Azure Tech Advent Calendar 2024](https://qiita.com/advent-calendar/2024/microsoft-azure-tech) 20 日目の記事です。

# はじめに
Tech Community[^1] にも投稿されている通り、Azure Firewall に関していくつか管理上のユーザーエクスペリエンスを向上するようなアップデートがありました。

Azure Firewall はその環境のネットワーク制御の中核をなすリソースであることが多く、設定ミスや悪意のある変更が及ぼす影響が大きいです。
また、頻繁にルールが変更されることはなく、管理者としてはルールの変更をいち早く検知することでそのようなリスクを逓減できます。

Azure Resource Graph(ARG) の Change Analysis は Azure Resource Manager 上のプロパティの変更を確認できる機能です。そこに Azure Firewall の RuleCollectionGroups のサポートが追加され、ARG の特定テーブルに対するクエリから差分をとってこれるようになりました。これを、Log Analytics 連携[^2] と組み合わせることで、その変更をトリガーにした通知まで行うことができます。

[^1]:https://techcommunity.microsoft.com/blog/azurenetworksecurityblog/enhancements-to-the-azure-firewall-user-experience/4297129
[^2]:https://learn.microsoft.com/ja-jp/azure/governance/resource-graph/alerts-query-quickstart?tabs=azure-resource-graph

# 構成手順

## ARG 上での変更検知
Azure Firewall のリソースを準備します。デプロイが完了したら Firewall Policy リソース上で Rule Collection Group を作成します。Azure Resource Graph 上で次のクエリを実行すると、`properties.provisioningState` が `Succeeded` に変化している記録が確認できます。

```
networkresourcechanges
| where properties contains "microsoft.network/firewallpolicies/rulecollectiongroups"
```
![](/images/20241220-azfwrule-notify/01.png)

Rule Collection Group に Rule Collection を追加します。
![](/images/20241220-azfwrule-notify/02.png)

追加完了後、ARG 上で改めて次のようなクエリを実行します。

```
networkresourcechanges
| where properties contains "microsoft.network/firewallpolicies/rulecollectiongroups"
| where properties contains "properties.ruleCollections"
```
次のような properties カラムを持つレコードが表示されます。Rule Collection に追加したルールは新規で作成しているため、`previousValue` は `null` になっています。

```
{
    "targetResourceType": "microsoft.network/firewallpolicies/rulecollectiongroups",
    "targetResourceId": "/subscriptions/xxxxxx/resourceGroups/20241216-azfw-law/providers/Microsoft.Network/firewallPolicies/azfw1216pol/ruleCollectionGroups/aaacg001",
    "changeAttributes": {
        "previousResourceSnapshotId": "08584669399142129914_81485f8d-2b1c-b17b-67df-e4fed4a36f09_580266158_1734666971",
        "newResourceSnapshotId": "08584669374144029886_cde5c2c8-0268-977d-c210-85001e6d0449_3150122394_1734669471",
        "correlationId": "f35008d6-cb45-4d40-b0e7-aec54d9e3d96",
        "changedByType": "Unspecified",
        "changesCount": 13,
        "clientType": "Unspecified",
        "timestamp": "2024-12-20T04:37:51.074Z",
        "changedBy": "Unspecified",
        "operation": "Unspecified"
    },
    "changeType": "Update",
    "changes": {
        "properties.provisioningState": {
            "previousValue": "Succeeded",
            "newValue": "Updating"
        },
        "properties.size": {
            "previousValue": "0.001170158 MB",
            "newValue": "0.001578331 MB"
        },
        "properties.ruleCollections[0].action.type": {
            "previousValue": null,
            "newValue": "Allow"
        },
        "properties.ruleCollections[0].name": {
            "previousValue": null,
            "newValue": "test-network-rule-collection"
        },
        "properties.ruleCollections[0].priority": {
            "previousValue": null,
            "newValue": "1000"
        },
        "properties.ruleCollections[0].ruleCollectionType": {
            "previousValue": null,
            "newValue": "FirewallPolicyFilterRuleCollection"
        },
        "properties.ruleCollections[0].rules[0].destinationAddresses[0]": {
            "previousValue": null,
            "newValue": "10.0.0.0/8"
        },
        "properties.ruleCollections[0].rules[0].destinationPorts[0]": {
            "previousValue": null,
            "newValue": "*"
        },
        "properties.ruleCollections[0].rules[0].ipProtocols[0]": {
            "previousValue": null,
            "newValue": "Any"
        },
        "properties.ruleCollections[0].rules[0].ipv6Rule": {
            "previousValue": null,
            "newValue": "False"
        },
        "properties.ruleCollections[0].rules[0].name": {
            "previousValue": null,
            "newValue": "allow-private-traffic"
        },
        "properties.ruleCollections[0].rules[0].ruleType": {
            "previousValue": null,
            "newValue": "NetworkRule"
        },
        "properties.ruleCollections[0].rules[0].sourceAddresses[0]": {
            "previousValue": null,
            "newValue": "10.0.0.0/8"
        }
    }
}
```

![](/images/20241220-azfwrule-notify/03.png)

既存の規則を変更した場合（プロトコル/ポートを Any/* から TCP/443 に絞る）は次のように差分だけが載るイメージになります。

```
{
    "targetResourceType": "microsoft.network/firewallpolicies/rulecollectiongroups",
    "changeAttributes": {
        "previousResourceSnapshotId": "08584669372302829552_b3e773ef-1b43-fa01-c96c-f18aa6c09d97_3596448472_1734669655",
        "newResourceSnapshotId": "08584669365577337842_7f137196-cb2b-1dc0-0ff8-ece512881480_1205258449_1734670327",
        "correlationId": "5c645848-2485-49eb-9084-35d116ef6fe8",
        "changedByType": "Unspecified",
        "changesCount": 4,
        "clientType": "Unspecified",
        "timestamp": "2024-12-20T04:52:07.743Z",
        "changedBy": "Unspecified",
        "operation": "Unspecified"
    },
    "targetResourceId": "/subscriptions/xxxxx/resourceGroups/20241216-azfw-law/providers/Microsoft.Network/firewallPolicies/azfw1216pol/ruleCollectionGroups/aaacg001",
    "changeType": "Update",
    "changes": {
        "properties.provisioningState": {
            "previousValue": "Succeeded",
            "newValue": "Updating"
        },
        "properties.size": {
            "previousValue": "0.001578331 MB",
            "newValue": "0.001580238 MB"
        },
        "properties.ruleCollections[\"test-network-rule-collection\"].rules[\"allow-private-traffic\"].ipProtocols[0]": {
            "previousValue": "Any",
            "newValue": "TCP"
        },
        "properties.ruleCollections[\"test-network-rule-collection\"].rules[\"allow-private-traffic\"].destinationPorts[0]": {
            "previousValue": "*",
            "newValue": "443"
        }
    }
}
```

## ルール変更を通知する
先述の通り、Azure Resouce Graph のテーブルに含まれるレコードをもとにしたアラートを構成できます[^3]。Log Analytics Workspace 上で、`arg("").<table名>` とするだけで参照できるため非常に簡単です。Log Analytics Workspace 上で次のクエリを実行します。
```
arg("").networkresourcechanges
| where properties contains "microsoft.network/firewallpolicies/rulecollectiongroups"
| where properties contains "properties.ruleCollections"
```
すると、ルールの変更に関するレコードが取得できます。
![](/images/20241220-azfwrule-notify/04.png)


そのまま、[+New alert rule] からアラートを作成します。以下では、5 分の評価ウィンドウで見たときに、検出されるレコード数が 0 より大きい場合( 1 件でも変更ログがある場合)にアラートを上げるように設定しています。
![](/images/20241220-azfwrule-notify/05.png)

また、連動する Action Group としては、Contributor ロールを持つメンバーに通知するような設定をしてみます。
![](/images/20241220-azfwrule-notify/06.png)

システム割り当てマネージド ID を付与します。記載の通り、権限の付与が必要になります。
![](/images/20241220-azfwrule-notify/07.png)

作成したアラート ルールの [identity] から Reader ロールを付与します。これにより、アラート ルールというリソースが、ARG の情報を取得できるようになります。
![](/images/20241220-azfwrule-notify/08.png)

[^3]:https://zenn.dev/microsoft/articles/00cf34cb7e53cd

# おわりに
今回使用したクエリは非常にシンプルですが、例えば特定の変更だけ（削除や Any の開放など）を検出するようにするのもいいかもしれません。とはいえ、重要なリソースに関する変更はある程度リアルタイムに検知できた方が運用健全性上よいはずですので、本記事の方法を一例として認識しておくのはよいと思います。Azure Resource Graph で色々な運用改善に使用できる情報が拡充されていくとともに、ユースケースをきちんと考えていくと負荷軽減につながるでしょう。 