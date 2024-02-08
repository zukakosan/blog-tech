---
title: "ExpressRoute Microsoft Peering で流れてくるアドレスプレフィックスとサービスの対応を取得する"
emoji: "🚄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","expressroute","bgp"]
published: true
publication_name: "microsoft"

---
# はじめに
Azure で ExpressRoute の Microsoft Peering を利用する場合、Microsoft 365 や Azure PaaS などのパブリック IP アドレスプレフィックスがオンプレミス側のルーターに広報されてきます。あくまで IP アドレスプレフィックスとしてしか見えないので、そのアドレスが何のサービスに利用されているアドレスなのかまでは分かりません。不要なアドレスプレフィックスを取得していると、望まない通信が Microsoft Peering を通ることになるため帯域のひっ迫やコンプライアンス要件の問題等に該当してしまうかもしれません。

# アドレスプレフィックスとサービスの対応を取得する
ぱっと思いつくのがこちらのドキュメントではないでしょうか[^1]。このサイトでの公開情報は、利用する可能性のあるアドレス帯をすべて記載をしているのですが、実際に利用しているアドレス帯は ExpressRoute で広報されてくるため、CIDR 形式的に完全に一致するとは限りません。また、そもそもこちらのドキュメントでは Azure サービスについては触れられていません。
[^1]: https://learn.microsoft.com/ja-jp/microsoft-365/enterprise/urls-and-ip-address-ranges?view=o365-worldwide

:::message

ちなみに、ルートフィルターで `Azure Active Directory` を広報すると、以下のような経路を受け取ります。

```json
{
	"communityName": "Azure Active Directory",
	"communityPrefixes": [
		"40.126.0.0/19",
		"40.126.32.0/19",
		"20.190.128.0/19",
		"20.190.160.0/19"
	],
	"communityValue": "12076:5060",
	"isAuthorizedToUse": true,
	"serviceGroup": "Azure",
	"serviceSupportedRegion": "Global"
}
```

一方、`Other Office 365 Services` では、`20.190.128.0/18`と `40.126.0.0/18` にまとめられて広報されます。
```json
{
	"communityName": "Other Office 365 Services",
	"communityPrefixes": [
		"13.107.6.171/32",
		"13.107.6.192/32",
		"13.107.9.192/32",
		"13.107.18.15/32",
		"13.107.140.6/32",
		"20.20.32.0/19",
		"20.190.128.0/18",
		"20.231.128.0/19",
		"40.126.0.0/18",
		"52.108.0.0/14",
		"52.244.37.168/32"
	],
	"communityValue": "12076:5100",
	"isAuthorizedToUse": false,
	"serviceGroup": "O365",
	"serviceSupportedRegion": "Global"
}
```
:::

ではどのように対応を取得するかというと、PowerShell コマンドレットもしくは Azure CLI のコマンドを利用します。 実はこちらのルートフィルターに関するドキュメント[^2] にしれっと記載があります。
[^2]: https://learn.microsoft.com/ja-jp/azure/expressroute/how-to-routefilter-cli#prefixes

Azure PowerShell では `Get-AzBgpServiceCommunity`[^3] コマンドレットを利用し、Azure CLI では `az network route-filter rule list-service-communities`[^4] を利用します。後者についてはまだプレビュー中のようです。
[^3]: https://learn.microsoft.com/ja-jp/powershell/module/az.network/get-azbgpservicecommunity?view=azps-10.4.1
[^4]: https://learn.microsoft.com/ja-jp/cli/azure/network/route-filter/rule?view=azure-cli-latest#az-network-route-filter-rule-list-service-communities

Azure PowerShell の方は出力が少し扱いづらかったため、Azure CLI の方を試しました。以下を実行してファイルに出力できます。

```bash
$ route=$(az network route-filter rule list-service-communities)
$ echo $route > route.json
```

作成されたファイルの中に、ずらーっとルートフィルターで設定する `communityName`、`communityPrefixes`、`communityValue` といった情報が記載されています。

