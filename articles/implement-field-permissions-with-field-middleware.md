---
title: '[NestJS] FieldMiddleware で Field Permissions を実装する'
emoji: '🐈‍⬛'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['NestJS', 'Apollo', 'GraphQL', 'TypeScript']
published: true
published_at: 2022-10-14 10:00
publication_name: 'spacemarket'
---

:::message
本記事は Code First を使用したプロジェクトにのみ適用できます。
:::

# はじめに

[スペースマーケット](https://www.spacemarket.com/)でバックエンドエンジニアをしている choco です。

GraphQL で API を実装していて、ある Type のフィールドに対してアクセス制御を適用したいといったケースがあると思います。例えば、ログインユーザはアクセスできる、特定の権限を持つユーザのみアクセスできるといったものが挙げられます。

上記のようなフィールドに対する制御を NestJS では FieldMiddleware を使うことで実現することができます。本記事では NestJS のプロジェクト作成から FieldMiddleware の適用までを解説します。

この記事で作成したコードは下記リポジトリから確認できます。
https://github.com/choco14t/field-middleware-example

# FieldMiddleware について

FieldMiddleware は名前のとおりフィールドを解決する前後に処理を追加するための機能を提供します。

FieldMiddleware は関数で定義し、`MiddlewareContext` と `NextFn` 型の 2 つの引数を受け取ります。 `MiddlewareContext` は Nest が定義している型ですが、渡される値は GraphQL Resolver が受け取る値（`{ source, args, context, info }`）と変わりありません。値の詳細については [GraphQL 公式ドキュメント](https://graphql.org/learn/execution/#root-fields-resolvers) を参照してください。

注意点として、FieldMiddleware は DI コンテナにアクセスすることができません。もし外部 API の実行や DB へのアクセスを行いたい場合は `MiddlewareContext` から受け取れるようにする必要があります。

# context と組み合わせた実装例

以下の内容を例として実装を進めていきます。

- id から単一のユーザを取得できる `Query.user` を定義する
- User は id、name、email のフィールドを持つ
- email は管理者権限を持つユーザのみがアクセスできる。それ以外のユーザが取得しようとした場合は null を返す

今回は下記を使用してプロジェクトを作成します。

- @nestjs/cli 8.2.6
- yarn 1.22.19
- Node.js 16.15.0

```shell
nest new field-middleware-example
```

また、[Quick start](https://docs.nestjs.com/graphql/quick-start) に倣って Express、Apollo を使用します。

```shell
yarn add @nestjs/graphql @nestjs/apollo graphql apollo-server-express
```

## GraphQLModule の import

まずはじめに `AppModule` に `GraphQLModule` を import します。

```typescript:src/app.module.ts
import { join } from 'path';
import { ApolloDriverConfig, ApolloDriver } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
      sortSchema: true,
    }),
  ],
})
export class AppModule {}
```

## UserModule の作成

この時点でアプリケーションを起動しようとすると、Query または Mutation の定義がされていない旨のエラーが発生します。まずアプリケーションが起動できるようにするために `UserModule` を作成していきます。

```shell
nest generate module user
# output
CREATE src/user/user.module.ts (81 bytes)
UPDATE src/app.module.ts (478 bytes)
```

次に Type の定義を行います。今回はアクセス制限のある `email` と常にアクセスできる `name` を定義します。この時点では `email`、`name` どちらも常にアクセスできます。

```typescript:src/user/user.type.ts
import { Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class User {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;

  @Field({ nullable: true })
  email: string;
}
```

次に Resolver を作成します。

```shell
nest generate resolver user
# output
CREATE src/user/user.resolver.spec.ts (456 bytes)
CREATE src/user/user.resolver.ts (86 bytes)
UPDATE src/user/user.module.ts (158 bytes)
```

記事内では値の確認ができれば良いので、固定値を返す `Query.user` を定義します。

```typescript:src/user/user.resolver.ts
import { Query, Resolver } from '@nestjs/graphql';
import { User } from './user.type';

@Resolver(() => User)
export class UserResolver {
  @Query(() => User)
  user() {
    return {
      id: '1',
      name: 'user 1',
      email: 'user_1@example.com',
    };
  }
}
```

ここでアプリケーションが正常に起動され、クエリが実行できるようになりました。

## context のセットアップ

次にログインしているユーザを保持するために context の設定を行います。context は GraphQLModule で定義することができます。

余談ですが、context が未定義の場合は使用している Apollo パッケージで予め定義されているオブジェクトが context として設定されます。NestJS の場合は以下のどちらかがデフォルトで設定されます。

| フレームワーク | context                                          |
| -------------- | ------------------------------------------------ |
| Express        | { req: express.Request, res: express.Response }  |
| Fastify        | { request: FastifyRequest, reply: FastifyReply } |

今回はリクエストの Authorization ヘッダーを確認し、アクセスしているユーザを判別するようにします。型やダミーデータはひとまず `AppModule` にベタ書きします。

```diff typescript:src/app.module.ts
 import { join } from 'path';
 import { ApolloDriverConfig, ApolloDriver } from '@nestjs/apollo';
 import { Module } from '@nestjs/common';
 import { GraphQLModule } from '@nestjs/graphql';

+ enum ViewerRole {
+   MEMBER,
+   ADMIN,
+ }
+
+ interface Viewer {
+   userName: string;
+   role: ViewerRole;
+ }
+
+ interface AppContext {
+   viewer: Viewer | undefined;
+ }
+
+ const users: Viewer[] = [
+   { userName: 'user_1', role: ViewerRole.MEMBER },
+   { userName: 'user_2', role: ViewerRole.ADMIN },
+ ];

 @Module({
   imports: [
     GraphQLModule.forRoot<ApolloDriverConfig>({
       driver: ApolloDriver,
       autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
       sortSchema: true,
+      context: ({ req }): AppContext => {
+        const token = req.headers.authorization || '';
+        const viewer = users.find((user) => user.userName === token);
+
+        return { viewer };
+      },
     }),
     UserModule,
   ],
 })
 export class AppModule {}
```

## FieldMiddleware の実装

管理者権限を持つかを判別する `hasAdminRole` を実装します。

`FieldMiddleware` は `FieldMiddleware<TSource, TContext, TArgs, TOutput>` の Generics で、引数はそれぞれ次の型情報と対応しています。

| 引数     | 対応する型情報                                                     |
| -------- | ------------------------------------------------------------------ |
| TSource  | 渡されたフィールドの ObjectType。                                  |
| TContext | GraphQL サーバで定義した context。本記事では `AppContext` となる。 |
| TArgs    | フィールドで使用される引数。引数がない場合は `{}` となる。         |
| TOutput  | FieldMiddleware が返却する値。                                     |

上記の引数と照らし合わせると、実装は下記のようになります。

```typescript:src/user/field-middlewares/has-admin-role.middleware.ts
import { FieldMiddleware } from '@nestjs/graphql';
import { AppContext, ViewerRole } from '../../app.module';
import { User } from '../user.type';

export const hasAdminRole: FieldMiddleware<User, AppContext> = async (
  ctx,
  next,
) => {
  const {
    context: { viewer },
  } = ctx;

  return viewer?.role === ViewerRole.ADMIN ? next() : null;
};
```

`hasAdminRole` 内で `TSource` の参照は行われていないですが、`User` に対して使用する Middleware なので明示しています。

## FieldMiddleware の適用

前項で実装した FieldMiddleware を `User.email` に適用します。`FieldOptions` の `middleware` プロパティに適用したい FieldMiddleware を配列で渡すことで動作します。

```diff typescript:src/user/user.type.ts
 import { Field, ID, ObjectType } from '@nestjs/graphql';
 import { hasAdminRole } from './field-middlewares/check-admin-role.middleware';

 @ObjectType()
 export class User {
   @Field(() => ID)
   id: string;

   @Field()
   name: string;

+  @Field({ nullable: true, middleware: [hasAdminRole] })
   email: string;
 }
```

## 動作確認

最後に FieldMiddleware が動作するか確認してみます。

```sh
# 管理者権限でないユーザでのリクエスト
curl -H 'Content-Type: application/json' -H 'Authorization: user_1' -d '{ "query": "query { user { email } }" }' http://localhost:3000/graphql
# output
{"data":{"user":{"email":null}}}

# 管理者権限を持つユーザでのリクエスト
curl -H 'Content-Type: application/json' -H 'Authorization: user_2' -d '{ "query": "query { user { email } }" }' http://localhost:3000/graphql
# output
{"data":{"user":{"email":"user_1@example.com"}}}
```

# さいごに

簡易的な FieldMiddleware を定義し、ObjectType に適用するまでを行いました。FieldMiddleware は複数適用させたり、グローバルに適用してアプリケーション全体に適用することもできるので興味がある方は試してみてください。

NestJS は Code First で実装する上で便利な機能が提供されています。しかし、FieldMiddleware に関しては [graphql-shield](https://www.the-guild.dev/graphql/shield) と比較すると `or` のようなルールのいずれかを満たすか判別する機能が提供されていないため、複雑なルールを適用したい場合は FieldMiddleware だと実現が難しくなると思います。

NestJS 上で graphql-shield を使うことも可能なので、プロジェクトに応じてライブラリを使い分けると良いかもしれません。

# 参考

- [Field middleware | NestJS](https://docs.nestjs.com/graphql/field-middleware)
- [Extensions | NestJS](https://docs.nestjs.com/graphql/extensions)
- [API Reference: ApolloServer - Apollo GraphQL Docs](https://www.apollographql.com/docs/apollo-server/api/apollo-server/#middleware-specific-context-fields)

# 宣伝

スペースマーケットでは一緒に働く仲間を募集しています。
サービス内容や、使用技術に興味を持たれた方は是非ご応募ください！！

https://www.wantedly.com/projects/1113570
https://www.wantedly.com/projects/1113544
https://www.wantedly.com/projects/1061116

カルチャーの概要や使用技術が知りたい方はこちら ↓↓

https://spacemarket.co.jp/recruit/engineer/
