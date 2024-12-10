---
title: "データ主権の観点から Azure OpenAI Service のデプロイの種類をガードレールにより制限して利用する"
emoji: "🍩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure","microsoft","AOAI","AI"]
published: true
publication_name: "microsoft"
published_at: 2024-12-11 09:00
---
この記事は、[Microsoft Azure Tech Advent Calendar 2024](https://qiita.com/advent-calendar/2024/microsoft-azure-tech) 11 日目の記事です。
# はじめに
Azure OpenAI Service のモデルにはデプロイの種類[^1]という概念があります。PTU[^2] を除くと、Standard、Global-Standard、Global-Batch の 3 種類に分けられます。 それぞれの違いを表にまとめると次のようになります。

| サービス形態 | Global-Batch | Global-Standard | Standard |
| ---- | ---- | ---- | ---- |
| 用途 | ・オフラインでの推論処理<br>・遅延に敏感ではなく数時間かかってもよいワークロード | ・最初の出発点として考慮する<br>・Global-Standard では、Standard よりも高い既定クォータとより多くのモデルを利用可能 | ・データ所在地の要件がある場合<br>・中程度以下のボリューム用に最適化 |
| コスト | Global Standard の価格と比べて 50% 安価[^3] | 相対的にやや高価[^4] | 相対的に高価[^4] |
| メリット | Global-Standard と比較しても大幅にコストダウン | ・既定の TPM としては最も高く、すべての新しいモデルに簡単にアクセスできる<br>・使用量が多いと、待ち時間の変動が大きくなる可能性がある  | ・データ処理が同じリージョンに留まる<br>・可用性に関する SLA が存在 |
| デメリット | ・リアルタイム呼び出しは不可<br>・推論のためのデータ処理に関して任意の Azure OpenAI Service の場所で実行される可能性がある | 推論のためのデータ処理に関して任意の Azure OpenAI Service の場所で実行される可能性がある | TPM が相対的に低いため、相対的にみると遅延が発生しやすい |

[^1]:https://learn.microsoft.com/ja-jp/azure/ai-services/openai/how-to/deployment-types
[^2]:https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/provisioned-throughput
[^3]:https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/
[^4]:ttps://azure.microsoft.com/pricing/details/cognitive-services/openai-service/


デプロイの種類の選択によって、データが処理されるリージョンが異なるということは、一部の要件に抵触してしまう可能性があります。コンプライアンスを意識して安全に AOAI を利用するには、この違いを意識しておく必要があります。

また、PoC やサンドボックス環境の活用を推進するためにも、そのような観点を抑え、ガードレールを用意していくことが重要です。

# デプロイの種類をガードレールで制御
AI の民主化を進めるには、最低限のガードレールを用意することが重要になります。組織のデータは日本国外のリージョンに流出してはいけないという要件がある場合は、そのような要件が確実に満たされるように運用やガバナンスを設計しなくてはいけません。

## Azure Policy の構成
Azure でのガードレール実装には、Azure Policy が利用できます。今回のケースでは、組み込みポリシーは提供されていないため、以下のようなカスタム ポリシーを作成します。例えば、今回は「データがリージョン内に収まるように `Standard` デプロイを強制する（それ以外はデプロイ不可）」を実現したいとします。ある程度 AI を自由に使って検証させたいけど、本番データが外に出ると困る、というケースですね。

実際に作っていきます。まず、[Policy] > [Definition] > [+ Policy definition] からポリシー定義を作成します。
![](/images/20241210-aoaideploytyperegulate/01.png)

[POLICY RULE] には以下の内容を記述します。必要に応じて、パラメータなどを使いながら使いやすい定義にしてください。

```json:regulate-model-deploy-type
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.CognitiveServices/accounts/deployments"
        },
        {
          "not": {
            "field": "Microsoft.CognitiveServices/accounts/deployments/sku.name",
            "equals": "Standard"
          }
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```
作成出来たらこのポリシーを割り当てていきます。スコープに任意のものを設定します。
![](/images/20241210-aoaideploytyperegulate/02.png)

また、わかりやすいように非準拠時のメッセージを記載しておきます。
![](/images/20241210-aoaideploytyperegulate/03.png)

割り当てが完了すると、Assignments に表示されるため、それを開きます。
![](/images/20241210-aoaideploytyperegulate/04.png)

数分待ったうえで、[View compliance] から状況を確認します。すでにデプロイ済みのモデルのうち、非準拠のものがこの時点でマークされます。
![](/images/20241210-aoaideploytyperegulate/05.png)

## Azure AI Foundry 上でのモデルの新規デプロイ
Azure AI Foundry 上で改めてモデルをデプロイします。`Global Standard` を選択した場合は、モデルのデプロイに失敗し、メッセージが表示されます。
![](/images/20241210-aoaideploytyperegulate/06.png)

`Standard` を選択すると、問題なくデプロイできます。
![](/images/20241210-aoaideploytyperegulate/07.png)


# おわりに
今回は、安全な AI 活用に向けたガードレール整備として、データ主権の観点からモデルのデプロイ タイプを制限するということを試しました。アジリティを担保して、AI 活用を民主化するためにもこうした安全に運用する仕組みは非常に重要です。

モデルの種類自体を制限する[^5] という過去の記事も合わせてみていただくことをお勧めします。

[^5]: https://zenn.dev/microsoft/articles/20241115-modelcontrol