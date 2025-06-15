---
title: "Azure REST API で Azure リソースを変更しよう"
emoji: "🚽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","api"]
publication_name: "microsoft"
published: false
---

# Azure の管理操作 と Azure REST API
Azure の管理操作のリクエストは、Azure portal、 Azure Powershell/CLI、 REST API、SDK 経由問わず、Azure Reource Manager を経由してサービスに送られます[^1]。

Azure REST API は新サービス、新 API がすぐに登場するため、Azure CLI でカバーしていないサービスや設定を行えます（Request Body 次第）。

また、Azure REST API は Azure CLI の `az rest` コマンドから叩けるため、認証まわりや HTTP リクエストに不慣れであっても気軽に試せます。特に、Azure Cloud Shell では、基本的にログイン済みのため、かなり気軽です。

## az rest コマンド
az rest コマンドの構文は基本的に以下のようになります。

```
$ az rest --method {HTTPメソッド} --url {API-URL} --body '{JSON-PAYLOAD}'
```

以下 MS Learn より抜粋
> # az rest を使用するためのヒント
az rest コマンドは、ログインした資格情報を使用して自動的に認証します。
Authorization ヘッダーが設定されていない場合は、ヘッダー Authorization: Bearer <token>が追加されます。ここで、<token> は を介して Microsoft Entra IDから取得されます。
--url パラメーターが --url コマンドの出力からエンドポイントで始まる場合、トークンのターゲット リソースは az cloud show --query endpoints パラメーターから派生します。 --url パラメーターが必要です。
カスタム リソースの --resource パラメーターを使用します。
Content-Type ヘッダーが設定されておらず、--body が有効な JSON 文字列である場合、Content-Type ヘッダーは既定で "application/json" になります。

# リソースの情報取得
では実際に、GET でリソースの情報を取得してみましょう。あらかじめデプロイしていた Virtual Network を取得するリクエストを実行します。`--url` に設定するリソースの URI は Azure portal のリソース画面 [JSON View] から取得できます。
```
$ az rest --method GET --url /subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1?api-version=2024-10-01
```
レスポンスは以下のようになります。
```json
{
  "etag": "W/\"fa4496d3-f571-45d5-8846-129d7430bc91\"",
  "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1",
  "location": "japaneast",
  "name": "vnet-japaneast-1",
  "properties": {
    "addressSpace": {
      "addressPrefixes": [
        "172.17.0.0/16"
      ]
    },
    "enableDdosProtection": false,
    "privateEndpointVNetPolicies": "Disabled",
    "provisioningState": "Succeeded",
    "resourceGuid": "4a86cce3-642f-487a-a316-d443fef8cec2",
    "subnets": [
      {
        "etag": "W/\"fa4496d3-f571-45d5-8846-129d7430bc91\"",
        "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1/subnets/snet-japaneast-1",
        "name": "snet-japaneast-1",
        "properties": {
          "addressPrefixes": [
            "172.17.0.0/24"
          ],
          "delegations": [],
          "ipConfigurations": [
            {
              "id": "/subscriptions/xxxx/resourceGroups/20250611-LBTEST/providers/Microsoft.Network/networkInterfaces/VNET-JAPANEAST-1-NIC01-0382EAE9/ipConfigurations/VNET-JAPANEAST-1-NIC01-DEFAULTIPCONFIGURATION",
              "resourceGroup": "20250611-LBTEST"
            },
            {
              "id": "/subscriptions/xxxx/resourceGroups/20250611-LBTEST/providers/Microsoft.Network/networkInterfaces/VNET-JAPANEAST-1-NIC01-D4B6A4EA/ipConfigurations/VNET-JAPANEAST-1-NIC01-DEFAULTIPCONFIGURATION",
              "resourceGroup": "20250611-LBTEST"
            }
          ],
          "privateEndpointNetworkPolicies": "Disabled",
          "privateLinkServiceNetworkPolicies": "Enabled",
          "provisioningState": "Succeeded"
        },
        "resourceGroup": "20250611-lbtest",
        "type": "Microsoft.Network/virtualNetworks/subnets"
      }
    ],
    "virtualNetworkPeerings": []
  },
  "resourceGroup": "20250611-lbtest",
  "tags": {},
  "type": "Microsoft.Network/virtualNetworks"
}
```


# リソースの変更

# おわりに

[^1]:https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/overview#consistent-management-layer