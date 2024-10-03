# BOLT #10: DNSブートストラップとアシストノードロケーション

## 概要

この仕様は、ドメインネームシステム (DNS) に基づくノード発見メカニズムについて説明します。その目的は二つあります：

 - ブートストラップ：ネットワーク内に既知の連絡先がないノードに対する初期ノード発見を提供すること
 - アシストノードロケーション：以前に知られているピアの現在のネットワークアドレスの発見をサポートすること

この仕様を実装するドメインネームサーバは、_DNSシード_ と呼ばれ、RFC 1035<sup>[1](#ref-1)</sup>、3596<sup>[2](#ref-2)</sup>、および2782<sup>[3](#ref-3)</sup>で指定されているように、`A`、`AAAA`、または `SRV` タイプのDNSクエリに応答します。DNSサーバは、_シードルートドメイン_ と呼ばれるサブドメインに対して権限を持ち、クライアントはサブドメインをクエリできます。

サブドメインは、望ましい結果をさらに絞り込むための、ドットで区切られた複数の _条件_ で構成されます。

## 目次

  * [DNSシードクエリ](#dns-seed-queries)
    * [クエリの意味](#query-semantics)
  * [返信の構築](#reply-construction)
  * [ポリシー](#policies)
  * [例](#examples)
  * [参考文献](#references)
  * [著者](#authors)

## DNSシードクエリ

クライアントは、`A`、`AAAA`、または `SRV` クエリタイプを使用してクエリを発行し、シードが返すべき望ましい結果の条件を指定してもかまいません。

クエリは、`l` キーが設定されているかどうかに応じて、_ワイルドカード_ クエリと _ノード_ クエリを区別します。

### クエリの意味

条件はキーと値のペアであり、キーは一文字で、キーと値のペアの残りが値です。DNSシードがサポートしなければならないキーと値のペアは以下の通りです：

 - `r`: レルムバイト
   - 返されるノードがサポートしなければならないレルムを指定するために使用
   - デフォルト値：0 (Bitcoin)
 - `a`: アドレスタイプ
   - ビットフィールドで、[BOLT #7](07-routing-gossip.md) のタイプをビットインデックスとして使用
   - `SRV` クエリに対して返されるべきアドレスタイプを指定するために使用
   - `SRV` クエリにのみ使用してもかまいません
   - デフォルト値：6 (つまり、`2 || 4`、ビット1とビット2がIPv4とIPv6に対して設定されているため)
 - `l`: `node_id`
   - 特定のノードのbech32エンコードされた `node_id`
   - ランダムな選択ではなく、単一のノードを要求するために使用
   - デフォルト値：null
 - `n`: 望ましい返信レコードの数
   - デフォルト値：25

条件は、DNS シードクエリに個別のドットで区切られたサブドメインコンポーネントとして渡されます。

例えば、`r0.a2.n10.lseed.bitcoinstats.com` というクエリは、Bitcoin をサポートするノードの IPv4 (`a2`) レコードを 10 (`n10`) 件返すことを意味します。

### 要件

DNS シード：
  - _シードルートドメイン_ から条件を評価する際には、「ツリーを上る」ように、すなわち完全修飾ドメイン名を右から左に評価しなければなりません (MUST)。
    - 例：上記のケースを評価するには、まず `n10` を評価し、次に `a2`、最後に `r0` を評価します。
  - 条件 (キー) が複数回指定された場合：
    - その条件の以前の値をすべて破棄し (MUST)、代わりに新しい値を使用しなければなりません (MUST)。
      - 例：`n5.r0.a2.n10.lseed.bitcoinstats.com` の場合、結果は：~~`n10`~~、`a2`、`r0`、`n5`。
  - すべての条件に一致する結果を返すべきです (SHOULD)。
  - 特定の条件によるフィルタリングを実装していない場合：
    - 条件を完全に無視してもよいです (MAY)（つまり、シードフィルタリングはベストエフォートのみです）。
  - `A` および `AAAA` クエリの場合：
    - [BOLT #1](01-messaging.md) で定義されているデフォルトポート 9735 でリスニングしているノードのみを返さなければなりません (MUST)。
  - `SRV` クエリの場合：
    - 非デフォルトポートでリスニングしているノードを返してもよいです (MAY)、`SRV` レコードは _(hostname,port)_-タプルを返すためです。
  - _ワイルドカード_ クエリを受け取った場合：
    - 接続を待ち受けているノードの IPv4 または IPv6 アドレスのランダムなサブセットを最大 `n` 件選択しなければなりません (MUST)。
  - _ノード_ クエリを受け取った場合：
    - `node_id` に一致するレコードを選択し (MUST)、そのノードに関連付けられたすべてのアドレスを返さなければなりません (MUST)。

クエリを行うクライアント：
  - 結果が特定の条件を満たすことに依存してはなりません (MUST NOT)。

### 応答の構築

結果は、クライアントのクエリタイプに一致するクエリタイプで応答にシリアル化されます。例えば、`A`、`AAAA`、および `SRV` クエリはそれぞれ `A`、`AAAA`、および `SRV` 応答を生成します。さらに、応答は追加のレコードで拡張されることがあります（例えば、返された `SRV` レコードに一致する `A` または `AAAA` レコードを追加するため）。

`A` および `AAAA` クエリの場合、応答にはドメイン名と結果の IP アドレスが含まれます。

ドメイン名はクエリ内のドメインと一致しなければなりません (MUST)。これは中間のリゾルバによってフィルタリングされないようにするためです。

`SRV` クエリの場合、返信は (仮想ホスト名、ポート) の組で構成されます。仮想ホスト名は、ネットワーク内のノードを一意に識別するシードルートドメインのサブドメインです。これは `node_id` 条件をシードルートドメインに付加することで構成されます。

DNSシード：
  - 返信の追加セクションで `SRV` エントリの IP アドレスを示す対応する `A` および `AAAA` レコードを追加で返してもかまいません (MAY)。
  - 繰り返しのクエリを検出した場合、これらの追加レコードを省略してもかまいません (MAY)。
    - 理由：結果として得られる返信のサイズが大きいため、中間のリゾルバによって返信がドロップされる可能性があります。
  - すべての条件に一致するエントリがない場合：
    - 空の返信を返さなければなりません (MUST)。

## Policies

DNSシード：
  - TTL が 60 秒未満の返信を返してはなりません (MUST NOT)。
  - 故障したノード、不安定なノード、スパム防止などの理由でローカルビューからノードをフィルタリングしてもかまいません (MAY)。
  - ランダムなクエリ (すなわち、シードルートドメインおよび `SRV` クエリの `_nodes._tcp.` エイリアスへのクエリ) に対して、すべての既知の良好なノードのセットからランダムで偏りのないサンプルを返さなければなりません (MUST)。これは Bitcoin DNS Seed ポリシー<sup>[4](#ref-4)</sup>に従います。

## Examples

`AAAA` レコードのクエリ：

	$ dig lseed.bitcoinstats.com AAAA
	lseed.bitcoinstats.com. 60      IN      AAAA    2a02:aa16:1105:4a80:1234:1234:37c1:9c9

`SRV` レコードのクエリ：

	$ dig lseed.bitcoinstats.com SRV
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 6331 ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qv2w3tledmzczw227nnkqrrltvmydl8gu4w4d70g9td7avke6nmz2tdefqp.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qtynyymv99pqf0r9cuexvvqtxrlgejuecf8myfsa96vcpflgll5cqmr2xsu.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4280 ln1qdfvlysfpyh96apy3w3qdwlu8jjkdhnuxa689ka540tnde6gnx86cf7ga2d.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4281 ln1qwf789tlcpe4n34649xrqllxt97whsvfk5pm07ggqms3vrjwdj3cu6332zs.lseed.bitcoinstats.com.

前の例から最初の仮想ホスト名の `A` を問い合わせる：

	$ dig ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com A
	ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com. 60 IN A 139.59.143.87

シードフィルタリングを通じて IPv4 ノード（`a2`）のみを問い合わせる：

	$dig a2.lseed.bitcoinstats.com SRV
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1q2jy22cg2nckgxttjf8txmamwe9rtw325v4m04ug2dm9sxlrh9cagrrpy86.lseed.bitcoinstats.com.
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1qfrkq32xayuq63anmc2zp5vtd2jxafhdzzudmuws0hvxshtgd2zd7jsqv7f.lseed.bitcoinstats.com.

シードフィルタリングを通じて Bitcoin（`r0`）をサポートする IPv6 ノード（`a4`）のみを問い合わせる：

	$dig r0.a4.lseed.bitcoinstats.com SRV
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwx3prnvmxuwsnaqhzwsrrpwy4pjf5m8fv4m8kcjkdvyrzymlcmj5dakwrx.lseed.bitcoinstats.com.
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwr7x7q2gvj7kwzzr7urqq9x7mq0lf9xn6svs8dn7q8gu5q4e852znqj3j7.lseed.bitcoinstats.com.

## References
- <a id="ref-1">[RFC 1035 - Domain Names](https://www.ietf.org/rfc/rfc1035.txt)</a>
- <a id="ref-2">[RFC 3596 - DNS Extensions to Support IP Version 6](https://tools.ietf.org/html/rfc3596)</a>
- <a id="ref-3">[RFC 2782 - A DNS RR for specifying the location of services (DNS SRV)](https://www.ietf.org/rfc/rfc2782.txt)</a>
- <a id="ref-4">[Expectations for DNS Seed operators](https://github.com/bitcoin/bitcoin/blob/master/doc/dnsseed-policy.md)</a>

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
