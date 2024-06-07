---
title: "Azure Bastion Developer SKU で気軽に VM に接続する"
emoji: "🏯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","bastion","RDP"]
published: true
publication_name: "microsoft"
---

# はじめに
Azure Bastion はご存じの通り、VM にアクセスするための踏み台サーバーのようなサービスです。Azure Bastion を経由することで、VNet 内にある VM に対してパブリック IP を付与することなくアクセスできます。

従来、Azure Bastion では、VNet 内に AzureBastionSubnet というサブネットを立てて、そこにデプロイする必要がありました。デプロイにもある程度の時間がかかり、VM に接続していない時間でも Bastion ホストがデプロイされている間は課金されていました。
新しい Azure Bastion Developer SKU が登場し、無料かつ、一瞬で立ち上がるようになったため気軽に使いやすくなりました。

# Azure Bastion Developer SKU とは
Developer SKU はなんと言っても無料というところが大きな特徴です。VM の接続画面から直接 Bastion 経由で接続できます。

また、アーキテクチャ上の特徴として、Developer SKU の場合は、bastion ホストは自分の VNet にはデプロイされず、共有リソースを間借りする形になります。専用ではないことから機能的な制約も多いのは事実です。ピアリング越しの接続もサポートされていません。

現時点で利用できるリージョンは限られています。最新のものはドキュメントをご確認ください[^1]。接続先の VM が Bastion と同じリージョンに存在する必要があるため、実質的に Developer SKU で接続できる VM は現時点で限りがあります。
[^1]:https://learn.microsoft.com/ja-jp/azure/bastion/quickstart-developer-sku

# 試してみる
適当な VNet を作成し、内部に Windows VM を作成します。前述のとおり、この VM は Developer SKU がサポートしているリージョンに作成する必要があります。
![](/images/20240607-bastion-dev/bastion-1.png)

作成した VM の Bastion メニューを開くと、何も構成せず Developer SKU での接続が可能な状態となっています。
![](/images/20240607-bastion-dev/bastion-2.png)

この際、NSG ルールでプライベート IP アドレス 168.63.129.16[^2] からポート 22 および 3389 へのトラフィックが許可されている必要があります。共有リソースということで、Azure 基盤側の機能にアクセスするエンドポイントを経由しての接続になるようです。
[^2]:https://learn.microsoft.com/ja-jp/azure/virtual-network/what-is-ip-address-168-63-129-16

ユーザー名とパスワードを入力すると、**数秒**で Bastion Dev SKU が作成され、VM に接続できます。
![](/images/20240607-bastion-dev/bastion-3.png)

また、専用の bastion ホストをデプロイしたい場合には、下部のメニューから構成してデプロイできます。
![](/images/20240607-bastion-dev/bastion-4.png)


# おわりに
Azure Bastion Developer SKU による VM 接続を試してみました。組織によっては VM にパブリック IP 付与することが許可されていなかったり、NSG のルールが自動で構成されて VM に接続する際に手間がかかるという場合があると思います。

そのようなケースにおいて、無料でサクッと VM に接続できる選択肢が増えたのはいいことではないでしょうか。自分も検証がはかどるので非常にありがたい機能だと感じました。