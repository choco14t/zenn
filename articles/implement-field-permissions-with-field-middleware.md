---
title: 'FieldMiddleware を使って Field Level Permissions を実装する'
emoji: '🐈‍⬛'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['NestJS', 'Apollo', 'GraphQL', 'TypeScript']
published: false
---

:::message
本記事は Code First を使用したプロジェクトにのみ適用できます。
:::

# はじめに

GraphQL で API を実装していて、ある Type のフィールドに対してアクセス制御を適用したいといったケースがあると思います。例えば、ログインユーザはアクセスできる、特定の権限を持つユーザのみアクセスできるといったものが挙げられます。

上記のようなフィールドに対する制御を NestJS では FieldMiddleware を使うことで実現することができます。

:::message
TODO: GraphQL に関する詳細の説明はしないことを明記する。
TODO: サンプルリポジトリのリンクを貼る
:::

# FieldMiddleware について

FieldMiddleware は名前のとおりフィールドを解決する前後に処理を追加するための機能を提供します。

FieldMiddleware は関数で定義し、`MiddlewareContext` と `NextFn` 型の 2 つの引数を受け取ります。 `MiddlewareContext` は Nest が定義している型ですが、渡される値は GraphQL Resolver が受け取る値（`{ source, args, context, info }`）と変わりありません。値の詳細については [GraphQL 公式ドキュメント](https://graphql.org/learn/execution/#root-fields-resolvers) を参照してください。

注意点として、FieldMiddleware は DI コンテナにアクセスすることができません。もし外部 API の実行や DB へのアクセスを行いたい場合は `MiddlewareContext` から受け取れるようにする必要があります。

# context と組み合わせた実装例

それでは実際に FieldMiddleware 実装していきます。以下の内容を例として進めていきます。

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

次に Type の定義を行います。今回はアクセス制限のある `email` と常にアクセスできる `name` を定義します。

```typescript:src/user/user.type.ts
import { Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class User {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;

  @Field()
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

仮実装として、固定値を返す `Query.user` を定義します。

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

次にログインしているユーザを保持するために context の設定を行います。context は GraphQLModule に定義することができます。

# さいごに

# 参考

- [Field middleware | NestJS](https://docs.nestjs.com/graphql/field-middleware)
- [Extensions | NestJS](https://docs.nestjs.com/graphql/extensions)
