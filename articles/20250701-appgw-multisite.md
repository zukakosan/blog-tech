---
title: "Application Gateway のマルチサイト リスナーを構成する"
emoji: "⛩️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","app","microsoft"]
published: false
publication_name: "microsoft"
---

# はじめに
Azure Application Gateway には マルチ サイトホスティング[^1] の機能があります。これにより、Application Gateway の同一ポート上で複数ドメインの Web アプリケーションを配信できます。通常、リスナーは特定のポートに対して 1 角見構成できますが、マルチサイト リスナーを使用することで、最大で 100 個のアプリケーションをホストできます。本記事では、実際の構成の流れを見ていきます。
![](/images/20250701-appgw-multisite/01.png)

[^1]: https://learn.microsoft.com/ja-jp/azure/application-gateway/multiple-site-overview

# リソース作成と設定
マルチサイトであることから、Application Gateway x1 と App Service x2 を用意します。
## VNET の作成
Application Gateway の作成には、VNET が必要になります。また、特定のサブネットを委任する形になるため、専用のサブネットを切っておきます。
![](/images/20250701-appgw-multisite/02.png)

## Application Gateway の作成
Application Gateway をデプロイしていきます。のちに構成するため、バックエンドプール等は空にして、最低限の構成のみ行います。
![](/images/20250701-appgw-multisite/03.png)

リスナーは HTTP とし、ルーティング規則も既定のもので作成します(`Listenr Type` が `Basic` になっていますが、のちの手順で `Multisite` に変更します)。
![](/images/20250701-appgw-multisite/04.png)
![](/images/20250701-appgw-multisite/05.png)

## App Service の作成
App Service を 2 つ作成していきます。違いが分かるように `sampleapp-1` のランタイムを `node.js`, `sampleapp-2` のランタイムを `Python` にしています。
![](/images/20250701-appgw-multisite/06.png)
![](/images/20250701-appgw-multisite/07.png)

## Application Gateway の設定
### バックエンドプールの作成
バックエンドプールをマルチサイト用に二つ構成します。Application Gateway の作成時には、一つしか作成していないため、二つ目のアプリ用にもう一つ作成します。ぞれぞれに対して、sampleapp-1 と sampleapp-2 を指定します。
![](/images/20250701-appgw-multisite/08.png)

### Multisite Listner の構成
最初に作成済みのリスナーはシングルサイト用 (Basic) のため、マルチサイトに設定しなおします。`Multisite` を選択すると、Host name を指定する必要がありますが、ここを正しく設定することで、同一のフロント IP に対するリクエストが Host Header ベースで振り分けられるようになります。つまり、アクセスさせたい FQDN を適切に入れる必要があります。
![](/images/20250701-appgw-multisite/09.png)

### Backend Health の確認
この時点で、バックエンドに到達できるか確認します。おそらく、どちらのバックエンドに対しても `Unhealthy` と表示されているはずです。これは、Backend settings において、カスタム プローブを使用していないことに起因します。
![](/images/20250701-appgw-multisite/10.png)

App Serivce へのリクエスト (Probe) では、Host ヘッダー が対象のアプリケーション用の `xxx.azurewebsites.net` になっていないと 404 を返します。

よって、カスタム プローブによって、明示的に Host 名を指定します。
![](/images/20250701-appgw-multisite/11.png)
![](/images/20250701-appgw-multisite/12.png)
![](/images/20250701-appgw-multisite/13.png)

改めて、Backend Health を見ると、`Healthy` になっています。
![](/images/20250701-appgw-multisite/14.png)

# hosts ファイルの設定
# 接続のプライベート化
# ログの確認
# おわりに
