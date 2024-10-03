# BOLT #11: Lightning支払いのためのインボイスプロトコル

Lightningを介した支払いを要求するための、シンプルで拡張可能なQRコード対応プロトコルです。

# 目次

  * [エンコーディングの概要](#encoding-overview)
  * [人間が読みやすい部分](#human-readable-part)
  * [データ部分](#data-part)
    * [タグ付きフィールド](#tagged-fields)
    * [機能ビット](#feature-bits)
  * [支払者／受取者の相互作用](#payer--payee-interactions)
    * [支払者／受取者の要件](#payer--payee-requirements)
  * [実装](#implementation)
  * [例](#examples)
  * [著者](#authors)

# エンコーディングの概要

Lightningインボイスのフォーマットは、すでにBitcoinのSegregated Witnessで使用されている
[bech32エンコーディング](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)を使用します。Lightningインボイスの長さを考えると手動入力が頻繁に行われる可能性は低いですが、その6文字のチェックサムは手動入力に最適化されているため、Lightningインボイスにも簡単に再利用できます。

URIスキームが必要な場合、現在の推奨はBOLT-11エンコーディングの前に「lightning:」をプレフィックスとして使用することです（注：「lightning://」ではありません）。また、Bitcoin支払いへのフォールバックとしては、BIP-21に従い「bitcoin:」を使用し、キー「lightning」と値をBOLT-11エンコーディングとすることです。

## 要件

ライターは：
   - Bech32で支払い要求をエンコードしなければなりません（BIP-0173参照）
   - QRコードには大文字を使用するべきです（BIP-0173参照）
   - BIP-0173で指定された90文字の制限を超えてもかまいません。

リーダーは：
  - BIP-0173で指定されたように、アドレスをBech32として解析しなければなりません（文字制限なしでも）。
	- 注：これはBIP-0173で指定された大文字の処理を含みます
  - チェックサムが間違っている場合：
    - 支払いを失敗させなければなりません。

# 人間が読みやすい部分

Lightningインボイスの人間が読みやすい部分は、次の2つのセクションで構成されます：
1. `prefix`: `ln` + BIP-0173通貨プレフィックス（例：Bitcoinメインネットの場合は`lnbc`、Bitcoinテストネットの場合は`lntb`、Bitcoinシグネットの場合は`lntbs`、Bitcoinレグテストの場合は`lnbcrt`）
1. `amount`: その通貨でのオプションの数値、その後にオプションの`multiplier`文字。ここでエンコードされる単位は支払い単位の「社会的」慣習であり、Bitcoinの場合は単位は「ビットコイン」であり、サトシではありません。

以下の `multiplier` 文字が定義されています：

* `m` (ミリ)：0.001 倍
* `u` (マイクロ)：0.000001 倍
* `n` (ナノ)：0.000000001 倍
* `p` (ピコ)：0.000000000001 倍

## 要件

ライター：

- `prefix` は、支払いを成功させるために必要な通貨を使ってエンコードしなければなりません (MUST)。
- 支払いを成功させるために特定の最小 `amount` が必要な場合：
  - その `amount` を含めなければなりません (MUST)。
  - `amount` は先頭に 0 を付けずに正の10進整数としてエンコードしなければなりません (MUST)。
  - `p` マルチプライヤーが使用される場合、`amount` の最後の小数は `0` でなければなりません (MUST)。
  - 可能な限り短い表現を使用するべきです (SHOULD)。最大のマルチプライヤーを使用するか、マルチプライヤーを省略します。

リーダー：

- `prefix` を理解できない場合：
  - 支払いを失敗させなければなりません (MUST)。
- `amount` が空の場合：
  - 支払者に金額が未指定であることを示すべきです (SHOULD)。
- それ以外の場合：
  - `amount` に数字以外が含まれているか、`multiplier` (上記の表を参照) 以外のものが続いている場合：
    - 支払いを失敗させなければなりません (MUST)。
  - `multiplier` が存在する場合：
    - 支払いに必要な金額を導出するために `amount` を `multiplier` 値で掛けなければなりません (MUST)。
    - マルチプライヤーが `p` であり、`amount` の最後の小数が 0 でない場合：
      - 支払いを失敗させなければなりません (MUST)。

## 理論的根拠

`amount` は人間が読める部分にエンコードされており、どれだけの金額が要求されているかを示すのに便利です。

寄付用アドレスには通常、関連する金額がないため、その場合 `amount` はオプションです。通常、提供されるものに対して最低限の支払いが必要です。

`p` マルチプライヤーは、ネットワーク上で転送できないサブミリサトシの金額を指定することを可能にします。HTLC はミリサトシ単位で表されるためです。末尾に `0` の小数を要求することで、`amount` がミリサトシの整数を表すことを保証します。

# データ部分

Lightning インボイスのデータ部分は複数のセクションで構成されています：

1. `timestamp`：1970 年からの秒数 (35 ビット、ビッグエンディアン)
1. 0 個以上のタグ付き部分
1. `signature`：上記の Bitcoin スタイルの署名 (520 ビット)

## 要件

ライターは以下を行う必要があります：
  - `timestamp` を 1970 年 1 月 1 日午前 0 時（UTC）からの秒数としてビッグエンディアンで設定しなければなりません。
  - `signature` を SHA2 256ビットハッシュの有効な 512ビット secp256k1 署名に設定しなければなりません。これは、UTF-8 バイトとして表現された人間が読める部分と、データ部分（署名を除く）を連結し、次のバイト境界までデータをパッドするために 0 ビットを追加し、リカバリー ID（0, 1, 2, または 3）を含む末尾のバイトを追加したものです。

リーダーは以下を行う必要があります：
  - `signature` が有効であることを確認しなければなりません（以下で指定される `n` タグ付きフィールドを参照）。

## 理論的根拠

`signature` は、SHA2 標準が実際にはビット境界でのハッシュをサポートしているにもかかわらず、正確なバイト数をカバーします。これは広く実装されていないためです。リカバリー ID により公開鍵のリカバリーが可能になり、支払い先ノードのアイデンティティを暗示できます。

## タグ付きフィールド

各タグ付きフィールドは以下の形式です：

1. `type` (5 ビット)
1. `data_length` (10 ビット、ビッグエンディアン)
1. `data` (`data_length` x 5 ビット)

タグ付きフィールドの `data` の最大長は `data_length` の最大値によって制限されます。これは 1023 x 5 ビット、つまり 639 バイトです。

現在定義されているタグ付きフィールドは以下の通りです：

* `p` (1): `data_length` 52。256ビット SHA256 payment_hash。このプレイメージは支払いの証明を提供します。
* `s` (16): `data_length` 52。この 256ビットの秘密は、転送ノードが支払い受取人をプロービングするのを防ぎます。
* `d` (13): `data_length` 可変。支払いの目的の短い説明 (UTF-8)、例：「1 cup of coffee」または「ナンセンス 1杯」
* `m` (27): `data_length` 可変。支払いに添付する追加のメタデータ。このフィールドのサイズは最大ホップペイロードサイズによって制限されます。長いメタデータフィールドは最大ルート長を減少させます。
* `n` (19): `data_length` 53。支払い先ノードの 33 バイトの公開鍵
* `h` (23): `data_length` 52。支払いの目的の 256ビットの説明 (SHA256)。これは 639 バイトを超える関連説明にコミットするために使用されますが、その場合の説明の転送メカニズムは転送固有であり、ここでは定義されていません。
* `x` (6): `data_length` 可変。`expiry` 時間を秒単位で（ビッグエンディアン）指定します。指定されていない場合のデフォルトは 3600（1 時間）です。
* `c` (24): `data_length` 可変。ルートの最後の HTLC に使用する `min_final_cltv_expiry_delta`。指定されていない場合のデフォルトは 18 です。
* `f` (9): `data_length` バージョンに依存して可変。フォールバックオンチェーンアドレス：Bitcoin の場合、これは 5 ビットの `version` で始まり、witness プログラムまたは P2PKH または P2SH アドレスを含みます。
* `r` (3): `data_length` 可変。プライベートルートのための追加のルーティング情報を含む 1 つ以上のエントリが含まれる場合があります。`r` フィールドは複数存在することがあります。
   * `pubkey` (264 ビット)
   * `short_channel_id` (64 ビット)
   * `fee_base_msat` (32 ビット、ビッグエンディアン)
   * `fee_proportional_millionths` (32 ビット、ビッグエンディアン)
   * `cltv_expiry_delta` (16 ビット、ビッグエンディアン)
* `9` (5): `data_length` 可変。この支払いを受け取るためにサポートまたは必要とされる機能を含む 1 つ以上の 5 ビット値。
  [Feature Bits](#feature-bits) を参照してください。

### 要件

ライターは以下を満たす必要があります：

- 正確に 1 つの `p` フィールドを含めなければなりません。
- 正確に 1 つの `s` フィールドを含めなければなりません。
- 支払いの見返りとして提供される `payment_preimage` の SHA2 256 ビットハッシュを `payment_hash` に設定しなければなりません。
- 正確に 1 つの `d` または正確に 1 つの `h` フィールドを含めなければなりません。
  - `d` が含まれている場合：
    - `d` を有効な UTF-8 文字列に設定しなければなりません。
    - 支払いの目的を完全に説明することを推奨します。
  - `h` が含まれている場合：
    - ハッシュ化された説明のプレイメージを `h` で利用可能にしなければなりません（手段は指定されていません）。
      - 支払いの目的を完全に説明することを推奨します。
- `x` フィールドを含めることができます。
  - `x` が含まれている場合：
    - 最小の `data_length` を使用することを推奨します。
- `c` フィールド（`min_final_cltv_expiry_delta`）を含めることを推奨します。
  - ルートの最後の HTLC に対して受け入れる最小の `cltv_expiry` に `c` を設定しなければなりません。
  - 最小の `data_length` を使用することを推奨します。
- `n` フィールドを含めることができます。（そうでない場合は署名のリカバリが必要です）
  - `n` を `signature` を作成するために使用された公開鍵に設定しなければなりません。
- 1 つ以上の `f` フィールドを含めることができます。
  - ビットコインの支払いの場合：
    - 有効なウィットネスバージョンとプログラム、または `17` に続く公開鍵ハッシュ、または `18` に続くスクリプトハッシュに `f` フィールドを設定しなければなりません。
- 公開鍵に関連付けられた公開チャネルがない場合：
  - 少なくとも 1 つの `r` フィールドを含めなければなりません。
    - `r` フィールドは、公開ノードから最終目的地への転送ルートを示す 1 つ以上の順序付けられたエントリを含まなければなりません。
      - 注：各エントリについて、`pubkey` はチャネルの開始ノードの ID であり、`short_channel_id` はチャネルを識別するための短いチャネル ID フィールドです。また、`fee_base_msat`、`fee_proportional_millionths`、`cltv_expiry_delta` は [BOLT #7](07-routing-gossip.md#the-channel_update-message) で指定されています。
  - 複数のルーティングオプションを提供するために、複数の `r` フィールドを含めることができます。
- `9` にゼロ以外のビットが含まれている場合：
  - 最小の `data_length` を使用することを推奨します。
- それ以外の場合：
  - `9` フィールドを完全に省略しなければなりません。
- フィールドデータを 5 ビットの倍数に 0 でパディングしなければなりません。
- ライターが任意のフィールドタイプを複数提供する場合：
  - 最も優先されるフィールドを最初に指定し、続いて優先度の低いフィールドを順に指定しなければなりません。

読者は以下を行う必要があります：

- 未知のフィールド、または未知の `version` を持つ `f` フィールド、または `data_length` がそれぞれ 52、52、52、53 でない `p`、`h`、`s`、`n` フィールドをスキップしなければなりません。
- `9` フィールドに未知の _奇数_ ビットが含まれており、それがゼロでない場合：
  - ビットを無視しなければなりません。
- `9` フィールドに未知の _偶数_ ビットが含まれており、それがゼロでない場合：
  - 支払いを失敗させなければなりません。
  - 未知のビットをユーザに示すべきです。
- `h` フィールドの SHA2 256ビットハッシュが、ハッシュ化された説明と完全に一致することを確認しなければなりません。
- 有効な `n` フィールドが提供されている場合：
  - 署名のリカバリを行う代わりに、`n` フィールドを使用して署名を検証しなければなりません。
- 有効な `s` フィールドがある場合：
  - それを [`payment_secret`](04-onion-routing.md#tlv_payload-payload-format) として使用しなければなりません。
- `c` フィールド (`min_final_cltv_expiry_delta`) が提供されていない場合：
  - 支払いを行う際に、少なくとも 18 の有効期限デルタを使用しなければなりません。
- `m` フィールドが提供されている場合：
  - それを [`payment_metadata`](04-onion-routing.md#tlv_payload-payload-format) として使用しなければなりません。

### 理論的根拠

型と長さのフォーマットは、将来の拡張を後方互換性のあるものにします。`data_length` は常に 5 ビットの倍数で、エンコードとデコードが容易です。読者は、異なる長さのフィールドを無視します。期待されるフィールドは変更される可能性があります。

`p` フィールドは現在の 256ビットの支払いハッシュをサポートしますが、将来の仕様では異なる長さの新しいバリアントを追加する可能性があります。その場合、ライターは古いバリアントと新しいバリアントの両方をサポートでき、古い読者は正しい長さでないバリアントを無視します。

`d` フィールドはインラインの説明を許可しますが、複雑な注文には不十分かもしれません。したがって、`h` フィールドは要約を許可します。ただし、説明が提供される方法はまだ未定であり、おそらくトランスポートに依存します。`h` フォーマットは将来的に長さを変更することで変わる可能性があるため、読者はそれが 256 ビットでない場合は無視します。

`m` フィールドは支払いにメタデータを添付することを許可します。これは、受取人が支払いのコンテキストを保持しないアプリケーションをサポートします。

`n` フィールドは、署名の復元を必要とせずに、宛先ノード ID を明示的に指定するために使用できます。

`x` フィールドは、支払いが拒否される可能性がある場合の警告を提供します。これは主に混乱を避けるためです。デフォルトは、ほとんどの支払いに対して合理的であり、必要に応じてオンチェーン支払いのための十分な時間を確保するように選ばれています。

`c` フィールドは、宛先ノードが受信する HTLC に対して特定の最小 CLTV 期限を要求する方法を提供します。宛先ノードは、デフォルトよりも高く、より保守的な値を要求するためにこれを使用することがあります（手数料の見積もりポリシーや時間ロックに対する感度に応じて）。ルート内のリモートノードは、`channel_update` メッセージで必要な `cltv_expiry_delta` を指定し、これを常に更新することができます。

`f` フィールドはオンチェーンフォールバックを可能にします。ただし、これは小額または時間に敏感な支払いには意味がないかもしれません。新しいアドレス形式が登場する可能性があるため、複数の `f` フィールド（暗黙の優先順序で）が移行を助け、バージョン 19-31 の `f` フィールドは読み手によって無視されます。

`r` フィールドは限定的なルーティング支援を可能にします。指定されたように、これはプライベートチャネルを使用するための最小限の情報のみを許可しますが、将来的には部分的な知識ルーティングを支援することもできます。

### 支払い説明に関するセキュリティ考慮事項

支払い説明はユーザ定義であり、レンダリングと永続化の両方のプロセス中に注入攻撃の可能性を提供します。

支払い説明は、HTML/JavaScript コンテキスト（または他の動的に解釈されるレンダリングフレームワーク）で表示される前に常にサニタイズされるべきです。実装者は、支払い説明をデコードして表示する際に、反射型 XSS 攻撃の可能性に特に注意を払うべきです。すべての検証、確認、およびサニタイズプロセスが正常に完了するまで、支払い要求の内容を楽観的にレンダーすることは避けてください。

さらに、SQL や他の動的に解釈されるクエリ言語をサポートする永続化エンジンにおける注入脆弱性から保護するために、準備されたステートメント、入力検証、および/またはエスケープを使用することを検討してください。

* [Stored and Reflected XSS Prevention](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)
* [DOM-based XSS Prevention](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL Injection Prevention](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

[Little Bobby Tables](https://xkcd.com/327/) のようにならないようにしましょう。

## Feature Bits

Feature bits は前方互換性と後方互換性を可能にし、_it's ok to be odd_ ルールに従います。`9` フィールドで使用するのに適した機能は [BOLT 9](09-features.md) に記載されています。

フィールドはビッグエンディアンです。最下位ビットは 0 番で、これは _even_ であり、次に重要なビットは 1 番で、これは _odd_ です。

### Requirements

ライター：
  - [BOLT 9 origin node requirements](09-features.md#requirements) に準拠した機能ベクターに `9` フィールドを設定しなければなりません。

リーダー：
  - 機能ベクターがすべての既知の推移的機能依存関係を設定していない場合：
    - 支払いを試みてはなりません。
  - インボイスで `basic_mpp` 機能が提供されている場合：
    - [Basic multi-part payments](04-onion-routing.md#basic-multi-part-payments) を使用してもかまいません。
  - それ以外の場合：
    - [Basic multi-part payments](04-onion-routing.md#basic-multi-part-payments) を使用してはなりません。

# Payer / Payee Interactions

これらは一般的に Lightning BOLT シリーズの残りによって定義されていますが、[BOLT #4](04-onion-routing.md#requirements-2) では、支払受取人が予想される `amount` の最大 2 倍を受け入れるべきであると指定されていることに注意する価値があります。これにより、支払者は小さな変動を加えることで支払いを追跡しにくくすることができます。

意図としては、支払者が署名から受取人のノード ID を回復し、手数料、有効期限、ブロックタイムアウトなどの条件が受け入れ可能であることを確認した後、支払いを試みます。最終ノードに到達するために必要であれば、`r` フィールドを使用してルーティング情報を補強することができます。

支払いが成功したが後で紛争が発生した場合、支払者は受取人からの署名付きオファーと成功した支払いの両方を証明することができます。

## 支払者 / 受取者の要件

支払者：

- `timestamp` プラス `expiry` が経過した後：
  - 支払いを試みるべきではありません。
- それ以外の場合：
  - Lightning 支払いが失敗した場合：
    - 理解できる最初の `f` フィールドに示されたアドレスを使用して支払いを試みてもかまいません。
- `r` フィールドで指定されたチャネルのシーケンスを使用して受取者にルートすることができます。
- 支払いを開始する前に、手数料の金額と支払いのタイムアウトを考慮するべきです。
- スキップしなかった最初の `p` フィールドを支払いハッシュとして使用するべきです。

受取者：

- `timestamp` プラス `expiry` が経過した後：
  - 支払いを受け入れるべきではありません。

# 実装

https://github.com/rustyrussell/lightning-payencode

# 例

注意：以下のすべての例は `priv_key`=`e126f68f7eafcc8b74f54d269fe206be715000f94dac067d1c04a8ca3b2db734` で署名されています。
特に記載がない限り、すべての請求書には `payment_secret`=`1111111111111111111111111111111111111111111111111111111111111111` が含まれています。

> ### 支払いハッシュ 0001020304050607080900010203040506070809000102030405060708090102 を使用して、任意の金額を私 @03e7156ae33b0a208d0744199163177e909e80176e55d97a2f221ede0f934dd9ad に寄付してください
> lnbc1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpl2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq9qrsgq357wnc5r2ueh7ck6q93dj32dlqnls087fxdwk8qakdyafkq3yap9us6v52vjjsrvywa6rt52cm9r9zqt8r2t7mlcwspyetp5h2tztugp9lfyql

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs`: 支払いシークレット 1111111111111111111111111111111111111111111111111111111111111111
* `p`: 支払いハッシュ
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypq`: 支払いハッシュ 0001020304050607080900010203040506070809000102030405060708090102
* `d`: 短い説明
  * `pl`: `data_length` (`p` = 1, `l` = 31; 1 * 32 + 31 == 63)
  * `2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq`: 'このプロジェクトをサポートしてください'
* `9`: 機能
  * `qr`: `data_length` (`q` = 0, `r` = 3; 0 * 32 + 3 == 3)
  * `sgq`: b100000100000000
* `357wnc5r2ueh7ck6q93dj32dlqnls087fxdwk8qakdyafkq3yap9us6v52vjjsrvywa6rt52cm9r9zqt8r2t7mlcwspyetp5h2tztugp`: 署名
* `9lfyql`: Bech32 チェックサム
* 署名の内訳：
  * `8d3ce9e28357337f62da0162d9454df827f83cfe499aeb1c1db349d4d81127425e434ca29929406c23bba1ae8ac6ca32880b38d4bf6ff874024cac34ba9625f1` 署名データの 16 進数 (32 バイト r, 32 バイト s)
  * `1` (int) 署名に含まれるリカバリーフラグ
  * `6c6e62630b25fe64500d04444444444444444444444444444444444444444444444444444444444444444021a00008101820283038404800081018202830384048000810182028303840480810343f506c6561736520636f6e736964657220737570706f7274696e6720746869732070726f6a6563740500e08000` 署名用データの 16 進数 (プレフィックス + セパレータ後のデータから署名開始まで)
  * `6daf4d488be41ce7cbb487cab1ef2975e5efcea879b20d421f0ef86b07cbb987` プレイメージの SHA256 の 16 進数

> ### 同じピアに1分以内にコーヒー1杯のために $3 を送ってください
> lnbc2500u1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5xysxxatsyp3k7enxv4jsxqzpu9qrsgquk0rl77nj30yxdy8j9vdx85fkpmdla2087ne0xh8nhedh8w27kyke0lp53ut353s06fv3qfegext0eh0ymjpf39tuven09sam30g4vgpfna3rh

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `2500u`: 金額 (2500 マイクロビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット...
* `p`: 支払いハッシュ...
* `d`: 短い説明
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `xysxxatsyp3k7enxv4js`: '1 cup coffee'
* `x`: 有効期限
  * `qz`: `data_length` (`q` = 0, `z` = 2; 0 * 32 + 2 == 2)
  * `pu`: 60 秒 (`p` = 1, `u` = 28; 1 * 32 + 28 == 60)
* `9`: 機能
  * `qr`: `data_length` (`q` = 0, `r` = 3; 0 * 32 + 3 == 3)
  * `sgq`: b100000100000000
* `uk0rl77nj30yxdy8j9vdx85fkpmdla2087ne0xh8nhedh8w27kyke0lp53ut353s06fv3qfegext0eh0ymjpf39tuven09sam30g4vgp`: 署名
* `fna3rh`: Bech32 チェックサム
* 署名の内訳：
  * `e59e3ffbd3945e4334879158d31e89b076dff54f3fa7979ae79df2db9dcaf5896cbfe1a478b8d2307e92c88139464cb7e6ef26e414c4abe33337961ddc5e8ab1` 署名データの 16 進数 (32 バイト r, 32 バイト s)
  * `1` (int) 署名に含まれるリカバリーフラグ
  * `6c6e626332353030750b25fe64500d04444444444444444444444444444444444444444444444444444444444444444021a000081018202830384048000810182028303840480008101820283038404808103414312063757020636f66666565030041e140382000` 署名用データの 16 進数 (プレフィックス + セパレータ後のデータから署名の開始まで)
  * `047e24bf270b25d42a56d57b2578faa3a10684641bab817c2851a871cb41dbc0` プレイメージの SHA256 の 16 進数

> ### 同じピアに1分以内にナンセンス 1杯のために 0.0025 BTC を送ってください
> lnbc2500u1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpquwpc4curk03c9wlrswe78q4eyqc7d8d0xqzpu9qrsgqhtjpauu9ur7fw2thcl4y9vfvh4m9wlfyz2gem29g5ghe2aak2pm3ps8fdhtceqsaagty2vph7utlgj48u0ged6a337aewvraedendscp573dxr

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `2500u`: 金額 (2500 マイクロビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット...
* `p`: 支払いハッシュ...
* `d`: 短い説明
  * `pq`: `data_length` (`p` = 1, `q` = 0; 1 * 32 + 0 == 32)
  * `uwpc4curk03c9wlrswe78q4eyqc7d8d0`: 'ナンセンス 1杯'
* `x`: 有効期限
  * `qz`: `data_length` (`q` = 0, `z` = 2; 0 * 32 + 2 == 2)
  * `pu`: 60 秒 (`p` = 1, `u` = 28; 1 * 32 + 28 == 60)
* `9`: 機能
  * `qr`: `data_length` (`q` = 0, `r` = 3; 0 * 32 + 3 == 3)
  * `sgq`: b100000100000000
* `htjpauu9ur7fw2thcl4y9vfvh4m9wlfyz2gem29g5ghe2aak2pm3ps8fdhtceqsaagty2vph7utlgj48u0ged6a337aewvraedendscp`: 署名
* `573dxr`: Bech32 チェックサム
* 署名の内訳：
  * `bae41ef385e0fc972977c7ea42b12cbd76577d2412919da8a8a22f9577b6507710c0e96dd78c821dea16453037f717f44aa7e3d196ebb18fbb97307dcb7336c3` 署名データの 16 進数 (32 バイト r, 32 バイト s)
  * `1` (int) 署名に含まれるリカバリーフラグ
  * `6c6e626332353030750b25fe64500d04444444444444444444444444444444444444444444444444444444444444444021a000081018202830384048000810182028303840480008101820283038404808103420e3838ae383b3e382b92031e69daf30041e14038200` 署名用データの 16 進数 (プレフィックス + セパレータ後から署名開始までのデータ)
  * `f140d992ba419578ba9cfe1af85f92df90a76f442fb5e6e09b1f0582534ba87d` プレイメージの SHA256 の 16 進数

> ### さて、リスト全体のために $24 を送信します (ハッシュ化済み)
> lnbc20m1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqs9qrsgq7ea976txfraylvgzuxs8kgcw23ezlrszfnh8r6qtfpr6cxga50aj6txm9rxrydzd06dfeawfk6swupvz4erwnyutnjq7x39ymw6j38gp7ynn44

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `20m`: 金額 (20 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット...
* `p`: 支払いハッシュ...
* `h`: タグ付きフィールド：説明のハッシュ
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `8yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqs`: 'チョコレートケーキ 1 切れ、アイスクリームコーン 1 個、ピクルス 1 個、スイスチーズ 1 切れ、サラミ 1 切れ、ロリポップ 1 個、チェリーパイ 1 切れ、ソーセージ 1 本、カップケーキ 1 個、スイカ 1 切れ' の SHA256 (SHA256 16 進数：`3925b6f67e2c340036ed12093dd44e0368df1b6ea26c53dbe4811f58fd5db8c1`)
* `9`: 機能
  * `qr`: `data_length` (`q` = 0, `r` = 3; 0 * 32 + 3 == 3)
  * `sgq`: b100000100000000
* `7ea976txfraylvgzuxs8kgcw23ezlrszfnh8r6qtfpr6cxga50aj6txm9rxrydzd06dfeawfk6swupvz4erwnyutnjq7x39ymw6j38gp`: 署名
* `7ynn44`: Bech32 チェックサム
* 署名の内訳：
  * `f67a5f696648fa4fb102e1a07b230e54722f8e024cee71e80b4847ac191da3fb2d2cdb28cc32344d7e9a9cf5c9b6a0ee0582ae46e9938b9c81e344a4dbb5289d` 署名データの 16 進数 (32 バイト r, 32 バイト s)
  * `1` (int) 署名に含まれるリカバリーフラグ
  * `6c6e626332306d0b25fe64500d04444444444444444444444444444444444444444444444444444444444444444021a000081018202830384048000810182028303840480008101820283038404808105c343925b6f67e2c340036ed12093dd44e0368df1b6ea26c53dbe4811f58fd5db8c10280704000` 署名用データの 16 進数 (プレフィックス + セパレータ後から署名開始までのデータ)
  * `e2ffa444e2979edb639fbdaa384638683ba1a5240b14dd7a150e45a04eea261d` プレイメージの SHA256 の 16 進数


> ### 同じく、テストネットで、フォールバックアドレス mk2QpYatsKicvFVuTAQLBryyccRXMUaGHP を使用
> lntb20m1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygshp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfpp3x9et2e20v6pu37c5d9vax37wxq72un989qrsgqdj545axuxtnfemtpwkc45hx9d2ft7x04mt8q7y6t0k2dge9e7h8kpy9p34ytyslj3yu569aalz2xdk8xkd7ltxqld94u8h2esmsmacgpghe9k8

内訳：

* `lntb`: プレフィックス、Bitcoin テストネット上の Lightning
* `20m`: 金額 (20 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `h`: タグ付きフィールド：説明のハッシュ...
* `s`: 支払いシークレット...
* `p`: 支払いハッシュ...
* `f`: タグ付きフィールド：フォールバックアドレス
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `3` = 17、したがって P2PKH アドレス
  * `x9et2e20v6pu37c5d9vax37wxq72un98`: 160ビット P2PKH アドレス
* `9`: 機能...
* `dj545axuxtnfemtpwkc45hx9d2ft7x04mt8q7y6t0k2dge9e7h8kpy9p34ytyslj3yu569aalz2xdk8xkd7ltxqld94u8h2esmsmacgp`: 署名
* `ghe9k8`: Bech32 チェックサム
* 署名の内訳：
  * `6ca95a74dc32e69ced6175b15a5cc56a92bf19f5dace0f134b7d94d464b9f5cf6090a18d48b243f289394d17bdf89466d8e6b37df5981f696bc3dd5986e1bee1` 署名データの 16進数 (32バイト r, 32バイト s)
  * `1` (int) 署名に含まれるリカバリーフラグ
  * `6c6e746232306d0b25fe64500d044444444444444444444444444444444444444444444444444444444444444442e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc608043400010203040506070809000102030405060708090001020304050607080901020484313172b5654f6683c8fb146959d347ce303cae4ca728070400` 署名用データの 16進数 (プレフィックス + セパレータ後のデータから署名開始まで)
  * `33bc6642a336097c74299cadfdfdd2e4884a555cf1b4fda72b095382d473d795` プレイメージの SHA256 の 16進数

> ### メインネットで、フォールバックアドレス 1RustyRX2oai4EYYDpQGWvEL62BBGqN9T を使用し、ノード 029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255 と 039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255 を経由する追加のルーティング情報を含む
> lnbc20m1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqsfpp3qjmp7lwpagxun9pygexvgpjdc4jdj85fr9yq20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzq9qrsgqdfjcdk6w3ak5pca9hwfwfh63zrrz06wwfya0ydlzpgzxkn5xagsqz7x9j4jwe7yj7vaf2k9lqsdk45kts2fd0fkr28am0u4w95tt2nsq76cqw0

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `20m`: 金額 (20 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット...
* `p`: 支払いハッシュ...
* `h`: タグ付きフィールド：説明のハッシュ...
* `f`: タグ付きフィールド：フォールバックアドレス
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `3` = 17、したがって P2PKH アドレス
  * `qjmp7lwpagxun9pygexvgpjdc4jdj85f`: 160ビット P2PKH アドレス
* `r`: タグ付きフィールド：ルート情報
  * `9y`: `data_length` (`9` = 5, `y` = 4; 5 * 32 + 4 == 164)
    * `q20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzq`:
      * 公開鍵: `029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`: 66051x263430x1800
      * `fee_base_msat`: 1 ミリサトシ
      * `fee_proportional_millionths`: 20
      * `cltv_expiry_delta`: 3
      * 公開鍵: `039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`: 197637x395016x2314
      * `fee_base_msat`: 2 ミリサトシ
      * `fee_proportional_millionths`: 30
      * `cltv_expiry_delta`: 4
* `9`: 機能...
* `dfjcdk6w3ak5pca9hwfwfh63zrrz06wwfya0ydlzpgzxkn5xagsqz7x9j4jwe7yj7vaf2k9lqsdk45kts2fd0fkr28am0u4w95tt2nsq`: 署名
* `76cqw0`: Bech32 チェックサム
* 署名の内訳：
  * `6a6586db4e8f6d40e3a5bb92e4df5110c627e9ce493af237e20a046b4e86ea200178c59564ecf892f33a9558bf041b6ad2cb8292d7a6c351fbb7f2ae2d16b54e` 署名データの 16 進数 (32 バイト r, 32 バイト s)
  * `0` (int) 署名に含まれるリカバリーフラグ
  * `6c6e626332306d0b25fe64500d04444444444444444444444444444444444444444444444444444444444444444021a000081018202830384048000810182028303840480008101820283038404808105c343925b6f67e2c340036ed12093dd44e0368df1b6ea26c53dbe4811f58fd5db8c104843104b61f7dc1ea0dc99424464cc4064dc564d91e891948053c07520370aa69fe3d258878e8863ef9ce408c0c1f9ef52b86fc291ef18ee4aa020406080a0c0e1000000002000000280006073c07520370aa69fe3d258878e8863ef9ce408c0c1f9ef52b86fc291ef18ee4aa06080a0c0e101214000000040000003c00080500e08000` 署名用データの 16 進数 (プレフィックス + セパレータ以降のデータの署名開始まで)
  * `b342d4655b984e53f405fe4d872fb9b7cf54ba538fcd170ed4a5906a9f535064` プレイメージの SHA256 の 16 進数


> ### メインネットで、フォールバック (P2SH) アドレス 3EktnHQD7RiAE6uzMj2ZifT9YgRrkSgzQX
> lnbc20m1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygshp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppj3a24vwu6r8ejrss3axul8rxldph2q7z99qrsgqz6qsgww34xlatfj6e3sngrwfy3ytkt29d2qttr8qz2mnedfqysuqypgqex4haa2h8fx3wnypranf3pdwyluftwe680jjcfp438u82xqphf75ym

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `20m`: 金額 (20 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット...
* `h`: タグ付きフィールド：説明のハッシュ...
* `p`: 支払いハッシュ...
* `f`: タグ付きフィールド：フォールバックアドレス
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `j` = 18、したがって P2SH アドレス
  * `3a24vwu6r8ejrss3axul8rxldph2q7z9`: 160ビット P2SH アドレス
* `9`: 機能...
* `z6qsgww34xlatfj6e3sngrwfy3ytkt29d2qttr8qz2mnedfqysuqypgqex4haa2h8fx3wnypranf3pdwyluftwe680jjcfp438u82xqp`: 署名
* `hf75ym`: Bech32 チェックサム
* 署名の内訳：
  * `16810439d1a9bfd5a65acc61340dc92448bb2d456a80b58ce012b73cb5202438020500c9ab7ef5573a4d174c811f669885ae27f895bb3a3be52c243589f87518` 署名データの 16 進数 (32 バイト r, 32 バイト s)
  * `1` (int) 署名に含まれるリカバリーフラグ
  * `6c6e626332306d0b25fe64500d044444444444444444444444444444444444444444444444444444444444444442e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc608043400010203040506070809000102030405060708090001020304050607080901020484328f55563b9a19f321c211e9b9f38cdf686ea0784528070400` 署名用データの 16 進数 (プレフィックス + セパレータ後から署名開始までのデータ)
  * `9e93321a775f7dffdca03e61d1ac6e0e356cc63cecd3835271200c1e5b499d29` プレイメージの SHA256 の 16 進数

> ### メインネットで、フォールバック (P2WPKH) アドレス bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4
> lnbc20m1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygshp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppqw508d6qejxtdg4y5r3zarvary0c5xw7k9qrsgqt29a0wturnys2hhxpner2e3plp6jyj8qx7548zr2z7ptgjjc7hljm98xhjym0dg52sdrvqamxdezkmqg4gdrvwwnf0kv2jdfnl4xatsqmrnsse


* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `20m`: 金額 (20 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット...
* `h`: タグ付きフィールド：説明のハッシュ...
* `p`: 支払いハッシュ...
* `f`: タグ付きフィールド：フォールバックアドレス
  * `pp`: `data_length` ( `p` = 1; 1 * 32 + 1 == 33)
  * `q`: 0、したがって witness バージョン 0
  * `w508d6qejxtdg4y5r3zarvary0c5xw7k`: 160 ビット = P2WPKH。
* `9`: 機能...
* `t29a0wturnys2hhxpner2e3plp6jyj8qx7548zr2z7ptgjjc7hljm98xhjym0dg52sdrvqamxdezkmqg4gdrvwwnf0kv2jdfnl4xatsq`: 署名
* `mrnsse`: Bech32 チェックサム
* 署名の内訳：
  * `5a8bd7b97c1cc9055ee60cf2356621f8752248e037a953886a1782b44a58f5ff2d94e6bc89b7b514541a3603bb33722b6c08aa1a3639d34becc549a99fea6eae` 署名データの 16 進数 (32 バイトの r, 32 バイトの s)
  * `0` (int) 署名に含まれるリカバリーフラグ
  * `6c6e626332306d0b25fe64500d044444444444444444444444444444444444444444444444444444444444444442e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc60804340001020304050607080900010203040506070809000102030405060708090102048420751e76e8199196d454941c45d1b3a323f1433bd628070400` セパレータ後のデータの開始までのプレフィックス + データの署名用 16 進数
  * `44fbec32cdac99a1a3cd638ec507dad633a1e5bba514832fd3471e663a157f7b` プレイメージの SHA256 の 16 進数

> ### メインネット上で、フォールバック (P2WSH) アドレス bc1qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3qccfmv3
> lnbc20m1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygshp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfp4qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q9qrsgq9vlvyj8cqvq6ggvpwd53jncp9nwc47xlrsnenq2zp70fq83qlgesn4u3uyf4tesfkkwwfg3qs54qe426hp3tz7z6sweqdjg05axsrjqp9yrrwc

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `20m`: 金額 (20 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `s`: 支払いシークレット...
* `h`: タグ付きフィールド：説明のハッシュ...
* `p`: 支払いハッシュ...
* `f`: タグ付きフィールド：フォールバックアドレス
  * `p4`: `data_length` ( `p` = 1, `4` = 21; 1 * 32 + 21 == 53)
  * `q`: 0、したがって witness バージョン 0
  * `rp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q`: 260 ビット = P2WSH。
* `9`: 機能...
* `9vlvyj8cqvq6ggvpwd53jncp9nwc47xlrsnenq2zp70fq83qlgesn4u3uyf4tesfkkwwfg3qs54qe426hp3tz7z6sweqdjg05axsrjqp`: 署名
* `9yrrwc`: Bech32 チェックサム
* 署名の内訳：
  * `2b3ec248f80301a421817369194f012cdd8af8df1c279981420f9e901e20fa3309d791e11355e609b59ce4a220852a0cd55ab862b1785a83b206c90fa74d01c8` 署名データの 16 進数 (32 バイトの r, 32 バイトの s)
  * `1` (int) 署名に含まれるリカバリーフラグ
  * `6c6e626332306d0b25fe64500d044444444444444444444444444444444444444444444444444444444444444442e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc608043400010203040506070809000102030405060708090001020304050607080901020486a01863143c14c5166804bd19203356da136c985678cd4d27a1b8c63296049032620280704000` セパレータ後のデータの開始までのプレフィックス + データの署名用 16 進数
  * `865a2cc6730e1eeeacd30e6da8e9ab0e9115828d27953ec0c0f985db05da5027` プレイメージの SHA256 の 16 進数

> ### 1週間以内にリスト内のアイテムのために 0.00967878534 BTC を送金してください。金額は pico-BTC です
> lnbc9678785340p1pwmna7lpp5gc3xfm08u9qy06djf8dfflhugl6p7lgza6dsjxq454gxhj9t7a0sd8dgfkx7cmtwd68yetpd5s9xar0wfjn5gpc8qhrsdfq24f5ggrxdaezqsnvda3kkum5wfjkzmfqf3jkgem9wgsyuctwdus9xgrcyqcjcgpzgfskx6eqf9hzqnteypzxz7fzypfhg6trddjhygrcyqezcgpzfysywmm5ypxxjemgw3hxjmn8yptk7untd9hxwg3q2d6xjcmtv4ezq7pqxgsxzmnyyqcjqmt0wfjjq6t5v4khxsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygsxqyjw5qcqp2rzjq0gxwkzc8w6323m55m4jyxcjwmy7stt9hwkwe2qxmy8zpsgg7jcuwz87fcqqeuqqqyqqqqlgqqqqn3qq9q9qrsgqrvgkpnmps664wgkp43l22qsgdw4ve24aca4nymnxddlnp8vh9v2sdxlu5ywdxefsfvm0fq3sesf08uf6q9a2ke0hc9j6z6wlxg5z5kqpu2v9wz

内訳：

* `lnbc`: プレフィックス、ビットコインメインネット上の Lightning
* `9678785340p`: 金額 (9678785340 pico-bitcoin = 967878534 ミリサトシ)
* `1`: Bech32 セパレータ
* `pwmna7l`: タイムスタンプ (1572468703)
* `s`: 支払いシークレット...
* `p`: 支払いハッシュ
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `gc3xfm08u9qy06djf8dfflhugl6p7lgza6dsjxq454gxhj9t7a0s`: 支払いハッシュ 462264ede7e14047e9b249da94fefc47f41f7d02ee9b091815a5506bc8abf75f
* `d`: 短い説明
  * `8d`: `data_length` (`8` = 7, `d` = 13; 7 * 32 + 13 == 237)
  * `gfkx7cmtwd68yetpd5s9xar0wfjn5gpc8qhrsdfq24f5ggrxdaezqsnvda3kkum5wfjkzmfqf3jkgem9wgsyuctwdus9xgrcyqcjcgpzgfskx6eqf9hzqnteypzxz7fzypfhg6trddjhygrcyqezcgpzfysywmm5ypxxjemgw3hxjmn8yptk7untd9hxwg3q2d6xjcmtv4ezq7pqxgsxzmnyyqcjqmt0wfjjq6t5v4khx`: 'Blockstream Store: 88.85 USD for Blockstream Ledger Nano S x 1, 「Back In My Day」ステッカー x 2, 「I Got Lightning Working」ステッカー x 2 and 1 more items'
* `x`: 有効期限
  * `qy`: `data_length` (`q` = 0, `y` = 2; 0 * 32 + 4 == 4)
  * `jw5q`: 604800 秒 (`j` = 18, `w` = 14, `5` = 20, `q` = 0; 18 * 32^3 + 14 * 32^2 + 20 * 32 + 0 == 604800)
* `c`: `min_final_cltv_expiry_delta`
  * `qp`: `data_length` (`q` = 0, `p` = 1; 0 * 32 + 1 == 1)
  * `2`: min_final_cltv_expiry_delta = 10
* `r`: タグ付きフィールド：ルート情報
  * `zj`: `data_length` (`z` = 2, `j` = 18; 2 * 32 + 18 == 82)
  * `q0gxwkzc8w6323m55m4jyxcjwmy7stt9hwkwe2qxmy8zpsgg7jcuwz87fcqqeuqqqyqqqqlgqqqqn3qq9q`:
    * pubkey: 03d06758583bb5154774a6eb221b1276c9e82d65bbaceca806d90e20c108f4b1c7
    * short_channel_id: 589390x3312x1
    * fee_base_msat = 1000
    * fee_proportional_millionths = 2500
    * cltv_expiry_delta = 40
* `9`: 機能...
* `rvgkpnmps664wgkp43l22qsgdw4ve24aca4nymnxddlnp8vh9v2sdxlu5ywdxefsfvm0fq3sesf08uf6q9a2ke0hc9j6z6wlxg5z5kqp`: 署名
* `u2v9wz`: Bech32 チェックサム


> ### 同じピアに対して、コーヒー豆のために $30 を送ってください。ピアは機能 8、14、99 をサポートしており、秘密は 0x1111111111111111111111111111111111111111111111111111111111111111 です。
> lnbc25m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5vdhkven9v5sxyetpdeessp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9q5sqqqqqqqqqqqqqqqqsgq2a25dxl5hrntdtn6zvydt7d66hyzsyhqs4wdynavys42xgl6sgx9c4g7me86a27t07mdtfry458rtjr0v92cnmswpsjscgt2vcse3sgpz3uapa

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `25m`: 金額 (25 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `p`: 支払いハッシュ...
* `d`: 短い説明
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `vdhkven9v5sxyetpdees`: 'coffee beans'
* `s`: 支払いの秘密
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs`: 0x1111111111111111111111111111111111111111111111111111111111111111
* `9`: 機能
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `sqqqqqqqqqqqqqqqqsgq`: b1000....00000100000100000000
* `2a25dxl5hrntdtn6zvydt7d66hyzsyhqs4wdynavys42xgl6sgx9c4g7me86a27t07mdtfry458rtjr0v92cnmswpsjscgt2vcse3sgp`: 署名
* `z3uapa`: Bech32 チェックサム

> ### 同じ内容ですが、すべて大文字です。
> LNBC25M1PVJLUEZPP5QQQSYQCYQ5RQWZQFQQQSYQCYQ5RQWZQFQQQSYQCYQ5RQWZQFQYPQDQ5VDHKVEN9V5SXYETPDEESSP5ZYG3ZYG3ZYG3ZYG3ZYG3ZYG3ZYG3ZYG3ZYG3ZYG3ZYG3ZYG3ZYGS9Q5SQQQQQQQQQQQQQQQQSGQ2A25DXL5HRNTDTN6ZVYDT7D66HYZSYHQS4WDYNAVYS42XGL6SGX9C4G7ME86A27T07MDTFRY458RTJR0V92CNMSWPSJSCGT2VCSE3SGPZ3UAPA

> ### 同じ内容ですが、無視すべきフィールドを含んでいます。
> lnbc25m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5vdhkven9v5sxyetpdeessp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9q5sqqqqqqqqqqqqqqqqsgq2qrqqqfppnqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqppnqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqpp4qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhpnqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqhp4qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqspnqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqsp4qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqnp5qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqnpkqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqz599y53s3ujmcfjp5xrdap68qxymkqphwsexhmhr8wdz5usdzkzrse33chw6dlp3jhuhge9ley7j2ayx36kawe7kmgg8sv5ugdyusdcqzn8z9x

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `25m`: 金額 (25 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `p`: 支払いハッシュ...
* `d`: 短い説明
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `vdhkven9v5sxyetpdees`: 'coffee beans'
* `s`: 支払いシークレット
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs`: 0x1111111111111111111111111111111111111111111111111111111111111111
* `9`: 機能
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `sqqqqqqqqqqqqqqqqsgq`: b1000....00000100000100000000
* `2`: 不明なフィールド
  * `qr`: `data_length` (`q` = 0, `r` = 3; 0 * 32 + 3 == 3)
  * `qqq`: ゼロ
* `f`: タグ付きフィールド：フォールバックアドレス
  * `pp`: `data_length` (`p` = 1, `p` = 1; 1 * 32 + 1 == 33)
  * `nqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`: フォールバックアドレスタイプ 19 (無視)
* `p`: 支払いハッシュ
  * `pn`: `data_length` (`p` = 1, `n` = 19; 1 * 32 + 19 == 51) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `p`: 支払いハッシュ
  * `p4`: `data_length` (`p` = 1, `4` = 21; 1 * 32 + 21 == 53) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `h`: 説明のハッシュ
  * `pn`: `data_length` (`p` = 1, `n` = 19; 1 * 32 + 19 == 51) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `h`: 説明のハッシュ
  * `p4`: `data_length` (`p` = 1, `4` = 21; 1 * 32 + 21 == 53) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `s`: 支払いシークレット
  * `pn`: `data_length` (`p` = 1, `n` = 19; 1 * 32 + 19 == 51) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `s`: 支払いシークレット
  * `p4`: `data_length` (`p` = 1, `4` = 21; 1 * 32 + 21 == 53) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `n`: ノード ID
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `n`: ノード ID
  * `pk`: `data_length` (`p` = 1, `k` = 22; 1 * 32 + 22 == 54) (無視)
  * `qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqq`
* `z599y53s3ujmcfjp5xrdap68qxymkqphwsexhmhr8wdz5usdzkzrse33chw6dlp3jhuhge9ley7j2ayx36kawe7kmgg8sv5ugdyusdcq`: 署名
* `zn8z9x`: Bech32 チェックサム

> ### 0.01 BTC を支払いメタデータ 0x01fafaf0 と共に送信してください
> lnbc10m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdp9wpshjmt9de6zqmt9w3skgct5vysxjmnnd9jx2mq8q8a04uqsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9q2gqqqqqqsgq7hf8he7ecf7n4ffphs6awl9t6676rrclv9ckg3d3ncn7fct63p6s365duk5wrk202cfy3aj5xnnp5gs3vrdvruverwwq7yzhkf5a3xqpd05wjc

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `10m`: 金額 (10 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `p`: 支払いハッシュ
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypq`: 支払いハッシュ 0001020304050607080900010203040506070809000102030405060708090102
* `d`: 短い説明
  * `p9`: `data_length` (`p` = 1, `9` = 5; 1 * 32 + 5 == 37)
  * `wpshjmt9de6zqmt9w3skgct5vysxjmnnd9jx2`: '支払いメタデータ内'
* `m`: メタデータ
  * `q8`: `data_length` (`q` = 0, `8` = 7; 0 * 32 + 7 == 7)
  * `q8a04uq`: 0x01fafaf0
* `s`: 支払いシークレット
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs`: 0x1111111111111111111111111111111111111111111111111111111111111111
* `9`: 機能
  * `q2`: `data_length` (`q` = 0, `2` = 10; 0 * 32 + 10 == 10)
  * `gqqqqqqsgq`: [b01000000000000000000000000000000000100000100000000] = 8 + 14 + 48
* `7hf8he7ecf7n4ffphs6awl9t6676rrclv9ckg3d3ncn7fct63p6s365duk5wrk202cfy3aj5xnnp5gs3vrdvruverwwq7yzhkf5a3xqp`: 署名
* `d05wjc`: Bech32 チェックサム

# 無効なインボイスの例

> # 同じですが、無効な未知の機能 100 を追加
> lnbc25m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5vdhkven9v5sxyetpdeessp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9q4psqqqqqqqqqqqqqqqqsgqtqyx5vggfcsll4wu246hz02kp85x4katwsk9639we5n5yngc3yhqkm35jnjw4len8vrnqnf5ejh0mzj9n3vz2px97evektfm2l6wqccp3y7372

内訳：

* `lnbc`: プレフィックス、Bitcoin メインネット上の Lightning
* `25m`: 金額 (25 ミリビットコイン)
* `1`: Bech32 セパレータ
* `pvjluez`: タイムスタンプ (1496314658)
* `p`: 支払いハッシュ...
* `d`: 短い説明
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `vdhkven9v5sxyetpdees`: 'コーヒー豆'
* `s`: 支払いシークレット
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs`: 0x1111111111111111111111111111111111111111111111111111111111111111
* `9`: 機能
  * `q4`: `data_length` (`q` = 0, `4` = 21; 0 * 32 + 21 == 21)
  * `psqqqqqqqqqqqqqqqqsgq`: b000011000....00000100000100000000
* `tqyx5vggfcsll4wu246hz02kp85x4katwsk9639we5n5yngc3yhqkm35jnjw4len8vrnqnf5ejh0mzj9n3vz2px97evektfm2l6wqccp`: 署名
* `3y7372`: Bech32 チェックサム

> ### Bech32 チェックサムが無効です。
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpquwpc4curk03c9wlrswe78q4eyqc7d8d0xqzpuyk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esqchsrnt

> ### Bech32 文字列が不正です (no 1)
> pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpquwpc4curk03c9wlrswe78q4eyqc7d8d0xqzpuyk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esqchsrny

> ### Bech32 文字列が不正です (混在ケース)
> LNBC2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpquwpc4curk03c9wlrswe78q4eyqc7d8d0xqzpuyk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esqchsrny

> ### 署名が回復不能です。
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5xysxxatsyp3k7enxv4jsxqzpusp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9qrsgqwgt7mcn5yqw3yx0w94pswkpq6j9uh6xfqqqtsk4tnarugeektd4hg5975x9am52rz4qskukxdmjemg92vvqz8nvmsye63r5ykel43pgz7zq0g2

> ### 文字列が短すぎます。
> lnbc1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpl2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6na6hlh

> ### 無効な乗数です。
> lnbc2500x1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5xysxxatsyp3k7enxv4jsxqzpusp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9qrsgqrrzc4cvfue4zp3hggxp47ag7xnrlr8vgcmkjxk3j5jqethnumgkpqp23z9jclu3v0a7e0aruz366e9wqdykw6dxhdzcjjhldxq0w6wgqcnu43j

> ### 無効なサブミリサトシ精度です。
> lnbc2500000001p1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5xysxxatsyp3k7enxv4jsxqzpusp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygs9qrsgq0lzc236j96a95uv0m3umg28gclm5lqxtqqwk32uuk4k6673k6n5kfvx3d2h8s295fad45fdhmusm8sjudfhlf6dcsxmfvkeywmjdkxcp99202x

# 著者

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。

ファイルがまだ提供されていないようです。Markdown ファイルをアップロードしてください。
