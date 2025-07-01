---
title: "一つの Azure Application Gateway で複数の Web アプリを管理しよう"
emoji: "⛩️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","app","microsoft"]
published: true
publication_name: "microsoft"
---

# はじめに
Azure Application Gateway には マルチ サイトホスティング[^1] の機能があります。これにより、Application Gateway の同一ポート上で複数ドメインの Web アプリケーションを配信できます。通常、リスナーは特定のポートに対して 1 つのリスナーを構成できますが、マルチサイト リスナーを使用することで、最大で 100 個のアプリケーションをホストできます。本記事では、実際の構成の流れを見ていきます。
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

リスナーは HTTP とし、ルーティング規則も既定のもので作成します(`Listener Type` が `Basic` になっていますが、のちの手順で `Multisite` に変更します)。
![](/images/20250701-appgw-multisite/04.png)
![](/images/20250701-appgw-multisite/05.png)

## App Service の作成
App Service を 2 つ作成していきます。違いが分かるように `sampleapp-1` のランタイムを `node.js`, `sampleapp-2` のランタイムを `Python` にしています。
![](/images/20250701-appgw-multisite/06.png)
![](/images/20250701-appgw-multisite/07.png)

## Application Gateway の設定
### バックエンドプールの作成
バックエンドプールをマルチサイト用に二つ構成します。Application Gateway の作成時には、一つしか作成していないため、二つ目のアプリ用にもう一つ作成します。それぞれに対して、sampleapp-1 と sampleapp-2 を指定します。
![](/images/20250701-appgw-multisite/08.png)

### Multisite Listener の構成
最初に作成済みのリスナーはシングルサイト用 (Basic) のため、マルチサイトに設定しなおします。`Multisite` を選択すると、Host name を指定する必要がありますが、ここを正しく設定することで、同一のフロント IP に対するリクエストが Host Header ベースで振り分けられるようになります。つまり、アクセスさせたい FQDN を適切に入れる必要があります。
![](/images/20250701-appgw-multisite/09.png)

### Backend Health の確認
この時点で、バックエンドに到達できるか確認します。おそらく、どちらのバックエンドに対しても `Unhealthy` と表示されているはずです。これは、Backend settings において、カスタム プローブを使用していないことに起因します。
![](/images/20250701-appgw-multisite/10.png)

App Service へのリクエスト (Probe) では、Host ヘッダー が対象のアプリケーション用の `xxx.azurewebsites.net` になっていないと 404 を返します。

よって、カスタム プローブで、明示的に Host 名を指定します。
![](/images/20250701-appgw-multisite/11.png)
![](/images/20250701-appgw-multisite/12.png)
![](/images/20250701-appgw-multisite/13.png)

改めて、Backend Health を見ると、`Healthy` になっています。
![](/images/20250701-appgw-multisite/14.png)

# hosts ファイルの設定
マルチサイト ホスティングの場合、複数のドメイン名を Application Gateway のフロント IP に解決する必要があります。今回は、手軽に手元の PC 上で、hosts ファイルに記述します。管理者モードで以下に存在する hosts ファイルを開き、末尾に IP アドレスと FQDN の対応を追記します。

```
"C:\Windows\System32\drivers\etc\hosts"
```

これにより、各 App Service に対するリクエストは Application Gateway を向くようになります。

```txt
# Copyright (c) 1993-2009 Microsoft Corp.
#
# This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
#
# This file contains the mappings of IP addresses to host names. Each
# entry should be kept on an individual line. The IP address should
# be placed in the first column followed by the corresponding host name.
# The IP address and the host name should be separated by at least one
# space.

48.xxx.xxx.15 sampleapp-1-eyb6gkabbhh6bwg6.canadacentral-01.azurewebsites.net
48.xxx.xxx.15 sampleapp-2-hbewgra6bxf2arhy.canadacentral-01.azurewebsites.net

``` 

設定を保存したら、反映されていることを確認します。`nslookup` では、hosts ファイルを参照しないため、`Resolve-DnsName` コマンドを使用します。

```powershell
PS> Resolve-DnsName sampleapp-2-hbewgra6bxf2arhy.canadacentral-01.azurewebsites.net

Name                                           Type   TTL   Section    IPAddress
----                                           ----   ---   -------    ---------
sampleapp-2-hbewgra6bxf2arhy.canadacentral-01. A      16368 Answer     48.xxx.xxx.15
azurewebsites.net    
```

