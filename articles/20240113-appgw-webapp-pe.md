---
title: "Application Gateway の バックエンドの Web Apps を Private Endpoint で保護する"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","network","paas","loadbalancer" ]
published: false
publication_name: "microsoft"
---

# はじめに
Azure Application Gateway (AppGw) の L7 のロードバランサーであり、バックエンドに Azure App Service の Web Apps を配置することが可能です。Web Apps は PaaS サービスなので、ネットワークを気にせず利用する場合、AppGw -> Web Apps の通信はインターネット側を通る構成となります。
閉域化の要件がある場合は、こちらのドキュメント[^1] に記載のように、AppGw -> Web Apps の通信も Private Endpoint 経由とするのが良いでしょう。本記事ではその構成を試してみます。

![](/images/20240113-appgw-webapp-pe/private-endpoint-appgw.png)

[^1]: https://learn.microsoft.com/ja-jp/azure/app-service/overview-app-gateway-integration#considerations-for-using-private-endpoints

# 試してみる

## Web Apps のデプロイ
Web Apps をデプロイします。デプロイ後、展開されるURLにアクセスして、起動していることを確認します。この時点ではパブリックアクセス可能な状態です。
![](/images/20240113-appgw-webapp-pe/01.png)

## Application Gateway のバックエンドプールに Web Apps を配置して構成
パブリックな Application Gatway をデプロイします。作成時もしくは作成後にバックエンドプールの設定を開き、App Service としてターゲットを追加します。 
![](/images/20240113-appgw-webapp-pe/02.png)

以下のようなカスタムプローブを用意します。
![](/images/20240113-appgw-webapp-pe/03.png)

バックエンド設定で以下のように構成します。
![](/images/20240113-appgw-webapp-pe/04.png)

検証用なので、Web Apps の HTTP 強制をオフにします。
![](/images/20240113-appgw-webapp-pe/05.png)

Application Gateway のパブリック IP 経由で Web Apps にアクセスできることを確認します。
![](/images/20240113-appgw-webapp-pe/06.png)

## Private Endpoint の設定
Web Apps に対して Private Endpoint の設定を入れていきます。これにより、Private Endpoint に到達性のある範囲(ここでは VNET 内)からしか呼び出せなくなります。設定自体は Web Apps の[ネットワーク]から行っていきます。
![](/images/20240113-appgw-webapp-pe/07.png)
![](/images/20240113-appgw-webapp-pe/08.png)

パブリックアクセスも拒否します。
![](/images/20240113-appgw-webapp-pe/09.png)

ネットワークの設定としてはパ広州ネットワークアクセスが無効、Private Endpoint が 1 つ構成された状態になります。
![](/images/20240113-appgw-webapp-pe/10.png)

この状態で、Application Gateway のパブリック IP を叩くと、まだ Private Endpoint のための DNS 設定が反映されていないためパブリック回しで Web Apps にアクセスしようとして `403 Error` となります。Application Gateway が DNS参照 の結果をキャッシュしているため、再起動によるリフレッシュが必要です。以下のコマンドで起動・停止が実行可能です。

```bash
az network application-gateway stop --resource-group myRG --name myAppGw
az network application-gateway start --resource-group myRG --name myAppGw
```

その後再度 Application Gateway 経由でアクセスすると、問題なく初期画面が表示されました。
![](/images/20240113-appgw-webapp-pe/11.png)

# まとめ
- Application Gateway のバックエンドに Web Apps を置く構成について、Private Endpoint 経由にする構成を試しました。
- 多少のハマりポイントは DNS のキャッシュ更新くらいですが、Application Gateway の構成自体ややこしいので、どなたかの役に立てば幸いです。