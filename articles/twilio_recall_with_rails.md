---
title: "Railsで相手が応答しなかった場合にリコールする機能を作ってみた"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [twilio, rails, ruby]
published: true
---

# はじめに

この記事は[「Twilioを使うためのコツ、TIPSなど、Twilioのことなら何でも共有しよう！」](https://qiita.com/advent-calendar/2021/twilio)の15日目の記事です。
実務でリコールの機能を実装する際に詰まった部分の知見を共有しようかと思います。

Twilioのアカウント作成・電話番号の購入の説明はしていません。
使用可能な電話番号が用意できている前提で説明させていただきます。

また、実際に実務で使っているコードではなく今回の記事用に再実装したコードになるため、
こちらで紹介するコードが実際に動作できるかは確認できていませんのでご了承ください。

# 背景

実務で以下の要件の対応をすることがありました。

- 任意のイベント発生時にイベントが発生したユーザーへコールを行う
- 応答しない場合は3回までリコールを行う

Twilioはコールされる側の実装などの経験はありましたが、コールする側は初めてだったのでまず方法から調査していきます。

## 任意の電話番号へコールする方法

Rubyで発信を行う際、Twilioが提供する[SDK](https://github.com/twilio/twilio-ruby)を使って以下のコードを実行することで簡単に実現することができます。

公式ドキュメント: <https://www.twilio.com/docs/voice/api/call-resource>

```ruby
require 'twilio-ruby'

account_sid = 'ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
auth_token = 'yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy'

twilio_phone_number = '+815012345678'
call_phone_number = '+819012345678'

client = Twilio::REST::Client.new(account_sid, auth_token)

call_options = {
  twiml: '<Response><Say>Hello world!</Say></Response>',
  from: twilio_phone_number,
  to: call_phone_number,
}
client.calls.create(**call_options)
```

## `timeout`オプションについて

`call_options`に設定しているプロパティは[こちら](https://www.twilio.com/docs/voice/api/call-resource#create-a-call-resource)に記述されていますが、
その中に`timeout`が任意で設定できるようになっており、応答されるまでTwilio側で勝手にリコールをしてくれます。
初期値が60秒となっていて、1コールにおよそ30秒(厳密には29秒ぐらい？)ほど要するので2回かかってくることになります。

単純計算をすればこのオプションには90秒と設定すれば要件に対応できそうですよね。
確かにこれでも実現はできるのですが、そこに落とし穴がありました。

## Twilioからのコールが非通知になる場合がある

2回目以降のコールからほぼほぼ非通知としてコールされてしまったのです。
今回発信先のユーザーは社内ではなく一般の方になるので、非通知設定されてしまっているとこちらの要件は満たせないため、なんとか解決する必要がありました。

FAQにて[Twilio から発信した通話が、非通知でかかってきます。なぜですか？](https://cloudapi.zendesk.com/hc/ja/articles/206420981-Twilio-%E3%81%8B%E3%82%89%E7%99%BA%E4%BF%A1%E3%81%97%E3%81%9F%E9%80%9A%E8%A9%B1%E3%81%8C-%E9%9D%9E%E9%80%9A%E7%9F%A5%E3%81%A7%E3%81%8B%E3%81%8B%E3%81%A3%E3%81%A6%E3%81%8D%E3%81%BE%E3%81%99-%E3%81%AA%E3%81%9C%E3%81%A7%E3%81%99%E3%81%8B-)というものがあったので確認してみましたが、`from`などの指定が足りていないとのことで、今回は全く関係なしでした。
そもそも1回目は非通知になることはなかったので、そりゃそうですよね...

めぼしい情報もなく詰んでしまったため以下ツイートをしたところ、Twilioの事業部に所属する方からリプライをいただきました。

https://twitter.com/chiino58/status/1445573287971483661

Twilioの仕組みとして、1回目のコールで応答されなかった場合の冗長化として、
海外のルートを用意しているためにおこりうる現象とのことでした。

今回検証する上で1回目のコールが非通知になることはなく、ほぼほぼ2回目以降がすべて非通知になってしまっていたため、こちらの説明いただいた内容で腑に落ちました。

ということで対応方法としては自前で応答されなかった場合にリコールする機能を作ってみました。

# 実装

今回実装したコードは以下リポジトリで公開していますので、コードだけ見たい方はこちらからどうぞ。
<https://github.com/n04h/rails-twilio-recall-example>

## 全体のフロー

![全体のフロー](https://storage.googleapis.com/zenn-user-upload/28f9cf26f309-20211214.png)

## 環境の準備

Rails + MySQLのDocker環境を用意します。
このあたりはほか記事で丁寧にまとめてくださっている方がたくさんいるかと思いますので割愛します。

## SDKのインストール

SDKのGitHubのリポジトリは以下になります。
<https://github.com/twilio/twilio-ruby>

こちらをGemfileに追加して、`bundle install`を実行します。

```ruby:Gemfile
gem 'twilio-ruby'
```

## コールした履歴の管理を行うモデルの作成

モデルには以下の機能を用意しておきます。

- コールの実行
- リコールの実行可否確認
- リコールの実行

```ruby:app/models/twilio_call_resource.rb
class TwilioCallResource < ApplicationRecord
  class << self
    def call!(client:, **call_options)
      resource = client.calls.create(**call_options)
      create!(
        call_sid: resource.sid,
        call_options_json: call_options.to_json
      )
    end
  end

  def call_options
    JSON.parse(call_options_json).symbolize_keys
  end

  def can_recall?
    called_count < 3
  end

  def recall!(client:)
    raise "リコールできません" if can_recall?

    resource = client.calls.create(**call_options)
    update!(
      call_sid: resource.sid,
      called_count: called_count + 1
    )
  end
end
```

スキーマの定義は今回Railsが用意しているマイグレーションは使わず、`ridgepole`を使います。

```ruby:Schemafile
create_table "twilio_call_resources", force: :cascade do |t|
  t.string "call_sid", null: false
  t.json "call_options_json", null: false
  t.integer "called_count", default: 1, null: false
  t.datetime "created_at"
  t.datetime "updated_at"
  t.index ["call_sid"], unique: true
end
```

ドライランして問題ないかチェック。

```shell
bundle exec ridgepole --config ./config/database.yml --file ./db/Schemafile --apply --dry-run
```

問題なさそうなのでこれでマイグレーション実行。

```shell
bundle exec ridgepole --config ./config/database.yml --file ./db/Schemafile --apply
```

## トリガーとなるAPIの作成

任意のイベント時にユーザーへコールするというコードのサンプルとして、トリガーとなるAPIを用意していきます。

ルーティングは`POST /twilio_api/calls`としましょう。

```ruby:config/routes.rb
Rails.application.routes.draw do
  namespace :twilio_api do
    resources :calls, only: :create
  end
end
```

コールの実行・コール履歴を作成するようにします。

```ruby:app/controllers/twilio_api/calls_controller.rb
module TwilioApi
  class CallsController < ApplicationController
    def create
      TwilioCallResource.call!(client: client, **call_options)
    end

    private

    def client
      Twilio::REST::Client.new(ENV['TWILIO_ACCOUNT_SID'], ENV['TWILIO_AUTH_TOKEN'])
    end

    def call_options
      {
        twiml: '<Response><Say>Hello world!</Say></Response>',
        status_callback: status_callback_url,
        status_callback_event: ['completed'], # Twilio側で発信の処理が終了時にコールバックされるようにする
        status_callback_method: 'POST',
        from: '+8105012345678',
        to: '+8109012345678',
        timeout: 30 # デフォルトは60秒なので2回コールされてしまうのを防ぐ
      }
    end

    def status_callback_url
      "#{ENV["HOST_URL"]}/twilio_api/callback"
    end
  end
end
```

## コールバック用APIの作成

コールのステータスをもとにリコールするかを処理するAPI、`POST /twilio_api/callback`を用意します。

```ruby:config/routes.rb
Rails.application.routes.draw do
  namespace :twilio_api do
    resources :calls, only: :create
    resource :callback, only: :create
  end
end
```

以下の場合はユーザー側で応答されないケースになるので、全て指定してしまいます。

- `busy`
- `failed`
- `no-answer`

```ruby:app/controllers/twilio_api/callbacks_controller.rb
module TwilioApi
  class CallbacksController < ApplicationController
    def create
      if need_recall?
        call_resource.recall!(client: client)
      else
        call_resource.destroy!
      end
    end

    private

    def client
      Twilio::REST::Client.new(ENV["TWILIO_ACCOUNT_SID"], ENV["TWILIO_AUTH_TOKEN"])
    end

    def call_resource
      TwilioCallResource.find_by!(call_sid: params[:CallSid])
    end

    def need_recall?
      %w[busy failed no-answer].include?(params[:CallStatus])
    end
  end
end
```

以上で実装できるかと思います。コード自体はシンプルですね。

# さいごに

Twilio + Railsはそこそこ知見がたまってきているのですがアウトプットが全然できていないため、これを機会に余裕のある時にちょこちょこまとめていこうかと思っています。
Twitterなどでなにか実装したい要件で悩んでいましたら是非お声がけください。
