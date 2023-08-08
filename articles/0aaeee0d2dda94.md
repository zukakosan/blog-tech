---
title: "Log Analyticsの変換を利用してAzureにインジェストするログを絞って課金を抑える"
emoji: "💸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","log","AzureMonitor","monitoring"]
published: true
publication_name: "microsoft"
---
# モチベ
- データ収集ルール(Data Collection Rule/DCR)でAzure VM/オンプレミスマシンからデータを収集する際、DCR上ではあまり細かいルールを作ることはできず、不要なログを過剰に取得しすぎてしまうことがある
![](/images/20230808-LogAnalyticsTransform/01.png)

- DCRの中で[カスタム]を選択し、XPathクエリによって収集するデータを絞るというやり方も1つあるのだが、今回は馴染みのあるKQLでやってみたいと思い、Log Analyiticsの変換を試してみる

https://learn.microsoft.com/ja-jp/azure/azure-monitor/agents/data-collection-rule-azure-monitor-agent?tabs=portal#filter-events-using-xpath-queries

:::message
XPath：XML形式の文書から、特定の部分を指定して抽出するための簡潔な構文

https://www.w3schools.com/xml/xpath_syntax.asp
:::

# Log Analyticsの変換とは？
- Log Analyiticsのテーブルにデータをインジェストする前にKQLによるデータ加工ができるというもの
- 本記事のタイトルではコスト削減をテーマにしているが、他にもいくつかの用途が考えられる

|カテゴリ|詳細|
| --- | --- |
|機密データを削除する|機密情報をフィルター処理：機密情報を含む行全体、または特定の列を除外できます。<br><br>機密情報を難読化：IP アドレスや電話番号の数字を共通の文字に置き換えます。<br><br>別のテーブルに送信：ロールベースのアクセス制御構成が異なる別のテーブルに機密レコードを送信します。|
|詳細情報や計算された情報を使用してデータを強化する|詳細情報を含む列を追加：たとえば、別の列内の IP アドレスが内部のものか外部のものかを識別する列を追加することもできます。<br><br>ビジネス固有の情報を追加：たとえば、他の列内の位置情報に基づいて会社の部門を示す列を追加することもできます。|
|データ コストを削減する|**行全体を削除**：たとえば、特定のリソースからリソース ログを収集する診断設定があるが、生成されるすべてのログ エントリが必要であるわけではない場合があります。 特定の条件に一致するレコードを除外する変換を作成します。<br><br>**各行から列を削除**：たとえば、データには、冗長なデータや最小値データを含んだ列が含まれている場合があります。 そのような場合は、不要な列を除外する変換を作成できます。<br><br>列内の重要なデータを解析：貴重なデータが特定の列に埋もれているテーブルがある場合があります。 変換を使用すると、重要なデータを解析して新しい列を生成し、元の列を削除できます。<br><br>特定の行を基本ログに送信：基本的なクエリ機能を必要とするデータ内の行を基本ログ テーブルに送信することで、インジェスト コストを削減できます。|

https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/data-collection-transformations

# コスト削減としての変換を試す
シナリオを「すべてのWindows Event Logは過剰なので、特定のイベントのみインジェストするように変換をかけたい場合」と想定(これはXPathでやった方がサクッとできるかもしれないという突っ込みは置いておいて)
## 環境の構築
- Azure Windows VMからEvent Logを収集するLog Analyiticsの環境を用意しておく

https://learn.microsoft.com/ja-jp/azure/azure-monitor/vm/monitor-virtual-machine

- とりあえずEvent LogがLog Analyticsに来ていることを確認する
![](/images/20230808-LogAnalyticsTransform/02.png)

## 想定クエリの検証
- 実際にインジェスト前にかけたいフィルタ・加工をするために必要なKQLをLog Analyitics上で試しておく
- 今回は特定のEventIDのものに絞るだけのシンプルなフィルタだが、**コスト削減的な観点ではいらないカラムを落としたりするともっと効果的**
![](/images/20230808-LogAnalyticsTransform/03.png)

## DCRへのKQLの組み込み
- DCRのリソースを開き、[テンプレートのエクスポート]から[デプロイ]を選択
![](/images/20230808-LogAnalyticsTransform/04.png)

- [テンプレートの編集]から編集していく
- 今回は、下記のような式に落ち着いたが、Log Analyitics上で動作するクエリとここの`transformKql`プロパティで動作するクエリに若干の差があるため、後述するGUIのやり方で検証するのがおすすめ

