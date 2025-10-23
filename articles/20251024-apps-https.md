---
title: "Azure App Service を Key Vault 上 の証明書で HTTPS 化するための知識地図"
emoji: "🗺️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","App","HTTPS"]
published: true
publication_name: "microsoft"
---
# はじめに
Azure App Service はコードを配置するだけで Web アプリケーションを展開できる人気の PaaS です。この App Service は既定の状態では、`xxx.azurewebsites.net` という App Service の既定のドメインでホストされます。

ただ使えればいい、というだけであれば既定のドメインで HTTPS を強制してしまえば、それで終わりです。しかし、基本的には組織のドメインを使ってアプリケーションを公開するはずです。「社内から、社外から問わずカスタム ドメインを使ってアプリを公開し、HTTPS で接続させたい」という要件を Azure 上で実現します。

MSLearn に必要な情報は基本的に記載されているのですが、複数のシナリオが同じページに記載されていたり、必要な情報が複数ページに分断されていたりするので、本記事では全体的な知識地図としてまとめておきます。

# 設定の流れ
## カスタム ドメインの取得
前提としてカスタム ドメインは必須となります。Azure 上では、App Service ドメイン^[https://learn.microsoft.com/ja-jp/azure/app-service/manage-custom-dns-buy-domain] というサービスから取得することも可能です。自分が保有しているドメインがあることが前提です。

## 証明書の取得と Azure Key Vault へのインポート
ドメインを取得したら、対応する証明書を取得しましょう。証明書の管理は Azure での一貫性を考えて Azure Key Vault を使用します。DigiCert での証明書の取得の仕方および、Azure Key Vault へのインポートに関しては以下の記事にまとめています。

https://zenn.dev/microsoft/articles/20250711-digicertonazure


## Azure App Service へのカスタムドメイン設定
~~（App Service ドメインを使っていないケースが多いと思うので）~~ カスタム ドメインは使いたい FQDN を直接入力します。A レコードまたは CNAME レコードによってドメインの検証を行います。

こちらのドキュメントの「カスタム ドメイン構成」~「ドメインの検証」まで行いましょう。

https://learn.microsoft.com/ja-jp/azure/app-service/app-service-web-tutorial-custom-domain?tabs=root%2Cazurecli#configure-a-custom-domain

## カスタム ドメインと証明書のバインド
カスタムドメインに対して証明書をバインド^[https://learn.microsoft.com/ja-jp/azure/app-service/configure-ssl-bindings#add-the-binding] していきます。本構成では、証明書のストアとして Azure Key Vault を使用するため、証明書の連携に少し追加設定が必要です。何も設定していない状態でバインドしようとしても権限不足でエラーが発生するはずです。 

ここでは、`Microsoft App Service` というサービス ID（プリンシパル ID としては `abfa0a7c-a6b6-4736-8310-5855508787cd`） に対して、Azure Key Vault に対する `Key Vault Certificate User` 権限を付与します。

うまく ID が見つからない場合は、Azure Cloud Shell から以下のようなコマンドを実行しても同じことができます。

```bash
az role assignment create --role "Key Vault Certificate User" --assignee "abfa0a7c-a6b6-4736-8310-5855508787cd" --scope "/subscriptions/{subscriptionid}/resourcegroups/{resource-group-name}/providers/Microsoft.KeyVault/vaults/{key-vault-name}"
```

その後、対象の Key Vault リソースを選択して証明書をインポートします。

:::message
Azure Key Vault において Private Endpoint を構成し、パブリック アクセスを拒否している場合は閉域内に配置した作業端末等から Azure portal 操作を行います。
:::

以下ドキュメントの、「Key Vault から証明書をインポートする」を実行しています。

https://learn.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate?tabs=apex%2Crbac%2Cazure-cli#import-a-certificate-from-key-vault

## カスタム ドメイン解決用 A レコードの追加
App Service がカスタム ドメインへの要求に応答できるように、DNS ゾーンに A レコードを追加します。カスタムドメインを払い出したタイミングで、App Service 側から特定のパブリック IP も払い出されるため、外部権威 DNS サーバでレコードを追加しましょう。

App Service ドメインを使っている場合には、同時に作成される Azure Public DNS Zone に追加すれば OK です。

Windows の場合は、`Resolve-DnsName`^[https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname?view=windowsserver2025-ps] コマンド等により適切に名前解決ができていることを確認しましょう。

:::message
手前に Application Gateway などのリバースプロキシを挟む場合には、そちらのパブリック IP に解決されるべきケースもあります。今回は App Service 単体での攻勢が前提です。
:::

## 疎通確認
カスタム ドメインに対してブラウザからアクセスを試みます。ブラウザから https でアクセスし、適切な証明書が返ってくることを確認します。

![](/images/20251024-apps-https/01.png)

# おわりに
App Service で HTTPS を構成するための流れをまとめました。Private DNS Zone でカスタム ドメインを App Service の Private Endpoint の プライベート IP に解決すれば、内部公開アプリでも HTTPS 化できます。App Service では、ホスト名が必要となるため、カスタム ドメインとの整合を取ることを意識しながら作業するとよいでしょう。