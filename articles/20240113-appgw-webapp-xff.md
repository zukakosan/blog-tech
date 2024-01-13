---
title: "Application Gateway 経由の Web Apps 接続時にソース IP にるアクセス制御を行う"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","xff","paas"]
published: false
---
# はじめに
Application Gateway のバックエンドに Web Apps を配置した場合、Web Apps 側から見るとソース IP は Application Gateway の IP アドレスに見えます。これは、Application Gateway がプロキシとして通信を仲介しているためです。この状況では、Application Gateway に到達できるソースからは Web Apps にも到達できるということになります。とはいえ、本来のリクエストのオリジンとなるソース IP(=クライアント IP) の情報をもとに Web Apps 側でアクセス制御を行いたい状況もあるでしょう。その場合には、リクエストヘッダーの `X-Forwarded-For` を利用することで Web Apps 側での制御が可能になります。

:::message
前回の記事[^1] の延長で、Application Gateway が Private Endpoint 経由で Web Apps にトラフィックを流す環境下で一度試したのですが、今回の検証内容は反映されなかったため、あくまで公衆ネットワークに対するアクセス制御の場合に利用できると思った方がよさそうです。
:::

前回の記事[^1] の続きとして、Application Gateway のバックエンドに Web Apps を配置した場合に

# 試してみる

# まとめ