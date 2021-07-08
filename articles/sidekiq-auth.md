---
title: "Sidekiqの管理画面を自前で用意した認証機能でアクセス制限する"
emoji: "🧑‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [sidekiq, rails, ruby]
published: false
---

自前で用意した認証機能でSidekiqの管理画面へのアクセスを制限する方法についての情報が見当たらなかったので備忘録として残しておきます。

Railsにて非同期処理をする際にSidekiqを使う場合に、
Sidekiqが用意している管理画面をルーティングにマウントすることがあると思います。

```ruby
require 'sidekiq/web'

Rails.application.routes.draw do
  mount Sidekiq::Web, at: '/sidekiq'
end
```

ただ、本番環境でも同様に確認したいこの画面に第三者がアクセスできてしまうのは困るので、
認証をつけるのがベターなのかなと思います。

## BASIC認証を使った方法

こちらの方法を使うなら環境変数などを使ってください。

```ruby
require 'sidekiq/web'

Rails.application.routes.draw do
  Sidekiq::Web.use(Rack::Auth::Basic) do |user, password|
    [user, password] == ['user', 'password']
  end
end
```

## Deviceを使った方法

`authenticated`を使うことで簡単に実装できます。

```ruby
authenticate :user do #authenticate
  mount Sidekiq::Web => '/sidekiq'
end
```

## 自前の認証機能(セッションを使った方法)

[constraints](https://guides.rubyonrails.org/routing.html#advanced-constraints)を使うことでいい感じに実装できます。

例えば`Admin`というモデルでセッションに`admin_id`が入っていたら認証が通る機能だとした場合、
以下の実装で実現可能です。

```ruby
class AdminAuthConstraint
  def matches?(request)
    return false if request.session[:admin_id].blank?

    Admin.exists?(request.session[:admin_id])
  end
end

Rails.application.routes.draw do
  mount Sidekiq::Web => '/sidekiq', constraints: AdminAuthConstraint.new
end
```

もし他に良い方法ありましたら、ご教授ください🙇‍♂️
