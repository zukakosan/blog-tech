---
title: "Azure リソースにリソースグループのタグを継承させる"
emoji: "🏷️"
type: "tech"
topics:
  - "azure"
  - "microsoft"
  - "tag"
published: true
published_at: "2022-11-30 10:44"
---

# モチベ
- 直近のCost Managementのアップデートでタグの継承機能がPreview
- そもそもタグは課金の棲み分けで利用されることが多いが、リソースグループから継承されないので若干扱いが手間だった

# 設定
- Cost Managementのここから設定。リソースに同じ名前のタグが付いているときにどちらを優先するかを設定可能。
![](https://storage.googleapis.com/zenn-user-upload/c5b2d4f298c3-20221130.png)

# 検証
- 適当にリソースグループと複数のリソースを作成してみる
- いや、既存の適当なリソースを使う

## リソースグループでタグを設定
- ![](https://storage.googleapis.com/zenn-user-upload/b461e8b7fd25-20221130.png)

- 反映されないなと思っていたところ、最大24時間かかるような記述が。
- Cost Management側に表示されるまでのラグ？タグ反映までのラグ？
- https://learn.microsoft.com/ja-jp/azure/cost-management-billing/costs/enable-tag-inheritance#usage-record-updates

## ということで結果待ち
反映されたら順次Update