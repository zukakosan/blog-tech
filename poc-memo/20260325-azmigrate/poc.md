# 目的
Azure Migrate を使って Azure 間でのマイグレーションを試す
エージェントベース移行を試す

---

# 前提条件・構成概要

## アーキテクチャ

- **移行元**: Azure VM（リージョン A）
- **移行先**: Azure VM（リージョン B、または別サブスクリプション）
- **レプリケーションアプライアンス**: 移行元と同じ VNet に配置する Azure VM（Windows Server 2022 推奨）
- **Mobility Service**: 移行対象の VM にインストールするエージェント

> **注意**: Azure VM 間の移行では「物理サーバーとしての移行」を選択する（仮想化の種類は "Not virtualized/Other"）。エージェントベース移行は Azure Site Recovery の機能をベースとしている。

## 必要なリソース

| リソース | 用途 |
|---|---|
| Azure Migrate プロジェクト | 移行の管理・追跡 |
| レプリケーションアプライアンス VM | レプリケーションデータの中継 |
| Recovery Services コンテナー | バックグラウンドで自動作成される |
| キャッシュストレージアカウント | レプリケーションデータの一時保存 |
| 移行先 VNet / サブネット | 移行後の VM が参加するネットワーク |

## レプリケーションアプライアンスの要件

| 項目 | 要件 |
|---|---|
| OS | Windows Server 2016 / 2022 |
| CPU | 8 vCPU 以上 |
| メモリ | 16 GB 以上 |
| ディスク | OS ディスク + キャッシュディスク（600 GB 以上の空き） |
| ネットワーク | 移行元 VM から HTTPS 443 / TCP 9443 で到達可能 |

## ポート要件

| 通信経路 | ポート | 説明 |
|---|---|---|
| 移行元 VM → レプリケーションアプライアンス | HTTPS 443, TCP 9443 | 制御チャネル / レプリケーションデータ転送 |
| レプリケーションアプライアンス → Azure | HTTPS 443 | レプリケーションデータの送信 |

---

# 手順

## 1. Azure の準備

### 1-1. Azure Migrate プロジェクトの作成

1. Azure ポータルで **Azure Migrate** を検索して開く
2. **サーバー、データベース、Web アプリ** を選択
3. **プロジェクトの作成** をクリック
4. サブスクリプション、リソースグループ、プロジェクト名、リージョンを入力して作成

### 1-2. 権限の確認

- Azure アカウントに **共同作成者** または **所有者** ロールがあること
- VM の作成、マネージドディスクへの書き込み権限があること
- レプリケーションアプライアンスを Microsoft Entra ID に登録するため、**アプリケーション開発者** ロールが必要（グローバル管理者であれば不要）

### 1-3. 移行先ネットワークの準備

1. 移行先リージョンに VNet / サブネットを作成
2. 必要に応じて NSG を構成

---

## 2. レプリケーションアプライアンスの構築

### 2-1. アプライアンス用 VM の作成

1. 移行元と同じ VNet（または VNet ピアリングで接続されたネットワーク）に Windows Server 2022 の VM を作成
   - 推奨スペック: 8 vCPU / 16 GB RAM / 600 GB 以上の空きディスク
2. NSG でポート HTTPS 443 と TCP 9443 のインバウンドを許可

### 2-2. リソースの作成とインストーラーのダウンロード（Simplified Experience 推奨）

> **重要**: Classic Experience は 2026 年 3 月 31 日以降、新規レプリケーションがブロックされる。Simplified Experience を使用すること。

1. Azure Migrate プロジェクト → **サーバー、データベース、Web アプリ** → **Migration and modernization** → **検出** をクリック
2. **マシンは仮想化されていますか？** → **Not virtualized/Other** を選択
3. **ターゲットリージョン** → 移行先の Azure リージョンを選択・確認
4. **リソースの作成** をクリック（バックグラウンドで Recovery Services コンテナーが作成される）
5. **新しいレプリケーションアプライアンスをインストールしますか？** → **レプリケーションアプライアンスをインストール** を選択
6. アプライアンスのインストーラーと登録キーをダウンロード（キーの有効期限は 5 日間）

### 2-3. アプライアンスのインストール

1. ダウンロードしたインストーラーと登録キーをアプライアンス VM にコピー
2. **管理者として** インストーラーを実行

**OVA テンプレートを使用する場合（VMware 環境向け）:**
- OVA テンプレートをダウンロードしてデプロイ

**PowerShell を使用する場合:**
1. インストーラー ZIP をダウンロード・展開
2. `DRInstaller.ps1` を管理者として実行

### 2-4. アプライアンスの登録

1. インストール完了後、**Microsoft Azure Appliance Configuration Manager** が自動起動
2. 前提条件の検証（インターネット接続、時刻同期、システム構成など）が行われる
3. ダウンロードした登録キーを使ってアプライアンスを Azure Migrate プロジェクトに登録
4. 登録が完了したことを確認

---

## 3. 移行元 VM の準備

### 3-1. Mobility Service のインストール

移行対象の各 VM に Mobility Service をインストールする。

**方法 A: プッシュインストール（自動）**
- レプリケーション有効化時に自動でインストールされる
- 移行元 VM に対する管理者アカウント（ドメインまたはローカル）が必要
- Windows の場合: ドメインアカウントでない場合、レジストリで `LocalAccountTokenFilterPolicy` を `1` に設定
  ```
  HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
  値: LocalAccountTokenFilterPolicy = 1 (DWORD)
  ```
