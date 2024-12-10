---
title: "Azure 生成 AI 活用のガードレール整備:「Azure AI Foundry にデプロイ可能な LLM モデルの種類を制限する」編"
emoji: "🍩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure","AOAI","microsoft","genai"]
published: true
publication_name: "microsoft"
---

# はじめに
生成 AI の活用がある程度進んできて、最近ではどのように安全に使用するか、安全に開発者に使ってもらうかという点がフォーカスされています。

ロール ベースのアクセス制御 (RBAC) により、Azure AI Foundry など AI 用の統合開発環境の払い出しは簡単にできるようになっています。そして、モデルカタログには多種多様な LLM モデルが提供されており、意図しないモデルの利用は意図しない応答を伴う可能性があり、現場に混乱を生んでしまう可能性もあります。

Azure Policy を利用することで、Azure AI Foundry 上で利用する LLM モデルの種類を制限することができます。

# 手順

## カスタム ポリシーの作成
この制御には、カスタム ポリシーを利用します。Azure Policy の画面から、新規作成します。
![](/images/20241115-modelcontrol/modelgov01.png)

定義の内容としては、以下の内容を入力します。そこまで複雑なものではないため、解読できると思います。`Microsoft.CognitiveServices/accounts/deployments` から、`model.name` と `model.version` を抜き出したものを連結し、それがポリシーの割り当て時に入力する文字列と一致するかを判断しています。
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
                "value": "[concat(field('Microsoft.CognitiveServices/accounts/deployments/model.name'), ',', field('Microsoft.CognitiveServices/accounts/deployments/model.version'))]",
                "in": "[parameters('allowedModels')]"
              }
            }
          ]
        },
        "then": {
          "effect": "deny"
        }
      },
      "parameters": {
        "allowedModels": {
          "type": "Array",
          "metadata": {
            "displayName": "Allowed AI models",
            "description": "The list of allowed models to be deployed."
          }
        }
      }
}
```

## カスタム ポリシーの割り当て
ポリシー定義が作成出来たら、一覧からそれを選択し、割り当てます。
![](/images/20241115-modelcontrol/modelgov02.png)

![](/images/20241115-modelcontrol/modelgov03.png)

パラメータの設定で、許可するモデルを "モデル名,バージョン" の組として複数指定し、角かっこで囲います(これが、先述の concat した文字列との比較対象になります)。ここでは、`gpt-4` のバージョン `0613` と、`gpt-35-turbo` の `0613` を許可しています。
![](/images/20241115-modelcontrol/modelgov04.png)

必要に応じて非準拠の場合に表示するメッセージを設定します。
![](/images/20241115-modelcontrol/modelgov05.png)

設定できたらそのまま割り当てます。

## 挙動の確認
ポリシーを割り当ててしばらくすると、スコープ内に既にデプロイされているモデル一覧が表示され非準拠のものがマークされます。非準拠の理由には先ほど設定したメッセージ、`Current Value` には現在の値、`Target Value` には許可されている値が表示されます。
![](/images/20241115-modelcontrol/modelgov06.png)

次は、ポリシーを割り当てたスコープ内の Azure AI Foundry 上でモデルを新規でデプロイします。許可した `gpt-4` のバージョン `0613` は何事もなくデプロイできます。
![](/images/20241115-modelcontrol/modelgov07.png)

一方で、`gpt-4` の許可されていないバージョンをデプロイしようとすると、`permission error` が表示され、デプロイがブロックされます(エラー メッセージが分かりづらいですが)。
![](/images/20241115-modelcontrol/modelgov08.png)

:::message
### 2024/11/28 追記
試してみると次のようにポリシーによって拒否されていることが分かりやすくなりました。Ignite で Azure AI Foundry にリブランドされるにあたって改善されている模様です。
![](/images/20241115-modelcontrol/update01.png)
:::

逆に、Azure Policy の割り当て時のパラメータにこのバージョンを追加すると、問題なくデプロイできます。
![](/images/20241115-modelcontrol/modelgov09.png)

![](/images/20241115-modelcontrol/modelgov10.png)

このようにして、モデルの種類自体に対する予防的ガードレールを設定することができます。

# おわりに
実は、カスタムではなく組み込みでも同じようなポリシーが出てきているのですが、若干挙動が怪しいところがあり今回は紹介から外しています。興味のある方はこちらのドキュメント[^1] をご覧ください。Azure Policy を使いこなしながら、アジリティとコントロールを両立していきましょう。

[^1]: https://learn.microsoft.com/ja-jp/azure/ai-studio/how-to/built-in-policy-model-deployment