---
title: "Azure REST API で Azure リソースを変更しよう"
emoji: "🚽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","api"]
publication_name: "microsoft"
published: true
---
:::message
本記事は筆者個人の検証結果に基づくものであり、所属する組織の公式見解を示すものではありません。正確な情報や最新の仕様については、公式ドキュメントをご確認ください。
:::

# Azure の管理操作 と Azure REST API
Azure の管理操作のリクエストは、Azure portal、 Azure Powershell/CLI、 REST API、SDK 経由問わず、Azure Reource Manager を経由してサービスに送られます[^1]。

Azure REST API は新サービス、新 API がすぐに登場するため、Azure CLI でカバーしていないサービスや設定を行えます（Request Body 次第）。

また、Azure REST API[^2] は Azure CLI の `az rest` コマンドから叩けるため、認証まわりや HTTP リクエストに不慣れであっても気軽に試せます。特に、Azure Cloud Shell では、基本的にログイン済みのため、かなり気軽です。

## az rest コマンド
az rest コマンドの構文は基本的に以下のようになります。

```bash
$ az rest --method {HTTPメソッド} --url {API-URL} --body '{JSON-PAYLOAD}'
```

以下 MS Learn より抜粋
> # az rest を使用するためのヒント
> - az rest コマンドは、ログインした資格情報を使用して自動的に認証します。
> - Authorization ヘッダーが設定されていない場合は、ヘッダー Authorization: Bearer <token>が追加されます。ここで、<token> は を介して Microsoft Entra IDから取得されます。
> - --url パラメーターが --url コマンドの出力からエンドポイントで始まる場合、トークンのターゲット リソースは az cloud show --query endpoints パラメーターから派生します。 --url パラメーターが必要です。
> - カスタム リソースの --resource パラメーターを使用します。
> - Content-Type ヘッダーが設定されておらず、--body が有効な JSON 文字列である場合、Content-Type ヘッダーは既定で "application/json" になります。

# リソースの情報取得: GET
では実際に、GET でリソースの情報を取得してみましょう。あらかじめデプロイしていた Virtual Network を取得するリクエストを実行します。`--url` に設定するリソースの URI は Azure portal のリソース画面 [JSON View] から取得できます。
```bash
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

# リソースの変更: PUT
GET メソッドで取得できた情報をもとに、リソースの変更をしてみましょう。今回は PUT メソッドを使用します。
Azure REST API では、GET レスポンスと PUT リクエストで受け付けられるプロパティが異なることがよくあります。GET はリソースの完全な情報（参照情報を含む）を返しますが、PUT では実際に更新可能なプロパティのみを受け付けます。
例えば、リソースグループ名は URL パスで既に指定されているため、Request Body で再度指定する必要はなく、API はそれを余分なプロパティと見なしています。
よって、そのまま PUT で投げ返すと、PUT API が受け付けていないプロパティも投げ込もうとしてエラーになることがあります。

```bash
$ az rest --method PUT --url /subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1?api-version=2024-10-01 --body @./azure-put-api/payload.json 
Bad Request({"error":{"code":"InvalidRequestFormat","message":"Cannot parse the request.","details":[{"code":"InvalidJson","message":"Could not find member 'resourceGroup' on object of type 'Subnet'. Path 'properties.subnets[0].resourceGroup', line 1, position 1435."}]}})
```

余分なところを削除して、必要な変更を加えます。以下では VNETのアドレス空間(`10.0.0.0/16`)の追加をしてみました。
```json
{
  "etag": "W/\"fa4496d3-f571-45d5-8846-129d7430bc91\"",
  "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1",
  "location": "japaneast",
  "name": "vnet-japaneast-1",
  "properties": {
    "addressSpace": {
      "addressPrefixes": [
        "172.17.0.0/16",
        "10.0.0.0/16"
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
        "type": "Microsoft.Network/virtualNetworks/subnets"
      }
    ],
    "virtualNetworkPeerings": []
  },
  "tags": {},
  "type": "Microsoft.Network/virtualNetworks"
}
```
上記の JSON を投げ込みます。

```bash
$ az rest --method PUT --url /subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1?api-version=2024-10-01 --body @./azure-put-api/payload.json

