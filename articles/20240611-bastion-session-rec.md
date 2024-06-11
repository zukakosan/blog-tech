---
title: "Azure Bastion でセッションを録画して閉域内のストレージに作業内容を保存する"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","vm"]
published: false
publication_name: "microsoft"
---
# はじめに
VM にアクセスして管理作業をする際にコンプライアンスの観点から作業内容を録画しなければならないという要件が存在します。そのため、一部の VDI ソリューションでは画面録画機能[^1]が付随していたり、専用のソフトウェア[^2]によって強制的に画面収録を行うケースがあります。
[^1]:https://docs.vmware.com/jp/VMware-Horizon/2312/virtual-desktops/GUID-DE409B35-A487-48A1-BEBF-02CA400FA119.html
[^2]:https://www.manageengine.jp/products/Password_Manager_Pro/session-recording.html
Azure Bastion においてもセッション録画機能が公開され、Azure VM に対する管理作業をシームレスに録画できるようになりました。

https://learn.microsoft.com/ja-jp/azure/bastion/session-recording

# Azure Bastion とは
Azure Bastion は VM に対してパブリック IP を付与せずにリモートアクセスを可能とする際の踏み台サーバーのサービスです。

## セッションの録画
Azure Bastion の Premium SKU を利用することで、セッションの録画機能を有効化できます。Azure Bastion のセッション記録機能が有効になっている場合は、bastion ホスト経由で仮想マシンに対して行われた接続 (RDP と SSH) のグラフィカル セッションを記録できます。 

セッションが閉じられたり切断されたりすると、記録されたセッションはストレージ アカウント内の BLOB コンテナーに (SAS URL 経由で) 格納されます。 セッションが切断されると、Azure portal の [セッション記録] ページで、記録されたセッションにアクセスして表示できます。


# Azure Bastion のセッション録画を構成する
ドキュメント[^3]にしたがって、環境を構築していきます。
[^3]:https://learn.microsoft.com/ja-jp/azure/bastion/session-recording

## サブネットの追加
Azure Bastion Premium SKU の利用には専用のサブネットが必要なため、`AzureBastionSubnt` として作成します。
![](/images/20240611-bastion-session-rec/bastionrec-01.png)

## セッション録画機能の有効化
Azure Bastion を Premium SKU で作成(またはアップグレード)し、Session recording を有効化します。
![](/images/20240611-bastion-session-rec/bastionrec-02.png)

## ストレージ アカウントの作成
録画データの保存用にストレージアカウントを作成します。
![](/images/20240611-bastion-session-rec/bastionrec-03.png)

また、適当な名前のコンテナーも作成しておきます。
![](/images/20240611-bastion-session-rec/bastionrec-04.png)

### CORS の設定 
Azure Bastion からドメインの異なるストレージアカウントにファイルを吐き出すことになるため、そのアクセス許可のために CORS[^4] の設定を行います。

検証上はドキュメントに記載のようにワイルドカードでよいかもしれませんが、できれば Azure Bastion の DNS 名に絞りたいところです。Bastion の DNS 名は一度 Bastion 経由で VM に接続してみると表示されますが、次のようになっています。

```
https://bst-a4b8e50a-xxxx-xxxx-xxxx-2d4b9358b616.bastion.azure.com/
```

UUID が含まれるため、もう少し汎用化して次の DNS 名として入力しておきます。これで、`bastion.azure.com` のすべてのサブドメインからの接続が許可されます。

```
https://*.bastion.azure.com/
```

![](/images/20240611-bastion-session-rec/bastionrec-17.png)


[^4]:https://learn.microsoft.com/ja-jp/rest/api/storageservices/cross-origin-resource-sharing--cors--support-for-the-azure-storage-services

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

[View recording] をクリックすると、対象の録画ファイルをブラウザ上で確認できます。
![](/images/20240611-bastion-session-rec/bastiongif.gif)

また、ストレージ アカウント側を見ると、同じファイルが存在することが確認できます。
![](/images/20240611-bastion-session-rec/bastionrec-07.png)

# 閉域化
録画によって作業内容の監視はできることが分かりました。ただ、この録画データは機微な情報を含むため、パブリックなネットワークに置きたくないはずです。

よって、続いては Private Endpoint でストレージアカウントを保護したい場合を考えます。

## ストレージ アカウントのプライベート エンドポイント
一般的な手順[^5] でストレージアカウントを閉域化していきます。
[^5]:https://learn.microsoft.com/ja-jp/azure/storage/common/storage-private-endpoints

![](/images/20240611-bastion-session-rec/bastionrec-09.png)
![](/images/20240611-bastion-session-rec/bastionrec-10.png)
![](/images/20240611-bastion-session-rec/bastionrec-11.png)

## ストレージ アカウントのパブリック アクセス無効化
パブリック アクセスを無効化します。
![](/images/20240611-bastion-session-rec/bastionrec-12.png)

## 再接続
この状態で録画できることを確認するため、再度 Azure Bastion 経由で VM に接続し、切断します。
![](/images/20240611-bastion-session-rec/bastionrec-13.png)

## セッション録画の確認
Azure Bastion の [Session recordings] に `Unable to fetch blobs` と表示され見れなくなってしまいました。
![](/images/20240611-bastion-session-rec/bastionrec-14.png)

これは、裏側に存在するストレージ アカウントに対して、現在 Azure portal にアクセスしているユーザーの IP から到達できないためです。
よって、クライアント IP をストレージ アカウントのネットワーク制限で許可すると、録画が確認できるようになりました。
![](/images/20240611-bastion-session-rec/bastionrec-15.png)

また、Azure Bastion でログインした VM から Azure portal にアクセスする場合は、そもそも閉域内のアクセスとなるため、ストレージ アカウントでの穴あけ不要で録画データを確認できました。
![](/images/20240611-bastion-session-rec/bastionrec-16.png)

# おわりに
要件として、録画が必要なケースに Azure Bastion も対応できるようになると、Azure のサービスだけでシステムが完結するため非常にシンプルになります。Premium SKU の価格次第なところもありますが、別途ソリューションを導入するよりははるかにハードルが低い形で運用できるようになるのは間違いないです。