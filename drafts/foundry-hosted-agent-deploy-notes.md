# Foundry Hosted Agent デプロイ入門ノート (code vs container)

`azd ai agent init` によるスキャフォールドから `azd up` によるデプロイまでの流れと、`--deploy-mode <code|container>` で選べる 2 つのデプロイ方式の比較をまとめたメモです。どちらの方式も **最終的にエージェントが動く場所は Foundry のマネージドランタイム** で共通で、違いは「何を梱包して、その中継に何を使うか」です。

関連する Lab 手順は [Lab 3](../../docs/03-foundry-deploy.md)、API リファレンスは [`kb-1.8.0/api-reference/1.8.0/hosted-agent-deploy.md`](../../kb-1.8.0/api-reference/1.8.0/hosted-agent-deploy.md) を参照してください。

---

## 開発フロー全体

```text
1. azd ai agent init   ← scaffold (ローカルに雛形 + azure.yaml を生成。Azure は触らない)
2. 手元で main.py を実装 (FoundryChatClient で Foundry 上のモデルを使う)
3. azd up              ← provision (Azure リソース作成) + deploy を一括実行
      ├ code モード      → src/<agent>/ を zip して Foundry へ push
      └ container モード → Dockerfile から image を build → ACR 経由で Foundry へ
```

- `azd ai agent init` は **ローカルにファイルと定義を用意するだけ**。実リソース作成は `azd up`（または `azd provision` + `azd deploy`）で初めて走る。
- 2 回目以降のコード変更反映は provision 不要なので `azd deploy` だけで速い ([Lab 3-6](../../docs/03-foundry-deploy.md))。

---

## `azd ai agent init` が生成するもの

init の役割は大きく「① 雛形ファイル一式の配置」と「② `azure.yaml` への service 定義の追記」の 2 つです。

| 種別 | 生成/更新されるもの | 役割 |
|---|---|---|
| ① 雛形ファイル配置 | `src/<agent>/main.py` | エントリポイントのテンプレート (自分のロジックに差し替える) |
| | `src/<agent>/requirements.txt` | 依存定義 |
| | `src/<agent>/Dockerfile` / `.dockerignore` / `.azdignore` / `.env.example` | デプロイ補助ファイル |
| ② `azure.yaml` 生成/更新 | `services.<agent>` ブロック | どの deploy モード・runtime・model・entryPoint で動かすかを定義 |
| | `infra.provider` | provision に使う IaC プロバイダ (`microsoft.foundry` 等) |

---

## モードは `azure.yaml` が決める (Dockerfile の有無ではない)

「Dockerfile を置けば container になる」ではなく、**どちらのモードで動くかは init 時に生成される `azure.yaml` の記述で決まります**。

```yaml
# code モードの azure.yaml (抜粋)
services:
    <agent>:
        host: azure.ai.agent
        language: python              # ← code モードを示す
        codeConfiguration:            # ← このブロックがあると code (zip) デプロイ
            dependencyResolution: remote_build
            entryPoint: main.py
            runtime: python_3_13
```

- code モードでも `src/<agent>/` に `Dockerfile` は同梱されるが、`language: python` + `codeConfiguration` なので **その Dockerfile は使われず zip デプロイ**になる。
- container にしたい場合は `azure.yaml` が container 用 (`language: docker` + `docker.remoteBuild` 等、[hosted-agent-deploy.md](../../kb-1.8.0/api-reference/1.8.0/hosted-agent-deploy.md)) になっている必要がある。

> [!TIP]
> code モードで init 済みのプロジェクトを container に切り替えたいときは、`azure.yaml` を手編集するより **`--deploy-mode container` で init し直す (別ディレクトリ推奨)** のが安全。`codeConfiguration` と `docker`/`container` 設定は排他的で、手編集だと不整合になりやすい。

---

## zip で送られるファイル (code モード)

zip 対象は **`azure.yaml` の `services.<agent>.project` が指すディレクトリの中身だけ**です。

| 対象 | zip に入るか | 理由 |
|---|---|---|
| `src/<agent>/` 配下のファイル | 入る | `project:` が指すデプロイ単位 |
| `.env.example` | 入らない | `.azdignore` で除外 |
| `azure.yaml` | 入らない | azd プロジェクト定義 (ローカルでの provision 用メタデータ) |
| `.azure/` | 入らない | azd の環境状態 (ローカル) |
| ルート直下の `README.md` 等 | 入らない | `project:` ディレクトリの外 |

