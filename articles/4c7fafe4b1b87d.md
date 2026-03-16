---
title: "Bicep で AMPLS を使ったプライベートな Log Analyitics 環境をササっと作る"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bicep","azure","IaC","terraform","monitoring"]
published: true
publication_name: "microsoft"

---
:::message
これは個人の検証であり、サポートの可否を保証するものではありません。
:::
# はじめに
Azure Monitor Private Link Scope (AMPLS) の環境が不意に欲しくなることがありますよね。AMPLS の構成、たまにやると結構面倒なのでテンプレート化しておくと楽ではないでしょうか。ということで作成してみました。

# AMPLSとは
大雑把には Azure Monitor 関連リソースを Private Endpoint 経由で利用するための仕組みです。例えば Log Analyitics へのデータインジェストは Private Endpoint から直接行えるわけではなく、AMPLS を構成してそこを経由させる必要があります。いわゆる PaaS の閉域化とは少し毛色が異なるため、なかなかシンプルに構成するのは難しいところです。また、関連するリソースの種類がいくつかありますが、カスタムログ取得の要否等によっても少し変わってきます。

![](/images/20230818-bicep-ampls/private-link-basic-topology.png)

# 作成する環境
作成したソースを次のリポジトリに格納しています。

https://github.com/zukakosan/bicep-ampls-quickstarter

次のような環境がデプロイされる想定です。
![](/images/20230818-bicep-ampls/ampls-arch.png)

# サンプル コード内のポイント
サンプル コードは次のような構成となっています。

```
main.bicep
∟ modules/
    ∟ vm.bicep
    ∟ dcr-dce.bicep
```

## VM の作成と DCR 割り当ての処理
モジュール化できるところは切り出しておきたいので、VM の作成は`vm.bicep`で行います。その都合として、`dcr-dce.bicep`も作成しています。

データ収集ルール (DCR) の割り当てに際して、対象の VM を割り当てる必要があります。モジュールの中で VM を作成すると、`main.bicep`上では作成された VM のリソースを参照できないため、`existing` を使った形で既存リソースを参照しています。

モジュールの実行と `existing` による既存リソースの取得は非同期のため、VM が作成される前に既存リソースを参照しようとしてエラーになります。さらに `existing` は `dependsOn` による依存関係が作成できません。

よって、あえてDCRやデータ収集エンドポイント(DCE)周りの処理は`dcr-dce.bicep`としてモジュール化することで対処しています（モジュールについては`dependsOn`が利用できるため）。

```bicep:main.bicep
// Call VM module
module CreateVM './modules/vm.bicep' = {
  name: 'vm-module'
  params: {
    location: location
    subnetId: filter(Vnet.properties.subnets, subnet => subnet.name == 'subnet-main')[0].id
    vmName: vmName
    vmAdminUserName: vmAdminUserName
    vmAdminPassword: vmAdminPassword
  }
}

// To execute "resource~existing" after "CreateVM" module, include process in the same module and use "dependsOn"
module DcrDce './modules/dcr-dce.bicep' = {
  name: 'attachDcrDce-module'
  params: {
    location: location
    vmName: vmName
    LawName: LawAmpls.name
    LawId: LawAmpls.id
    suffix: suffix
    // AMPLS: AMPLS
  }
  dependsOn:[
    CreateVM
  ]
}
```

## Azure Monitor Agent のインストールとマネージド ID
Azure portal では VM に対して DCR を割り当てるという動作をすると自動で Azure Monitor Agent がインストールされ、そのエージェントが利用するマネージド ID も VM に対して有効化されます。

検証中の動きを見るに、Bicep では DCR を割り当てるだけではこのあたりの処理が適切な順序で完了しないため、「ユーザー割り当てマネージド ID の作成」と「Azure Monitor Agent (拡張機能)のインストール」を Bicep のコードとして記述するようにしました。

```bicep:vm.bicep
resource UaMid 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${vmName}-mid'
  location: location
}

resource WindowsVm 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: vmName
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${UaMid.id}': {}
    }
  }

//... 中略

resource AMA 'Microsoft.Compute/virtualMachines/extensions@2021-11-01' = {
  name: 'AzureMonitorWindowsAgent'
  parent: WindowsVm
  location: location
  properties: {
    publisher: 'Microsoft.Azure.Monitor'
    type: 'AzureMonitorWindowsAgent'
    typeHandlerVersion: '1.0'
    autoUpgradeMinorVersion: true
    settings: {
      authentication: {
        managedIdentity: {
          'identifier-name': 'mi_res_id'
          'identifier-value': UaMid.id
        }
      }
    }
  }
}
```



## Private DNS Zone 周りの処理
### Private DNS Zone の作成と VNet への割り当て
Private Endpoint を利用するので、DNS 周りの処理もきちんと作っていく必要があります。`param` で作成する必要があるゾーンを array で定義しておき、以下部分で Private DNS Zone の作成と VNet へのアタッチをループを使って一気に行っています。

```bicep:main.bicep
// Create Private DNS Zone
// Define zone name as array
resource privateDnsZoneForAmpls 'Microsoft.Network/privateDnsZones@2020-06-01' = [for zone in zones: {
  name: 'privatelink.${zone}'
  location: 'global'
  properties: {
  }
}]

// Connect Private DNS Zone to VNet
resource privateDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = [for (zone,i) in zones: { 
  parent: privateDnsZoneForAmpls[i]
  name: '${zone}-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: VNetCloud.id
    }
  }
}]
```
### Private DNS Zone Group
さらに、Private Endpoint に対する A レコードを各種 Private DNS Zone に自動登録するための仕組みとして`Microsoft.Network/privateEndpoints/privateDnsZoneGroups`という概念があり、そこも処理する必要があります。(Azure portal で Private Endpoint を構成する場合には、ここはあまり意識しなくていいようになっています。)

また、1 つの Private Endpoint に対しては 1 つの Private DNS Zone Group しか定義できないため、`privateDnsZoneConfigs` 配下でループを回します。AMPLS の場合は1つの Private Endpoint NIC に対して多数の IP 構成を持つのでこのあたりの処理が若干煩雑です。この部分についてはこちらのブログ記事[^1] がかなり参考になりました。
[^1]:https://blog.aimless.jp/archives/2022/07/use-integration-between-private-endpoint-and-private-dns-zone-in-bicep/ 

```bicep:main.bicep
// Create Private DNS Zone Group for "pe-ampls" to register A records automatically
resource peDnsGroupForAmpls 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2021-05-01' = {
  parent: PeAmpls
  name: 'pvtEndpointDnsGroupForAmpls'
  properties: {
    privateDnsZoneConfigs: [
      for (zone,i) in zones: {
        name: privateDnsZoneForAmpls[i].name
        properties: {
          privateDnsZoneId: privateDnsZoneForAmpls[i].id
        }
      }
    ]
  }
}

```

# 注意
## 懸念
`Internal Server Error`で初回のデプロイが失敗することがあります。
## ワークアラウンド
再度デプロイを実行すると成功します。内部的なリソース作成のタイミングによる問題と推定されますが、原因が分かり次第修正対応予定です。

# AMPLS 経由となっていることの確認
デプロイ完了後、ログが流れるまで暫く待ち、Log Analytics を開いて `Heartbeat` テーブルにクエリをかけてみると `ComputerIP` が IPv6 表記で記録されています。これが、プライベートな通信になっている証拠です。
![](/images/20230818-bicep-ampls/law.png)

# おわりに
細かいところはソースをご参照ください。ご意見、改善案等は GitHub にて Issue や PR を上げていただければと思います。