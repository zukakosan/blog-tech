---
title: "Azure 上の Linux VM におけるプロキシ構成を疎通確認してみる"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure","microsoft","ubuntu","squid"]
published: true
publication_name: "microsoft"
---
# はじめに
Linux でプロキシ設定を試したことがなかったため、超基礎的な内容ですが備忘録的に残しておきます。

# 環境準備

## アーキテクチャイメージ
以下のようなイメージで構成します。
![](/images/20240115-linux-proxy-usage/lin-proxy.png)

## プロキシサーバーの構成

Linux (Ubuntu) VM を プロキシサーバー用とクライアント用に2台立てます。プロキシサーバー用 VM にはこちらの記事[^1] の流れで Squid をインストールして構成しておきます。

[^1]: https://zenn.dev/zukako/articles/b0a5f794d16341

## クライアント側 プロキシ設定

クライアント用の Ubuntu マシンに SSH で接続します。この状態で curl コマンドを利用して自分の IP を確認します。

```bash
$ curl inet-ip.info
13.90.24.211
```

![](/images/20240115-linux-proxy-usage/01.png)


`~/.bashrc` に以下のような設定を記述します。その後、`source ~/.bashrc` で更新を反映します。

```bash
proxy="10.0.0.5:3128"
export http_proxy="http://$proxy"
export https_proxy=$http_proxy
export no_proxy="127.0.0.1, localhost, url"
```

curl コマンドで自分の IP を確認してみると、プロキシサーバーのパブリック IP が返ってきます。

```bash
$ curl inet-ip.info
52.146.43.227
```

![](/images/20240115-linux-proxy-usage/02.png)


# まとめ

- 本記事では疎通確認レベルですが、厳密にやるならもっと規則を作りこむ必要がありますね。