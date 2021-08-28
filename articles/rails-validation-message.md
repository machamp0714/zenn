---
title: "Railsの日本語化とエラーメッセージのカスタマイズ"
emoji: "🦁"
type: "tech"
topics: ["rails"]
published: false
---

Rails を日本語化したり、エラーメッセージを修正するとき、いつも調べている気がするので
この辺サクッと書けるように調べてまとめました。

## 前提

rails 6.1.3
ruby 2.7.0

## 目次

1. メッセージの確認
2. 日本語化
3. エラーメッセージのカスタマイズ
4. フォーマットをカスタマイズする

## 1. メッセージの確認

この様な User モデルを使って行きます。

```ruby
class User < ApplicationRecord
  validates :name, presence: true
  validates :password, presence: true
end
```

rails console を起動してエラーメッセージを見ていきます。

```ruby
user = User.new
user.valid?
user.errors.full_messages
#=> ["Name can't be blank", "Password can't be blank"]
```

こんな感じでエラーメッセージが英語で表示されてしまうので、モデルの属性とメッセージを
まずは日本語化して、さらにメッセージやフォーマットを自由に設定できるようにして行きます。

## 2.rails-i18n を使って日本語化する

```ruby
gem 'rails-i18n'
```

まず、 `rails-i18n` をインストールします。

```ruby
# config/application.rb

# デフォルトのロケールを:en以外に変更する
config.i18n.default_locale = :ja

# I18nライブラリに訳文の探索場所を指示する
config.i18n.load_path += Dir[Rails.root.join('config/locales/**/*.yml').to_s]
```

次に `config/application.rb` に上記の設定を追加します。
そして翻訳ファイルを `config/locales/models` 配下に作成します。

```ruby
ja:
  activerecord:
    attributes:
      user:
        name: 名前
	password: パスワード
```

コンソールを再起動して、エラーメッセージを確認すると日本語化されていることが分かります。

```ruby
user.errors.full_messages
#=> ["名前を入力してください", "パスワードを入力してください"]
```

ちなみに、この翻訳文は `User.human_attirbute_name(attribute)` で参照することも可能です。

## 3. エラーメッセージのカスタマイズ

ひとまずメッセージを日本語化することが出来たので、次はメッセージをカスタマイズして
行きます。Rails は以下の順番で翻訳ファイルを探索して最初に見つかったものを返します。

```ruby
activerecord.errors.models.[model_name].attributes.[attribute_name].key
activerecord.errors.models.[model_name].key
activerecord.errors.messages.key
errors.attributes.[attribute_name].key
errors.messages.key
```

これを参考にして翻訳ファイルを次の様に修正してみます。

```ruby
ja:
  activerecord:
    attributes:
      user:
        name: 名前
    errors:
      models:
        user:
	  blank: hogehoge
```

エラーメッセージを確認すると、

```ruby
user.errors.full_messages
#=> ["名前hogehoge", "パスワードhogehoge"]
```

とメッセージが変わっていることが分かります。
ただ、この書き方だとキーが `blank` の全ての属性が同じメッセージになってしまいます。

実務でアプリケーションを開発していると、属性ごとにメッセージを変えたい
ということもあります。そんな時は上記の探索方法を参考にして次の様に修正します。

```ruby
ja:
  activerecord:
    attributes:
      user:
        name: 名前
    errors:
      models:
        user:
	  blank: foobar
	attributes:
	  name:
	    blank: hogehoge
```

`activerecord.errors.models.[model_name].attributes.[attribute_name].key` が
一番優先順位が高いため、メッセージはこの様になります。

```ruby
user.errors.full_messages
#=> ["名前hogehoge", "パスワードfoobar"]
```

これで属性ごとにメッセージを変更することができる様になりました。

Rails は `uniqueness` や `numericality` など様々なヘルパーが用意されていますが、
それぞれのヘルパーとキーの関係は次の様にまとめられます。

| ヘルパー     | キー                      |
| ------------ | ------------------------- |
| confirmation | :confirmation             |
| acceptance   | :accepted                 |
| presence     | :blank                    |
| absence      | :present                  |
| length       | :too_short                |
| length       | :too_long                 |
| length       | :wrong_length             |
| length       | :too_short                |
| length       | :too_long                 |
| uniqueness   | :taken                    |
| format       | :invalid                  |
| inclusion    | :inclusion                |
| exclusion    | :exclusion                |
| associated   | :invalid                  |
| non-optional | association :required     |
| numericality | :not_a_number             |
| numericality | :greater_than             |
| numericality | :greater_than_or_equal_to |
| numericality | :equal_to                 |
| numericality | :less_than                |
| numericality | :less_than_or_equal_to    |
| numericality | :not_an_integer           |
| numericality | :odd                      |
| numericality | :even                     |

これを参考にして翻訳ファイルを書いて行きましょう〜

## 4. フォーマットをカスタマイズする

メッセージを修正する事はできたのですが、フォーマットは `%{attribute} %{message}` で
固定されていることが分かります。Rails6 以前はフォーマットを修正する事はできなかった様なのですが、Rails6 ではフォーマットもカスタマイズ出来る様になりました！！！

```ruby
# デフォルトはfalse
config.active_model.i18n_customize_full_message = true
```

そして翻訳ファイルを修正します。

```yaml
ja:
  activerecord:
    attributes:
      user:
        name: 名前
    errors:
      models:
        user:
          format: "%{attribute}: %{message}"
          blank: 必須項目です
```

エラーメッセージは次の様になります。

```ruby
user.errors.full_messages
#=> ["パスワード: 必須項目です", "名前: 必須項目です"]
```

これでフォーマットも修正することが出来ました！
ちなみにフォーマットも属性ごとに設定することが可能です。

```yaml
ja:
  activerecord:
    attributes:
      user:
        name: 名前
    errors:
      models:
        user:
          attributes:
            name:
              format: "%{attribute}: %{message}"
              blank: 必須項目です
```

この様に修正してみます。すると...

```yaml
user.errors.full_messages
#=> ["パスワードを入力してください", "名前: 必須項目です"]
```

と `name` と `password` で異なるメッセージを表示することが出来ました。

## 参考資料

[https://railsguides.jp/i18n.html#active-record モデルで翻訳を行なう](https://railsguides.jp/i18n.html#active-record%E3%83%A2%E3%83%87%E3%83%AB%E3%81%A7%E7%BF%BB%E8%A8%B3%E3%82%92%E8%A1%8C%E3%81%AA%E3%81%86)
[https://www.bigbinary.com/blog/rails-6-allows-to-override-the-activemodel-errors-full_message-format-at-the-model-level-and-at-the-attribute-level](https://www.bigbinary.com/blog/rails-6-allows-to-override-the-activemodel-errors-full_message-format-at-the-model-level-and-at-the-attribute-level)
