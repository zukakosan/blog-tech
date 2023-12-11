---
title: "Azure Route Server を利用してオンプレミス拠点のデフォルトルートを Azure Firewall に向ける"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","IaaS","network","microsoft"]
published: false
publication_name: "microsoft"
---

# はじめに
ExpressRoute で Azure VNet と接続されたオンプレミスからのインターネット向けの通信を、Azure 上の NVA に集約したいこともあると思います(きっと)。Azure Route Server(ARS) を利用して Azure からオンプレミスにデフォルトルートを広報し、Next Hop として NVA の IP を指定することによって逆強制トンネリングのような構成をとることが可能です。本記事では、この構成の検証結果をまとめます。

# 環境準備

## アーキテクチャ
以下のような構成を用意します。下半分(ブランチ)のネットワークについては経路確認のときにどうしても入ってくるので存在感だけ出しております。
![](/images/20231211-reverse-forced-tunneling/arch.png)

## 経路広報用の NVA の準備
オンプレミスに対して 0.0.0.0/0 を広報するための NVA を準備します。手順はこちらの記事[^1]に記載があるようなF流れに従い、FRRouting を構成します。
[^1]: https://zenn.dev/microsoft/articles/azure-route-server-frrouting#nva-%E3%81%AE%E4%BD%9C%E6%88%90

アドレス空間が上記アーキテクチャのようになっている場合は、以下のようになります。
```
ip route 10.0.2.0/24 10.0.1.1
!
router bgp 65001
 neighbor 10.0.2.4 remote-as 65515
 neighbor 10.0.2.4 ebgp-multihop 255
 neighbor 10.0.2.5 remote-as 65515
 neighbor 10.0.2.5 ebgp-multihop 255
 !
 address-family ipv4 unicast
  neighbor 10.0.2.4 soft-reconfiguration inbound
  neighbor 10.0.2.4 route-map rmap-bogon-asns in
  neighbor 10.0.2.4 route-map rmap-azure-asns out
  neighbor 10.0.2.5 soft-reconfiguration inbound
  neighbor 10.0.2.5 route-map rmap-bogon-asns in
  neighbor 10.0.2.5 route-map rmap-azure-asns out
 exit-address-family
exit
!
bgp as-path access-list azure-asns seq 5 permit _65515_
bgp as-path access-list bogon-asns seq 5 permit _0_
bgp as-path access-list bogon-asns seq 10 permit _23456_
bgp as-path access-list bogon-asns seq 15 permit _1310[0-6][0-9]_|_13107[0-1]_
bgp as-path access-list bogon-asns seq 20 deny _65515_
bgp as-path access-list bogon-asns seq 25 permit ^65
!
route-map rmap-bogon-asns deny 5
 match as-path bogon-asns
exit
!
route-map rmap-bogon-asns permit 10
exit
!
route-map rmap-azure-asns deny 5
 match as-path azure-asns
exit
!
route-map rmap-azure-asns permit 10
exit
!
```

この時点で、FRRouting と ARS の間で BGP が上がっていることを確認します。
```
vm-frr# show ip bgp nei 10.0.2.4
BGP neighbor is 10.0.2.4, remote AS 65515, local AS 65001, external link
  Local Role: undefined
  Remote Role: undefined
  BGP version 4, remote router ID 0.0.0.0, local router ID 10.0.1.6
  BGP state = Active
  Last read 00:09:17, Last write 00:00:59
  Hold time is 180 seconds, keepalive interval is 60 seconds
  Configured hold time is 180 seconds, keepalive interval is 60 seconds
  Configured tcp-mss is 0, synced tcp-mss is 0
  Configured conditional advertisements interval is 60 seconds
```