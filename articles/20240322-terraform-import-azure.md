---
title: "Azure 環境への IaC 導入前に作成した既存リソースを Terraform に取り込む"
emoji: "🛕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","terraform","microsoft","IaC"]
published: true
publication_name: "microsoft"
---

# はじめに

Terraform で Azure 上の既存リソースを参照したい場合があると思います。IaC を導入する前に Azure portal からデプロイしていたリソースについては、Terraform のステートファイルに記述されていないため、認識できません。Bicep であれば、existing で既存リソースを参照できますが、Terraform の場合にはステートファイルの管理下に置く必要があるため import コマンドを利用します。この記事では、Azure における `terraform import` をリソースグループと Virtual Network (VNet) について試します。

# リソースグループの import
まずは、非常にシンプルにだけを Azure portal 側で作成しておき、それを取り込むところをやってみます。
作業ディレクトリに `main.tf` を作成し、リソースの定義の枠組みだけ書いておきます。まずは、リソースグループの定義だけを書いてみます。

```hcl
resource "azurerm_resource_group" "test" {
}
```

`providers.tf` 等も書いておき、`terraform init` します。
その状態で、以下を実行します。最後の文字列はリソース ID です。

```
$ terraform import azurerm_resource_group.test /subscriptions/xxxx/resourceGroups/20240322-terraform-import
```

:::message

手元の git bash でこのコマンドを実行すると、エラーになってしまったため PowerShell にて実行しました。既知のissue[^1] のようです。
[^1]:https://github.com/hashicorp/terraform-provider-azurerm/issues/7604

:::

実行すると以下のように `Import successful!` と表示されます。

```
azurerm_resource_group.test: Import prepared!
  Prepared azurerm_resource_group for import
azurerm_resource_group.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.

```

backend で設定した箇所にステートファイルが作成されます。今回はローカルで実行しているため、テキストエディタ上で確認します。すると、指定したリソースグループの情報が記載されていることが確認できます。

```json

{
  "version": 4,
  "terraform_version": "1.6.2",
  "serial": 1,
  "lineage": "eca378be-576d-87aa-3c52-7f050262096f",
  "outputs": {},
  "resources": [
	{
	  "mode": "managed",
	  "type": "azurerm_resource_group",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import",
			"location": "japaneast",
			"managed_by": "",
			"name": "20240322-terraform-import",
			"tags": {},
			"timeouts": null
		  },
		  "sensitive_attributes": [],
		  "private": "xxxxx"
		}
	  ]
	}
  ],
  "check_results": null
}
```

この状態ではまだ `main.tf` 側の `azurerm_resource_group.test` については中身が空っぽなので、ステートファイルを見て合わせていきます。

ステートファイルに準拠するように `main.tf` を以下のように変更します。

```hcl
resource "azurerm_resource_group" "test" {
  name     = "20240322-terraform-import"
  location = "japaneast"
}
```

保存して、`plan` をしてみましょう。問題なければ、変更が生じないはずです。

```
$ terraform plan    
azurerm_resource_group.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are
needed.

```
`No changes` となっているため、`import` した環境とマッチしていることが分かります。

# VNet リソースを Azure portal から追加
続いて、実際にリソースがあった場合にどうするかという点を確認します。Azure portal で VNet のリソースを作成します( IaC の規範には反します)。


# VNet の import
先ほどと同様に、`main.tf` に箱を作ります。

```hcl
resource "azurerm_resource_group" "test" {
  name     = "20240322-terraform-import"
  location = "japaneast"
}
resource "azurerm_virtual_network" "test" {
}
```

そこに対して `import` します。

```
$ terraform import azurerm_virtual_network.test /subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import
```

成功すると、ステートファイルにVnet の情報が追記されます。
```json
{
	  "mode": "managed",
	  "type": "azurerm_virtual_network",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_space": [
			  "10.0.0.0/16"
			],
			"bgp_community": "",
			"ddos_protection_plan": [],
			"dns_servers": [],
			"edge_zone": "",
			"encryption": [],
			"flow_timeout_in_minutes": 0,
			"guid": "e795c79f-f007-4db3-b7b3-0fa4200a5c42",
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import",
			"location": "japaneast",
			"name": "vnet-import",
			"resource_group_name": "20240322-terraform-import",
			"subnet": [
			  {
				"address_prefix": "10.0.0.0/24",
				"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default",
				"name": "default",
				"security_group": ""
			  }
			],
			"tags": {},
			"timeouts": null
		  },
		  "sensitive_attributes": [],
		  "private": "xxxx"
		}
	  ]
	}
```

