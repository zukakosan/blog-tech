# GitHub Actions 学習メモ

このリポジトリで扱った、再利用可能ワークフロー、Action、権限、入力・出力についてのまとめです。

## 1. ワークフローの入力 `inputs`

再利用可能ワークフローでは、`on.workflow_call.inputs` に呼び出し元から受け取る値を定義します。

```yaml
on:
  workflow_call:
    inputs:
      node-version:
        description: Node.js version
        required: false
        default: "20"
        type: string
```

呼び出し元は、ジョブ直下の `with` から値を渡します。

```yaml
jobs:
  quality:
    uses: ./.github/workflows/reusable-node-quality.yml
    with:
      node-version: "24"
```

呼び出された側では、`${{ inputs.node-version }}` として参照します。プログラムに例えると、再利用可能ワークフローが関数、`inputs` が関数の引数に相当します。

シークレットは通常の入力と分けて、呼び出し元の `secrets` と呼び出し先の `on.workflow_call.secrets` で受け渡します。

## 2. `${{ }}` とコンテキスト

`${{ }}` は、GitHub Actionsの式を記述する構文です。変数参照だけでなく、条件式や関数も記述できます。

```yaml
${{ inputs.node-version }}
${{ env.NODE_ENV }}
${{ secrets.TOKEN }}
${{ github.repository }}
${{ always() }}
```

`run` で起動したシェルの中では、環境変数を通常のシェル構文でも参照できます。

```yaml
- run: echo "$NODE_ENV"
```

## 3. `runs-on` とコンテナ

通常のジョブでは、`runs-on` がジョブを実行するランナーを指定します。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

ジョブをDockerコンテナ内で実行する場合も、コンテナを動かすホストとして `runs-on` が必要です。

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: node:24
    steps:
      - run: node --version
```

この場合、`ubuntu-latest` はホスト、`node:24` は各 `run` ステップを実行する環境です。独自VMを使う場合はセルフホステッドランナーを用意し、そのラベルを `runs-on` に指定します。

## 4. `uses` の2つの意味

`uses` は、記述する階層によって呼び出す対象が異なります。

### Step内の `uses`: Actionを呼び出す

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
```

Actionは、1つのStepとして実行される再利用可能な処理です。実装形式には次の種類があります。

- JavaScript Action: ビルド済みJavaScriptをNode.jsで実行する
- Docker Action: Dockerコンテナを実行する
- Composite Action: 複数のStepをまとめて実行する

### Job直下の `uses`: 再利用可能ワークフローを呼び出す

```yaml
jobs:
  quality:
    uses: ./.github/workflows/reusable-node-quality.yml
```

こちらはワークフローのジョブ構成全体を呼び出します。呼び出し元の同じジョブには `runs-on` や `steps` を書きません。実際の `runs-on` と `steps` は呼び出されたワークフロー側に定義します。

まとめると、Step内の `uses` はAction、Job直下の `uses` は再利用可能ワークフローです。

## 5. Actionの構造

JavaScript Actionは、一般にリポジトリ内の `action.yml` で入力と実行ファイルを宣言します。

```yaml
name: Example Action
inputs:
  message:
    required: true
runs:
  using: node24
  main: dist/index.js
```

利用側は `with` で入力を渡します。

```yaml
steps:
  - uses: owner/action@v1
    with:
      message: Hello
```

たとえば `GrantBirki/comment@v2.1.1` は、`issue-number` と `body` などを入力として受け取り、GitHub APIを通じてIssueまたはPRへコメントする外部Actionです。PRはIssueの仕組みを内包しているため、コメント処理ではIssueコメント用APIが使われます。

第三者製Actionを本番で利用する場合、タグではなく検証済みのコミットSHAに固定すると、参照先の変更リスクを抑えられます。

## 6. `permissions` と `GITHUB_TOKEN`

`permissions` は、ジョブに自動発行される `GITHUB_TOKEN` がGitHub APIやリポジトリに対して実行できる操作を制限します。

```yaml
permissions:
  contents: read
  pull-requests: write
```

代表例は次のとおりです。

- `contents: read`: リポジトリの内容を読み取る
- `contents: write`: コミット、タグ、リリースなどを書き込む
- `pull-requests: write`: PRを更新したりコメントしたりする
- `issues: write`: Issueを更新したりコメントしたりする
- `pages: write`: GitHub Pagesへデプロイする
- `id-token: write`: OIDCトークンを発行する

ワークフロー全体とジョブ単位の両方で設定できます。必要最小限の権限だけを与えるのが基本です。

