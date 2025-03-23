---
title: "NewRelicにSendGridのReputationを送る方法"
emoji: "📩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NewRelic", "SendGrid", "o11y"]
published: false
publication_name: "irsc"
---

## はじめに
こんにちは！！株式会社Inner Resourceの檜野です。
NewRelicのダッシュボードでSendGridのReputationを監視したいと思った際に、公式ドキュメントだけだと連携設定が完全には出来ず試行錯誤する事になりました…

同様の事を試される方の一助となればと考え、今回は最終的な実装内容を共有します！

## NewRelic公式が提示するSendGridとの連携方法
公式に記載のある連携方法は以下の2パターンがあります。
1. AWS S3にあるログデータをAWS Lambda等を使用してNewRelicに送信する
2. SendGridのEvent Webhookを用いて、設定されたEventの度にNewRelicのLog APIを用いてログデータを送信する

2番だとReputationを送る事が出来ない為、Reputationを送るという目的の為には1番にて実装することになります。
https://newrelic.com/jp/instant-observability/sendgrid-quickstart

## 実装の大まかな流れ
1. データを取得するSendGridのWeb APIの確認
2. SendGridのWeb APIを叩く為のAPI Keyの発行
3. S3, Lambdaの作成
4. 3で作成したS3に対しNewRelicへ送信設定
5. NewRelicのダッシュボードにReputationを監視するwidgetを追加

## データを取得するSendGridのWeb APIの確認
SendGridのWeb APIで取得した情報を、S3に保存する流れになるので、Reputationを取得出来るAPIを探し、curl等を用いて実際の挙動を確認して使用する必要があります。

今回使用するAPIは以下です。
### API名
GET User Account [GET]

### Request
```
GET https://api.sendgrid.com/v3/user/account HTTP/1.1
```

### Response
```
HTTP/1.1 200
{
  "type": "free",
  "reputation": 99.7
}
```

### curl例
```
curl -X GET \
  https://api.sendgrid.com/v3/user/account \
  -H "Authorization: Bearer YOUR_API_KEY"
```

https://sendgrid.kke.co.jp/docs/API_Reference/Web_API_v3/user.html#GET-User-Account-GET

:::message
APIの内容は執筆時点から変わっている可能性があるので、しっかりと公式のドキュメントを確認した上で使用しましょう。
:::

## SendGridのWeb APIを叩く為のAPI Keyの発行
APIを叩くためには、API Keyが必要になります。
API Keyを発行する際には権限を設定する事が出来ます。

必要最低限な権限だけを与えたAPI Keyを発行したいのですが、通常のAPI Key作成画面で設定出来る権限だと今回のAPIを叩く事が出来ず、Full Accessな権限を与えるしかなくセキュリティ的によろしくない事になってしまいます。

そこで裏技的に、APIでAPI Keyの権限を設定する方法を用いて、必要最低限な権限のAPI Keyを作成します。下記に手順を記載します。

### 前提条件
- Reputation取得に使うAPI Keyの名前を `fetch-reputation` とします

### 手順の流れ
1. `fetch-reputation` の作成の際に、API Key Permissionsを Restricted Access かつ Access Details を全て No Access で設定
2. Full Access 権限の一時的なAPI Keyを発行
3. 2で作成したAPI Keyを使用して `fetch-reputation` の権限をAPIで設定する
4. `fetch-reputation` を使用しReputation取得のAPIをcurlで叩きReputationが取得出来ることを確認する

### 使用するAPI名
Generate a new API Key for the authenticated user [POST]

### Request
```
POST https://api.sendgrid.com/v3/api_keys HTTP/1.1
```

### Request Body
```
{
  "name": "My API Key",
  "scopes": [
    "mail.send",
    "alerts.create",
    "alerts.read"
  ]
}
```

### Response
```
HTTP/1.1 201
{
  "api_key": "SG.xxxxxxxx.yyyyyyyy",
  "api_key_id": "xxxxxxxx",
  "name": "My API Key",
  "scopes": [
    "mail.send",
    "alerts.create",
    "alerts.read"
  ]
}
```

