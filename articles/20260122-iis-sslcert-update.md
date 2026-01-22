---
title: "Azure VM 上の IIS で証明書を自動更新する（Key Vault VM 拡張機能）"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure", "IIS", "KeyVault", "SSL", "WindowsServer"]
published: false
---

## TL;DR

## はじめに
- 課題：Azure VM上のIISでSSL証明書を運用する際、証明書の更新作業が手動になりがち
- 解決策：App Service 証明書 + Key Vault + VM拡張機能で自動更新を実現
- 本記事のゴール：証明書の取得から IIS へのバインド、自動更新までを一気通貫で構成する

2025年4月、CA/Browser Forum（CA/B フォーラム）は **Ballot SC-081v3** を承認し、SSL/TLS証明書の最大有効期間を段階的に短縮することを決定しました。最終的には **47日** まで短縮されます。

| 施行日 | 最大有効期間 | DCV再利用期間 |
|--------|-------------|---------------|
| 2026年3月15日 | 200日 | 200日 |
| 2027年3月15日 | 100日 | 100日 |
| 2028年3月15日 | 100日 | 10日 |
| 2029年3月15日 | 47日 | 10日 |

### なぜ47日なのか

47日という数字は、以下のロジックに基づいています：

> **47日 = 42日（6週間）+ 5日（早期更新猶予）**

同様に、200日 = 180日（6ヶ月）+ 20日、100日 = 90日（3ヶ月）+ 10日という計算になっています。

### 短縮の目的

CA/Browser Forum[^1]がこの決定を下した背景には、3つのセキュリティ上の要請があります：

1. **攻撃の露出期間の最小化**
   - 証明書や秘密鍵が漏洩した場合でも、有効期間が短ければ悪用される期間を限定できる
   - IBM Cost of a Data Breach Report 2024 によると、認証情報の侵害は検知・封じ込めに平均292日を要し、最もコストの高い攻撃経路の一つ

2. **ドメイン検証の信頼性強化**
   - DCV（Domain Control Validation）の再利用期間も10日に短縮され、正当なドメイン所有者のみが有効な証明書を維持できる

3. **暗号アジリティの促進**
   - 短い証明書ライフサイクルにより、組織は迅速な暗号移行のためのインフラ構築を迫られる
   - 2024年8月に NIST が発表したポスト量子暗号標準（FIPS 203, 204, 205）への移行準備にも寄与

### 背景と経緯

- **Google** が "Moving Forward, Together" ロードマップで90日への短縮を提案
- **Apple** が47日への段階的短縮を提案し、Ballot SC-081v3 として提出[^2]
- **Apple、Google、Mozilla、Microsoft** の主要ブラウザベンダーすべてが賛成票を投じ、業界として統一された方向性を示した

### 影響

現在の398日から47日への短縮は、**証明書更新頻度が約8倍**になることを意味します。手動で証明書を管理している環境では、この更新頻度に対応することは現実的ではありません。
本記事では、Azure VM でホストされている IIS について、**VM 拡張機能**を用いてどのように SSL 証明書の更新を自動化できるかを紹介します。

## 前提条件
本記事では、証明書の更新自動化のために、**Azure Key Vault VM 拡張機能**[^3] を使用します。前提条件として以下の構成が必要です。Windows Server 2019 以降（本記事では Windows Server 2025 を使用）が必要です。また、VM にはマネージド ID が必要です。

### 環境構成

## 1. App Service 証明書の Key Vault への格納
今回は、検証用に App Service 証明書を使用します。DigiCert や GlobalSign などの、外部の証明書を利用する場合は、このパートはスキップしてください。また、外部の証明書を Azure Key Vault で自動更新する手順についてはこちらの記事をご参照ください。
https://zenn.dev/microsoft/articles/20250711-digicertonazure

### App Service 証明書とは
App Service 証明書は、Azure が提供する SSL/TLS 証明書の購入・管理サービスです。GoDaddy が発行元となる公的に信頼された証明書を、Azure Portal から直接購入・管理できます。名前の通り、App Service と合わせて使うことが主な用途ですが、証明書(.pfx) をエクスポートして Web サーバへインストールすることも可能です。App Service にバインドする場合、Azure Key Vault 内に証明書を格納し、そこを読みに行く構成になります。
:::message
App Service 証明書は、Azure Key Vault 上のシークレット ストアに格納されます。
:::
Azure portal から必要な情報を入れ、良しなに作成します。

