---
title: "ALB + RailsでSSLを有効にする"
emoji: "📑"
type: "tech"
topics: ["rails", "AWS"]
published: false
---

ALB を使っている状態で `config.force_ssl` を true に変更したら
ヘルスチェックで失敗する様になったので、対策と SSL の手順を残しておきます。

## 前提

- ALB -> Rails という構成
- Rails 6.1.4

## 目次

1. `config.force_ssl` を true にする
2. ヘルスチェックのリクエストはリダイレクトしない様にする

## 1. force_ssl を true にする

```ruby
config.force_ssl = true
```

Rails では `force_ssl` を true にすると http でアクセスした時 https へ
リダイレクトするようになり、さらに `Strict-Transport-Security` がヘッダーに
追加されます。

https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/ssl.rb
この辺のコードを読むと force_ssl を true にした時の挙動が分かります。

```ruby
def call(env)
  request = Request.new env

  if request.ssl?
    @app.call(env).tap do |status, headers, body|
      set_hsts_header! headers
      flag_cookies_as_secure! headers if @secure_cookies && !@exclude.call(request)
    end
  else
    return redirect_to_https request unless @exclude.call(request)
    @app.call(env)
  end
end
```

`request.ssl?` の部分をさらに追いかけると

```ruby
# https://github.com/rack/rack/blob/master/lib/rack/request.rb#L344

def ssl?
  scheme == 'https' || scheme == 'wss'
end
```

となっています。

ロードバランサーはターゲットへリクエストを送るときにヘッダーに
`HTTP_X_FORWARDED_PROTO` を自動で追加してくれるので、訪問者が
https でアクセスしたときここの schema には https が入ります。

まぁ設定自体はこれで終わりなのですが、これだけだとロードバランサーの
ヘルスチェックが失敗してしまいました。

## 2. ヘルスチェックのリクエストはリダイレクトしない様にする

原因ですが、ロードバランサーがヘルスチェックで使うプロトコルを http に設定していたので
レスポンスで 301 を返しヘルスチェックで失敗しているようでした。

```ruby
  # https://github.com/rails/rails/blob/main/actionpack/lib/action_dispatch/middleware/ssl.rb#L16
  # Requests can opt-out of redirection with +exclude+:
  #
  # config.ssl_options = { redirect: { exclude: -> request { /healthcheck/.match?(request.path) } } }
```

を読むと特定のリクエストはリダイレクトしない様に設定出来そうです。

```ruby
config.force_ssl = true
config.ssl_options = { redirect: { exclude: -> request { request.env['HTTP_USER_AGENT'].include?('ELB-HealthChecker') } } }
```

と修正してヘルスチェックのリクエストはリダイレクトしない様に設定したことで
ヘルスチェックが無事パスしました。

## 最後に

ここまで調べて、そもそもヘルスチェックで使うプロトコルを https にすれば良いのでは？？
と思いました。。。(まだ試してない)