### curl例
scopesの権限がReputation取得に必要な権限です。

```
curl -X PUT \
  https://api.sendgrid.com/v3/api_keys/{fetch-reputationのAPI Key ID}\
  -H "Authorization: Bearer {手順2で作成したAPI Key}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "fetch-reputation",
    "scopes": [
      "user.account.read",
      "user.credits.read",
      "user.email.read",
      "user.profile.read",
      "user.username.read"
    ]
  }'
```

https://sendgrid.kke.co.jp/docs/API_Reference/Web_API_v3/API_Keys/index.html#Generate-a-new-API-Key-for-the-authenticated-user-POST

:::message alert
この設定方法は公式ドキュメントに載っていないものです。
私がSendGridの日本の正規代理店である構造計画研究所のサポートに問い合わせた結果を元に作成したものです。
執筆時では動いていますが、実際に実装される際にはしっかりと動作確認を行なった上で自己責任で使用して下さい。
:::

## S3, Lambdaの作成
Reputationのログを貯めていくS3バケットの作成、またAPIを叩きReputationを取得しS3に保存を行うLambdaを作成します。
定期的にReputationを取得する場合はLambdaにEvent Bridgeを設定してあげてください。
詳細な設定手順は省きますが、作成したlambdaの実装例(Ruby)を記載しておきます。適宜参考にしてください。

```
# runtime: AWS Lambda Ruby 3.3 / x86_64
require 'aws-sdk-s3'
require 'net/http'
require 'uri'
require 'json'

def lambda_handler(event:, context:)
  sendgrid_api_key = ENV['SENDGRID_API_KEY']
  s3_bucket = ENV['S3_BUCKET']

  raise StandardError.new("sendgrid_api_key is not set") if sendgrid_api_key.nil? || sendgrid_api_key.empty?
  raise StandardError.new("s3_bucket is not set") if s3_bucket.nil? || s3_bucket.empty?

  s3_key = "sendgrid_reputation_#{Time.now.strftime('%Y%m%d')}.json"

  uri = URI.parse('https://api.sendgrid.com/v3/user/account')

  begin
    request = Net::HTTP::Get.new(uri)
    request['Authorization'] = "Bearer #{sendgrid_api_key}"

    response = Net::HTTP.start(uri.hostname, uri.port, use_ssl: true) do |http|
      http.request(request)
    end

    if response.is_a?(Net::HTTPSuccess)
      # responseにはSendGridのプラン情報等も含まれており、ログとして必要のない情報は残さない為に、reputationのみを取得
      response_data = JSON.parse(response.body)
      reputation = response_data['reputation']
      reputation_json = { reputation: reputation }.to_json

      s3 = Aws::S3::Resource.new
      obj = s3.bucket(s3_bucket).object(s3_key)
      obj.put(body: reputation_json)
      puts "Data saved to S3: #{s3_key}"
    else
      raise StandardError.new("Failed to retrieve data: #{response.code} #{response.message}")
    end
  rescue => e
    raise StandardError.new("#{e.class}: #{e.message}")
  end
end
```

## S3に対しNewRelicへ送信設定
NewRelic が提供している AWS Serverless Application Repository の Lambdaアプリケーション を使用します。詳細は以下の公式ドキュメントや記事を参考にしてください。
https://docs.newrelic.com/docs/logs/forward-logs/aws-lambda-sending-logs-s3/

https://qiita.com/kooohei/items/7eee1445b8fea7308c54

## NewRelicのダッシュボードにReputationを監視するwidgetを追加
詳細な設定手順は省きますが、widgetを作成するNRQLの例を以下に記載します。
監視方法に応じて適宜変更してください。

```
SELECT latest(reputation) AS 'latest reputation (%)'
FROM Log
WHERE aws.s3_bucket_name = '{Reputationのログが保存されているS3のバケット名}'
SINCE 1 day ago
```

## まとめ
この記事が実装に困っている方の助けになれば嬉しいです！！