- 境界は **`project:` ディレクトリ**。その外 (`azure.yaml` 自身やルートの README 等) はアプリ資材としては送られない。
- 除外は **`.azdignore`** (container 方式なら `.dockerignore`) で制御する。`.venv` や秘密情報ファイルはここに書いて除外する。

---

## 全体像

```text
code モード                                  container モード
──────────                                  ───────────────
main.py + requirements.txt                  Dockerfile から build した image
        │ zip 化                                    │ build
        ▼                                           ▼
   Foundry へ push                             ACR に push
        │                                           │ Foundry が pull
        ▼                                           ▼
 Foundry マネージドランタイムで起動         Foundry マネージドランタイムで起動
 (runtime を用意し pip install)             (image をそのまま実行)
```

- **code**: ソース一式 (`main.py` / `requirements.txt` / `agent.yaml` など deploy 単位のフォルダ) を zip にして Foundry へアップロードし、Foundry 側が runtime を用意して `pip install` → `main.py` を起動する。
- **container**: `Dockerfile` から image を build し、ACR (Azure Container Registry) 経由で Foundry が pull して実行する。ACR は「image の置き場（レジストリ）」であって実行環境ではない。

---

## 比較表

| 観点 | code モード | container モード |
|---|---|---|
| 梱包物 | ソース一式の zip | ビルド済み Docker image |
| `Dockerfile` | 生成されない | 生成される (自分で管理) |
| 実行環境の組み立て | Foundry が runtime を用意し `pip install` | 自分で固めた image を Foundry が pull |
| 中継ストレージ | Foundry 内部 (ACR 不要) | ACR (作成される) |
| 実行場所 | Foundry マネージドランタイム | 同左 (共通) |
| デプロイ時間 | 速い (概ね 1〜2 分) | 5〜10 分長い |
| 追加権限 / 事前ビルド | 不要 | ACR 作成・build が必要 |

---

## FROM 相当はどうなるか

Dockerfile を書かない code モードでも「実行環境の土台」は必要で、その **FROM 相当を担うのが `--runtime` で指定する Foundry マネージドのベースランタイム** です。

| Dockerfile の世界 | code モードでの相当物 |
|---|---|
| `FROM python:3.13-slim` | `--runtime python_3_13` (Foundry 提供のベースイメージ) |
| `COPY . /app` | zip されたソース一式の展開 |
| `RUN pip install -r requirements.txt` | Foundry 側が自動で `pip install` |
| `CMD ["python", "main.py"]` | `--entry-point main.py` の起動 |

つまり **Dockerfile を「自分で書く」代わりに Foundry が等価な処理を内部で組み立てて実行** します。base image の選択肢を `--runtime` の enum (`python_3_13` など) に絞ることで、Dockerfile を書かずに済ませています。

> [!NOTE]
> `--runtime` で選べる値は `azure.ai.agents` 拡張のバージョンに依存します。一覧は `azd ai agent init --help` で確認できます。ワークショップで Python 3.13 が必須なのも、この Hosted Agent ランタイム (FROM 相当) が Python 3.13 だからです。

---

## container モードのメリット

主に「環境の再現性・制御」と「エンタープライズ要件への適合」です。

| メリット | 説明 |
|---|---|
| Bring-your-own base image | `Dockerfile` の `FROM` を自由に選べる。社内認定の hardened イメージや独自 CA 証明書入りイメージを土台にできる |
| OS レベルの依存を同梱 | `apt-get install` 系のネイティブライブラリ・システムパッケージ・外部バイナリを入れられる (code モードは pip のみ) |
| ビルド環境の完全な固定 | ベースイメージ・システムパッケージまで image digest で pin でき、監査・サプライチェーン要件に合わせやすい |
| 既存 CI/CD・レジストリ資産の再利用 | イメージスキャン・署名・ACR のリテンションポリシー等の既存運用にそのまま乗せられる |
| ポータビリティ | 同じ image を Container Apps / AKS 等でも動かせる |
| 細かいランタイム制御 | `startupCommand`・CPU/メモリ (既定 0.5 core / 1Gi) 等をコンテナ単位で調整できる |

---

## code モードで十分なケース

- 依存が **pip パッケージだけ**で完結する (今回の MRC MCP エージェントはこれに該当)
- とにかく**速くデプロイ**したい
- ACR の運用・コスト・イメージ管理を**持ちたくない**
- Docker の知識なしでチームを回したい

---

## ネットワーク分離 (VNet / Private Endpoint)

