---
title: "DigiCert で SSL 証明書を発行して Azure Key Vault で管理する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","tls","microsoft"]
published: true
publication_name: "microsoft"
---

# Azure における証明書
Azure でホストするアプリケーションに HTTPS を適用する場合、Azure 上で SSL 証明書を準備する必要があります。いわゆるオレオレ証明書を使えば検証等は一応できますが、正式に外部公開サービスをホストする場合、専用の信頼された CA から発行された SSL 証明書を利用する必要があります。

Azure 上でも、App Service 証明書という証明書を購入して利用できるのですが、Azure 上での取り回しに若干癖があります。よって、本記事では、一般的によく使われている DigiCert 上で証明書を発行し、Azure に取り込むところまでを確認します。


# CSR の発行
DigiCert に対して Certificate Signing Request を発行します。これは、公開鍵証明書を発行してもらうために CA に提出するデータです。公開鍵やサブジェクト情報、サブジェクト識別名、秘密鍵による署名が含まれます。

## DigiCert Certificate Utility 
DigiCert を使う場合は、DigiCert Certificate Utility[^1] というツールが公開されており、これを利用することで簡単に CSR を作成できます。

[^1]:https://www.digicert.com/jp/support/tools/certificate-utility-for-windows

DigiCert Certificate Utility をインストールして起動します。`Create CSR` をクリックして必要な情報を入力の上、`Generate` をクリックして作成します。のちに使用するため `Copy CSR` から、中身をコピーしておきましょう。

![](/images/20250711-digicertonazure/01.png)
![](/images/20250711-digicertonazure/02.png)
![](/images/20250711-digicertonazure/03.png)

## 証明書のオーダー
DigiCert のサイト上で証明書の申請を行います。以下の画面で適切な証明書タイプを選択します。
![](/images/20250711-digicertonazure/04.png)

CSR の部分に先ほどコピーした文字列を挿入します。
![](/images/20250711-digicertonazure/05.png)

指示に従ってドメインの検証等を行います。方法はいくつかありますが TXT レコードによる検証が一番楽だと思います。
![](/images/20250711-digicertonazure/06.png)


# Azure アップロード用の証明書の取得

しばらく待って証明書申請が承認されたら、ダウンロードしてローカル PC に Import していきます。
![](/images/20250711-digicertonazure/07.png)

`Individual .crts (zipped)` をダウンロードします。
![](/images/20250711-digicertonazure/08.png)

## ローカル PC への証明書の Import
DigiCert Certificate Utility に戻ってダウンロードした証明書を Import します。この際、必ず CSR を発行した端末上で行ってください。CSR 作成のタイミングで証明書に対応する Private Key が作成されるため、それがないとのちに `.pfx` 形式で証明書が Export できなくなります。
![](/images/20250711-digicertonazure/09.png)

適当な名称で Import します。
![](/images/20250711-digicertonazure/10.png)

対象の証明書を選択して、Export を選択します。
![](/images/20250711-digicertonazure/11.png)

`.pfx` のオプションがグレーアウトしてなければ問題なく進んでいます。次の画面で DigiCert のパスワード入力が求められますが、それを突破すれば Export に成功します。
![](/images/20250711-digicertonazure/12.png)
![](/images/20250711-digicertonazure/13.png)

# Azure への証明書アップロード
Azure Key Vault リソースを作成しておき、証明書としてアップロードします。機密情報のため、閉域内からのアップロードが推奨です。このパスワードは DigiCert のものと同様です。
![](/images/20250711-digicertonazure/14.png)

アップロードできました。
![](/images/20250711-digicertonazure/15.png)

# 証明書の自動更新の設定
信頼された CA として DigiCert を追加しておくことで、自動更新が可能になります。自動更新[^2] の設定や信頼された CA の登録[^3] は各ドキュメントを参照して使用ください。最終的には以下のようにポリシーを設定できます。
![](/images/20250711-digicertonazure/18.png)

[^2]:https://learn.microsoft.com/ja-jp/azure/key-vault/certificates/tutorial-rotate-certificates
[^3]:https://learn.microsoft.com/ja-jp/azure/key-vault/certificates/how-to-integrate-certificate-authority

# ハマりポイント
- データプレーンの権限
Azure Key Vault のデータプレーン用権限は別で用意されているため、サブスクリプションの所有者であっても別途データプレーン操作用の権限をアタッチしてください。
- ネットワーク設定
機密情報になるため、ネットワーク制限がかかっていることも多いと思います。その際は閉域内からアクセスするなど気を付けましょう。
- エラーメッセージの解読
エラーメッセージが失敗を伝える簡素なものしか表示されないことがあり、わかりにくいため注意。メッセージの詳細を確認すると実はヒントがあったりします。
![](/images/20250711-digicertonazure/20.png)
