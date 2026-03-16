---
title: "Azure Resource Manager を閉域化して VM から閉域で az login する"
emoji: "🙋🏻‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","network","vm"]
published: true
published_at: 2025-12-03 08:00
publication_name: "microsoft"
---
:::message
本記事は筆者個人の検証結果に基づくものであり、所属する組織の公式見解を示すものではありません。正確な情報や最新の仕様については、公式ドキュメントをご確認ください。
:::

# はじめに
閉域化された環境内の VM から Azure リソースを管理したいことがあります。Azure Resource Manager には実は、Private Endpoint が提供されており、リソースの管理アクセスを閉域化できます。通常の NSG によるアウトバウンド接続のフィルタリングによる制御も合わせて検証し、動作確認します。

# 検証
閉域化された VM から `az login --identity` (Managed Identity) を使用して Azure にログインする際の、Private Endpoint の必要性と動作を確認します。

## 用意するもの
- 検証用 VM (Managed Identity 有効化済み)
- VM の Managed Identity に対する権限（ここでは、VM の所属するリソースグループへの Contributor を付与）
- NSG

## Phase 0: 既定の状態確認 (NSG すべて許可)
1. NSG の既定状態でアタッチ
2. VM にログイン (Azure Bastion 経由)
3. `az login --identity` を実行
4. **結果**: ログイン成功
結果：ログイン成功
```
AzureAdmin@vm-mid-azlogin:~$ az login --identity
[
  {
    "environmentName": "AzureCloud",
    "homeTenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
    "id": "b957570e-6156-44f9-b1e5-xxxxxxxxxxxxx",
    "isDefault": true,
    "managedByTenants": [
      {
        "tenantId": "72f988bf-86f1-41af-91ab-xxxxxxxxxxxxx"
      },
      {
        "tenantId": "2f4a9838-26b7-47ee-be60-xxxxxxxxxxxxx"
      }
    ],
    "name": "KEDAMA-APAC-SDBX-001",
    "state": "Enabled",
    "tenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
    "user": {
      "assignedIdentityInfo": "MSI",
      "name": "systemAssignedIdentity",
      "type": "servicePrincipal"
    }
  }
]
```

## Phase 1: ベースライン確認(すべて拒否状態)
**目的**: 何も許可していない状態で失敗することを確認

1. NSG ですべてのアウトバウンドを拒否
2. VM にログイン (Azure Bastion 経由)
3. `az login --identity` を実行
4. **結果**: 応答なし
```
AzureAdmin@vm-mid-azlogin:~$ az login --identity
^C
```
---

## Phase 2: Service Tag でのアウトバウンド許可
**目的**: Private Endpoint なしで、Service Tag だけで動作するか確認

### 2-1. AzureActiveDirectory のみ許可
1. NSG で以下を許可:
   - Service Tag: `AzureActiveDirectory` (送信先)
   - ポート: 443
   - プロトコル: TCP
2. `az login --identity` を実行
3. **結果**:応答なし
```
AzureAdmin@vm-mid-azlogin:~$ az login --identity
^C
```
### 2-2. AzureActiveDirectory + AzureResourceManager 許可
1. NSG で追加許可:
   - Service Tag: `AzureResourceManager` (送信先)
   - ポート: 443
   - プロトコル: TCP
2. `az login --identity` を実行
3. **結果**: ✅ 成功
```
AzureAdmin@vm-mid-azlogin:~$ az login --identity
[
  {
    "environmentName": "AzureCloud",
    "homeTenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
    "id": "b957570e-6156-44f9-b1e5-xxxxxxxxxxxxx",
    "isDefault": true,
    "managedByTenants": [
      {
        "tenantId": "72f988bf-86f1-41af-91ab-xxxxxxxxxxxxx"
      },
      {
        "tenantId": "2f4a9838-26b7-47ee-be60-xxxxxxxxxxxxx"
      }
    ],
    "name": "KEDAMA-APAC-SDBX-001",
    "state": "Enabled",
    "tenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
    "user": {
      "assignedIdentityInfo": "MSI",
      "name": "systemAssignedIdentity",
      "type": "servicePrincipal"
    }
  }
]
AzureAdmin@vm-mid-azlogin:~$ ^C
AzureAdmin@vm-mid-azlogin:~$ az account show
{
  "environmentName": "AzureCloud",
  "homeTenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
  "id": "b957570e-6156-44f9-b1e5-xxxxxxxxxxxxx",
  "isDefault": true,
  "managedByTenants": [
    {
      "tenantId": "72f988bf-86f1-41af-91ab-xxxxxxxxxxxxx"
    },
    {
      "tenantId": "2f4a9838-26b7-47ee-be60-xxxxxxxxxxxxx"
    }
  ],
  "name": "KEDAMA-APAC-SDBX-001",
  "state": "Enabled",
  "tenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
  "user": {
    "assignedIdentityInfo": "MSI",
    "name": "systemAssignedIdentity",
    "type": "servicePrincipal"
  }
}
AzureAdmin@vm-mid-azlogin:~$ az group list
[
  {
    "id": "/subscriptions/b957570e-6156-44f9-b1e5-xxxxxxxxxxxxx/resourceGroups/20251121-arm-private",
    "location": "japaneast",
    "managedBy": null,
    "name": "20251121-arm-private",
    "properties": {
      "provisioningState": "Succeeded"
    },
    "tags": {},
    "type": "Microsoft.Resources/resourceGroups"
  }
]
```

