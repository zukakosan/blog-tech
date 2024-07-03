---
title: "Bicep でサブネットを定義するときにスタンドアロンで書いてもエラーが出なくなったのが嬉しい"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure","microsoft","bicep","IaC"]
published: false
---

# はじめに
自分が今まで Bicep を書いている中で直面した課題として非常に煩わしい問題として、サブネットを VNet の定義の外側に記述することができないというものがあります。

一般的な **VNet の内側**にサブネットを記述している例
```bicep
resource hubVnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: 'subnet-001'
        properties: {
          addressPrefix: '10.0.0.0/24'
        }
      }
    ]
  }
}
```

**VNet の外側**にサブネットを記述している例
```bicep
resource hubVnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
  }
}

resource mainSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-11-01' = {
  parent: hubVnet
  name: 'subnet-001'
  properties: {
    addressPrefix: '10.0.0.0/24'
    networkSecurityGroup: {
      id: nsg.id
    }
  }
}
```

正確には、記述できないというよりは、2 回目のデプロイのタイミングでサブネットが存在しない空の VNet をデプロイした後に、サブネットをデプロイするという順序になるため、まずサブネットを削除する動きになります。
そして、サブネットにリソースが存在しているともちろんそのサブネットはそのリソースが握っていることになるため、削除できずエラーになって終了します。

逆に言うと、このエラーは「イミュータブル インフラ」として継続的に運用していない場合、つまり初回の払い出しのみに Bicep を利用している場合は気づかない事象です。

とはいえ、スタンドアロンでサブネットを定義する書き方については、公式ドキュメントにも随所でみられていた書き方(最近では VNet 内で定義するように修正がかかっているようですが)であり、この事象については下記のスレッドにて活発に議論されているのが現状です。
https://github.com/Azure/bicep-types-az/issues/1687