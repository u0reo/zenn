---
title: "中間証明書がないcert.pemを設定したときの挙動調査"
emoji: "🛣️"
type: "tech"
topics: ["SSL証明書", "OpenSSL", "TLS", "SSL", "letsencrypt"]
published: true
---

https://kitakantech.com/lets-encrypt-certificates/

上記サイトにあるように、Let's Encryptをはじめとした所で発行されるSSl/TLS証明書ファイルには以下のようなものがあります。(ファイル名はLet's Encryptのもの)

- fullchain.pem (全部入りの証明書公開鍵)
- cert.pem (自ドメインの証明書公開鍵)
- chain.pem (中間証明書公開鍵)
- privkey.pem (証明書秘密鍵)

証明書公開鍵を設定するときは、`fullchain.pem`も`cert.pem`も`chain.pem`も同じように設定できてしまいます。
しかし、これらを間違えることにより、「うまく認証できない場合が」あります。

## ブラウザでは中間証明書がなくても大丈夫なことも…

https://qiita.com/nofrmm/items/5e50f077eb1602a3458e

「場合がある」と書いたのは、最近のブラウザは気を利かせて中間証明書を取りに行ってくれることもあるらしく、`cert.pem`を設定してもブラウザではエラーが出ないこともあるからです。
(とはいえ「おま環」なので、良いわけではないです。)

ファイル名的にもこれらの証明書ファイルは混同しやすいですが、認証できないのは手元でも検証することができます。
以下に実際の検証結果を載せます。

## 中間証明書がない場合 (certFile: cert.pem)
```
$ echo | openssl s_client -connect sub.example.com:443 2> /dev/null
CONNECTED(00000003)
---
Certificate chain
 0 s:CN = *.example.com
   i:C = US, O = Let's Encrypt, CN = R3
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Mar 28 08:25:24 2023 GMT; NotAfter: Jun 26 08:25:23 2023 GMT
---
Server certificate
～省略～
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1460 bytes and written 378 bytes
Verification error: unable to verify the first certificate
---
New, TLSv1.3, Cipher is TLS_CHACHA20_POLY1305_SHA256
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 21 (unable to verify the first certificate)
---
～省略～
```

Certificate chainが1つしかなく、今回設定した証明書を証明する上位の証明書が一切出てこないことがわかります。

> Verification error: unable to verify the first certificate

> Verify return code: 21 (unable to verify the first certificate)

初めの証明書、つまり設定したexample.comの証明書が認証できないよ！と怒られてしまいます。


## 中間証明書がある場合 (certFile: fullchain.pem)
```
reo@Ureo-Desktop:~$ echo | openssl s_client -connect sub.example.com:443 2> /dev/null
CONNECTED(00000003)
---
Certificate chain
 0 s:CN = *.example.com
   i:C = US, O = Let's Encrypt, CN = R3
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Mar 28 08:25:24 2023 GMT; NotAfter: Jun 26 08:25:23 2023 GMT
 1 s:C = US, O = Let's Encrypt, CN = R3
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Sep  4 00:00:00 2020 GMT; NotAfter: Sep 15 16:00:00 2025 GMT
 2 s:C = US, O = Internet Security Research Group, CN = ISRG Root X1
   i:O = Digital Signature Trust Co., CN = DST Root CA X3
   a:PKEY: rsaEncryption, 4096 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan 20 19:14:03 2021 GMT; NotAfter: Sep 30 18:14:03 2024 GMT
---
Server certificate
～省略～
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 4156 bytes and written 378 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_CHACHA20_POLY1305_SHA256
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
～省略～
```

今度はCertificate chainが3つに増え、ISRGが自分の証明書を証明してくれていることがわかるかと思います。

> Verification: OK

> Verify return code: 0 (ok)

認証ステータスもOKとなっています。

### サンプルドメインも存在

中間証明書をわざと設定していないサイトがあるので、このドメインに対してチェックするのもありかもしれないです。

incomplete-chain.badssl.com

注: 現在、そもそものドメインの証明書の有効期限が過ぎているため、別のエラーとなります。

## 必ずWebサーバーには全部入りの証明書を設定しよう！

opensslやcurlなど、限られたリソースで認証するアプリケーションではまず中間証明書を勝手に取得なんてことはしてくれないので、必ずopenssl等で正しくセットされているかチェックしましょう！

また、[SSL Server Test](https://www.ssllabs.com/ssltest/)などの外部ツールでも確認ができます。ぜひ活用しましょう！
