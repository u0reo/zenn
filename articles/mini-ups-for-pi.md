---
title: "Raspberry Pi に最適なミニUPSを探し"
emoji: "🔋"
type: "tech"
topics: ["Raspberry Pi", "UPS", "USB"]
published: true
---

元々ラズパイ向けUPSスタックは存在します。

[Amazon.co.jp : raspberry pi ups](https://www.amazon.co.jp/s?k=raspberry+pi+ups&dc&ref=a9_asc_1)

ですがこの記事は

- そんなにお金かけたくない
- バッテリーでも長時間動作をしたい

という人向けの記事です。


### 過去に調べられているサイト

https://kitto-yakudatsu.com/archives/2669

Panasonicのモバイルバッテリーでいくつか調べられていて、QE-AL201がOK、それ以外はダメという結果に。


https://qiita.com/clses/items/3aec1b0bf975f614723c

割と初めのほうに発見したQiitaの記事。cheeroのDANBOARDバージョンがOK、RAVPowerのRP-PB060とROMOSSなどがダメなようです。


https://qiita.com/PINTO/items/7ca638b27a340e0c38da

こちらではELECOMのDE-M04L-3015がOK、SharllenとXiaomiとplus usとADATAがダメという結果に。

OKなエレコムにおいて

```
ケーブルは同時接続可能だが本当の意味での放電／充電の同時実施はされない
バッテリ残量がゼロになったときに初めて放電を止めて充電モードへ切り替わる
```

という注記がよくわからなかったものの、モバイルバッテリーの電池残量が100%ならばUPSとしても使えるということ？それともモバイルバッテリーの充放電がずっと繰り返されるということ？

いずれにしてもUPSとしては微妙な感じがするため除外とします。


https://blog.goo.ne.jp/shizuka2/e/bb929a906d34cd168a6e5f8d63c35ce5

こちらではPowerocksのST-PR-0CはOK、AnkerとAukeyとEnecycleがダメな模様です。

ですが、このPowerocksのモバイルバッテリーは日本では売られておらず、入手性が全くないため除外とします。


https://blog.goo.ne.jp/shizuka2/e/b96d53655670427d0681f041fcc6ae88

続報ではRAVPOWERのRP-PB043はOK、Ankerはダメだったそうです。

興味深いのは充電はmicroUSBでもType-Cでも良いが、出力はQuick Charge対応端子は不可で普通のUSB-A端子ならOKというところです。

急速充電系はネゴシエーションの問題か出力上限の問題かで瞬断を防ぐことはできないようです。


http://7th-chord.jp/sara_tsukiyono/index.php?cl=rp&rp=191213

こちらもかなり調べられている記事でPanasonicのQE-AL201とcheeroのDANBOARDバージョンとEasyAccのPB20000MSがOK、PanasonicのQE-AL101とRAVPowerのRP-PB060とROMOSSがダメなようです。

この記事でもZero Wを対象に書かれているため、低消費電流でも連続動作するという条件が加わっています。


https://note.com/k2sa/n/n51f8f15364d4

こちらではRAVPOWERのRP-PB058はOK、写真からAnkerはダメだったそうです。RAVPOWERは結構期待できそう。

そして、ドラレコのバックアップ電源でも流用できるとの情報が。とはいえ、UPSとして使うには少し工夫が必要そう。

[モバイルバッテリーを使ったドライブレコーダーへの駐車監視機能の追加方法 | LIFE with NX](https://with-x.com/archives/1261)

[Amazon.co.jp : ドライブレコーダー バックアップ電源](https://www.amazon.co.jp/s?k=%E3%83%89%E3%83%A9%E3%82%A4%E3%83%96%E3%83%AC%E3%82%B3%E3%83%BC%E3%83%80%E3%83%BC+%E3%83%90%E3%83%83%E3%82%AF%E3%82%A2%E3%83%83%E3%83%97%E9%9B%BB%E6%BA%90)


https://qiita.com/frost-tb-voo/items/8556a8ea207804bf6058

こちらがミニUPS系では最新であろう記事。eneloopのKBC-L2BSがOK、cheeroのCHE-071とTOPLANDがダメなようです。


### こんな解決法も

https://pa5m-med60.hatenablog.com/entry/2023/03/25/200000

パススルー対応だけども瞬断してしまうモバイルバッテリーなら**電源ラインにスーパーキャパシタ**を挟むという方法です。

急速充電対応ポートでは誤作動の元なので不可能かと思われます。どちらにしろ急速充電使用でのUPS動作は期待できませんが、百均の製品で実現できるのは魅力的。


### RAVPOWER に問い合わせたところ

動作報告の多かったRAVPOWER二は期待ができたので問い合わせてみることに。その回答がこちらです。

```
1．パススルー充電に対応している商品を生産停止のものも含め教えていただけますでしょうか。

弊社の下記の製品はパススルー充電に対応しています：
RP-PB186/RP-PB201/RP-PB125/RP-PB232

2.モバイルバッテリーへの給電開始時や給電停止時に瞬断がないものだと助かります。
スタッフに確認したところ、デバイスを認識するため、瞬断が発生したことはございます。

3.PDポートからも同じくモバイルバッテリーから機器への給電が続くか教えてほしいです
PDポートは輸出入でき、USB-Aポートは輸出しかできないので、
パススルー機能をご利用の場合には、PDポートがモバイルバッテリーへ給電、USB-Aポートからデバイスを充電できます。
```

この公式回答によりさらに4製品も候補が上がりました！

ですが、瞬断を確認しているとのことで確実とはいえません。。。原因は先ほど述べた急速充電かと思われますが、検証した方是非教えてください。

また、Amazonから姿を消しているため入手方法は少ないと思われますが、別名で販売しているとの噂もあります。(後述)


## 改めて候補一覧

最大入出力電力はどれか1ポートを給電に使った上でほかのすべてのポートに5V機器をつないだ時の理論上維持できる電力になります。

| 型番 | USB-C | USB-A | 最大入出力電力 | 発売日 | 最終中古確認日 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| KBC-L2BS  | 0 | 2 |  5W | 2010/10/21 | 2023/04/20 |
| cheero Power Plus 10400mAh DANBOARD Version | 0 | 2 | 5W | 2013/06/04 | 2023/04/20 |
| QE-AL201  | 0 | 2 |  9W | 2014/10/31 | 2023/04/20 |
| PB20000MS | 0 | 4 | 20W | 2015年ごろ | 不明 |
| RP-PB043  | 1※ | 2※ | 18W | 2016年ごろ | 2023/04/20 |
| RP-PB058  | 1 | 2 | 30W | 2019/07/09 | 2023/04/20 |
| RP-PB125  | 0 | 2 | 15W | 2019/07/04 | 2023/04/20 |
| RP-PB201  | 1 | 1 | 15W | 2020/02/07 | 2023/04/15 |
| RP-PB186  | 1 | 1 | 15W | 2020/09/24 | 2023/02/19 |
| RP-PB232  | 1 | 1 | 15W | 2021/05/21 | 2023/04/20 |

※PD非対応、USB-Aのうち1つはUPS運用不可

### eneloop KBC-L2BS

eneloopですので、三洋電機時代の製品になります。

出力は最大5Wで、mini USBによる充電は2.5Wまで、同梱のDCアダプタで5Wでの充電とZeroでない限り厳しそう。

発売は10年以上前ですが入手性はそこそこ。


### cheero Power Plus 10400mAh DANBOARD Version

そもそも有名なこともあり、ネット上で一番候補に挙がっていたもの。
しかし、最大入力電力は5WなためZeroでない限り厳しい気も。

有名ではあるものの10年前の製品とあって入手性は微妙。


### Panasonic QE-AL201

こちらはACプラグ付きでQiも対応しているもの。最大出力電力が10Wに届かないのは不安材料。

発売は10年前に近いものの入手性はそこそこ。分解記事もあるため技術があれば半永久的に使える？かも？


### EasyAcc PB20000MS

2入力4出力という変わり種のモバイルバッテリーなものの、最大20Wも入力でき、合計24Wも出力できるため、ラズパイの周辺機器の電力も余裕で賄えそう。

以下のサイトから各ポート1A位は出そうな雰囲気です。

https://push-switch.com/3409

ですが、入手性がかなり悪く、見つかればラッキー、、、


### RAVPOWER RP-PB043

公式回答にはなかったものの、瞬断なしも確認できているもの。UPS用途ではQCポートが使えず出力が1ポートに限られてしまうのが惜しい。

PD非対応なので、Type-Cを出力としてもUPS運用できる可能性あり。(未検証)

入力にもQCを使えるが、その時のUPSとしての動作は不明。入力をType-Cにした際は最大入出力電力は15Wに。

だいぶ古いうえに入手性はかなり悪い。


### RAVPOWER RP-PB058

こちらも公式回答にはなかったものの、瞬断なしも確認できているもの。

UPS運用の場合、Type-Cを出力に使えるかは不明。しかし、Type-Cを出力に使ってしまった場合はmicroUSBの入力が最大10Wなため、最大入出力電力は10Wに。

以下のサイトからは少なくとも**9W**は確実に出力が得られそう。

https://sidedish.org/archives/18219

中古も含め入手性は微妙。


### RAVPOWER RP-PB125

ACプラグ付きなため、USB充電器すら不要で運用できます。ですが、その分コンセントに余裕は必要です。microUSBでの充電もできますが、その場合の最大入出力電力は10Wに。

以下のサイトからは少なくとも**10W**は確実に出力は得られそう

https://tabihack.jp/ravpower-pb125/

古い製品とあって販売しているところは無さそう。でも、中古の入手性は良好。


### RAVPOWER RP-PB201

RP-PB186の上位互換品なので、ラズパイの電力ならばRP-PB186を入手するほうがよい気がします。

以下のサイトからは少なくとも**11W**は確実に出力が得られそう。

https://www.kobi-gadgetlife.com/ravpower-rppb201/

見た目や仕様がそっくりなこちらがおそらく同等品と思われるものの、レビューによると一応違うらしい。パススルー対応はレビューから散見されるが瞬断については不明。

https://amzn.asia/d/4HVFa43


### RAVPOWER RP-PB186

細かい仕様は以下が参考になりそうです。
パススルー運用する場合は仕様違反はあまり関係なく、**最大17.5W**まで供給できるようです。

https://hanpenblog.com/11851


ヨドバシなら新品が入手可能？中古は流通が少ない気がします。

https://www.yodobashi.com/product/100000001006498854/


### RAVPOWER RP-PB232

こちらは所持していたので試してみましたが、**パススルー充電可能**かつ**瞬断なし**でした。

とはいえ、30,000mAhの容量とPDにおいて最大90W出力、PPS(PDのオプション規格で出力電圧を小刻みに変更可能)の対応と、ラズパイのUPSとしてはオーバースペックな気もします。

以下のサイトからは少なくとも**11W**は確実に出力が得られそう。

https://www.kobi-gadgetlife.com/ravpower-rppb-232/

見た目や仕様がそっくりなこちらがおそらく同等品？(未検証)

https://amzn.asia/d/dJxAkpF


## 結論

実用上ベストなのは

1. EasyAcc PB20000MS
2. RAVPOWER RP-PB125

の2製品になりそうです。あとのRAVPOWERの15Wの製品たちでもよいですが、PDを無駄にしているため低コストという条件にそぐわない気がしますね。。。RP-PB232はバリバリ現役で使わせてもらってますしね。

ですが、PB20000MSは全然見かけないため、ひとまずはRP-PB125を買うことにします。後日報告いたします。
