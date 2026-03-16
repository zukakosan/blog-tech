---
title: "Azure AD joinのセッションホストを利用する場合のAVDにおける注意とSSOの構成"
emoji: "🖥️"
type: "tech"
topics:
  - "azure"
  - "cloud"
  - "microsoft"
  - "avd"
published: true
published_at: "2022-10-20 16:41"
---
:::message
本記事は筆者個人の検証結果に基づくものであり、所属する組織の公式見解を示すものではありません。正確な情報や最新の仕様については、公式ドキュメントをご確認ください。
:::

# とりあえずメモ
- 適切にAVD関連リソースをデプロイしたはずが、セッションホストへアクセスしようとしたとき(2回目のPassword要求時に正しく入力しても)以下のようなエラーが出現

![](https://storage.googleapis.com/zenn-user-upload/a73d52d3e4da-20221020.png)

https://learn.microsoft.com/ja-jp/azure/virtual-desktop/troubleshoot-azure-ad-connections#the-user-name-or-password-is-incorrect

# イメージの差異
- 選択できる中で最新のWindows 11 22H2でセッションホストを作成したところ、必ずDomainJoinedCheckに失敗してしまった
- 最新版は避けるべきかも(リリース時期にもよるか)

![](https://storage.googleapis.com/zenn-user-upload/902e1c0a55cb-20221020.png)

# どうやらAAD joinのVMでは少し設定が必要らしい
- サブスクリプション所有者ロールだが一応「VMへの管理者ログイン」のRBACを追加

::: message
- 「VMへの管理者ログイン権限」は、サブスクリプションの「所有者」ロールに含まれないため必ず追加が必要。
- サブスクリプションのページで「マイアクセス」を表示すると「所有者」の中に下記権限がないことが確認できる
```
Microsoft.Compute/virtualMachines/loginAsAdmin/action
```
- ![](https://storage.googleapis.com/zenn-user-upload/53826fa43339-20221203.png)

https://learn.microsoft.com/ja-jp/azure/role-based-access-control/built-in-roles#virtual-machine-administrator-login
:::

- とりあえずDocsに記載の通りtargetisaadjoined:i:1をプロパティに追加した
- 反映まで少しラグがあったっぽいが接続はできた

![](https://storage.googleapis.com/zenn-user-upload/dc4d9d44f2d7-20221020.png)

# ただSSOを試したかったので構成確認
- ここで構成するだけっぽい

![](https://storage.googleapis.com/zenn-user-upload/bd0e790d213f-20221020.png)

- AAD認証の情報でRDPしますの意だと思われる
- Hybrid AAD joinのセッションホストとホストプールは分けたほうがよさそう

## Single Sign Onの確認
- 上記設定を行うと、仮想デスクトップ選択時にAAD認証の情報使いますか？の確認画面が出る

![](https://storage.googleapis.com/zenn-user-upload/e1b48428e2dd-20221020.png)

![](https://storage.googleapis.com/zenn-user-upload/71146fb172ab-20221020.png)

クリックするだけで仮想デスクトップに入れるようになった
![](https://storage.googleapis.com/zenn-user-upload/4d41c7c1d6d6-20221020.gif)

# 反省
- クラウドは設定の反映がいきわたるまで時間がかかる