---
title: "Synology NASでワイルドカード証明書を取得する"
emoji: "🪪"
type: "tech"
topics: ["Synology", "NAS", "Docker", "letsencrypt"]
published: true
---


大本の話はこちらで詳しく書かれています。

https://eng-blog.iij.ad.jp/archives/14198

ですが、今回はこれをDockerで作ることによってNASでも手軽にワイルドガード証明書を運用できるようにします！SynologyのNASは証明書の自動取得には対応しているものの、http-01チャレンジのみなので、これでバンバンOSSをホストできますね！

## Github

[![u0reo/certbot-dns01 - GitHub](https://gh-card.dev/repos/u0reo/certbot-dns01.svg)](https://github.com/u0reo/certbot-dns01)

## Dockerイメージ

[![dockeri.co](https://dockerico.blankenship.io/image/ureo/certbot-dns01)](https://hub.docker.com/r/ureo/certbot-dns01)


初めはNAS向けに作っていたのですが、ほかにも使いまわせるようにしました。

おおむね手順は同じですが、/certificate向けのボリューム設定は不要です。(消しても余計なボリュームは作成されません。)その代わり、毎回certbot_dataボリュームから直接証明書データをコピーする必要があります。

## 環境変数

- DOMAIN
  - 証明書を取得したいドメイン
  - *.example.comとexample.comの両方の証明書を作ります
  - acme.example.comのAレコードやAAAAレコードは$ADDRに割り当てが必要
  - (*.example.comをまるごとマシンに割り当てていてもよい)
- ADDR
  - DNSサーバーを立てるマシンのIPアドレス
  - 実行時は53ポートの開放が必要
  - IPv4だとルーターのキャッシュDNSサーバーと被ることもあるのでIPv6を推奨
- EMAIL
  - certbotで使うメールアドレス
  - 期限切れが近いとメールをくれる
  - ニュースレターはなしに設定済み
- TEST
  - trueでcertbotをステージングモードに

本来Docker非対応なSynology NASではブリッジが使えないので、ネットワークは常にホストと共有させます。

一時的にDNSサーバーを立てる都合上、NASのDNSサーバーパッケージを使っているとうまく動かないので、不要な場合は削除するか別途IPv6アドレスを生成する必要があります？(IIJのブログ参照)

また、NASのファイアウォールを使っている前提で、証明書発行のときのみ無効化することにします。ファイアウォールをそもそも使っていなかったり、53ポートを常に開ける等は各自読み替えてください。

## 下準備

DockerをNASにインストールします。
https://qiita.com/AtsushiYokokawa/items/72157a676866296d8bbe

レコードをいくつか設定します。

- `acme.example.com` に `$ADDR` を設定する (AまたはAAAAレコード)
- `_acme-challenge.example.com` にNS(CNAME)レコードを `acme.example.com` 向けで設定  

`$ADDR`を53ポートでアクセスできるようポート開放をします。

SSH上で
```shell
sudo docker pull ureo/cert-dns01
```
コンテナをプルし、
```shell
sudo docker run --rm -i --net host \
-e DOMAIN=example.com \
-e ADDR=2001:db8::1 \
-e EMAIL=letsencrypt@example.com \
--name certbot-dns01 \
-v certbot_data:/etc/letsencrypt \
-v knot_config:/config \
-v knot_rundir:/rundir \
-v knot_storage:/storage \
-v /usr/syno/etc/certificate:/certificate \
ureo/certbot-dns01:0.1 ./init.sh
```
すると、
```
12345 11 2 0123456789abcdef...
```
のようなテキストが出力されるので、`acme.example.com`のDSレコードとして設定します。
レコードが浸透したかどうかは
```shell
docker run -rm ... kdig acme.example.com ds
```
で確認できるので、浸透してそうだったら次に証明書を取得します。

## 証明書を取得

SSH上で
```shell
sudo docker run --rm -i --net host \
-e DOMAIN=example.com \
-e ADDR=2001:db8::1 \
-e EMAIL=letsencrypt@example.com \
--name certbot-dns01 \
-v certbot_data:/etc/letsencrypt \
-v knot_config:/config \
-v knot_rundir:/rundir \
-v knot_storage:/storage \
-v /usr/syno/etc/certificate:/certificate \
ureo/certbot-dns01:0.1 ./first.sh
```
を実行、すると
```
Submitting DS Record Registration...OK
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-rfc2136, Installer None
Account registered.
Running pre-hook command: knotd -d
Requesting a certificate for example.com and *.example.com
Performing the following challenges:
dns-01 challenge for example.com
dns-01 challenge for example.com
Waiting 5 seconds for DNS changes to propagate
Waiting for verification...
Cleaning up challenges
Running post-hook command: knotc stop > /dev/null
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example.com/privkey.pem
   Your certificate will expire on 2023-03-28. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
となれば成功。
発行された証明書は`/docker/volumes/certbot_data/_data/live/`以下にあるので、はじめだけはどうにか引き出して手動で登録する必要があります。NASだと共有フォルダにコピーしてしまうのが楽。
```shell
sudo cp -Lr /docker/volumes/certbot_data/_data/live/${DOMAIN}/ ~/new_cert/
sudo chown -R `whoami`:users ~/new_cert/
```

これらのキーをDSMのコントロールパネル->セキュリティ->証明書で登録します。すでに同じドメインの証明書を登録していても、登録から置き換えが可能。その際、各ファイルは
![DSMで証明書をインポート](/images/docker-certbot-dns01_dsm_cert_import.jpg)
のように対応します。

### よくあるエラー

**DNS problem: SERVFAIL looking up TXT for _acme-challenge.example.com - the domain's nameservers may be malfunctioning**  
**DNS problem: query timed out looking up TXT for _acme-challenge.example.com**  
**Encountered error when making query: The DNS operation timed out.**

おそらくDSレコードが正しく設定されていないため、下準備からやり直してinit.shを実行するか
```shell
docker run -rm ... ./get_ds.sh
```
で得られるDSレコード(本来設定されるべきもの)と
```shell
docker run -rm ... kdig acme.example.com ds
```
で得られるDSレコード(現在設定されていてネームサーバーから取得できるもの)が一致するか確認し、違う場合は前者でとれたDSレコードを設定し再度証明書を取得する。

**Encountered exception during recovery: certbot.errors.PluginError: Unable to determine base domain for _acme-challenge.example.com using names: ['_acme-challenge.example.com', 'example.com', 'com'].**

何度か遭遇したが、Docker上に立てているKnot DNSに外からアクセスできていないのが大体の原因。NASのファイアウォール、ルーターのポート解放、そもそものアドレス等を再度確認する。
今回のコンテナでは`./script.sh`を`sh -c "knotd -d && sleep 3600 && knotc stop"`などと書き換えることでDNSサーバーだけをずっと立てることができるので、その間に外からアクセスできるかを確かめる。

**Submitting DS Record Registrationがエラーに**

```
Submitting DS Record Registration...error: [_acme-challenge.example.com] (no key ready for submission)
```
既にacme.example.comのDSレコードを設定し、それをKnot DNSに通知済みなだけなので、最後に成功した後にinit.shを実行していなければ無視してOK

## 証明書を自動で更新するように設定

SSH上で
```shell
sudo docker create -i --net host \
-e DOMAIN=example.com \
-e ADDR=2001:db8::1 \
-e EMAIL=letsencrypt@example.com \
--name certbot-dns01 \
-v certbot_data:/etc/letsencrypt \
-v knot_config:/config \
-v knot_rundir:/rundir \
-v knot_storage:/storage \
-v /usr/syno/etc/certificate:/certificate \
ureo/certbot-dns01:0.1 ./renew.sh
```

今回作成するコンテナは使いまわせるため、renew.shを使うだけでなく、

- runではなくcreateのみ
- 終了時にコンテナ削除はしない

とオプションを変えています。
それと、renewの場合は証明書を手動で登録する必要はありません。

DSMに入り、
コントロールパネル->タスクスケジューラー->作成->ユーザー定義のスクリプト
ユーザーはrootにし、スクリプトは
```shell
synofirewall --disable &&
docker start certbot-dns01 -ai &&
synosystemctl restart nginx &&
synofirewall --enable
```
にします。日付は好きなように設定してください。certbotのrenewは失効1か月を切ると行われるので毎週やれば十分だと思い、何かあっても対応できるであろう毎週土曜日の明け方に設定しました。

証明書を使うパッケージがほかにある場合はreload nginxの後に続けてください。
参考: [CLI script to programmatically replace SSL certs on Synology NAS](https://gist.github.com/catchdave/69854624a21ac75194706ec20ca61327)

```shell
synosystemctl restart nmbd
synosystemctl restart avahi
synosystemctl reload ldap-server
```

### 実行結果

更新があるとき
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/example.com.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert is due for renewal, auto-renewing...
Plugins selected: Authenticator dns-rfc2136, Installer None
Running pre-hook command: knotd -d
Renewing an existing certificate for *.example.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
new certificate deployed without reload, fullchain is
/etc/letsencrypt/live/example.com/fullchain.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all renewals succeeded:
  /etc/letsencrypt/live/example.com/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Running post-hook command: knotc stop > /dev/null


Generated certificate file copied to /certificate/ReverseProxy/12345678-90ab-cdef-1234-567890abcdef
...
Generated certificate file copied to /certificate/ReverseProxy/567890ab-cdef-1234-5678-90abcdef1234
Generated certificate file copied to /certificate/_archive/aBcDeF

The certificate for the domain (example.com) successfully renewed & replaced!!
```

更新がないとき
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/example.com.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert not yet due for renewal

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
The following certificates are not due for renewal yet:
  /etc/letsencrypt/live/example.com/fullchain.pem expires on 2023-03-26 (skipped)
No renewals were attempted.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -


The certificate for the domain (example.com) has not been renewed.
```

## 所感

ラズパイでやる分には爆速でできるのに、HDDのみのNASでやるとすごく遅い。(毎回のDockerコマンドに数分)
ただ、Dockerのコンテナ生成に時間がかかっているように見えるため、renewのときはそんなに負担かからなそうです。
