---
title: "Azure AI Search の既存インデックスにベクトルフィールドを追加し、AOAI から利用する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","aoai","rag","ai"]
published: false
---

# はじめに
Azure Open AI Service (AOAI) を利用して、社内ドキュメント検索等のシナリオにおいて Retrieval-Augumented Generation (RAG) アーキテクチャを採用する場合、Azure AI Search を利用することが多いと思います。最近では、GUI で [データのインポートとベクター化] というメニューも選択できるようになっており[^1]、ベクトル化込みのインデックスを取得すること自体へのハードルは下がっています。
一方で既存のインデックスに対して埋め込みベクトルを手組で追加するようなナレッジはあまり見当たらず、試行錯誤することになったためここにまとめておきます。

[^1]: https://qiita.com/tmiyata25/items/ef231ea340e681a44970

# 前提
こちら[^2]で提供されているハンズオンの構成をベースとしています。画像データを複数 BLOB ストレージに保存し、そのデータを Azure AI Search 側でデータソースとし、インデックスを作成しています。この時点ではまだベクターフィールドを持たない状況です。
[^2]: https://github.com/nohanaga/Azure-AI-Search-Workshop/blob/main/CreateIndex.md

## BLOB ストレージへの画像の格納
BLOB ストレージには以下のような画像データを入れています。
:::details 入力画像サンプル
OCR を試すために敢えてスクリーンショットの画像を追加しています。
![](/images/20240211-aisearch-add-vector/aoai-gen.png)
![](/images/20240211-aisearch-add-vector/aoai-func.png)
![](/images/20240211-aisearch-add-vector/aoai-limit.png)
![](/images/20240211-aisearch-add-vector/aoai-responsible.png)
:::

## Azure AI Search でのインデックス作成
Azure AI Search 初期構築は具体的には以下のような設定で行っています。
<!-- :::details Azure AI Search の構築手順 -->
データのインポート
![](/images/20240211-aisearch-add-vector/ais-01.png)
OCR 含むスキルセットの構成
![](/images/20240211-aisearch-add-vector/ais-02.png)
インデックスのカスタマイズ (ハンズオン資料より簡素化しています。)
![](/images/20240211-aisearch-add-vector/ais-03.png)
データのインポート
![](/images/20240211-aisearch-add-vector/ais-04.png)
OCR スキルの言語を `ja` に変更
![](/images/20240211-aisearch-add-vector/ais-05.png)
インデクサーの実行
![](/images/20240211-aisearch-add-vector/ais-06.png)
<!-- ::: -->