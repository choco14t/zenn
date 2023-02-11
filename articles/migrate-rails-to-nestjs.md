---
title: 'RailsからNestJSへの移行に挑戦してみて'
emoji: '🐈‍⬛'
type: 'idea'
topics: ['nestjs', 'apollo', 'graphql', '振り返り']
published: false
---

# はじめに

[スペースマーケット](https://www.spacemarket.com/)でバックエンドエンジニアをしている choco です。

直近の主な業務として、Rails で実装されている GraphQL API を NestJS に移行していました。
まだ移行完了までは到達できていませんが、どういうことをしてきたか棚卸しの意味も込めて書き出してみました。

Rails・Node.js・GraphQL すべて業務ではスペースマーケットに入社してから使い始めた技術にもかかわらず、チャレンジングな機会を提供してくれたことに感謝しています。また、レビューや相談に協力してくれた方々もありがとうございました。

# エンドポイント移行

大体の作業時間はエンドポイント移行を行っていました。Mutation はデータ更新や非同期ジョブを含むものが多いので Query の移行から着手し、その中でも利用目的や会場タイプと呼ばれるマスタデータ系統の Query から着手し始めて実装に慣れていきました。

マスタデータの移行が完了した後はよく使われていそうな機能のエンドポイント移行にも着手しました。外部 API との連携やメール通知処理が必要な Mutation については別の API に実装がある都合上、Message Queue を使うことで連携を図りました。

# パフォーマンス改善

マスタデータの移行が無事完了して一区切りついたのも束の間、特定の時間帯になると NestJS アプリケーションのアラートが通知されるようになりました。発生当初は何故かわからなかったのですが、実装や過去の PR、ドキュメントを読み直してみるとどうやら Guard の機能と Federation の相性が悪いらしいという仮説が立てられました。NestJS のドキュメントには以下の記載があります。

> Enabling enhancers for field resolvers can cause performance issues when you are returning lots of records and your field resolver is executed thousands of times. For this reason, when you enable fieldResolverEnhancers, we advise you to skip execution of enhancers that are not strictly necessary for your field resolvers. You can do this using the following helper function
> https://docs.nestjs.com/graphql/other-features より引用

改めてコードを確認すると `fieldResolverEnhancers` の設定がされていることがわかりました。実装当初、Guard を使い Guard 内で DB アクセスすることで認証情報を取得していました。また、Guard をグローバルに設定していたことで移行したマスタデータ取得時にも Guard の処理が実行されていることが判明しました。この問題については GraphQL の context 内で認証情報を取得するように実装を修正することで回避しました。

:::message
補足として、今回は Guard の使い方が適切でなかったことが問題であり、Guard そのものについて問題があったわけではありません。
:::

# NestJS に contribute

次に、別チームの施策であるページのパフォーマンスを向上させるために一部データのキャッシュを行いたいという要望が出ました。Server-side caching できるか検証し始めたところ、Apollo のドキュメント通りに記述しても動作しませんでした。NestJS のプロジェクトでは Code First を採用して実装をしていたのですが、試しに Schema First に切り替えて実行してみると問題なく動作することがわかり、フレームワークのバグを踏んだ...となりました。

ひとまず原因がわかったので issue を作成しました。しかし、修正されるのにも時間がかかりそうだったこと、自分で PR が作成できる内容かもしれないと思ったことで修正に取り組み始めました。結果として動作する修正 PR を作成することができ無事 merge されました。OSS への contribute は初めてだったので非常に嬉しかったです。

https://github.com/nestjs/graphql/pull/2139

# Job Queue の検証

Mutation のエンドポイント移行時に非同期ジョブの実装が課題に挙がりました。Node.js の Job Queue は Redis を使用しているものがほとんどで永続化をどうするかが課題になりました。テックリードに相談したところ、MemoryDB というサービスが AWS にあるのでこれを使ってみたらどうかとなり検証を開始しました。

NestJS がライブラリを提供していることもあり Bull を採用して実装を始めたのですが、Cluster Mode の設定に苦戦してアクセス可能になるまでかなり時間がかかったのを覚えています（MemoryDB は Cluster Mode のみサポート）。Bull のドキュメントには Cluster Mode の設定方法が記載されているのですが、それでは不十分で Bull が依存している ioredis の issue などを読むことで動作する設定を行なうことができました。（このあたりも別記事で書けたらなと思います）

Job のやり取りができるところまでは確認できたのですが、別の事情で移行予定だった Job の処理が行えないことが判明したので本番運用までは持って行けていません...。実装で手一杯になり仕様の確認を怠っていたのは反省点です。

# Field Level Permission の実装

一部 Field ではアクセスユーザに応じて値を返すか制御するような実装が必要でした。NestJS には FieldMiddleware という機能が提供されているのでこの機能を使ってアクセス制御の実装を実現しました。FieldMiddleware の実装にあたり、GraphQL から認証情報や DataSource オブジェクトを取得したためパフォーマンス改善時に context を用いた実装に変更したことが活きました。

FieldMiddleware の実装サンプルは記事にもしているので良ければ読んでみてください。

https://zenn.dev/spacemarket/articles/implement-field-permissions-with-field-middleware

同機能を実現するライブラリとして graphql-shield があります。私が実装していた時点ではメンテナンスがされていなかったので採用を見送りましたが、現時点ではメンテナンスが再開されているので今後 GraphQL を使ってアクセス制御の実装を行おうとしている場合は検討しても良いかもしれません。and や or のような組み合わせが使えるため、FieldMiddleware より複雑な条件を実現できるはずです。

# さいごに

これまで私が取り組んだ移行業務を振り返ってみました。まだ道半ばな状態ではありますが、他のメンバーもチャプターと呼ばれる学習時間を使って移行に取り組んでくれているので徐々にエンドポイントが NestJS へ移行されていくことでしょう。

移行に取り組んだ際に感じた NestJS や関連ライブラリに対する辛さはやはりあったので、それについてもどこかで書けたらなと考えています。

# 宣伝

スペースマーケットでは一緒に働く仲間を募集しています。
サービス内容や、使用技術に興味を持たれた方はぜひご応募ください。

https://www.wantedly.com/projects/1113570
https://www.wantedly.com/projects/1113544
https://www.wantedly.com/projects/1061116

カルチャーの概要や使用技術が知りたい方はこちら。

https://spacemarket.co.jp/recruit/engineer/
