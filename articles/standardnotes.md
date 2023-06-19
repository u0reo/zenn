---
title: "Notion代替のセルフホストにStandard Notesをお勧めしない話"
emoji: "🗒️"
type: "tech"
topics: ["oss", "self-hoted", "notion", "standardnotes"]
published: true
---


## TL;DR

Standard NotesはOSSのノートアプリでセルフホストが可能。

MarkDown等の書式を使ったりファイルをアップロードする機能が有料機能に含まれるため、有料版の利用をしたいところ。

サーバー版の有料機能はDBの変更で解放できるが、クライアント(Webも含む)の有料機能は公式の半額でサブスクが必要だが渋い。


## いきさつ

Notionは個人ならば無料版でも十分ですが、なにかのキーとかの秘密にしたいデータとかもどんどん保管したいとなると少しながらセキュリティが心配になります。
オープンソースのノートアプリはたくさんあり、多くがローカルで完結させることで機密性をありますが、クラウド同期できるものはとても少ないです。
しかし、セルフホストに興味がある方は端末がいくつかない人なんていないでしょうかし、クラウドで同期するオプションがないのは少し不便なものです。

そこでネットを漁っていたところ、[Standard Notes](https://standardnotes.com)が良いのではないかと思い、セルフホストを試してみました。

## 構築方法

docker-composeで簡単に構築できます。

公式のドキュメントは[こちら](https://standardnotes.com/help/self-hosting/docker)。

https://github.com/u0reo/docker-compose-files/blob/main/standardnotes.yml

後述する大きいデメリットによりおすすめしないので、構築方法は省略します。

[公式のcomposeファイル](https://github.com/standardnotes/server/blob/main/docker-compose.example.yml)との違いは次のようになります。

- webを追加しているが動作しない
- 環境変数が直書き
- dbやredisがない
- MySQLをソケット接続させるためdocker-entrypoint.shに細工がある

Webに関しては

```
exec ./docker/entrypoint.sh: exec format error
```

とログが出るが細かい理由は不明。

## 何が気に入らなかったか

デフォルトの設定だと無料版と同じ扱いなので、有料版の機能を有効化させてやります。

サーバー側は[こちら](https://standardnotes.com/help/self-hosting/subscriptions)の通りにやれば有料版と同じ扱いになります。(DBに変更を加えて再起動してもOK)

**ですが**、このままではアプリ側の有料機能は解放されません。それには[こちら](https://standardnotes.com/help/48/can-i-use-extensions-with-a-self-hosted-server)にあるように**毎年39ドル(2023年4月現在)のサブスク**を購入しなければならないのです。

有料機能には

- MarkDownやリッチテキストの使用
- ToDoリストの使用
- スプレッドシートの使用(テーブル？)
- ほかのクラウドへのバックアップ
- タグのフォルダ作成

などがあり、どれも捨てがたいものになっていますが、ほとんど使えないものとなっています。

そもそも、有料機能をほしいがためにセルフホストするのは少し違うものの、サブスクまで要求するのもひどいかと思います。

## あとがき

[海外の掲示板](https://www.reddit.com/r/selfhosted/comments/11v1z36/standard_notes_releasing_simpler_20_selfhosting/)にあるように早く気付けばよかったのですが、、、

そこで紹介されているように[Notesnook](https://notesnook.com)の[セルフホスト版が今開発中](https://github.com/streetwriters/notesnook-sync-server)とのことらしいので、出来次第そちらに移ろうかなと思ってます！

https://kewpie13.hatenablog.com/entry/2022/08/25/080913

また、こちらの[AFFiNE.PRO](https://affine.pro)もいいな～と考えていますが、2023年現在では絶賛開発中となっています。