- Linux の場合: root アカウントを準備

**方法 B: 手動インストール**
- Mobility Service インストーラーを各 VM に手動でコピー・実行

### 3-2. VM の事前変更

移行前に以下を確認・必要に応じて変更:

- **Windows**: Hyper-V ゲストサービスの有効化、SAN ポリシーの設定など
- **Linux**: Azure Linux Agent のインストール、initramfs の再構成など

> 移行前にこれらの変更を行わないと、Azure で VM が起動しない可能性がある。

---

## 4. レプリケーションの開始

1. Azure Migrate プロジェクト → **Migration and modernization** → **レプリケート** をクリック
2. **ソース設定**:
   - **マシンは仮想化されていますか？** → **Not virtualized/Other**
   - **オンプレミスアプライアンス** → セットアップしたアプライアンスを選択
   - **プロセスサーバー** → レプリケーションアプライアンスを選択
   - **ゲスト資格情報** → Mobility Service インストール用のアカウントを選択
3. **仮想マシン** → 移行対象の VM を選択
4. **ターゲット設定**:
   - サブスクリプション、移行先リージョン、リソースグループを選択
   - 移行先 VNet / サブネットを選択
   - キャッシュストレージアカウントを選択
5. **可用性オプション**:
   - 可用性ゾーン / 可用性セット / 冗長性不要 から選択
6. **ディスク暗号化の種類**:
   - プラットフォームマネージドキー / カスタマーマネージドキー / ダブル暗号化 から選択
7. **コンピューティング**:
   - VM 名、VM サイズ、OS ディスクタイプを確認・調整
8. **ディスク**:
   - レプリケート対象のディスクを選択（不要なディスクは除外可能）
   - ディスクタイプ（Premium SSD / Standard SSD / Standard HDD）を選択
9. **タグ**: 必要に応じてタグを設定
10. **レプリケーションの開始** をクリック

> レプリケーションが開始されると、初期レプリケーション → デルタレプリケーションの順に進む。

---

## 5. テスト移行の実行

> **推奨**: 本番移行の前に必ずテスト移行を実行すること。

1. Azure Migrate → **Migration and modernization** → **レプリケートされたサーバー** を選択
2. 対象の VM を右クリック → **テスト移行** を選択
3. テスト用の Azure VNet を選択（本番用ではない VNet を推奨）
4. **テスト移行** ジョブが開始される → ポータル通知で進捗を監視
5. 移行完了後、Azure ポータルの **仮想マシン** でテスト VM（名前に `-Test` サフィックスが付く）を確認
6. アプリケーションの動作確認・接続性テストを実施
7. テスト完了後、**レプリケートされたマシン** で対象 VM を右クリック → **テスト移行のクリーンアップ** を実行

---

## 6. 本番移行の実行

1. Azure Migrate → **Migration and modernization** → **レプリケートされたサーバー** を選択
2. 対象の VM を右クリック → **移行** を選択
3. **仮想マシンをシャットダウンし、データ損失のない計画済みの移行を実行しますか？**
   - **はい** → 移行元 VM がシャットダウンされ、最小限のデータ損失で移行
   - **いいえ** → 移行元 VM はシャットダウンされない（データ損失の可能性あり）
4. 移行ジョブが開始される → Azure 通知で進捗を追跡
5. ジョブ完了後、Azure ポータルの **仮想マシン** で移行された VM を確認

---

## 7. 移行の完了・後処理

1. 移行完了後、対象 VM を右クリック → **レプリケーションの停止** を選択
   - 移行元のレプリケーションが停止される
   - レプリケーションの状態情報がクリーンアップされる
2. **Windows のライセンス認証** の確認・トラブルシューティング
3. **アプリケーションの調整**:
   - ホスト名の更新
   - データベース接続文字列の更新
   - Web サーバーの構成変更
4. **最終テスト**: アプリケーションと移行の受け入れテスト
5. **トラフィックの切り替え**: DNS やロードバランサーの設定を移行先 VM に切り替え
6. **移行元 VM の処理**:
   - 移行元 VM の停止・削除（必要に応じて）
   - ローカルバックアップからの移行元 VM の削除
7. **ドキュメントの更新**: 新しい IP アドレスとロケーション情報を反映

---

# 注意事項・Tips

- レプリケーションアプライアンスは、移行対象の VM や Discovery アプライアンスとは **別の VM** にインストールすること
- 一度に最大 **10 台** の VM をレプリケーション可能。10 台以上の場合はバッチに分ける
- ポータルでレプリケーション進捗を監視し、**429 エラー（Request Rate Too Large）** が出た場合はリトライを待つ
- Classic Experience は **2026/3/31 に新規レプリケーションがブロック**、**2026/9/30 にサポート終了** のため、Simplified Experience を使用すること

# 参考リンク

- [Migrate VMware vSphere VMs to Azure (agent-based)](https://learn.microsoft.com/azure/migrate/tutorial-migrate-vmware-agent?view=migrate)
- [Migrate machines as physical servers to Azure](https://learn.microsoft.com/azure/migrate/tutorial-migrate-physical-virtual-machines?view=migrate)
- [Replication appliance requirements](https://learn.microsoft.com/azure/migrate/migrate-replication-appliance?view=migrate)
- [Agent-based migration architecture](https://learn.microsoft.com/azure/migrate/agent-based-migration-architecture?view=migrate)
- [Support matrix for VMware vSphere migration](https://learn.microsoft.com/azure/migrate/migrate-support-matrix-vmware-migration?view=migrate)