ステートファイル合わせて `main.tf` を更新します。

```hcl
resource "azurerm_resource_group" "test" {
  name     = "20240322-terraform-import"
  location = "japaneast"
}
resource "azurerm_virtual_network" "test" {
	name = "vnet-import"
	location = azurerm_resource_group.test.location
	resource_group_name = azurerm_resource_group.test.name
	address_space = ["10.0.0.0/16"]
}
```

一旦この状態で `plan` してみます。サブネットの定義は書いていないのですが、`No changes` となりました。ステートファイルとは合致している認識のようです。

```
$ terraform plan    
azurerm_resource_group.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import]
azurerm_virtual_network.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are        
Needed.
```
# サブネットを別で定義
Terraform ではサブネットをリソースとして宣言することもあると思います。ステートファイルの中身に合うようにサブネットの宣言を追加してみます。

```hcl
resource "azurerm_resource_group" "test" {
  name     = "20240322-terraform-import"
  location = "japaneast"
}
resource "azurerm_virtual_network" "test" {
	name = "vnet-import"
	location = azurerm_resource_group.test.location
	resource_group_name = azurerm_resource_group.test.name
	address_space = ["10.0.0.0/16"]
}
resource "azurerm_subnet" "test" {
	name = "default"
	resource_group_name = azurerm_resource_group.test.name
	virtual_network_name = azurerm_virtual_network.test.name
	address_prefixes = ["10.0.0.0/24"]
}
```

`plan` して差分を確認します。サブネットが `add` されるように認識されています。

```
Terraform will perform the following actions:

  # azurerm_subnet.test will be created
  + resource "azurerm_subnet" "test" {
	  + address_prefixes                               = [
		  + "10.0.0.0/24",
		]
	  + enforce_private_link_endpoint_network_policies = (known after apply)
	  + enforce_private_link_service_network_policies  = (known after apply)
	  + id                                             = (known after apply)
	  + name                                           = "default"
	  + private_endpoint_network_policies_enabled      = (known after apply)
	  + private_link_service_network_policies_enabled  = (known after apply)
	  + resource_group_name                            = "20240322-terraform-import"
	  + virtual_network_name                           = "vnet-import"
	}

Plan: 1 to add, 0 to change, 0 to destroy.
```

リソースとして宣言する際には、リソースとしての `import` が必要なのかもしれません。そこで、対象のサブネットを `import` してみます。

```
$ terraform import azurerm_subnet.test /subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default
```

`Import successful!` と表示されたら、ステートファイルを確認します。`azurerm_subnet` が追加されています。

```json
{
	  "mode": "managed",
	  "type": "azurerm_subnet",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_prefixes": [
			  "10.0.0.0/24"
			],
			"delegation": [],
			"enforce_private_link_endpoint_network_policies": true,
			"enforce_private_link_service_network_policies": false,
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default",
			"name": "default",
			"private_endpoint_network_policies_enabled": false,
			"private_link_service_network_policies_enabled": true,
			"resource_group_name": "20240322-terraform-import",
			"service_endpoint_policy_ids": [],
			"service_endpoints": [],
			"timeouts": null,
			"virtual_network_name": "vnet-import"
		  },
		  "sensitive_attributes": [],
		  "private": "xxxx"
		}
	  ]
	},
	{
	  "mode": "managed",
	  "type": "azurerm_virtual_network",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_space": [
			  "10.0.0.0/16"
			],
			"bgp_community": "",
			"ddos_protection_plan": [],
			"dns_servers": [],
			"edge_zone": "",
			"encryption": [],
			"flow_timeout_in_minutes": 0,
			"guid": "e795c79f-f007-4db3-b7b3-0fa4200a5c42",
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import",
			"location": "japaneast",
			"name": "vnet-import",
			"resource_group_name": "20240322-terraform-import",
			"subnet": [
			  {
				"address_prefix": "10.0.0.0/24",
				"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default",
				"name": "default",
				"security_group": ""
			  }
			],
			"tags": {},
			"timeouts": null
		  },
		  "sensitive_attributes": [],
		  "private": "xxxx"
		}
	  ]
	}
```

アドレスとしては `azurerm_virtual_network` の定義と重複してしまっているのですが、とりあえず `plan` でステートファイルと `main.tf` の差分を確認します。出力を見る限りこれで問題なさそうです。

```
$ terraform plan
azurerm_resource_group.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import]
azurerm_virtual_network.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import]
azurerm_subnet.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes   
are needed.
```

## サブネットをスタンドアロンで定義する際にどうなるか

サブネットのアドレス重複が気になるので、新規で `apply` した際にどうなるか確認します。

