---
title: "Azure の Terraform コードの妥当性を apply 前に検証して構成ミスを検知する"
emoji: "🛕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","terraform","test","microsoft"]
published: true
publication_name: "microsoft"
---

# はじめに
Terraform のコードを手元で作成して plan を確認し、問題なければ apply するというのが通常の流れです。Terraform v1.6.0 からは `terraform test` というコマンド[^1] が追加され、重大な変更が意図せず行われないことを確認できるようになります。本記事では `terraform test` および v1.5.0 から利用可能な check ブロック[^2] をAzure 環境に対して適用してみます。

[^1]:https://developer.hashicorp.com/terraform/language/tests
[^2]:https://developer.hashicorp.com/terraform/language/checks

# 環境
コードとしては、以前 `terraform import` を試す際に作成したプロジェクト[^3] を利用します。

[^3]:https://github.com/zukakosan/terraform-learn/tree/main/20240322-import


# terraform test
プロジェクトの範囲内(planの範囲)に対して `.tftest.hcl` を書きます。
簡単にリソースのリージョンを確かめるテストファイルを書いてみます。
ここではあえて失敗するように、テンプレートファイルと異なるリージョンを指定しています。

```hcl
run "validate_subnet_private" {
	command = plan
	assert {
		condition = azurerm_virtual_network.test.location == "eastus"
		error_message = "${azurerm_virtual_network.test} のリージョンが適切ではありません"
	}
}
```

ファイル作成後、`init` してから `terraform test` します。想定通り、失敗で終了します。

```
$ terraform test
vnetprivate-test.tftest.hcl... in progress
  run "validate_subnet_private"... fail
╷
│ Error: Test assertion failed
│
│   on vnetprivate-test.tftest.hcl line 4, in run "validate_subnet_private":
│    4:                 condition = azurerm_virtual_network.test.location == "eastus"
│     ├────────────────
│     │ azurerm_virtual_network.test.location is "japaneast"
│
│ vnet-import のリージョンが適切ではありません
╵
vnetprivate-test.tftest.hcl... tearing down
vnetprivate-test.tftest.hcl... fail
```

ちなみに `.tftest.hcl` のリージョン部分を `japaneast` にすると成功します。変更したら必ず `terraform init` しましょう。

```hcl
run "validate_subnet_private" {
	command = plan
	assert {
		condition = azurerm_virtual_network.test.location == "japaneast"
		error_message = "${azurerm_virtual_network.test} のリージョンが適切ではありません"
	}
}
```

`terraform test` します。成功しました。

```
$ terraform test     
vnetprivate-test.tftest.hcl... in progress
  run "validate_subnet_private"... pass
vnetprivate-test.tftest.hcl... tearing down
vnetprivate-test.tftest.hcl... pass

Success! 1 passed, 0 failed.
```

`.tftest.hcl` で `command = apply` にすると、既存インフラが存在する場合、エラーになるとその既存インフラを削除してしまう点に注意が必要です。

# check ブロック
terraform には check ブロックも利用できます。記法としてはテストファイルと似たような形式になります。

`check.tf` を作成して以下のように記載します。

```hcl
check "vnet_location_check" {
  	assert {
		condition = azurerm_virtual_network.test.location == "eastus"
		error_message = "${azurerm_virtual_network.test.name} のリージョンが適切ではありません"
	}
}
```

このまま、`plan` してみましょう。構成は変更していないのですが、check ブロックを追加したことにより `failed` となっています。test 同様に構成ミスに気付くことができます。 

```
$ terraform plan
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
╷
│ Warning: Check block assertion failed
│
│   on check.tf line 3, in check "vnet_location_check":
│    3:                 condition = azurerm_virtual_network.test.location == "eastus"
│     ├────────────────
│     │ azurerm_virtual_network.test.location is "japaneast"
│
│ vnet-import のリージョンが適切ではありません
```

# おわりに

Azure ではこうした構成ガバナンスのための機能として Azure Policy というものがあるのですが、適用対象はもちろん実在するリソースになります。`test` や check ブロックは `plan` の段階で確認できるため、シフトレフト的な観点からもより安全であるといえます。check ブロックでは e2e テストもできるようなので、これはまた試したいと思って ToDo に積み残しています。

全体構成の確認には check ブロックが推奨されているようです。[^4]
> We recommend using check blocks to validate the status of infrastructure as a whole. We only recommend using postconditions when you want a guarantee on a single resource based on that resource's configuration.

[^4]:https://developer.hashicorp.com/terraform/language/checks#syntax
