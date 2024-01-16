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

### ファイル構成
今回は以下のような構成にしました。`spoke-vnet.tf` を使いまわして任意の数の Spoke ネットワークをデプロイできるように作りこむことも考えたのですが、今回そこまでは行っていません。
```
/
├── hub-vnet.tf
├── spoke-vnet.tf
├── variables.tf
├── providers.tf
└── output.tf
```

### Azure Firewall のルール作成順序
Bicep で記述したときは、ネットワークルールの作成と DNAT ルールの作成が同時に走るとエラーになるような状況に遭遇したため、明示的な依存関係を記述しました。Terraform の場合はそこは特に気にせずともエラーは生じませんでした。

```hcl:hub-vnet.tf
# Create Azure Firewall Policy Rule Collection Group (Network Rule)
resource "azurerm_firewall_policy_rule_collection_group" "azfw_netrule_collection" {
  name               = "DefaultNetworkRuleCollectionGroup"
  firewall_policy_id = azurerm_firewall_policy.azfw_hub_policy.id
  priority           = 200
  network_rule_collection {
    name     = "DefaultNetworkRuleCollection"
    action   = "Allow"
    priority = 200
    rule {
      name                  = "allow-east-west-traffic"
      protocols             = ["Any"]
      source_addresses      = [element(local.spoke001_address_space, 0), element(local.spoke002_address_space, 0)]
      destination_ports     = ["*"]
      destination_addresses = [element(local.spoke001_address_space, 0), element(local.spoke002_address_space, 0)]
    }
  }
}


# Create Azure Firewall Policy Rule Collection Group (DNAT Rule)
resource "azurerm_firewall_policy_rule_collection_group" "azfw_natrule_collection" {
  name               = "DefaultDnatRuleCollectionGroup"
  firewall_policy_id = azurerm_firewall_policy.azfw_hub_policy.id
  priority           = 200
  nat_rule_collection {
    name     = "DefaultDnatRuleCollection"
    priority = 200
    action   = "Dnat"
    rule {
      name                = "nat-jumpbox"
      source_addresses    = ["*"]
      destination_address = azurerm_public_ip.azfw_hub_pip.ip_address
      destination_ports   = ["22"]
      translated_address  = azurerm_network_interface.vm_jumpbox_nic.ip_configuration[0].private_ip_address
      translated_port     = "22"
      protocols           = ["TCP"]
    }
  }
}

```

### サブネットのループによるデプロイ
ループの使い方に慣れるためにサブネットについては~~無理やり~~ループでデプロイしてみました。
```hcl:hub-vnet.tf
resource "azurerm_subnet" "hub_subnets" {
  for_each             = { for i in var.hub_subnets : i.name => i }
  name                 = each.value.name
  resource_group_name  = azurerm_resource_group.rg_hub.name
  virtual_network_name = azurerm_virtual_network.hub_vnet.name
  address_prefixes     = [each.value.address_prefixes]
}
```

### VM の構成変更時に NIC が邪魔をする？
トライアンドエラーを繰り返しながら VM のプロパティを更新したので再デプロイしようとしたところ、以下のようなエラーに遭遇しました。NIC が使用中なので削除できない、といった趣旨のエラーでした。NIC 自体を削除したいわけではないのですが、対処が必要です。

```
│ Error: deleting Network Interface (Subscription: "xxxx"
│ Resource Group Name: "tf-deploy-spoke-rg"
│ Network Interface Name: "vm-spoke001-nic"): performing Delete: unexpected status 400 with error: NicInUse: Network Interface /subscriptions/xxxx/resourceGroups/tf-deploy-spoke-rg/providers/Microsoft.Network/networkInterfaces/vm-spoke001-nic is used by existing resource /subscriptions/xxxx/resourceGroups/tf-deploy-spoke-rg/providers/Microsoft.Compute/virtualMachines/vm-spoke001. In order to delete the network interface, it must be dissociated from the resource. To learn more, see aka.ms/deletenic.
```
NIC が使用中だから消せないということであれば、その NIC に依存しているリソース(つまりは VM)を削除することで解決できました。削除方法としては、Terraform のソースコード上で VM の宣言部分を丸ごとコメントアウトしました。
![](/images/20240116-terraform-hubspoke/message-01.png)
![](/images/20240116-terraform-hubspoke/message-02.png)

Azure portal で VM の側だけ削除したときに NIC が宙に浮いて残るようなイメージですね。その後、コメントアウトを外し、VM 宣言部分も含めてデプロイすると、望ましい形で VM が作成されました。
![](/images/20240116-terraform-hubspoke/message-03.png)

# まとめ