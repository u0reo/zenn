---
title: "Traefikで躓いたところまとめ"
emoji: "🛣️"
type: "tech"
topics: ["Traefik", "Tips", "HTTP", "Proxy", "Docker"]
published: true
---

各所で説明されている通り、個人でDockerによる運用であればnginxより、Traefikが便利です。

とはいえ、日本語記事がいかんせん少なく、「Apacheなら、XXX」「nginxなら、XXX」ということは多々あります。

Dockerを使うことによりネットワーク構成が複雑化し、余計にわかりにくくなったのも大きいと思います。

https://www.smarthomebeginner.com/docker-authelia-tutorial/

今回は主にこの記事を中心に構成しつつ、躓いたポイントを以下で紹介します。

なお、Autheliaによる認証追加は別記事でもう少し細かく紹介します。

## 証明書ファイルの指定は環境変数では不可能

https://stackoverflow.com/questions/58584625/how-do-i-reference-a-self-signed-ssl-certificates-for-traefik-v2-in-a-docker-com

> It would be cool to have everything configured in a single docker-compose file but unfortunately the self-signed related configuration must be stored in a separate file.

とあるように、証明書ファイルの設定は別途ファイルを読み込ませる必要があります。しかし、この回答での`traefik.yml`だとワイルドガード証明書を使っているせいか、上手く適用されませんでした。

https://doc.traefik.io/traefik/https/tls/#default-certificate

公式のドキュメントにあるデフォルト設定で指定すると解決しました。

## 証明書ファイルはfullchain.pemを指定すべき

https://github.com/u0reo/docker-compose-files/blob/main/traefik/config_cert.yml

`certFile`は中間証明書のない証明書を指定してしまうと、環境によっては「ルート証明書までたどり着けない」などとエラーが表示されます。

そのため、Let's Encryptでは`fullchain.pem`を指定しましょう！

実際に検証してみましたので、興味ある方はご覧ください。

https://zenn.dev/ureo/articles/intermediate-ca-lost

## 構成に誤りがあってもエラー等は表示されない

モダンなプロキシらしく、変更した構成や設定は基本的に即時適用されます。Dockerのラベルに記載した設定でもです。
ですが、構成に誤りがあった場合、ログを見てもダッシュボードを見てもエラー等は一切吐かれません…

その代わり、**一番最後に読み込めた設定**が適用されます。
しかも、プロバイダー(Dockerとかファイルなど)の単位で検証&適用されるので、結構厄介です。
そのため、初めはダッシュボードをHTTPで開放して確認するのをオススメします。

ちなみに、再起動時には構成が正しいかどうかで、プロバイダー単位で読み込まれます。つまり、ファイル構成の1つでもおかしい場合、ファイルで書いた構成はすべて適用されません。
これでも、ある程度の原因の切り分けがつくかと思います。

なお、yamlの仕様の話ですが、`*`などはシングルクォーテーションによる囲いが必要なので、そういったところも気を付けましょう…


## 開放するポートは一つのみに

traefikは解放されるであろうポートを勝手に判別しているので、開放するポートをexposeで1つだけ指定する必要があります。portsで開放する必要もないので安全に設定することができます。

Dockerfileですでに複数設定されている場合や複数のポートを使いたい場合は次のセクションをご覧ください。

## 1つのコンテナで複数ポート使用する場合

https://kuttsun.blogspot.com/2022/06/docker-traefik_15.html

```
traefik.http.services.XXX.loadbalancer.server.port: 80
```

でロードバランサーを定義し、

```
traefik.http.routers.YYY.service: XXX
```

ルーターに先ほどのロードバランサーを設定したサービスを指定することで可能です。なお、XXXやYYYはほかのコンテナで設定する文字列と被らなければなんでもOKです。

なお、ルート設定である`http.routers.YYY.rule`はほかのコンテナ等の構成とかぶらないように設定する必要があります。

### 設定例

https://github.com/u0reo/docker-compose-files/blob/main/vaultwarden.yml#L32-L46

## IPv6対応をしたい

https://qiita.com/shigeokamoto/items/f0d6dbb87d3fe8858d61

Dockerコンテナなので、以上の方法が使えるかと思います。
今回は`network_mode: host`を利用しましが、それは後日。

## QUICやHTTP3に対応したい

https://zenn.dev/pitekusu/books/traefik-pitekusu/viewer/http3

Dockerのラベルで記述すると

https://github.com/u0reo/docker-compose-files/blob/26ad367597ee0edbc191876d331d5bc705ad96fa/traefik.yml#L42-L43

の2行でOKです。
