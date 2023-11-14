---
title: "GraphQL自体の雰囲気について調査(主にNode.js)"
emoji: "🔍"
type: "tech"
topics: ["nodejs", "graphql"]
published: true
---

https://zenn.dev/ureo/articles/graphql-nodejs

にて言及したとおり、Node.jsにてGraphQLスキーマ周りのライブラリへの活動が活発でないことから、もしかしたらNode.jsだけではないのではと思い少し調査してみました。

# GraphQLの各指標の推移を調査

https://npmtrends.com/graphql

まず、NPMでは今年に入って公式ライブラリである[graphql](https://npmjs.com/package/graphql)自体が伸び悩んでいます。

https://trends.google.co.jp/trends/explore?date=today%205-y&q=GraphQL&hl=ja

Google Trendsでは人気度の動向はほぼ変わらないようです。

https://star-history.com/#graphql/graphql&graphql/graphql-js&Date

GitHubのStar Historyでは、jsのライブラリが特に伸び悩んでいます。

https://insights.stackoverflow.com/trends?tags=graphql

最後にStack Overflowでのトレンドも伸び悩んでいるようです。

# Apollo関連についても同じく調査

https://trends.google.co.jp/trends/explore?date=today%205-y&q=Apollo%20Server,Apollo%20Studio&hl=ja

https://star-history.com/#apollographql/apollo-server&apollographql/apollo-client&Date

https://insights.stackoverflow.com/trends?tags=apollo-server%2Capollo-client

Star Historyは伸びているものの、そのほかはGraphQLと同じ傾向です。

以上から

**GraphQL全体の人気の熱が冷め始めている？**

と考察できます。よくプロダクトライフサイクルとして、

1. 導入期
2. 成長期
3. 成熟期
4. 衰退期

とあります。GraphQL自体は製品ではないですし、OSSで維持されているものですので、売り上げといったものはありませんが、支持数に置き換えることも可能かと思います。

https://globis.jp/article/6695

すると、グラフ的には「成熟期」相当であり、今後伸びていくのか、さらに下降してしまうのかは注視したいところではあります。

# 成熟期を迎えた理由は…？

個人的観点での理由もいくつか上げてみようかと思います。

なお、すでに詳しい記事は既にいくつかあります。

https://engineering.mercari.com/blog/entry/20220303-concerns-with-using-graphql/

https://www.docswell.com/s/LIFULL/5GXNYP-2023-09-05-100929#p16

## シンプルに学習コストが高い

GraphQLはRESTと異なるコンセプトを持ち合わせており、初学者はクエリとミューテーションの違いやスキーマの記法、ID型などを理解する必要があるため、学習コストは高いです。しかし、後発であるがゆえにRESTよりもドキュメントが少なかったり、実際のAPIというのも少なかったりするのもハードルを高く感じさせます。

## gRPCなどの競合技術の台頭

GraphQL以外の技術により、トレンドが移行していった、あるいは単に選択肢が増えたからというのが1つです。

gRPCについては全く触れていないので、ここでの言及はしません。

## HTTPとの親和性の問題

GraphQLはあくまでもWeb技術上で作られたAPIで、そのWeb技術とはHTTPであり、RESTです。つまり、HTTPの進化によってによってRESTが最適化される未来があっても、GraphQLが最適化されることは期待しにくいです。

その代表としてGraphQLはすべてがPOSTリクエストであるため、デフォルトではHTTPキャッシュが効きません。これは、**「HTTPではGETメソッドしかキャッシュしない」**という原則があるためです。代替案としては**一部のクエリをGETメソッドに置き換える**という手法もありますが、リクエストURIの文字数制限があるため、全てのクエリで適用できるわけではありません。

また、GraphQLのエンドポイントは原則1つですが、これもHTTPの思想と大きく異なるため、キャッシュの構築に影響します。GraphQLはクライアントが欲しいリソースに合わせて細かくクエリが変化しますが、**その分レスポンスが変わる**ことからキャッシュヒット率がRESTに比べて大きく下がります。あらかじめ想定されるユースケースに合わせてリクエストを飛ばし、キャッシュを生成させることもできますが、リクエストを網羅できるかはわかりません。

ネットワークレベルのキャッシュは現状だとHTTPキャッシュ以外の選択肢がないことから、大規模サービスとして展開するにはCDNを活用できるHTTPキャッシュへの対応は避けて通れません。

## 潜在的セキュリティホールの残存

クライアントにリクエストの自由度があるということはその分**脆弱性やセキュリティホールが発生し得る**ということになります。GraphQLにセキュリティホールがあるというわけではありませんが、クライアント側のリクエストを網羅できないことはリスクにつながります。

細かい詳細は以下にお譲りいたします。

https://logmi.jp/tech/articles/329491

# まとめ

GraphQLはこの秋に少し触っただけなので、REST信者なところはあるかと思いますが、実際に開発したり色々調べたりすることで、GraphQLのメリデメがよく分かったと思います。

近年では、GitHubがGraphQLをフルサポートしていますし、それをラッパーするライブラリがあれば細かい話は開発側としては関係なく使えるかもしれません。また、いつか検証できたらいいなと思っています。