`main.tf` を新規( Azure 上にリソースがない状態)で `apply` してみると、ステートファイル内でのネットワーク部分については次のようになります。

```json
{
	  "mode": "managed",
	  "type": "azurerm_subnet",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_prefixes": [
			  "10.0.0.0/24"
			],
			"delegation": [],
			"enforce_private_link_endpoint_network_policies": false,
			"enforce_private_link_service_network_policies": false,
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import-exp/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default",
			"name": "default",
			"private_endpoint_network_policies_enabled": true,
			"private_link_service_network_policies_enabled": true,
			"resource_group_name": "20240322-terraform-import-exp",
			"service_endpoint_policy_ids": null,
			"service_endpoints": null,
			"timeouts": null,
			"virtual_network_name": "vnet-import"
		  },
		  "sensitive_attributes": [],
		  "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjoxODAwMDAwMDAwMDAwLCJkZWxldGUiOjE4MDAwMDAwMDAwMDAsInJlYWQiOjMwMDAwMDAwMDAwMCwidXBkYXRlIjoxODAwMDAwMDAwMDAwfX0=",
		  "dependencies": [
			"azurerm_resource_group.test",
			"azurerm_virtual_network.test"
		  ]
		}
	  ]
	},
	{
	  "mode": "managed",
	  "type": "azurerm_virtual_network",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_space": [
			  "10.0.0.0/16"
			],
			"bgp_community": "",
			"ddos_protection_plan": [],
			"dns_servers": [],
			"edge_zone": "",
			"encryption": [],
			"flow_timeout_in_minutes": 0,
			"guid": "e8b6d3c8-e5ff-4cab-b707-37b84550b61c",
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import-exp/providers/Microsoft.Network/virtualNetworks/vnet-import",
			"location": "japaneast",
			"name": "vnet-import",
			"resource_group_name": "20240322-terraform-import-exp",
			"subnet": [],
			"tags": null,
			"timeouts": null
		  },
		  "sensitive_attributes": [],
		  "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjoxODAwMDAwMDAwMDAwLCJkZWxldGUiOjE4MDAwMDAwMDAwMDAsInJlYWQiOjMwMDAwMDAwMDAwMCwidXBkYXRlIjoxODAwMDAwMDAwMDAwfX0=",
		  "dependencies": [
			"azurerm_resource_group.test"
		  ]
		}
	  ]
	}
```

`azurerm_virtual_network` では `"subnet":[]` となっています。重複は生じていません。 `import` した場合のステートファイルにおいても同じようにきれいにしてみた場合にどう影響するか確認してみます。

ステートファイル内のVnet のパートにおいて `subnet:[]` の中身を削除しました。

```json
   {
	  "mode": "managed",
	  "type": "azurerm_virtual_network",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_space": [
			  "10.0.0.0/16"
			],
			"bgp_community": "",
			"ddos_protection_plan": [],
			"dns_servers": [],
			"edge_zone": "",
			"encryption": [],
			"flow_timeout_in_minutes": 0,
			"guid": "e795c79f-f007-4db3-b7b3-0fa4200a5c42",
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import",
			"location": "japaneast",
			"name": "vnet-import",
			"resource_group_name": "20240322-terraform-import",
			"subnet": [],
			"tags": {},
			"timeouts": null
		  },
		  "sensitive_attributes": [],
		  "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjoxODAwMDAwMDAwMDAwLCJkZWxldGUiOjE4MDAwMDAwMDAwMDAsInJlYWQiOjMwMDAwMDAwMDAwMCwidXBkYXRlIjoxODAwMDAwMDAwMDAwfSwic2NoZW1hX3ZlcnNpb24iOiIwIn0=",
		  "dependencies": [
			"azurerm_resource_group.test"
		  ]
		}
	  ]
	}
```

この状態で plan をしてみます。一応 `No changes` となりました。とはいえステートファイルを直接触りたくはないので、動くようであれば触らない方がいいかもしれません。

```
$ terraform plan 
azurerm_resource_group.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import]
azurerm_virtual_network.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import]
azurerm_subnet.test: Refreshing state... [id=/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes   
are needed.
```

別のサブネットを追加してみましょう。

```hcl
resource "azurerm_resource_group" "test" {
  name     = "20240322-terraform-import"
  location = "japaneast"
}
resource "azurerm_virtual_network" "test" {
	name = "vnet-import"
	location = azurerm_resource_group.test.location
	resource_group_name = azurerm_resource_group.test.name
	address_space = ["10.0.0.0/16"]
}
resource "azurerm_subnet" "test" {
	name = "default"
	resource_group_name = azurerm_resource_group.test.name
	virtual_network_name = azurerm_virtual_network.test.name
	address_prefixes = ["10.0.0.0/24"]
}
resource "azurerm_subnet" "subnet-001" {
	name = "subnet-001"
	resource_group_name = azurerm_resource_group.test.name
	virtual_network_name = azurerm_virtual_network.test.name
	address_prefixes = ["10.0.1.0/24"]
}
```