## Phase 3: ARM Private Endpoint 化
**目的**: Private Endpoint 経由での動作確認

### 3-1. Private Endpoint の作成
こちらのドキュメントに従い、ARM の Private Endpoint と DNS を構成します。

https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/create-private-link-access-portal

1. ARM 用の Private Endpoint を作成:
   - リソースタイプ: `Microsoft.ResourceGraph/resources` または ARM エンドポイント
   - サブリソース: 該当するもの
   - ターゲット VNet: VM と同じ VNet (または Peering された VNet)

2. Private DNS Zone の構成:
   - Zone 名: `privatelink.azure.com`
   - A レコード: Private Endpoint の IP にマッピング
   - VNet Link: VM の VNet にリンク

### 3-2. DNS 解決の確認
VM から以下を実行:
```powershell
nslookup management.azure.com
```
**結果**: Private Endpoint の Private IP が返される 
```
AzureAdmin@vm-mid-azlogin:~$ nslookup management.azure.com
Server:127.0.0.53
Address:127.0.0.53#53

Non-authoritative answer:
management.azure.comcanonical name = management.privatelink.azure.com.
Name:management.privatelink.azure.com
Address: 172.18.0.5
```

### 3-3. NSG ルールの調整
1. `AzureResourceManager` Service Tag を削除 (または優先度を下げる)
2. VNet 内通信(Private Endpoint 宛)を許可

### 3-4. 動作確認
1. `az login --identity` を実行
2. **結果**: ✅ 成功 (Private Endpoint 経由)

```
AzureAdmin@vm-mid-azlogin:~$ az login --identity
[
  {
    "environmentName": "AzureCloud",
    "homeTenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
    "id": "b957570e-6156-44f9-b1e5-xxxxxxxxxxxxx",
    "isDefault": true,
    "managedByTenants": [
      {
        "tenantId": "72f988bf-86f1-41af-91ab-xxxxxxxxxxxxx"
      },
      {
        "tenantId": "2f4a9838-26b7-47ee-be60-xxxxxxxxxxxxx"
      }
    ],
    "name": "KEDAMA-APAC-SDBX-001",
    "state": "Enabled",
    "tenantId": "5dcb0955-58b3-4f7f-9faf-xxxxxxxxxxxxx",
    "user": {
      "assignedIdentityInfo": "MSI",
      "name": "systemAssignedIdentity",
      "type": "servicePrincipal"
    }
  }
]
AzureAdmin@vm-mid-azlogin:~$    az resource list --query "[].{Name:name, Type:type}" --output table
Name                                                      Type
--------------------------------------------------------  ------------------------------------------------------
vm-mid-azlogin-ip                                         Microsoft.Network/publicIPAddresses
vnet-japaneast-1                                          Microsoft.Network/virtualNetworks
vm-mid-azlogin894                                         Microsoft.Network/networkInterfaces
vm-mid-azlogin                                            Microsoft.Compute/virtualMachines
vm-mid-azlogin_OsDisk_1_4cd369441b5f4e41bc6b098744f9f38b  Microsoft.Compute/disks
nsg-arm-cli                                               Microsoft.Network/networkSecurityGroups
vnet-japaneast-1-bastion                                  Microsoft.Network/bastionHosts
vm-mid-azlogin/MDE.Linux                                  Microsoft.Compute/virtualMachines/extensions
flowlogarmprivate                                         Microsoft.Storage/storageAccounts
root-mg-rmpl                                              Microsoft.Authorization/resourceManagementPrivateLinks
pe-arm-pvt                                                Microsoft.Network/privateEndpoints
pe-arm-pvt-nic                                            Microsoft.Network/networkInterfaces
privatelink.azure.com                                     Microsoft.Network/privateDnsZones
privatelink.azure.com/q7khdwwl2fsna                       Microsoft.Network/privateDnsZones/virtualNetworkLinks
```
# おわりに
個人的に盲点だったため、ARM への Private Endpont を試しました。Service Tag による制限的なアウトバウンド許可でも、ARM の Private Endpoint 化でも動作することを確認しました。ARM を閉域化する場合、Entra ID も閉域化したくなってくるはずで、そこは現状直接の閉域化ができないため、完全閉域はまだ先の話ですね。多くの場合、NSG や FW による制限的なアウトバウンド制御で成立するかと思います。