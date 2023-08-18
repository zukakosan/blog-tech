---
title: "Azure Monitor Private Link Scopeをササっと作るBicep"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bicep","azure","IaC","terraform"]
published: false
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
## main.bicep
- 基本的にはこのBicepファイル上でリソースを作成していく
- 1ファイルですべてのリソースを作成するのでもよいのだが、個人的な好みとしてモジュール化できるところはしたいので、VMの作成は`vm.bicep`で行う
    - その都合として、`DCRDCE.bicep`も作る羽目になった

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
```
# おわり
- 細かいところはソースをご参照くださいませ