{
  "etag": "W/\"c0107139-5e9d-4332-b0b5-f3013b37e19a\"",
  "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1",
  "location": "japaneast",
  "name": "vnet-japaneast-1",
  "properties": {
    "addressSpace": {
      "addressPrefixes": [
        "172.17.0.0/16",
        "10.0.0.0/16"
      ]
    },
    "enableDdosProtection": false,
    "privateEndpointVNetPolicies": "Disabled",
    "provisioningState": "Updating",
    "resourceGuid": "4a86cce3-642f-487a-a316-d443fef8cec2",
    "subnets": [
      {
        "etag": "W/\"c0107139-5e9d-4332-b0b5-f3013b37e19a\"",
        "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1/subnets/snet-japaneast-1",
        "name": "snet-japaneast-1",
        "properties": {
          "addressPrefixes": [
            "172.17.0.0/24"
          ],
          "delegations": [],
          "ipConfigurations": [
            {
              "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/networkInterfaces/vnet-japaneast-1-nic01-0382eae9/ipConfigurations/vnet-japaneast-1-nic01-defaultIpConfiguration",
              "resourceGroup": "20250611-lbtest"
            },
            {
              "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/networkInterfaces/vnet-japaneast-1-nic01-d4b6a4ea/ipConfigurations/vnet-japaneast-1-nic01-defaultIpConfiguration",
              "resourceGroup": "20250611-lbtest"
            }
          ],
          "privateEndpointNetworkPolicies": "Disabled",
          "privateLinkServiceNetworkPolicies": "Enabled",
          "provisioningState": "Updating"
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
ただし、`172.17.0.0/24` のサブネットはすでにアロケーションがされているため、そこのサイズ変更などはできません。`BadRequest Error` となります。
```
Bad Request({"error":{"code":"InUsePrefixCannotBeDeleted","message":"IpPrefix 172.17.0.0/24 on Subnet snet-japaneast-1 has active allocations and cannot be deleted.","details":[]}})
```
VNET のサイズを `172.17.0.0/24` にすることはできます。
```bash
$ az rest --method PUT --url /subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1?api-version=2024-10-01 --body @./azure-put-api/payload.json 
{
  "etag": "W/\"30b5279d-d974-4c93-9aad-8dd73c9de140\"",
  "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1",
  "location": "japaneast",
  "name": "vnet-japaneast-1",
  "properties": {
    "addressSpace": {
      "addressPrefixes": [
        "172.17.0.0/24",
        "10.0.0.0/16"
      ]
    },
    "enableDdosProtection": false,
    "privateEndpointVNetPolicies": "Disabled",
    "provisioningState": "Updating",
    "resourceGuid": "4a86cce3-642f-487a-a316-d443fef8cec2",
    "subnets": [
      {
        "etag": "W/\"30b5279d-d974-4c93-9aad-8dd73c9de140\"",
        "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/virtualNetworks/vnet-japaneast-1/subnets/snet-japaneast-1",
        "name": "snet-japaneast-1",
        "properties": {
          "addressPrefixes": [
            "172.17.0.0/24"
          ],
          "delegations": [],
          "ipConfigurations": [
            {
              "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/networkInterfaces/vnet-japaneast-1-nic01-0382eae9/ipConfigurations/vnet-japaneast-1-nic01-defaultIpConfiguration",
              "resourceGroup": "20250611-lbtest"
            },
            {
              "id": "/subscriptions/xxxx/resourceGroups/20250611-lbtest/providers/Microsoft.Network/networkInterfaces/vnet-japaneast-1-nic01-d4b6a4ea/ipConfigurations/vnet-japaneast-1-nic01-defaultIpConfiguration",
              "resourceGroup": "20250611-lbtest"
            }
          ],
          "privateEndpointNetworkPolicies": "Disabled",
          "privateLinkServiceNetworkPolicies": "Enabled",
          "provisioningState": "Updating"
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

# おわりに
Azure REST API を az rest コマンドで叩いてみました。GET メソッドでのリソース情報取得から、JSON の編集、PUT メソッドでのリソース更新の流れは一部のリソースのトラブルシューティングとしても使われていたりします。直 API をたたけるため、柔軟性が高く、覚えておくと役に立つ場面が来るかもしれません。

[^1]:https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/overview#consistent-management-layer
[^2]:https://learn.microsoft.com/ja-jp/rest/api/azure/