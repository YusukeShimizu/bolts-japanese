# BOLT #12: Lightning 支払いのためのネゴシエーションプロトコル

# 目次

  * [BOLT 11 の制限](#limitations-of-bolt-11)
  * [支払いフローのシナリオ](#payment-flow-scenarios)
  * [エンコーディング](#encoding)
  * [署名の計算](#signature-calculation)
  * [オファー](#offers)
  * [インボイスリクエスト](#invoice-requests)
  * [インボイス](#invoices)
  * [インボイスエラー](#invoice-errors)

# BOLT 11 の制限

BOLT 11 のインボイスフォーマットは広く普及していますが、次のようないくつかの制限があります。

1. bech32 エンコーディングが密に絡んでいるため、別の形式 (例えば Lightning ネットワーク内部) で送信するのが扱いにくいです。
2. 署名がインボイス全体に適用されるため、インボイスの全体を開示せずに、その一部だけを証明することができません。
3. フィールドを外部利用のために抽出することが基本的にできません。`h` フィールドは `d` フィールドだけを特別に抽出するための仕組みでした。
4. 「奇数なら無視してよい」というルールが存在しないため、後方互換性の確保が難しくなっています。
5. 「人間に読みやすい」金額表記を分けるという発想は問題が多いことが判明しました。`p` はしばしば誤って扱われ、ピコビットコイン単位の金額は現代のサトシベースの計算より扱いにくいです。
6. 開発者は、bech32 エンコーディングが拡張に向かないと感じており、いずれにせよ置き換えるか廃棄したいと考えています。
7. 経路上の他のノードによるプロービングを防ぐために設計された `payment_secret` は、インボイスが支払者と受取人の間で非公開に保たれている場合にのみ有効でした。
8. インボイスはユーザーごとに発行しなければならず、同じユーザーに対して 2 回支払いを試みると深刻な問題を引き起こします。


# 支払いフローのシナリオ

ここでは「ユーザー」を個々のユーザーの Lightning ノードを指す略称として、「商人」を何かを販売している (または販売した) 主体のノードを指す略称として用います。

BOLT 12 がサポートする基本的な支払いフローは 2 つあります。

ユーザーが商人に支払う一般的なフロー。
1. 商人がウェブページや QR コードなどで *オファー* を公開します。
2. 各ユーザーが、オファーフィールドを含む *invoice_request* メッセージを使って、Lightning ネットワーク経由で固有の *インボイス* を要求します。
3. 商人が *インボイス* を返します。
4. ユーザーがインボイスの指示にしたがって商人へ支払いを行います。

商人がユーザーに支払うフロー (例: ATM や返金)。
1. 商人は、ユーザーに送りたい金額を含む *invoice_request* を公開します。
2. ユーザーは、(一時的なものでもよい) *invoice_node_id* を用いて、*invoice_request* に示された金額の *インボイス* を Lightning ネットワーク経由で送信します。
3. 商人は *invoice_node_id* を確認して正しい相手に支払おうとしていることを確かめ、そのインボイスへの支払いを実行します。

## 支払い証明と支払者証明

通常の Lightning における「支払い証明」は、(`payment_hash` のプリイメージを示すことで) インボイスが支払われたことを示せるだけで、誰が支払ったかを示すことはできません。商人はインボイスが支払われたと主張でき、いったんプリイメージが明らかになると、誰でも自分がそのインボイスを支払ったと主張できてしまいます。[1]

*invoice_request* に鍵を含めることで、支払者は自分がそのインボイスを要求した本人であることを証明できます。さらに、BOLT 12 のインボイス署名はマークル構造を採用しているため、紛争が生じた場合にユーザーはインボイスのフィールドを選択的に開示できます。

# エンコーディング

ここで定義される各形式は [TLV](01-messaging.md#type-length-value-format) 形式です。

サポートされる ASCII エンコーディングは、人間可読プレフィックスに `1` を続け、その後に bech32 形式で TLV のデータ文字列を順に並べたものです。途中に `+` を任意に挟んで、続きのデータがあることを示すことができます。bech32m と異なり、チェックサムはありません。

## 要件

bolt12 文字列の作成者は次のとおりです。
- すべて小文字、またはすべて大文字のいずれかを使用しなければなりません。
- QR コード向けには大文字を使用すべきです。
- それ以外では小文字を使用すべきです。
- 大きな bolt12 文字列を区切るために、任意で空白を伴う `+` を使用してよいです。

bolt12 文字列の読者は次のとおりです。
- すべて小文字、またはすべて大文字の文字列を扱えなければなりません。
- 2 つの bech32 文字の間に、0 個以上の空白文字を伴って `+` が現れた場合:
  - その `+` と空白を取り除かなければなりません。

## 理論的根拠

bech32 を使うのは恣意的な選択ですが、ビットコインの世界ですでに利用されている方式です。現状では末尾の 6 文字のチェックサムを省略しています。QR コード自体がチェックサムを備えており、誤りがあっても資金喪失にはつながらず、単に無効なオファー (またはパース不能) になるだけだからです。

`+` (これは無視されます) を使うことで、Twitter のような文字数制限のあるテキスト欄でも利用できます。

```
lno1xxxxxxxx+

yyyyyyyyyyyy+

zzzzz
```

[format-string-test.json](bolt12/format-string-test.json) を参照してください。

# Signature Calculation

すべての署名は [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) にしたがって作成され、そこで推奨されているとおりにタグ付けされます。すなわち、H(`tag`,`msg`) を SHA256(SHA256(`tag`) || SHA256(`tag`) || `msg`) と定義し、SIG(`tag`,`msg`,`key`) を `key` を用いた H(`tag`,`msg`) の署名と定義します。

各形式は 1 つ以上の *signature TLV elements* (TLV タイプ 240 から 1000 まで、両端を含む) を用いて署名されます。これらのタグは "lightning" || `messagename` || `fieldname` であり、`msg` はマークルルートです。"lightning" はリテラルな 9 バイトの ASCII 文字列で、`messagename` は署名対象の TLV ストリーム名 (例えば "invoice_request" や "invoice")、`fieldname` は署名を格納する TLV フィールド名 (例えば "signature") です。

マークルツリーの構成は [BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) で提案されているものに似ており、証明の中で隣接ノードを露呈しないよう、各 TLV リーフはノンスリーフと対にされます。

マークルツリーのリーフは、各 TLV について TLV の昇順で次のとおりです。
1. H("LnLeaf",tlv)。
2. H("LnNonce"||first-tlv,tlv-type)。ここで first-tlv はストリーム内で数値的に最初の TLV エントリ、tlv-type は現在の TLV の "type" フィールド (1〜9 バイト) です。

マークルツリーの内部ノードは H("LnBranch", lesser-SHA256||greater-SHA256) です。この順序付けにより、左右が本質的に決まるので、証明をよりコンパクトにできます。

リーフの数がちょうど 2 のべき乗でない場合、ツリーの深さは不揃いになり、最も低い順位のリーフが最も深くなります。

例として、TLV0、TLV1、TLV2 (それぞれタイプ 0、1、2) を持つ `invoice` の `signature` のエンコードを考えます。

```
L1=H("LnLeaf",TLV0)
L1nonce=H("LnNonce"||TLV0,0)
L2=H("LnLeaf",TLV1)
L2nonce=H("LnNonce"||TLV0,1)
L3=H("LnLeaf",TLV2)
L3nonce=H("LnNonce"||TLV0,2)

Assume L1 < L1nonce, L2 > L2nonce and L3 > L3nonce.

   L1    L1nonce                      L2   L2nonce                L3   L3nonce
     \   /                             \   /                       \   /
      v v                               v v                         v v
L1A=H("LnBranch",L1||L1nonce) L2A=H("LnBranch",L2nonce||L2)  L3A=H("LnBranch",L3nonce||L3)
                 
Assume L1A < L2A:

       L1A   L2A                                 L3A=H("LnBranch",L3nonce||L3)
         \   /                                    |
          v v                                     v
  L1A2A=H("LnBranch",L1A||L2A)                   L3A=H("LnBranch",L3nonce||L3)
  
Assume L1A2A > L3A:

  L1A2A=H("LnBranch",L1A||L2A)          L3A
                          \            /
                           v          v
                Root=H("LnBranch",L3A||L1A2A)

Signature = SIG("lightninginvoicesignature", Root, nodekey)
```

# Offers

オファーは invoice_request の前段階のものです。読者はオファーに基づいてインボイス (またはその複数) を要求します。オファーは特定のインボイスよりもずっと長く存続することがあるため、いくつか異なる特徴を持っています。特に、金額が Lightning 以外の通貨で表されることがあります。また、QR コードに無理なく収まるよう、コンパクトさを重視して設計されています。

非署名 TLV 要素は invoice_request および invoice メッセージに反映されるため、これらは互いに重ならない、固有の TLV 範囲を持っています。

オファーの人間可読プレフィックスは `lno` です。

## オファーの TLV フィールド

1. `tlv_stream`: `offer`
2. types:
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 22 (`offer_issuer_id`)
    2. data:
        * [`point`:`id`]

## オファーの要件

オファーの作成者は次のとおりです。
  - 1 から 79、および 1000000000 から 1999999999 (両端を含む) の範囲外の TLV フィールドを設定してはなりません。
  - インボイスのチェーンがビットコインのみではない場合:
    - そのオファーが有効な `offer_chains` を指定しなければなりません。
  - そうでない場合:
    - `offer_chains` を省略すべきです (これによりチェーンがビットコインのみであることを暗黙に示します)。
  - 支払いを成立させるために特定の最小 `offer_amount` が必要な場合:
    - `offer_amount` を期待される金額 (1 アイテムあたり) に設定しなければなりません。
    - `offer_amount` をゼロより大きく設定しなければなりません。
    - `offer_amount` の通貨が `chains` のすべてのエントリの通貨と同じ場合:
      - `offer_amount` を、Lightning における最小支払単位 (例えばビットコインならミリサトシ) の倍数で指定しなければなりません。
    - そうでない場合:
      - `offer_currency` `iso4217` を ISO 4217 の 3 文字コードとして指定しなければなりません。
      - `offer_amount` を、ISO 4217 の指数で調整した通貨単位 (例えば USD のセント) で指定しなければなりません。
    - 支払いの目的を完全に説明する `offer_description` を設定しなければなりません。
  - そうでない場合:
    - `offer_amount` を設定してはなりません。
    - `offer_currency` を設定してはなりません。
    - `offer_description` を設定してよいです。
  - 自分用の用途で `offer_metadata` を設定してよいです。
  - bolt12 のオファー機能をサポートする場合:
    - `offer_features`.`features` を bolt12 機能のビットマップに設定しなければなりません。
  - オファーが期限切れになる場合:
    - `offer_absolute_expiry` `seconds_from_epoch` を、1970 年 1 月 1 日午前 0 時 (UTC) からの秒数で、その時刻以降は invoice_request を試みるべきでないという値に設定しなければなりません。
  - プライベートチャネルだけで接続されている場合:
    - 公開到達可能なノードから自ノードへ至る 1 つ以上のパスを含む `offer_paths` を含めなければなりません。
  - そうでない場合:
    - `offer_paths` を含めてよいです。
  - `offer_paths` を含める場合:
    - `offer_issuer_id` を設定してよいです。
  - そうでない場合:
    - `offer_issuer_id` を、インボイスを要求する相手ノードの公開鍵に設定しなければなりません。
  - `offer_issuer` を設定する場合:
    - インボイスの発行者を明確に識別できる値に設定すべきです。
    - ドメイン名を含める場合:
      - user@domain か domain のいずれかで始めるべきです。
      - その後ろにスペースとさらにテキストを続けてよいです。
  - 1 つのインボイスで複数のアイテムを供給できる場合:
    - 最大数量がわかっている場合:
      - その最大値を `offer_quantity_max` に設定しなければなりません。
      - `offer_quantity_max` を 0 に設定してはなりません。
    - そうでない場合:
      - `offer_quantity_max` を 0 に設定しなければなりません。
  - そうでない場合:
    - `offer_quantity_max` を設定してはなりません。

オファーの読者は次のとおりです。
  - オファーが、1 から 79 および 1000000000 から 1999999999 (両端を含む) の範囲外の TLV フィールドを含む場合:
    - そのオファーに応答してはなりません。
  - `offer_features` に未知の _奇数_ ビットが非ゼロで含まれる場合:
    - そのビットを無視しなければなりません。
  - `offer_features` に未知の _偶数_ ビットが非ゼロで含まれる場合:
    - そのオファーに応答してはなりません。
    - 未知のビットがあることをユーザーに示すべきです。
  - `offer_chains` が設定されていない場合:
    - 自ノードがビットコインのインボイスを受け入れないなら:
      - そのオファーに応答してはなりません。
  - そうでない場合 (`offer_chains` が設定されている場合):
    - 自ノードが `chains` のいずれのチェーンのインボイスも受け入れないなら:
      - そのオファーに応答してはなりません。
  - `offer_amount` が設定されているが `offer_description` が設定されていない場合:
    - そのオファーに応答してはなりません。
  - `offer_amount` が設定されており、その値がゼロより大きくない場合:
    - そのオファーに応答してはなりません。
  - `offer_currency` が設定されているが `offer_amount` が設定されていない場合:
    - そのオファーに応答してはなりません。
  - `offer_issuer_id` と `offer_paths` のいずれも設定されていない場合:
    - そのオファーに応答してはなりません。
  - `offer_paths` 内のいずれかの `blinded_path` で `num_hops` が 0 の場合:
    - そのオファーに応答してはなりません。
  - `offer_amount` を使ってユーザーにコスト見積もりを提示する場合:
    - `offer_amount` の通貨単位を考慮しなければなりません。
      - `offer_currency` フィールドが設定されている場合はその通貨。
      - そうでない場合は、Lightning における最小支払単位 (例えばビットコインならミリサトシ)。
    - 受け取った `invoice_amount` がその見積もりと大きく異なる場合、ユーザーに警告しなければなりません。
  - 現在時刻が `offer_absolute_expiry` を過ぎている場合:
    - そのオファーに応答してはなりません。
  - インボイスリクエストを送ると決めた場合、オニオンメッセージを送信します。
    - `offer_paths` が設定されている場合:
      - `offer_paths` 内のいずれかのパスを経由し、そのパスの最終 `onion_msg_hop`.`blinded_node_id` 宛にオニオンメッセージを送信しなければなりません。
    - そうでない場合:
      - `offer_issuer_id` 宛にオニオンメッセージを送信しなければなりません。
    - 同時に複数のインボイスリクエストオニオンメッセージを送信してよいです。

## 理論的根拠

オファー全体は invoice_request に反映されます。これは情報の完全性のため (すべての情報をインボイスに返せるようにするため) と、オファー側のノードをステートレスにできるようにするためです。これにより、`offer_metadata` は特に有用になります。なぜなら、他のフィールドを検証するための認証クッキーをそこに格納できるからです。

オファーの各フィールドはインボイスリクエスト (さらにインボイス) にコピーされるため、それぞれ別々の TLV 範囲が必要になります。範囲 1〜79 が通常の範囲で、もう 10 億分は自己割当ての実験用範囲として確保されています。

署名は不要であり、付ければ文字列が長くなる (低性能カメラでは QR コードが読めなくなることがある) だけです。オファーに誤りがあった場合でも、リクエストには非署名フィールドがすべて含まれているため、インボイスは発行されません。

`offer_paths` が設定されている場合、`offer_issuer_id` は省略してよいです。なぜなら、各パスの最終 `blinded_node_id` が宛先の有効な公開鍵として機能できるからです。

`offer_amount` は (`offer_currency` フィールドにより) 異なる通貨で指定できるため、あくまで目安にすぎません。発行者はインボイスを生成する時点でこれを `invoice_amount` のミリサトシ数に変換するか、あるいはインボイスリクエスト側が `invreq_amount` で正確な金額を指定することもできますが、その場合、発行者が同意しなければそれを拒否することがあります。

`offer_quantity_max` は 1 を設定することも許されています。一見無意味に思えますが、在庫数に応じてオファーを生成するシステムでは有用です。「残り 1 つだけ」のオファー生成を特別扱いしなくて済むからです。

オファーは、見返りを期待せずに単にお金を送る用途 (チップ、賛辞、寄付など) にも使えます。その場合、説明フィールドは任意です (この用途では `offer_issuer` フィールドが非常に有用です)。一方、特定の物に対して課金する場合には、ユーザーが何の対価として支払ったのかを知るために説明が不可欠です。

中身が空の `offer_chains` (フィールドは存在するがエントリが 0 個) は、明示的に無効です。なぜなら、その状態ではインボイスリクエストを構成できないからです。`offer_chains` にチェーンが 1 つも列挙されていなければ、支払者は `invreq_chain` を「`offer_chains` のいずれか」に設定できません。このようなオファーを早期に拒否することで、インボイスリクエスト段階で実装が失敗するのを避け、明確なフィードバックを返せます。

# インボイスリクエスト

インボイスリクエストは、その名のとおりインボイスを求める要求です。インボイスリクエストの人間可読プレフィックスは `lnr` です。

インボイスリクエストには、ワークフローとしてはほぼ同一でも、ユーザーから見るとかなり異なる、よく似た 2 つの用途があります。

1 つ目は、オファーに対する応答としての利用です。この場合、`offer_issuer_id` または `offer_paths` を含む他のすべてのオファー詳細が含まれており、通常はオニオンメッセージで受信されます。受け取ったリクエストが有効で既知のオファーを参照しているなら、応答は通常、オニオンメッセージの `reply_path` フィールドを使って `invoice` を返すことです。

2 つ目は、オファーを介さずに、例えば QR コードなどでインボイスリクエストそのものを公開する用途です。この場合は `offer_issuer_id` も `offer_paths` も含まず、代わりに支払者として `invreq_payer_id` (および必要なら `invreq_paths`) を設定します。他のオファー由来のフィールドは `invoice_request` の作成者によって埋められ、いわば「お金を送るためのオファー」を構成します。

注: `invreq_metadata` は番号 0 が割り当てられており (他の invreq フィールドの 80〜159 範囲には含まれません)、[Signature Calculation](#signature-calculation) における「数値的に最初の TLV エントリ」となります。これによりマークルリーフが推測不可能になり、フィールドを隠したまま署名検証を可能にする、将来のコンパクトな表現を許容します。


## `invoice_request` の TLV フィールド

1. `tlv_stream`: `invoice_request`
2. types:
    1. type: 0 (`invreq_metadata`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 22 (`offer_issuer_id`)
    2. data:
        * [`point`:`id`]
    1. type: 80 (`invreq_chain`)
    2. data:
        * [`chain_hash`:`chain`]
    1. type: 82 (`invreq_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 84 (`invreq_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 86 (`invreq_quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 88 (`invreq_payer_id`)
    2. data:
        * [`point`:`key`]
    1. type: 89 (`invreq_payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 90 (`invreq_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 91 (`invreq_bip_353_name`)
    2. data:
        * [`u8`:`name_len`]
        * [`name_len*byte`:`name`]
        * [`u8`:`domain_len`]
        * [`domain_len*byte`:`domain`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]

## インボイスリクエストの要件

書き手は次のとおりです。

  - オファーに応答する場合:
    - オファーからすべてのフィールド (未知のフィールドを含む) をコピーしなければなりません。
    - `offer_chains` が設定されている場合:
      - `invreq_chain` を `offer_chains` のいずれかに設定しなければなりません。ただし、そのチェーンがビットコインの場合は `invreq_chain` を省略すべきです。
    - そうでない場合:
      - `invreq_chain` を設定するなら、ビットコインに設定しなければなりません。
    - `signature`.`sig` を、`invreq_payer_id` を用いて [Signature Calculation](#signature-calculation) のとおりに設定しなければなりません。
    - `offer_amount` が存在しない場合:
      - `invreq_amount` を指定しなければなりません。
    - そうでない場合:
      - `invreq_amount` を省略してよいです。
      - `invreq_amount` を設定する場合:
        - `invreq_amount`.`msat` を、`offer_amount` (および存在する場合は `offer_currency` と `invreq_quantity`) から期待される金額以上に指定しなければなりません。
    - `invreq_payer_id` を一時的な公開鍵に設定しなければなりません。
    - `invreq_payer_id` に対応する秘密鍵を覚えておかなければなりません。
    - `offer_quantity_max` が存在する場合:
      - `invreq_quantity` を 0 より大きく設定しなければなりません。
      - `offer_quantity_max` が非ゼロの場合:
        - `invreq_quantity` を `offer_quantity_max` 以下に設定しなければなりません。
    - そうでない場合:
      - `invreq_quantity` を設定してはなりません。
  - そうでない場合 (オファーへの応答ではない場合):
    - 支払いの目的を完全に説明する `offer_description` を設定しなければなりません。
    - `offer_absolute_expiry` と `offer_issuer` を、オファーの場合と同じ要領で設定 (または不設定に) しなければなりません。
    - `invreq_payer_id` を設定しなければなりません (オファーにおける `offer_issuer_id` と同様の意味で)。
    - `invreq_paths` を、オファーにおける `offer_paths` と同じ要領で設定 (または不設定に) しなければなりません。
    - `signature`、`offer_metadata`、`offer_chains`、`offer_amount`、`offer_currency`、`offer_features`、`offer_quantity_max`、`offer_paths`、`offer_issuer_id` を含めてはなりません。
    - インボイスのチェーンがビットコインのみではない場合:
      - 有効な `invreq_chain` を指定しなければなりません。
    - `invreq_amount` を設定しなければなりません。
  - 0 から 159 および 1000000000 から 2999999999 (両端を含む) の範囲外に、署名以外の TLV フィールドを設定してはなりません。
  - `invreq_metadata` を予測不可能なバイト列に設定しなければなりません。
  - `invreq_amount` を設定する場合:
    - `msat` を、`invreq_chain` (または `invreq_chain` がない場合はビットコイン) における最小 Lightning 支払単位 (例えばビットコインならミリサトシ) の倍数で設定しなければなりません。
  - bolt12 のインボイスリクエスト機能をサポートする場合:
    - `invreq_features`.`features` を機能のビットマップに設定しなければなりません。
  - この `invoice_request` を構築する元となったオファーを BIP 353 解決を用いて取得した場合:
    - `invreq_bip_353_name` を含めなければなりません。
      - `name` は BIP 353 の人間可読名 (HRN) のうち、₿ より後ろで @ より前の部分に設定します。
      - `domain` は同 HRN のうち、@ より後ろの部分に設定します。

読者は次のとおりです。

  - `invreq_payer_id` または `invreq_metadata` が存在しない場合、インボイスリクエストを拒否しなければなりません。
  - 0 から 159 および 1000000000 から 2999999999 (両端を含む) の範囲外に署名以外の TLV フィールドがある場合、インボイスリクエストを拒否しなければなりません。
  - `invreq_features` に未知の _奇数_ ビットが非ゼロで含まれる場合:
    - そのビットを無視しなければなりません。
  - `invreq_features` に未知の _偶数_ ビットが非ゼロで含まれる場合:
    - インボイスリクエストを拒否しなければなりません。
  - `signature` が `invreq_payer_id` を用いた [Signature Calculation](#signature-calculation) に従って正しくない場合、インボイスリクエストを拒否しなければなりません。
  - `invreq_paths` 内のいずれかの `blinded_path` で `num_hops` が 0 の場合:
    - インボイスリクエストを拒否しなければなりません。
  - `offer_issuer_id` が存在し、`invreq_metadata` が以前のある `invoice_request` と同一である場合:
    - 単にそのときと同じインボイスを返してよいです。
  - そうでない場合:
    - 以前のインボイスを返してはなりません。
  - `offer_issuer_id` または `offer_paths` が存在する場合 (オファーへの応答):
    - オファーフィールドが、有効で未失効のオファーと完全には一致しない場合、インボイスリクエストを拒否しなければなりません。
    - `offer_paths` が存在する場合:
      - そのいずれかのパスを経由して到着していないインボイスリクエストは無視しなければなりません。
    - そうでない場合:
      - ブラインドパスを経由して到着したインボイスリクエストは無視しなければなりません。
    - `offer_quantity_max` が存在する場合:
      - `invreq_quantity` フィールドが存在しない場合、インボイスリクエストを拒否しなければなりません。
      - `offer_quantity_max` が非ゼロの場合:
        - `invreq_quantity` が 0 か、または `offer_quantity_max` を超える場合、インボイスリクエストを拒否しなければなりません。
    - そうでない場合:
      - `invreq_quantity` フィールドが存在する場合、インボイスリクエストを拒否しなければなりません。
    - `offer_amount` が存在する場合:
      - `offer_amount` を用いて *expected amount (期待金額)* を計算しなければなりません。
        - `offer_currency` が `invreq_chain` の通貨でない場合、`invreq_chain` の通貨に変換します。
        - `invreq_quantity` が存在する場合、`invreq_quantity`.`quantity` を掛けます。
      - `invreq_amount` が存在する場合:
        - `invreq_amount`.`msat` が *期待金額* 未満であれば、インボイスリクエストを拒否しなければなりません。
        - `invreq_amount`.`msat` が *期待金額* を大幅に超える場合、インボイスリクエストを拒否してよいです。
    - そうでない場合 (`offer_amount` がない場合):
      - `invreq_amount` を含まないなら、インボイスリクエストを拒否しなければなりません。
    - 応答のインボイスは `onionmsg_tlv` の `reply_path` を用いて送信すべきです。
  - そうでない場合 (`offer_issuer_id` も `offer_paths` も存在せず、自分のオファーへの応答ではない場合):
    - 次のいずれかが存在する場合、インボイスリクエストを拒否しなければなりません。
      - `offer_chains`、`offer_features`、`offer_quantity_max`。
    - `invreq_amount` が存在しない場合、インボイスリクエストを拒否しなければなりません。
    - `offer_amount` (または `offer_currency`) を、ユーザーへの情報表示用に使ってよいです。
    - 応答としてインボイスを送る場合:
      - `invreq_paths` が存在すればそれを使い、なければ `invreq_payer_id` を送信先のノード ID として送信しなければなりません。
  - `invreq_chain` が存在しない場合:
    - ビットコインがサポート対象のチェーンでないなら、インボイスリクエストを拒否しなければなりません。
  - そうでない場合:
    - `invreq_chain`.`chain` がサポート対象のチェーンでないなら、インボイスリクエストを拒否しなければなりません。
  - `invreq_bip_353_name` が存在する場合:
    - `name` または `domain` に、`0`-`9`、`a`-`z`、`A`-`Z`、`-`、`_`、`.` 以外のバイトが含まれていれば、インボイスリクエストを拒否しなければなりません。

## Rationale

`invreq_metadata` には通常、`invreq_payer_id` の導出に関する情報が入ります。この値はいかなる情報も漏らすべきではありません (例えば単純な BIP-32 導出パスを使うべきではありません)。妥当な仕組みとしては、ノードがベースとなる支払者鍵を保持し、ここに 128 ビットのトゥイークをエンコードする方法が考えられます。支払者 ID は SHA256(payer_base_pubkey || tweak) でベース鍵をトゥイークすることで導出します。`invreq_metadata` は (存在する場合) 最初のエントリでもあり、これによってハッシュ計算用に予測不可能なノンスが得られます。

`invreq_payer_note` を使えば、インボイスに賛辞や挑発、その他の落書きを刻んで、誰の目にも見える形で残せます。

ユーザーは、オファーが `offer_amount` を指定していても、インボイスリクエストで `invreq_amount` を指定することで、チップを上乗せしたり (送信額をぼかしたり) できます。受取人は、インボイスリクエストの金額が自分の期待する金額 (すなわち通貨換算後の `offer_amount` に、もしあれば `invreq_quantity` を掛けたもの) を上回る場合にのみこれを受け入れます。

オファーへの応答ではないインボイスリクエストは、現在のところチェーン通貨での `invreq_amount` を明示することが求められています。そのため `offer_amount` と `offer_currency` は冗長ですが、送信者が `invreq_amount` をどう導出したかを支払者に知らせる情報源としては有用な場合があります。

`offer_paths` が存在するならそれを使わなければならない、という要件は、ノードが直接問い合わされたときに自分がそのオファーの発行元であると暴露しないようにするためです。同様に、オファーに対して正しいパスを使わなければならないという要件は、別のオファーを作成したのと同じノードであることが推測できないようにするためです。

# インボイス

インボイスは支払いの要求であり、支払いが行われると、その支払いプリイメージとインボイスを組み合わせて暗号学的な領収書を作ることができます。

受取人は `invoice_request` への応答として、`onion_message` の `invoice` フィールドを用いて `invoice` を送信します。

1. `tlv_stream`: `invoice`
2. types:
    1. type: 0 (`invreq_metadata`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 22 (`offer_issuer_id`)
    2. data:
        * [`point`:`id`]
    1. type: 80 (`invreq_chain`)
    2. data:
        * [`chain_hash`:`chain`]
    1. type: 82 (`invreq_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 84 (`invreq_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 86 (`invreq_quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 88 (`invreq_payer_id`)
    2. data:
        * [`point`:`key`]
    1. type: 89 (`invreq_payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 90 (`invreq_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 91 (`invreq_bip_353_name`)
    2. data:
        * [`u8`:`name_len`]
        * [`name_len*byte`:`name`]
        * [`u8`:`domain_len`]
        * [`domain_len*byte`:`domain`]
    1. type: 160 (`invoice_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 162 (`invoice_blindedpay`)
    2. data:
        * [`...*blinded_payinfo`:`payinfo`]
    1. type: 164 (`invoice_created_at`)
    2. data:
        * [`tu64`:`timestamp`]
    1. type: 166 (`invoice_relative_expiry`)
    2. data:
        * [`tu32`:`seconds_from_creation`]
    1. type: 168 (`invoice_payment_hash`)
    2. data:
        * [`sha256`:`payment_hash`]
    1. type: 170 (`invoice_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 172 (`invoice_fallbacks`)
    2. data:
        * [`...*fallback_address`:`fallbacks`]
    1. type: 174 (`invoice_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 176 (`invoice_node_id`)
    2. data:
        * [`point`:`node_id`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]


1. subtype: `blinded_payinfo`
2. data:
   * [`u32`:`fee_base_msat`]
   * [`u32`:`fee_proportional_millionths`]
   * [`u16`:`cltv_expiry_delta`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u64`:`htlc_maximum_msat`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]

1. subtype: `fallback_address`
2. data:
   * [`byte`:`version`]
   * [`u16`:`len`]
   * [`len*byte`:`address`]

## Invoice Features

| Bits | Description                      | Name           |
|------|----------------------------------|----------------|
| 16   | Multi-part-payment support       | MPP/compulsory |
| 17   | Multi-part-payment support       | MPP/optional   |

「MPP サポート」インボイス機能は、支払者がインボイスを支払うために複数の部分支払いを使用しなければならない (16)、または使用してよい (17) ことを示します。

実装によっては (例えば少額支払いの場合) MPP をサポートしないことがあり、また (1 本のチャネルの容量上限のため) MPP を必須とすることもあります。

## Requirements

インボイスの作成者は次のとおりです。
  - インボイスを作成した時点の、1970 年 1 月 1 日午前 0 時 (UTC) からの秒数を `invoice_created_at` に設定しなければなりません。
  - `invoice_amount` を、自分が受け入れる最小金額として、`invreq_chain` における最小 Lightning 支払単位 (例えばビットコインならミリサトシ) で設定しなければなりません。
  - インボイスが `invoice_request` への応答である場合:
    - インボイスリクエストから、署名以外のすべてのフィールド (未知のフィールドを含む) をコピーしなければなりません。
    - `invreq_amount` が存在する場合:
      - `invoice_amount` を `invreq_amount` に設定しなければなりません。
    - そうでない場合:
      - `invoice_amount` を *期待金額* に設定しなければなりません。
  - 支払いと引き換えに渡す `payment_preimage` の SHA256 ハッシュを `invoice_payment_hash` に設定しなければなりません。
  - `offer_issuer_id` が存在する場合:
    - `invoice_node_id` を `offer_issuer_id` に設定しなければなりません。
  - そうでなく、`offer_paths` が存在する場合:
    - インボイスリクエストを受け取ったパスの最終 `blinded_node_id` を `invoice_node_id` に設定しなければなりません。
  - 署名 TLV 要素 `signature` をちょうど 1 つ指定しなければなりません。
    - `sig` は、[Signature Calculation](#signature-calculation) のとおり `invoice_node_id` を用いて求めた署名値に設定しなければなりません。
  - インボイスを支払うために複数パートが必須の場合:
    - `invoice_features`.`features` のビット `MPP/compulsory` を立てなければなりません。
  - あるいは、複数パートでの支払いを許容する場合:
    - `invoice_features`.`features` のビット `MPP/optional` を立てなければなりません。
  - 支払いを受け付ける有効期限が `invoice_created_at` から 7200 秒後でない場合:
    - `invoice_relative_expiry`.`seconds_from_creation` を、`invoice_created_at` から数えてこのインボイスへの支払いを試みるべきでないまでの秒数に設定しなければなりません。
  - オンチェーン支払いを受け付ける場合:
    - `invoice_fallbacks` を指定してよいです。
    - 優先順位がある場合は、`invoice_fallbacks` を最も優先するものから最も優先しないものへの順で並べるべきです。
    - ビットコインチェーン向けには、各 `fallback_address` の `version` を有効なウィットネスバージョンに、`address` を有効なウィットネスプログラムに設定しなければなりません。
  - 自ノードへ至る 1 つ以上のパスを含む `invoice_paths` を含めなければなりません。
    - 優先順位がある場合は、`invoice_paths` を最も優先するものから最も優先しないものへの順で並べなければなりません。
    - `paths` 内の各 `blinded_path` に対応して、ちょうど 1 つの `blinded_payinfo` を順序を保って含む `invoice_blindedpay` を含めなければなりません。
    - 各 `blinded_payinfo` の `features` を、`encrypted_data_tlv`.`allowed_features` と一致するように (あるいは `allowed_features` がない場合は空に) 設定しなければなりません。
    - これらのいずれのパスも経由しない支払いは無視すべきです。

インボイスの読者は次のとおりです。

  - `invoice_amount` が存在しない場合、インボイスを拒否しなければなりません。
  - `invoice_created_at` が存在しない場合、インボイスを拒否しなければなりません。
  - `invoice_payment_hash` が存在しない場合、インボイスを拒否しなければなりません。
  - `invoice_node_id` が存在しない場合、インボイスを拒否しなければなりません。
  - `invreq_chain` が存在しない場合:
    - ビットコインがサポート対象のチェーンでないなら、インボイスを拒否しなければなりません。
  - そうでない場合:
    - `invreq_chain`.`chain` がサポート対象のチェーンでないなら、インボイスを拒否しなければなりません。
  - `invoice_features` に未知の _奇数_ ビットが非ゼロで含まれる場合:
    - そのビットを無視しなければなりません。
  - `invoice_features` に未知の _偶数_ ビットが非ゼロで含まれる場合:
    - インボイスを拒否しなければなりません。
  - `invoice_relative_expiry` が存在する場合:
    - 1970-01-01 (UTC) からの現在時刻が `invoice_created_at` + `seconds_from_creation` を超えていれば、インボイスを拒否しなければなりません。
  - そうでない場合:
    - 1970-01-01 (UTC) からの現在時刻が `invoice_created_at` + 7200 を超えていれば、インボイスを拒否しなければなりません。
  - `invoice_paths` が存在しない、または空の場合、インボイスを拒否しなければなりません。
  - `invoice_paths` 内のいずれかの `blinded_path` で `num_hops` が 0 の場合、インボイスを拒否しなければなりません。
  - `invoice_blindedpay` が存在しない場合、インボイスを拒否しなければなりません。
  - `invoice_blindedpay` が `invoice_paths`.`blinded_path` ごとにちょうど 1 つの `blinded_payinfo` を含まない場合、インボイスを拒否しなければなりません。
  - 各 `invoice_blindedpay`.`payinfo` について:
    - `payinfo`.`features` に未知の偶数ビットが立っている場合、対応する `invoice_paths`.`path` を使用してはなりません。
    - これにより使用可能なパスが残らなくなる場合、インボイスを拒否しなければなりません。
  - インボイスが `invoice_request` への応答である場合:
    - 0 から 159 および 1000000000 から 2999999999 (両端を含む) の範囲のすべてのフィールドが、インボイスリクエストと完全に一致しない場合、インボイスを拒否しなければなりません。
    - `offer_issuer_id` が存在する場合 (オファー向けの invoice_request):
      - `invoice_node_id` が `offer_issuer_id` と等しくない場合、インボイスを拒否しなければなりません。
    - そうでなく、`offer_paths` が存在する場合 (ID なしオファー向けの invoice_request):
      - `invoice_node_id` が、自分がインボイスリクエストを送った先の最終 `blinded_node_id` と等しくない場合、インボイスを拒否しなければなりません。
    - そうでない場合 (オファーを伴わない invoice_request):
      - `invoice_node_id` が正しいことを帯域外で確認できない場合、インボイスを拒否してよいです。
  - `signature` が `invoice_node_id` を用いた [Signature Calculation](#signature-calculation) どおりの有効な署名でない場合、インボイスを拒否しなければなりません。
  - 他に優先する理由がない場合、後ろの `invoice_paths` よりも前のものを優先して使うべきです。
  - `invoice_features` に MPP/compulsory ビットが立っている場合:
    - 複数の別々のブラインドパスを通じてインボイスを支払わなければなりません。
  - そうでなく、`invoice_features` に MPP/optional ビットが立っている場合:
    - 複数の別々の支払いを通じてインボイスを支払ってよいです。
  - そうでない場合:
    - 複数のパートを使ってインボイスを支払ってはなりません。
  - `invreq_amount` が存在する場合:
    - `invoice_amount` が `invreq_amount` と等しくない場合、インボイスを拒否しなければなりません。
  - そうでない場合:
    - `invoice_amount`.`msat` が承認済みの金額範囲に収まっていなければ、承認を確認すべきです。
  - ビットコインチェーンの場合で、インボイスが `invoice_fallbacks` を指定するとき:
    - `version` が 16 を超える `fallback_address` は無視しなければなりません。
    - `address` が 2 バイト未満または 40 バイト超の `fallback_address` は無視しなければなりません。
    - 指定された `version` に対する既知の要件を満たさない `fallback_address` は無視しなければなりません。
  - `invreq_paths` が存在する場合:
    - そのいずれかのパスを経由して到着していないインボイスは拒否しなければなりません。
  - そうでなく、`offer_issuer_id` も `offer_paths` も存在しない場合 (オファーから派生していない場合):
    - ブラインドパスを経由して到着したインボイスは拒否しなければなりません。
  - そうでない場合 (オファーから派生している場合):
    - インボイスリクエストの `onionmsg_tlv` `reply_path` を経由して到着していないインボイスは拒否しなければなりません。

## 理論的根拠

メッセージング層は信頼できないため、同じオファーに対するリクエストを複数回受け取ることは十分にあり得ます。`invreq_metadata` を予測不可能かつ一意にする責任は呼び出し側にあるので、書き手はすべてのフィールドが重複しているかを確かめなくても、単に以前のインボイスを返すことができます。なお、こうしたキャッシングは任意であり、例えば通貨換算が絡む場合や、インボイスがすでに失効している場合には慎重に制限すべきです。

インボイスは、以前の invreq にコミットするのではなく、フィールドを重複保持します。この平坦化された形式は若干の容量コストを伴いますが、支払者は返金や証明のためにインボイスだけを覚えておけばよく、ストレージが簡素化されます。

インボイスの読者はインボイスが invreq の各フィールドを正しく反映していると信頼できないため、それらが正しいことを確認する必要があります。とはいえ、要求されていないインボイスを直接送ることも認められています。

インボイスの受取人は、受け取ったオファー、または送信した invreq から期待金額を判断できるため、ほとんどの場合すでにその金額に対する承認を持っています。

`invoice_relative_expiry` のデフォルト値である 7200 秒は、新しいチャネルを開く必要がある場合でも、支払いには通常十分な時間です。

ブラインドパスは、BOLT 11 で使われている `payment_secret` と `payment_metadata` に相当する機能を提供します。`invoice_node_id` や `invreq_payer_id` が公開されている場合でも、これらの機能を維持するためにブラインドパスの利用を強制しています。受取人がブラインドパスによる追加のプライバシーを必要としない場合は、自分自身だけを含む長さ 1 のパスを作成すればよいです。

ブラインドパス内の各ホップごとに詳細な per-hop-payinfo を提供するのではなく、手数料と CLTV デルタを集約します。これは、パスを区別する手がかりになりかねない、目立った非一様性が漏れるのを避けるためです。

オファーが存在せず、インボイスリクエストだけがあったケースのインボイスでは、支払者はそのインボイスが本来の支払先からのものであることを確認する必要があります。これが、このケースで `invoice_node_id` を確認することを推奨する根拠です。

インボイスリクエストに基づかない生のインボイスは一般にはサポートされていませんが、実装側がサポートすることは許されており、将来その挙動を定義する可能性があります。`invreq_chain` を明示的に検査するという冗長な要件はそのための布石です。インボイスがインボイスリクエストへの応答であれば、インボイスリクエストの要件によりそのフィールドは存在していたはずであり、ここでもそれを反映するよう要求しています。


# インボイスエラー

情報提供のためのエラーは、オニオンメッセージの `invoice_error` フィールドを用いて (オニオンの `reply_path` 経由で)、`invoice_request` または `invoice` への応答として返すことができます。

## `invoice_error` の TLV フィールド

1. `tlv_stream`: `invoice_error`
2. types:
    1. type: 1 (`erroneous_field`)
    2. data:
        * [`tu64`:`tlv_fieldnum`]
    1. type: 3 (`suggested_value`)
    2. data:
        * [`...*byte`:`value`]
    1. type: 5 (`error`)
    2. data:
        * [`...*utf8`:`msg`]

## 要件

invoice_error の作成者は次のとおりです。
  - `error` を説明的な文字列に設定しなければなりません。
  - 問題のあった `invoice` または `invoice_request` の特定のフィールド番号に `erroneous_field` を設定してよいです。
  - `erroneous_field` を設定する場合:
    - `suggested_value` を設定してよいです。
    - `suggested_value` を設定する場合:
      - その `tlv_fieldnum` に対する有効な値となるように `suggested_value` を設定しなければなりません。
  - そうでない場合:
    - `suggested_value` を設定してはなりません。

invoice_error の読者は:
   FIXME!

## 理論的根拠

通常はエラーメッセージだけで診断には十分ですが、将来の拡張により自動処理が有用になる可能性があります。

特に、将来的にはオファーへの応答ではない `invoice_request` で `invreq_amount` を省略し、代わりにオファーフィールドを用いて代替通貨を示すことを許す可能性があります (「10 セント送ります!」のようなケース)。その場合、インボイスの送信者は、それが何ミリサトシに相当するかを推定することになり、受取人がその換算に同意しないときには `invoice_error` でその旨を示し、送信者が新しいインボイスを送り直せるようにすることが考えられます。

# FIXME: 将来の拡張案:

1. オファーは `invoice_request` に配送情報を要求できるようにする。
2. オファーは更新可能にする。`invoice_request` への応答が新たなオファーとなり、必要に応じて元の `offer_issuer_id` による署名を含む形にする。
3. 中身が空の TLV フィールドは、その値が他の手段 (すなわちトランスポート固有の方法) で既知であることを意味するが、署名のためにはハッシュされる、という扱いにする。
4. 1 つの invreq とインボイスで複数のオファーを扱えるようにアップグレードし、買い物リストを表現できるようにする。
7. すべてゼロの offer_id を「無償の支払い」を意味するものとして扱う。
8. ストリーミングインボイス?
9. 繰り返し (recurrence) の再追加。
10. 証明をサポートするための `invreq_refund_for` の再追加。
11. (支払いが滞っている) インボイスを新しいインボイスへ差し替えるための `invoice_replace` の再追加。
12. 代替通貨を伴う、オファーなしの `invoice_request` を許可するか?
13. 数量のステップを示す `offer_quantity_unit` の追加 (例: 100 グラム単位)。

[1] https://www.youtube.com/watch?v=4SYc_flMnMQ
