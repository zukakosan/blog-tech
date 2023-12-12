---
title: "Azure Route Server を利用した Branch-to-branch 接続で Azure Firewall を経由させてみる"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに
Azure Route Server (ARS) を利用するシナリオとして、ExpressRoute 拠点と VPN 拠点のブランチ間接続を実現するというものがあります。ドキュメント[^1]で示されているイメージとしてが以下画像です。これ自体は、ARS を同一 VNet にデプロイして Branch-to-branch を Enabled にするだけで実現することができます。そのブランチ間接続のデータパスは ExpressRoute Gateway と VPN Gatewayの間が直接つながるようなイメージになります。この場合では、例えばブランチ間の接続の間に Azure Firewall 等の NVA を経由させたいという状況に対応することができません。このシナリオでどのように構成すれば実現できるかを検証しました。
![](/images/20231212-branch2branch-azfw/expressroute-and-vpn-with-route-server.png)
[^1]:https://learn.microsoft.com/ja-jp/azure/route-server/expressroute-vpn-support


# アーキテクチャ
最終的に疎通確認ができたアーキテクチャはこちらです(ボツになった案は Appendix として載せておきます)。ハブ VNet 上の`subnet-001`に立てた NVA から無理やりオンプレミスとブランチの経路を NEXT_HOP 属性を付与したうえで広報しようとしたのですが、それでは実現できませんでした。
![](/images/20231212-branch2branch-azfw/arch-success.png)

## 疎通確認
上記のようなアーキテクチャにおいて、以下のように疎通確認が取れています。オンプレミスからブランチの VM に対して疎通確認を取っています。Azure Firewall のネットワークルールでは全許可としています。traceroute の結果としても AzureFirewallSubnet を経由していることがうかがえます。

```
$ nc -v 10.50.1.4 22
Connection to 10.50.1.4 22 port [tcp/ssh] succeeded!

$ ssh AzureAdmin@10.50.1.4
The authenticity of host '10.50.1.4 (10.50.1.4)' can't be established.
ECDSA key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Are you sure you want to continue connecting (yes/no/[fingerprint])?

$ traceroute 10.50.1.4
traceroute to 10.50.1.4 (10.50.1.4), 30 hops max, 60 byte packets
 1  10.20.10.133 (10.20.10.133)  2.637 ms  2.585 ms  2.545 ms
 2  * * *
 3  10.0.3.7 (10.0.3.7)  5.255 ms  4.781 ms  4.745 ms
 4  * * 10.50.1.4 (10.50.1.4)  309.329 ms

```

# アーキテクチャ上のポイント

## GatewaySubnet に対するルートテーブルのアタッチ
結局のところここが重要で、GatewaySubnet に対してオンプレミス・ブランチそれぞれの宛先に対しては Azure Firewall を経由させるという UDR を含んだルートテーブルをアタッチすることにより実現できました。ここで、慣れている方なら疑問に思うかもしれません。例えば、`オンプレミス -> ER Gateway -> Azure Firewall -> VPN Gateway` と渡ってきたパケットがルートテーブルによって再度 Azure Firewall にループしてしまうんじゃないかという点ですね。ただこれは検証結果からもわかる通り、ループしません。これは、VPN Gateway から出ていくパケットは VPN を利用するためにカプセル化されており、宛先が UDR で記述したアドレス範囲に該当しないためです。直感に反するのでかなり混乱しました。

## ルートテーブルで制御するなら ARS は不要なのか？
ARS においてブランチ間接続を Disabled にして確認をしたところ、疎通ができなくなりました。結果だけ見ればもちろん ARS が必要という結論になるだけなのですが、こちらの記事[^2]で細かい解説をしてくれています。結局のところ、ExpressRoute Gateway と VPN Gateway の間で直接の経路交換がなされないため、ARS が必要です。オンプレミス側にそもそもブランチの経路が流れてこなくなります。
[^2]:https://zenn.dev/skmkzyk/articles/udr-is-not-effective-for-os

## ブランチが Azure VNet の場合は VNet-to-VNet の VPN を利用する
ARS を利用する際の制約として、BGP ピアとなる VPN Gateway の ASN は 65515 にする必要があります。一方で、Local Network Gateway は ASN に 65515 を設定することができないため、所謂一般的な S2S(IPsec) で VPN を張ることができません。本記事の構成としては VNet-to-VNet の VPN を利用したうえで BGP を有効化しています。
![](/images/20231212-branch2branch-azfw/arch-b2b-disabled.png)

# まとめ
- ARS を利用した ExpressRoute と VPN の折り返しにおいて、間に Azure Firewall を挟む構成を実現しました。
- 各種 Gateway から出るパケットのカプセル化が影響して、UDR の宛先 IP に該当しない点に起因しています。
- この構成がサポートされるかどうかは明言できないのでご容赦ください。

# Appendix
この構成は疎通できませんでした。NVA から広報するアドレスをオンプレミスやブランチから広報されてくるものより more specific にしても失敗しました。
![](/images/20231212-branch2branch-azfw/arch-failed.png)

