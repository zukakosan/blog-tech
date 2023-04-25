---
title: "検証用テナントでMicrosoft Authenticatorを誤削除して焦った"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","aad","authenticator"]
published: true
publication_name: "microsoft"
---
# モチベ
- 「Microsoft Authenticatorのアカウント整理しよ」
- 「あ、やべ、検証用テナントのアカウント消してもうた」
- 「MS Authenticatorで再サインインすればいいんやろ」
- MS Authenticator:「デバイスに表示されているコードを入力して下さい」
- 「そのコードが見れないねん、、、。」
- 「myprofileからMFAのオプション変更すればいいんやろ」
- myprofile:「デバイスに表示されているコードを入力して下さい」
- 「そのコードが見れないねん、、、。」

汗。。。

状況としては、自分がグローバル管理者ロールを持つのAzure ADテナントにおいて、MS Authenticatorの設定を削除してしまったがゆえにMFAが必要になるタイミングで詰む。という話。

# 対処
- Azure PortalにはMFAなしでログインが出来ているため、Azure PortalのAzure AD管理画面からユーザ情報をいじる
- 結論としてはAzure ADの自分のユーザから「認証方法」->「多要素認証の再登録」を押せばよい
![](/images/20230425-azureadmfareset/01.png)
- ただ、これを自分のユーザに対して実行しようとするとエラーになる
![](/images/20230425-azureadmfareset/02.png)
- よって、もう一人のグローバル管理者ロールを持つユーザを作成
![](/images/20230425-azureadmfareset/03.png)
- 作成したユーザでAzure Portalへログイン
- そのユーザで自分のユーザの先ほどの「多要素認証の再登録」を押すことでMFAのリセットが可能
![](/images/20230425-azureadmfareset/04.png)

## 自分のユーザでへAzure AD認証が挟まるサイトへログイン
- 初回のログイン時に、「追加の情報が必要」という画面が表示され、MS Authenticator等の設定画面が出てくる
- 画面に従って、QRコードを読み込んだりして設定する

# おわり
- MS Authenticator、削除ダメ絶対