この状態で `plan` してみます。

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with  
the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_subnet.subnet-001 will be created
  + resource "azurerm_subnet" "subnet-001" {
	  + address_prefixes                               = [
		  + "10.0.1.0/24",
		]
	  + enforce_private_link_endpoint_network_policies = (known after apply)
	  + enforce_private_link_service_network_policies  = (known after apply)
	  + id                                             = (known after apply)
	  + name                                           = "subnet-001"
	  + private_endpoint_network_policies_enabled      = (known after apply)
	  + private_link_service_network_policies_enabled  = (known after apply)
	  + resource_group_name                            = "20240322-terraform-import"
	  + virtual_network_name                           = "vnet-import"
	}

Plan: 1 to add, 0 to change, 0 to destroy.
```

`apply` までやってみましょう。ステートファイルは次のようになります。サブネットのリソースが 1 つ追加されましたが、VNet 側の`subnet:[]` に追加されているわけではありません。

```json
{
	  "mode": "managed",
	  "type": "azurerm_subnet",
	  "name": "subnet-001",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_prefixes": [
			  "10.0.1.0/24"
			],
			"delegation": [],
			"enforce_private_link_endpoint_network_policies": false,
			"enforce_private_link_service_network_policies": false,
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/subnet-001",
			"name": "subnet-001",
			"private_endpoint_network_policies_enabled": true,
			"private_link_service_network_policies_enabled": true,
			"resource_group_name": "20240322-terraform-import",
			"service_endpoint_policy_ids": null,
			"service_endpoints": null,
			"timeouts": null,
			"virtual_network_name": "vnet-import"
		  },
		  "sensitive_attributes": [],
		  "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjoxODAwMDAwMDAwMDAwLCJkZWxldGUiOjE4MDAwMDAwMDAwMDAsInJlYWQiOjMwMDAwMDAwMDAwMCwidXBkYXRlIjoxODAwMDAwMDAwMDAwfX0=",
		  "dependencies": [
			"azurerm_resource_group.test",
			"azurerm_virtual_network.test"
		  ]
		}
	  ]
	},
	{
	  "mode": "managed",
	  "type": "azurerm_subnet",
	  "name": "test",
	  "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
	  "instances": [
		{
		  "schema_version": 0,
		  "attributes": {
			"address_prefixes": [
			  "10.0.0.0/24"
			],
			"delegation": [],
			"enforce_private_link_endpoint_network_policies": true,
			"enforce_private_link_service_network_policies": false,
			"id": "/subscriptions/xxxx/resourceGroups/20240322-terraform-import/providers/Microsoft.Network/virtualNetworks/vnet-import/subnets/default",
			"name": "default",
			"private_endpoint_network_policies_enabled": false,
			"private_link_service_network_policies_enabled": true,
			"resource_group_name": "20240322-terraform-import",
			"service_endpoint_policy_ids": [],
			"service_endpoints": [],
			"timeouts": null,
			"virtual_network_name": "vnet-import"
		  },
		  "sensitive_attributes": [],
		  "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjoxODAwMDAwMDAwMDAwLCJkZWxldGUiOjE4MDAwMDAwMDAwMDAsInJlYWQiOjMwMDAwMDAwMDAwMCwidXBkYXRlIjoxODAwMDAwMDAwMDAwfSwic2NoZW1hX3ZlcnNpb24iOiIwIn0=",
		  "dependencies": [
			"azurerm_resource_group.test",
			"azurerm_virtual_network.test"
		  ]
		}
	  ]
	}
```

リソースが `destroy` されるわけではないので、挙動上問題はなさそうですが、どうするべきか難しいですね。

# おわりに
だいぶ本筋からそれてしまった部分もありましたが、既存の Azure リソースを Terraform に取り込む流れを確認しました。全く管理されていないリソースではなく、別のステートファイルで管理されているリソースの場合は terraform_remote_state によって対応できるかと思います。IaC を始める前に存在していたリソースや、運用する中で誤って、もしくは障害等でやむを得ず変更してしまった場合は適宜 import するなどして対処する必要があります。適宜 plan を行って差分を確認しながら慎重に進めないと、思わぬアクシデントにぶつかりそうです。。。

