# BOLT #10: DNSブートストラップとアシストノードロケーション

## 概要

この仕様は、ドメインネームシステム (DNS) に基づくノード発見メカニズムについて説明します。その目的は二つあります。

 - ブートストラップ: ネットワーク内に既知の連絡先を持たないノードに対して、最初のノード発見手段を提供すること
 - アシストノードロケーション: 以前から知られているピアの現在のネットワークアドレスを発見するノードを支援すること

この仕様を実装するドメインネームサーバは _DNSシード_ と呼ばれ、RFC 1035<sup>[1](#ref-1)</sup>、3596<sup>[2](#ref-2)</sup>、2782<sup>[3](#ref-3)</sup> でそれぞれ規定されている `A`、`AAAA`、`SRV` タイプの DNS クエリに応答します。DNS サーバは _シードルートドメイン_ と呼ばれるサブドメインに対して権限を持ち、クライアントはそのサブドメインに対してクエリを発行できます。

サブドメインは、求める結果をさらに絞り込むための、ドット区切りの複数の _条件_ から構成されます。

## 目次

  * [DNSシードクエリ](#dns-seed-queries)
    * [クエリの意味](#query-semantics)
  * [返信の構築](#reply-construction)
  * [ポリシー](#policies)
  * [例](#examples)
  * [参考文献](#references)
  * [著者](#authors)

## DNSシードクエリ

クライアントは、`A`、`AAAA`、または `SRV` のクエリタイプを用いてクエリを発行してよく、その際にシードが返すべき結果に対する条件を指定できます。

クエリは、`l` キーが設定されているかどうかによって、_ワイルドカード_ クエリと _ノード_ クエリに区別されます。

### クエリの意味

条件はキーと値のペアであり、キーは 1 文字、それ以外の部分が値となります。DNS シードは、以下のキーと値のペアをサポートしなければなりません。

 - `r`: realm バイト
   - 返されるノードがサポートしていなければならない realm を指定するために使用する
   - デフォルト値: 0 (Bitcoin)
 - `a`: アドレスタイプ
   - [BOLT #7](07-routing-gossip.md) で定義されているタイプをビットインデックスとして使用するビットフィールド
   - `SRV` クエリで返すべきアドレスタイプを指定するために使用する
   - `SRV` クエリでのみ使用してよい
   - デフォルト値: 6 (すなわち `2 || 4`。IPv4 用にビット 1、IPv6 用にビット 2 が立っているため)
 - `l`: `node_id`
   - 特定のノードの bech32 エンコードされた `node_id`
   - ランダムな選択ではなく、単一のノードを問い合わせるために使用する
   - デフォルト値: null
 - `n`: 求める応答レコードの件数
   - デフォルト値: 25

条件は DNS シードクエリの中で、ドットで区切られた個々のサブドメインコンポーネントとして渡されます。

例えば、`r0.a2.n10.lseed.bitcoinstats.com` というクエリは、Bitcoin (`r0`) をサポートするノードの IPv4 (`a2`) レコードを 10 件 (`n10`) 返すことを意味します。

### 要件

DNSシード:
  - _シードルートドメイン_ から条件を「ツリーを上る」ように、つまり完全修飾ドメイン名を右から左へ評価しなければならない。
    - 例: 上記の例を評価する場合、まず `n10` を評価し、次に `a2`、最後に `r0` を評価する。
  - 同じ条件 (キー) が複数回指定された場合:
    - その条件について以前の値はすべて破棄し、新しい値を代わりに使用しなければならない。
      - 例: `n5.r0.a2.n10.lseed.bitcoinstats.com` の場合、結果は ~~`n10`~~、`a2`、`r0`、`n5` となる。
  - すべての条件に一致する結果を返すべきである。
  - ある条件によるフィルタリングを実装していない場合:
    - その条件を完全に無視してよい (すなわちシードのフィルタリングはベストエフォートにすぎない)。
  - `A` および `AAAA` クエリの場合:
    - [BOLT #1](01-messaging.md) で定義されているデフォルトポート 9735 で待ち受けているノードのみを返さなければならない。
  - `SRV` クエリの場合:
    - `SRV` レコードは _(hostname,port)_ のタプルを返すため、デフォルト以外のポートで待ち受けているノードを返してよい。
  - _ワイルドカード_ クエリを受け取った場合:
    - 着信接続を待ち受けているノードの IPv4 または IPv6 アドレスから、最大 `n` 個のランダムなサブセットを選択しなければならない。
  - _ノード_ クエリを受け取った場合:
    - `node_id` に一致するレコードがあればそれを選択し、そのノードに関連付けられたすべてのアドレスを返さなければならない。

クエリを行うクライアント:
  - 結果が任意の条件を満たしていることに依存してはならない。

### 返信の構築

結果は、クライアントのクエリタイプに一致するクエリタイプの返信としてシリアライズされます。例えば、`A`、`AAAA`、`SRV` の各クエリには、それぞれ `A`、`AAAA`、`SRV` の返信が返されます。さらに、返信には追加レコードを付加してもかまいません (例えば、返された `SRV` レコードに対応する `A` または `AAAA` レコードを追加するなど)。

`A` および `AAAA` クエリの場合、返信にはドメイン名と結果の IP アドレスが含まれます。

ドメイン名は、中間のリゾルバによってフィルタリングされないようにするため、クエリ内のドメインと一致しなければなりません。

`SRV` クエリの場合、返信は (_仮想ホスト名_, port) のタプルから構成されます。仮想ホスト名はシードルートドメインのサブドメインであり、ネットワーク内のノードを一意に識別します。これは `node_id` 条件をシードルートドメインの前に付加することで構成されます。

DNSシード:
  - 返信の追加セクションに、`SRV` エントリの IP アドレスを示す対応する `A` および `AAAA` レコードを併せて返してよい。
- 繰り返しのクエリを検出した場合、これらの追加レコードを省略してよい。
  - 理由: 返信のサイズが大きくなるため、中間のリゾルバによって返信がドロップされる可能性があるため。
- すべての条件に一致するエントリが存在しない場合:
  - 空の返信を返さなければならない。

## ポリシー

DNSシード:
  - TTL が 60 秒未満の返信を返してはならない。
  - 故障したノード、不安定なノード、スパム防止など、さまざまな理由でローカルビューからノードをフィルタリングしてよい。
  - ランダムなクエリ (すなわちシードルートドメインへのクエリ、および `SRV` クエリにおける `_nodes._tcp.` エイリアスへのクエリ) に対しては、Bitcoin DNS Seed ポリシー<sup>[4](#ref-4)</sup>に従い、既知の良好なノード集合から _ランダムかつ偏りのない_ サンプルを返答しなければならない。

## 例

`AAAA` レコードの問い合わせ:

	$ dig lseed.bitcoinstats.com AAAA
	lseed.bitcoinstats.com. 60      IN      AAAA    2a02:aa16:1105:4a80:1234:1234:37c1:9c9

`SRV` レコードの問い合わせ:

	$ dig lseed.bitcoinstats.com SRV
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 6331 ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qv2w3tledmzczw227nnkqrrltvmydl8gu4w4d70g9td7avke6nmz2tdefqp.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qtynyymv99pqf0r9cuexvvqtxrlgejuecf8myfsa96vcpflgll5cqmr2xsu.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4280 ln1qdfvlysfpyh96apy3w3qdwlu8jjkdhnuxa689ka540tnde6gnx86cf7ga2d.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4281 ln1qwf789tlcpe4n34649xrqllxt97whsvfk5pm07ggqms3vrjwdj3cu6332zs.lseed.bitcoinstats.com.

直前の例の最初の仮想ホスト名に対する `A` レコードの問い合わせ:

	$ dig ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com A
	ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com. 60 IN A 139.59.143.87

シードフィルタリングを用いて IPv4 ノード (`a2`) のみを問い合わせる:

	$dig a2.lseed.bitcoinstats.com SRV
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1q2jy22cg2nckgxttjf8txmamwe9rtw325v4m04ug2dm9sxlrh9cagrrpy86.lseed.bitcoinstats.com.
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1qfrkq32xayuq63anmc2zp5vtd2jxafhdzzudmuws0hvxshtgd2zd7jsqv7f.lseed.bitcoinstats.com.

シードフィルタリングを用いて Bitcoin (`r0`) をサポートする IPv6 ノード (`a4`) のみを問い合わせる:

	$dig r0.a4.lseed.bitcoinstats.com SRV
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwx3prnvmxuwsnaqhzwsrrpwy4pjf5m8fv4m8kcjkdvyrzymlcmj5dakwrx.lseed.bitcoinstats.com.
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwr7x7q2gvj7kwzzr7urqq9x7mq0lf9xn6svs8dn7q8gu5q4e852znqj3j7.lseed.bitcoinstats.com.

## 参考文献
- <a id="ref-1">[RFC 1035 - Domain Names](https://www.ietf.org/rfc/rfc1035.txt)</a>
- <a id="ref-2">[RFC 3596 - DNS Extensions to Support IP Version 6](https://tools.ietf.org/html/rfc3596)</a>
- <a id="ref-3">[RFC 2782 - A DNS RR for specifying the location of services (DNS SRV)](https://www.ietf.org/rfc/rfc2782.txt)</a>
- <a id="ref-4">[Expectations for DNS Seed operators](https://github.com/bitcoin/bitcoin/blob/master/doc/dnsseed-policy.md)</a>

## 著者

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
