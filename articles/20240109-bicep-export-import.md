---
title: "Bicep の @export() と import で Terraform ライクにファイルを管理する"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","bicep","iac"]
published: true
publication_name: "microsoft"

---
:::message
本記事は筆者個人の検証結果に基づくものであり、所属する組織の公式見解を示すものではありません。正確な情報や最新の仕様については、公式ドキュメントをご確認ください。
:::

# はじめに

ネットサーフィンをしていたら Bicepの `@export()` と `import` という機能がプレビュー利用できるという記事[^1] を発見しました。これを利用することで Terraform っぽい変数分離ができそうなので試してみます。分離できると何が嬉しいのかというと、別のファイルで定義している変数を使いまわせるっていうことですね。Bicep では、ファイルの単位で変数が閉じられているので変数定義自体を共有することができなかったわけですね。

[^1]: https://johnlokerse.dev/2024/01/08/reusability-with-export-and-import-in-azure-bicep/

この機能の詳細についてはすでにドキュメント[^2] があったのでこちらをご参照ください。端的には、別のファイルに存在する変数をインポートできる機能になります。

[^2]: https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/bicep-import

# 試してみる

今回使用するファイルは GitHub[^3] に置いています。ファイル構造としては以下のようにしました。変数を完全に分離しています。中身は VNet と NSG しかデプロイしない簡素なものです。

[^3]: https://github.com/zukakosan/bicep-exprt-import-param

```
/
├── main.bicep
├── variables.bicep
└── bicepconfig.json
```

## プレビュー機能の有効化

ドキュメント[^4] に従い、Bicep 構成ファイルを作成しプレビュー機能を追加します。

[^4]: https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/bicep-config#create-the-config-file-in-visual-studio-code

## 変数のインポート

外部からインポート(=元ファイルからエクスポートしたい) 変数には `@export()` デコレーターで修飾します。

```bicep
@export()
var suffix = 'zukako'

@export()
var hubVnetPrefix = '10.0.0.0/16'

@export()
var subnetPrefix = '10.0.0.0/24'

@export()
type tag = 'PRD' | 'DEV' | 'QA'
```

変数をインポートする側のファイルでは、`import` 構文を使用します。ワイルドカードで`@export()` デコレーターの付いた変数をすべて持ってくることも、個別に変数を指定することも可能です。また、インポートする際に `as` を付けてエイリアス名を設定することも可能です。

```bicep
// import all variables from variables.bicep with wildcard
import * as vars from './variables.bicep'

// you can import each variable separately as well
import { location, suffix, hubVnetPrefix } from './variables.bicep'
```

インポートさえしてしまえば、あとは利用するだけです。ワイルドカードでインポートした場合は、以下のように `.` で変数を参照します。個別にインポートした場合はそのままの変数名またはエイリアス名で参照します。

```bicep
import * as vars from './variables.bicep'
location: vars.location
```

## ユーザー定義のデータ型との組み合わせ

Bicep には ユーザー定義型[^5] という型があります。これはユーザ側で変数の型をカスタマイズして作成できるものになります。

[^5]: https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/bicep/user-defined-data-types

このユーザー定義のデータ型と組み合わせることによって、デプロイ時にエンドユーザーが指定するパラメータの値をカスタムのデータ型に合わせるようにすることが可能です。分かり易い例として、タグの付与を試してみます。

`variables.bicep` にて以下のように宣言します。

```bicep
@export()
type tag = 'PRD' | 'DEV' | 'QA'
```

`main.bicep` でparamとして呼び出します。`tagchoice` はエイリアス名です。

```bicep
import { tag as tagchoice } from './variables.bicep'
param tag tagchoice
```

param で値を設定していないため、デプロイのタイミングでエンドユーザーが値を指定する必要がありますが、ユーザー定義のデータ型に沿わない場合は Validation Error とすることができます。

```bash
$ az deployment group create -g 20240109-exp --template-file main.bicep 
WARNING: The following experimental Bicep features have been enabled: Extensibility, Compile-time imports. Experimental features should be enabled for testing purposes only, as there are no guarantees about the quality or stability of these features. Do not enable these settings for any production usage, or your production environment may be subject to breaking.

Please provide string value for 'tag' (? for help): aa
{"code": "InvalidTemplate", "message": "Deployment template validation failed: 'The provided value for the template parameter 'tag' is not valid. The value 'aa' is not part of the allowed value(s): 'DEV,PRD,QA'.'.", "additionalInfo": [{"type": "TemplateViolation", "info": {"lineNumber": 19, "linePosition": 24, "path": "properties.template.definitions.tagchoice.allowedValues"}}]}
```

