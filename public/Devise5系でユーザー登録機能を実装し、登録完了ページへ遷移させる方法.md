---
title: 【Rails】Devise 5系でユーザー登録機能を実装し、登録完了ページへ遷移させる方法
tags:
  - RUNTEQ
  - Rails
  - devise
  - プログラミング初学者
private: false
updated_at: '2026-10-09T17:59:40+09:00'
id: eea97bcdb7fa206ed73f
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

この記事では、プログラミング初学者の僕が、1つのアプリを本リリースするまでに開発中につまずいたところや苦戦したところを、自分なりに整理してまとめ、理解を深めることを目的としています。

今回は、RailsアプリでDevise 5系を使ったユーザー登録機能の実装と、登録完了後のリダイレクト先を変更する方法についてです。

Deviseを使えばユーザー登録やログイン機能を比較的簡単に実装できますが、登録完了後の遷移先を変更しようとしたところ、思ったように動作せず苦戦しました。

その原因を調べる中で、Deviseが用意しているControllerやメソッドの役割について理解が深まったので、備忘録としてまとめます。

# 実装環境

今回の実装では、Rails 7以降とDevise 5系を使用しています。

※ 実際に使用したRuby・Rails・Deviseのバージョンをここに追記してください。

|技術|バージョン|
|---|---|
| Ruby | 3.4.10 |
| Rails | 8.1.3 |
| devise | 5.0.4 |


# 背景

Railsアプリのトップページを作成した後、最初にユーザー登録機能を実装しようと考えました。

今回は、ユーザー登録時に以下の3項目を入力してもらう設計にしています。

- ニックネーム
- メールアドレス
- パスワード

また、登録完了後はトップページではなく、独自の「アカウント作成完了ページ」に遷移させたいと考えました。

そこで、Railsの認証機能を効率よく実装できるGemである**Devise**を導入しました。

# Deviseとは？

Deviseは、Railsアプリに認証機能を追加するためのGemです。

例えば、以下のような機能を提供しています。

- ユーザー登録
- ログイン・ログアウト
- パスワードの再設定
- メールアドレスの確認（設定した場合）

通常、これらの機能を自分で実装するには、ControllerやModelなどにさまざまな処理を書く必要があります。

しかし、Deviseには認証に必要な処理があらかじめ用意されているため、比較的少ないコードで実装できます。


# 1. Deviseの導入
まずは以下の手順を踏んでDeviseをアプリに導入させました。
1. `Gemfile`で`gem "devise"`をインストール
2. Deviseの初期設定ファイルを生成
```zsh
bin/rails generate devise:install
```

# 2. Deviseを使うUserモデルの作成

続いて、Deviseを利用するUserモデルを作成します。
```zsh
bin/rails generate devise User
```
このコマンドを実行すると、UserモデルとDeviseに必要なカラムを定義したマイグレーションファイルなどが生成されます。

通常のrails generate model Userとは異なり、Deviseの認証機能に必要な設定が自動的に追加されるため便利です。

なお、Devise専用の特殊なモデルを作るわけではなく、通常のRailsモデルにDeviseの機能を組み込むイメージ。

# nicknameカラムを追加する

今回のアプリでは、メールアドレスとパスワードに加えてニックネームも登録してもらいます。

そこで、生成されたマイグレーションファイルに以下のカラムを追加しました。

```ruby
t.string :nickname, null: false
```

その後、マイグレーションを実行。

# ルーティングの確認
`config/routes.rb`
```ruby
devise_for :users
```
上記の記載があればOK！

これによって、
- ユーザー登録（`new` / `create` アクション）
- ログイン/ログアウト(`session`)
などの必要なルーティングが設定できたことになりました。

# DeviseのControllerとViewについて

ここで、Deviseを使う際のControllerとViewについて簡単に整理します。

### Controller

通常のRailsアプリでは、必要なControllerを作成し、newやcreateなどのアクションを自分で実装します。

一方、Deviseには認証処理を担当するControllerがあらかじめ用意されています。

ただ、Devise標準の動作を変更したい場合は、Controllerを継承してカスタマイズできます。
（後ほど使用します。）

### View

Deviseには、ユーザー登録やログイン画面などのViewも用意されています。

デザインやフォームの内容を変更したい場合は、以下のコマンドでViewファイルを生成できます。

```zsh
bin/rails generate devise:views
```
これにより、`app/views/devise/`配下にDeviseのViewファイルが生成されます。

# 4. nicknameを登録できるようにする

Deviseの標準的な登録処理では、独自に追加した`nickname`はそのままでは登録用パラメータとして許可されません。

なので、**DeviseのStrong Parameters**の設定を変更します。

`app/controllers/application_controller.rb`
```ruby
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:nickname])
  end
end
```

| コード | 意味 |
|---|---|
| `before_action` | 「処理を実行する前に」 |
| `configure_permitted_parameters` | 「DeviseControllerで、任意のストロングパラメータを許可するメソッド」 |
| `if: :devise_controller?` | 「もしこれが Decvise 標準の Controllerだったら」|

# 5. 登録完了後の遷移先を変更したい

ここまでで、Deviseを利用したユーザー登録機能の基本的な準備ができました。

次に、登録完了後の遷移先を変更します。

今回実装したかったのは、以下のような動作です。

```
ユーザー登録フォーム
       ↓
登録情報を送信
       ↓
ユーザー登録成功
       ↓
アカウント作成完了ページ
```

Deviseでは、ユーザー登録に成功すると、登録後の遷移先を決めるメソッドが呼び出されます。

そのメソッドがこれ。

```ruby
after_sign_up_path_for(resource)
```

このメソッドの戻り値によって、登録成功後のリダイレクト先が決まります。


