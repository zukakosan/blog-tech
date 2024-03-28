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

```bicep:main.bicep
targetScope = 'tenant'
```

テナントスコープで作成するリソース[^1] の一例として、管理グループの定義は次のように記述できます。

[^1]:https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/deploy-to-tenant?tabs=azure-cli#supported-resources

```bicep
targetScope = 'tenant'

@description('The name of the main group')
param mainManagementGroupName string = 'mg-bicep'

@description('The display name for the main group')
param mainMangementGroupDisplayName string = 'Bicep Management Group'
 
resource mainManagementGroup 'Microsoft.Management/managementGroups@2020-02-01' = {
  name: mainManagementGroupName
  properties: {
    displayName: mainMangementGroupDisplayName
  }
}
```
