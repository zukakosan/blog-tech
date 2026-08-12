# Foundry IQ と MCP ツール選択の整理メモ

Foundry IQ / ナレッジベース（KB）が MCP ツールとしてどう機能し、エージェントの
ツール選択（呼び出し）がどう決まるかを、これまでの議論に沿ってまとめる。

> 参考（一次情報）
> - [Foundry IQ とは / Information retrieval](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-information-retrieval#extend-retrieval-by-using-foundry-iq-and-work-iq)
> - [Connect a Foundry IQ knowledge base to Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/foundry-iq-connect)
> - [Query a knowledge base using the retrieve action or MCP endpoint](https://learn.microsoft.com/azure/search/agentic-retrieval-how-to-retrieve#call-the-mcp-endpoint)
> - [Create a knowledge base in Azure AI Search](https://learn.microsoft.com/azure/search/agentic-retrieval-how-to-create-knowledge-base)

---

## 0. 結論（理解の確認・TL;DR）

- **KB は「AI Search が提供する MCP サーバー」としてシンプルに見える。**
  Foundry Agent から見ても、外部クライアントから見ても同じ（内外で挙動は変わらない）。
- **ツール選択は name / description で呼び出し元 LLM が判断する。**
  ただし KB のツール名は全 KB 共通で `knowledge_base_retrieve` なので、
  KB 同士の見分けは **実質 description（＋server_label）が担う**。
- **agent instructions が効くのは、Foundry IQ 固有の特別機構ではない。**
  単に「KB をちゃんと使え」と呼び出し元 LLM に指示すれば、そう動くというだけ。
  KB 側には何のフックもかかっていない（＝呼び出し元の挙動制御）。

| 観点 | 一般 MCP ツール | KB（Foundry IQ） |
| --- | --- | --- |
| ツール名 | 固有 → **効く** | `knowledge_base_retrieve` 固定 → **効かない** |
| description | 効く | **効く（見分けの主役）** |
| 呼ぶ/呼ばないの精度 | instructions で制御 | instructions で制御（同じ） |

---

## 1. Foundry IQ とは何か

- **実体は Azure AI Search 上の「マネージドな agentic retrieval 実装」**。
  Foundry 専用の閉じたロジックではない。
- 取り込み（chunk 化・埋め込み生成・権限適用）を自動化し、その上に
  検索オーケストレーション（クエリ分解 → 並列検索 → セマンティック再ランク →
  出典付き統合）を載せたもの。
- **各ナレッジベース（KB）が「単独の MCP サーバー」**として公開される。
- Foundry Agent Service はその MCP サーバーに接続する「利用者の1つ」にすぎない
  （GitHub Copilot / Claude / Cursor など他の MCP クライアントからも呼べる）。

### 一言でいうと
> 「Azure AI Search が提供する agentic 検索を、MCP サーバーとして公開したもの」。
> Foundry からは MCP ツールとして接続して使う。

---

## 2. MCP サーバーとしての公開形

- 各 KB は次の MCP エンドポイントを持つ：
  ```
  https://<search-service>.search.windows.net/knowledgebases/<kb-name>/mcp?api-version=<version>
  ```
- 公開されるツールは **固定名で 1 つだけ**：`knowledge_base_retrieve`
- 呼ばれると内部で以下を実行し、出典付きの統合レスポンスを返す：
  1. ユーザークエリをサブクエリに分解
  2. keyword / vector / hybrid で並列検索
  3. セマンティック再ランク
  4. 結果を統合し、source 参照付きで返す

---

## 3. いつ Foundry IQ がキックされるか

- 常時走る前処理ではなく、**エージェント（LLM）が `knowledge_base_retrieve` の
  ツール呼び出しを選択したとき**に初めて起動する（毎回ではない）。
- 「呼ぶ / 呼ばない」を左右する主要因は **agent instructions（システムプロンプト）**。
  - ドキュメント推奨テンプレートも「KB ツールを必ず使え」と明示して
    **呼び出し率を上げる**ことを狙っている。

```text
Use the knowledge base tool to answer user questions.
If the knowledge base doesn't contain the answer, respond with "I don't know".
When you use information from the knowledge base, include citations to the retrieved sources.
```

> MS Copilot の「agent instructions に書くと Foundry Agent からの呼び出し精度が
> 上がる」は **正しい**。

---

## 4. ツール選択のメカニズム（重要）

LLM のツール選択は **description 単独ではない**。次を合わせて判断する：

```
ツールを呼ぶか / どれを呼ぶか
  = LLM の推論(
        agent instructions
      + 各ツールの name / description（MCP tools/list 由来）
      + input schema
      + ユーザー発話・会話履歴
    )
```

2 つの判断を分けて考えると整理しやすい：

| 判断 | 主に効くもの |
| --- | --- |
| そもそも呼ぶべきか（呼ぶ / 呼ばない） | **agent instructions が支配的** |
| 複数ツール／複数 KB のどれを呼ぶか | 各ツールの **name / description** |

---

## 5. KB の設定値と役割

KB のスキーマ（抜粋）：

```json
{
  "name": "my-kb",
  "description": "この KB が何を答えられるかの自然言語説明",
  "retrievalInstructions": "KB 内部で複数 knowledge source をどう振り分けるか",
  "answerInstructions": "回答の整形指示",
  "outputMode": "answerSynthesis",
  "knowledgeSources": [ ... ],
  "models": [ ... ],
  "retrievalReasoningEffort": { "kind": "low" }
}
```

| フィールド | 役割 | ツール選択への影響 |
| --- | --- | --- |
| `name` | KB 識別子（エンドポイント URL に入る） | ツール名は固定 `knowledge_base_retrieve` |
| **`description`** | KB の自然言語説明（tool-facing） | **複数 KB の見分けに効く** |
| `retrievalInstructions` | KB 内部の source ルーティング | 外部ツール選択には出ない（内部用） |
| `answerInstructions` | 回答生成の整形 | 選択には無関係 |

> 注意：「tools/list の description に KB の `description` がそのまま入る」と逐語的に
> 明記した公式記述は確認できていない。ただしスキーマ上、外部向けの自然言語説明は
> `description` のみで、これが tool-facing の説明として機能する位置づけ。

---

## 6. 複数 KB を使う場合

- **各 KB = それぞれ独立した MCP サーバー**（エンドポイントも別）。
- **各 KB がそれぞれ自分の `description` を持つ**。
- ただし **ツール名は全 KB 共通で `knowledge_base_retrieve`**（ここが盲点）。

| MCP サーバー | server_label（任意で付与） | ツール名（固定） | description |
| --- | --- | --- | --- |
| hr-policy-kb | `hr-kb` | `knowledge_base_retrieve` | KB1 の説明 |
| it-faq-kb | `it-kb` | `knowledge_base_retrieve` | KB2 の説明 |
| product-kb | `product-kb` | `knowledge_base_retrieve` | KB3 の説明 |

Python（Foundry）での登録イメージ：

```python
from azure.ai.projects.models import PromptAgentDefinition, MCPTool

mcp_hr = MCPTool(
    server_label="hr-kb",
    server_url="{search}/knowledgebases/hr-policy-kb/mcp?api-version=2026-05-01-preview",
    allowed_tools=["knowledge_base_retrieve"],
    project_connection_id="hr-conn",
    require_approval="never",
)
mcp_it = MCPTool(
    server_label="it-kb",
    server_url="{search}/knowledgebases/it-faq-kb/mcp?api-version=2026-05-01-preview",
    allowed_tools=["knowledge_base_retrieve"],
    project_connection_id="it-conn",
    require_approval="never",
)

agent = project_client.agents.create_version(
    agent_name=agent_name,
    definition=PromptAgentDefinition(
        model=agent_model,
        instructions=instructions,
        tools=[mcp_hr, mcp_it],   # 別サーバーとして並ぶ
    ),
)
```

ツール名が全部同じなので、複数 KB の見分けは
**description（＋ server_label ＋ instructions）** で行わせる。

---

## 7. 一般 MCP ツールと混在する場合

一般 MCP ツールは **name も description も個別に意味を持つ**ので、LLM は両方を読む。

| ツール | name | description | 見分けの主軸 |
| --- | --- | --- | --- |
| 天気 | `get_weather` | "Get current weather for a city" | name も description も効く |
| 送金 | `send_payment` | "Transfer money between accounts" | name だけでほぼ判別可能 |
| 社内 KB | `knowledge_base_retrieve` | KB の description | name 共通 → **description 頼み** |

→ 混在環境では **二層構造**で判断される：

- **一般ツール**：固有の name で大枠を絞り、description で詰める
- **KB 群**：name が共通なので description / server_label / instructions で見分ける

---

## 8. 実務上のまとめ

1. **呼び出し率・精度を上げたい** → まず **agent instructions** に「この種の質問では
   必ず KB を使え」と明示（最優先）。
2. **複数 KB / 一般ツールを見分けさせたい** → 各ツールの **description を具体的に**、
   KB は **server_label** も意味のある名前に。
3. **一般ツールは name を役割が伝わる命名**にすると選択が安定する
   （例：`search_hr_docs`）。
4. **KB 内部の source 振り分け**が必要なときだけ `retrievalInstructions` を使う。

### 優先度（ノブの効き順）

| 順位 | ノブ | 主な効果 |
| --- | --- | --- |
| 1 | agent instructions | 呼ぶ / 呼ばないの精度（支配的） |
| 2 | 各ツールの description（＋name） | 複数候補からの選択 |
| 3 | server_label | 人間・ログ上の識別、選択補助 |
| 4 | retrievalInstructions | KB 内部の source ルーティング |