# 6. ApplicationControllerに書いても反映されない？

最初は、`ApplicationController`に以下のメソッドを定義すれば良いのだと考えてました。

```ruby
def after_sign_up_path_for(resource)
  registration_complete_path
end
```

しかし、この方法では意図したページに遷移しませんでした。。

## `after_sign_in_path_for` との違い

Deviseには、以下2つのメソッドが存在しています。

- `after_sign_in_path_for`：ログイン後の遷移先を決める
- `after_sign_up_path_for`：ユーザー登録後の遷移先を決める


`after_sign_in_path_for`は、`ApplicationController`で定義し上書きするが、

一方、`after_sign_up_path_for`は、`Devise::RegistrationsController`側に定義されています。

なので、`ApplicationController`に同じメソッドを書いても、言うことを聞きません。。

Devise 5系の公式ソースコードでも、`after_sign_up_path_for`は`Devise::RegistrationsController`内に定義されています。

今回の問題は、
最初はDevise 5系の仕様変更で従来の設計から変わったのかなというように考えていたのですが、
Devise 4系でもRegistrationsController側に存在していたようなので、
単に自分が**どのメソッドがどこのControllerに定義されているのか**を理解出来ていなかったことが原因でした。

# 7. カスタムRegistrationsControllerを作成する

今回の解決方法は、Deviseの`RegistrationsController`を継承したControllerを作成し、`after_sign_up_path_for`を上書きすることです。

### ① Controllerを作成

`app/controllers/users/registrations_controller.rb`を作成。
```ruby
class Users::RegistrationsController < Devise::RegistrationsController
  protected

  def after_sign_up_path_for(_resource)
    registration_complete_path
  end
end
```

`after_sign_up_path_for(_resource)`
登録完了後の遷移先を指定するメソッド.
引数の`_resource`には登録対象のユーザーが渡されますが、今回の処理では使用しないため、先頭に`_`を付けています。

### ② ルーティングを変更

次に、Deviseが今回作成したControllerを使用するように設定します。

`config/routes.rb`
```ruby
devise_for :users, controllers: {
  registrations: "users/registrations"
}
```

これで、ユーザー登録に関する処理には、カスタマイズした`Users::RegistrationsController`が使用されます。

Devise標準のControllerを丸ごと書き換えるのではなく、**継承して必要なメソッドだけを変更**するようにしています。

# 8. アカウント作成完了ページを作成する

続いて、登録成功後に表示するページを作成します。

今回は、Deviseの認証処理とは別の通常のRails Controllerを使用しました。

### ① Controllerを作成
```zsh
bin/rails generate controller RegistrationComplete show
```

### ② ルーティングを設定

`config/routes.rb`
```ruby
Rails.application.routes.draw do
  devise_for :users, controllers: {
    registrations: "users/registrations"
  }

  get "registration/complete",
      to: "registration_complete#show",
      as: :registration_complete

end
```

### ③ ControllerとViewを確認

`app/controllers/registration_complete_controller.rb`

`app/views/registration_complete/show.html.erb`

これで、ユーザー登録に成功すると、登録完了ページに遷移する構成になりました。

# 9. Turboを使用している場合の注意点

今回、登録後の画面遷移について、Rails7以降で使用されることの多いTurboとの関係性についても少し実装時に引っ掛かったので、
記載しておきたいと思います。

### Turboとは？

Turboは、JavaScriptを大量に記述しなくても、ページ遷移やフォーム送信を効率よく扱える仕組みです。

Rails 7以降の標準的な構成では、Turboが利用されることがあります。

### Deviseのnavigational_formats

Deviseには、通常の画面遷移として扱うリクエスト形式を指定する設定があります。

`config/initializers/devise.rb`
```ruby
config.navigational_formats = ["*/*", :html, :turbo_stream]
```
これは、DeviseがHTMLやTurbo Streamなどのリクエスト形式をどのように扱うかに関係する設定です。

Rails7以降、
**`form_for`で作られたフォームはデフォルトで`turbo_stream`形式として送信される**一方で、

Devise側は`respond_with`でHTML形式を前提にリダイレクト先を決めるから、
`turbo_stream`形式だと自分で設定した遷移先には飛ばない問題も発生していました。


そこで、上記の箇所を見てみると、`:turbo_stream`が記載されていなかったので、
追加して再度確認したところ正常に動作するようになりました。


もしフォーム送信後の画面遷移が想定どおりにならない場合は、次の点を確認すると原因を切り分けやすくなります。

- 実際に使用しているDeviseのバージョン
- config.navigational_formatsの設定
- フォーム送信時のリクエスト形式
- Controllerで呼び出されているメソッド
- レスポンスのステータスコードとリダイレクト先

Railsのログやブラウザの開発者ツールを確認することで、どのようなリクエストが送信され、どのようなレスポンスが返されているかを調べられます。

# 10. まとめ

今回は、Devise 5系を使用したユーザー登録機能の実装と、登録完了後のリダイレクト先を変更する方法についてまとめました。

特に勉強になったのは、Deviseが用意しているControllerやメソッドの仕組みです。

最初は、`ApplicationController`にメソッドを書けば動くものだと思っていました。

しかし、実際には`after_sign_in_path_for`と`after_sign_up_path_for`では定義場所やカスタマイズ方法が異なり、
今回のケースでは**`RegistrationsController`を継承する**必要がありました。

今後も「なぜその方法で動くのか」という部分をしっかり理解していきたいと思います。

自分と同じようにプログラミングを学び始めた方が、Deviseの実装でつまずいた際に、少しでも参考になればうれしいです。

# グラレコ
![grareco_devise5_user-registrations.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/911542/07b99d07-d0cf-4305-ba10-7d7538f732ea.png)
