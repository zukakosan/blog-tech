---
title: "Azure Bastion でセッションを録画して作業内容を保管する"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","vm"]
published: false
publication_name: "microsoft"
---
# はじめに
VM にアクセスして管理作業をする際にコンプライアンスの観点から作業内容を録画しなければならないという要件が存在します。そのため、一部の VDI ソリューションでは画面録画機能が付随していたり、専用のソフトウェアをインストールしてスクリプトを走らせ強制的に画面収録を行うケースがあります。
Azure Bastion でセッション録画機能が公開され、Azure VM に対する管理作業をシームレスに録画できるようになりました。

https://learn.microsoft.com/ja-jp/azure/bastion/session-recording

# Azure Bastion とは
Azure Bastion は VM に対してパブリック IP を付与せずにリモートアクセスを可能とする際の踏み台サーバーのサービスです。

## セッションの録画
Azure Bastion の Premium SKU を利用することで、セッションの録画機能を有効化できます。Azure Bastion のセッション記録機能が有効になっている場合は、bastion ホスト経由で仮想マシンに対して行われた接続 (RDP と SSH) のグラフィカル セッションを記録できます。 

セッションが閉じられたり切断されたりすると、記録されたセッションはストレージ アカウント内の BLOB コンテナーに (SAS URL 経由で) 格納されます。 セッションが切断されると、Azure portal の [セッション記録] ページで、記録されたセッションにアクセスして表示できます。


# Azure Bastion のセッション録画を構成する
ドキュメント[^1]にしたがって、環境を構築していきます。
[^1]:https://learn.microsoft.com/ja-jp/azure/bastion/session-recording

## サブネットの追加
Azure Bastion Premium SKU の利用には専用のサブネットが必要なため、`AzureBastionSubnt` として作成します。
![](/images/20240611-bastion-session-rec/bastionrec-01.png)

## ストレージ アカウントの作成
録画データの保存用にストレージアカウントを作成します。
![](/images/20240611-bastion-session-rec/bastionrec-02.png)

また、適当な名前のコンテナーも作成しておきます。
![](/images/20240611-bastion-session-rec/bastionrec-03.png)

### CORS の設定 
Azure Bastion からドメインの異なるストレージアカウントにファイルを吐き出すことになるため、そのアクセス許可のために CORS[^2] の設定を行います。

検証上はドキュメントに記載のようにワイルドカードでよいかもしれませんが、できれば Azure Bastion の DNS 名に絞りたいところです。Bastion の DNS 名は一度 Bastion 経由で VM に接続してみると表示されますが、次のようになっています。

```
https://bst-a4b8e50a-xxxx-xxxx-xxxx-2d4b9358b616.bastion.azure.com/
```

UUID が含まれるため、もう少し汎用化して次の DNS 名として入力しておきます。これで、`bastion.azure.com` のすべてのサブドメインからの接続が許可されます。

```
https://*.bastion.azure.com/
```

![](/images/20240611-bastion-session-rec/bastionrec-04.png)


[^2]:https://learn.microsoft.com/ja-jp/rest/api/storageservices/cross-origin-resource-sharing--cors--support-for-the-azure-storage-services

### SAS URL の取得と設定
Azure Bastion は現状 SAS URL を使って録画データをストレージ アカウントに書き込む仕様となっているため、コンテナー レベルで SAS URL を発行します。

また、権限としては `READ`、`WRITE`、`CREATE`、`LIST` を許可します。

期限はかなり遠い日付に設定します。
![](/images/20240611-bastion-session-rec/bastionrec-05.png)

ここで取得した SAS URL を Azure Bastion の [Session recordings] の画面で登録します。
![](/images/20240611-bastion-session-rec/bastionrec-06.png)


# セッション録画の動作確認
ここまでの設定で、録画できるようになったはずなので、Azure Bastion 経由で VM に接続します。VM にログイン出来たら、ブラウザを開くなどの操作をしたのち切断します。

Azure Bastion の [Session recordings] を開くと、作成されたファイルが確認できます。
![](/images/20240611-bastion-session-rec/bastionrec-08.png)

また、ストレージ アカウント側を見ると、同じファイルが存在することが確認できます。
![](/images/20240611-bastion-session-rec/bastionrec-07.png)
