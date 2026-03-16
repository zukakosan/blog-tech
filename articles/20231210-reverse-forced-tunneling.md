---
title: "Azure Route Server を利用してオンプレミス拠点のデフォルトルートを Azure Firewall に向ける"
emoji: "🫎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","IaaS","network","microsoft"]
published: true
published_at: 2023-12-13 09:00
publication_name: "microsoft"
---
:::message
本記事は筆者個人の検証結果に基づくものであり、所属する組織の公式見解を示すものではありません。
正確な情報や最新の仕様については、公式ドキュメントをご確認ください。
:::
本記事は、Microsoft Azure Tech Advent Calendar 2023[^1] の 13 日目の記事です。

[^1]:https://qiita.com/advent-calendar/2023/microsoft-azure-tech

https://qiita.com/advent-calendar/2023/microsoft-azure-tech

# はじめに
ExpressRoute で Azure VNet と接続されたオンプレミスからのインターネット向けの通信を、Azure 上の NVA に集約したいこともあると思います(きっと)。Azure Route Server(ARS) を利用して Azure からオンプレミスにデフォルトルートを広報し、NEXT_HOP として Azure Firewall の IP を指定することによって逆強制トンネリングのような構成をとることが可能です。本記事では、この構成の検証結果をまとめます。

# 環境準備

## アーキテクチャ
以下のような構成を用意します。下半分(ブランチ)のネットワークについては経路確認のときにどうしても入ってくるので存在感だけ出しております。
![](/images/20231211-reverse-forced-tunneling/arch.png)

## 経路広報用の NVA の準備
オンプレミスに対して 0.0.0.0/0 を広報するための NVA を準備します。手順はこちらの記事[^2]に記載があるような流れに従い、FRRouting を構成します。
[^2]: https://zenn.dev/microsoft/articles/azure-route-server-frrouting#nva-%E3%81%AE%E4%BD%9C%E6%88%90

アドレス空間が上記アーキテクチャのようになっている場合は、以下のようになります。

:::details FRRouting Config
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
:::

## ARS に BGP ピアの追加
ARS のリソースから BGP ピアを登録します。
![](/images/20231211-reverse-forced-tunneling/01.png)

## FRRouting が受け取っている経路の確認
この時点で、NVA 側で受けている経路を確認します。BGPでオンプレミス側 (10.20.10.0/24)、Hub(10.0.0.0/16)、VPN ブランチ (10.50.0.0/16) の経路を受け取っていることが分かります。

```
vm-frr# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/100] via 10.0.1.1, eth0, src 10.0.1.6, 00:37:22
B>  10.0.0.0/16 [20/0] via 10.0.2.4 (recursive), weight 1, 00:02:03
  *                      via 10.0.1.1, eth0, weight 1, 00:02:03
                       via 10.0.2.5 (recursive), weight 1, 00:02:03
                         via 10.0.1.1, eth0, weight 1, 00:02:03
C>* 10.0.1.0/24 is directly connected, eth0, 00:37:22
S>* 10.0.2.0/24 [1/0] via 10.0.1.1, eth0, weight 1, 00:27:49
B>  10.20.10.0/24 [20/0] via 10.0.2.4 (recursive), weight 1, 00:02:03
  *                        via 10.0.1.1, eth0, weight 1, 00:02:03
                         via 10.0.2.5 (recursive), weight 1, 00:02:03
                           via 10.0.1.1, eth0, weight 1, 00:02:03
B>  10.50.0.0/16 [20/0] via 10.0.2.4 (recursive), weight 1, 00:02:03
  *                       via 10.0.1.1, eth0, weight 1, 00:02:03
                        via 10.0.2.5 (recursive), weight 1, 00:02:03
                          via 10.0.1.1, eth0, weight 1, 00:02:03
K>* 168.63.129.16/32 [0/100] via 10.0.1.1, eth0, src 10.0.1.6, 00:37:22
K>* 169.254.169.254/32 [0/100] via 10.0.1.1, eth0, src 10.0.1.6, 00:37:22
```

## NVA から デフォルトルートを広報

BGP で FRRouting からデフォルトルートを広報してみます。以下のように `network 0.0.0.0/0` として BGP の構成を追加しました。

