---
title: "Microsoft Defender for Cloud のセキュリティ アラートで Logic Apps のフローをトリガーして自動アクションを実行する"
emoji: "🔫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","defender","microsoft","automation","logicapps"]
published: true
published_at: 2024-12-9 09:00
---
# はじめに
Azure における CSPM/CWPP といえば、Microsoft Defender Cloud が挙げられます。この統合コンソールを通して、IT 管理者はセキュリティやリソース構成の観点から全体を把握できます。

MDfC は推奨事項やセキュリティ アラートをこのコンソールの随所に提示をしてくれるのですが、その変化にいち早く気付き、場合によっては自動対応までを行うことが求められます。

実は、MDfC には Logic Apps との連携機能が組み込まれており、アラートなどをトリガーにしてフローを実行できます。うまく使いこなすと単なるアラート以外の処理も行えるため、プレイ ブックに沿った対応が自動で構成できる可能性があります。

今回はその設定の流れを確認します。

# 設定方法
Microsoft Defender for Cloud のを開き、設定していきます。
![](/images/20241208-mdfc-logicapps/mdfc01.png)

## 自動ワークフローの構成
[Workflow automation] > [+ workflow automation] から作成していきます。`Trigger conditions` には `Security alert` を設定します。
![](/images/20241208-mdfc-logicapps/mdfc02.png)

[Actions] では、トリガーするフロー用の Logic Apps が必要なため、[visit the Logic Apps page] をクリックします。Logic Apps のページから [+ Add] をクリックし、`Consumption` を選択の上、任意のパラメータを入れて作成します。
![](/images/20241208-mdfc-logicapps/mdfc03.png)

完了後、作成した Logic Apps のページへアクセスし、[Overview] の [Get Started] から、[Choose template] を選択します。
![](/images/20241208-mdfc-logicapps/mdfc04.png)

ここで、[When a HTTP request is received] を選択します。
![](/images/20241208-mdfc-logicapps/mdfc05.png)

Logic Apps デザイナーが開いたら、一旦 [Save] をして MDfC の画面に戻ります。
![](/images/20241208-mdfc-logicapps/mdfc06.png)

先ほどの、[Actions] の項目で [Refresh] をクリックし、作成した Logic Apps リソースを選択します。そのまま、[Create] します。
![](/images/20241208-mdfc-logicapps/mdfc07.png)

すると、MDfC にワークフローが追加されたことが確認できます。
![](/images/20241208-mdfc-logicapps/mdfc08.png)

## サンプル アラートの生成
続いて、Security Alert の画面を開き [Sample alerts] からサンプルのアラートを作成します。自動化ワークフローでスコープに設定されているサブスクリプションを選択し、アラートを発生させます。
![](/images/20241208-mdfc-logicapps/mdfc09.png)

サンプル アラートは以下のように生成されます。
![](/images/20241208-mdfc-logicapps/mdfc10.png)

ここで、Logic Apps リソースを開き、[Run history] を見てみると、大量に実行履歴が残っていることが分かります。
![](/images/20241208-mdfc-logicapps/mdfc11.png)

実行履歴のうちの1つをクリックし、中身を見てみると、確かに MDfC からのものであるとわかります。
![](/images/20241208-mdfc-logicapps/mdfc12.png)

さらに、Show raw outputs から詳細に中身を見てみると、`AlertDisplayName` や `Description` といった内容が確認できます。

```
(省略)
...
"AlertDisplayName": "[SAMPLE ALERT] Unusually large response payload transmitted between a single IP address and an API endpoint",
"Description": "THIS IS A SAMPLE ALERT: A suspicious spike in API response payload size was observed for traffic between a single IP and one of the API endpoints. Based on historical traffic patterns from the last 30 days, Defender for APIs learns a baseline that represents the typical API response payload size between a specific IP and API endpoint. The learned baseline is specific to API traffic for each status code (e.g., 200 Success). The alert was triggered because an API response payload size deviated significantly from the historical baseline. \n\n",
...
(省略)
```
このように、アラートを起点にして Logic Apps のフローをトリガーできることと、その中身のスキーマが分かったため、Logic Apps に簡単なフローを追加してもう一度試してみます。

## Logic Apps のフロー作成
ここからは、Logic Apps の設定の話になります。MDfC のセキュリティ アラートをベースに実行したい内容を自由に作っていけばよいです。ここでは例として、メールを飛ばしてみます。

Gmail の [Send email] コンポーネントを配置し、接続名を入れたうえで Gmail アカウントでサインインします。そのアカウントから、任意の宛先にメールを通知できるようになります。つまり、管理用のアカウントがあれば、そこから担当者にメールを飛ばすことができます。

メッセージの `Body` には、動的なコンテンツとして、[manual コンポーネント] の `output` の `body` をとりあえずそのまま入れています。
![](/images/20241208-mdfc-logicapps/mdfc13.png)


保存して、[Run with payload] からテストをします。`Body` には、実行履歴からコピーした Body のテキストをそのままペーストします。
![](/images/20241208-mdfc-logicapps/mdfc14.png)

すると、確かにサイン インした Gmail アカウントからメールが飛んで来ることを確認できます。
![](/images/20241208-mdfc-logicapps/mdfc15.png)

## フローで扱うデータの体裁の調整
あとは作り込みの問題なので、ここまででも挙動確認としては十分ですが、もう少し体裁を整えます。

[manual] と [Send email] の間に [Parse JSON] というデータ操作のコンポーネントを挟みます。`Content` には 動的コンテンツとして [manual] の `Body` を入れ、スキーマはサンプル ペイロードから生成します。
![](/images/20241208-mdfc-logicapps/mdfc16.png)

サンプルのペイロードは、実行履歴からとってきたものを貼り付けます。
![](/images/20241208-mdfc-logicapps/mdfc17.png)