# App Service の http 許可
今回は検証上 http で疎通確認を行います。App Service では、https を強制させることもできるので、その設定が `off` になっていることを確認します。もちろん、本来は `On` のほうがいいです。

![](/images/20250701-appgw-multisite/15.png)

# 接続テスト
モダンブラウザから接続すると、https にリダイレクトされてしまう可能性があるため、シンプルに `curl` で疎通確認を行います。

sampleapp-1 向けに対しては、node.js を使用した場合のページが返ってきています (例: `src="https://appservice.azureedge.net/images/linux-landing-page/v4/built-nodejs.svg"`)。
```powershell
PS> curl http://sampleapp-1-eyb6gkabbhh6bwg6.canadacentral-01.azurewebsites.net/
```
:::details Response
```txt
StatusCode        : 200
StatusDescription : OK
Content           : <!DOCTYPE html>
                    <html lang="en">

                    <head>
                        <meta charset="utf-8" />
                        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
                        <meta http-equiv="X-UA-Compatible" content="IE=ed...
RawContent        : HTTP/1.1 200 OK
                    Connection: keep-alive
                    request-context: appId=cid-v1:
                    Accept-Ranges: bytes
                    Content-Length: 4560
                    Cache-Control: public, max-age=0
                    Content-Type: text/html; charset=utf-8
                    Date: Tue...
Forms             : {}
Headers           : {[Connection, keep-alive], [request-context, appId=cid-v1:], [Accept-Ranges, bytes],
                    [Content-Length, 4560]...}
Images            : {@{innerHTML=; innerText=; outerHTML=<img width="270" height="108" alt=""
                    src="https://appservice.azureedge.net/images/app-service/v4/azurelogo.svg">; outerText=;
                    tagName=IMG; width=270; height=108; alt=;
                    src=https://appservice.azureedge.net/images/app-service/v4/azurelogo.svg}, @{innerHTML=;
                    innerText=; outerHTML=<img src="https://appservice.azureedge.net/images/app-service/v4/web.svg">;
                    outerText=; tagName=IMG; src=https://appservice.azureedge.net/images/app-service/v4/web.svg},
                    @{innerHTML=; innerText=; outerHTML=<img width="50" height="50"
                    src="https://appservice.azureedge.net/images/linux-landing-page/v4/built-nodejs.svg">; outerText=;
                    tagName=IMG; width=50; height=50;
                    src=https://appservice.azureedge.net/images/linux-landing-page/v4/built-nodejs.svg}, @{innerHTML=;
                    innerText=; outerHTML=<img src="https://appservice.azureedge.net/images/app-service/v4/web.svg">;
                    outerText=; tagName=IMG; src=https://appservice.azureedge.net/images/app-service/v4/web.svg}}
InputFields       : {}
Links             : {@{innerHTML=<button class="btn btn-primary mt-4" id="deplCenter" type="submit">Deployment
                                                        center</button>; innerText=Deployment center; outerHTML=<a
                    id="depCenterLink" href="https://go.microsoft.com/fwlink/?linkid=2057852"><button class="btn
                    btn-primary mt-4" id="deplCenter" type="submit">Deployment
                                                        center</button></a>; outerText=Deployment center; tagName=A;
                    id=depCenterLink; href=https://go.microsoft.com/fwlink/?linkid=2057852}, @{innerHTML=<button
                    class="btn btn-primary mt-4" id="quickStart" type="submit">Quickstart</button>;
                    innerText=Quickstart; outerHTML=<a href="https://go.microsoft.com/fwlink/?linkid=2084231"><button
                    class="btn btn-primary mt-4" id="quickStart" type="submit">Quickstart</button></a>;
                    outerText=Quickstart; tagName=A; href=https://go.microsoft.com/fwlink/?linkid=2084231}}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 4560

```
:::

