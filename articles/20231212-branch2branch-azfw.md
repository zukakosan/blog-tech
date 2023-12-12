---
title: "Azure Route Server を利用した Branch-to-branch 接続で Azure Firewall を経由させてみる"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに
Azure Route Server (ARS) を利用するシナリオとして、ExpressRoute 拠点と VPN 拠点のブランチ間接続を実現するというものがあります。ドキュメント[^1]で示されているイメージとしてが以下画像です。これ自体は、ARS を同一 VNet にデプロイして Branch-to-branch を Enabled にするだけで実現することができます。そのブランチ間接続のデータパスは ExpressRoute Gateway と VPN Gatewayの間が直接つながるようなイメージになります。この場合では、例えばブランチ間の接続の間に Azure Firewall 等の NVA を経由させたいという状況に対応することができません。その実現のためにいろいろと試したのでまとめとして残しておきます。
![](/images/20231212-branch2branch-azfw/expressroute-and-vpn-with-route-server.png)
[^1]:https://learn.microsoft.com/ja-jp/azure/route-server/expressroute-vpn-support


# アーキテクチャ

# 動作確認