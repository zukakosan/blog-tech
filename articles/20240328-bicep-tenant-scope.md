---
title: "Bicep を targetScope = 'tenant' でデプロイする"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","bicep","iac","microsoft"]
published: true
publication_name: "microsoft"
---
:::message
これは個人の検証であり、サポートの可否を保証するものではありません。
:::

# はじめに
Bicep を利用して環境構築を行う際、多くの場合はデフォルトのスコープであるリソースグループ単位でデプロイすることが多いのではないでしょうか。Azure 上で扱うリソースには必ずしもリソースグループの中にデプロイするものだけではありません。例えば、リソースグループを作成したい場合はサブスクリプションのスコープで定義する必要があります。

サブスクリプションスコープや管理グループスコープと比較して、テナントスコープの扱いが若干難易度が高かったため記録しておきます。

# スコープの設定
Bicep ファイルのスコープをテナントに設定するには次のように設定します。

```bicep
targetScope = 'tenant'
```

テナントスコープで作成するリソース[^1] の一例として、管理グループの定義[^2] は次のように記述できます。

[^1]:https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/deploy-to-tenant?tabs=azure-cli#supported-resources

[^2]:https://learn.microsoft.com/en-us/azure/templates/microsoft.management/managementgroups?pivots=deployment-language-bicep

```bicep:main.bicep
targetScope = 'tenant'

@description('The name of the main group')
param mainManagementGroupName string = 'mg-bicep'

@description('The display name for the main group')
param mainMangementGroupDisplayName string = 'Bicep Management Group'
 
resource mgmtGpBicep 'Microsoft.Management/managementGroups@2020-02-01' = {
  name: mainManagementGroupName
  scope: tenant()
  properties: {
    displayName: mainMangementGroupDisplayName
  }
}
```

ここまでは定義を素直に書くだけです。テナントスコープに対するデプロイには次のコマンド `az deployment tenant create`[^3] が利用できますが、そのままデプロイすると権限がないぞと怒られます。

[^3]:https://learn.microsoft.com/ja-jp/cli/azure/deployment/tenant?view=azure-cli-latest#az-deployment-tenant-create

```
$ az deployment tenant validate -l japaneast --template-file ./20240328-deploy-mg/main.bicep

A new Bicep release is available: v0.26.54. Upgrade now by running "az bicep upgrade".
{"code": "AuthorizationFailed", "message": "The client 'xxxx@xxxx.com' with object id '187cd1e5-91be-4fd5-98db-8eea62f64221' does not have authorization to perform action 'Microsoft.Resources/deployments/validate/action' over scope '/providers/Microsoft.Resources/deployments/main' or the scope is invalid. If access was recently granted, please refresh your credentials."}
```

# 権限設定
テナントスコープへのデプロイには追加で手順が必要です。基本的に Azure RBAC ロールは少なくとも Root Management Group 配下のどこかのレイヤーで設定されるものですが、テナントスコープのデプロイにはその 1 つ上の Root レベルでの権限が必要になります。

![](/images/20240328-bicep-tenant-scope/elevate-access.png)

Root の範囲で権限を設定するには、Microsoft Entra ID ロールのグローバル管理者の権限を昇格させて操作する必要があります。全体管理者として Azure portal にログインし、[Microsoft Entra ID] > [Properties] > [Access management for Azure resources] でトグルを `Yes` に変更します。
![](/images/20240328-bicep-tenant-scope/elevate-setting.png)

こちらを有効化すると、Azure RBAC の `/` スコープでユーザーアクセス管理者ロールが割り当てられます。つまり `/` に対して権限設定ができるようになります。その後、次のコマンドを実行し、Bicep ファイルをデプロイするユーザーに対して `/` スコープでの `Owner` または `Contributer` を付与します( `Microsoft.Resources/deployments/validate/action` が含まれていれば問題ないです)。 

```bash
$ az role assignment create --assignee "[userId]" --scope "/" --role "Owner"
```

:::message

- 次のようにすると、ログインしているユーザーの ID を拾ってきてロールをアサインできるので楽です。

```bash
$ az role assignment create --assignee $(az ad signed-in-user show --query id --output tsv) --scope "/" --role "Owner"
```

- ローカル環境でうまくいかない場合は Azure Cloud Shell を利用すると確実です。

:::

# Bicep ファイルのデプロイ
権限の設定はできたので、VSCode から再度デプロイしてみると成功しました。再度エラーが出る場合は、`az login` コマンドにて再度ログインしなおすことでクレデンシャルが更新されて権限が付与されます。

```
$ az deployment tenant create -n tenantDeploy --location japaneast --template-file ./20240328-deploy-mg/main.bicep 
A new Bicep release is available: v0.26.54. Upgrade now by running "az bicep upgrade".
{
  "id": "/providers/Microsoft.Resources/deployments/tenantDeploy",
  "location": "japaneast",
  "name": "tenantDeploy",
  "properties": {
    "correlationId": "f115e0a6-f6e7-47b8-9f27-1ef7b41cc76e",
    "debugSetting": null,
    "dependencies": [],
    "duration": "PT27.7986402S",
    "error": null,
    "mode": "Incremental",
    "onErrorDeployment": null,
    "outputResources": [
      {
        "id": "/providers/Microsoft.Management/managementGroups/mg-bicep"
      }
    ],
    "outputs": null,
    "parameters": {
      "mainManagementGroupName": {
        "type": "String",
        "value": "mg-bicep"
      },
      "mainMangementGroupDisplayName": {
        "type": "String",
        "value": "Bicep Management Group"
      }
    },
    "parametersLink": null,
    "providers": [
      {
        "id": null,
        "namespace": "Microsoft.Management",
        "providerAuthorizationConsentState": null,
        "registrationPolicy": null,
        "registrationState": null,
        "resourceTypes": [
          {
            "aliases": null,
            "apiProfiles": null,
            "apiVersions": null,
            "capabilities": null,
            "defaultApiVersion": null,
            "locationMappings": null,
            "locations": [
              null
            ],
            "properties": null,
            "resourceType": "managementGroups",
            "zoneMappings": null
          }
        ]
      }
    ],
    "provisioningState": "Succeeded",
    "templateHash": "8582005334857032986",
    "templateLink": null,
    "timestamp": "2024-03-28T07:02:50.992397+00:00",
    "validatedResources": null
  },
  "tags": null,
  "type": "Microsoft.Resources/deployments"
}
```

Azure portal を確認すると、確かに管理グループが作成されています。
![](/images/20240328-bicep-tenant-scope/mg.png)

# おわりに
スコープをテナントに設定した Bicep ファイルのデプロイを試しました。管理グループ以外にも、サブスクリプションの作成や、課金の管理までできるようです(意外と何でもできる)。その際も同様に権限設定が必要になるのでご注意ください。また、作業用に昇格した権限や、`/` で付与した権限は作業後に取り上げるなどの運用も大事です。