適切な値(今回は `PRD`, `DEV`, `QA` のいずれか)を指定した場合には問題なくデプロイできます。

```bash
$ az deployment group create -g 20240109-exp --template-file main.bicep 
WARNING: The following experimental Bicep features have been enabled: Extensibility, Compile-time imports. Experimental features should be enabled for testing purposes only, as there are no guarantees about the quality or stability of these features. Do not enable these settings for any production usage, or your production environment may be subject to breaking.

Please provide string value for 'tag' (? for help): DEV
{
  "id": "/subscriptions/xxxx/resourceGroups/20240109-exp/providers/Microsoft.Resources/deployments/main",
  "location": null,
  "name": "main",
  "properties": {
    "correlationId": "d95d8c14-2cec-49f5-aaa7-ea6a09cf7958",
    "debugSetting": null,
    "dependencies": [
      {
        "dependsOn": [
          {
            "id": "/subscriptions/xxxx/resourceGroups/20240109-exp/providers/Microsoft.Network/networkSecurityGroups/nsg-zukako",
            "resourceGroup": "20240109-exp",
            "resourceName": "nsg-zukako",
            "resourceType": "Microsoft.Network/networkSecurityGroups"
          }
        ],
        "id": "/subscriptions/xxxx/resourceGroups/20240109-exp/providers/Microsoft.Network/virtualNetworks/vnet-zukako",
        "resourceGroup": "20240109-exp",
        "resourceName": "vnet-zukako",
        "resourceType": "Microsoft.Network/virtualNetworks"
      }
    ],
    "duration": "PT5.7891062S",
    "error": null,
    "mode": "Incremental",
    "onErrorDeployment": null,
    "outputResources": [
      {
        "id": "/subscriptions/xxxx/resourceGroups/20240109-exp/providers/Microsoft.Network/networkSecurityGroups/nsg-zukako",
        "resourceGroup": "20240109-exp"
      },
      {
        "id": "/subscriptions/xxxx/resourceGroups/20240109-exp/providers/Microsoft.Network/virtualNetworks/vnet-zukako",
        "resourceGroup": "20240109-exp"
      }
    ],
    "outputs": null,
    "parameters": {
      "tag": {
        "type": "String",
        "value": "DEV"
      }
    },
    "parametersLink": null,
    "providers": [
      {
        "id": null,
        "namespace": "Microsoft.Network",
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
              "japaneast"
            ],
            "properties": null,
            "resourceType": "networkSecurityGroups",
            "zoneMappings": null
          },
          {
            "aliases": null,
            "apiProfiles": null,
            "apiVersions": null,
            "capabilities": null,
            "defaultApiVersion": null,
            "locationMappings": null,
            "locations": [
              "japaneast"
            ],
            "properties": null,
            "resourceType": "virtualNetworks",
            "zoneMappings": null
          }
        ]
      }
    ],
    "provisioningState": "Succeeded",
    "templateHash": "3469291649850883275",
    "templateLink": null,
    "timestamp": "2024-01-09T01:38:50.325641+00:00",
    "validatedResources": null
  },
  "resourceGroup": "20240109-exp",
  "tags": null,
  "type": "Microsoft.Resources/deployments"
}
```

つまり、データ型のレベルでポリシーをかけられるようなイメージですね。シフトレフト的な観点からすると、デプロイを未然に防ぐという意味で強力です。

作成されたリソースには勿論タグが付いています。
![](/images/20240109-bicep-export-import/01.png)

# まとめ

- Bicep の `@export()` デコレーターと `import` によって簡単に変数の取り出しができることを確認しました。
- 参照した記事ではユーザー定義型と組み合わせていたのですが、バリデーションがかけられるという意味で価値がありそうです。
- Terraform の慣例として変数を `variables.tf` に切り出すことが多いですが、Bicep でも同様のディレクトリ構造をとれるということを確認しました。
- 本筋のステート管理はまた別の機能差異として考える必要があります。