sampleapp-2 宛てでは、`Python` 用のトップページが返ってきています（`src="https://appservice.azureedge.net/images/linux-landing-page/v4/built-python.svg"`）。
```powershell
PS> curl http://sampleapp-2-hbewgra6bxf2arhy.canadacentral-01.azurewebsites.net
```
:::details Response
```txt
StatusCode        : 200
StatusDescription : OK
Content           : <!DOCTYPE html>
                    <html lang="en">

                    <head>
                        <meta charset="utf-8" />
                        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
                        <meta http-equiv="X-UA-Compatible" content="IE=ed...
RawContent        : HTTP/1.1 200 OK
                    Connection: keep-alive
                    Content-Disposition: inline; filename=hostingstart.html
                    Content-Length: 4560
                    Cache-Control: no-cache
                    Content-Type: text/html; charset=utf-8
                    Date: Tue, 01 J...
Forms             : {}
Headers           : {[Connection, keep-alive], [Content-Disposition, inline; filename=hostingstart.html],
                    [Content-Length, 4560], [Cache-Control, no-cache]...}
Images            : {@{innerHTML=; innerText=; outerHTML=<img width="270" height="108" alt=""
                    src="https://appservice.azureedge.net/images/app-service/v4/azurelogo.svg">; outerText=;
                    tagName=IMG; width=270; height=108; alt=;
                    src=https://appservice.azureedge.net/images/app-service/v4/azurelogo.svg}, @{innerHTML=;
                    innerText=; outerHTML=<img src="https://appservice.azureedge.net/images/app-service/v4/web.svg">;
                    outerText=; tagName=IMG; src=https://appservice.azureedge.net/images/app-service/v4/web.svg},
                    @{innerHTML=; innerText=; outerHTML=<img width="50" height="50"
                    src="https://appservice.azureedge.net/images/linux-landing-page/v4/built-python.svg">; outerText=;
                    tagName=IMG; width=50; height=50;
                    src=https://appservice.azureedge.net/images/linux-landing-page/v4/built-python.svg}, @{innerHTML=;
                    innerText=; outerHTML=<img src="https://appservice.azureedge.net/images/app-service/v4/web.svg">;
                    outerText=; tagName=IMG; src=https://appservice.azureedge.net/images/app-service/v4/web.svg}}
InputFields       : {}
Links             : {@{innerHTML=<button class="btn btn-primary mt-4" id="deplCenter" type="submit">Deployment
                                                        center</button>; innerText=Deployment center; outerHTML=<a
                    id="depCenterLink" href="https://go.microsoft.com/fwlink/?linkid=2057852"><button class="btn
                    btn-primary mt-4" id="deplCenter" type="submit">Deployment
                                                        center</button></a>; outerText=Deployment center; tagName=A;
                    id=depCenterLink; href=https://go.microsoft.com/fwlink/?linkid=2057852}, @{innerHTML=<button
                    class="btn btn-primary mt-4" id="quickStart" type="submit">Quickstart</button>;
                    innerText=Quickstart; outerHTML=<a href="https://go.microsoft.com/fwlink/?linkid=2084231"><button
                    class="btn btn-primary mt-4" id="quickStart" type="submit">Quickstart</button></a>;
                    outerText=Quickstart; tagName=A; href=https://go.microsoft.com/fwlink/?linkid=2084231}}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 4560
```
:::

# App Service への通信のプライベート化
現在の構成では、App Service はパブリック許可の状態になっているため、DNS によって Application Gateway 経由になるとしてもパブリックに面している状態とも言えます。

Application Gateway から App Service の通信自体もパブリックのため、App Service 側を完全に閉塞はできません。よって、プライベート化をしたい場合には、Private Endpoint を使用します。ところどころ端折るので、細かい点は以前の記事[^2] をご確認ください。

（Application Gateway 用のサブネットにサービスエンドポイントを設定することもできるかもしれませんが、サービスエンドポイントポリシーが非サポートなど制約があるようです[^3]。）

`sampleapp-1` に対して Private Endpoint を作成します。
![](/images/20250701-appgw-multisite/16.png)

Private DNS Zone の統合によって、Application Gateway をはじめ VNET 内からの App Service の FQDN の名前解決は、プライベートに解決されます。外部との接点を閉じるために App Service のパブリックアクセスも拒否します。
![](/images/20250701-appgw-multisite/17.png)

この状態で、Application Gateway 側の Backend Health を確認しても、Healthy 状態を維持しています。
![](/images/20250701-appgw-multisite/18.png)


手元から curl を実行して疎通確認をします（とはいえ、手元の PC と Application Gateway 間の通信に変化はないです）。

```powershell
PS> curl http://sampleapp-1-eyb6gkabbhh6bwg6.canadacentral-01.azurewebsites.net/
```

