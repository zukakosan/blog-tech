---
title: "Azure REST API で Azure リソースを変更しよう"
emoji: "🚽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","api"]
publication_name: "microsoft"
published: false
---

# Azure の管理操作 と Azure REST API
Azure の管理操作のリクエストは、Azure portal、 Azure Powershell/CLI、 REST API、SDK 経由問わず、Azure Reource Manager を経由してサービスに送られます[^1]。

Azure REST API は新サービス、新 API がすぐに登場するため、Azure CLI でカバーしていないサービスや設定を行えます（Request Body 次第）。

また、Azure REST API は Azure CLI の `az rest` コマンドから叩けるため、認証まわりや HTTP リクエストに不慣れであっても気軽に試せます。特に、Azure Cloud Shell では、基本的にログイン済みのため、かなり気軽です。

## az rest コマンド
az rest コマンドの構文は基本的に以下のようになります。

```
$ az rest --method {HTTPメソッド} --url {API-URL} --body '{JSON-PAYLOAD}'
```

以下ドキュメントより抜粋
> az rest コマンドは、ログインした資格情報を使用して自動的に認証します。
Authorization ヘッダーが設定されていない場合は、ヘッダー Authorization: Bearer <token>が追加されます。ここで、<token> は を介して Microsoft Entra IDから取得されます。
--url パラメーターが --url コマンドの出力からエンドポイントで始まる場合、トークンのターゲット リソースは az cloud show --query endpoints パラメーターから派生します。 --url パラメーターが必要です。
カスタム リソースの --resource パラメーターを使用します。
Content-Type ヘッダーが設定されておらず、--body が有効な JSON 文字列である場合、Content-Type ヘッダーは既定で "application/json" になります。
OData の形式で要求に --uri-parameters を使用する場合は、$、Bash として $ をエスケープし、\$で PowerShell を $としてエスケープするなど、さまざまな環境で `$ をエスケープしてください。

# リソースの情報取得

# リソースの変更

# おわりに

[^1]:https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/management/overview#consistent-management-layer