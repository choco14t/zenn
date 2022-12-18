---
title: 'データアクセスレイヤに対するテストの考察'
emoji: '🦁‍'
type: 'idea' # tech: 技術記事 / idea: アイデア
topics: ['test']
published: true
published_at: 2022-12-01 12:00
---

https://twitter.com/t_wada/status/1595920886926036992

上記のツイート見かけたので引用されていた記事の内容を読んだ上で、データアクセスレイヤに対するテストについて自分なりの考えを書いてみました。

データアクセスレイヤは ActiveRecord のような ORM だったり、リポジトリパターンを用いたようなクラスだったりと各々のプロジェクトによって異なると思います。

データアクセスはアプリケーション独自の実装になるので、自分はモックせずにテストを書く意味があると思っています。例えば ID からレコードを取得する、指定された条件のレコードのみ取得するといった実装に対するテストは、その条件を満たすクエリを正しく発行しているといったことが保証できるので実装に対する安心感が増します。

対して、データアクセスレイヤのテストで DB をモックすると条件に合ったクエリが発行されているかなどは無視されてしまい、条件を満たしているであろう値を返すことでテストがとりあえずパスしてしまいます。DB のモックはインメモリ上でデータを保持したり、テキストファイルを使用したりなどが挙げられます。モックはコードの実行結果を書き手が指定するもので、テストはコードの実行結果を評価するものと解釈すると自作自演のテストと言われていることが腑に落ちました。

どこからモックするかは規模やチームによっても異なるので解が 1 つに定まることはないと思います。データアクセスレイヤに対するテストが書けている場合、依存している別レイヤではモックしてしまっても良いというのが個人的な意見です。別のレイヤではそのレイヤでの実装に対してテストを行いたいはずだからです。（例外として e2e テストは外部 API 以外は基本モックせずテストする方が良いと現状は考えています。）