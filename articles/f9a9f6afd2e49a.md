---
title: "FSLogix Profile Containerのグループポリシー設定項目"
emoji: "👮"
type: "tech"
topics:
  - "azure"
  - "avd"
  - "gpo"
  - "fslogix"
published: true
published_at: "2023-03-24 13:35"
---

# モチベ
- AVD文脈で必ず出てくるFSLogixのGPOをきちんと押さえたい

![](https://storage.googleapis.com/zenn-user-upload/e98c6135d32b-20230324.png)

## Delet Local Profile When VHD Should Apply
- FSLogix側のユーザプロファイルがある場合、そのユーザのローカルプロファイルを削除するよ

## Enabled
- FSLogixのProfile Container機能自体が有効化されているかどうか

## Install Appx Package
- Appx Package機能が有効化されているかどうか
- デフォルトはEnabled

## is Dynamic(VHD)
- VHDのサイズを動的に増やしていくかどうか
- デフォルトはEnabled

## Outlook Cached Mode
- Office 365 Containerのアタッチに成功した際に、Outlookがキャッシュモードで動くようにするかどうか

## Profile Type
- VHD/Xファイルが直接アクセスされるのか、別のディスクを使うのか
- Read/Write権限を与えるのか、Read権限だけなのか

## Roam Identity
- Web Account Manager Systemによって生成されたレガシーのクレデンシャルやトークンを許可するか
- デフォルトはDisabled

## Roam Search Database
- FSLogix Search Roamingを有効化し、検索データをOffice 365 Containerに格納するための設定

## Size in MBs
- 自動で作成されるVHD(X)ファイルのサイズ指定
- デフォルトでは30GB

## VHD Locations
- VHD(X)ファイルを保存しておくための場所
- Azure NetApp FilesとかAzure Filesとかへのパス