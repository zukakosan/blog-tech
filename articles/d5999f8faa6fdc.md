---
title: "Azure Resource MoverでVMをいろいろお引越ししてみる"
emoji: "🚚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","VM","cloud"]
published: true
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

## リージョン間の移動
- 先ほどサブスクリプション間で移動したVM群をそのまま別リージョンに持っていこうと設定
![](/images/20230901-rscmvr/10.png)

- ただし、以下のようなエラーで終了
```
{"code":"ResourceValidationFailed","message":"Azure リソース: /subscriptions/cc690af5-c4e8-4a14-aeda-xxxxxxxx/resourceGroups/20230831-moved-rg/providers/Microsoft.Compute/virtualMachines/winserv-001 の検証に失敗しました。\n    考えられる原因: このリソースを検証できませんでした。\n    推奨アクション: この操作をもう一度お試しください。\n    ","details":[{"code":"VmSecurityTypeNotSupported","message":"\n      仮想マシンにサポートされていない構成があります。\n      考えられる原因: 仮想マシン - /subscriptions/cc690af5-c4e8-4a14-aeda-xxxxxxxx/resourceGroups/20230831-moved-rg/providers/Microsoft.Compute/virtualMachines/winserv-001 は、信頼されているか、機密情報で、サポートされていないものです。\n      推奨アクション: 仮想マシンのサポートされている構成を指定していることを確認してから、もう一度お試しください。\n    "}]}
```
- `信頼されているか、機密情報で、サポートされていない`ということでトラステッド起動が原因なんじゃないかと疑い、トラステッド起動ではないVMを作成して同じ設定をしてみるとうまくいった。

:::message
Azure Site Recoveryでもまだトラステッド起動のVMはサポートしていないので、そこは優劣はないと思われる

https://learn.microsoft.com/ja-jp/azure/virtual-machines/trusted-launch#unsupported-features
:::

![](/images/20230901-rscmvr/11.png)

- [依存関係の追加]をクリックして、依存関係の確認をしたのち、[準備]へと進む
![](/images/20230901-rscmvr/12.png)

![](/images/20230901-rscmvr/13.png)

- 先にリソースグループを移動しないとエラーになったため、先に移動させる
![](/images/20230901-rscmvr/14.png)

- 移動のコミット
![](/images/20230901-rscmvr/15.png)

- Errorの原因となっていた依存リソースのコミットが完了したため、VM群の準備に進む
- コミット完了後、リソースグループの削除ができるようになるが、そこは後回しにしておく

![](/images/20230901-rscmvr/16.png)

- VMの準備にかなり時間がかかったが、準備が終わったので移動を開始する
![](/images/20230901-rscmvr/17.png)

- 移動のコミットが保留中、という状態になったのでリソース自体のリージョン移行は完了しているはず
![](/images/20230901-rscmvr/18.png)

- 移行先のリソースグループを見てみる
- すると、確かに既存リソースグループを移行した`xxxx-westus`に対して`westus`でリソースが追加されていることがわかる
![](/images/20230901-rscmvr/19.png)

- 移動のコミットを実施し、移動を確定させる(この感じはASRに似ている)
![](/images/20230901-rscmvr/20.png)

- ソースの削除を実施しない限りは、元のリソースグループにもリソースが存在する状況となる
![](/images/20230901-rscmvr/21.png)
![](/images/20230901-rscmvr/22.png)

- ソースとしてリソースグループの削除は非対応らしいので、それ以外の削除を進める
![](/images/20230901-rscmvr/23.png)

- 依存関係があって削除できないリソースはエラーを見ながらいい感じに対処
- 移動が完了したことを確認し、元のリソースグループを見てみると空になっている
![](/images/20230901-rscmvr/24.png)
![](/images/20230901-rscmvr/25.png)

- これでソースのリソースグループを削除すれば、移動完了!

# おわり
- やはりVMの移行についてはASRを利用するのが無難そう
- ASRっぽい名前のディスクが途中作成されていたりしたので、内部的には一部ASRを使っていそうな気もする
- なんでも移動できるわけじゃないので、Azure Resource Moverで移動できるリソースなのかという確認は必須
