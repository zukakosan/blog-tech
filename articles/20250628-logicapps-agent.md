---
title: "Azure Logic Apps の Agent Workflow を使って AIOps を加速する"
emoji: "🥷🏻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Agent","ai","azure"]
published: true
publication_name: "microsoft"
---
# はじめに
Microsoft Build 2025 にて Azure Logic Apps の Agent 機能が発表されました。Logic Apps 内のワークフロー内で、Tool を使う Agent を構成できるというもので、Agent を使ったワークフローが非常に簡単に作れます。AIOps 待ったなし、という匂いがプンプンしています。

詳細は下記のセッションで確認できます。
https://www.youtube.com/watch?v=4PCotbZjXPc

本記事では、MS Learn に記載の内容をもとに実際に検証を行い、動きを確認します。

# リソース作成＆設定
## Azure OpenAI Service (AOAI)
Azure OpenAI Service をデプロイし、モデルをデプロイしておきます。現在、Agent Workflow では以下のモデルに対応していますが、モデルの比較[^1] を参考に、`GPT-4o` あたりにしておきます。
- **gpt-4.1**
- **gpt-4.1-mini**
- **gpt-4.1-nano**
- **gpt-4o**
- **gpt-4o-mini**
- **gpt-4**
- **gpt-35-turbo**

![](/images/20250628-logicapps-agent/01.png)
![](/images/20250628-logicapps-agent/02.png)

[^1]: https://llm-stats.com/

## Logic Apps
Logic Apps は従量課金ではなく `Standard` を選択する必要があります。

### マネージド ID の有効化
Standard SKU の Logic Apps では、既定でマネージド ID が有効化されていますが、念のため確認します。
![](/images/20250628-logicapps-agent/03.png)

### マネージド ID への権限付与
最小権限として、AOAI 上のモデルを呼び出せればよいので、`Cognitive Services OpenAI User` を付与します。別のリソースグループに存在する AOAI 上のモデルを触る可能性もあるため、サブスクリプション スコープにしていますが適宜調整してください。
![](/images/20250628-logicapps-agent/04.png)

# Agent Workflow の作成
Logic Apps > ワークフロー から `Agent` を選択して作成します。
![](/images/20250628-logicapps-agent/05.png)

## HTTP Trigger
トリガーとしては、設定項目が少なくてシンプルな `When a HTTP request is received` を使います。
> この特定のトリガーは、ワークフロー外の呼び出し元から送信された HTTPS 要求の受信を待機します。 これらの要求を使用して、トリガーがワークフロー入力として受け入れるデータを含めることができます。 トリガーはこのデータを出力として渡し、後続のアクションはワークフローで使用できます。

## Agent の設定
Agent のブロックをクリックし、AOAI モデルと接続します。最初は何も表示されないため、Logic Apps から AOAI リソースへのコネクションを作成します。これにより、そのリソース上のモデルを使用できるようになります。`Authentication Type` は `Managed Service Identity` としましょう。これにより、マネージド ID ベースで接続できます。
![](/images/20250628-logicapps-agent/06.png)

モデルが選択出来たら、プロンプトも入力しておきましょう。
![](/images/20250628-logicapps-agent/07.png)

## Tool の追加
Agent に接続した LLM が使用するツールをここで定義します。ツールの中に入れ子のワークフローを組めるので、すでに Logic Apps で実装しているワークフローも１つのツールとして登録できます。ここでは、`Get current weather`[^2]というコネクタを使います。
> MSN Weather gets you the very latest weather forecast, including temperature, humidity, precipitation for your location.

![](/images/20250628-logicapps-agent/08.png)

このワークフローを実現する重要なポイントとして、`Get current weather` コネクタの `Location` 変数には、Agent Parameter と呼ばれるパラメータを使用しています。ドキュメントに動作の詳細は記載されていませんが、エージェントに渡された文脈 (Input) から動的に Tool に与えるべき Input を抽出するという機能です（という認識）。そのため、`Type` と `Description` を設定する必要があります。

以前では、渡すデータの処理用の AOAI のモジュールを使うといった工夫が必要だったポイントが Agent 内部、そして Tool 側の設定として指定できるのはかなり楽ですね。個人的にはこの機能はかなり強力に感じます。

![](/images/20250628-logicapps-agent/09.png)
また、この Agent Parameter ですが、以下のような特徴があります。
- エージェント パラメーターは、定義したツールにのみ適用されます。 この制限は、エージェント パラメーターを他のツールと共有できないことを意味します。 これに対して、ワークフロー内の操作および制御フロー構造と従来のパラメーターをグローバルに共有できます。
- ワークフローの実行を開始するときに、エージェント パラメーターに解決された値がありません。 エージェント パラメーターは、エージェントが特定の引数を使用してツールを呼び出した場合にのみ値を受け取ります。 これらの引数は、ツールを呼び出すためのエージェント パラメーターになります。
- エージェントは、同じループ イテレーション内に同じツールが存在する場合でも、異なるエージェント パラメーター値で同じツールを複数回呼び出すことができます。 たとえば、ツールはシアトルとロンドンの両方で天気を確認できます。


[^2]: https://learn.microsoft.com/en-us/connectors/msnweather/

# ワークフローの実行
HTTP のリクエストを裁くために、Parse JSON も入れておきます。想定として以下のようなシンプルなスキーマが飛んでくるとします。
```json
{
  "request":"今日の東京の天気は？"
}
```

これをサンプルスキーマとして使用すると、以下のようになります。
![](/images/20250628-logicapps-agent/10.png)
```json
{
  "type": "object",
  "properties": {
    "request": {
      "type": "string"
    }
  }
}
```

`Run with payload` から、投げてみます。実行履歴を確認すると、きちんと Tool が使われていることがわかります。また、Tool への `INPUTS` の `Location` が「東京」として抽出されています。
![](/images/20250628-logicapps-agent/12.png)

# Teams Chat への投稿
ちょっとした運用イメージですが、最終的にはユーザに通知することも考慮します。HTTP で受け取って Teams で返すという非対称ルーティング的ですが、Timer Trigger のようなイメージで思ってもらえればと。
![](/images/20250628-logicapps-agent/13.png)

同じようにリクエストを投げると、Teams 側に投稿されてきます。
![](/images/20250628-logicapps-agent/14.png)

# おわりに
以上のような形で、従来通りの Logic Apps の使い勝手はそのままに、AI Agent を含めたワークフローを簡単に作成できます。Agent Parameter も非常に使い勝手がよく、AIOps による運用改善を考える組織においては非常に強力な手段になることは間違いないと思います。