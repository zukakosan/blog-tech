---
title: "Microsoft Copilot in Azure for Networking と戯れる"
emoji: "🎅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","copilot","microsoft"]
published: true
published_at: 2024-12-25 09:00
---
皆さんこんにちは。そして、メリークリスマス。こちらの記事は Microsoft xxx アドベントカレンダー2024 最終日の記事です。

# はじめに
Microsoft Copilot in Azure は、Azure portal に統合された Copilot です。Microsoft Build 2024 あたりでプレビュー提供が開始され、日々そのケイパビリティを拡張しながら進化しています。

さて、先日行われた Microsoft Ignite 2024 では、ネットワーク関連のアップデートとして Microsoft Copilot in Azure for Networking[^1] が発表されました。for Networking とついているといっても何か特殊なサービスがリリースされたわけではなく、既存の Microsoft Copilot in Azure の機能（スキル）が拡張した、というものになります。
[^1]: https://techcommunity.microsoft.com/blog/azurenetworkingblog/introducing-copilot-in-azure-for-networking-your-ai-powered-azure-networking-ass/4298663

共通基盤によるクラウドの民主化や、AI 需要の高まりによって、ネットワークについてはあまり詳しくないけれど、クラウドを活用したシステムを作りたい、という需要が高まっています。  そのような方々にとっては Copliot と相談しながらトラブルシューティングできるのは非常に有益ではないでしょうか。

今回は、いったいどこまでできるのか？できないのか？という現状の把握をするために、実際に戯れてみたいと思います。

# Microsoft Copilot in Azure for Networking の機能
ドキュメント[^2] では、次のようなシナリオでの利用が紹介されています。
- ネットワーク製品情報のクエリ
    - 公開ドキュメントの内容に基づいて、製品に関する質問に回答する
- ネットワーク製品の選択とアーキテクチャのガイダンス クエリ
    - 製品に対する質問にや製品選択、ネットワーク計画、回復性、オンプレミスからの移行に関するガイダンスを提供する
    - 2024/12/07 時点では、まだまだ対応範囲が狭く、制限が多いです
- ネットワーク リソース インベントリ、トポロジ、トラフィック パスのクエリ
    - 顧客のネットワークリソース、トポロジ、送信元から送信先までのトラフィックパスを検出する
- ネットワーク接続のトラブルシューティングとサービス診断のクエリ
    - ネットワーク データとコントロール プレーン全体のさまざまな接続、構成、環境の問題にわたって顧客ネットワークのトラブルシューティングを実行する

[^2]: https://learn.microsoft.com/ja-jp/azure/copilot/network-management#scenarios

# 試してみる
先ほど挙げた機能のうち、インフラ管理の観点からどのように使いこなせるのか、模索してみます。


検証環境として、以下のリポジトリから Hub＆Spoke 型のネットワーク環境クイック スタートをデプロイし、いろいろと試行錯誤してみます。

https://github.com/zukakosan/bicep-hub-spoke-quickstarter

<!-- スクリーンショットでは、Azure portal の画面全体を載せています。Microsoft Copilot in Azure はユーザが現在見ている画面からサブスクリプションやリソースグループ、サービスの種類といったコンテキストを把握するためです。 -->

## ネットワーク経路の確認
VM 間での疎通確認をしたいとき、もしくは通信経路を再確認したい場合が多いです。そのような際に使えるのか確認してみます。

`VM-A から VM-B へのネットワーク経路を教えて` と聞いてみます。すると、送信元 VM を改めて選択するように促され、専用 UI が提供されます。
![](/images/20241207-networkcopilot/netcp01.png)
![](/images/20241207-networkcopilot/netcp02.png)

その後、内容の確認と編集が提案されます。一旦、TCP 22 でテストしてみます。
![](/images/20241207-networkcopilot/netcp07.png)

すると、`Blocked` が表示されました。テンプレートからデプロイしたため本来その間は疎通できているはずです。

ちなみに、提示されるボタンを押下することで、検出されている経路を表示できます。
![](/images/20241207-networkcopilot/netcp05.png)

よく画面を見てみると以下のような注意書きが提示されていました。
```
Prerequisites:

Ensure you have an Azure account with an active subscription.
Network Watcher must be enabled in the region of the virtual machine (VM) you want to troubleshoot.
You need a virtual machine with the Network Watcher agent VM extension installed, which should have outbound TCP connectivity to specific IP addresses and ports.
You can use Azure Cloud Shell or Azure PowerShell to run the necessary commands.
```
ということで、両側の VM に Network Watcher Agent を導入して再度確認してみます。
![](/images/20241207-networkcopilot/netcp06.png)

なかなか思うようにはいきませんね、また `block` になってしまいました。
![](/images/20241207-networkcopilot/netcp08.png)


以前 Network Watcher 側の機能で Hub&Spoke 構成において、アプライアンスが通信を仲介するパターンの疎通確認を行った際にも同じ問題が発生していたため、裏で同じ基盤が動いているとすると、同様の事象が発生していると思われます。

通常の VNet Peering 越しであれば、ドキュメントに記載の通りうまく動くはずです。

<!-- もう少し簡単にして、アプライアンスを経由しない VM でのインターネット アクセスの経路を聞いてみましょう。
![](/images/20241207-networkcopilot/netcp12.png) -->


<!-- 一旦、ほかの可能性を探ってみることにしましょう。 -->

## ネットワーク規模の確認
ネットワークの規模が大きくなってくると、設定変更に対するインパクトの大きさを調査するためにネットワークの規模感を把握したくなることがあると思います。そのようなタイミングで Copilot に聞いてみる、というイメージです。

まずはざっくりと聞いてみます。Azure Resource Graph のクエリを生成し、実行してくれます。
![](/images/20241207-networkcopilot/netcp09.png)

IP アドレスの枯渇的な観点などで、消費されている IP アドレスを知りたいこともあると思います。そのようなケースを想定して、追加で聞いてみます。

すると、VNet リソースの選択画面が提供され、その中から 1 つを選択します。
![](/images/20241207-networkcopilot/netcp10.png)

期待に包まれながら待っていると、そこまではできないという回答が得られました。
![](/images/20241207-networkcopilot/netcp11.png)

# おわりに
まだまだ複雑なネットワークアーキテクチャや、複雑なクエリが必要になるものについてはそのまま適用するのが難しい部分も確かにありそうです。

今後は、予測的なトラブルシューティングや、パフォーマンスや信頼性の観点でのネットワーク最適化提案、コンプライアンス要件を満たすためのセキュリティに関する能力も追加していく予定ということで、何ができるかを意識しなくても運用に使える実用的なものになってくることを期待しながらクリスマスを過ごしたいと思います。

では、よいクリスマスを。