```json
[
	{
		"bgpCommunities": [
			{
				"communityName": "Exchange",
				"communityPrefixes": [
					"13.107.6.152/31",
					"13.107.18.10/31",
					"13.107.128.0/22",
					"23.103.160.0/20",
					"40.92.0.0/15",
					"40.96.0.0/13",
					"40.104.0.0/15",
					"40.107.0.0/16",
					"52.96.0.0/14",
					"52.100.0.0/14",
					"52.238.78.88/32",
					"104.47.0.0/17",
					"131.253.33.215/32",
					"132.245.0.0/16",
					"150.171.32.0/22",
					"204.79.197.215/32",
					"13.107.128.0/24",
					"13.107.129.0/24",
					"150.171.32.0/24",
					"150.171.34.0/24",
					"150.171.35.0/24",
					"52.96.38.0/24"
				],
				"communityValue": "12076:5010",
				"isAuthorizedToUse": false,
				"serviceGroup": "O365",
				"serviceSupportedRegion": "Global"
			},
			{
				"communityName": "Exchange IPv6",
				"communityPrefixes": [
					"2603:1006::/40",
					"2603:1016::/36",
					"2603:1026::/36",
					"2603:1036::/36",
					"2603:1046::/36",
					"2603:1056::/36",
					"2620:1ec:4::152/128",
					"2620:1ec:4::153/128",
					"2620:1ec:c::10/128",
					"2620:1ec:c::11/128",
					"2620:1ec:d::10/128",
					"2620:1ec:d::11/128",
					"2620:1ec:8f0::/46",
					"2620:1ec:900::/46",
					"2620:1ec:a92::152/128",
					"2620:1ec:a92::153/128",
					"2a01:111:f400::/48",
					"2a01:111:f403::/48"
				],
				"communityValue": "12076:5010",
				"isAuthorizedToUse": false,
				"serviceGroup": "O365",
				"serviceSupportedRegion": "Global"
			}
		],
		"id": "/subscriptions//resourceGroups//providers/Microsoft.Network/bgpServiceCommunities/Exchange",
		"name": "Exchange",
		"resourceGroup": "",
		"serviceName": "Exchange",
		"type": "Microsoft.Network/bgpServiceCommunities"
	},
	{
		"bgpCommunities": [
			{
				"communityName": "Other Office 365 Services",
				"communityPrefixes": [
					"13.107.6.171/32",
					"13.107.6.192/32",
					"13.107.9.192/32",
					"13.107.18.15/32",
					"13.107.140.6/32",
					"20.20.32.0/19",
					"20.190.128.0/18",
					"20.231.128.0/19",
					"40.126.0.0/18",
					"52.108.0.0/14",
					"52.244.37.168/32"
				],
				"communityValue": "12076:5100",
				"isAuthorizedToUse": false,
				"serviceGroup": "O365",
				"serviceSupportedRegion": "Global"
			},
			{
				"communityName": "Other Office 365 Services IPv6",
				"communityPrefixes": [
					"2603:1006:1400::/40",
					"2603:1006:2000::/48",
					"2603:1007:200::/48",
					"2603:1016:1400::/48",
					"2603:1016:2400::/40",
					"2603:1017::/48",
					"2603:1026:2400::/40",
					"2603:1026:3000::/48",
					"2603:1027:1::/48",
					"2603:1036:2400::/40",
					"2603:1036:3000::/48",
					"2603:1037:1::/48",
					"2603:1046:1400::/40",
					"2603:1046:2000::/48",
					"2603:1047:1::/48",
					"2603:1056:1400::/40",
					"2603:1056:2000::/48",
					"2603:1057:2::/48",
					"2603:1063:2000::/38",
					"2620:1ec:4::192/128",
					"2620:1ec:c::15/128",
					"2620:1ec:8fc::6/128",
					"2620:1ec:a92::171/128",
					"2620:1ec:a92::192/128",
					"2a01:111:f100:2000::a83e:3019/128",
					"2a01:111:f100:2002::8975:2d79/128",
					"2a01:111:f100:2002::8975:2da8/128",
					"2a01:111:f100:7000::6fdd:6cd5/128",
					"2a01:111:f100:a004::bfeb:88cf/128"
				],
				"communityValue": "12076:5100",
				"isAuthorizedToUse": false,
				"serviceGroup": "O365",
				"serviceSupportedRegion": "Global"
			}
		],
		"id": "/subscriptions//resourceGroups//providers/Microsoft.Network/bgpServiceCommunities/OtherOffice365Services",
		"name": "OtherOffice365Services",
		"resourceGroup": "",
		"serviceName": "OtherOffice365Services",
		"type": "Microsoft.Network/bgpServiceCommunities"
	},
	...(中略)

				{
				"communityName": "Azure SIP Trunking IPv6",
				"communityPrefixes": [],
				"communityValue": "12076:5250",
				"isAuthorizedToUse": true,
				"serviceGroup": "AzureSIPTrunking",
				"serviceSupportedRegion": "Global"
			}
		],
		"id": "/subscriptions//resourceGroups//providers/Microsoft.Network/bgpServiceCommunities/AzureSIPTrunking",
		"name": "AzureSIPTrunking",
		"resourceGroup": "",
		"serviceName": "AzureSIPTrunking",
		"type": "Microsoft.Network/bgpServiceCommunities"
	}
]
```

# まとめ
オンプレミス側に広報されてくるアドレスプレフィックスの招待を確かめるためには、`Get-AzBgpServiceCommunity` コマンドレットや `az network route-filter rule list-service-communities` コマンドを利用しましょう。