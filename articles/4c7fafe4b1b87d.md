---
title: "Azure Monitor Private Link Scopeをササっと作るBicep"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bicep","azure","IaC","terraform","monitoring"]
published: false
publication_name: "microsoft"

---
# モチベ
- Azure Monitor Private Link Scope(AMPLS)の環境が不意に欲しくなることがある
- AMPLSの構成、たまにやると結構面倒なのでテンプレート化しておくと楽そう
- Bicepの復習がてら自分用に作っておこう

# AMPLSとは
- 大雑把にはAzure Monitor関連リソースをPrivate Endpoint経由で利用するための仕組み
- 例えばLog AnalyiticsへのデータインジェストはPrivate Endpointから直接行えるわけではなく、AMPLSを構成してそこを経由させる必要がある。これが面倒。

![](/images/20230818-bicep-ampls/private-link-basic-topology.png)

# 作成する環境
作成したソース

https://github.com/zukakosan/bicep-ampls

以下のような環境がデプロイされる
![](/images/20230818-bicep-ampls/ampls-arch.png)

# サンプルコード内のポイント
構成

```
main.bicep
∟ modules/
    ∟ vm.bicep
    ∟ DCRDCE.bicep
```
## VMの作成とDCR割り当ての処理
- 1ファイルですべてのリソースを作成するのでもよいのだが、個人的な好みとしてモジュール化できるところはしたいので、VMの作成は`vm.bicep`で行う
    - その都合として、`DCRDCE.bicep`も作る羽目になった
- データ収集ルール(DCR)の割り当てに際して、対象のVMを割り当てる必要があるのだが、以下部分でモジュールの中でVMを作成すると、`main.bicep`上では作成されたVMのリソースを参照できないため、参照したければ`existing`を使った形で既存リソースを参照する必要がある
- しかし、モジュールの実行と`existing`による既存リソースの取得は非同期のため、VMが作成される前に既存リソースを参照しようとしてエラーになる。さらに`existing`は`dependsOn`による依存関係が作れない
- よってあえてDCRやデータ収集エンドポイント(DCE)周りの処理は`DCRDCE.bicep`としてモジュール化することで対処した（モジュールについては`dependsOn`が利用できるため）。

```bicep:main.bicep
// Call VM module
module CreateVM './modules/vm.bicep' = {
  name: 'vm'
  params: {
    location: location
    subnetId: VNetCloud::SubnetMain.id
    vmName: vmName
    vmAdminUserName: vmAdminUserName
    vmAdminPassword: vmAdminPassword
  }
}

// To execute "resource~existing" after "CreateVM" module, include process in the same module and use "dependsOn"
module DCRDCE './modules/DCRDCE.bicep' = {
  name: 'attachDcrDce'
  params: {
    location: location
    vmName: vmName
    LawName: LawAmpls.name
    LawId: LawAmpls.id
    // AMPLS: AMPLS
  }
  dependsOn:[
    CreateVM
  ]
}
```

## Private DNS Zone周りの処理
### Private DNS Zoneの作成とVNetへの割り当て
- Private Endpointを利用するので、DNS周りの処理もきちんと作っていく必要がある
- `param`でarrayとして作成する必要があるゾーンを定義しておき、以下部分でPrivate DNS Zoneの作成とVNETへのアタッチをループを使って一気に行う

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
- さらに、Private Endpointに対するAレコードを各種Private DNS Zoneに自動登録するための仕組みとして`Microsoft.Network/privateEndpoints/privateDnsZoneGroups`という概念があり、そこも処理する必要がある
    - Azure PortalでPrivate Endpointを構成する場合には、ここはあまり意識しなくていいようになっている
- また、1つのPrivate Endpointに対しては1つのPrivate DNS Zone Groupしか定義できないため、`privateDnsZoneConfigs`配下でループを回す
- AMPLSの場合は1つのPrivate Endpoint NICに対して多数のIP構成を持つのでこのあたりの処理が若干煩雑

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

この部分についてはこちらのブログ記事がかなり参考になりました

https://blog.aimless.jp/archives/2022/07/use-integration-between-private-endpoint-and-private-dns-zone-in-bicep/ 

# おわり
- 細かいところはソースをご参照くださいませ