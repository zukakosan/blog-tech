---
title: "Azure VPN Gatewayをパケットフォワーダとして動作させてみる"
emoji: "⛩️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","vpn","squid"]
published: true
publication_name: "microsoft"

---
# モチベ
- 実は、VPNGWもパケットフォワーダとして利用できるらしい
- 知らなかったので、試してみる
- 恐らく、Sopkeからのデフォルトルートの向き先をUDRでVPNGWにしておけばよいのではという想定
- SPOKE(Ubuntu)-Hub(VPNGW)-SPOKE(Ubuntu)の構成を作る

# 検証
## VNET x3 の作成
- vnet-hub: 10.0.0.0/16, GatewaySubnetを作成
![](/images/20230620-vpngw-forwarder/01.png)
- vnet-spoke-001: 10.1.0.0/16
- vnet-spoke-002: 10.2.0.0/16

## VPNGWの作成
- vnet-hub/GatewaySubnetにVPNGWを作成
- 一旦BGPは無効で作成
- もうちょっと短時間で作れてほしい

## VM x2の作成
- 各SpokeにUbuntuのVM1台ずつ立てる
- vm-spoke-001(vnet-spoke-001)
- vm-spoke-002(vnet-spoke-002)

## NSGの作成
- SSH用のNSGを作成
- VMが存在するそれぞれのサブネットに割り当て
![](/images/20230620-vpngw-forwarder/02.png)

## ピアリングの作成
- リモートゲートウェイを利用する設定を忘れない
![](/images/20230620-vpngw-forwarder/03.png)

#### 疎通テスト-1
- vm-spoke-001にSSH接続し、vm-spoke-002にping
- もちろん通るわけない

```powershell
AzureAdmin@vm-spoke-001:~$ ping 10.2.0.4
PING 10.2.0.4 (10.2.0.4) 56(84) bytes of data.
^C
--- 10.2.0.4 ping statistics ---
8 packets transmitted, 0 received, 100% packet loss, time 7152ms

```
## ルートテーブル x1の作成
- UDRを作成してデフォルトルートをVPNGWに向ける
- 今回はデフォルトルートを向けるだけなので、共通のものを使いまわす
![](/images/20230620-vpngw-forwarder/04.png)

### [ネクストホップ：仮想ネットワークゲートウェイ]の場合
- デフォルトルート用を追加
![](/images/20230620-vpngw-forwarder/05.png)
:::message alert
- この設定のみだと、SSHに必要な通信もデフォルトルートに巻き込まれてしまう
- 慌てて、自宅IP宛ての通信をInternetに向けるUDRを追加
![](/images/20230620-vpngw-forwarder/06.png)
:::

#### 疎通テスト-2
- この状態でvm-spoke-001にSSH接続し、vm-spoke-002にping
![](/images/20230620-vpngw-forwarder/07.png)

```powershell
AzureAdmin@vm-spoke-001:~$ ping 10.2.0.4
PING 10.2.0.4 (10.2.0.4) 56(84) bytes of data.
64 bytes from 10.2.0.4: icmp_seq=1 ttl=63 time=5.91 ms
64 bytes from 10.2.0.4: icmp_seq=2 ttl=63 time=3.83 ms
64 bytes from 10.2.0.4: icmp_seq=3 ttl=63 time=4.08 ms
64 bytes from 10.2.0.4: icmp_seq=4 ttl=63 time=3.90 ms
^C
--- 10.2.0.4 ping statistics ---
7 packets transmitted, 7 received, 0% packet loss, time 6007ms
rtt min/avg/max/mdev = 3.695/4.353/5.906/0.720 ms
```
- VPNGW経由で本当に疎通できた、、、！！！

### [ネクストホップ：仮想アプライアンス]の場合
- VPNGWのIPアドレスを取得する必要がある
- よって、ネクストホップ[仮想ネットワークゲートウェイ]とした状態で、tracerouteをとる
![](/images/20230620-vpngw-forwarder/08.png)

```powershell
AzureAdmin@vm-spoke-001:~$ traceroute 10.2.0.4
traceroute to 10.2.0.4 (10.2.0.4), 30 hops max, 60 byte packets
 1  10.0.1.4 (10.0.1.4)  2.899 ms * *
 2  10.2.0.4 (10.2.0.4)  5.638 ms  5.622 ms *

```
- これを見る限り、GatewaySubnetの若番で応答していそうなので、そのIPをUDRに入れる
![](/images/20230620-vpngw-forwarder/09.png)

#### 疎通テスト-3
- この状態で、再度vm-spoke-002向けにpingを投げる
- きちんと返ってきた
- ただ、このIPは変わるかもしれない？ので[仮想ネットワークゲートウェイ]で抽象化しておくのが無難そう
![](/images/20230620-vpngw-forwarder/10.png)

```powershell
AzureAdmin@vm-spoke-001:~$ ping 10.2.0.4
PING 10.2.0.4 (10.2.0.4) 56(84) bytes of data.
64 bytes from 10.2.0.4: icmp_seq=1 ttl=63 time=2.19 ms
64 bytes from 10.2.0.4: icmp_seq=2 ttl=63 time=2.36 ms
64 bytes from 10.2.0.4: icmp_seq=3 ttl=63 time=2.98 ms
64 bytes from 10.2.0.4: icmp_seq=4 ttl=63 time=2.31 ms
^C
--- 10.2.0.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 2.189/2.460/2.981/0.307 ms

```

# おわり
- VPNGW自体をパケットフォワーダとして利用するというのは意識外だった
- Azure Firewallと違ってただフォワードするだけなので、利用ケースはだいぶ限られそうな気がしている
- Azure Firewall立てる前の疎通テスト用としては使えそうな印象？