```
router bgp 65001
 neighbor 10.0.2.4 remote-as 65515
 neighbor 10.0.2.4 ebgp-multihop
 neighbor 10.0.2.5 remote-as 65515
 neighbor 10.0.2.5 ebgp-multihop
 !
 address-family ipv4 unicast
  network 0.0.0.0/0
  neighbor 10.0.2.4 soft-reconfiguration inbound
  neighbor 10.0.2.4 route-map rmap-bogon-asns in
  neighbor 10.0.2.4 route-map rmap-azure-asns out
  neighbor 10.0.2.5 soft-reconfiguration inbound
  neighbor 10.0.2.5 route-map rmap-bogon-asns in
  neighbor 10.0.2.5 route-map rmap-azure-asns out
 exit-address-family
exit
```

一方でこのままだと 次ホップ が `0.0.0.0` になってしまっています。
```
vm-frr# show bgp ipv4 unicast neighbors 10.0.2.4 advertised-routes
BGP table version is 7, local router ID is 10.0.1.6, vrf id 0
Default local pref 100, local AS 65001
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

    Network          Next Hop            Metric LocPrf Weight Path
 *> 0.0.0.0/0        0.0.0.0                  0         32768 i

Total number of prefixes 1
```

次ホップ を Azure Firewall に向けるため、ルートマップに NEXT_HOP 属性を設定します。
```
route-map rmap-azure-asns permit 10
 set ip next-hop 10.0.3.4
exit
```

:::message alert
- この時点で、Azure Firewall のネクストホップの Azure Firewall になってしまうので、AzureFirewallSubnet の ルートテーブルで `0.0.0.0/0->Internet` の UDR が必要となります。
- また、Azure Firewall のアプリケーション規則にインターネットへの http/https アクセス許可の規則を追加しておきます。この許可の規則がない場合、インターネットアクセス不可となります。
:::

# 経路確認
どの様な経路が伝搬されているか、オンプレミス・ハブ・ブランチのそれぞれで確認してみましょう。

## オンプレミス

オンプレミス側に来ている経路を確認してみると、`0.0.0.0/0`が(`172.16.0.2` = MSEE)を向いていることがわかります。
```
er#show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is 172.16.0.2 to network 0.0.0.0

B*    0.0.0.0/0 [20/0] via 172.16.0.2, 01:09:00
      10.0.0.0/8 is variably subnetted, 4 subnets, 3 masks
B        10.0.0.0/16 [20/0] via 172.16.0.2, 1d01h
C        10.20.10.0/24 is directly connected, Vlan10
L        10.20.10.1/32 is directly connected, Vlan10
B        10.50.0.0/16 [20/0] via 172.16.0.2, 22:09:35
      172.16.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.16.0.0/30 is directly connected, GigabitEthernet8.708
L        172.16.0.1/32 is directly connected, GigabitEthernet8.708

```

## ハブ VNet 側
ハブ VNet の VPN Gateway に来ている経路を確認すると、デフォルトルートの次ホップが Azure Firewall になっていることがわかります。
```
$ az network vnet-gateway list-learned-routes -g 20231208-cloudlab -n vpngw-clab-hub --output table
Network        NextHop    Origin    SourcePeer    AsPath       Weight
-------------  ---------  --------  ------------  -----------  --------
10.0.0.0/16               Network   10.0.0.14                  32768
10.50.0.5/32              Network   10.0.0.14                  32768
10.50.0.5/32   10.0.0.15  IBgp      10.0.0.15                  32768
10.20.10.0/24  10.0.0.4   IBgp      10.0.2.4      12076-65150  32768
10.20.10.0/24  10.0.0.4   IBgp      10.0.2.5      12076-65150  32768
10.50.0.4/32              Network   10.0.0.14                  32768
10.50.0.4/32   10.0.0.15  IBgp      10.0.0.15                  32768
10.50.0.0/16   10.50.0.4  EBgp      10.50.0.4     65020        32768
10.50.0.0/16   10.0.0.15  IBgp      10.0.2.4      65020        32768
10.50.0.0/16   10.0.0.15  IBgp      10.0.2.5      65020        32768
10.50.0.0/16   10.50.0.5  EBgp      10.50.0.5     65020        32768
10.50.0.0/16   10.0.0.15  IBgp      10.0.0.15     65020        32768
0.0.0.0/0      10.0.3.4   IBgp      10.0.2.4      65001        32768
0.0.0.0/0      10.0.3.4   IBgp      10.0.2.5      65001        32768
10.0.0.0/16               Network   10.0.0.15                  32768
10.50.0.5/32              Network   10.0.0.15                  32768
10.50.0.5/32   10.0.0.14  IBgp      10.0.0.14                  32768
10.20.10.0/24  10.0.0.4   IBgp      10.0.2.4      12076-65150  32768
10.20.10.0/24  10.0.0.4   IBgp      10.0.2.5      12076-65150  32768
10.50.0.4/32              Network   10.0.0.15                  32768
10.50.0.4/32   10.0.0.14  IBgp      10.0.0.14                  32768
10.50.0.0/16   10.50.0.4  EBgp      10.50.0.4     65020        32768
10.50.0.0/16   10.50.0.5  EBgp      10.50.0.5     65020        32768
10.50.0.0/16   10.0.0.14  IBgp      10.0.0.14     65020        32768
0.0.0.0/0      10.0.3.4   IBgp      10.0.2.4      65001        32768
0.0.0.0/0      10.0.3.4   IBgp      10.0.2.5      65001        32768

```

