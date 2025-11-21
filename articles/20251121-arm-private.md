---
title: "Azure Resource Manager を閉域化して VM から閉域でログインする"
emoji: "🙋🏻‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","network","vm"]
published: false
---

# Azure Resource Manager の閉域化検証

## 検証目的
閉域化された VM から `az login --identity` (Managed Identity) を使用して Azure にログインする際の、Private Endpoint の必要性と動作を確認する

## 前提条件
- 検証用 VM (Managed Identity 有効化済み)
- 閉域化された VNet/Subnet
- NSG ですべてのアウトバウンドをデフォルト拒否

---

## 検証ステップ
az login の結果を参照するには、何等かの Managed Identity が何らかの権限を持っている必要があるため、あらかじめリソースグループに対する Contributor 権限を付与

### Phase 0: 既定の状態確認 (NSG すべて許可)
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

### Phase 1: ベースライン確認(すべて拒否状態)
**目的**: 何も許可していない状態で失敗することを確認

1. NSG ですべてのアウトバウンドを拒否
2. VM にログイン (Azure Bastion 経由)
3. `az login --identity` を実行
4. **期待結果**: エラーで失敗
結果：応答なし
```
AzureAdmin@vm-mid-azlogin:~$ az login --identity
^C
```
---

### Phase 2: Service Tag でのアウトバウンド許可
**目的**: Private Endpoint なしで、Service Tag だけで動作するか確認

#### 2-1. AzureActiveDirectory のみ許可
1. NSG で以下を許可:
   - Service Tag: `AzureActiveDirectory` (送信先)
   - ポート: 443
   - プロトコル: TCP
2. `az login --identity` を実行
3. **期待結果**: ARM にアクセスできないため失敗
結果：応答なし
```
AzureAdmin@vm-mid-azlogin:~$ az login --identity
^C
```
#### 2-2. AzureActiveDirectory + AzureResourceManager 許可
1. NSG で追加許可:
   - Service Tag: `AzureResourceManager` (送信先)
   - ポート: 443
   - プロトコル: TCP
2. `az login --identity` を実行
3. **結果**: ✅ 成功
4. 追加確認:
   ```powershell
   az account show
   az group list
   az vm list
   ```

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

<!-- #### 2-3. ネットワークトラフィックの確認
1. NSG Flow Logs または Azure Network Watcher で通信先を確認
2. 実際にアクセスしている IP アドレス/FQDN を記録:
   - `login.microsoftonline.com` (Entra ID)
   - `management.azure.com` (ARM)

--- -->

### Phase 3: ARM Private Endpoint 化
**目的**: ARM を Private Endpoint 化した場合の動作確認

#### 3-1. Private Endpoint の作成
こちらのドキュメントに従い以下の手順を実施

1. ARM 用の Private Endpoint を作成:
   - リソースタイプ: `Microsoft.ResourceGraph/resources` または ARM エンドポイント
   - サブリソース: 該当するもの
   - ターゲット VNet: VM と同じ VNet (または Peering された VNet)

2. Private DNS Zone の構成:
   - Zone 名: `privatelink.azure.com`
   - A レコード: Private Endpoint の IP にマッピング
   - VNet Link: VM の VNet にリンク

#### 3-2. DNS 解決の確認
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

#### 3-3. NSG ルールの調整
1. `AzureResourceManager` Service Tag を削除 (または優先度を下げる)
2. VNet 内通信(Private Endpoint 宛)を許可

#### 3-4. 動作確認
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
<!-- 
### Phase 4: 完全閉域化(オプション)
**目的**: Entra ID も Private Link 化できるか確認

#### 4-1. Entra ID Private Link の構成
1. Microsoft Entra Private Link を構成 (Premium P1/P2 必要)
2. Private Endpoint を作成
3. Private DNS Zone を構成:
   - `privatelink.aadg.windows.net.nsatc.net`
   - その他必要な Zone

#### 4-2. 動作確認
1. すべての Service Tag を削除
2. VNet 内通信のみ許可
3. `az login --identity` を実行
4. **期待結果**: ✅ 成功 (完全閉域) -->

<!-- ---

## 検証ポイント/注意事項

### トラブルシューティング
- DNS 解決が正しく動作しているか確認 (`nslookup`, `Resolve-DnsName`)
- NSG Flow Logs で通信がブロックされていないか確認
- Managed Identity のロールアサインメントが適切か確認
- Private DNS Zone が VNet に正しくリンクされているか確認

### 確認すべきログ
- NSG Flow Logs
- Azure Monitor (VM Insights)
- Private Endpoint のメトリクス
- DNS クエリログ

### コスト見積もり
- Private Endpoint: ~$7-10/月/エンドポイント
- NSG Flow Logs: データ量に応じた課金
- Private DNS Zone: ~$0.50/月/ゾーン

--- -->

## おわりに
- **Phase 2-2**: Service Tag による制限的なアウトバウンド許可で十分動作する
- **Phase 3**: ARM を Private Endpoint 化しても動作するが、追加のコストと複雑性が発生
<!-- - **Phase 4**: 完全閉域化は可能だが、Entra ID Premium ライセンスと高度な構成が必要 -->

**推奨**: 多くの場合、Phase 2-2 (Service Tag によるアウトバウンド制御) が最もコストパフォーマンスが高い