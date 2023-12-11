---
title: "Azure Route Server を利用してオンプレミス拠点のデフォルトルートを Azure Firewall に向ける"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","IaaS","network","microsoft"]
published: false
publication_name: "microsoft"
---

# はじめに
ExpressRoute で Azure VNet と接続されたオンプレミスからのインターネット向けの通信を、Azure 上の NVA に集約したいこともあると思います(きっと)。Azure Route Server を利用して Azure からオンプレミスにデフォルトルートを広報し、Next Hop として NVA の IP を指定することによって逆強制トンネリングのような構成をとることが可能です。本記事では、この構成の検証結果をまとめます。

# アーキテクチャ
以下のような構成を用意します。下半分(ブランチ)のネットワークについては経路確認のときにどうしても入ってくるので存在感だけ出しております。
![](/images/20231211-reverse-forced-tunneling/arch.png)

