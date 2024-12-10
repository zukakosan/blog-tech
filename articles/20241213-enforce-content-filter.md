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
Azure OpenAI Service では責任ある AI[^1] の観点から、コンテンツ フィルター[^2] の設定が義務付けられています。コンテンツ フィルターとは、LLM との対話aaaaaaaaaaaaaaaaaa

一部の信頼されたお客様においては、Default のコンテンツ フィルターを緩める方向性での調整が可能ですが、それ以外の場合はより厳しくする方向にのみ調整可能です。

組織の倫理的な基準やユーザ保護の観点、また法的基準への準拠の観点から、適切なレベルで LLM の出力を制御する必要があります。

[^1]:https://learn.microsoft.com/en-us/legal/cognitive-services/openai/overview
[^2]:https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/content-filter?tabs=warning%2Cuser-prompt%2Cpython-new