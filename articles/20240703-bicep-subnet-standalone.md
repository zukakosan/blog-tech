---
title: "2023-11-01 API バージョンを使って Bicep で Azure VNet のサブネットを自由にデプロイする"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure","microsoft","bicep","IaC"]
published: true
publication_name: "microsoft"
published_at: 2024-07-04 09:30
---
:::message
本記事は筆者個人の検証結果に基づくものであり、所属する組織の公式見解を示すものではありません。
正確な情報や最新の仕様については、公式ドキュメントをご確認ください。
:::

# はじめに
自分が今まで Bicep を書いている中で直面した課題の中で非常に煩わしい問題として、 Bicep の記法において、サブネットの定義を VNet の外側に記述することができるのに、**実質的にサブネットを VNet の定義の外側に記述することができない**というものがありました。

Bicep ではこれまで、サブネットを VNet の外側に定義した場合、2 回目のデプロイのタイミングでは、まず空の VNet をデプロイした後に、サブネットをデプロイするという挙動でした。

つまり、**既存のサブネットを削除**して真っ新な VNet を作成し、そのあとでサブネットを追加していました。

その際、サブネットにリソースが存在していると、そのサブネットはそのリソースが握っていることになるため、削除できずエラーになって終了します。要するに、2 回目以降のデプロイに失敗します。

逆に言うと、このエラーは「イミュータブル インフラ」として継続的に運用していない場合、つまり初回の払い出しのみに Bicep を利用している場合は気づかない事象です。

とはいえ、スタンドアロンでサブネットを定義する書き方については、公式ドキュメントにも随所でみられていた書き方（最近では VNet 内で定義するようにドキュメントの修正がかかっているようですが）であり、この事象については下記のスレッドにて活発に議論されているのが現状です。
https://github.com/Azure/bicep-types-az/issues/1687


- 一般的な **VNet の内側**にサブネットを記述している例
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

- **VNet の外側**にサブネットを記述している例
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

# 何が変わったか
2024/02/27 時点で以下のような Tech Blog が公開されました。
https://techcommunity.microsoft.com/t5/azure-networking-blog/azure-virtual-network-now-supports-updates-without-subnet/ba-p/4067952

要するに、VNet のデプロイ時にサブネットのプロパティを指定しない場合、以前は既存のサブネットが削除される挙動だったものが、`2023-09-01` 以降は既存のサブネットに対しては何も行わないという挙動になるという話です。

この記事を発見したタイミングで「これは…望んでいたやつだ！」と思い、一度試した結果うまくいったので歓喜していたのですが、しばらくして再度試すと以前と同様のエラーが生じたため記事化するのは避けていました。

ところが、ドキュメントに記載の`2023-09-01` ではなく `2023-11-01` の API バージョンが利用できるようになったようで、改めてそちらで試したところ既存のサブネットに影響せずデプロイできることを確認しています。当初、この挙動は `eastus2euap` リージョンでのみ展開されていたのですが、現在はほかのリージョンでも同じ挙動となっているようです。

注意点としては、**VNet のデプロイ時にサブネットのプロパティを指定しない** 状態である必要があるため、1 つでも VNet 内にサブネットのプロパティがあるとエラーになります。「すべてスタンドアロンで定義する」か「すべて VNet のプロパティとして定義する」という形になります。

## エラーが生じるケース
- ソースコード(抜粋)[^1]
[^1]:https://github.com/zukakosan/bicep-learn/blob/main/20240319-standalonevnet/main-error.bicep
```bicep:main-error.bicep
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

resource vmSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-09-01' = {
  parent: hubVnet
  name: 'subnet-vm'
  properties: {
    addressPrefix: '10.0.1.0/24'
  }
}
```

- 発生するエラー
```bash
$ az deployment group create -g '20240703-mainerror-2' --template-file main-error.bicep 
Please provide string value for 'adminUsername' (? for help): AzureAdmin
Please provide securestring value for 'adminPassword' (? for help):
{"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/xxxx/resourceGroups/20240703-mainerror-2/providers/Microsoft.Resources/deployments/main-error","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"InUseSubnetCannotBeDeleted","message":"Subnet subnet-vm is in use by /subscriptions/xxxx/resourceGroups/20240703-mainerror-2/providers/Microsoft.Network/networkInterfaces/vm-ubuntu-test-nic/ipConfigurations/ipconfig1 and cannot be deleted. In order to delete the subnet, delete all the resources within the subnet. See aka.ms/deletesubnet.","details":[]}]}}
```

## エラーが生じないケース
- ソースコード(抜粋)[^2] 
[^2]:https://github.com/zukakosan/bicep-learn/blob/main/20240319-standalonevnet/main.bicep
```bicep:main.bicep
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

resource vmSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-11-01' = {
  parent: hubVnet
  name: 'subnet-vm'
  properties: {
    addressPrefix: '10.0.1.0/24'
    networkSecurityGroup: {
      id: nsg.id
    }
  }
  // subnet の同時更新はエラーになるので dependsOn で依存関係を追加
  dependsOn: [
    mainSubnet
  ]
}
```
# 恩恵を受けるケース
サブネットをスタンドアロンで定義した方がよい(書きやすい)ケースとして、条件分岐 (if) を使って特定のサブネットをデプロイするかどうかの選択肢を用意する場合でしょう。パラメータによって環境自体の構成に大きな差異がでることが望ましいかは別の議論として、このようなケースではスタンドアロン型の定義が刺さるのではないでしょうか。

他にもケースはあると思いますが、一例として挙げてみました。

# おわりに
先の GitHub 上の issue を見るに、回答している側としてはスタンドアロン型での定義はそもそも非推奨なようですが、個人的にはこの書き方が扱いやすいです。Terraform ではこのようなエラーに遭遇したことがないため、当初よりユーザビリティを考えてこのあたりをを裏でうまいことやっていたのだろうと推察しています。