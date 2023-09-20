---
title: "BicepでAzure DNS Private Resolver(DNS集約型)をサクッと作る"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","bicep","IaC"."DevOps"]
published: true
publication_name: "microsoft"
---
# 記事の位置づけ
- Azure DNS Private Resolverをサクッと作るBicepの各コンポーネントにおける補足

# デプロイされる環境

## ソース
以下Repositoryの`main.bicep`によりデプロイする

https://github.com/zukakosan/bicep-adpr-quick-starter

## Architecuture
以下のようなコンポーネントがデプロイされる

![](/images/20230920-bicep-adpr-quickstarter/adpr-arch.png)


# 各種コンポーネント補足

## Azure DNS Private Resolver(ADPR)
- PaaSのDNSフォワーダー的なイメージ
- クロスプレミスで名前解決が必要な場合の推奨コンポーネントとなっている
- 転送ルールセットに条件付きフォワーダに類する設定を記述し、VNETにリンクする
- このBicepではHubのVNETにのみ転送ルールセットをリンクする

## Azure DNS
- `168.63.129.16`でホストされるVNETごとに設定された既定のDNSサーバ
- リンクローカルアドレスとなっているため外部から(例えばExpressRoute越しにオンプレミスから)叩けない

## Azure Private DNS Zone
- プライベートなドメイン名の管理および解決をする
- VNETに関連付けておくとAzure DNSが解決してくれる
- Private Endpointを作成すると専用のPrivate DNS Zoneを作ってくれる

## ADDS/DNS
- オンプレミス側のDNSサーバーは直接Azure DNSにフォワードができないので、条件付きフォワーダの設定としてADPRのinbound endpoint IPを指定する
- 今回のBicepではOS上の構成まで実施しないため、OSにログインしてDNSサーバを構成する必要がある
- 必要に応じてPublic IPの付与が必要となる

## VPN Gateway
- オンプレミス―クラウド間の接続の模倣のために`VNET2VNET`のS2SVPNを構成している
- 簡単のために、Local Network Gatewayを使わないパターンとした

# おわり
- これでいつでもADPR環境をデプロイできてうれしい
- 今後もエンタープライズのガチガチの大規模環境というよりは、自分がサクッと使いまわせるような、ある機能にフォーカスしたネタ帳的Bicepを生み出していきたい所存
- 頑張るぞ～