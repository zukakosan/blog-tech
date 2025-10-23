---
title: "Azure AppService を KeyVault上 の証明書で HTTPS 化する"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに
Azure App Service はコードを配置するだけで Web アプリケーションを展開できる人気の PaaS です。この App Service は既定の状態では、`xxx.azurewebsites.net` という App Service の既定のドメインでホストされます。

ただ使えればいい、というだけであれば既定のドメインで HTTPS を強制してしまえば、それで終わりです。しかし、基本的には組織のドメインを使ってアプリケーションを公開するはずです。「社内から、社外から問わずカスタム ドメインを使ってアプリを公開し、HTTPS で接続させたい」という要件を Azure 上で実現します。

# 設定の流れ
## カスタム ドメインの取得
前提としてカスタム ドメインは必須となります。Azure 上では、App Service ドメインというサービスから取得することも可能です。自分が保有しているドメインがあることが前提です。

## 証明書の取得
ドメインを取得したら、対応する証明書を取得しましょう。DigiCert での証明書の取得の仕方および、Azure Key Vault へのインポートに関しては以下の記事にまとめています。

https://zenn.dev/microsoft/articles/20250711-digicertonazure


## Azure App Service へのカスタムドメイン設定
~~（App Service ドメインを使っていないケースが多いと思うので）~~ カスタム ドメインは使いたい FQDN を直接入力します。A レコードまたは CNAME レコードによってドメインの検証を行います。

こちらのドキュメントの「カスタム ドメイン構成」~「ドメインの検証」まで行いましょう。

https://learn.microsoft.com/ja-jp/azure/app-service/app-service-web-tutorial-custom-domain?tabs=root%2Cazurecli#configure-a-custom-domain