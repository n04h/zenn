---
title: "Railsで相手が応答しなかった場合にリコールする機能を作ってみた"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [twilio, rails, ruby]
published: false
---

# はじめに

この記事は[「Twilioを使うためのコツ、TIPSなど、Twilioのことなら何でも共有しよう！」](https://qiita.com/advent-calendar/2021/twilio)の15日目の記事です。
実務でリコールの機能を実装する際に詰まった部分の知見を共有しようかと思います。

Twilioのアカウント作成・電話番号の購入の説明はしていません。
使用可能な電話番号が用意できている前提で説明させていただきます。

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

# aaa