```
source | where EventID in (int(4673),int(4674))
```
- `transformKql`にクエリを挿入
![](/images/20230808-LogAnalyticsTransform/05.png)

:::message
- ARMテンプレートで使われるオペレータに競合する場合や、KQLの中で文字列を使用する場合、特殊文字を必要とする場合は適宜エスケープの処理などによってエラーを回避する必要がある
- この辺も、GUIの手順であれば検証できる
:::

- テンプレートを保存して、そのままインクリメンタルにデプロイしていく
![](/images/20230808-LogAnalyticsTransform/06.png)

- デプロイ後、JSONビューを開き、`transfromKql`にきちんとクエリが入っていることを確認
![](/images/20230808-LogAnalyticsTransform/07.png)

## 動作検証
- 暫く放置したのち、Log Analyitics上で変換されたデータが収集されていることを確認する
- `Event`テーブル全体を抽出しているにもかかわらず、変換によって抜き出されたログ(EventID=4673,4674)のみ入ってきていることが確認できる
![](/images/20230808-LogAnalyticsTransform/08.png)

# おまけ：GUIでの変換クエリの検証
- Log Analytics側で変換を作成することで、GUIをベースに進めることができる
    - 実体としては、新規のDCRを作るような動きになる
- 変換を作成したいテーブルの[・・・]から[変換の作成]をクリック
![](/images/20230808-LogAnalyticsTransform/09.png)

- [変換エディター]から、どの様なクエリをかけるか動的検証が可能
    - **Log Analyitics上で実行できるクエリと`transfromKql`で指定できるクエリに差がある**ため、こちらでのクエリの作りこみが推奨
    - Log Analyticsでは実行できたがエラーになった図
    ![](/images/20230808-LogAnalyticsTransform/10.png)
    - `in`オペレータの左右で型が違うぞ、と言われているので、かっこ悪くもすべてに`int`を付けたら解決
    ![](/images/20230808-LogAnalyticsTransform/11.png)

こんな感じで、先のARMてんぷれーとにねじこめる(エラーが出ない)式を取得できる

## そのままDCRを作成すると？
- ちなみに、そのままデータ収集ルールを作成すると`Microsoft-Table-Event`をデータソースとして持つだけのDCRが作成される。特に編集はできない。
- ややこしいのでデータ"変換"ルール、とか別のリソース扱いにしてほしいところではあるのだが…
![](/images/20230808-LogAnalyticsTransform/12.png)

- リソースは空になっている
![](/images/20230808-LogAnalyticsTransform/13.png)

- JSON定義を見てみると、`transformKql`にKQL式が入力されている
    - なので、テーブルに格納する前にそのKQLによるフィルタがかかるだけっていう動作イメージだろうという感じ
- たしかに、やっていることはいわゆる正規のDCRで記述した(`transformKql`)ことと同じことを別のDCRとして実行するっていうだけになる
![](/images/20230808-LogAnalyticsTransform/14.png)

イメージ的には「正規のDCRによるVMからのデータ収集>正規のDCRの中での`transformKql`>テーブル側でインジェストする前の変換用DCRの中での`transformKql`」的な感じだろうと解釈

# コスト
- この変換の処理によって、課金が発生する可能性がある
    - 加工によってデータが増えるとき：当然増えた分は取り込み料金がかかる
    - めっちゃデータを削減したとき：削減率50％を超えた場合にその超えた分だけデータ処理料金がかかる
        - Docsの例：20GBのデータの12GBを削減した場合、50%の10GBを超えた2GB分はデータ処理料金が発生

> 変換そのものには直接コストは発生しませんが、次のシナリオでは追加料金が発生する可能性があります。
たとえば計算列の追加のように、変換によって受信データのサイズが大きくなる場合は、その追加データに対して標準の取り込み料金が課金されます。
取り込まれたデータが変換によって 50% を超えて削減された場合、50% を超えるフィルター処理されたデータ量に対して課金されます。

https://learn.microsoft.com/ja-jp/azure/azure-monitor/essentials/data-collection-transformations#cost-for-transformations

# おわり
- XPathとLog Analyiticsの変換、どっちが推奨かというと難しい
- ただ、複雑なことをするのであればKQLが自由に書けるLog Analyiticsの変換じゃないと機能的に実現できないこともありそう
