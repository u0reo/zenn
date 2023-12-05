---
title: "GraphQLにおけるキャッシュ戦略"
emoji: "🗄️"
type: "tech"
topics: ["typescript", "nodejs", "graphql", "apollo", "cache"]
published: true
---

# 課題

GraphQLでAPIを実装したはいいものの、さらなる高速化をするためキャッシュをしたい…

→ 超難しい

そもそも、Webの世界のキャッシュはREST APIを念頭に行われています。そのうえで以下の3つのレイヤーに分けて、キャッシュを考えます。今回選定した戦略も以下に示します。

1. HTTP
  a. ブラウザ → Cache-Controlに基づくキャッシュ
  b. CDN → Cache-Controlに基づくキャッシュ
2. Webサーバー
  a. fastify → 設定しない
3. GraphQL
  a. Apollo Server → Cache-Controlの設定と、それに基づくキャッシュ&ステータスコードの設定
  b. Pothosr → Cache-Controlの動的設定
4. DB
  a. Prisma → なし
  b. MySQL / PostgreSQL など → 設定しない

-----

# 謝辞

今回はサイバーエージェントの[次世代トップエンジニア創出インターンシップACE](https://www.cyberagent.co.jp/careers/students/event/detail/id=28690)に参加いたしました。そこで得た知見となります。バックエンドを共にした[ずーまさん](https://twitter.com/azuma_alvin)やメンターの皆様をはじめアドバイスありがとうございました。


# ブラウザやCDN、fastifyによるキャッシュ

Webの世界でキャッシュというと、まずこれかと思います。フレームワークでは、あらかじめ設定されていたり、そうでなくてもCDNをかますだけで適用できたりします。

ここで大事になってくるのが、**Cache-Control**と呼ばれるHTTPヘッダーです。ペイロードに合わせて、適切なCache-Controlを設定することで、キャッシュさせる範囲などを設定できます。

https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Cache-Control

* no-cache: キャッシュしてもいいけど、サーバーに変更ないか問い合わせ必要
* no-store: キャッシュ禁止
* private: ブラウザのみキャッシュ可能。(不特定多数に配信するキャッシュは禁止)
* public: 常にキャッシュOK

これらのヘッダーは**リクエスト単位**で設定するため、その分パスが違う必要があります。

また、HTTPのメソッドはそれぞれ意味があります。

https://developer.mozilla.org/ja/docs/Web/HTTP/Methods

そして、何か変更を加えるGET以外のリクエストはキャッシュされません。

## REST APIの場合

例えば、ブラウザがCDN経由でサーバーにアクセスする一般的なシチュエーションの場合、

* /user/info では Cache-Control: private → ブラウザのみキャッシュ
* /user/edit では Cache-Control: no-store → キャッシュしない
* /public では Cache-Control: public → ブラウザとCDNでキャッシュ

のように使い分けられます。

## GraphQLの場合

一方、GraphQLでは基本的に

* エンドポイントは単一
* 全てPOSTリクエスト

なため、キャッシュはされません。

また、リクエストの内容はPOSTのボディでまとめて指定するため、キャッシュと非常に相性が良くありません。REST APIの例だと

* /user/info + /user/edit + /public
* /user/info + /user/edit
* /user/info + /public
* /user/edit + /public
* /user/info
* /user/edit
* /public

の8種類あり(= 2³ - 1)、その分のキャッシュが必要です。一方、REST APIでは3種類で済みます。APIのバリエーションが増えるだけ、指数関数的にキャッシュも必要になります。この問題は解決できないため、フロントエンドの設計を踏まえて、APIを事前に叩いてキャッシュさせておくなどの対策をするしかありません。

### さらなる追加対応

一方、GraphQLはエラーでもHTTPステータスコードは常に200 OKとなってしまいますが、これはCDNなどによってはエラー結果もキャッシュされてしまうことになります。これを避けるため、以下のようにエラーをキャッチした場合、ステータスコードを書き換えます。

https://www.apollographql.com/docs/apollo-server/data/errors/#setting-http-status-code-and-headers

```ts
import { ApolloServer, ApolloServerPlugin } from '@apollo/server';
import Fastify from 'fastify';
import createContext, { Context } from './context';
import getBuilder from './builder';
import { GraphQLCustomErrorCode } from './utils/CustomGraphQLError';

const ThrowHttpErrorPlugin: ApolloServerPlugin<Context> = {
  async requestDidStart() {
    return {
      async willSendResponse({ response }) {
        if (response.body.kind !== 'single' || !response.body.singleResult.errors) {
          return;
        }
        for (const error of response.body.singleResult.errors) {
          const code = error.extensions?.code as GraphQLCustomErrorCode | 'UNKNOWN';
          switch (code) {
            case 'DATA_CONFLICT':
              response.http.status = 409;
              return;
            case 'DATA_NOT_FOUND':
              response.http.status = 400;
              return;
            default:
              return;
          }
        }
      },
    };
  },
};

export const makeFastifyServer = async () => {
  const app = Fastify();

  const apollo = new ApolloServer<Context>({
    schema: getBuilder().toSchema(),
    plugins: [
      ThrowHttpErrorPlugin,
      ...
    ],
  });
  ...
};
```

```ts:CustomGraphQLError.ts
import { GraphQLError } from 'graphql';

export const customErrorMessage = {
  DATA_NOT_FOUND: 'Not Found',
  DATA_CONFLICT: 'Conflict',
} as const;

export type GraphQLCustomErrorCode = keyof typeof customErrorMessage;
```

このよう設定したうえで、リゾルバ内で以下のようにエラーをスローするとステータスコードも変化します。

```ts
throw new GraphQLError('Error Message', {
  extensions: {
    code,
  },
});
```

# fastifyによるキャッシュ

ほぼ仕組みはCDNによるキャッシュと同じですが、サーバー側で制御できるのが大きなメリットです。その一方で、CDNと二重にかませる必要はないので、サーバーのスペックを圧迫しないCDNによるキャッシュが手軽で最適です。

https://www.npmjs.com/package/@fastify/caching

# Apollo ServerやPothosによるキャッシュ

前述のようにリクエストデータ単位でのキャッシュはHTTPで行いにくいため、その分データを生成するGraphQLサーバー関連のライブラリでのキャッシュが大事となります。

https://dev.classmethod.jp/articles/graphql-apollo-server-cache-control/

基本的にはこのサイトの動的制御のようにキャッシュを設定すれば、適切にCache-Controlヘッダーが設定されるため、問題ありません。

```ts
builder.queryFields((t) => ({
  query: t.field({
    type: 'Boolean',
    nullable: false,
    resolve: async (_root, args, ctx, info) => {
      info.cacheControl.setCacheHint({ maxAge: 0, scope: 'PRIVATE' });
      ...
      return data;
    },
  }),
  ...
});
```

しかし、pothosではリレーション周りの設定が悪いのか、キャッシュが全然効きませんでした。

> このようにレスポンス全体のフィールドで最も最小のmaxAgeとscopeが設定されます。 フィールドはtypeの`@cacheControl`を継承しますが、type自体に `@cacheControl`が設定されていない場合は`@cacheControl(maxAge: 0)`と同じ扱いになることに注意してください。

このようにtypeにキャッシュ設定がなされていなかったのかもしれませんが、簡便のため以下のデフォルトキャッシュ時間を設定することをお勧めします。

```ts:server.ts
const apollo = new ApolloServer<Context>({
  schema: getBuilder().toSchema(),
  plugins: [
    fastifyApolloDrainPlugin(app),
    ApolloServerPluginCacheControl({
      defaultMaxAge: 86400, // 24時間
    }),
    ...
  ],
  ...
});
```

これでCache-Controlヘッダーは適切に設定されるようになります。

また、以下の24行目のようにregisterでApollo Serverをfastifyに登録すればGETリクエストも受け付けるため、リクエスト時にGETリクエストを使うようにすればキャッシュ自体が動作します。

```ts
import { ApolloServer } from '@apollo/server';
import { ApolloServerPluginCacheControl } from '@apollo/server/plugin/cacheControl';
import Fastify from 'fastify';
import fastifyApollo, { fastifyApolloDrainPlugin } from '@as-integrations/fastify';
import createContext, { Context } from './context';
import getBuilder from './builder';

...

const app = Fastify();
const apollo = new ApolloServer<Context>({
  schema: getBuilder().toSchema(),
  plugins: [
    fastifyApolloDrainPlugin(app),
    ApolloServerPluginCacheControl({
      defaultMaxAge: 86400, // 24時間
    }),
    ...
  ],
  ...
});
await apollo.start();
...
await app.register(fastifyApollo(apollo), {
  context: createContext,
});
```

**Apollo Clientの場合 (HttpLink → useGETForQueries)**

https://www.apollographql.com/docs/react/api/link/apollo-link-http/#usegetforqueries

**urqlの場合 (Client → preferGetMethod)**

https://formidable.com/open-source/urql/docs/api/core/

## responseCachePluginの活用

更にこのプラグインを設定することで、サーバー内にキャッシュをもたせることができます。

```ts
import responseCachePlugin from '@apollo/server-plugin-response-cache';

const apollo = new ApolloServer<Context>({
  schema: getBuilder().toSchema(),
  plugins: [
    fastifyApolloDrainPlugin(app),
    ApolloServerPluginCacheControl({
      defaultMaxAge: 86400, // 24時間
    }),
    responseCachePlugin(),
    ...
  ],
  ...
});
```

これにより、初回アクセスでCDNがなくてもキャッシュにより素早くデータが返ってきます。サーバーを複数持たせる場合はRedisやMemcachedの使用もできます。

https://www.apollographql.com/docs/apollo-server/performance/caching/#caching-with-responsecacheplugin-advanced

## Persisted Queriesについて

ここまで設定をしても、まだ課題は残っています。GETリクエストでGraphQLを叩く場合、本来POSTのボディに乗せるデータをGETのクエリ文字列として載せることになります。しかし、URLの最大長は2048文字あたりなため、それ以降はPOSTリクエストに切り替わりキャッシュが効かなくなってしまいます。

そこで、あらかじめクエリ対応するIDをサーバーに発行してもらい、そのIDでリクエストを送るのがPersisted Queriesです。どうしてもキャッシュに乗せたいけどクエリが大きすぎるという現状の妥協案と言えるでしょう。

今回はあまりクエリが複雑になる見込みがなかったうえ、あらかじめ登録されていなかったクエリの場合、リクエストが1つ増えてしまうため、採用を見送りました。以下のサイトがとても参考になるかと思います。

https://engineering.mercari.com/blog/entry/20220303-concerns-with-using-graphql/

https://www.apollographql.com/docs/apollo-server/performance/apq

https://www.apollographql.com/docs/react/api/link/persisted-queries/

# PrismaやDBによるキャッシュ

一番実データに近いレイヤーであるため、キャッシュ方法は限られます。

Prismaは調べた限りではキャッシュプラグインは存在しませんでした。

一方、DBによるクエリキャッシュは存在します。

https://qiita.com/ryurock/items/9f561e486bfba4221747

しかし、正しくインデックスを張ればほぼデータ取得に時間がかからない上、キャッシュレイヤーが増えすぎてしまうとキャッシュ管理が煩雑になってしまうため、今回は実装しないことにしました。

# 最後に

今回は

1. HTTP
  a. ブラウザ → Cache-Controlに基づくキャッシュ
  b. CDN → Cache-Controlに基づくキャッシュ
2. GraphQL
  a. Apollo Server → Cache-Controlの設定と、それに基づくキャッシュ&ステータスコードの設定
  b. Pothosr → Cache-Controlの動的設定

4つのレイヤーの中で3つのキャッシュレイヤーと、2つのキャッシュ設定レイヤーを設定しました。

これ以上増えてしまうと管理も煩雑になるうえに、大きくレイテンシが改善されたので対策としては十分かと考えられます。キャッシュしてはいけないクエリにPRIVATEと設定し忘れないようにチェックし、Cache-Controlが正しくセットされているかを別途テストなどで検証することが大事がになりそうです。
