---
title: Dockerfile.devでapt-keyを使わずにNode.jsとYarnをインストールする
tags:
  - RUNTEQ
  - Docker
  - Node.js
  - YARN
  - 環境構築
private: false
updated_at: '2026-09-24T19:55:29+09:00'
id: 26839cf628e764013b66
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
## はじめに

この記事では、プログラミング初学者の僕が
1つのアプリを本リリースするまでに開発中に躓いたところや苦戦したところを
自分なりに整理してまとめ、理解を深めることを目的としています。

今回は、
**Dockerfile.devでNode.jsとYarnをインストールしようとした時に起きたエラー**
についてです。

## 背景

アプリの開発に必要なソフトウェアを用意するため、
**Dockerfile.dev**を作成しました。

ベースイメージには**ruby:3.4.10**を使っています。

最初はNode.jsとYarnの配布元をAPTに登録し、apt-getでインストールしようとしていました。
APTは、Debianなどでソフトウェアを管理・インストールするための仕組みです。

#### 当初のDockerfile.dev
```dockerfile
# ベースイメージの指定
FROM ruby:3.4.10

# ...（省略）...

# 必要なパッケージのインストール
RUN apt-get update -qq \
# 必要なパッケージをインストール
&& apt-get install -y ca-certificates curl gnupg \
# キーリングディレクトリを作成
&& mkdir -p /etc/apt/keyrings \
&& curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
&& NODE_MAJOR=26 \
&& echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list \
&& wget --quiet -O - /tmp/pubkey.gpg https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
&& echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs yarn vim

# ...（省略）...
 
```

当初のコードには、Yarnの配布元を登録するために、次の処理が含まれていました。

```dockerfile
wget --quiet -O - /tmp/pubkey.gpg https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
```

ところが、Dockerイメージのビルドに失敗し、エラーの末尾に次のように表示されました。

```bash
did not complete successfully: exit code: 127
```

**exit code: 127**は、実行しようとしたコマンドが見つからない時に出る終了コードです。
ただ、この１行だけだと、どのコマンドが見つからなかったのか特定できません。

なので直前のエラー表示も確認してみたところ、
`apt-key add -`が見つからないということが分かりました。

## apt-keyとは

apt-keyは、APTが配布元の署名を確認するために使う鍵を管理するコマンドです。

僕は最初、apt-keyを使ってパッケージをインストールしているのだと思っていました。
しかし、実際にインストールするのはapt-getです。

apt-keyは、その前段階にある鍵の登録に使われていました。

## 今回つまずいた原因

ruby:3.4.10の標準イメージは、Debian 13をベースにしています。
Debian 13ではapt-keyが削除されているので、
`apt-key add -`を含む手順はそのままでは動きません。

また、`apt-key add`による鍵の登録は非推奨となっています。
この方法で追加した鍵は、特定の配布元だけではなく、
APT全体の信頼済み鍵として扱われてしまうため、セキュリティ上のリスクが高まるからという理由みたいです。

その代わり現在は**Signed-By**を使って、
「この配布元の署名を確認するときは、この鍵を使う」と指定する方法が推奨されるようになりました。

つまり今回は、
apt-keyが使えない環境だったことと、そもそも古い鍵の登録方法を使っていたことに気付きました。

（なぜかNode.js側は最初から新しい書き方でインストールしてましたが。。w）


## 解決方法

僕はYarnの配布元をAPTに登録する処理をやめ、
Node.jsをインストールした後にnpmを使ってYarnをインストールする方法に変更しました。

```dockerfile
RUN apt-get update -qq \
&& apt-get install -y ca-certificates curl \
&& curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
&& apt-get install -y build-essential libpq-dev nodejs vim libvips libvips-tools imagemagick \
&& npm install --global yarn
```

このコードでは、次の順番で処理しています。

:::note info
1. `curl`など、準備に必要なパッケージをインストール。
2. `curl`で**NodeSourceの設定スクリプト**を取得し、bashで実行。
  このスクリプトが、Node.jsの配布元（NodeSourceリポジトリ）と署名確認用の鍵を設定する。
3. `apt-get`で**Node.js**などをインストール。
4. npmを使って**Yarn**をインストール。
:::

**NodeSourceの設定スクリプト**は`apt-key`を使わず、**`Signed-By`オプション**で鍵を指定しています。
**Yarn**は**npm**経由でインストールするため、Yarnの配布元をAPTに追加する必要もなくなりました。

なお、変更前のコードでは**Node.js 26系**を指定していましたが、
変更後の`setup_lts.x`は**LTS版**を設定しています。

変更前後で**Node.jsのバージョンが同じになるとは限らない**ため、
アプリで使う**Node.jsのバージョン**を確認する必要があります。


ちなみに、今回は**NodeSourceの設定スクリプト**の中で勝手に実行していましたが、
`apt-key`を使わなくなった今、推奨されている方法はざっくりと次のような手順を踏むのかなと思います。

:::note info
#### 「公開鍵をファイルに保存して、リポジトリ設定の`Signed-By`でそのファイルを指定する」

1. `/etc/apt/keyrings`ディレクトリを作成
2. リポジトリの公開鍵をダウンんロードして、`gpg`でバイナリ形式に変換（`--dearmor`オプション）
3. `keyrings`ディレクトリに格納
4. gpgファイルの場所を`signed-by`オプションで指定して、リポジトリのソースリストに追加
5. `apt-get update`でパッケージを更新してから、`apt-get install`でインストール

:::


## 忘れないように学んだコマンド・パッケージ
#### `bash -`
シェルスクリプトを実行するコマンド。

#### `apt-get`
パッケージを管理・インストールするコマンド。

#### `ca-certificates`
証明書の検証に必要なパッケージ。

#### `curl`
ダウンロードに必要なパッケージ。

#### `gpg --dearmor`
ASCII形式の鍵をバイナリ形式に変換する。


## まとめ

僕の環境では、
apt-keyを使う処理をやめ、Yarnをnpm経由でインストールすることで
Dockerイメージをビルドできました。

環境構築では、エラーが出ても何から調べれば良いか分からず、
手が止まってしまうことがあります。

今回調べたことが、同じように困っている方の助けになれば幸いです。


## グラレコ
![grareco_without_apt-key.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/911542/ad89fbae-e6d4-4dc8-a1cf-0d6ca6b9e678.png)