:::details Response
```txt
StatusCode        : 200
StatusDescription : OK
Content           : <!DOCTYPE html>
                    <html lang="en">

                    <head>
                        <meta charset="utf-8" />
                        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
                        <meta http-equiv="X-UA-Compatible" content="IE=ed...
RawContent        : HTTP/1.1 200 OK
                    Connection: keep-alive
                    request-context: appId=cid-v1:
                    Accept-Ranges: bytes
                    Content-Length: 4560
                    Cache-Control: public, max-age=0
                    Content-Type: text/html; charset=utf-8
                    Date: Tue...
Forms             : {}
Headers           : {[Connection, keep-alive], [request-context, appId=cid-v1:], [Accept-Ranges, bytes],
                    [Content-Length, 4560]...}
Images            : {@{innerHTML=; innerText=; outerHTML=<img width="270" height="108" alt=""
                    src="https://appservice.azureedge.net/images/app-service/v4/azurelogo.svg">; outerText=;
                    tagName=IMG; width=270; height=108; alt=;
                    src=https://appservice.azureedge.net/images/app-service/v4/azurelogo.svg}, @{innerHTML=;
                    innerText=; outerHTML=<img src="https://appservice.azureedge.net/images/app-service/v4/web.svg">;
                    outerText=; tagName=IMG; src=https://appservice.azureedge.net/images/app-service/v4/web.svg},
                    @{innerHTML=; innerText=; outerHTML=<img width="50" height="50"
                    src="https://appservice.azureedge.net/images/linux-landing-page/v4/built-nodejs.svg">; outerText=;
                    tagName=IMG; width=50; height=50;
                    src=https://appservice.azureedge.net/images/linux-landing-page/v4/built-nodejs.svg}, @{innerHTML=;
                    innerText=; outerHTML=<img src="https://appservice.azureedge.net/images/app-service/v4/web.svg">;
                    outerText=; tagName=IMG; src=https://appservice.azureedge.net/images/app-service/v4/web.svg}}
InputFields       : {}
Links             : {@{innerHTML=<button class="btn btn-primary mt-4" id="deplCenter" type="submit">Deployment
                                                        center</button>; innerText=Deployment center; outerHTML=<a
                    id="depCenterLink" href="https://go.microsoft.com/fwlink/?linkid=2057852"><button class="btn
                    btn-primary mt-4" id="deplCenter" type="submit">Deployment
                                                        center</button></a>; outerText=Deployment center; tagName=A;
                    id=depCenterLink; href=https://go.microsoft.com/fwlink/?linkid=2057852}, @{innerHTML=<button
                    class="btn btn-primary mt-4" id="quickStart" type="submit">Quickstart</button>;
                    innerText=Quickstart; outerHTML=<a href="https://go.microsoft.com/fwlink/?linkid=2084231"><button
                    class="btn btn-primary mt-4" id="quickStart" type="submit">Quickstart</button></a>;
                    outerText=Quickstart; tagName=A; href=https://go.microsoft.com/fwlink/?linkid=2084231}}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 4560
```
:::

[^2]:https://zenn.dev/microsoft/articles/20240113-appgw-webapp-pe
[^3]:https://learn.microsoft.com/ja-jp/azure/application-gateway/configuration-infrastructure#virtual-network-and-dedicated-subnet

# ログの確認
もう少し変化がわかりやすいように、ログを確認します。sampleapp-1 (Private) と sampleapp-2 (Public) でログの内容を比較します。まずは、それぞれに対して以下のような Diagnostic setting を追加します。
![](/images/20250701-appgw-multisite/19.png)

それぞれについて `AppServiceHTTPLogs` テーブルのログを比較します。
## sampleapp-1 (Private)
定期的に Probe の通信が 200 で返っている中で、手元からの curl は UserAgent 付きのログとして記録されています。また、Private Endpoint 経由であることから、CIp (ClientIP) は IPv6 の形式になっています。
![](/images/20250701-appgw-multisite/20.png)

## sampleapp-2 (Public)
こちらも基本的には、Probe のログが記録されていますが、手元からの curl に対しては、UserAgent が記録されています。そして、CIp (Client  IP) は Application Gateway の Public IP になっています。 このように、ログの観点からも違いが確認できました。
![](/images/20250701-appgw-multisite/21.png)

# おわりに
本記事では、マルチサイト リスナーの構成とプライベート化を検証しました。Application Gateway は設定メニューを行ったり来たりする都合上、思わぬ設定ミスが発生しやすいです。適宜 Backend Health の画面に立ち戻り、接続が確立されていることを確認したうえで設定を進めると切り分けがしやすいと改めて感じました。
