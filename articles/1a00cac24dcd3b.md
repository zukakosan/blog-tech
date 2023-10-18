---
title: "Azure Lighthouseによるマルチテナント管理のためのオンボーディング"
emoji: "🪬"
type: "tech"
topics:
  - "azure"
  - "tech"
  - "azuread"
  - "azurelighthouse"
published: true
publication_name: "microsoft"
published_at: "2022-08-12 14:06"
---

# モチベ
社内都合で検証用サブスクリプションのテナント毎の移動が発生したため、マルチテナント管理が楽にできるらしいAzure Lighthouseを使ってみたくなった

# Azure Lighthouse
- Azure AD B2Bを使ってゲスト招待することなく、同一のアカウントで別テナントのリソースを管理するためのサービス
- A社がB社のAzure環境を特定のスコープで管理する場合、A社がサービスプロバイダー、B社が顧客、という位置付けとなる
- A社がAzure環境管理のための情報を組み込んだARMテンプレートをB社と共有し、B社はそのARMテンプレートからサービスプロバイダーとしてA社をB社テナントに登録する
- これにより委任関係が成立し、A社はAzure Lighthouse経由でB社テナントを管理できる

# 手順
## サービスプロバイダ側でのARMテンプレート作成
- サービスプロバイダー側テナントにてAzure Lighthouseを検索
- 「作業の開始」⇒「顧客の管理」⇒「概要」から「ARMテンプレートの作成」を選択
![](https://storage.googleapis.com/zenn-user-upload/d7462d94ced5-20220812.png)
- 適切な認可の追加、閲覧者を含む権限がないとポータル上で確認出来ません
![](https://storage.googleapis.com/zenn-user-upload/87bdebf9c84a-20220812.png)
![](https://storage.googleapis.com/zenn-user-upload/d909482a332e-20220812.png)
- テンプレートのダウンロード
![](https://storage.googleapis.com/zenn-user-upload/f0c337920744-20220812.png)

## 顧客側でのARMテンプレートデプロイ
- 顧客側テナントにてAzure Lighthouseを開く
- 「サービスプロバイダ-プランの表示」⇒「サービスプロバイダ－のオファー」⇒「プランの追加」⇒「テンプレート経由で追加」を選択
![](https://storage.googleapis.com/zenn-user-upload/001b4e9712b9-20220812.png)
![](https://storage.googleapis.com/zenn-user-upload/418e41abf89a-20220812.png)
- 先ほどダウンロードしたARMテンプレートをアップロード
- 委任するサブスクリプションを選択(ARMテンプレート作成時のスコープがサブスクリプションになっていれば)
- リージョンはとりあえず東日本にしてデプロイ
![](https://storage.googleapis.com/zenn-user-upload/8fd7f25b9dad-20220812.png)
- デプロイ完了を待つ
## オンボーディング状態の確認
- サービスプロバイダ側のテナントのAzure Lighthouse画面を開く
- 「顧客の管理」⇒「顧客」と選択
- ここで何も表示されていなければ、サブスクリプションのフィルタがかかっているため「グローバルサブスクリプションフィルター」を選択
![](https://storage.googleapis.com/zenn-user-upload/670d69920ea4-20220812.png)
- 対象のディレクトリとサブスクリプションを選択
![](https://storage.googleapis.com/zenn-user-upload/4f1962965fde-20220812.png)
- 再度「顧客」画面を開いて確認
![](https://storage.googleapis.com/zenn-user-upload/d977074cb031-20220812.png)
::: message alert
- デフォルトだと恐らく表示するサブスクリプションのフィルターがかかっており、「顧客」ブレードに対象リソースが表示されない
- グローバルサブスクリプションフィルターで、対象のテナント(ディレクトリ)及び委任されたサブスクリプションにチェックを入れて明示的に表示する必要がある
- 筆者はこれに気付かず時間無駄にしました
:::

# おわり
シームレスに別テナントの管理ができるため、社内で複数のテナントを扱っている場合にも有用です。
![](https://storage.googleapis.com/zenn-user-upload/44df227c9221-20220812.png)