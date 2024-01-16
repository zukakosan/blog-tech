---
title: "Terraform で Azure 上に Hub-Spoke 構成のアーキテクチャをデプロイする"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに
Azure にリソースを立てる上で Hub-Spoke 構成のアーキテクチャを意識することが多いと思います。Azure ネイティブの IaC ツールである Bicep で Hub-Spoke 構成を作るという検証は以前の記事[^1] にまとめています。今回は、似たような構成を Terraform で作ってみようというお話です。書き方はいろいろあると思うので、あくまで一例としてご参考にいただければと思います。検証したタイミングとまとめているタイミングにラグがあり、おぼろげな記憶でポイントを記載しております。

[^1]: https://zenn.dev/microsoft/articles/3121cdcbdace45


# アーキテクチャ
今回の構成では、Hub ネットワークに Azure Bastion をデプロイせず、Azure Firewall の DNAT によって Jumpbox にログインして VM を管理するようなイメージとしました。

![](/images/20240116-terraform-hubspoke/terraform-hub-spoke-archtecture.png)

## Terraform 上のポイント
ソースコードは GitHub[^2] で公開しています。
[^2]: https://github.com/zukakosan/terraform-learn/tree/main/20231030-HubSpoke


:::message
## VM の構成を変更したいときに NIC が邪魔をする？
トライアンドエラーを繰り返しながら VM のプロパティを更新したので再デプロイしようとしたところ、以下のようなエラーに遭遇しました。NIC が使用中なので削除できない、といった趣旨のエラーでした。NIC 自体を削除したいわけではないのですが、対処が必要です。

```
│ Error: deleting Network Interface (Subscription: "xxxx"
│ Resource Group Name: "tf-deploy-spoke-rg"
│ Network Interface Name: "vm-spoke001-nic"): performing Delete: unexpected status 400 with error: NicInUse: Network Interface /subscriptions/xxxx/resourceGroups/tf-deploy-spoke-rg/providers/Microsoft.Network/networkInterfaces/vm-spoke001-nic is used by existing resource /subscriptions/xxxx/resourceGroups/tf-deploy-spoke-rg/providers/Microsoft.Compute/virtualMachines/vm-spoke001. In order to delete the network interface, it must be dissociated from the resource. To learn more, see aka.ms/deletenic.
```

## 解決策: 先に NIC を参照している VM を削除
NIC が使用中だから消せないということであれば、その NIC に依存しているリソース(つまりは VM)を削除することで解決できました。削除方法としては、Terraform のソースコード上で VM の宣言部分を丸ごとコメントアウトしました。
![](/images/20240116-terraform-hubspoke/message-01.png)
![](/images/20240116-terraform-hubspoke/message-02.png)

Azure portal で VM の側だけ削除したときに NIC が宙に浮いて残るようなイメージですね。その後、コメントアウトを外し、VM 宣言部分も含めてデプロイすると、望ましい形で VM が作成されました。
![](/images/20240116-terraform-hubspoke/message-03.png)

:::

# まとめ