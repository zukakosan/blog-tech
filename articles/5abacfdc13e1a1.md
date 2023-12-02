---
title: "Terraform でディレクトリ構成を意識しながら Azure OpenAI Service の閉域環境を作る"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","terraform","IaC","microsoft","AOAI"]
published: true
published_at: 2023-12-21 09:00
---
本記事は、Microsoft Azure Tech Advent Calendar 2023[^1] の 21 日目の記事です。

[^1]:https://qiita.com/advent-calendar/2023/microsoft-azure-tech

https://qiita.com/advent-calendar/2023/microsoft-azure-tech

# はじめに

最近 IaC にハマっており、Bicep では飽き足らず Terraform に手を出し始めた zukako です。今回は、**Microsoft Azure** Tech Advent Calender 2023 にも関わらず Terraform を触る系の話です。構築する対象としては、Azure OpenAI Service の閉域化環境としました。ネットをいろいろと検索してみたのですが、まだ Terraform x Azure OpenAI Service の記事が少なそうに見えたので、ちょうどいい規模感かな～と思った次第です。閉域化といっても、独自のデータで Chat モデルを利用する、所謂「Add Your Data」の部分は含みませんのでご了承ください。イメージとしては過去の記事[^2] で触れているようなシンプルなものです。

[^2]:https://zenn.dev/microsoft/articles/198989f60eba61

また、Terraform を扱うにあたってある程度ディレクトリ構成も意識をしてみました。この規模の環境であればそこまで意識する必要はないかもしれないのですが、ご参考まで。

# 環境

## アーキテクチャ全体像
全体像としては以下のようなイメージになります。とてもシンプルです。
![](/images/20231221-terraform-aoai/architecture.png)

デプロイされるリソース一覧はこちらになります。
![](/images/20231221-terraform-aoai/resources.png)


## ソースコード

今回の環境をデプロイするためのファイルはこちらに配置しております。
https://github.com/zukakosan/terraform-learn/tree/main/20231201-aoai-private

### ディレクトリ構成

今回は Terraform らしい構成にするために敢えて以下のような構成としました。

```
.
├── env
│   ├── dev
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   ├── provider.tf
│   │   ├── variables.tf
│   │   └── dev.tfvars
│   └── prod
│        :(省略)
├── vnet
│   ├── main.tf
│   └── outputs.tf
└── vm
│   └── main.tf
└── aoai
    └── main.tf
```

# 実装上のポイント

## 各種リソースの module 化

これくらいの規模であれば module として読み込む必要があるのか微妙なところですが、なるべく疎結合になるようにしてみました。例えば `vnet` の作成は以下のように module として呼び出しています。`resource_group_name` は variable に同じ文字列を持っているため、そこから読み込むこともできるのですが、リソースグループ作成に対して依存関係を示すために `azurerm_resource_group` のインスタンスから参照しています。

```hcl
module "vnet" {
  source = "../../vnet"
  resource_group_name = azurerm_resource_group.aoai.name
  location = var.location
  vnet_name = var.vnet_name
  address_space = var.address_space
  jumpbox_subnet_address_space = var.jumpbox_subnet_address_space
  pe_subnet_address_space = var.pe_subnet_address_space
  client_ip = var.client_ip
}
```

## module 間の値の受け渡し

それぞれのパーツを module で作成しているため、作成したリソースの id の参照ができません。そこで output を使って値を渡しています。例えば Azure OpenAI Service 用の module では Private Endpoint や Private DNS Zone のために Virtual Network や Subnet の id が必要になるのですが、そこは `vnet/outputs.tf` で出力した値を拾ってきています。

```hcl
module "aoai" {
  source = "../../aoai"
  resource_group_name = azurerm_resource_group.aoai.name
  location = var.location
  suffix = var.env
  vnet_id = module.vnet.vnet_id
  subnet_id = module.vnet.pe_subnet_id
  random_id = random_id.aoai.hex
}
```

## Azure Open AI Service のデプロイ

`azurerm` プロバイダーでは、`azurerm_coginitive_account` というリソースを参照し、`kind = OpenAI` とすることで Azure Open AI Service をデプロイします。Azure Open AI Service を Azure Portal で利用しているユーザからすると、ここは若干わかりにくいかもしれません。そして、パブリックアクセスを拒否したいので、`public_network_access_enabled = false` も忘れずに入れておきます。

```hcl
 resource "azurerm_cognitive_account" "aoai" {
  name                = "aoai-${var.suffix}"
  location            = var.location
  resource_group_name = var.resource_group_name
  kind                = "OpenAI"
  sku_name            = "S0"
  custom_subdomain_name = "aoai-${var.random_id}-${var.suffix}"
  public_network_access_enabled = false
}
```

また、Azure OpenAI Service のリソースだけでなく、その内部で利用する chat モデルも `azurerm_cognitive_deployment` リソースでデプロイすることが可能[^4]です。

[^4]:https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cognitive_deployment

```hcl
resource "azurerm_cognitive_deployment" "chat" {
  name                 = "aoai-${var.suffix}-chat-model"
  cognitive_account_id = azurerm_cognitive_account.aoai.id
  model {
    format  = "OpenAI"
    name    = "gpt-35-turbo"
    version = "0613"
  }
  scale {
    type = "Standard"
  }
}
```

## subnet における Private Endpoint Policy の有効化

`azurerm_subnet` リソースでは、subnet に対して Private Endpoint Policy は既定で有効[^3]になるようです。よって、jumpbox 用の subnet では Private Endpoint Policy を無効化しています。ここも Azure Portal との違いですね。

```hcl
resource "azurerm_subnet" "jumpbox" {
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.workload.name
  name                 = "subnet-jumpbox"
  address_prefixes     = [var.jumpbox_subnet_address_space]
  private_endpoint_network_policies_enabled = false
}
```

[^3]:https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet#private_endpoint_network_policies_enabled

# 動作検証

## 環境のデプロイ

`/envs/dev/` に移動して以下の流れで実行します。暫らくするとリソースが出来上がります。

```bash
$ terraform init
$ terraform plan
$ terraform apply -var-file="dev.tfvars"
```

:::message
## Tips

plan を実行するときに以下のようなオプションを付けると plan の結果がファイルに出力可能です。

```bash
$ terraform plan -out="tfplan"
```

出力した plan に従って apply を実行したい場合には以下のように、plan のファイルを指定します。

```bash
$ terraform apply "tfplan"
```

::: 

## ネットワーク閉域構成の確認

手元の PC から Azure OpenAI Studio にアクセスしてチャットをしてみると、ネットワークの外部からの接続に該当するため、想定通りエラーにより応答が得られません。
![](/images/20231221-terraform-aoai/01.png)

一方で Terraform でデプロイした jumpbox からの場合は、同じネットワーク内であるためチャットができています。
![](/images/20231221-terraform-aoai/02.png)


# おわりに

今回は、Microsoft Azure Advent Calender 2023 にかこつけて Terraform で Azure OpenAI Service の閉域お試し環境を(折角なのでディレクトリ構成も少しこだわりながら)作ってみました。 Terraform では Azure OpenAI Service のリソース定義まわりが若干癖があるのが注意点といったところでしょうか。またその他の細かいところについてはソースコードをご参考にいただければと思います。では、よいお年を！