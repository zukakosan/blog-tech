---
title: "Azure Monitor Agent を Azure CLI で Windows VM に手動インストールする"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","monitoring","windows"]
published: true
publication_name: "microsoft"
---
:::message
これは個人の検証であり、サポートの可否を保証するものではありません。
:::

# はじめに
社内のアラートを確認していると、Azure VM に Azure Monitor Agent (AMA) をインストールせよ。という通達が来ていました。Azure Policy を適切に設定していれば自動でインストールしてくれたりもするのですが、今回は対象が Windows VM 数台だったため、動作確認も含めて AMA を手動インストールしてみました。

AMA は Azure VM 拡張機能[^1] として実装されます。インストール前の VM 拡張機能の状況を確認します。AMA はインストールされていません。

[^1]:https://learn.microsoft.com/ja-jp/azure/virtual-machines/extensions/overview

![](/images/20240105-azure-monitor-agent-install/01.png)

# AMA のインストール
手動インストールのための手順を実行していきます。

## マネージド ID の有効化
まずは、マネージド ID を有効化します。Azure VM に対して「システム割り当てマネージド ID」を付与します。VM ごとに ID が作成されるため、VM 数が大規模である場合は、「ユーザー割り当てマネージド ID」が推奨となります。一方、Azure Arc 対応サーバーの場合は、Azure Arc エージェントをインストールするとすぐにシステム割り当てマネージド ID が有効になります。

![](/images/20240105-azure-monitor-agent-install/02.png)
![](/images/20240105-azure-monitor-agent-install/03.png)


## マネージド ID に対する権限付与
ID が作成されたら、必要な権限を割り当てていきます。ドキュメント[^2] に記載の通り、組み込みロールとして、「Virtual Machine Contributor」と「Log Analytics 共同作成者」を付与します。

[^2]:https://learn.microsoft.com/ja-jp/azure/azure-monitor/agents/azure-monitor-agent-manage?tabs=azure-portal#prerequisites

![](/images/20240105-azure-monitor-agent-install/04.png)
![](/images/20240105-azure-monitor-agent-install/05.png)

## AMA インストール用コマンドの実行
適切に権限が割り当たったので、Azure Cloud Shell からコマンドを実行してAMA をインストールします。使用するコマンドはこちらです。

```bash
az vm extension set --name AzureMonitorWindowsAgent --publisher Microsoft.Azure.Monitor --ids <vm-resource-id> --enable-auto-upgrade true
```
VM のリソース ID はこちらから確認できます。
![](/images/20240105-azure-monitor-agent-install/06.png)
![](/images/20240105-azure-monitor-agent-install/07.png)

実行完了すると以下のような内容が出力されます。

```bash
{
  "autoUpgradeMinorVersion": true,
  "enableAutomaticUpgrade": true,
  "forceUpdateTag": null,
  "id": "/subscriptions/xxxxxxx/resourceGroups/20230331-imagebuilder/providers/Microsoft.Compute/virtualMachines/win11single/extensions/AzureMonitorWindowsAgent",
  "instanceView": null,
  "location": "eastus",
  "name": "AzureMonitorWindowsAgent",
  "protectedSettings": null,
  "protectedSettingsFromKeyVault": null,
  "provisioningState": "Succeeded",
  "publisher": "Microsoft.Azure.Monitor",
  "resourceGroup": "20230331-imagebuilder",
  "settings": null,
  "suppressFailures": null,
  "tags": null,
  "type": "Microsoft.Compute/virtualMachines/extensions",
  "typeHandlerVersion": "1.21",
  "typePropertiesType": "AzureMonitorWindowsAgent"
}
```

# 確認
AMA がインストールされたことを Azure portal 上で確認します。
![](/images/20240105-azure-monitor-agent-install/08.png)

# おわりに
簡単ではありますが、AMA を Azure CLI にてインストールしました。大規模環境である場合は、**システム割り当て**マネージド ID ではなく、**ユーザー割り当て**マネージド ID を用いるのが推奨です。
また、Azure Policy では、AMA のインストールを実行する**組み込みポリシー**[^3] と、データ収集ルールの割り当てまで行ってくれる**組み込みのイニシアチブ**[^4] が用意されています。ガバナンスの観点からは事前にこのような機能でガードレールを用意しておくと良いでしょう。
[^3]:https://portal.azure.com/#view/Microsoft_Azure_Policy/PolicyDetailBlade/definitionId/%2Fproviders%2FMicrosoft.Authorization%2FpolicyDefinitions%2F637125fd-7c39-4b94-bb0a-d331faf333a9
[^4]:https://portal.azure.com/#view/Microsoft_Azure_Policy/InitiativeDetailBlade/id/%2Fproviders%2FMicrosoft.Authorization%2FpolicySetDefinitions%2F0d1b56c6-6d1f-4a5d-8695-b15efbea6b49/scopes~/%5B%22%2Fsubscriptions%2Fae71ef11-a03f-4b4f-a0e6-ef144727c711%22%5D