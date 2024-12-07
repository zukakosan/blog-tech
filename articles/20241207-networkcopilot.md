---
title: "Microsoft Copilot in Azure for Network のケイパビリティを確かめる"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","copilot","microsoft"]
published: false
published_at: 2024-12-25 09:00
---
皆さんこんにちは。そして、メリークリスマス。こちらの記事は Microsoft xxx アドベントカレンダー2024 最終日の記事です。

# はじめに
Microsoft Copilot in Azure は、Azure portal に統合された Copilot です。Microsoft Build 2024 あたりでプレビュー提供が開始され、日々そのケイパビリティを拡張しながら進化しています。

さて、先日行われた Microsoft Ignite 2024 では、ネットワーク関連のアップデートとして Microsoft Copilot in Azure for Network が発表されました。for Network とついているといっても何か特殊なサービスがリリースされたわけではなく、既存の Microsoft Copilot in Azure の機能（スキル）が拡張した、というものになります。

共通基盤によるクラウドの民主化や、AI 需要の高まりによって、ネットワークについてはあまり詳しくないけれど、クラウドを活用したシステムを作りたい、という需要が高まっています。  そのような方々にとっては Copliot と相談しながらトラブルシューティングできるのは非常に有益ではないでしょうか。

# Microsoft Copilot in Azure for Network の機能
ドキュメント[^1] では、次のようなシナリオでの利用が紹介されています。
- ネットワーク製品情報のクエリ
    - 公開ドキュメントの内容に基づいて、製品に関する質問に回答する
- ネットワーク製品の選択とアーキテクチャのガイダンス クエリ
    - 製品に対する質問にや製品選択、ネットワーク計画、回復性、オンプレミスからの移行に関するガイダンスを提供する
    - 2024/12/07 時点では、まだまだ対応範囲が狭く、制限が多いです
- ネットワーク リソース インベントリ、トポロジ、トラフィック パスのクエリ
    - 顧客のネットワークリソース、トポロジ、送信元から送信先までのトラフィックパスを検出する
- ネットワーク接続のトラブルシューティングとサービス診断のクエリ
    - ネットワーク データとコントロール プレーン全体のさまざまな接続、構成、環境の問題にわたって顧客ネットワークのトラブルシューティングを実行する

[^1]: https://learn.microsoft.com/ja-jp/azure/copilot/network-management#scenarios

# 試してみる
先ほど挙げた機能のうち、インフラ管理の観点からどのように使いこなせるのか、模索してみます。検証環境として、以下のリポジトリからクイックスタートをデプロイし、Hub＆Spoke型のネットワーク環境を用意しておきます。

https://github.com/zukakosan/bicep-hub-spoke-quickstarter

## ネットワーク経路の確認

送信元 VM を改めて選択するように促され、専用 UI が提供されます。ここでは、`vm-spoke-1` を選択します。同様にして、宛先 VM も選択します。
![](/images/20241207-networkcopilot/netcp01.png)
![](/images/20241207-networkcopilot/netcp02.png)

その後、内容の確認と編集が提案されます。一旦、TCP 22 でテストしてみます。
![](/images/20241207-networkcopilot/netcp03.png)

すると、`Blocked` が表示されました。テンプレートからデプロイしたため本来動くはずです。

ちなみに、提示されるボタンを押下することで、全体のトポロジを表示できます。
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

なかなか思うようにはいきませんね、また `Block` になってしまいました。以前 Network Watcher 側の機能で Hub&Spoke でアプライアンスが挟まるパターンの疎通確認を行った際にも同じ問題が発生していたため、裏で同じ基盤が動いているとすると、同様の事象が発生していると思われます。