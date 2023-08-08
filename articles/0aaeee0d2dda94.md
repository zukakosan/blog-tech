---
title: "Log Analyticsの変換を利用してAzureにインジェストするログを絞って課金を抑える"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","log","AzureMonitor","monitoring"]
published: false
---
# モチベ
- データ収集ルール(Data Collection Rule/DCR)でAzure VMからデータを収集する際、DCR上ではあまり細かいルールを作ることはできず、不要なログを過剰に取得しすぎてしまうことがある
![](/images/20230808-LogAnalyticsTransform/01.png)

- DCRの中で[カスタム]を選択し、XPathクエリによって収集するデータを絞るというやり方も1つあるのだが、今回は馴染みのあるKQLでやってみたいと思い、Log Analyiticsの変換を試してみる

https://learn.microsoft.com/ja-jp/azure/azure-monitor/agents/data-collection-rule-azure-monitor-agent?tabs=portal#filter-events-using-xpath-queries

:::message
XPath＝XML形式の文書から、特定の部分を指定して抽出するための簡潔な構文

https://www.w3schools.com/xml/xpath_syntax.asp
:::

# Log Analyticsの変換とは？
- Log Analyiticsのテーブルにデータをインジェストする前にKQLによるデータ加工ができるというもの
- 本記事のタイトルではコスト削減をテーマにしているが、他にもいくつかの用途が考えられる

|カテゴリ|詳細|
| --- | --- |
|機密データを削除する|機密情報をフィルター処理：機密情報を含む行全体、または特定の列を除外できます。<br>機密情報を難読化：IP アドレスや電話番号の数字を共通の文字に置き換えます。<br>別のテーブルに送信：ロールベースのアクセス制御構成が異なる別のテーブルに機密レコードを送信します。|
|詳細情報や計算された情報を使用してデータを強化する|詳細情報を含む列を追加：たとえば、別の列内の IP アドレスが内部のものか外部のものかを識別する列を追加することもできます。<br>ビジネス固有の情報を追加：たとえば、他の列内の位置情報に基づいて会社の部門を示す列を追加することもできます。|
|データ コストを削減する|行全体を削除：たとえば、特定のリソースからリソース ログを収集する診断設定があるが、生成されるすべてのログ エントリが必要であるわけではない場合があります。 特定の条件に一致するレコードを除外する変換を作成します。<br>各行から列を削除：たとえば、データには、冗長なデータや最小値データを含んだ列が含まれている場合があります。 そのような場合は、不要な列を除外する変換を作成できます。<br>列内の重要なデータを解析：貴重なデータが特定の列に埋もれているテーブルがある場合があります。 変換を使用すると、重要なデータを解析して新しい列を生成し、元の列を削除できます。<br>特定の行を基本ログに送信：基本的なクエリ機能を必要とするデータ内の行を基本ログ テーブルに送信することで、インジェスト コストを削減できます。|

https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/data-collection-transformations

# 

# コスト
- この変換の処理によって、課金が発生する可能性がある