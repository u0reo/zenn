---
title: "fastifyとsecure-sessionで簡単&高速なステートフル認証"
emoji: "🗝"
type: "tech"
topics: ["typescript", "nodejs", "fastify"]
published: true
---

今回の例ではApollo Serverを使っています。GraphQLを使っていない場合などでは、HooksでcreateContextの内容を実装してください。

https://fastify.dev/docs/latest/Reference/Hooks/

# 先に実装方法

```ts:server.ts
import { ApolloServer } from '@apollo/server';
import Fastify from 'fastify';
import csrfProtection from '@fastify/csrf-protection';
import secureSession from '@fastify/secure-session';
import fastifyApollo from '@as-integrations/fastify';

import createContext, { Context } from './context';
import getBuilder from './builder';
import { GraphQLCustomErrorCode } from './utils/CustomGraphQLError';

const nowDevelopment = NODE_ENV === 'development';

export const makeFastifyServer = async () => {
  const app = Fastify();

  const apollo = new ApolloServer<Context>({
    schema: getBuilder().toSchema(),
  });

  await app.register(secureSession, {
    secret: SESSION_SECRET,
    salt: SESSION_SALT,
    cookie: {
      domain: undefined, // ほかのドメインでクッキーを共用しないため未定義
      path: '/graphql', // graphqlのエンドポイントのみで使用
      httpOnly: true, // JavaScriptで操作できないように
      secure: !nowDevelopment, // HTTPS通信でのみクッキーを送信
      expires: undefined, // 有効期限日時は設けない(ブラウザを閉じたとき)
      maxAge: SESSION_MAX_AGE, // 有効時間(秒)
      sameSite: nowDevelopment ? 'strict' : 'none', // 他ドメイン間での遷移時に関係するセキュリティ設定、ドメインを跨ぐため一番弱いものに
      // options for setCookie, see https://github.com/fastify/fastify-cookie
    },
  });
  await app.register(csrfProtection, { sessionPlugin: '@fastify/secure-session' });

  await app.register(fastifyApollo(apollo), {
    context: createContext,
  });

  return app;
};
```

```ts:createContext.ts
import { ApolloFastifyContextFunction } from '@as-integrations/fastify';
import { prisma } from './prisma';
import { repositories } from './repository';

export interface Context {
  repos: ReturnType<typeof repositories>;
  userId: string;
}


const createContext: ApolloFastifyContextFunction<Context> = async (request) => {
  const repos = repositories(prisma);

  let userId: string | undefined | null = request.session.userId;

  // ユーザーの存在チェック
  if (userId) {
    const user = await repos.user.getUserById(userId);
    if (!user) {
      userId = null;
    }
  }

  // セッションが空だったり、セッションが古ければセッションの再作成(ユーザーの再作成)
  if (
    !userId ||
    !request.session.lastRefreshedAt ||
    request.session.lastRefreshedAt < Date.now() - SESSION_MAX_AGE * 1000
  ) {
    const newUser = await repos.user.createUser();
    userId = newUser.id;
    request.session.set('userId', userId);
  }

  // 最終更新日時を更新(タイムゾーン考慮不要)
  request.session.set('lastRefreshedAt', Date.now());

  return {
    repos,
    userId,
  };
};

export default createContext;
```

```ts:SessionData.d.ts
import '@fastify/secure-session';

declare module '@fastify/secure-session' {
  interface SessionData {
    userId: string;
    lastRefreshedAt: number;
  }
}
```

## 満たせる要件

* 外部APIがなく高速
* サーバー側でセッションをすべて管理
  * セッションをユーザーに紐づければ、保存できるデータに制限なし
  * 外部サービスと二重管理にならない
* フロントは非同期通信時にクッキーを有効化するだけ

## 満たせない要件

* DBなしの運用(セッションにデータを保持できない)
* 多くのプロバイダーのOIDCによるログインを簡単に設定したい場合(シンプルに大変)

-----

# 謝辞

