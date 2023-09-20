---
title: "BicepでAzure DNS Private Resolver(DNS集約型)をサクッと作る"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
publication_name: "microsoft"
---
# 記事の位置づけ
- Azure DNS Private Resolverをサクッと作るBicepの補足

# Azure DNS Private Resolver(ADPR)
- PaaSのDNSフォワーダー的なイメージ
- クロスプレミスで名前解決が必要な場合の推奨コンポーネントとなっている
- 転送ルールセットに条件付きフォワーダに類する設定を記述し、VNETにリンクする

# Azure DNS
- `168.63.129.16`でホストされるVNETごとに設定された既定のDNSサーバ
- リンクローカルアドレスとなっているため外部から(例えばExpressRoute越しにオンプレミスから)叩けない

# Azure Private DNS Zone
- プライベートなドメイン名の管理および解決をする
- VNETに関連付けておくとAzure DNSが解決してくれる
- Private Endpointを作成すると専用のPrivate DNS Zoneを作ってくれる

# ADDS/DNS
- オンプレミス側のDNSサーバーは直接Azure DNSにフォワードができないので、条件付きフォワーダの設定としてADPRのinbound endpoint IPを指定する


