---
title: "Azure VM の透過的なメンテナンスを監視し、ログに記録してアラートを発報する"
emoji: "⚠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","vm","monitoring"]
published: true
publication_name: "microsoft"

---

# はじめに
Azure で提供されている VM において、ホストマシン側の更新プログラムの適用やハードウェアの性能低下等のイベントに対応し、サービス自体のセキュリティを最新の状態にするために日々メンテナンスが実行されています。ライブ マイグレーションとメモリ保持メンテナンスにおいては、ほとんどの場合マシンが数秒停止するのみで、透過的に行われています。ただし、アプリケーションによってはこのレベルの停止でも影響を受けるような場合があります。

その場合、Instance Metadata Service (IMDS) で提供される専用のエンドポイントに能動的に問い合わせることで Scheduled Events として、今後のメンテナンスに関する通知を **15 分前**に受け取ることができます。

しかし、あくまで能動的に取りに行く必要があるため、監視のためにはそれ用の仕組みが必要となります。

本記事では、こちらのドキュメント[^1] に記載の内容をもとに補足を加えながら実際に試してみます。

[^1]: https://learn.microsoft.com/ja-jp/azure/virtual-machines/windows/scheduled-event-service

# 環境準備

## Scheduled Events をイベントログに載せる
検証用に Windows Server 2022 の VM を作成します。その VM 上に、先述のドキュメントでも参照されているこちらの GitHub プロジェクト[^2] の Zip ファイルを展開します。

[^2]:https://github.com/microsoft/AzureScheduledEventsService

![](/images/20240221-scheduled-event/scheve-01.png)

プロジェクト内の `Powershell` フォルダに格納されている `SchService.ps1` を実行していきます。

外部からダウンロードしたファイルのため、必要に応じて実行ポリシーを変更します。

```powershell
PS> Set-ExecutionPolicy Unrestricted
```
実行ポリシーを変更したうえで、以下のような流れで設定していきます。

```powershell
# setup service
PS> .\SchService.ps1 -Setup
[D] Do not run  [R] Run once  [S] Suspend  [?] Help (default is "D"): R

# start service
PS> .\SchService.ps1 -Start
[D] Do not run  [R] Run once  [S] Suspend  [?] Help (default is "D"): R

# Check status
PS> .\SchService.ps1 -Status
Running
```

状態が `Running` となっていることを確認出来たら、Azure portal から VM を**再デプロイ**します。
![](/images/20240221-scheduled-event/scheve-02.png)


すると、次のようなイベントが Scheduled Events の計画に入ります。

```powershell
# Instance Metadate Service に問い合わせてキューに入っているイベントを確認
PS C:\Users\AzureAdmin\Desktop\Powershell> Invoke-RestMethod -Headers @{"Metadata"="true"} -Method GET -Uri "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01" | ConvertTo-Json -Depth 64
{
"DocumentIncarnation":  1,
"Events":  [
{
"EventId":  "A9A902A0-B0EA-4E7C-B970-E734ECF66A35",
"EventStatus":  "Scheduled",
"EventType":  "Redeploy",
"ResourceType":  "VirtualMachine",
"Resources":  [
"win-scheve-eus"
],
"NotBefore":  "Wed, 21 Feb 2024 03:08:47 GMT",
"Description":  "Virtual machine is going to be redeployed as requested by authorized user.",
"EventSource":  "User",
"DurationInSeconds":  -1
}
]
}
```

VM の再デプロイ後、再度 VM に接続し、イベントログを確認します。先ほどの `Redeploy` イベントが、イベントID `1234` で確かに記録されています。

![](/images/20240221-scheduled-event/scheve-03.png)

## 対象のログを Log Analytics ワークスペースに飛ばす
イベントログに書き込まれることは確認できたため、あとは通常通り Azure Monitor と連携して Azure Monitor Logs (Log Analytics ワークスペース)に同期します。

データ収集ルール(DCR) を作成します。先の検証から、Application ログに書き込まれることは分かっているため、例えば以下のように Application ログを収集するように設定します。

![](/images/20240221-scheduled-event/scheve-04.png)

設定が完了したら、改めて VM を再デプロイします。しばらく待つと、先ほどサーバー上で確認した `EventID==’1234’` のログが記録されます。

![](/images/20240221-scheduled-event/scheve-05.png)

## アラートの発報
ここまでで、VM 上の Scheduled Events がログとして記録されることを確認しました。最後に、そのログが発生したタイミングでアラートが飛ぶように設定します。

Azure Monitor からアラートルールを作成します。
![](/images/20240221-scheduled-event/scheve-06.png)

ログに対するクエリの結果に対してアラートを発報するため、[カスタムログ検索] でルールを作成します。
![](/images/20240221-scheduled-event/scheve-07.png)

最小の 1 分単位で評価し、1 件でもログに記録されていればアラートを上げる設定としてみます。
![](/images/20240221-scheduled-event/scheve-08.png)

さらに、次のように設定します。作成されたマネージドIDについては、監視対象に関するログが取得できる権限を付与しておきます。
![](/images/20240221-scheduled-event/scheve-09.png)


アラートルール一覧から対象のアラートルールを開き、[ID] > [Azure ロールの割り当て]を選択します。
![](/images/20240221-scheduled-event/scheve-10.png)


対象リソースグループに対して「閲覧者」権限を付与します。
![](/images/20240221-scheduled-event/scheve-11.png)

:::message

マネージド ID に必要な権限についてはこちらのドキュメント[^3] をご参照ください。今回はリソースコンテキストで閲覧権限を与えています。

[^3]:https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/manage-access?tabs=portal#compare-access-modes

:::

アラート発報時に電子メールを送信するアクショングループも設定しておきます。
![](/images/20240221-scheduled-event/scheve-12.png)

改めて、VM の再デプロイをしてみましょう。Azure portal で Azure Monitor のアラート画面を開くと、実際に発報されたアラートを確認できます。
![](/images/20240221-scheduled-event/scheve-13.png)

また、このような形でメールが届きます。これで、Scheduled Event が発生した際に手元で確認できますね。
![](/images/20240221-scheduled-event/scheve-14.png)

# おわりに
実際にアラートが発砲されてメールが届くまで、猶予 15 分のうちの5分程度は使ってしまうと思います。残った 10 分前後の間にできることは限られてしまうかもしれませんが、自動化されたアクションを定義しておくことでより安全な運用につなげることも可能かもしれませんね。

Scheduled Events についてもう少し知りたい方はこちら[^4] をご参照ください。

[^4]: https://learn.microsoft.com/ja-jp/azure/virtual-machines/windows/scheduled-events
