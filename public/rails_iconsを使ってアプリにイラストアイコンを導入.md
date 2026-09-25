---
title: rails_iconsを使ってアプリにSVGアイコンを導入
tags:
  - RUNTEQ
  - Rails
  - Lucide
  - アイコン
  - プログラミング初学者
private: false
updated_at: '2026-09-25T19:11:20+09:00'
id: 5b630b8ec086b0dbe8dc
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
Railsアプリに**rails_iconsライブラリを導入し、外部サービスからイラストアイコンを活用すること**
についてです。

## 背景

Railsアプリを開発している中で、
当初は自分で作成したPNG形式のイラストを静的アセットで保存し、
要所要所でアイコンとして使用していました。

例えば、
- パズル
- タイマー
- ユーザー

などのアイコンです。

しかし、開発を進めていくうちに、
***「この方式で入れ続けたら、少しずつデータ量が増えてしまうのでは？」***
***「必要になるたびに自分で作成するのは時間もかかるし、管理も大変になるのでは？」***

と思うようになりました。
そこで、既存のアイコンライブラリをRailsアプリから利用する方法を調べ、実際に導入してみることにしました。

## *rails_icons* とは

今回利用したのが、[rails_icons](https://github.com/Rails-Designer/rails_icons) というgemです。

`rails_icons`は、RailsアプリでさまざまなSVGアイコンライブラリを利用しやすくするためのgemです。
Lucide、Heroisons、Tabler Iconsなど、
複数のアイコンライブラリに対応していて、RailsのViewから共通のiconヘルパーを使ってSVGアイコンを表示できます。

まずは`Gemfile`に追加します。
```ruby
gem "rails_icons"
```

その後、
```bash
bundle install
```
を実行します。

または、以下のコマンドでも追加できます。
```bash
bundle add rails_icons
```

## アイコンライブラリ

ところで、一口にアイコンライブラリといってもさまざまな種類があります。
以前からよく知られているものとして **[Font Awesome](https://fontawesome.com/)** がありましたが、
今回は他の選択肢についても調べてみました。

### Heroicons
[Heroicons](https://heroicons.com/)

Tailwind CSS公式チームが開発してるから、Tailwindとの親和性が高い。

シンプルでモダンなデザインが特徴（アウトライン / ソリッドなどのスタイルが用意されている。）


### Lucide
[Lucide](https://lucide.dev/)

*Feather Icons*をベースに発展した、オープンソースのアイコンライブラリ。
シンプルな線で構成されたアイコンが多い。
WebアプリのUIにも馴染みやすいデザイン。


### Tabler Icons
[Tabler Icons](https://tabler.io/icons)

非常に多くのアイコンが用意されているオープンソースのアイコンライブラリ。
種類が豊富なので、管理画面、ダッシュボードなど、多くのアイコンを使うUIでも選択肢を見つけやすそう。

### 今回はLucideを選択

今回のアプリではTailwind CSSを使用しており、アイコンを大量に使う予定もありません。

そのため、シンプルなデザインで必要なアイコンも揃っていた***Lucide***を使ってみることにしました。


## ***rails_icons***の導入

`rails_icons`をインストールしたら以下のコマンドを実行します。
```bash
bin/rails generate rails_icons:install --library=lucide
```
これによって`rails_icons`を利用するための設定が追加されます。

例えば、
```bash
config/initializers/rails_icons.rb
```
が生成され、利用するアイコンライブラリなどの設定を管理できるようになります。

また、`/rails_icons`にアイコンのプレビューページが用意されるため、利用できるアイコンをブラウザ上から探すこともできます。

:::note warn
`/rails_icons`のプレビュールートは、使用しているバージョンや設定によってはそのままアクセス可能な状態になります。
本番環境で公開したくない場合は、development環境だけで利用できるようにルーティングを制限するなどの対応を検討した方が良さそうです。
（自分も今回、`routes.rb`にdevelopment環境に限定してmountさせました。）
:::

## LucideのSVGを同期する

Lucideのアイコンをローカルに同期する場合は、以下のコマンドを実行します。

```bash
bin/rails generate rails_icons:sync --library=lucide
```

同期されたSVGファイルは、
```
app/assets/svg/icons/lucide/outline/
```

などに保存されます。


:::note warn
**アイコン数には注意**
`rails_icons:sync`を実行すると、指定したライブラリのアイコンがまとめて同期されます。
Lucideだけでも多数のSVGファイル（1,500個以上）が存在するため、
「数個のアイコンしか使わないのに大量のSVGをGit管理したくない」という場合は、
必要なアイコンだけを配置する方法も検討できます。

今回は上記コマンドは実行せずに、
必要な数個のSVGだけを`app/assets/svg/icons/lucide/outline/`に格納して、
Viewから呼び出せるようにしました！
:::

## Viewでアイコンを表示する

SVGを利用できる状態になったら、Viewでは `icon`ヘルパーを使って表示できます。
```erb
<%= icon "SVGファイル名", class: "size-5 text-teal-700", "aria-hidden": true %>
```

`class`にはTailwind CSSクラスを指定できます。

例えば、
```
size-5
```
でサイズを指定したり、

```
text-steal−700
```
で色を指定できます。
SVGが`currentColor`を利用している場合、このように文字色のCSSクラスを使ってアイコンの色も変更できます。


また、
```ruby
"aria-hidden": true
```
を指定すると、そのアイコンを音声読み上げソフト（スクリーンリーダー）から隠すことができます。
（読まれないってこと。）

今回のように、アイコンの隣に同じ意味を表すテキストがあり、アイコン自体には追加の情報がない場合などに利用できます。

## JPEG と PNG と SVG

今回SVGアイコンを使うにあたって、
そもそもJPEG、PNG、SVGでは何が違うのかもここで改めて整理しました。

| | JPEG | PNG | SVG |
| --- | --- | --- | --- |
|種類|ラスター|ラスター|ベクター|
|圧縮|非可逆（戻らない）|可逆|図形データ|
|透明|できない|できる|できる|
|拡大|ボヤける|ボヤける|きれい|
|色の後変更|しにくい|しにくい|CSSで変えやすい（`text-teal-700`など）|
|向いてるもの|写真|ロゴ・透過画像・イラスト|UIアイコン・ロゴなど|
|Railsでの使い方|`image_tag "photo.jpg"`|`image_tag "stopwatch_24.png"`|`icon "timer", class: "size-5"`|

かなりざっくり分けると、

写真 → ***JPEG***
透過が必要なイラスト → ***PNG***
拡大縮小したり、CSSで色を変更したりしたいUIアイコン → ***SVG***

というように、用途にお応じて使い分けると良さそうです。

今回のようなWebアプリ内の小さなUIアイコンについては、SVGを使うメリットが大きいことが分かりました。

## まとめ

今回は***rails_icons***というgemを使って、
***Lucide***のSVGアイコンをRailsアプリに導入する方法を試してみました。

これまではアイコンが必要になるたびにPNG画像を作成していましたが、
既存のSVGアイコンライブラリを利用することで、

- **自分でアイコンを作成する手間を減らせる**
- **UI全体のアイコンデザインを統一しやすい**
- **拡大・縮小しても画質が劣化しにくい**
- **Tailwind CSSなどからサイズや色を調整しやすい**

といったメリットがあることが分かりました。

エンジニアの方にとっては当たり前の内容かもしれませんが、僕自身、今回調べて実際に導入したことで
「PNGとSVGをどう使い分ければ良いのか」についても理解が深まりました。

もし自分と同じようにプログラミングを学び始めたばかりで、
「Railsでアイコンを使いたいけど、毎回画像を用意するのは大変......」
と思っている方がいれば、この記事が少しでも参考になれば幸いです。



## グラレコ
![grareco_rails_icons.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/911542/7522f895-33ca-47f7-a032-c660198e8d94.png)
