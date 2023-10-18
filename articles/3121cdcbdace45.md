---
title: "Bicepで簡易的なHub-Spoke環境をサクッと作る"
emoji: "🐕"
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

## アーキテクチャ上のポイント
### Hub VNET
HubにはAzure Firewallをデプロイします。Azure Firewallにはネットワークルールとして東西方向のトラフィック許可のための簡易的なルールが追加されています(from:10.0.0.0/8, to:10.0.0.0/8, Allow)。もっと限定したい場合には適宜経路を絞る必要があります。

また、HubのAzure Firewall上でDNATルールにより、Port50000でJumpbox仮想マシンのPort22にDNATできるようになっています。Jumpbox仮想マシンとDNATルールは既定で作成されます。

一方でAzure Bastionをデプロイしたい場合には、パラメータとして`bastionEnabled`を`Enabled`として渡します。必要に応じてデプロイすることが可能です。

### Spoke VNET
Spokeには、VM1台をデプロイし、接続されたサブネットにはNSGとルートテーブルを設定します。ルートテーブルでは、`0.0.0.0/0 > Azure Firewall`とするUDRを定義しています。これにより、Spoke1-AzureFirewall-Spoke2の通信が可能となっています。

Spokeの個数はパラメータで渡すことで、横方向にスケーリングできます。Spokeの構成はすべて同じになります。