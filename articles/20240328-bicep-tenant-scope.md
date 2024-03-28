---
title: "Bicep を targetScope = 'tenant' でデプロイする"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

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
  properties: {
    displayName: mainMangementGroupDisplayName
  }
}
```

ここまでは定義を素直に書くだけです。テナントスコープに対するデプロイには次のコマンド `az deployment tenant create`[^3] が利用できますが、そのままデプロイすると怒られます。

[^3]:https://learn.microsoft.com/ja-jp/cli/azure/deployment/tenant?view=azure-cli-latest#az-deployment-tenant-create

```
$ az deployment tenant validate -l japaneast --template-file ./20240328-deploy-mg/main.bicep

A new Bicep release is available: v0.26.54. Upgrade now by running "az bicep upgrade".
{"code": "AuthorizationFailed", "message": "The client 'xxxx@xxxx.com' with object id '187cd1e5-91be-4fd5-98db-8eea62f64221' does not have authorization to perform action 'Microsoft.Resources/deployments/validate/action' over scope '/providers/Microsoft.Resources/deployments/main' or the scope is invalid. If access was recently granted, please refresh your credentials."}
```




