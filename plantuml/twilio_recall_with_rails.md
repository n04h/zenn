<!--
https://www.plantuml.com/plantuml/uml/jPDF3z8m6CRl_HJlKG-6WpUI6HAvUD88daqPEonE85k9Lxh6FuA9n8XAGC4e7Y0WHcA8ckp3l7JWj_3j9ambAfh8eIrht-VvFUkxGY2uBvetFTQWsRNT1gy9kCfTBm0UW6yMhCe5h_300VeLzslQhLP-g2hVSZstRZP4icBS5dKVhWMERnusQMRCiW33zwW-evg1PJ6MMC2v2wbN7gvSBfKXbofSvmqtPtFF2x9ZwKuUKBqnAk47CyeroiL1Loz6kvSFcE-8fb_BPtwnkgt7xp1yj8iUe-ndtfJawDYPY-HRkbGyI-StqNCzVqOb67OOImbCcObqJBBNeKB2G7adIQsZuNPEjazHZlHufRltA7uczlly5MxV-0KjgYugXU5Rb3qCoMxq3HpbRBlgefoWV8ZgRe8O4PE-t_xIVvs6-PbiYhA3yjK_8pBk2JYl1oysdoQRTTjSYfeUKEblAii0_aSOfrRD2EBXDFxOEeNWz8u2-7DF0PpMLItvFPW1xYr99XwiuYVB_Uq0_7MpfM-XG2DzrLy1
-->

```plantuml
@startuml
participant Rails as rails
database    Database as db
participant Twilio as twilio
actor User as user

== コール ==

rails -> rails: POST /twilio_api/calls
rails -> twilio: ユーザーへのコール実行
return: コール情報を返す

rails -> db: コール情報(CallSID)を保存する

twilio -> user: ユーザーへコールを行う

== 応答した場合 ==

twilio -> rails: POST /twilio_api/callback
rails -> db: コールバックされたCallSIDに紐づくデータを削除する

== 応答しなかった場合 ==

twilio -> rails: POST /twilio_api/callback

alt 3回以上のリコールの場合
  rails -> db: コールバックされたCallSIDに紐づくデータを削除する
end

alt 3回未満のリコールの場合
  rails -> twilio: ユーザーへのコール実行
  return: コール情報を返す

  rails -> db: コール情報(CallSID)、リコール回数を更新する

  twilio -> user: ユーザーへコールを行う
end

@enduml
```
