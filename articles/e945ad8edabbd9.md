---
title: "Microsoft Defender CSPMがGAしたのでセキュリティグラフ(エクスプローラー)と攻撃パス分析を触ってみる"
emoji: "🛡️"
type: "tech"
topics:
  - "azure"
  - "cloud"
  - "cspm"
  - "defender"
published: true
published_at: "2023-04-07 20:28"
publication_name: "microsoft"
---

# モチベ
- 2023年3月末にMicrosoft Defender CSPMがGA
- 概要を調べていて特にセキュリティグラフ、攻撃パス分析が有用そうだったので、試したい

https://learn.microsoft.com/ja-jp/azure/defender-for-cloud/concept-attack-path

# Microsoft Defender CSPM
![](https://storage.googleapis.com/zenn-user-upload/a47e3bda3aa0-20230407.png)
- Foundational CSPMとDefender CSPMがある
- Foundational CSPMは無料で以下の機能が利用可能
	- セキュリティ設定のアセスメント
	- 推奨事項
	- セキュアスコア
- Defender CSPMは有料で下記の機能が含まれる
	- ID とネットワーク露出の検出
	- **Network exposure detection**
	- **攻撃パスの分析**
	- リスク追求用のクラウド セキュリティ エクスプローラー
	- エージェントレスの脆弱性スキャン
	- タイムリーに修復と説明責任を促進するためのガバナンス ルール
	- 規制コンプライアンスと業界のベスト プラクティス
	- Data-aware security posture
	- Agentless discovery for Kubernetes - coming soon (mid-April)
	- Agentless vulnerability assessments for container images, including registry scanning - coming soon (mid-April)

## セキュリティグラフとは
> クラウド セキュリティ グラフは、Defender for Cloud 内に存在するグラフベースのコンテキスト エンジンです。 クラウド セキュリティ グラフは、マルチクラウド環境やその他のデータ ソースからデータを収集します。 たとえば、クラウド資産のインベントリ、リソース間の接続と横移動の可能性、インターネットへの露出、アクセス許可、ネットワーク接続、脆弱性などです。 収集されたデータは、マルチクラウド環境を表すグラフを作成するために使用されます。

つまり、マルチクラウド環境を表すグラフを作成し、クラウドセキュリティエクスプローラーからクエリをかけて分析できる状態となっている

## 攻撃パス分析とは
> クラウド セキュリティ グラフをスキャンするグラフベースのアルゴリズムです。 スキャンにより、攻撃者が環境を侵害して影響の大きい資産に到達するために使用する可能性のある悪用可能なパスが公開されます。 攻撃パス分析は、攻撃パスを公開し、攻撃パスを中断し、侵害の成功を防ぐ問題を修復する最善の方法に関するレコメンデーションを提案します。

つまり、作成されたセキュリティグラフに対してDefenderがスキャンをかけて外部から攻撃され得るパスを提示してくれる

# 検証
## Defender CSPMの有効化
- [環境設定]からサブスクリプションを選択
![](https://storage.googleapis.com/zenn-user-upload/c2f02ef9d94b-20230407.png)
- Defender CSPMをオンに設定する
![](https://storage.googleapis.com/zenn-user-upload/2f1de2f16dc8-20230407.png)
## セキュリティグラフ
- [セキュリティグラフ]を開き、クラウドセキュリティエクスプローラーを表示
	- クエリのテンプレートを選択(ここでは、インターネットに公開されているVM)
![](https://storage.googleapis.com/zenn-user-upload/d82169b65751-20230407.png)
- 検索を実行する
![](https://storage.googleapis.com/zenn-user-upload/36b371b7efab-20230407.png)
- 結果として、インターネットに面しているVM一覧が出力される
![](https://storage.googleapis.com/zenn-user-upload/f17d586e5c63-20230407.png)
- `Exposed to the internet`の条件としてポートの設定も可能
![](https://storage.googleapis.com/zenn-user-upload/411a7b5c5627-20230407.png)

### 複雑な条件
- `Exposed to the internet`に加えて、その他の条件も加えることでユーザの見たい条件でリスクを評価できる
- ＋ボタンをクリックすると条件を追加でき、その条件の中身は組み込みの項目を選択可能
	- ここでは`Vulnerable to remote code execution`を選択
![](https://storage.googleapis.com/zenn-user-upload/aa06f5b29537-20230407.png)
- それぞれに合致するVMが出力される
![](https://storage.googleapis.com/zenn-user-upload/22c7fa3bf639-20230407.png)

### 対象リソースの変更
- 画像赤枠部分から対象リソースの選択が可能
![](https://storage.googleapis.com/zenn-user-upload/b0119d15482b-20230407.png)

## 攻撃パス分析
- 場所が非常にわかりにくいが[推奨事項]の中にある
![](https://storage.googleapis.com/zenn-user-upload/f6b62c88971f-20230407.png)
- セキュリティグラフを分析し、攻撃パスが存在するものについてのリストが確認できる
![](https://storage.googleapis.com/zenn-user-upload/c3a2e2af9991-20230407.png)
- 攻撃パスの項目をクリックすると、より詳細なビューを確認出来る
- ここでは、どこをエントリポイントにしてどこまで到達可能なのかというマップが表示され、文字通りパスが見えるようになっている
![](https://storage.googleapis.com/zenn-user-upload/d925c21fb295-20230407.png)
- この例だと、「このVMにはパッチ当てないとヤバいよね」ということが見えていることになる

# おわり
- Defender CSPMによって今までよりDefender for Cloudの可能性が大分広がった気がする
- 依然としてUIはところどころ使いづらいので、そこが何とかなれば、、、という感じ
