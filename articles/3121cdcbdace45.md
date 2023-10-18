---
title: "Bicepで簡易的なHub-Spoke環境をサクッと作る"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bicep","network","azure"]
published: false
---

# これはなに...?
- Hub-Spoke環境をサクッと立てるためのBicepを作ったのでその共有
- 構成と書き方のポイントを紹介
- GitHubにあるソースコードの補足説明的なもの

# 該当リポジトリ
https://github.com/zukakosan/bicep-hub-spoke-quickstarter/tree/main


# デプロイされる環境
以下のような環境をデプロイします。
![](/images/20231018-bicep-hubspoke-minimum/hubspoke-architecture.png)

# アーキテクチャ上のポイント
## Hub VNET
HubにはAzure Firewallをデプロイします。Azure Firewallにはネットワークルールとして東西方向のトラフィック許可のための簡易的なルールが追加されています(from:10.0.0.0/8, to:10.0.0.0/8, Allow)。もっと限定したい場合には適宜経路を絞る必要があります。

また、HubのAzure Firewall上でDNATルールにより、Port50000でJumpbox仮想マシンのPort22にDNATできるようになっています。Jumpbox仮想マシンとDNATルールは既定で作成されます。

一方でAzure Bastionをデプロイしたい場合には、パラメータとして`bastionEnabled`を`Enabled`として渡します。必要に応じてデプロイすることが可能です。

## Spoke VNET
Spokeには、VM1台をデプロイし、接続されたサブネットにはNSGとルートテーブルを設定します。ルートテーブルでは、`0.0.0.0/0 > Azure Firewall`とするUDRを定義しています。これにより、Spoke1-AzureFirewall-Spoke2の通信が可能となっています。

Spokeの個数はパラメータで渡すことで、横方向にスケーリングできます。Spokeの構成はすべて同じになります。

# Bicep上のポイント
ファイルの構成としては以下のような形になっています。
```
main.bicep
   └ modules/
        └ azfw.bicep
        └ bastion.bicep
        └ hubVnet.bicep
        └ spokeVnet.bicep
        └ vm.bicep
```
## モジュールの分け方
この環境ではSpokeが横方向にスケーリングする可能性があり、Hubと性質が全く異なるため、VNET用のモジュールではなくHub用、Spoke用で分割しています。

VMの作成はHub/Spokeそれぞれで流用できるため共通化しました。NICとVMを作成するだけなので、分離させなくてもさほど影響はないと思っていますが、少しでも可視性を上げようというモチベーションです。

## Azure Bastionのデプロイ
またAzure Bastionについては、ユーザの入力に応じてデプロイ要否を決めるため、Hub用のモジュールからさらに独立させ、条件分岐によってデプロイを制御します。

```bicep:hubVnet.bicep
// create azure bastion after hub vnet creattion if deployAzureBastion is true
module createAzureBastion './bastion.bicep' = if(deployAzureBastion) {
  name: 'createAzureBastion'
  params: {
    location: location
  }
  dependsOn: [
    hubVnet
  ]
}
```

## Azure Firewallのルールの更新
Azure FirewallのDNATルールとネットワークルールを作成していますが、同時に更新するとエラーになるので依存関係を明示的に持たせています。

```bicep:azfw.bicep
// create dnat rule collection group on firewall policy
// this rule collection group will be used to translate the destination address of the incoming packet
// rule description: ssh traffic from internet to the firewall public ip address is translated to the private ip address of the vm
resource dnatRuleCollectionGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-04-01' = {
  parent: firewallPolicy
  name: 'DefaultDnatRuleCollectionGroup'
  properties: {
    priority: 100
    ruleCollections: [
      {
        ruleCollectionType: 'FirewallPolicyNatRuleCollection'
        action: {
          type: 'DNAT'
        }
        name: 'dnat-rule'
        priority: 1000
        rules:[
          {
            ruleType: 'NatRule'
            name: 'ssh-dnat-rule'
            destinationAddresses: [
              azfwPip.properties.ipAddress
            ]
            destinationPorts: [
              '50000'
            ]
            ipProtocols: [
              'Any'
            ]
            sourceAddresses: [
              '*'
            ]
            translatedAddress: dnatAddress
            translatedPort: '22'
          }
        ]
      }
    ]
  }
}

// after creating the dnat rule collection group, create the network rule collection group
// these creating two rule collection groups cannot be parallelized
// rule description: east west traffic between the vnets is allowed
resource networkRuleCollectionGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-04-01' = {
  parent: firewallPolicy
  name: 'DefaultNetworkRuleCollectionGroup'
  properties: {
    priority: 200
    ruleCollections: [
      {
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        action: {
          type: 'Allow'
        }
        name: 'east-west-rule'
        priority: 1250
        rules: [
          {
            ruleType: 'NetworkRule'
            name: 'vnet-to-vnet'
            ipProtocols: [
              'Any'
            ]
            destinationAddresses: [
              '10.0.0.0/8'
            ]
            destinationPorts: [
              '*'
            ]
            sourceAddresses: [
              '10.0.0.0/8'
            ]
          }
        ]
      }
    ]
  }
  dependsOn: [
    dnatRuleCollectionGroup
  ]
}
```

## Hubのルートテーブルの設定
今回の環境では以下の条件があります。
- Azure FirewallのDNATルールの作成時に、DNAT先のVMのPrivate IPアドレスが必要なため、VMはAzure Firewallより先にデプロイする必要がある
- VMのデプロイに必要なNICのデプロイには、もちろんサブネットが必要
- HubのルートテーブルはAzure FirewallのPrivate IPに向けたいため、それを動的に取得するにはAzure Firewallより後にデプロイする必要がある
- サブネットのPropertyでルートテーブルを指定することで、サブネットにルートテーブルがアタッチされる

ということで、本来ルートテーブルを先に作成してVNET/Subnet作成時にアタッチするというのが楽なのですが、そうもいかないのでサブネットを先に作ってから後でルートテーブルを作ってアタッチするということをしています。以下がルートテーブルのアタッチのパートです。サブネット作成時に同様の宣言をしているので冗長に見えますが、やむを得ず。

```bicep:hubVnet.bicep
// update subnet to use route table
resource subnetRouteTableAssociation 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = {
  name: hubVnet::hubSubnet.name
  parent: hubVnet
  properties: {
    addressPrefix: '10.0.0.0/24'
    networkSecurityGroup: {
      id: nsgDefault.id
    }
    routeTable: {
      id: routeTable.id
    }
  }
}
```

# おわりに
- GitHubの方は適宜リファクタリングしていく予定です。
- この環境があれば、サクッと検証したいときはかなり楽できるかなーと思いますのでぜひご活用ください。