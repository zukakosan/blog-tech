---
title: "Azure Resource MoverでVMをいろいろお引越ししてみる"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","VM","cloud"]
published: false
publication_name: "microsoft"
---
# モチベ
- Azure Resource Moverというサービス？仕組みを最近知ったので動作検証がてらまとめてみる
- このサービス自体は結構前からあるみたいです
- 今回のようなVMの移行の文脈では、Azure Site Recoveryを利用した方が無難そうではある
- ただ、Azure Resource Moverを試してみたい

# Azure Resource Mover
- Azureリソースをサブスクリプション間・リソースグループ間・リージョン間で移動できるサービス

https://learn.microsoft.com/ja-jp/azure/resource-mover/overview

# とりあえず触ってみる
## 事前準備：VMの作成
- East USにVMを作成
![](/images/20230901-rscmvr/01.png)

## サブスクリプション間の移動
- Azure Resource Moverを開き、[サブスクリプション間での移動]を選択
![](/images/20230901-rscmvr/02.png)

- ターゲットの情報を入れていく
![](/images/20230901-rscmvr/03.png)

- 移動したいリソースを選択(VMをVNetなしで移動できるのか？と思いつつチェックを外している)
![](/images/20230901-rscmvr/04.png)

- 一旦これで進めてみると、パブリックIPはサブスクリプション間で移動できないらしくエラーになった
![](/images/20230901-rscmvr/05.png)

```
{"message":"リソースの移動の検証に失敗しました。詳細を確認してください。 (コード: ResourceMoveProviderValidationFailed) 要求内の 1 つ以上のリソースを移動できません。詳細については、各リソースの情報を確認してください。 (コード: CannotMoveResource、ターゲット: Microsoft.Network/publicIPAddresses) タイプ publicIPAddresses のリソースの移動はサポートされていません。移動要求にそのタイプのリソース /subscriptions/SubID/resourceGroups/20230831-armover/providers/Microsoft.Network/publicIPAddresses/winserv-001-ip が含まれています。 (コード: MoveNotSupported)","code":"ResourceMoveProviderValidationFailed","name":"d34fcae2-15c6-48d4-8d6c-3fa907b14020","status":409}
```
- Public IPを対象リソースから外して再度チャレンジ
- 今度はPublic IPとVNETに対して移動したいリソースが依存関係を持っているためエラー終了
![](/images/20230901-rscmvr/06.png)

- よってNICのIP構成からPublic IPの割り当てを解除し、VNETを移動対象に追加したうえで再度チャレンジ
- するとレビューに成功
![](/images/20230901-rscmvr/07.png)

:::message
Azure Resource Moverのオプション、よく考えたらここと一緒だったということにここで気づく
![](/images/20230901-rscmvr/08.png)
:::

- そのまましばらく放置して、確認したところサブスクリプションが移動していた(かなり時間かかった)
![](/images/20230901-rscmvr/09.png)
