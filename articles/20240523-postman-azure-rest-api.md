---
title: "Postman を使って Azure REST API を叩く"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","postman","microsoft","api"]
published: true
publication_name: "microsoft"
---

# はじめに
Azure portal 上での GUI 操作、Azure PowerShell や Azure CLI による CLI 操作も裏側では Azure の REST API と通信することで特定の操作を実行しています。
一部の新機能については、Azure REST API を直接叩いてリソースの構成を変更する必要があるため、前述の 3 つの方法に加えて覚えておくと役に立つタイミングがあるでしょう。
特に HTTP クライアントツールとして人気のある Postman を使った場合の API の呼び出し方についてまとめておきます。

# Postman 
Postman[^1] は Web アプリもしくはクライアント アプリとして利用可能です。必要に応じてサインアップを行い使用可能な状態とします。
[^1]:https://www.postman.com/

# サービス プリンシパルの作成
Postman からの API コールのタイミングで必要となるアクセス トークンを取得するため、対象のテナントに対してアプリを登録し、サービスプリンシパルを作成します。

GUI 操作でも可能ですが、Azure CLI を使って次のようにして作成できます。

レスポンスとして得られた情報はメモしておきます。
```
$ az ad sp create-for-rbac -n sp-postman
{
  "appId": "c31971e6-8975-4eab-acef-76186edda222",
  "displayName": "sp-postman",
  "password": "<PASSWORD>",
  "tenant": "<TENANT_ID>"
}
```

作成されたサービス プリンシパルに対して、API コールに権限を付与します。例えば特定のリソース グループに対して共同作成者ロールを付与します。

割り当てにあたって、サービス プリンシパルのオブジェクト ID が必要となるため、取得しておきます。
```
$ az ad sp show --id c31971e6-8975-4eab-acef-76186edda222
~~~
"displayName": "sp-postman",
"homepage": null,
"id": "6af26483-31e4-4372-99c6-aca2a46477f9",
~~~

```
取得したオブジェクト ID に対して、事前作成済みの `postman-test` というリソース グループに対して共同作成者ロールを付与します。

```
$ az role assignment create --assignee 6af26483-31e4-4372-99c6-aca2a46477f9 --role Contributor --scope subscriptions/<SUBSCRIPTION_ID>/resourceGroups/postman-test

```

# API コール用の認証トークンの取得
Postman から `https://login.microsoftonline.com/<TENANT_ID>/oauth2/token` に対して POST リクエストを投げ、トークンを取得します。 
![](/images/20240523-postman-azure-rest-api/token.png)

次のようなレスポンスが得られます。`access_token` を控えておきましょう。

```
{
    "token_type": "Bearer",
    "expires_in": "3599",
    "ext_expires_in": "3599",
    "expires_on": "1716444420",
    "not_before": "1716440520",
    "resource": "https://management.azure.com/",
    "access_token": "eyJ0eXAiOiJK~~~~"
}
```

# Postman から Azure REST API を呼んでみる

## リソース グループの情報取得 (GET)
`postman-test` リソース グループの情報を GET で取得してみます。
ヘッダーに次を設定してリクエストを送信します。
- `Content-Type`:`application/json`
- `Authorization`:`Bearer eyJ0eXAiOiJKV1QiLCJhb~~~~~`

![](/images/20240523-postman-azure-rest-api/GET-RG.png)

次のようなレスポンスが得られます。
```
{
    "id": "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/postman-test",
    "name": "postman-test",
    "type": "Microsoft.Resources/resourceGroups",
    "location": "eastus",
    "tags": {},
    "properties": {
        "provisioningState": "Succeeded"
    }
}
```

## リソースの作成 (PUT)
ストレージ アカウントを API 経由で作成してみます。細かい点はこちらのドキュメント[^2] をご確認ください。
[^2]:https://learn.microsoft.com/ja-jp/rest/api/storagerp/storage-sample-create-account


リクエスト Header には [GETの場合](#リソース-グループの情報取得-get) と同様のものを設定します。
リクエスト Body には次のようなものを含めます。
```json
{
  "sku": {
    "name": "Standard_LRS"
  },
  "kind": "StorageV2",
  "location": "japaneast"
}
```
![](/images/20240523-postman-azure-rest-api/putreq.png)

Azure portal を見ると、リソースが作成されていることが確認できます。
![](/images/20240523-postman-azure-rest-api/portal-strg.png)

# おわりに
Bearer トークンを用意して HTTP クライアントから Azure REST API を叩く一連の流れを確認しました。
この記事をまとめた背景としては、Log Analytics のレプリケーション機能[^3] の有効化が直 API しか対応していなかったことにあります。
このように新機能については、API 利用が必須の可能性がありますので、Postman に限らず利用できるようにしておくのが良いでしょう。
[^3]:https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/workspace-replication