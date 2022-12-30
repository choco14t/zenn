---
title: 'リクエストIDを追加して調査を快適にする'
emoji: '‍🔍'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['graphql', 'apollo', 'typescript', 'nodejs', 'nestjs']
published: false
published_at: 2022-12-01 12:00
publication_name: 'spacemarket'
---

# はじめに

[スペースマーケット](https://www.spacemarket.com/)でバックエンドエンジニアをしている choco です。

スペースマーケットでは API Gateway として Apollo Gateway(Federation) を使用しています。

現在 API のログは出力しているのですが、1 リクエスト内で実行された GraphQL Query が把握しづらいという課題がありました。今回は対応策としてリクエストヘッダに ID を付与し、各アプリケーションに共有することで改善を試みました。

本記事では NestJS を使ってロギングが行えるまでのサンプルプロジェクトを作成していきます。サンプルプロジェクトでは下記のバージョンを使用しています。

- Node.js v18.12.1
- yarn v1.22.19
- @nestjs/cli v9.1.5

# プロジェクトの作成

はじめにルートディレクトリを作成します。

```sh
mkdir gateway-logging-example
```

次にゲートウェイ、アプリケーションディレクトリを作成します。ディレクトリはそれぞれ NestJS の CLI を使って作成します。

アプリケーションは NestJS 公式のサンプルを参考に投稿情報を扱う `application-posts` と ユーザ情報を扱う `application-users` を作成しました。

```sh
cd gateway-logging-example

# パッケージマネージャは全て yarn を選択してください。
nest new application-posts
nest new application-users
nest new gateway
```

各ディレクトリで必要なパッケージを追加しておきます。

:::details パッケージインストール
`cd` のパスは適宜書き換えてください。

```sh
cd application-posts
```

```sh
cd application-posts
```

```sh
cd gateway
```

:::

# ゲートウェイでリクエスト ID のヘッダを追加する

はじめに各アプリケーションに送信するリクエスト ID をゲートウェイに設定します。
Apollo Server の context でリクエスト ID を生成し、`RemoteGraphQLDataSource.willSendRequest` で生成された ID をヘッダに追加します。ヘッダは `x-request-id` とします。

# 各アプリケーションのログにリクエスト ID を追加する

次に 2 つのアプリケーションでログ出力できるようにします。
Apollo Server では `ApolloServerPlugin` を使ってロギングを行えます。

## Posts アプリケーション

## Users アプリケーション

# 動作確認

ゲートウェイ・アプリケーションの設定ができたので実際に動かしてログを確認してみます。

:::details ゲートウェイ・アプリケーションの起動
`cd` のパスは適宜書き換えてください。

```sh
cd application-posts && yarn start
```

```sh
cd application-posts && yarn start
```

```sh
cd gateway && yarn start
```

:::

# ex1: Rails のログにリクエスト ID を追加する

今回、NestJS のプロジェクトに加えて Rails アプリケーションにもリクエスト ID を反映させる対応をしていたのでおまけとして書いておきます。

Rails では `config.log_tags` を使うことでログにタグ付けが行えます。`config.log_tags = [:uuid]` とすることでログ 1 行に対して ID が付与されます。このとき、Rails では`x-request-id`のヘッダ情報が参照されるため、今回のようにゲートウェイで設定した ID を使うことができます。

:::details config/environments/development.rb

```rb:config/environments/development.rb
Rails.application.configure do
  # ...

  config.log_tags = [:uuid]

  # ...
end
```

:::

# ex2: Rails + lograge のログにリクエスト ID を追加する

lograge を使ってログ出力をしている場合は `config.lograge.custom_options` を使ってパラメータを追加することでログに ID を追加することができます。

Datadog と連携している場合は次の設定をすることで `request_id` によるフィルタリングが可能になります。

:::details config/environments/development.rb

```rb:config/environments/development.rb
Rails.application.configure do
  # ...

  config.lograge.enabled = true
  config.lograge.keep_original_rails_log = true
  config.lograge.logger = ActiveSupport::Logger.new "#{Rails.root}/log/lograge_#{Rails.env}.log"
  config.lograge.formatter = Lograge::Formatters::Json.new
  config.lograge.custom_options = lambda do |event|
    { request_id: event.payload[:request_id] }
  end

  # ...
end
```

:::

:::details app/controllers/application_controller.rb

```rb:app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  def append_info_to_payload(payload)
    super
    payload[:request_id] = request.headers['x-request-id']
  end
end
```

:::

# さいごに

今回はゲートウェイを使用したロギング改善の解説と、おまけとして Rails アプリケーションでの設定方法を紹介しました。

Apollo Gateway を使ったアプリケーションの運用をされている方、Apollo Gateway の使用を検討している方の参考になれば幸いです。

本記事で作成したサンプルプロジェクトは以下のリポジトリになります。

# 参考

- [API Reference: @apollo/gateway - Apollo GraphQL Docs](https://www.apollographql.com/docs/apollo-server/using-federation/api/apollo-gateway/#willsendrequest)
- [Rails のログに含まれる request_id についてコードリーディングしたメモ](https://zenn.dev/bisque/scraps/e0c58eb6fd07fa)
- [Rails の logger 周りのコードリーディング](https://blog.freedom-man.com/rails-logger-codereading)
- [roidrage/lograge](https://github.com/roidrage/lograge)

# 宣伝

スペースマーケットでは一緒に働く仲間を募集しています。
サービス内容や、使用技術に興味を持たれた方は是非ご応募ください！！

https://www.wantedly.com/projects/1113570
https://www.wantedly.com/projects/1113544
https://www.wantedly.com/projects/1061116

カルチャーの概要や使用技術が知りたい方はこちら ↓↓

https://spacemarket.co.jp/recruit/engineer/
