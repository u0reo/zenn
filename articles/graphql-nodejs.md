---
title: "Node.jsにおけるGraphQLスキーマ生成ライブラリ"
emoji: "🔍"
type: "tech"
topics: ["nodejs", "graphql"]
published: true
---

Node.jsにおけるGraphQLを実装する選択肢はたくさんあるものの、この記事では以下のようにレイヤーを分けた時

- GraphQLスキーマ
- GraphQLサーバー(ex: Apollo Server, GraphQL Yoga)
- WEBサーバー(ex: express, fastify)

のうちGraphQLスキーマにあたる部分のライブラリ選定について、分かったことを記していきます。

# 先に結論

## pothosがおすすめ

✓ Prismaを採用している場合(相性抜群)  
✓ コードファーストで実装する場合  
✓ 立ち上げスピードを重視する場合(コード量少)  
✓ 型が好き、入力補完が欲しい場合(Prisma経由で強力な補完)  
△ 本番採用する場合(個人リポジトリなため)  
✕ 実装をレイヤーごとにきちんと分けたい場合  
✕ 拡張性を重視する場合  

## TypeGraphQLがおすすめ

✓ GraphQLを以前から使用している場合(スキーマとの強い関係性)
✓ NestJSなどで使われるデコレータ記法になじみがある人
✓ ドキュメントなどを多く求める場合(古くから存在しているため)
✓ レイヤーを分けて開発したい場合
✕ 入力補完を期待する場合
✕ 立ち上げスピードを重視する場合(コード量は比較すると多め)

どれを選んだとしても、多くの場合でGraphQLサーバーやWEBサーバーを乗り換えることができます。けれども、開発の大きな部分を占めるのでしっかり吟味したいところです。

-----

# 謝辞