### App Service 証明書の作成
作成した App Service 証明書の [証明書の構成] から、手順に従って Azure Key Vault に証明書を格納します。
![](/images/20260122-iis-sslcert-update/appscert-01.png)

App Service 証明書用の Azure Key Vault では、**RBAC モデルが非サポートのため、コンテナー アクセス ポリシー モデルを使用します**。証明書の格納アクションの Caller は Microsoft Azure App Service となりますが、必要なポリシーは事前に追加されています。
![](/images/20260122-iis-sslcert-update/appscert-02.png)

ドメインの検証も行い、準備完了状態であることを確認します。
![](/images/20260122-iis-sslcert-update/appscert-03.png)

:::message
注意：App Service証明書はKey Vaultアクセスポリシーのみサポート（RBACは不可）
:::

## 2. IIS への証明書の手動バインド
### なぜ Key Vault から直接バインドできないのか
IIS から直接 Key Vault を見に行って、勝手にバインドしてくれればありがたいのですが、そのようなことはできません。これは以下の理由によります。当然といえば当然ですね。
  - IIS は Windows 証明書ストア（LocalMachine\My 等）のみ参照可能
  - Key Vault は外部の秘密鍵ストアであり、IIS から直接参照する仕組みがない

### 手動でのバインドの流れ（参考）
ここは、.pfx を Azure portal からエクスポートし、Windows Server 上でインポートするのが早いです。その後、IIS の Site Bindings で Type:HTTPS でバインドします。
![](/images/20260122-iis-sslcert-update/iis-01.png)
![](/images/20260122-iis-sslcert-update/iis-02.png)
![](/images/20260122-iis-sslcert-update/iis-03.png)

外部からアクセスすると有効な証明書が確認できます。
![](/images/20260122-iis-sslcert-update/iis-04.png)

:::message
### IIS の「バインディング」とは何か
IIS の Binding（バインディング） は、**この Web サイトは、どの IP / ポート / ホスト名 / プロトコルで通信を受け付けるか** を定義する設定です。HTTPS の場合は、これに どの SSL/TLS 証明書を使うか が追加されます。
:::
## 3. 本題: Azure Key Vault VM 拡張機能による証明書更新の自動化
- Key Vault VM拡張機能とは
- バージョンによる機能差（v3.0以降でIISバインド機能が追加）
- 拡張機能のインストール
  - settings.jsonの構成例
  - `observedCertificates`の指定
  - `linkOnRenewal`と`accounts`の設定（IIS AppPoolIdentityの指定方法）
- インストールコマンド例

## 4. IIS側の設定：証明書の自動再バインド
- IIS 8.5以降の「Centralized Certificate Store」または「自動再バインド」機能
- IIS Managerでの設定手順（Enable Automatic Rebind）
- この設定がないと、証明書ストアに新しい証明書が入ってもバインドが更新されない

## 5. 自動更新の検証
- App Service証明書のRekey（強制更新）で新バージョンを作成
- ポーリング間隔（デフォルト1時間）後の動作確認
- 確認コマンド
  - `netsh http show sslcert`でバインド中の証明書ハッシュを確認
  - Key Vault VM拡張機能のログ確認
  - Windowsイベントログでの証明書更新イベント確認
- ブラウザで証明書情報を確認

## 6. 運用上の考慮事項
- 古い証明書の削除（拡張機能では自動削除されない）
- Key VaultへのPrivate Endpoint構成（より安全な通信）
- ポーリング間隔のチューニング

## まとめ
- App Service証明書 + Key Vault + VM拡張機能 + IIS自動再バインドで完全自動化が可能
- 証明書更新の運用負荷を大幅に削減できる

## 参考リンク
- [Windows用のAzure Key Vault VM拡張機能](https://learn.microsoft.com/ja-jp/azure/virtual-machines/extensions/key-vault-windows)
- [App Service証明書の管理](https://learn.microsoft.com/ja-jp/azure/app-service/configure-ssl-certificate)
- [IIS 8.5での証明書の再バインド](https://learn.microsoft.com/ja-jp/iis/get-started/whats-new-in-iis-85/certificate-rebind-in-iis85)

[^1]:https://cabforum.org/2025/04/14/ballot-sc-081-v3-introduce-schedule-of-reducing-certificate-validity-and-data-reuse-periods/
[^2]:https://support.apple.com/en-us/102028
[^3]:https://learn.microsoft.com/ja-jp/azure/virtual-machines/extensions/key-vault-windows#features