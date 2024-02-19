---
title: "Azure Firewall のフロー トレースログでトラブルシューティングがどう変わるか"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","firewall","microsoft","network"]
published: true
publication_name: "microsoft"
---

# はじめに

Azure Firewall のフロー トレースログが昨日 GA しました[^1]。この機能を利用することで、Azure Firewall 側で `SYN-ACK`、`FIN` 、`FIN-ACK` 、`RST`、`INVALID` などの追加の TCP ハンドシェイク ログを示すログが表示されます。これは、パケットのドロップや非対称ルートを特定するのに特に役立ちます。`SYN` や `ACK` が確認できないのがなぜなのかはわかりませんが、トラブルシューティングのタイミングで参照できる情報として多少使えそうです。

[^1]:https://azure.microsoft.com/ja-jp/updates/azure-firewall-flow-trace-logs-and-autoscaling-based-on-number-of-connections-and-are-now-generally-available/

# 検証環境の構成

本検証では、以下のような環境を構成します。Spoke-hub-spoke および spoke-hub-hub-spoke で疎通が取れる環境となっています。

各スポークに配置した VM は Ubuntu 20.04 を利用しています。

Azure Firewall の規則としては、東西方向の通信のみを許可している状況です。 

![arch-flowlog-azfw.png](/images/20240219-azfw-flowtracelog/arch-flowlog-azfw.png)

## ログ取得の設定

こちらのドキュメント[^2] に記載の手順に従って、フロートレースログの設定をしていきます。GA と発表されているものの、ドキュメント上はプレビューとなっているため、念のため手順はドキュメントに倣います。

[^2]:https://learn.microsoft.com/ja-jp/azure/firewall/enable-top-ten-and-flow-trace#flow-trace

実行するコマンドとしては以下のようなものになります。

```powershell
Connect-AzAccount 
Select-AzSubscription -Subscription <subscription_id> or <subscription_name>
Register-AzProviderFeature -FeatureName AFWEnableTcpConnectionLogging -ProviderNamespace Microsoft.Network
Register-AzResourceProvider -ProviderNamespace Microsoft.Network
```

Azure Firewall の診断設定にて、`Azure Firewall Flow Trace Log` をチェックして保存します。Azure Firewall を経由する SSH 接続等の TCP コネクションを適当に張ったり切ったりしておきます。

![](/images/20240219-azfw-flowtracelog/azfw-diag.png)

:::message

ログが収集されるようになるまでしばらく待ちます。結構待ちます。未読スルーの LINE を待つ気持ちで待ちましょう。

:::

## ログの確認

しばらく待ったあと、Azure Firewall の [ログ] を選択し、`AZFWFlowTrace` というテーブル名を入力して[実行]をクリックします。いろいろと試していた SSH 接続に関するログの存在を確認できます。Spoke-Hub-Spoke でも、Spoke-Hub-Hub-Spoke でもログは確認できています。

`Flag` カラムにハンドシェイク ログの種類を示すフラグが格納され、トラブルシューティングに利用できます。

![](/images/20240219-azfw-flowtracelog/azfw-log-table.png)

SSH のコネクションでいえば確率から終了までこのように確認できますね。

![](/images/20240219-azfw-flowtracelog/azfw-log-table-02.png)

# トラブルシューティングへの活用

## 非対称ルーティングを構成してみる

当初のアーキテクチャにおいて、vnet-hub-jpe に VM を立て、以下のように Spoke 側のルートテーブルに UDR を追加し、あえて非対称ルーティングの構成にしてみます( VM を立てたサブネットに対する Azure Firewall のネットワークルールも追加しています)。

![assymetric-route.png](/images/20240219-azfw-flowtracelog/assymetric-route.png)

フロー トレースログ上は `INVALID` として記録されました。パケットを識別できないか、状態がないことを示しています。

![](/images/20240219-azfw-flowtracelog/azfw-log-invalid.png)

## 非対称ルーティングの解消

以下のように非対称ルーティングを解消してみます。

![resolve-assymetric.png](/images/20240219-azfw-flowtracelog/resolve-assymetric.png)

すると、`SYN-ACK` のログが記録されました。このようにして、多少トラブルシューティングに活かせるわけですね。

![](/images/20240219-azfw-flowtracelog/azfw-log-resolve.png)

# おわりに

ドキュメントにも記載がありますが、定常的にこのログを収集する場合、Azure Firewall 経由の TCP コネクションが発生するたびにログが記録されるためコストにも影響します。

診断目的で一時的に収集するようなログということですね。

また、`SYN-ACK` は取得できても `SYN` は記録されないので、ハンドシェイクのどこで詰まっているのかまでは判別できないという点も少し発展途上感がありますね。