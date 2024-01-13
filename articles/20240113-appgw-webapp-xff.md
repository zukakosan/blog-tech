---
title: "Application Gateway 経由の Web Apps 接続時にソース IP にるアクセス制御を行う"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","xff","paas"]
published: false
publication_name: "microsoft"
---
# はじめに
Application Gateway のバックエンドに Web Apps を配置した場合、Web Apps 側から見るとソース IP は Application Gateway の IP アドレスに見えます。これは、Application Gateway がプロキシとして通信を仲介しているためです。この状況では、Application Gateway に到達できるソースからは Web Apps にも到達できるということになります。

とはいえ、本来のリクエストのオリジンとなるソース IP(=クライアント IP) の情報をもとに Web Apps 側でアクセス制御を行いたい状況もあるでしょう。その場合には、http ヘッダーの `X-Forwarded-For` を利用することで Web Apps 側での制御が可能になります。これは、高度なシナリオとして、ドキュメント[^1] でも紹介されています。
[^1]: https://learn.microsoft.com/ja-jp/azure/app-service/app-service-ip-restrictions?tabs=azurecli#access-restriction-advanced-scenarios

Application Gateway では、要求をバックエンドに転送する前に、すべての要求に X-Forwarded-For ヘッダーが挿入されます[^2]。この http ヘッダーを見ることで、Web Apps はクライアント IP を知ることができ、アクセス制御ができるというわけですね。 
[^2]: https://learn.microsoft.com/ja-jp/azure/application-gateway/rewrite-http-headers-url#remove-port-information-from-the-x-forwarded-for-header

:::message
前回の記事[^3] の延長として、Application Gateway が Private Endpoint 経由で Web Apps にトラフィックを流す環境下で一度試したのですが、今回の検証内容は反映されなかったため、Private Endpoint は構成せずに試しています。

あくまで公衆ネットワークに対するアクセス制御の場合に利用できると思った方がよさそうです。
[^3]: https://zenn.dev/microsoft/articles/20240113-appgw-webapp-pe
:::

# 試してみる
Web Apps をバックエンドにもつ Application Gateway を構成します。こちらは前回の記事[^4] をご参照ください。ただし、Web Apps の Private Endpoint は構成しないでおきます。
[^3]: https://zenn.dev/microsoft/articles/20240113-appgw-webapp-pe

## アクセス制御規則の作成
Web Apps の[ネットワーク]設定から、[公衆ネットワークアクセス]を編集します。
![](/images/20240113-appgw-webapp-xff/01.png)


今回のルールの構成としては優先度の高い順に以下のイメージです。
- `X-Forwarded-For` で特定の IP (ここでは著者自宅 IP ) を拒否
- Application Gateway の存在するサブネットからのトラフィックを許可
- 該当しないものはすべて拒否

著者の自宅の IP からのみ、Application Gateway からのトラフィックをブロックしたうえで、Application Gateway からくるリクエストのみ許可するという設定です。Web Apps が外部から直接到達されることも防いでいます。
![](/images/20240113-appgw-webapp-xff/02.png)
![](/images/20240113-appgw-webapp-xff/03.png)

## 動作確認
ルールの構成をしてからキャッシュが消える程度待ってから、再度 Application Gateway のパブリック IP に http アクセスすると、想定通り `403 Error` となりました。

![](/images/20240113-appgw-webapp-xff/04.png)

# まとめ
- Application Gateway 経由の場合でも Web Apps 側でクライアント IP ベースで制御したい場合は、http ヘッダーを活用することで実現できることを確認しました。
- とはいえ、あくまで公衆ネットワークの設定であり、Private Endpoint とは別の設定となるため注意が必要です。