**code / container どちらの deploy-mode でも VNet 内デプロイ (エージェント計算のサブネット注入) は可能** です。ただしこれは deploy-mode の設定ではなく、**Foundry Agent Service のネットワーク構成側**で決まります。`azd ai agent init` で **既存 Foundry プロジェクトを選ぶと、そのプロジェクトのネットワーク姿勢 (Private Endpoint / VNet injection) をエージェントも継承**します。

### egress モデル (共通・deploy-mode 非依存)

| egress モデル | 内容 | inbound |
|---|---|---|
| Public egress | 分離なし (プロトタイプ向け) | public or private endpoint |
| BYO virtual network | 自分の VNet の delegated subnet にエージェントを注入。IP 範囲・peering・routing を自分で管理 | private endpoint |
| Managed virtual network | Microsoft 管理の VNet に注入 (IP 管理不要) | private endpoint |

- VNet injection の要件: `Microsoft.App/environments` に委任した **/27 以上のサブネット**。
- どちらの deploy-mode でも、エージェント計算 (あなたの `main.py` が動くコンテナ) を VNet に入れるかどうかは同じ仕組みで決まる。

### VNet 保護できるリソース (azd path)

| リソース | VNet 保護 | 備考 |
|---|---|---|
| AI Services アカウント / Foundry プロジェクト | ✅ | data-plane・ARM とも private endpoint 経由 |
| Azure Container Registry | ✅ | `publicNetworkAccess: Disabled`。build/push/pull が PE 経由 ← **container モードのみ関係** |
| Application Insights | ✅ | Azure Monitor Private Link Scope 経由 |
| Azure Storage | ✅ | Blob/Files/Queue を PE 化 |
| エージェントのエンドポイント URL 自体 | ❌ (現状 preview) | URL は public にアドレス可能。ユーザー単位のセッション分離で保護 |

### code / container の差 (ネットワーク観点)

ネットワーク分離での実質差は **ACR だけ**。

- container モード: image を ACR に build/push するため、ACR を `publicNetworkAccess: Disabled` + private endpoint にして build/pull を private 経路に閉じられる。
- code モード: ACR を使わない (zip を Foundry マネージド経路で送る) ため、**ACR の PE 化という論点自体が発生しない**。その分、分離構成がシンプル。

### 注意点 (公式の制約)

- 完全分離の Standard セットアップでは **Azure Storage / Azure AI Search / Azure Cosmos DB を BYO** する必要がある (データが自テナントに留まる要件)。
- Search / Storage / Cosmos DB の private endpoint は Foundry デプロイ時に **自動作成されない**ため、個別に作成が必要。
- **エージェントエンドポイント自体の private 化は preview では不可** (プラットフォーム側の将来機能)。
- IP 範囲 `172.17.0.0/16` は Docker bridge 予約のため VNet に使わない。

> [!NOTE]
> このワークショップ ([Lab 3](../../docs/03-foundry-deploy.md)) は public egress 前提のシンプル構成。上記の VNet 分離は「本番でネットワーク要件がある場合の拡張」という位置づけ。

出典:

- [Networking options for Foundry Agent Service (Microsoft Learn)](https://learn.microsoft.com/azure/foundry/agents/concepts/networking-options)
- [Set up private networking for Foundry Agent Service (Microsoft Learn)](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks)
- [How to configure network isolation for Microsoft Foundry (Microsoft Learn)](https://learn.microsoft.com/azure/foundry/how-to/configure-private-link)

---

## まとめ

- **code = 速度・手軽さ**を取る。中身 (ソース) をそのまま置き、環境構築は Foundry 任せ。
- **container = 柔軟性・制御・監査性**を取る。環境ごと固めた image を置き、中身は自分で保証。
- OS レベルの依存やカスタム base image が要らないなら **code モードで十分**。それらが要件になった時に container モードへ切り替えるのが基本方針。

本ワークショップは **code モードが主導線**で、container モードは付録 A の Stretch 扱いです ([Lab 3 付録 A](../../docs/03-foundry-deploy.md))。

## 参考

- [Lab 3: Hosted Agent を Foundry へデプロイ](../../docs/03-foundry-deploy.md)
- [`kb-1.8.0/api-reference/1.8.0/hosted-agent-deploy.md`](../../kb-1.8.0/api-reference/1.8.0/hosted-agent-deploy.md)
- [Microsoft Foundry で hosted agent をデプロイする (Microsoft Learn)](https://learn.microsoft.com/azure/ai-foundry/)
