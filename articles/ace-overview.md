---
title: "サイバーエージェントのインターンACEに参加しました"
emoji: "🧑‍💻"
type: "tech"
topics: ["intern", "cyberagent", "nodejs", "graphql", "cache"]
published: true
---

# ACEとは

https://www.cyberagent.co.jp/careers/students/event/detail/id=28690

株式会社サイバーエージェントが2週間にかけて開催する長期間インターンであり、ハッカソン的なもの。

1人ずつメンターが付き、開発の相談はもちろん、個人の目標やそれに向けての補助を毎日行ってくれました。(もちろん優しかったです)

チームはランダムに作られ、担当する領域は事前の希望どおりでした。3チームある中、自分のチームは4人でフロントとバックで半分ずつ担当が分かれていました。自分の相方は[ずーまさん](https://twitter.com/azuma_alvin)でした。本当に今回はありがとうございました。

# 期間中の流れ

- 顔合わせオンラインMTG
- 1日目～4日目 リモート開発
- 5日目 リモート開発&中間報告
- 6日目～8日目 リモート開発
- 9日目～10日目 対面開発
- 11日目 15時まで対面開発&成果発表会&表彰

要件は必須と任意のものが2つありましたが、ここではどれがどれかは取り上げないです。__無論、どちらも実装する心意気だったので。__(なお、認証は任意。)

フロントエンド自体はわかるものの、今回はほぼ開発に協力していなかったので、詳しくは触れません。

## 顔合わせオンラインMTG

チームビルディングをしていく→今回の開発の目標を決め、ベクトル合わせ

- フロントの1人が半分居ない
- 自分はチーム開発経験少ない

→ドキュメントを書こう！！(若干後付け)

\+パフォーマンス重視(Webアプリとしても、開発としても)

バックエンドで使う言語は2人ともわかるTypeScript(Node.js)に。しかし、Goのほうが高速な可能性もあり、スコープ外にはしないことに。

## 1日目

**<GraphQLの採用決定>**

事前にお題の概要は出ていたものの、チームメンバーは大して検討していなかった。(バックエンドの相方のみモックを用意👏)

詳細なお題が出たので、雑に作業コストを見積もる。

- 2日間 技術選定
- 1\~2日間 要件定義
- 3\~4日間 実装
- 4日間 マージン+フロントとのつなぎ合わせ

→ 技術選定にじっくり時間をかけられそう！

ただ、フロントを待たせられないので、APIを決定することに。RESTとGraphQLを比較し、GraphQLを使うことに決定。(経緯は別記事で)

他の部分の技術選定は調べたものの、翌日実際に触れて検証することに。

## 2日目

**<Apollo+pothosの採用決定>**

経緯は以下にあるように、じっくり時間を使って選定しました。

https://zenn.dev/ureo/articles/graphql-nodejs

使用するライブラリが決まったので、大まかに思いつくタスクを生成。

## 3日目

**<ER図によるDB設計、ライブラリの詳細な検証>**

相方が要件やフロントデザイン、モックデータの形からER図を書き起こしてくれました👏 複雑な時系列データはサーバで処理することがなさそうだったため、JSONで格納することに。

自分はライブラリ群を0から理解し、簡単なモックを作ってクラウド上でも動作することをいったん確認。

この辺から分業が本格的に進められるようになりました。

## ~~土日~~

~~ちょっと技術選定が押していたので、残業していました~~

**<認証はクッキーで自前実装(×Firebase ○周辺ライブラリ採用)に決定>**

ここの数日、幾度となく認証をどうするか議題に上がっては悩んでいました。

要件は匿名&自動ログインでお気に入りを保持で、運営はFirebase Authを使用できるよう用意してくれていましたが、

- フロントでログインはライブラリサイズが大きいので、あまりしたくない(後にAPIで手動でもできることが判明)
- バックエンドでログインしてもいいが、それだとクッキーでの管理でもいいのでは…

などと方向がチーム全体で迷走しました。パフォーマンス重視で進める上でいくら早いGoogleのAPIを叩くといっても、APIを叩くこと自体が遅いのです、、、

幸い自分はfastifyを使ったクッキー周りに経験があったので、後日expressから切り替えつつ、認証機能も実装することにしました。~~この部分も余裕があったら記事にしたいです。~~

(2023/12/04 追記)

記事にしました。

https://zenn.dev/ureo/articles/fastify-self-auth

加えてライブラリのドキュメントにほぼ言及のないユニットテストについて、どこまで実装できるかを試行錯誤し、独自のスクリプトに書き起こしました。(これはライブラリにしてもよさそう)

## 4日目

**<DB設計の練り直しと一部のスキーマを実装>**

未だにスキーマが定まっていなかったものの、GraphQLを使う以上、DB設計とスキーマは似るため、現状のER図をフロント側に見てもらうことに。

すると、__サンプルデータには直接存在しないが、フロントには必要そうなフィールドがいくつもある__ことが判明しました。

「どのような取り方がいいか」「このようフィールドを用意したら使えるか」などと議論を重ね、それをサンプルデータ取り込み時に生成するように。相方がサンプルデータを挿入するシードスクリプト自体を~~土日に~~作成してくれていたので、それを自分が少し修正。

ここに来てやっと各クエリの実装を開始できました。

そして夜になって明日の中間報告の存在を知って絶望😨

~~gitの履歴によると、日付またいだ朝方に新しいテーブルを追加したシードスクリプトが完成~~

## 5日目

**<ほとんどのクエリとリゾルバが実装済みに>**

採用したライブラリの力により、ER図や前日の議論に基づくGraphQLのスキーマの実装自体はほぼ完了に。

スキーマが早くできることでフロント側で足りないクエリやパラメータなどを検証しやすくなるため、2人でスキーマ実装に全力を捧げました。

## 中間発表

バックエンドのこの時点での成果は

- DBの大まかな設計
- サンプルデータの挿入
- 開発の楽さとわかりやすさを重視した技術選定

一方、タスクは

- クエリやミューテーションのブラッシュアップ
- fastifyへ移行&認証周り
- テスト&バリデーション
- DBのインデックス張り
- joinの制限付け
- (CI&CD)

と永遠に積もる状態に。。。

## 6日目

**<認証が一旦完成>**

fastifyへの切り替えをしつつ、認証を実装。これにより、認証の必要なスキーマも要件通りの実装に変更。ただ、不安な点も多かったため、メンターと相談。

~~この辺でデータをすべて挿入していたため、サンプルデータの異変にも気づき始めました。~~

## 7日目

**<ユニットテストの完成、認証周りの修正>**

相方が自分の書いたユニットテストを解読しつつ、ほかのスキーマにも実装してくれました👏

自分は認証の期限などの実装で悩みに悩むことに。とはいえ結局ユーザーIDと最終認証日時を暗号化した文字列をクッキーとして保存し、アクセス毎に延長していく方式に。

## 8日目

**<E2Eテストの一部実装、細かい部分の修正>**

相方がE2Eテストを実装してくれた上に、README書いてくれました👏👏

引数のバリデーションや存在しないリソースの取得などのエラーを普通のエラーとして出力していました。それを二人で相談し、種類別に分け、カスタムGraphQLエラーとしてスローするように。

また、GraphQLの最大の懸念点であったN+1問題を検証すべく、Prismaが生成した生クエリを出力したものの、テーブルごとにまとまっていて対策不要であることを確認👍

## 9日目

**<ドキュメントの追加、キャッシュの検討>**

この時点で要件のほとんどは満たしており、最終発表も迫ってきたため、ドキュメント(アピールポイント)を整理しました。

また、フロントも大きく変化するところが少なかったため、よりパフォーマンスが上げるべくキャッシュの追加を検討しました。しかし、__GraphQLのキャッシュは前途多難__。しかも、CloudFrontっておいしいの状態だったので、調査までが限界でした。

~~なお前日は暢気に引数でタイムゾーン指定ができるよう改善。~~

## 10日目

**<E2Eテストの完成、キャッシュの実装>**

相方がログインを含めたE2Eテストを完成させてくれた上、K6による負荷試験も実施してくれました👏👏

キャッシュは何種類かあったものの、サーバー内のメモリキャッシュに加え、余裕があればCDNキャッシュをすることに。

メモリキャッシュでは、pothosの仕様でなぜかキャッシュされないなどのトラブルもあり、「一旦すべてにキャッシュを設定して、一部キャッシュさせない」方針を取るように。

CDNキャッシュにはエラーなどをHTTPステータスコードへ反映が必須だったものの、GraphQLは全て200を返す仕様がデフォルトでその部分は自作が必要に。~~この辺はおいおい記事にしたいです。~~

(2023/12/05 追記)

記事にしました。

https://zenn.dev/ureo/articles/graphql-cache

## 11日目

**<CDNキャッシュも実装>**

CDNキャッシュに対応できるようステータスコードへの反映ができたものの、AWSの魔窟にはまり、CloudFrontを必死に設定しました。

また、開発利便性を上げるため有効化していたWeb上のGraphQLクライアントでしたが、セキュリティ上はとても良くないので、GitHub認証をかけようと実装はしたものの「ログインがうまくかない」「フロントのビルドができない(サーバー上のスキーマを参照している)」などで、直前で取り払うことに、、、。

## 成果発表会

スライドを用いて発表し、

- 技術選定
- DB設計
- キャッシュ戦略
- 負荷試験

をメインに発表し、1枚のスライドには

- **認証は自作に & 基本的なセキュリティの実装(CORSやCSRF…)**
  - 高速 & フロントにシークレット置かなくてOK & サーバー主導のユーザーの一元管理
- **レプリケーションDB対応**
  - ReadOnlyDBの高速性をバリデーションに生かして高速化 & DBセーフな設計
- **タイムゾーン対応**
  - 日付区切りでレスポンスを返す際に任意のタイムゾーンを指定可能
- **OpenTelemetry を使った トレース機能の追加 / OAuth2認証による攻撃防止 (検討のみ)**
- **パフォーマンス重視**
  - キャッシュや自作認証に加え、Node.jsのWebサーバーにExpressではなく、Fastifyを採用
- **フロントエンドのユースケースに沿ったE2Eテスト**
  - GitHub Actionsで認証を含めたE2Eテストを自動化 → 最終的には83.18%に
- **ユニットテストヘルパーを自作**
  - GraphQLのフィールドがあるかどうか、リゾルバの単体テスト
- **データをサーバー側で前処理して挿入**

などと、とりあえずアピールポイントをもりもりに。

# 結果

最優秀チーム賞をいただきました🎊

ありがとうございました！

# 感想

他チームも含め、周りのレベルはすごい高いうえに、年下ばっかりだったので、本当に優秀な人たちの中で開発できたと思います。また、結構要件外のことに触れてメンターやそれ以外の方も巻き込んで「これ使いたいです～」なんてことも多々あったのですが、ほんとに柔軟に親身に対応してくれました。

自分の目標は

- 要件をしっかり満たして優勝
- チーム開発に慣れ、大規模プロジェクトの中でも貢献できるように
- 他人ファーストなコードを書く

と3つあったのですが、どれも達成することができたと思います。今まで、個人開発や実務をしてきたものの、同じ人たちと集中して何日も開発するというものが意外となかったため、新しい環境下での経験により自分を成長させることができたと思います。

皆さんもぜひACEをはじめ、長期インターンでの開発をたくさん触れてほしいなと思います！最後まで、お読みいただきありがとうございました。そして、メンバーやメンター、そして運営の方々、ありがとうございました。

