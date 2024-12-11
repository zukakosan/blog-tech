---
title: "Azure 生成 AI 活用のガードレール整備:「責任ある AI の観点から LLM に対して特定のコンテンツ フィルターを強制する」編"
emoji: "🍩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure","microsoft","AOAI","GenAI"]
published: true
publication_name: "microsoft"
published_at: 2024-12-13 07:00
---
この記事は、[Microsoft Azure Tech Advent Calendar 2024](https://qiita.com/advent-calendar/2024/microsoft-azure-tech) 13 日目の記事です。

# はじめに
Azure OpenAI Service では責任ある AI[^1] の観点から、コンテンツ フィルター[^2] の設定が義務付けられています。コンテンツ フィルターとは、入力プロンプトと (出力される) 入力候補の両方で、有害な可能性があるコンテンツ特有のカテゴリを検出し、防止処理を行います。

Azure OpenAI Service では既定の状態でいくつかのコンテンツ フィルターを提供していますが、要件レベルに応じてカスタマイズできます。一部の信頼されたお客様においては、申請によって既定のコンテンツ フィルターを緩める方向へ調整が可能ですが、それ以外の場合はより厳しくする方向にのみ調整可能です。

組織の倫理的な基準やユーザ保護の観点、また法的基準への準拠の観点から、適切なレベルで LLM の出力を制御する必要があります。

[^1]:https://learn.microsoft.com/en-us/legal/cognitive-services/openai/overview
[^2]:https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/content-filter?tabs=warning%2Cuser-prompt%2Cpython-new

# 特定のコンテンツ フィルターの適用を強制する
ワークロードの厳密性によっては、特定のカスタマイズされたコンテンツ フィルターを適用しなくてはならないというケースもあるでしょう。実際にどのように実装していくのか見ていきましょう。

## カスタム コンテンツ フィルターの作成
まずは Azure AI Foundry 上で、コンテンツ フィルターを作成していきます。[Safety + security] > [+ Create content filter] から作成します。
![](/images/20241213-enforce-content-filter/01.png)

より厳しい倫理規制があるユース ケースを想定しているため、[Input filter] および [Output filter] において、すべてを `High` に設定します。
![](/images/20241213-enforce-content-filter/02.png)
![](/images/20241213-enforce-content-filter/03.png)

ここは一旦チェックを付けず、既存のモデルには適用せずに進めます。
![](/images/20241213-enforce-content-filter/04.png)

コンテンツ フィルターが作成されていることを確認します。あとで利用するため、ここで作成したコンテンツ フィルターの名前を控えておきます。
![](/images/20241213-enforce-content-filter/05.png)

## Azure Policy の構成
特定のコンテンツ フィルターを強制したい場合でも、やはり Azure Policy を使って制御できます。[Definitions] > [+ Policy definition] からポリシー定義を新規で作成します。
![](/images/20241213-enforce-content-filter/06.png)

[POLICY RULE] に以下の内容を記述します。

```json
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
						"field": "Microsoft.CognitiveServices/accounts/deployments/raiPolicyName",
						"in": "[parameters('filterNames')]"
					}
				}
			]
		},
		"then": {
			"effect": "deny"
		}
	},
	"parameters": {
		"filterNames": {
			"type": "Array",
			"metadata": {
				"displayName": "Allowed Content filters",
				"description": "The list of allowed Content filters"
			}
		}
	}
}
```

定義が作成出来たら、[Assign policy] から割り当てていきます。[Parameters] 画面で、`Allowed Content filters` に選択肢として与えたいコンテンツ フィルターのリストを配列として入力します。ここでは先ほど作成したものを一つだけ入力します(`["CustomContentFilter417"]`)。複数ある場合はカンマ区切りで入力します。ほかの設定項目は既定のまま割り当てます。
![](/images/20241213-enforce-content-filter/08.png)

数分待ったうえで、[View compliance] から状況を確認します。すでにデプロイ済みのモデルのうち、非準拠のものがこの時点でマークされます。
![](/images/20241213-enforce-content-filter/09.png)

## Azure AI Foundry 上でのモデルの新規デプロイ
既定で提供されているコンテンツ フィルタ(`DefaultV2`)を選択してデプロイすると、Azure Policy によって拒否されます。
![](/images/20241213-enforce-content-filter/10.png)

一方で、許可リストに入れた `CustomContentFilter417` を選択すると、問題なくデプロイできます。
![](/images/20241213-enforce-content-filter/11.png)

# おわりに
今回は、責任ある AI の観点から、特定のコンテンツ フィルタを適用するための方法について確認しました。原理的には LLM のデプロイの種類を制御する[^3] のとまったく同じですが、制御できることを知っておくと役に立つ場面はあると思います。また、モデルの種類を限定する方法[^4] も合わせてご確認いただき、Azure 生成 AI 活用のガードレール整備のイメージをお持ちいただけると幸いです。

[^3]:https://zenn.dev/microsoft/articles/20241210-aoaideploytyperegulate
[^4]:https://zenn.dev/microsoft/articles/20241115-modelcontrol