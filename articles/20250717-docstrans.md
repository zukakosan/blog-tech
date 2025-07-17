---
title: "Logic Apps + Azure AI Translator でドキュメントをまとめて翻訳する"
emoji: "🐯"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","ai","translation"]
published: true
publication_name: "microsoft"
---

# はじめに
近年の AI 活用の流れから、社内のドキュメント整備が改めて見直されています。グローバル化が進む中で、ドキュメントも多言語対応が求められています。

## Azure AI Translator
Azure には、Azure AI Services のサービス ファミリの中に、Azure AI Translator[^1] というサービスが存在します。このサービスを使うと、テキストの翻訳、ドキュメントの直接翻訳、カスタムモデルを使用した翻訳が可能です。ドキュメントの翻訳は、`.docx`や`.pdf` をはじめとして多くのファイル形式に対応しています。[^2]

[^1]: https://learn.microsoft.com/en-us/azure/ai-services/translator/
[^2]: https://learn.microsoft.com/en-us/azure/ai-services/translator/document-translation/overview#batch-supported-document-formats

## やりたいこと
すでに蓄積されている社内のドキュメントをバッチ処理的に非同期で適宜翻訳することを考えます。

# 構成
中心となるリソースは以下です。
- Azure AI Translator
    - ドキュメントの翻訳に使用
- Azure Storage Account
    - 翻訳前、翻訳後のドキュメント保存場所として利用
    - Azure Logic Apps を作成するとワークフローの状態管理等に使われる Storage Account が作成されますが、ネットワーク制御の都合上、別で用意するのが推奨
- Azure Logic Apps
    - ドキュメント翻訳を含むフローの実行に使用 

## Azure AI Translator に対するマネージド ID の設定
Azure AI Translator が Azure Storage Account のリソースを触る必要があるため、マネージド ID を有効化します。
![](/images/20250717-docstrans/01.png)

RBAC として、`Storage BLOB Contributor` を割り当てます。
![](/images/20250717-docstrans/02.png)

## Azure Storage Account に Source フォルダと Target フォルダを作成
翻訳対象のドキュメント格納用のフォルダ、翻訳後のドキュメント格納用のフォルダを作成します。また、対象のドキュメントは格納しておきます。
![](/images/20250717-docstrans/03.png)
![](/images/20250717-docstrans/04.png)
![](/images/20250717-docstrans/05.png)

## Azure Storage Account のネットワーク設定
パブリックアクセスを許可したくない場合は、`Enabled from selected virtual networks and IP addresses`を選択したうえで、`Allow Azure services on the trusted services list to access this storage account.`を有効にします。この設定により、Translator が trusted service として、Storage にアクセスできるようになります。
![](/images/20250717-docstrans/06.png)

## Azure Logic Apps のフロー作成
Standard のプランでワークフローを作成します。
![](/images/20250717-docstrans/07.png)

トリガーとしては手動トリガーとも呼ばれる `When a HTTP request is received` を使います。Agentic Workflow は使わないのでいったん削除します。

トリガーに続くアクションとしては、`Microsoft Translator v3` の `Start document translation` を利用します。AI Translator のリソース名とキーを入力し、コネクションを作成します。

パラメータとしては以下のような内容を設定します。言語についてはドキュメントに合わせます。設定したらワークフローを保存します。
- `Storage type of the input documents`: `Folder`
- `Location of the source documents`: `Storage Account Source Container URL`
- `Location of the translated documents`: `Storage Account Target Container URL`

![](/images/20250717-docstrans/08.png)

# 動作確認
ワークフローを実行します。実行履歴から、問題なく動作していることを確認します。
![](/images/20250717-docstrans/09.png)

Storage Account を確認しに行きます。`ja` というフォルダが切られており、翻訳後のドキュメントが出力されていることを確認します。
![](/images/20250717-docstrans/10.png)

ファイルをダウンロードして、中身を確認します。翻訳されています。
- `.docx`
![](/images/20250717-docstrans/11.png)
画像の中身の文字までは翻訳ができていません。ここまでやらなければいけない場合は、別途かなりの作り込みが必要そうです。ROI 的には、そこまで求めるのも割に合わない気がします。
![](/images/20250717-docstrans/12.png)

- `.pdf`
![](/images/20250717-docstrans/13.png)

# 終わりに
本記事では、Azure AI Translator を使ってバッチ処理的にドキュメントをバルクで翻訳しました。ある程度のクオリティで翻訳できれば良いということであれば、ファイルをストレージに配置さえすれば、シンプルに実現できるため多言語対応も簡単に実現できます。また、Logic Apps では Agent workflow が使えるため、高度なシナリオも含めて何か面白いことも実現できそうです。