通常の `GITHUB_TOKEN` は、そのワークフローを実行しているリポジトリを対象とします。他のプライベートリポジトリへアクセスする場合は、対象リポジトリへの権限を持つGitHub Appトークン、Fine-grained PAT、Deploy keyなどを別途用意します。

## 7. 再利用可能ワークフローへの権限継承

再利用可能ワークフローに渡されるトークン権限は、呼び出し元が上限を決めます。呼び出しが深くなっても、権限は維持または縮小しかできず、途中で拡大できません。

```text
caller: contents write
  -> reusable A: contents read
    -> reusable B: contents read
```

`reusable A` で `read` まで縮小した後、`reusable B` が `write` に戻すことはできません。また、呼び出し元が `contents: read` しか渡していない場合、呼び出し先が `contents: write` を要求しても、書き込み権限は得られません。

ネストされた再利用可能ワークフローは、最上位ではなく、直接の呼び出し元から渡された権限を上限として引き継ぎます。

## 8. `needs` による依存関係

`needs` は、ジョブ間の実行順序と参照可能な依存ジョブを定義します。

```yaml
jobs:
  comment:
    needs: github-pages
```

この定義には2つの役割があります。

1. `comment` ジョブを `github-pages` の完了後に実行する
2. `needs` コンテキストから `github-pages` の結果と出力を参照可能にする

参照できる主な値は次の2種類です。

```yaml
${{ needs.github-pages.result }}
${{ needs.github-pages.outputs.page_url }}
```

`result` は `success`、`failure`、`cancelled`、`skipped` のいずれかです。複数の依存ジョブを指定する場合は配列形式も使えます。

```yaml
needs: [quality, github-pages]
```

`needs: github-pages` が依存関係と参照可能範囲を宣言し、`needs.github-pages.outputs.page_url` が具体的な値を参照します。`needs` を宣言しただけでは、出力がローカル変数のように直接展開されるわけではありません。

## 9. Stepから呼び出し元までの出力

再利用可能ワークフローの値を呼び出し元へ返すには、出力を段階的に公開します。

```text
Stepの出力
  -> Jobの出力
    -> workflow_callの出力
      -> 呼び出し元のneedsコンテキスト
```

### StepからJobへ

出力元のStepに `id` を付け、Jobの `outputs` から参照します。

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      page_url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

### Jobから再利用可能ワークフロー外へ

`on.workflow_call.outputs` でJob出力を公開します。

```yaml
on:
  workflow_call:
    outputs:
      page_url:
        description: Deployed GitHub Pages URL
        value: ${{ jobs.deploy.outputs.page_url }}
```

### 呼び出し元で受け取る

呼び出し元は、対象ジョブを `needs` に指定して出力を参照します。

```yaml
jobs:
  comment:
    needs: github-pages
    steps:
      - run: echo '${{ needs.github-pages.outputs.page_url }}'
```

このリポジトリの流れは次のとおりです。

```text
steps.deployment.outputs.page_url
  -> jobs.deploy.outputs.page_url
    -> workflow_call.outputs.page_url
      -> needs.github-pages.outputs.page_url
```

## 10. `pull_request` イベント

```yaml
on:
  pull_request:
    branches:
      - main
```

この設定は、マージ先が `main` のPRを対象にします。デフォルトでは、PR作成時の `opened`、再オープン時の `reopened`、Open中のPRへ追加コミットがpushされたときの `synchronize` などで実行されます。

したがって、PRを作成した後に同じブランチへコミットを追加してpushすると、ワークフローは再実行されます。

## 11. テストカバレッジ

このリポジトリの `npm run test:coverage` は、次のスクリプトを実行します。

```bash
vitest run --coverage src
```

`src` 配下のテストを一度実行し、行、関数、分岐などがテストで実行された割合を計測します。結果はターミナルに表示され、詳細なレポートは `coverage/` に生成されます。

## 12. 全体像

```text
ci.yml
  |-- quality
  |     `-- reusable-node-quality.ymlをJobとして呼び出す
  |
  |-- github-pages
  |     `-- deploy-pages.ymlをJobとして呼び出す
  |           |-- build Job
  |           `-- deploy Job
  |                 `-- actions/deploy-pages ActionをStepとして呼び出す
  |
  `-- comment
        `-- GrantBirki/comment ActionをStepとして呼び出す
```

重要な区別は次のとおりです。

- `workflow_call.inputs`: 再利用可能ワークフローへの引数
- Job直下の `uses`: 再利用可能ワークフローの呼び出し
- Step内の `uses`: Actionの呼び出し
- `permissions`: `GITHUB_TOKEN` の権限
- `needs`: Job間の依存関係と出力の参照元
- `outputs`: Step、Job、ワークフローの境界を越えて値を渡す仕組み