## ブランチ VNet 側
ブランチ VNet の VPN Gateway で見えている経路を確認すると、デフォルトルートは受け取っていません。VPN Gateway はデフォルトルートを BGP ピアに広報しません[^3]。
[^3]: https://blog.aimless.jp/archives/2022/07/create-default-route-to-firewall-with-route-server-nexthop/

```
$ az network vnet-gateway list-learned-routes -g 20231208-cloudlab -n vpngw-clab-brach-eus --output table
Network        NextHop    Origin    SourcePeer    AsPath             Weight
-------------  ---------  --------  ------------  -----------------  --------
10.50.0.0/16              Network   10.50.0.5                        32768
10.0.0.14/32              Network   10.50.0.5                        32768
10.0.0.14/32   10.50.0.4  IBgp      10.50.0.4                        32768
10.0.0.15/32              Network   10.50.0.5                        32768
10.0.0.15/32   10.50.0.4  IBgp      10.50.0.4                        32768
10.0.0.0/16    10.0.0.15  EBgp      10.0.0.15     65515              32768
10.0.0.0/16    10.50.0.4  IBgp      10.50.0.4     65515              32768
10.0.0.0/16    10.0.0.14  EBgp      10.0.0.14     65515              32768
10.20.10.0/24  10.0.0.14  EBgp      10.0.0.14     65515-12076-65150  32768
10.20.10.0/24  10.0.0.15  EBgp      10.0.0.15     65515-12076-65150  32768
10.20.10.0/24  10.50.0.4  IBgp      10.50.0.4     65515-12076-65150  32768
10.50.0.0/16              Network   10.50.0.4                        32768
10.0.0.14/32              Network   10.50.0.4                        32768
10.0.0.14/32   10.50.0.5  IBgp      10.50.0.5                        32768
10.0.0.15/32              Network   10.50.0.4                        32768
10.0.0.15/32   10.50.0.5  IBgp      10.50.0.5                        32768
10.0.0.0/16    10.0.0.15  EBgp      10.0.0.15     65515              32768
10.0.0.0/16    10.50.0.5  IBgp      10.50.0.5     65515              32768
10.0.0.0/16    10.0.0.14  EBgp      10.0.0.14     65515              32768
10.20.10.0/24  10.0.0.14  EBgp      10.0.0.14     65515-12076-65150  32768
10.20.10.0/24  10.0.0.15  EBgp      10.0.0.15     65515-12076-65150  32768
10.20.10.0/24  10.50.0.5  IBgp      10.50.0.5     65515-12076-65150  32768

```

# 疎通確認
今までの構成を行ったうえでオンプレミスから例によって確認くん[^4]にアクセスしてみます。すると、SNAT されて Azure Firewall の PIP でアクセスしていることが分かります。
![](/images/20231211-reverse-forced-tunneling/02.png)
![](/images/20231211-reverse-forced-tunneling/03.png)

[^4]:https://www.ugtop.com/spill.shtml

# まとめ
- BGP を話せる NVA を用意して ARS とピアを張り、デフォルトルートの広報 + NEXT_HOP属性の追加しました。
- この構成により外部アクセスに Azure Firewall を経由させる構成が取れることを確認しました。