今回はサイバーエージェントの[次世代トップエンジニア創出インターンシップACE](https://www.cyberagent.co.jp/careers/students/event/detail/id=28690)に参加いたしました。そこで得た知見となります。バックエンドを共にした[ずーまさん](https://twitter.com/azuma_alvin)やメンターの皆様をはじめアドバイスありがとうございました。

# 前提条件

筆者はGraphQL初心者です。**JSONっぽい長い文**を送ると好きな部分だけリクエストできるという程度の前提知識しかありません。

GraphQLサーバーやWEBサーバーについてもしっかりと選定したいところでしたが、Apollo Serverが一強かつWEBサーバーも選べるということだったため、ひとまずApolloを採用しました。WEBサーバーも初めはデフォルトのExpressを使用していました。

# GraphQLスキーマをどうするか

まず、**JSONっぽい長い文**というのが「GraphQLスキーマ」であり、GraphQLではスキーマ駆動の開発とコードからスキーマの自動生成と選べることがわかりました。**GraphQLが初見の自分にとっては圧倒的に前者の方が楽**ですが、GraphQLならではのところもあるので、少し調査しました。

## スキーマファーストのメリデメ

**〇 スキーマをフロントエンドとバックエンドの双方で協力して定義できるため、お互いの理解度が同じ状態で開発を進めやすい**  
〇 スキーマがAPIドキュメントに近いため、いつでも必要な機能が何かを再確認しやすい。  
〇 必要なスキーマのみ定義することに意識が向きやすく、リクエストの最適化がしやすい。  
**× チーム全員に対してGraphQLへの理解が求められる。**  
**× スキーマそのものの定義に時間がかかってしまう可能性がある。**  
× 要件自体に変更があったり、スキーマ定義にミスがあったりした場合、再定義に時間がかかってしまう上に影響範囲が広くなってしまう。(特にバックエンドはデータベースの再定義に及ぶ場合も)  

## コードファーストのメリデメ

**〇 GraphQLの知識が浅くても、試行錯誤で実装がどうにかなりそう。**  
**〇 DBの型定義をそのままGraphQLのスキーマへと転用できる**  
〇 コードで実装するため、IDEにおける入力補完や型解決を行うことができる。  
△ スキーマは自動生成されたものになるため、フロントエンド側がそれぞれ理解する必要がある。(ライブラリで一部はスキップすることも可能)  
**× スキーマを定義する際にDB(レポジトリ層)の設計が必要になるため、フロントエンドへスキーマを渡せるのが遅くなってしまう。**  
× バックエンド側でどのようなスキーマ(クエリ等)が必要かを理解して実装する必要がある。  
× フロントのスキーマ変更要求に対する対応に時間がかかる場合がある。  

以上のように、メリットデメリットが多くありますが、

- 全員がGraphQLに詳しいわけではなかった
- 開発目標の1つとしてとにかくパフォーマンスを追い求めるというのがあった
- ~~開発を楽にしたかった~~

の理由で、コードファーストを採用しました。

# GraphQLスキーマ生成ライブラリの選定

コードファーストと決定したはいいものの、今度はスキーマ生成のライブラリもいくつかありました。(このカオス何とかならないものか…)

Apollo Serverの実装は以下のようになっているため、

```ts
const apollo = new ApolloServer({
  schema: buildSchema(),
  ...
});
```

graphql公式ライブラリのスキーマを生成できればOKです。色々調べた結果、

- TypeGraphQL
- Nexus
- Pothos

の3つが候補にあがりました。

(Google Trendsは正しく比較できなかったため、除外)

https://npmtrends.com/@pothos/core-vs-nexus-vs-type-graphql

https://tmokmss.hatenablog.com/entry/20230109/1673237629

正しい比較は上記のページにあるため割愛しつつ、以下は主にここ1年での変化をピックアップします。

## TypeGraphQL

https://github.com/MichalLytek/type-graphql

こちらは個人プロジェクトですが、スポンサーがついているライブラリで、この中でも一番シェアは大きいです。v2のベータ版の開発を進めており、執筆時点では2023年8月にbeta.3をリリースしています。

https://github.com/MichalLytek/type-graphql/releases

デコレータ記法を採用しているため、NestJSと雰囲気は似ている感じでしょうか。

```ts
@ObjectType()
class Recipe {
  @Field(type => ID)
  id: string;

  @Field()
  title: string;

  @Field(type => [Rate])
  ratings: Rate[];

  @Field({ nullable: true })
  averageRating?: number;
}

@Resolver(Recipe)
class RecipeResolver {
  // dependency injection
  constructor(private recipeService: RecipeService) {}

  @Query(returns => [Recipe])
  recipes() {
    return this.recipeService.findAll();
  }

  @Mutation()
  @Authorized(Roles.Admin) // auth guard
  removeRecipe(@Arg("id") id: string): boolean {
    return this.recipeService.removeById(id);
  }

  @FieldResolver()
  averageRating(@Root() recipe: Recipe) {
    return recipe.ratings.reduce((a, b) => a + b, 0) / recipe.ratings.length;
  }
}
```

## Nexus

https://github.com/graphql-nexus/nexus

こちらはチームのプロジェクトであり、Prismaの開発陣が3人のうち2人を占めているからか、記述もデコレータを使わないものになっています。

```ts
import { queryType, stringArg, makeSchema } from 'nexus'
import { GraphQLServer } from 'graphql-yoga'

const Query = queryType({
  definition(t) {
    t.string('hello', {
      args: { name: stringArg() },
      resolve: (parent, { name }) => `Hello ${name || 'World'}!`,
    })
  },
})

const schema = makeSchema({
  types: [Query],
  outputs: {
    schema: __dirname + '/generated/schema.graphql',
    typegen: __dirname + '/generated/typings.ts',
  },
})

const server = new GraphQLServer({
  schema,
})

server.start(() => `Server is running on http://localhost:4000`)
```

しかし、昨年の5月からリリースが止まっており、新規のIssueすら大きく減っている状況です。ここ1年でも3コミットしかないようです。

https://github.com/graphql-nexus/nexus/releases

よって、先述のブログので言及されていることを引きづっていることとなります。

> ただし nexus-prisma も若干雲行きが怪しく、2022半ば頃は開発が停滞していたようです。そんな中、2022/10からメンテナンスの主体がPrismaからコミュニティに移管されつつあります。移管先は現状開発者一人のため開発速度が大きくブーストされるかは不透明ですが、今は過渡期のため当面状況を見守る必要があるでしょう。

> Nexus自体も生存確認のIssueが立つ程度には開発が活発ではありません。後述のPothosに移行するためのツールを作る開発者もいるなど、穏やかではない状況です。また、メインのメンテナであるtgriesserさんが後述のPothosに感心し、NexusからPothosへの漸進的な移行方法を提供したいとも発言しています。

## Pothos

https://github.com/hayes/pothos

初版のリリースがほかふたつのライブラリに比べ大きく違い、最も若いライブラリです。記述はNexusに非常によく似ているようです。

```ts
import { createYoga } from 'graphql-yoga';
import { createServer } from 'node:http';
import SchemaBuilder from '@pothos/core';

const builder = new SchemaBuilder({});

builder.queryType({
  fields: (t) => ({
    hello: t.string({
      args: {
        name: t.arg.string(),
      },
      resolve: (parent, { name }) => `hello, ${name || 'World'}`,
    }),
  }),
});

const yoga = createYoga({
  schema: builder.toSchema(),
});

const server = createServer(yoga);

server.listen(3000);
```

Prismaプラグインが用意されており、強力な連携をすることが可能な様です。非常に多機能なライブラリとなっており、ユースケースによってはこのプラグインを全て頼って完成することもできそうです。

https://pothos-graphql.dev/docs/plugins/prisma

こちらも個人プロジェクトなのがやや不安ですが、規模はわからないもののスポンサーもついているようですし、今のところ次のメジャーバージョンへ向けて開発を進めているようです。

https://github.com/hayes/pothos/pull/855


このようにライブラリを少しだけ比較しましたが、実質PothosがNexusの後継とみなせるでしょうから、古参のTypeGraphQLか新参のPothosということになります。

しかし、今回は

- デコレータ記法への強い抵抗感
- Node.jsでのORMライブラリはPrisma一択
- GraphQL初心者

ということと、当時のNPMが以下のようであったため、今後も伸びていくと考えたPothosを採用して開発を進めていくことを決定しました。

![2023/9/7時点でのNPMトレンド](/images/graphql-nodejs_npmtrends_20230907.png)

しかし、記事執筆時点ではこの勢いは戻ってしまったため、1年経った2023年11月現在もライブラリを取り巻く環境は上記ブログと大きく変化していないことになります。

# 乱立状態から変化していない？

このようにライブラリが乱立していながらも1年もあって環境が変化していないのは少し疑問です。これには技術選定とは別の要因がある気がしています。そこで別記事でGraphQL自体についてもう少し考察したいと思います。

# サンプルコード

決定する際に肌感を知るため、一度書いてみたコードを紹介します。

## リポジトリ

```ts
import { Prisma, PrismaClient } from '@prisma/client';

export const gameRepository = (prisma: PrismaClient) =>
  (query: {
    include?: Prisma.gameInclude;
    where?: Prisma.gameWhereInput;
  }) => {
    const db = prisma.game;

    return {
      getGameById: async (id: string) => {
        return await db.findUnique({
          ...query,
          where: { id },
        });
      },
      getGamesByDates: async (
        tournament_id: string,
        start_at_gte?: Date,
        start_at_lte?: Date,
      ) => {
        return await db.findMany({
          ...query,
          where: {
            tournament_id: tournament_id,
            start_at: {
              gte: start_at_gte,
              lte: start_at_lte,
            },
          },
        });
      },
    };
  };
```

## スキーマ&リゾルバ

```ts
import { builder } from '@/builder';

export const gameSchema = (builder: builder) => {
  builder.prismaObject('game', {
    name: 'Game',
    fields: (t) => ({
      id: t.exposeID('id'),
      tournament: t.relation('tournament'),
      metadata: t.expose('metadata', { type: 'JSONObject' }),
      start_at: t.expose('start_at', { type: 'DateTime' }),
      end_at: t.expose('end_at', { type: 'DateTime' }),
      contestant_first: t.relation('contestant_first'),
      contestant_second: t.relation('contestant_second'),
    }),
  });

  builder.queryFields((t) => ({
    game: t.prismaField({
      type: 'game',
      nullable: true,
      args: {
        id: t.arg.id({ required: true }),
      },
      resolve: async (query, _root, args, ctx) =>{
        return await ctx.repos.game(query).getGameById(String(args.id));
      },
    }),

    games: t.prismaConnection({
      type: 'game',
      cursor: 'id',
      args: {
        tournament_id: t.arg.string({ required: true }),
        start_date: t.arg.string(),
        end_date: t.arg.string(),
      },
      resolve: async (query, _root, args, ctx) => {
        const start_at_gte = args.start_date ? new Date(args.start_date) : undefined;
        const start_at_lte = args.end_date ? new Date(args.end_date) : undefined;

        return await ctx.repos.game(query).getGamesByDates(
          args.tournament_id,
          start_at_gte,
          start_at_lte,
        );
      },
    }, {}, {}),
  }));

  builder.mutationFields((t) => ({
    createGame: t.prismaField({
      type: 'game',
      resolve: (_query, _root, args, ctx) =>
        ctx.prisma.game.create({
          data: {} as never,
        }),
    }),
  }));
};

export default gameSchema;
```

## 単体テスト

テストに使用した追加コードは別記事で紹介予定。

```ts
import { GraphQLID, GraphQLNonNull } from 'graphql';
import { GraphQLDateTime, GraphQLJSONObject } from 'graphql-scalars';
import { game } from '@prisma/client';
import { getMockSchemaType, omitPothos, testConvertFields } from '@/utils';

const dbData: game = {
  id: 'game_id',
  tournament_id: 'tournament_id',
  metadata: {
    group_name: 'グループA',
  },
  start_at: new Date('2023-09-06T10:00:00.000Z'),
  end_at: new Date('2023-09-06T14:00:00.000Z'),
  contestant_first_id: 'contestant_a',
  contestant_second_id: 'contestant_b',
  updated_at: new Date(),
  created_at: new Date(),
};

// GraphQL Schema
const fields = getMockSchemaType('Query', 'game', () => dbData).getFields();

describe('Game Schema', () => {
  test('Check "id" Field Type', () => {
    expect(fields).toHaveProperty('id');
    expect(fields.id.type).toEqual(new GraphQLNonNull(GraphQLID));
  });

  test('Check "metadata" Field Type', () => {
    expect(fields).toHaveProperty('metadata');
    expect(omitPothos(fields.metadata.type)).toEqual(new GraphQLNonNull(GraphQLJSONObject));
  });

  test('Check "start_at" Field Type', () => {
    expect(fields).toHaveProperty('start_at');
    expect(omitPothos(fields.start_at.type)).toEqual(new GraphQLNonNull(GraphQLDateTime));
  });

  test('Check "end_at" Field Type', () => {
    expect(fields).toHaveProperty('end_at');
    expect(omitPothos(fields.end_at.type)).toEqual(new GraphQLNonNull(GraphQLDateTime));
  });

  test('Check Logic', () => {
    const parsed = testConvertFields('game', dbData);

    expect(parsed).toEqual({
      id: 'game_id',
      metadata: {
        group_name: 'グループA',
      },
      start_at: new Date('2023-09-06T10:00:00.000Z'),
      end_at: new Date('2023-09-06T14:00:00.000Z'),
    });
  });
});
```