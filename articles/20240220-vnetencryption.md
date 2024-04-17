---
title: "Azure Virtual Network Encryption で VM 間の内部通信を暗号化してみる"
emoji: "🔏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","network","microsoft","iaas"]
published: true
publication_name: "microsoft"
---

# はじめに

VNet Encryption が~~一部地域~~ すべての地域にて GA されました[^1]。この機能を利用すると、同じ仮想ネットワーク内の VM 間および、ピアリング越しの仮想ネットワーク間のトラフィックの暗号化を有効化することができます。現在は、UK South   、Swiss North、West Central US リージョンにて利用可能です。

[^1]:https://azure.microsoft.com/ja-jp/updates/general-availability-azure-virtual-network-encryption-availability-in-all-regions/

# 環境準備

## VNet フローログの有効化

1 つの VNet とストレージアカウントを同一の対応リージョンにデプロイし、 VNet フローログを有効化[^2] します。

[^2]:https://learn.microsoft.com/ja-jp/azure/network-watcher/vnet-flow-logs-powershell

まずは次のコマンドにて、Insights プロバイダーを登録します。Azure CloudShell から実行するのが楽です。

```powershell
# Register Microsoft.Insights provider.
Register-AzResourceProvider -ProviderNamespace Microsoft.Insights
```

つづいて次のコマンドにて、VNet フローログを作成します。この構成により、ストレージアカウント上に VNet フローログが格納されるようになります。

```powershell
# Place the virtual network configuration into a variable.
$vnet = Get-AzVirtualNetwork -Name vnet-hub-wcus -ResourceGroupName 20240220-vnet-enc-wcus
# Place the storage account configuration into a variable.
$storageAccount = Get-AzStorageAccount -Name strgflowlogwcus -ResourceGroupName 20240220-vnet-enc-wcus
# Create a VNet flow log.
New-AzNetworkWatcherFlowLog -Enabled $true -Name vnetflowlog-wcus -NetworkWatcherName NetworkWatcher_westcentralus -ResourceGroupName NetworkWatcherRG -StorageId $storageAccount.Id -TargetResourceId $vnet.Id
```

VNet フローログを取得するだけであれば、ここまでの設定で問題ありません。

とはいえ、ストレージに保存された JSON 形式のログをそのまま分析するのは煩雑であるため、Log Analytics に流し込むことでクエリを用いた分析を可能にします。

次のコマンドにて、Log Analytics ワークスペースを作成し、それを VNet フローログと対応させます。そのために、Network Watcher のトラフィック分析という仕組みを利用しています。

上書きに関する注意書きが出ると思いますが、そのまま進めます。パラメーターは適宜変更してください。

```powershell
# Place the virtual network configuration into a variable.
$vnet = Get-AzVirtualNetwork -Name vnet-hub-wcus -ResourceGroupName 20240220-vnet-enc-wcus
# Place the storage account configuration into a variable.
$storageAccount = Get-AzStorageAccount -Name strgflowlogwcus -ResourceGroupName 20240220-vnet-enc-wcus

# Create a traffic analytics workspace and place its configuration into a variable.
$workspace = New-AzOperationalInsightsWorkspace -Name law-vnet-enc-wcus -ResourceGroupName 20240220-vnet-enc-wcus -Location westcentralus

# Create a VNet flow log.
New-AzNetworkWatcherFlowLog -Enabled $true -Name vnetflowlog-wcus -NetworkWatcherName NetworkWatcher_westcentralus -ResourceGroupName NetworkWatcherRG -StorageId $storageAccount.Id -TargetResourceId $vnet.Id -EnableTrafficAnalytics -TrafficAnalyticsWorkspaceId $workspace.ResourceId -TrafficAnalyticsInterval 10
```

作成した VNet の中に、ログ取得用 VM を 2 台用意します。対応している VM シリーズが限られているため、ドキュメントに記載のシリーズ[^3] から選択します。

[^3]:https://learn.microsoft.com/ja-jp/azure/virtual-network/virtual-network-encryption-overview#requirements

しばらく待った後、VNet フローログが Log Analytics 側の `NTANetAnalytics` テーブルにインジェストされていることを確認します。

![](/images/20240220-vnetencryption/vnetenc-01.png)

現時点では、VNet にデプロイした 2 台の VM 間の通信をフィルターして取得してみると `Unencrypted` 状態となっています。片方の VM に apache をインストールして http 接続 したり、ssh 接続を試したりしています。

![](/images/20240220-vnetencryption/vnetenc-02.png)

今までの手順でリソースの一覧としては次のようになります。

![](/images/20240220-vnetencryption/vnetenc-03.png)


## VM の高速ネットワークの有効化

VNet Encryption の要件として、VM の NIC 高速ネットワーク機能が必要です。必要に応じて、VM を停止してから次のコマンドにて高速ネットワークを有効化します。 2 台の VM に対してそれぞれ適用します。

```powershell
$nic = Get-AzNetworkInterface -ResourceGroupName "20240220-vnet-enc-wcus" -Name "ubuntu-2004-001288"

$nic.EnableAcceleratedNetworking = $true

$nic | Set-AzNetworkInterface
```

## VNet Encryption の有効化

仮想ネットワークのリソースを開き、[プロパティ] > [暗号化] > [無効] となっている部分を選択します。

![](/images/20240220-vnetencryption/vnetenc-04.png)


仮想ネットワークの暗号化にチェックを入れて、保存します。

![](/images/20240220-vnetencryption/vnetenc-05.png)


また、注意書きの通り、仮想マシンの再起動が必要となるため、デプロイした VM をそれぞれ再起動します。

# ログの確認

[先ほどの場合](#vnet-フローログの有効化) と同様に、http や ssh 接続を VM 間で何回かトライし、フローログに記録されるようにします。Traffic Analytics のデータのインジェストが 10 分ごとに設定されているため (1 時間ごとにも設定可能)、10 分程度待ったうえで Log Analytics ワークスペースを再度覗いてみます。

状態が `Encrypted` としてログが記録されていることが確認できます。

![](/images/20240220-vnetencryption/vnetenc-06.png)


# おわりに

VNet 内(正確には VM 間) のトラフィックが暗号化されていることを VNet フローログベースで確認しました。ドキュメントを参照すると次の制約があることも注意が必要でしょう。PaaS の対応については GA 後も変わらないような気もしますが…。

:::message

Azure 仮想ネットワーク暗号化には、パブリック プレビューの間は次の制限があります。

- PaaS が関係するシナリオでは、PaaS がホストされている仮想マシンによって、仮想ネットワーク暗号化がサポートされているかどうかが決まります。 仮想マシンは、一覧表示されている要件を満たしている必要があります。
- 内部ロード バランサーについては、ロード バランサーの背後にあるすべての仮想マシンは、サポートされている仮想マシン SKU である必要があります。

:::