---
title: "Bicepで可用性ゾーンを利用したWeb(IaaS)-DB(PaaS)環境をサクッと作る"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bicep","network","azure"]
published: false
publication_name: "microsoft"
---
# これはなに...?
- 可用性を意識してWeb(IaaS)-DB(PaaS)環境サクッと立てるためのBicepを作ったのでその共有
- 構成と書き方のポイントを紹介
- GitHubにあるソースコードの補足説明的なもの

# 該当リポジトリ
https://github.com/zukakosan/bicep-webdb-quickstrater

# デプロイされる環境
以下のような環境をデプロイします。
![](/images/20231020-bicep-webdb/webdb-arch.png)

# アーキテクチャ上のポイント
## Web用サブネット
WebサーバーとしてはIaaS VMを採用しています。可用性ゾーンを分けて冗長構成としています。デプロイするタイミングで`cloud-init.yml`を渡すことで、Apacheのインストールとサービスの起動を行い、Webサーバ化しています。

## DB用サブネット
データベースとしてはAzure Database for PostgreSQLを採用し、ゾーン冗長HA構成としています。SLAは99.99%となります。
> Azure Database for PostgreSQL - フレキシブル サーバーは、自動フェールオーバー機能による高可用性構成を提供します。 高可用性ソリューションは、コミットされたデータが障害によって失われないこと、データベースがアーキテクチャでの単一障害点にならないことを保証するように設計されています。 高可用性が構成されている場合、フレキシブル サーバーによってスタンバイが自動的にプロビジョニングされて管理されます。(https://learn.microsoft.com/ja-jp/azure/postgresql/flexible-server/concepts-high-availability)

# Bicep上のポイント
ディレクトリ構成としては以下のようなイメージです。
```
- main.bicep
- main.bicepparam
- modules/
        └ appgw.bicep
        └ bastion.bicep
        └ postgreSql.bicep
        └ vm.bicep
        └ vnet.bicep
        └ cloud-init.yml
```
## Application Gateway
ドキュメントを見ればわかるのですが、[Application Gatewayの定義](https://learn.microsoft.com/en-us/azure/templates/microsoft.network/applicationgateways?pivots=deployment-language-bicep)がかなり膨大で苦労しました。

また、Application Gateway作成時、バックエンドプールに追加するVMのプライベートIPが複数になるのでどう渡そうか迷ったのですが以下のように実装しました。
- `main.bicep`側で、ループを利用してVMを作成
- `vm.bicep`ではVMのプライベートIPを`output`に載せる
- `appgw.bicep`のパラメータとして各VM作成ループの出力から得られるVMのプライベートIPを`array`として渡す
そんな感じで試行錯誤したのが以下です。

```bicep:main.bicep
module createAppGw './modules/appgw.bicep' = {
  name: 'createAppGw'
  params: {
    location: location
    appGwSubnetId: createVnet.outputs.appGwSubnetId
    backendVmPrivateIps: [for i in range(0, webVmCount): createWebVms[i].outputs.vmPrivateIp]
  }
  dependsOn: [
    createVnet
  ]
}
```