今回はサイバーエージェントの[次世代トップエンジニア創出インターンシップACE](https://www.cyberagent.co.jp/careers/students/event/detail/id=28690)に参加いたしました。そこで得た知見となります。バックエンドを共にした[ずーまさん](https://twitter.com/azuma_alvin)やメンターの皆様をはじめアドバイスありがとうございました。

# 概要

Expressより速いといわれているfastifyを採用するケースは増えていますが、フロント&サーバー間の認証に外部サービスに頼ってしまうのは**怠慢**ですし、なにより**その分遅くなって**いますよね？？

そこで、今回は適切なクッキーの設定でセッション管理をすることで実現できる自前認証を構築します。

なお、後述しますがセッション(クッキー)は100%中身を見られない保証はないため、ステートフルの実装になっています。ステートレスの実装は以下が参考になります。

https://zenn.dev/calloc134/articles/fcff7a2ec2753c

# firebase Auth の流れ

まずは比較対象として、よく使われる認証サービスであるfirebase Authで考えます。

https://firebase.google.com/docs/auth/web/anonymous-auth?hl=ja

## フロント側

1. onAuthStateChanged()でログイン状況を確認
   a. ★内部ではfirebaseサーバーに問い合わせて、検証している
2. ログイン済みならば、ログインが必要なAPIのリクエスト時にIDトークンを付与
3. ★FirebaseのIDトークンを定期的に更新することで、セッションを維持

## バックエンド側 (ログインが必要なAPI)

1. リクエストに含まれるIDトークンを検証
   a. ★内部ではfirebaseサーバーに問い合わせて、検証している
2. 検証が成功すれば、`auth.token.sub`をユーザーIDとして利用

https://firebase.google.com/docs/auth/admin/verify-id-tokens?hl=ja#web

なお、`sub`は`uid`と同じな模様

https://stackoverflow.com/questions/40920403/what-is-the-difference-between-auth-uid-and-auth-token-sub-in-firebase-realtime

# 自前認証の流れ

## フロント側 (確認)

よく使われるクッキーと変わりありません。

1. ログインにまつわるクッキーの存在チェック → なければログイン
2. ★サーバーに保持しているクッキーを送信し、検証を依頼 → 無効だったり期限切れの場合はログイン
3. クッキーの妥当性が確認できれば、ログイン済みと判定

2番はwhoami等のログインが必要なAPIへリクエストを送ります。ログアウト時はクッキーを削除するだけでOK。(サーバーに任せても可)

## バックエンド側

### ログインAPI

1. リクエストをもとにユーザーを照合。 → なければログイン失敗
2. **\[クッキーを設定\]**
3. ログイン履歴の追加やログインメールの送信など…
4. レスポンスを返却

### ログインが必要なAPI

1. **\[クッキーを読取&検証\]**
2. **\[クッキーを設定\]**
3. (各処理)
4. レスポンスを返却

レスポンスには再度生成したSet-Cookieを含めることで、クッキーの有効期限を延長しています。

ログインが不要なAPIでも、クッキーの有効期限を延ばすならば同じ動作が必要です。

# 比較結果

firebase Auth を使うと認証にまつわるリクエストが

- APIリクエストにあたりバックエンド側で1つ
- 定期的なトークン更新リクエストがフロント側で1つ

と増えてしまう上、ログイン時やAPIリクエスト毎にもfirebaseを待つことになります。ライブラリで実現すると、これらを削減することができます。

しかし、認証に対する安全性を自分で管理する必要があるため、**仕組みやセキュリティを理解しておくことが大事**になります。

# 仕組み

## クッキーを設定

1. ユーザーIDと最終更新日時(現在日時)を`request.session`に設定。 ← `createContext()`
2. セッションを圧縮&暗号化し、クッキー文字列として生成。 ← @fastify/secure-session
3. HTTPヘッダーでSet-Cookieと共にレスポンスに含める。 ← fastify

ブラウザはSet-Cookieに従ってクッキーを保持。

## クッキーを読取&検証

クッキーの属性設定が正しければ、ブラウザはリクエスト時にクッキーを含めます。

1. クッキーの復号化を試みる。
2. 復号したクッキーをJSONとしてパースを試みる。 ← ここまで @fastify/secure-session
3. 最終更新日時が決めた時間より古くないことやユーザーIDが存在するかチェック。 ← `createContext()`

→ ユーザー認証が成功。クライアントがユーザーIDと同一とみなして処理を進める。

# セキュリティ

## クッキーの設定

`server.ts`のcookie内で設定します。以下の内容の日本語訳に近いです。

https://github.com/fastify/fastify-cookie#parseoptions

なお、クッキーの設定はブラウザにお願いする内容なので、**どれだけ準拠してくれるかはブラウザ次第**です。curl等でのリクエストやDevToolsで見られたSet-Cookieの値を保持すればいくらでも破ることができます。ステートフルクッキーにすることで状態管理が発生するものの、圧倒的にセキュアになります。

### domain

他のドメインで共用しないため、あえてundefinedを指定します。(記述しなくても可)

ドメインを書いてしまうとサブドメインにもクッキーが設定されてしまうとか？

https://blog.tokumaru.org/2011/10/cookiedomain.html

### path

あまり意味ないらしいですが、セッションを使う親パスが決まっていたり、GrpahQLの場合は設定するとよいかも。

指定しない場合は常に送信されるので、先ほどの実装は勝手に実行されます。

### httponly

JavaScriptで操作できないよう、trueを設定します。

### secure

HTTPSのみでクッキーを送信するかどうか。開発時以外はかならずtrueに設定します。

(常にtrueでもlocalhostは許されるとか許されないとか…？)

### expires

クッキー(セッション)の失効日時を文字列で指定します。後述のmaxAgeで設定すればよいので、undefinedとすることでブラウザを閉じたときに失効するようにします。遠めの日時を設定し、ブラウザ終了時の失効を防ぐことも可能です。

### maxAge

クッキー(セッション)の有効時間を秒単位で指定します。任意の時間で設定します。

### sameSite

違うドメイン間での遷移時にクッキーを付与するかの設定、ドメインを跨ぐ上、APIサーバーでクッキーを使う場合はnoneが必須。サンプルコードでは開発時はstrictに。

https://qiita.com/KWS_0901/items/695bd1ecc17a8c2e6d69

## クッキー(セッション)の暗号化

セッションを偽装されないために暗号化は必須です。@fastify/secure-sessionでは、いくつかの方法が提供されています。

* 秘密鍵方式(`key`、`secret`と`salt`)
* 秘密鍵をローテーションする方式(`key: [mySecureKey]`)

`secret`と`salt`を組み合わせると強力な秘密鍵を起動時に生成するため、一番簡便で強固です。今回はこの方式を採用しています。

適宜、ローテーションする方がもっと強固になりますが、今回は取り上げません。詳しいことは公式ドキュメントをぜひ見てください。

https://github.com/fastify/fastify-secure-session/#using-keys-with-key-rotation

## CSRFの設定

https://github.com/fastify/csrf-protection#use-with-fastifysecure-session

```ts
await app.register(csrfProtection, { sessionPlugin: '@fastify/secure-session' });
```

## セッション内に最新更新日時の追加

クッキーの有効期限はブラウザによって制御されるため、サーバー側で制御できるよう最終更新日時をセッション内に埋め込むことで、セキュリティを保持します。

具体的には、セッションの生成時に現在日時を埋め込み、セッション読み取り時に決められた日時より古くないかを検証します。

```ts
request.session.set('lastRefreshedAt', Date.now());
```

```ts
if (
  !request.session.lastRefreshedAt ||
  request.session.lastRefreshedAt < Date.now() - SESSION_MAX_AGE * 1000
) {
  // セッションの有効期限切れ
}
```

# 注意点1: セッションにJSONで扱えない型は含めない

`request.session`に設定するデータはなんでもエラーは出ませんが、`JSON.stringify()`や`JSON.parse()`を行います。

https://github.com/fastify/fastify-secure-session/blob/28b8f161513baa5a1e3aedb7ac6a57c18efd5e74/index.js#L202

https://github.com/fastify/fastify-secure-session/blob/28b8f161513baa5a1e3aedb7ac6a57c18efd5e74/index.js#L187

そのため、JSONで扱える型にしておくのがべきでしょう。例えば、Dateは以下のようなことが起こったり、`JSON.parse()`では文字列になってしまうため、`SessionData.d.ts`の型と不整合が生じてしまいます。

https://qiita.com/querykuma/items/c0c1775e9e8d74b1a378

つまり、使える型は

* 文字列
* 数値
* null
* 真偽値(true, false)
* オブジェクト(例: { a: 1, b: 2, … })
* 配列(例: \[ 1, 2, … \])

になります。このため、今回の実装では最終更新日時をUNIX時間で保存しました。

https://www.tohoho-web.com/ex/json.html

# 注意点2: 2種類の有効期限

今回の実装ではクッキーのmaxAgeとセッション内の最終更新日時による2つの有効期限が登場します。

| 種類 | クッキー | 最終更新日時 |
|----|----|----|
| 名前 | maxAge | lastRefreshedAt |
| 設定場所 | ブラウザ(フロントエンド) | サーバー |
| 設定タイミング | クッキー生成時 | クッキーの検証時 |
| 確実に適用されるか | **確実ではない** | 確実 |

APIサーバーを立てる側としては上のように比較されます。大事なのは、クッキーの有効期限は確実に適用されることが保証されないので、最終更新日時を別途設定することで、サーバーでセッションの有効期限を保証できます。

# 複数のセッションを構成するには

以下のような、`sessionName`をほかの設定と変えたものを挿入することで可能。デフォルトは`session`。また、セッションにアクセスする際は`request.[sessionName]`のようにアクセスできます。以下の例では`request.example`です。

server.ts

```ts
  await app.register(secureSession, {
    sessionName: 'example', // ここを被らないようにして追加
    secret: SESSION_SECRET,
    salt: SESSION_SALT,
    cookie: {
      // options for setCookie, see https://github.com/fastify/fastify-cookie
    },
  });
```

https://github.com/fastify/fastify-secure-session/#multiple-sessions

# @fastify/secure-sessionで追加した実装の型定義を追加

今回の実装では`SessionData.d.ts`のようなファイルを追加すれば、`request.session`でアクセスできるようになります。複数のセッションを定義する場合は`FastifyRequest`を追加する必要があります。

別ファイルで定義する場合は、

```ts
import '@fastify/secure-session';

declare module '@fastify/secure-session' {
  interface SessionData {
    foo: string;
    bar: number;
  }
}
```

や

```ts
import 'fastify';
import { Session } from '@fastify/secure-session';

interface ExampleSessionData {
  foo: string;
  bar: number;
}

declare module 'fastify' {
  interface FastifyRequest {
    example: Session<ExampleSessionData>;
  }
}
```

のようにimportが必要です。(要検証？)

https://github.com/fastify/fastify-secure-session/#add-typescript-types

# まとめ

認証のフローをよく理解していれば、自前での実装でも簡単に作れて、かつセキュリティリスクを抑えることができます。firebase Auth等のサービスを使ってもライブラリの更新は欠かせないですし、要件に応じて正しく選んで使っていきましょう。
