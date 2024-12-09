---
title: "データ主権の観点から Azure OpenAI Service のデプロイの種類をガードレールにより制限して利用する"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに
Azure OpenAI Service のモデルにはデプロイの種類[^1]という概念があります。PTU を除くと、Standard、Global-Standard、Global-Batch の 3 種類に分けられます。 それぞれの違いを表にまとめると次のようになります。

| サービス形態 | Global-Batch | Global-Standard | Standard |
| ---- | ---- | ---- | ---- |
| 用途 | ・オフラインでの推論処理<br>・遅延に敏感ではなく数時間かかってもよいワークロード | ・最初の出発点として考慮する<br>・Global-Standard では、Standard よりも高い既定クォータとより多くのモデルを利用可能 | ・データ所在地の要件がある場合<br>・中程度以下のボリューム用に最適化 |
| コスト | Global Standard の価格と比べて 50% 安価[^2] | https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/ | https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/ |
| メリット | Global-Standard と比較しても大幅にコストダウン | ・既定の TPM としては最も高く、すべての新しいモデルに簡単にアクセスできる<br>・使用量が多いと、待ち時間の変動が大きくなる可能性がある  | ・データ処理が同じリージョンに留まる<br>・可用性に関する SLA が存在 |
| デメリット | ・リアルタイム呼び出しは不可<br>・推論のためのデータ処理に関して任意の Azure OpenAI Service の場所で実行される可能性がある | 推論のためのデータ処理に関して任意の Azure OpenAI Service の場所で実行される可能性がある | TPM が相対的に低いため、相対的にみると遅延が発生しやすい |

[^1]:https://learn.microsoft.com/ja-jp/azure/ai-services/openai/how-to/deployment-types
[^2]:https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/

デプロイの種類の選択によって、データが処理されるリージョンが異なるということは、一部の要件に抵触してしまう可能性があります。コンプライアンスを意識して安全に AOAI を利用するには、この違いを意識しておく必要があります。

また、PoC やサンドボックス環境の活用を推進するためにも、そのような観点を抑え、ガードレールを用意していくことが重要です。

# デプロイの種類をガードレールで制御

