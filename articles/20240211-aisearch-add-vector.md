---
title: "Azure AI Search の既存インデックスにベクトル表現(埋め込み)を追加し、AOAI から独自データとして参照する"
emoji: "🍩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","aoai","rag","ai"]
published: true
publication_name: "microsoft"
---

# はじめに
Azure OpenAI Service (AOAI) を利用して、社内ドキュメント検索等のシナリオにおいて Retrieval-Augumented Generation (RAG) アーキテクチャを採用する場合、Azure AI Search を利用することが多いと思います。最近では、GUI で「データのインポートとベクター化」というメニューも選択できるようになっており[^1]、ベクトル化込みのインデックスを取得すること自体へのハードルは下がっています。
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
:::details Azure AI Search の構築手順
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
:::

## 生成されたインデックスの確認
この時点では、インデックスにベクトル フィールドを追加していないため、初期構築時のスキルセットに応じて、インデックスが張られています。

とりあえず、日本語の OCR 含めて情報は抜き出せていそうな状況です。
![](/images/20240211-aisearch-add-vector/ais-07.png)

# インデックスに対するベクトル フィールドの追加
## ベクター プロファイルの作成
インデックスが作成されていることを確認したので、本題のベクトル フィールドを作成していきます。
対象のインデックスに対して、「ベクター プロファイル」を追加します。「アルゴリズムの構成」や「ベクター化」もあわせて作成します。画面に従っていけば作成できます。
![](/images/20240211-aisearch-add-vector/vec-01.png)

## インデックス フィールドの追加
AOAI の埋め込みモデルによってベクトル化したデータを保存するためのフィールドを追加します。「フィールド」>「フィールドの追加」からインデックス フィールドを追加します。型は `Collection(Edm.Single)`、ディメンションは `1536`、ベクター プロファイルは先ほど作成したものを選択します。
![](/images/20240211-aisearch-add-vector/vec-02.png)
:::message alert
執筆時点では、ディメンションに `1536` 以外を設定すると検索時にエラーとなったので注意してください。
:::

# 埋め込みスキルの追加
[先の手順](#インデックス-フィールドの追加)では、ベクトルに埋め込んだデータを保存するための箱として、フィールドを作成しました。

テキストをベクトルに埋め込むためには、スキルとして AOAI による埋め込みを追加する必要があります。よって、初期構築時に作成したスキルセットに対してせってを加えていきます。

OCR した結果のテキストは `merged_content` に格納されます。そこに対してベクトル化のスキルを適用するため、以下のように設定しています。

![](/images/20240211-aisearch-add-vector/emb-01.png)

:::message
このスキル間でのデータの橋渡しがなかなか癖があり、少し苦労しました。

上記の例では、`#Microsoft.Skills.Text.AzureOpenAIEmbeddingSkill` に渡す `input` として、`/document/merged_content` を指定し、埋め込みの結果を `kedama_vector` という名前で出力しています。
:::

# インデックス フィールドへのマッピング
`#Microsoft.Skills.Text.AzureOpenAIEmbeddingSkill` による出力を実際のインデックス フィールドにマッピングし、検索可能な状態にします。`/documents/kedama_vector` に出力されたものを、[フィールドの追加](#インデックス-フィールドの追加) 手順で追加したインデックス上のフィールドに対応させます。変更後、インデクサーをリセットし、再実行します。

![](/images/20240211-aisearch-add-vector/index-01.png)

# インデックスの確認
対象のインデックスを見てみると、データ量が増え、埋め込みベクトルも追加されていることを確認できます。
![](/images/20240211-aisearch-add-vector/idx-ch-01.png)

このようにして、手組みで既存のインデックスに対してベクトル表現を追加することは一応可能です。JSON をごにょごにょするのが結構大変なので、初めから「データのインポート及びベクター化」をした方がいいかもしれませんね。

# AOAI から Add Your Data としての呼び出し
RAG システム構築の上で AOAI から呼び出せるか確認するため、AOAI のプレイグラウンドを使用して検証します。

## データの追加
「データの追加」から対象の Azure AI Search のリソースおよびそのインデックスを選択します。
検索時クエリもベクトル化が必要になるため、埋め込みモデルを選択します。
また、カスタム フィールドマッピングも使用します。
![](/images/20240211-aisearch-add-vector/ayd-01.png)

フィールドのマッピングとしては一旦以下とします。ファイル名に[初期構築時](#azure-ai-search-でのインデックス作成)に作成したフィールドである `metadata_storage_name` を指定して、情報ソースを明らかにします。
![](/images/20240211-aisearch-add-vector/ayd-02.png)

検索の種類としては、「ハイブリッド(ベクトル＋キーワード)」を選択します。
![](/images/20240211-aisearch-add-vector/ayd-03.png)

内容を確認して保存します。
![](/images/20240211-aisearch-add-vector/ayd-04.png)

## チャットによる確認
チャットプレイグラウンドで入力したデータに記載されていそうな内容について聞いてみます。それっぽい回答が返ってきていることを確認できました。
![](/images/20240211-aisearch-add-vector/chat-01.png)

チャンクをそのまま持ってきているようにも見えたので、文字数を制限してみました。よさそうですね。
![](/images/20240211-aisearch-add-vector/chat-02.png)


# おわり
Azure AI Search 上の既存のインデックスに対して埋め込みベクトルを追加し、AOAI 側から呼び出すことで RAG として動くことを確認しました。とはいえ JSON で Azure AI Search 上の設定を変更していくのが煩雑なので、可能であれば「データのインポートとベクター化」で楽をするのがよさそうだという所感です。

とはいえ、Azure AI Search 上でスキルセットを自由に触れるようになると、AOAI 文脈以外にも幅は広がるので、これを機にいろいろ試してみるのもよさそうですね(自戒)。
