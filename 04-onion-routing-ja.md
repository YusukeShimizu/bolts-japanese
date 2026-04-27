# BOLT #4: オニオンルーティングプロトコル

## 概要

このドキュメントでは、_起点ノード_ から _終点ノード_ への支払いを送るために使われるオニオンルーティングパケットの構築方法を説明します。パケットは _ホップ_ と呼ばれる複数の中継ノードを経由してルーティングされます。

ルーティング方式は [Sphinx][sphinx] 構造を基にしており、ホップごとのペイロードを追加した拡張になっています。

メッセージを転送する中継ノードはパケットの整合性を検証でき、次にどのノードへ転送すべきかも知ることができます。一方で、自身の前後のノード以外にどのノードがルートに含まれているかは知ることができず、ルート全体の長さや自分の位置も把握できません。パケットは各ホップで難読化されるため、ネットワーク層の攻撃者は同一ルートに属するパケットを関連付けられません（同じルートのパケット同士には相関する情報が含まれない）。ただし、トラフィック解析によって攻撃者がパケットを関連付ける可能性まで排除するものではありません。

ルートは起点ノードが構築します。起点ノードは各中継ノードと終点ノードの公開鍵を把握しているため、ECDH を用いて各ノードと共有秘密を確立できます。この共有秘密から、パケットを難読化するための _疑似ランダムバイトストリーム_ と、ペイロードの暗号化や HMAC 計算に用いる複数の _鍵_ が生成されます。HMAC は各ホップでパケットの整合性を保証するために使われます。

各ホップが目にするのは、送信者の身元を隠すために用意された起点ノードの一時鍵だけです。この一時鍵は次のホップへ転送される前に各中継ホップでブラインド化され、ルート上のオニオン同士をリンクできないようにします。

この仕様書では、パケットフォーマットとルーティング機構の _バージョン 0_ について説明します。

ノードは：
  - 実装しているバージョンより新しいバージョンのパケットを受信した場合：
    - 起点ノードにルートの失敗を報告しなければならない。
    - パケットを破棄しなければならない。

# 目次

  * [規約](#conventions)
  * [鍵生成](#key-generation)
  * [疑似ランダムバイトストリーム](#pseudo-random-byte-stream)
  * [パケット構造](#packet-structure)
    * [ペイロード形式](#payload-format)
    * [基本的なマルチパートペイメント](#basic-multi-part-payments)
  * [ルートブラインディング](#route-blinding)
    * [暗号化された受信者データ内: encrypted_data_tlv](#Inside-encrypted_recipient_data-encrypted_data_tlv)
  * [支払いの受け入れと転送](#accepting-and-forwarding-a-payment)
    * [最後のノードのペイロード](#payload-for-the-last-node)
    * [非厳密な転送](#non-strict-forwarding)
  * [共有秘密](#shared-secret)
  * [エフェメラルオニオンキーのブラインディング](#blinding-ephemeral-onion-keys)
  * [パケット構築](#packet-construction)
  * [オニオン復号](#onion-decryption)
  * [フィラー生成](#filler-generation)
  * [エラーの返却](#returning-errors)
    * [失敗メッセージ](#failure-messages)
    * [失敗コードの受信](#receiving-failure-codes)
  * [成功した支払いのホールド時間](#hold-times-for-successful-payments)
  * [オニオンメッセージ](#onion-messages)
  * [`max_htlc_cltv` の選択](#max-htlc-cltv-selection)
  * [テストベクター](#test-vector)
    * [エラーの返却](#returning-errors)
  * [参考文献](#references)
  * [著者](#authors)

# 規約

本ドキュメント全体で守られる規約は以下のとおりです。

 - HMAC: パケットの整合性検証には、[FIPS 198 Standard][fips198]/[RFC 2104][RFC2104] で定義された Keyed-Hash Message Authentication Code を `SHA256` ハッシュアルゴリズムとともに用います。
 - 楕円曲線: 楕円曲線が関わる計算ではすべて、[`secp256k1`][sec2] で規定された Bitcoin 曲線を使用します。
 - 疑似ランダムストリーム: 疑似ランダムバイトストリームは [`ChaCha20`][rfc8439] で生成します。生成にあたっては、固定の 96 ビットゼロノンス (`0x000000000000000000000000`)、共有秘密から導出した鍵、および所望の出力長と同じ長さの `0x00` バイトストリームをメッセージとして用います。
 - 用語 _origin node_ と _final node_ はそれぞれ最初のパケット送信者と最後のパケット受信者を指します。
 - 用語 _hop_ と _node_ は混用されることがありますが、_hop_ は通常ルート上の中継ノードを指し、両端のノードを指しません。
        _origin node_ --> _hop_ --> ... --> _hop_ --> _final node_
 - 用語 _processing node_ は、現在転送パケットを処理しているルート上の当該ノードを指します。
 - 用語 _peers_ はオーバーレイネットワーク上で直接隣接するホップのみを指します。より具体的には、_sending peers_ がパケットを _receiving peers_ に転送します。
 - ルート上の各ホップには可変長の `hop_payload` があります。
    - 可変長の `hop_payload` の先頭には、`bigsize` でエンコードされた長さ（プレフィックスと末尾の HMAC を除いたバイト数）が付与されます。

# 鍵生成

共有秘密から、暗号化および検証用の複数の鍵が導出されます。

 - _rho_: 各ホップの情報を難読化するための疑似ランダムバイトストリームを生成する鍵として用います
 - _mu_: HMAC 生成時に用います
 - _um_: エラー報告時に用います
 - _pad_: ミックスヘッダパケットの初期ランダムフィラーバイトを生成するために用います

鍵生成関数は、鍵タイプ (_rho_=`0x72686F`, _mu_=`0x6d75`, _um_=`0x756d`, _pad_=`0x706164`) と 32 バイトの秘密を入力として受け取り、32 バイトの鍵を返します。

鍵は、対応する鍵タイプ（_rho_, _mu_, _um_, _pad_ のいずれか）を HMAC 鍵に、32 バイトの共有秘密をメッセージにして、`SHA256` を用いた HMAC を計算することで得られます。算出された HMAC がそのまま鍵となります。

鍵タイプに C スタイルの `0x00` 終端バイトを含めない点に注意してください。例えば _rho_ の鍵タイプは 4 バイトではなく 3 バイトです。

# 疑似ランダムバイトストリーム

疑似ランダムバイトストリームは、経路上の各ホップでパケットを難読化するために用いられます。これにより各ホップは、自分の次のホップのアドレスと HMAC だけを取り出すことができます。生成方法は、必要な長さの `0x00` バイトストリームを、共有秘密から導出した鍵と 96 ビットのゼロノンス (`0x000000000000000000000000`) で初期化した `ChaCha20` で暗号化するというものです。

鍵が再利用されないため、固定ノンスを用いても安全です。

# パケット構造

パケットは次の 4 つのセクションで構成されます。

 - `version` バイト
 - 共有秘密の生成に使う 33 バイトの圧縮形式 `secp256k1` `public_key`
 - 複数の可変長 `hop_payload` ペイロードからなる 1300 バイトの `hop_payloads`
 - パケットの整合性検証用の 32 バイトの `hmac`

パケットのネットワーク表現は、各セクションを連続した 1 本のバイトストリームにシリアライズし、受信者へ送信することで作られます。パケットのサイズが固定であるため、接続上で転送する際に長さをプレフィックスとして付ける必要はありません。

パケット全体の構造は次のとおりです。


1. type: `onion_packet`
2. data:
   * [`byte`:`version`]
   * [`point`:`public_key`]
   * [`1300*byte`:`hop_payloads`]
   * [`32*byte`:`hmac`]

本仕様（_version 0_）では、`version` は `0x00` という定数値です。

`hop_payloads` フィールドは、難読化されたルーティング情報と対応する HMAC を保持する構造体です。長さは 1300 バイトで、以下の構造を持ちます。

1. type: `hop_payloads`
2. data:
   * [`bigsize`:`length`]
   * [`length*byte`:`payload`]
   * [`32*byte`:`hmac`]
   * ...
   * `filler`

ここで `length`、`payload`、`hmac` は各ホップに対して繰り返されます。`filler` は、[フィラー生成](#filler-generation) で詳述するように、難読化された決定論的なパディングで構成されます。さらに `hop_payloads` は、各ホップで段階的に難読化されます。

起点ノードは `payload` フィールドを通じて、各ホップで転送される HTLC の経路と構造を指定できます。`payload` はパケット全体の HMAC で保護されているため、HTLC 送信者（起点ノード）と経路上の各ホップとのペアの関係で、その情報は完全に認証されます。

このエンドツーエンド認証を利用することで、各ホップは HTLC パラメータを `payload` 内の指定値と突き合わせ、送信ピアが不正に作られた HTLC を転送していないことを確認できます。

`payload` の TLV 値は 2 バイト未満にならないため、`length` の値 0 と 1 は予約されています（`0` は既にサポート対象外となったレガシーフォーマットを示し、`1` は将来の利用のために予約されています）。

### `payload` フォーマット

`payload` は [BOLT #1](01-messaging.md#type-length-value-format) で定義された Type-Length-Value 形式に従って構成されます。

1. `tlv_stream`: `payload`
2. types:
    1. type: 2 (`amt_to_forward`)
    2. data:
        * [`tu64`:`amt_to_forward`]
    1. type: 4 (`outgoing_cltv_value`)
    2. data:
        * [`tu32`:`outgoing_cltv_value`]
    1. type: 6 (`short_channel_id`)
    2. data:
        * [`short_channel_id`:`short_channel_id`]
    1. type: 8 (`payment_data`)
    2. data:
        * [`32*byte`:`payment_secret`]
        * [`tu64`:`total_msat`]
    1. type: 10 (`encrypted_recipient_data`)
    2. data:
        * [`...*byte`:`encrypted_recipient_data`]
    1. type: 12 (`current_path_key`)
    2. data:
        * [`point`:`path_key`]
    1. type: 16 (`payment_metadata`)
    2. data:
        * [`...*byte`:`payment_metadata`]
    1. type: 18 (`total_amount_msat`)
    2. data:
        * [`tu64`:`total_msat`]

`short_channel_id` は、メッセージのルーティングに使用する送信チャネルの ID です。受信ピアはこのチャネルの反対側を運用している必要があります。

`amt_to_forward` は、ルーティング情報で指定された次の受信ピアまたは最終宛先に転送する金額（ミリサトシ単位）を表します。

非最終ノードの場合、この値には受信ピア向けに起点ノードが計算した _手数料_ が含まれます。手数料は受信ピアが広告している手数料スキーム（[BOLT #7](07-routing-gossip.md#htlc-fees) を参照）に従って計算されます。

`outgoing_cltv_value` は、パケットを運ぶ _送信_ HTLC が持つべき CLTV 値です。このフィールドが含まれることで、ホップは起点ノードが指定した情報と転送される HTLC のパラメータの双方を認証でき、起点ノードが現行の `cltv_expiry_delta` 値を使っていることも確認できます。

値が一致しない場合、転送ノードが意図された HTLC 値を改ざんしたか、起点ノードが古い `cltv_expiry_delta` を持っていることが示唆されます。

要件は、最終ノードかどうかにかかわらず予期しない `outgoing_cltv_value` への応答を統一することで、ルート内での自分の位置を漏らさないようにすることを目的としています。

### 要件

`encrypted_recipient_data` の作成者（通常は支払いの受取人）は：

  - ブラインドルート内の各ノード（自分自身を含む）に対して `encrypted_data_tlv` を作成しなければならない。
  - 各非最終ノードに対して `encrypted_data_tlv.payment_relay` を含めなければならない。
  - 各非最終ノードに対して `encrypted_data_tlv.short_channel_id` または `encrypted_data_tlv.next_node_id` のいずれか一つだけを含めなければならない。
  - 各非最終ノードに対して `encrypted_data_tlv.payment_constraints` を設定しなければならず、最終ノードに対しては設定してよい：
    - `max_cltv_expiry` には、ルートが利用可能な最大ブロック高を設定する。これは、最終ノードが選んだ「ルートを失効させたい `max_cltv_expiry` 高」に最終ノードの `min_final_cltv_expiry_delta` を加え、さらに各ホップで `encrypted_data_tlv.payment_relay.cltv_expiry_delta` を順次加えていくことで求める。
    - `htlc_minimum_msat` には、各ノードが許容しうる最小 HTLC 値の最大値を設定する。
  - `encrypted_data_tlv.allowed_features` を設定する場合：
    - 空の配列に設定しなければならない。
  - ルートの合計手数料と合計 CLTV デルタを以下のように計算し、送信者に伝えなければならない：
    - `total_fee_base_msat(n+1) = (fee_base_msat(n+1) * 1000000 + total_fee_base_msat(n) * (1000000 + fee_proportional_millionths(n+1)) + 1000000 - 1) / 1000000`
    - `total_fee_proportional_millionths(n+1) = ((total_fee_proportional_millionths(n) + fee_proportional_millionths(n+1)) * 1000000 + total_fee_proportional_millionths(n) * fee_proportional_millionths(n+1) + 1000000 - 1) / 1000000`
    - `total_cltv_delta = cltv_delta(0) + cltv_delta(1) + ... + cltv_delta(n) + min_final_cltv_expiry_delta`
  - [ルートブラインディング](#route-blinding) で要求されているとおり、`encrypted_data_tlv` から `encrypted_recipient_data` を作成しなければならない。

TLV `payload` の作成者は：

- ブラインドルート内の各ノードについて：
  - 受取人が提供した `encrypted_recipient_data` を含めなければならない。
  - ブラインドルートの最初のノードについて：
    - 受取人が提供した `path_key` を `current_path_key` として含めなければならない。
  - 最終ノードである場合：
    - `amt_to_forward`、`outgoing_cltv_value`、`total_amount_msat` を含めなければならない。
    - `outgoing_cltv_value` に設定する値については：
      - 現在のブロック高をベース値として用いなければならない。
      - プライバシー向上のために [ランダムオフセット](07-routing-gossip.md#recommendations-for-routing) を加える場合：
        - そのオフセットをベース値に加えるべきである。
  - 他の TLV フィールドを含めてはならない。
- ブラインドルート外の各ノードについて：
  - `amt_to_forward` と `outgoing_cltv_value` を含めなければならない。
  - 各非最終ノードについて：
    - `short_channel_id` を含めなければならない。
    - `payment_data` を含めてはならない。
  - 最終ノードについて：
    - `short_channel_id` を含めてはならない。
    - 受取人が `payment_secret` を提供した場合：
      - `payment_data` を含めなければならない。
      - `payment_secret` には提供された値を設定しなければならない。
      - `total_msat` には送信する総額を設定しなければならない。
    - 受取人が `payment_metadata` を提供した場合：
      - 各 HTLC に `payment_metadata` を含めなければならない。
      - 固定オニオンサイズによる暗黙の制限以外、`payment_metadata` のサイズに制限を設けてはならない。

読み手は：

- `encrypted_recipient_data` が存在する場合：
  - 受信した `update_add_htlc` に `path_key` が設定されている場合：
    - `current_path_key` が存在する場合はエラーを返さなければならない。
    - その `path_key` を復号のための `path_key` として用いなければならない。
  - それ以外の場合：
    - `current_path_key` が存在しない場合はエラーを返さなければならない。
    - その `current_path_key` を復号のための `path_key` として用いなければならない。
    - エラーを返す前にランダムな遅延を加えるべきである。
  - [ルートブラインディング](#route-blinding) で説明されているとおり、`path_key` で `encrypted_recipient_data` を復号できない場合はエラーを返さなければならない。
  - `payment_constraints` が存在する場合：
    - 以下のいずれかに該当するときはエラーを返さなければならない：
      - 有効期限が `encrypted_recipient_data.payment_constraints.max_cltv_expiry` を超えている。
      - 金額が `encrypted_recipient_data.payment_constraints.htlc_minimum_msat` を下回っている。
  - `allowed_features` が欠けている場合：
    - 存在して空配列を含むものとしてメッセージを処理しなければならない。
  - 以下のいずれかに該当する場合はエラーを返さなければならない：
    - `encrypted_recipient_data.allowed_features.features` に未知の機能ビットが含まれている（奇数であっても）。
    - `encrypted_recipient_data` に `short_channel_id` と `next_node_id` の両方が含まれている。
    - 支払いが `encrypted_recipient_data.allowed_features.features` に含まれない機能を使っている。
  - 最終ノードでない場合：
    - ペイロードに `encrypted_recipient_data` と `current_path_key` 以外の TLV フィールドが含まれている場合はエラーを返さなければならない。
    - `encrypted_recipient_data` が `short_channel_id` か `next_node_id` を含まない場合はエラーを返さなければならない。
    - `encrypted_recipient_data` が `payment_relay` を含まない場合はエラーを返さなければならない。
    - `encrypted_recipient_data.payment_relay` の値を用いて、`amt_to_forward` と `outgoing_cltv_value` を以下のように計算しなければならない：
      - `amt_to_forward = ((amount_msat - fee_base_msat) * 1000000 + 1000000 + fee_proportional_millionths - 1) / (1000000 + fee_proportional_millionths)`
      - `outgoing_cltv_value = cltv_expiry - payment_relay.cltv_expiry_delta`
  - 最終ノードである場合：
    - ペイロードに `encrypted_recipient_data`、`current_path_key`、`amt_to_forward`、`outgoing_cltv_value`、`total_amount_msat` 以外の TLV フィールドが含まれている場合はエラーを返さなければならない。
    - `amt_to_forward`、`outgoing_cltv_value`、`total_amount_msat` のいずれかが存在しない場合はエラーを返さなければならない。
    - `amt_to_forward` が支払いとして期待する金額を下回る場合はエラーを返さなければならない。
    - 受信した `cltv_expiry` が `outgoing_cltv_value` より小さい場合はエラーを返さなければならない。
    - 受信した `cltv_expiry` が `current_block_height` + `min_final_cltv_expiry_delta` より小さい場合はエラーを返さなければならない。
- それ以外の場合（ブラインドルートの一部ではない）：
  - 受信した `update_add_htlc` に `path_key` が設定されているか、`current_path_key` が存在する場合はエラーを返さなければならない。
  - `amt_to_forward` または `outgoing_cltv_value` が存在しない場合はエラーを返さなければならない。
  - 最終ノードでない場合：
    - 以下のいずれかに該当する場合はエラーを返さなければならない：
      - `short_channel_id` が存在しない。
      - チャネル `short_channel_id` によって指されるピアに HTLC を転送できない。
      - 受信 `amount_msat` - `fee` < `amt_to_forward`（`fee` は [BOLT #7](07-routing-gossip.md#htlc-fees) で説明される広告手数料）。
      - `cltv_expiry` - `cltv_expiry_delta` < `outgoing_cltv_value`。
- 最終ノードである場合：
  - `total_msat` が存在しない場合は `amt_to_forward` と等しいものとして扱わなければならない。
  - 以下のいずれかに該当する場合はエラーを返さなければならない：
    - 受信 `amount_msat` < `amt_to_forward`。
    - 受信 `cltv_expiry` < `outgoing_cltv_value`。
    - 受信 `cltv_expiry` < `current_block_height` + `min_final_cltv_expiry_delta`。

マルチパート支払いに関する追加要件は [こちら](#basic-multi-part-payments)、ブラインド支払いに関する追加要件は [こちら](#route-blinding) に記載されています。

### 基本的なマルチパート支払い

HTLC は、より大きな「マルチパート」支払いの一部となることがあります。この「基本」的なアトミックマルチパス支払いでは、すべての経路で同じ `payment_hash` を使用します。

`amt_to_forward` はその HTLC 単体の金額です。`total_msat` フィールドにより大きな値が含まれている場合、それは最終的な送信者が残額を後続の HTLC で送ることを約束していることを意味します。同じプリイメージを共有するこれらの未決 HTLC をまとめて「HTLC セット」と呼びます。

`total_msat` を伝えるために使える TLV フィールドは 2 種類あります。後者の `total_amount_msat` は、`payment_secret` が意味を持たないブラインドパスのために導入されました。

`payment_metadata` は、不正な支払い詳細をできるだけ早く検知できるよう、すべての支払いパートに含める必要があります。

#### 要件

書き手は：
  - インボイスが `basic_mpp` 機能を提供している場合：
    - インボイスを支払うために複数の HTLC を送信してよい。
    - セット内のすべての HTLC で同じ `payment_hash` を使用しなければならない。
    - すべての支払いをほぼ同時に送るべきである。
    - 各 HTLC ごとに受取人への異なる経路を使うよう努めるべきである。
    - 失敗した HTLC を再試行または再分割するべきである。
    - インボイスが `amount` を指定している場合：
       - `total_msat` をその `amount` 以上、`amount` の 2 倍以下に設定しなければならない。
    - そうでない場合：
      - `total_msat` には支払いたい金額を設定しなければならない。
    - 受取人に到着する HTLC セットの合計 `amt_to_forward` が `total_msat` 以上となるようにしなければならない。
    - HTLC セットの合計 `amt_to_forward` が既に `total_msat` 以上に達している場合、追加の HTLC を送信してはならない。
    - `payment_secret` を含めなければならない。
  - そうでない場合：
    - `total_msat` を `amt_to_forward` と等しく設定しなければならない。

最終ノードは：
  - [失敗メッセージ](#failure-messages) の要件に従って HTLC を失敗させなければならない。
    - 注：そこで言う「支払われた金額」とは `total_msat` フィールドのことを指します。
  - `basic_mpp` をサポートしない場合：
    - `total_msat` が `amt_to_forward` と完全に等しくないならば HTLC を失敗させなければならない。
  - そうではなく `basic_mpp` をサポートする場合：
    - その `payment_hash` に対応する HTLC セットに追加しなければならない。
    - セット内のすべての HTLC で `total_msat` が同一でないならば、HTLC セット全体を失敗させるべきである。
    - この HTLC セットの合計 `amt_to_forward` が `total_msat` 以上の場合：
      - HTLC セット内のすべての HTLC を履行するべきである。
    - そうではなく、合計 `amt_to_forward` が `total_msat` 未満の場合：
      - HTLC セット内のいかなる HTLC も履行してはならない。
      - 合理的なタイムアウトの後に HTLC セット内のすべての HTLC を失敗させなければならない。
        - 最初の HTLC から少なくとも 60 秒は待つべきである。
        - 失敗メッセージには `mpp_timeout` を使うべきである。
      - セット内のすべての HTLC に `payment_secret` を要求しなければならない。
    - HTLC セット内のいずれかの HTLC を履行する場合：
       - HTLC セット全体を履行しなければならない。

#### 根拠

`basic_mpp` が存在する場合、他の分割支払いが揃うのを待つために遅延が生じます。合計金額は、単一支払いと同様に、希望する支払いを満たすに足りる必要があります。同時に、サービス拒否を避けるために合理的な範囲に抑える必要もあります。

インボイスが必ずしも金額を指定するとは限らず、支払者は最終金額にノイズを加えることもできるため、合計金額は明示的に伝える必要があります。要件は若干の超過も許容しています。これは、分割時にノイズを加えやすくするためであり、また送信者同士が本当に独立しているケース（例えば友人同士が請求書を分担する場合）にも対応するためです。

希望する経路のチャネルが持つ `htlc_minimum_msat` の関係で、ノードは希望額より多く支払う必要が生じることがあります。そのため、指定した `total_msat` より多く支払うことが許容されています。そうでなければ、ある経路に沿って支払いを再試行する際にノードの取り得る経路が制約されてしまうからです。ただし、個々の HTLC は「支払総額と `total_msat` の差」を下回ってはなりません。

合意した合計を超えた後に追加 HTLC を送らない制約は、すべての分割支払いが到着する前にプリイメージが解放されるのを防ぎます。さもなければ、中継ノードが未到着の分割支払いを直ちに請求できてしまいます。

実装側は、金額条件を満たす HTLC セットを履行しないという選択も可能です（他の失敗やインボイスのタイムアウトなど）。ただし、その一部だけを履行すると、残りを中継ノードが単純に請求できてしまう点に注意が必要です。

## ルートブラインディング

1. サブタイプ: `blinded_path`
2. データ:
   * [`sciddir_or_pubkey`:`first_node_id`]
   * [`point`:`first_path_key`]
   * [`byte`:`num_hops`]
   * [`num_hops*blinded_path_hop`:`path`]

1. サブタイプ: `blinded_path_hop`
2. データ:
    * [`point`:`blinded_node_id`]
    * [`u16`:`enclen`]
    * [`enclen*byte`:`encrypted_recipient_data`]

ブラインドパスは以下の要素で構成されます。
1. 最初の導入点（`first_node_id`）
2. その最初のノード ID と秘密を共有するための初期鍵（`first_path_key`）
3. 一連の調整済みノード ID（`path.blinded_node_id`）
4. 次のホップを伝えるために各ノード向けに暗号化されたバイナリブロブ（`path.encrypted_recipient_data`）

例えば、Dave は Alice に対して、公開ノード Bob と Carol を経由して自分に到達してほしいとします。彼は Bob、Carol、そして自分自身のための鍵チェーン（"path_keys"）を作り、それぞれと秘密を共有できるようにします。これらの鍵は単純なチェーンになっているので、各ノードは明示的に教えられなくても次の `path_key` を導出できます。

これらの共有秘密から、Dave は 3 つの `encrypted_data_tlv` を作成して暗号化します。
1. encrypted_data_bob: Bob に Carol へ転送するよう指示するもの
2. encrypted_data_carol: Carol に Dave 自身へ転送するよう指示するもの
3. encrypted_data_dave: パスが使用されたことを示し、Dave が含めたい任意のメタデータを保持するもの

ノード ID を隠すため、彼は共有秘密から 3 つのブラインディングファクタも導出し、Bob を Bob'、Carol を Carol'、Dave を Dave' に変換します。

これが Alice に渡される `blinded_path` です。

1. `first_node_id`: Bob
2. `first_path_key`: Bob 用の最初のパスキー
3. `path`: [Bob', encrypted_data_bob], [Carol', encrypted_data_carol], [Dave', encrypted_data_dave]

Bob はおそらく Alice の直接のピアではないため、Alice が Bob に届くオニオンを構築する方法は 2 通りあります。詳細は後述の要件で説明します。

ただし Bob 以降の経路は常に同じです。Bob は導出した `path_key` をオニオンとともに Carol に送ります。Carol は `path_key` から（Alice は Carol ではなく Carol' 向けに暗号化しているため）オニオンの調整値を導き、復号します。さらに `encrypted_data_tlv` を復号する鍵も導出し、Dave へ転送するよう指示されます（場合によっては Dave が指定した追加の制約も含まれます）。

### 要件

ブラインドパスの作成者（つまり受取人）は、送信者がオニオンを構築するため、また中継ノードが指示を読むために、これを作成している点に注意してください。そのため読み手のセクションが二つに分かれています。

`blinded_path` の作成者は：

- 自身（$`N_r`$）への有効な経路を作成しなければならない。すなわち $`N_0 \rightarrow N_1 \rightarrow ... \rightarrow N_r`$。
- `first_node_id` を $`N_0`$ に設定しなければならない。
- 次のアルゴリズムでルート内の各ノード向けに一連の ECDH 共有秘密を作成しなければならない：
  - $`e_0 \leftarrow \{0;1\}^{256}`$ （$`e_0`$ は CSPRNG から取得すべきである）
  - $`E_0 = e_0 \cdot G`$
  - ルート内の各ノードについて：
    - $`N_i = k_i * G`$ を `node_id` とする（$`k_i`$ は $`N_i`$ の秘密鍵）。
    - $`ss_i = SHA256(e_i * N_i) = SHA256(k_i * E_i)`$（$`N_r`$ と $`N_i`$ のみが知る ECDH 共有秘密）。
    - $`rho_i = HMAC256(\text{"rho"}, ss_i)`$（$`N_r`$ が $`N_i`$ 向けに `encrypted_recipient_data` を暗号化する鍵）。
    - $`e_{i+1} = SHA256(E_i || ss_i) * e_i`$（一時的な秘密のパスキー。$`N_r`$ だけが知る）。
    - $`E_{i+1} = SHA256(E_i || ss_i) * E_i`$（`path_key`。注：$`N_i`$ は $`e_i`$ を知ってはならない）。
- `first_path_key` を $`E_0`$ に設定しなければならない。
- 次のアルゴリズムで各ノード向けに一連のブラインドノード ID $`B_i`$ を作成しなければならない：
  - $`B_i = HMAC256(\text{"blinded\_node\_id"}, ss_i) * N_i`$（$`N_i`$ 用のブラインド `node_id`。秘密鍵は $`N_i`$ のみが知る）。
  - `path` 内の各 `blinded_path_hop` の `blinded_node_id` を $`B_i`$ に設定しなければならない。
- $`E_{i+1}`$ を別の値に差し替えてもよい。差し替える場合は：
  - `encrypted_data_tlv[i].next_path_key_override` を $`E_{i+1}`$ に設定しなければならない。
- ルートが正しい文脈で使用され、自分が作成したものであることを検証するために、`encrypted_data_tlv[r].path_id` にプライベートデータを格納してよい。
- すべての `encrypted_data_tlv[i]` が同じ長さになるよう、パディングデータを追加するべきである。
- 各 `encrypted_data_tlv[i]` を、対応する $`rho_i`$ 鍵と全ゼロのノンスを用いた ChaCha20-Poly1305 で暗号化し、`encrypted_recipient_data[i]` を生成しなければならない。
- ルート長を隠すために、ルート末尾に追加の「ダミー」ホップ（受信時には無視される）を加えてもよい。

`blinded_path` の読み手は：

- `first_node_id` に到達するために、自分のオニオンペイロードを先頭に付加しなければならない。
- `path` 内の各オニオンペイロードに、対応する `encrypted_recipient_data` を含めなければならない。
- `path` の最初のエントリについて：
  - 支払いを送信する場合：
    - `first_node_id` 向けの非ブラインドオニオン支払いを作成し、`first_path_key` を `current_path_key` に含めるべきである。
  - そうでない場合：
    - 最初のブラインドパスオニオンを最初の `blinded_node_id` に対して暗号化しなければならない。
    - 直前のオニオンペイロードで `next_path_key_override` を `first_path_key` に設定しなければならない。
- `path` の以降の各エントリについて：
  - オニオンを対応する `blinded_node_id` に対して暗号化しなければならない。

`encrypted_recipient_data` の読み手は：

- 次を計算しなければならない：
  - $`ss_i = SHA256(k_i * E_i)`$（標準的な ECDH）
  - $`b_i = HMAC256(\text{"blinded\_node\_id"}, ss_i) * k_i`$
  - $`rho_i = HMAC256(\text{"rho"}, ss_i)`$
- $`rho_i`$ を鍵に、全ゼロのノンスを用いた ChaCha20-Poly1305 で `encrypted_recipient_data` フィールドを復号しなければならない。
- `encrypted_recipient_data` フィールドが欠落、または `encrypted_data_tlv` に復号できない、もしくは未知の偶数フィールドを含む場合：
  - エラーを返さなければならない。
- `encrypted_data_tlv` に `next_path_key_override` が含まれる場合：
  - それを次の `path_key` として用いなければならない。
- そうでない場合：
  - $`E_{i+1} = SHA256(E_i || ss_i) * E_i`$ を次の `path_key` として用いなければならない。
- オニオンを転送し、次ノード宛ての Lightning メッセージに次の `path_key` を含めなければならない。
- 最終受取人である場合：
  - `path_id` が自身がこの目的で作成したブラインドルートと一致しない場合は、メッセージを無視しなければならない。

### 根拠

ルートブラインディングは、受取人の匿名性を確保するための軽量な技法です。ランデブールーティングと比べてより柔軟で、ルート内ノードの公開鍵をランダムな公開鍵で置き換えつつ、各ホップのオニオンに何を入れるかは送信者に委ねます。ブラインドルートは、用途によっては再利用可能です（例：オニオンメッセージ）。

ブラインドルート内の各ノードは、オニオンと `encrypted_recipient_data` ペイロードを復号するために $`E_i`$ を受け取る必要があります。

異なるノードが生成した 2 本のブラインドルートを連結する場合、1 本目の最終ノードは 2 本目の最初の `path_key` を知る必要があります。この情報を伝えるには `next_path_key_override` フィールドを使わなければなりません。理屈としてはこの方式を支払い（オニオンメッセージ以外）にも使えますが、`first_node_id` への到達には非ブラインドのパスを用い、そこで `current_path_key` を使うことを推奨します。これにより、ノードは自分が導入点として使われていると識別でき、その地点に到達するためにブラインドパス対応を要求しなくて済み、また支払いの非ブラインド部分で意味のあるエラーを返せるようになります。

最終受取人は、ブラインドルートが正しい文脈（特定の支払いなど）で使われ、自分が作成したものであることを必ず検証しなければなりません。そうしないと、悪意ある送信者が「真の受取人かもしれない」全ノードに対して異なるブラインドルートを作成し、受け入れられるまで試行する攻撃が可能になります。これを防ぐには、受取人が $`E_r`$ と文脈情報（`payment_hash` など）を保存しておき、オニオン受信時に一致を確認します。あるいは、追加ストレージを避けるために、`path_id` フィールドに送信者がアクセスできないプライベート文脈情報（例えば `payment_preimage`）を入れ、オニオン受信時に検証する方法もあります。後者の場合、送信者から到達不能なプライベート情報を使うことが重要です。

導入点はブラインドルートから失敗を受け取った際、エラーを転送する前にランダムな遅延を加えるべきです。失敗はプロービング試行である可能性が高く、メッセージのタイミングから攻撃者が最終受取人までの距離を推測する手掛かりを得かねないためです。

なお、ブラインドルート内のノードは `update_fail_malformed_htlc` で失敗を返すため、アトリビューションデータ経由のタイミング情報を送信者に提供できませんし、する手段もありません。

`padding` フィールドは、すべての `encrypted_recipient_data` の長さを揃えるために利用できます。これは、ブラインドルート末尾にダミーホップを追加して、送信者がどのノードが最終受取人かを特定できないようにするのに特に有効です。

支払いでルートブラインディングを使う場合、ブラインドノードが支払いに適用する手数料や有効期限は、送信者ではなく受取人が指定します。受取人はまた、悪意あるノードがブラインドノードの正体を解明するプロービング攻撃を防ぐため、そのルートを通る支払いに追加制約を課します。中継ノードが手数料を更新して支払いを拒否するリスク（ルート内ノードのアンブラインドにつながり得る）を抑えるため、`payment_constraints.max_cltv_expiry` を設定してブラインドルートの寿命を制限すべきです。

### `encrypted_recipient_data` の内部: `encrypted_data_tlv`

`encrypted_recipient_data` は、特定のブラインドノード向けに暗号化された TLV ストリームで、以下の TLV フィールドを含み得ます。

1. `tlv_stream`: `encrypted_data_tlv`
2. types:
    1. type: 1 (`padding`)
    2. data:
        * [`...*byte`:`padding`]
    1. type: 2 (`short_channel_id`)
    2. data:
        * [`short_channel_id`:`short_channel_id`]
    1. type: 4 (`next_node_id`)
    2. data:
        * [`point`:`node_id`]
    1. type: 6 (`path_id`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 8 (`next_path_key_override`)
    2. data:
        * [`point`:`path_key`]
    1. type: 10 (`payment_relay`)
    2. data:
        * [`u16`:`cltv_expiry_delta`]
        * [`u32`:`fee_proportional_millionths`]
        * [`tu32`:`fee_base_msat`]
    1. type: 12 (`payment_constraints`)
    2. data:
        * [`u32`:`max_cltv_expiry`]
        * [`tu64`:`htlc_minimum_msat`]
    1. type: 14 (`allowed_features`)
    2. data:
        * [`...*byte`:`features`]

#### 根拠

`encrypted_recipient_data` は、メッセージをどう処理するかをノードに伝える指示を含み、最終受取人が送信者に渡すために作成します（送信者自身が作成することも可能で、転送ノードからは区別できません）。支払いオニオンでもオニオンメッセージのオニオンでも使用されます。詳細は [ルートブラインディング](#route-blinding) を参照してください。

# 支払いの受け入れと転送

ノードはペイロードをデコードした後、ローカルで支払いを受け入れるか、ペイロード内で次のホップとして示されたピアに転送します。

## 非厳密な転送

ノードは、`short_channel_id` で指定されたものとは異なる送信チャネルを介して HTLC を転送してよい。ただし、受信側は `short_channel_id` で意図されたものと同じノード公開鍵を持つ必要があります。したがって、`short_channel_id` がノード A と B を結ぶものであれば、A と B を結ぶどのチャネルを使っても HTLC を転送できます。これに従わない場合、受信側はオニオンパケットの次のホップを復号できません。

### 根拠

2 つのピア間に複数のチャネルがある場合、下流ノードはどのチャネルでパケットが送られても次ホップ用ペイロードを復号できます。

非厳密な転送を実装するノードは、特定ピアとのチャネル帯域をリアルタイムに評価し、その時点でローカルに最適なチャネルを選べます。

例えば、A と B を結ぶ `short_channel_id` 指定のチャネルが転送時に十分な帯域を持たない場合、A は十分な帯域を持つ別チャネルを使えます。これにより、`short_channel_id` での帯域不足による失敗を回避でき、送信者が A-B 間のチャネル違いだけの同じルートを再試行する手間を減らし、支払いの遅延も小さくできます。

非厳密な転送によって、ノードは公開チャネルグラフに載っていないプライベートチャネルを通じて受信ノードへ届けることもできるようになります。

### 推奨

非厳密な転送を採用する実装は、同じピアと結ばれるすべてのチャネルに同じ手数料スケジュールを適用することを検討すべきです。送信者は総コストが最も低くなるチャネルを選びがちであるため、ポリシーがチャネル間で異なると、送信者にとって最適な手数料スケジュールに基づいた料金しか得られなくなり、同じピアとの全チャネルで集約帯域を提供しているにもかかわらず、期待した手数料収入から乖離する恐れがあります。

代替案として、ポリシーが似通ったチャネルに対してのみ非厳密転送を適用し、代替チャネル使用による手数料収入のブレを抑える方法もあります。

## 最終ノード向けのペイロード

ルートを構築する際、起点ノードは最終ノード向けのペイロードに以下の値を設定しなければなりません。

* `payment_secret`: 受取人が指定した支払いシークレット（例：[BOLT #11](11-payment-encoding.md) インボイスの `payment_secret`）に設定する
* `outgoing_cltv_value`: 受取人が指定した最終期限（例：[BOLT #11](11-payment-encoding.md) インボイスの `min_final_cltv_expiry_delta`）に設定する
* `amt_to_forward`: 受取人が指定した最終金額（例：[BOLT #11](11-payment-encoding.md) インボイスの `amount`）に設定する

これにより最終ノードはこれらの値を検証し、必要であればエラーを返せます。同時に、最後から 2 番目のノードによるプロービング攻撃の可能性も排除されます。そうした攻撃は、金額や期限を変えた HTLC を再送して受信ピアが最終ノードかどうかを暴くことを狙うものです。最終ノードは受信 HTLC からオニオンペイロードを取り出し、その値を HTLC の値と突き合わせます。詳細は後述の [エラーの返却](#returning-errors) を参照してください。

上記の理由がなければ、最終ノードは支払いを転送しないので、ペイロードを単に破棄することもできます。

# 共有秘密

起点ノードは、送信者のホップ向け一時鍵とそのホップのノード ID 鍵で楕円曲線ディフィー・ヘルマン (Elliptic-curve Diffie-Hellman) を行うことで、ルート上の各ホップと共有秘密を確立します。得られた曲線上の点を圧縮形式にシリアライズし、`SHA256` でハッシュした出力が 32 バイトの共有秘密として用いられます。

楕円曲線ディフィー・ヘルマン (ECDH) は、EC 秘密鍵と EC 公開鍵から曲線上の点を出力する演算です。本プロトコルでは `libsecp256k1` に実装された `secp256k1` 楕円曲線上の ECDH バリアントを使います。パケット構築時、送信者は一時秘密鍵とそのホップの公開鍵を ECDH の入力に用います。一方、パケット転送時、ホップは一時公開鍵と自身のノード ID 秘密鍵を入力に用います。ECDH の性質により、両者は同じ値に至ります。

# 一時オニオン鍵のブラインド化

ルート上の複数ホップが目にする一時公開鍵によってホップ同士をリンクできないようにするため、各ホップで鍵をブラインド化します。ブラインド化は決定論的に行われ、送信者はパケット構築時に対応するブラインド化済み秘密鍵を計算できます。

EC 公開鍵のブラインド化とは、公開鍵を表す EC 点と 32 バイトのブラインディングファクタとのスカラー乗算 1 回のことです。スカラー乗算の可換性により、ブラインド化された秘密鍵は、入力の対応する秘密鍵と同じブラインディングファクタの積になります。

ブラインディングファクタ自体は、一時公開鍵と 32 バイトの共有秘密の関数として計算されます。具体的には、圧縮形式でシリアライズした公開鍵と共有秘密を連結したものを `SHA256` でハッシュした値です。

# パケット構築

以下の例では、_送信ノード_（起点ノード）`n_0` が _受信ノード_（最終ノード）`n_r` にパケットをルーティングしたいと仮定します。まず送信者はルート `{n_0, n_1, ..., n_{r-1}, n_r}` を計算します。`n_0` は送信者自身、`n_r` は最終受信者です。すべての隣接ノード `n_i` と `n_{i+1}` はオーバーレイネットワーク上のピアでなければなりません。次に送信者は `n_1` から `n_r` までの公開鍵を集め、ランダムな 32 バイトの `sessionkey` を生成します。任意で _関連データ_ を渡すこともできます。これはパケットがコミットしつつもパケット自体には含めないデータで、HMAC に含められ、各ホップの整合性検証時に渡される関連データと一致しなければなりません。

オニオンを構築するため、送信者は最初のホップ用の一時秘密鍵 `ek_1` を `sessionkey` で初期化し、それを `secp256k1` の基底点で乗算して対応する一時公開鍵 `epk_1` を求めます。ルート上の各ホップ `k` について、送信者は以下のように共有秘密 `ss_k` と次ホップの一時鍵 `ek_{k+1}` を反復的に計算します。

- 送信者はホップの公開鍵と一時秘密鍵で ECDH を行い曲線上の点を得て、それを `SHA256` でハッシュして共有秘密 `ss_k` を生成する。
- ブラインディングファクタは、一時公開鍵 `epk_k` と共有秘密 `ss_k` を連結したものの `SHA256` ハッシュ。
- 次ホップの一時秘密鍵 `ek_{k+1}` は、現在の一時秘密鍵 `ek_k` にブラインディングファクタを乗算して算出する。
- 次ホップの一時公開鍵 `epk_{k+1}` は、一時秘密鍵 `ek_{k+1}` に基底点を乗算して導出する。

上記の情報がすべて揃うと、送信者はパケットを構築できます。`r` ホップを通るパケットの構築には、32 バイトの一時公開鍵 `r` 個、32 バイトの共有秘密 `r` 個、32 バイトのブラインディングファクタ `r` 個、そして可変長の `hop_payload` ペイロード `r` 個が必要です。構築結果として、1366 バイトのパケットと最初の受信ピアのアドレスが得られます。

パケットの構築はルートの逆順で進めます。つまり最終ホップに対する操作が最初に適用されます。

パケットは CSPRNG（ChaCha20）から得た 1300 バイトの _ランダム_ バイトで初期化されます。前述の _pad_ 鍵は、ChaCha20 ストリームから追加のランダムバイトを取り出すための CSPRNG として使われます。`paddingKey` を得たら、全ゼロのノンスを用いた ChaCha20 で 1300 バイトのランダムバイトを生成し、これを構築するミックスヘッダの初期状態とします。

フィラーは共有秘密から生成します（[フィラー生成](#filler-generation) を参照）。

ルートの各ホップについて、逆順で送信者は以下の操作を行います。


 - ホップの共有秘密を用いて _rho_ 鍵と _mu_ 鍵を生成する。
 - `shift_size` は、`hop_payload` の長さに、その長さの bigsize エンコーディング分と HMAC 長を加えた値として定義する。すなわちペイロード長 `l` のとき、`l < 253` なら `shift_size` は `1 + l + 32`、そうでなければ `3 + l + 32`。
 - `hop_payload` フィールドを `shift_size` バイトだけ右にシフトし、1300 バイトを超えてあふれた末尾の `shift_size` バイトを破棄する。
 - bigsize でシリアライズした長さ、シリアライズした `hop_payload`、`hmac` をそれに続く `shift_size` バイトにコピーする。
 - _rho_ 鍵で 1300 バイトの疑似ランダムバイトストリームを生成し、`hop_payloads` フィールドに `XOR` で適用する。
 - これが最終ホップ、すなわち最初の反復であれば、`hop_payloads` フィールドの末尾をルーティング情報の `filler` で上書きする。
 - 連結した `hop_payloads` と関連データに対して、_mu_ 鍵を HMAC 鍵として用いて次の HMAC を計算する。

最終的に得られる HMAC 値が、ルート内の最初の受信ピアが使用する HMAC になります。

パケット生成は、`version` バイト、最初のホップ用の一時公開鍵、最初のホップ用の HMAC、および難読化された `hop_payloads` を含むシリアライズ済みパケットを返します。

以下の Go コードは、パケット構築の実装例です。

```Go
func NewOnionPacket(paymentPath []*btcec.PublicKey, sessionKey *btcec.PrivateKey,
	hopsData []HopData, assocData []byte) (*OnionPacket, error) {

	numHops := len(paymentPath)
	hopSharedSecrets := make([][sha256.Size]byte, numHops)

	// Initialize ephemeral key for the first hop to the session key.
	var ephemeralKey big.Int
	ephemeralKey.Set(sessionKey.D)

	for i := 0; i < numHops; i++ {
		// Perform ECDH and hash the result.
		ecdhResult := scalarMult(paymentPath[i], ephemeralKey)
		hopSharedSecrets[i] = sha256.Sum256(ecdhResult.SerializeCompressed())

		// Derive ephemeral public key from private key.
		ephemeralPrivKey := btcec.PrivKeyFromBytes(btcec.S256(), ephemeralKey.Bytes())
		ephemeralPubKey := ephemeralPrivKey.PubKey()

		// Compute blinding factor.
		sha := sha256.New()
		sha.Write(ephemeralPubKey.SerializeCompressed())
		sha.Write(hopSharedSecrets[i])

		var blindingFactor big.Int
		blindingFactor.SetBytes(sha.Sum(nil))

		// Blind ephemeral key for next hop.
		ephemeralKey.Mul(&ephemeralKey, &blindingFactor)
		ephemeralKey.Mod(&ephemeralKey, btcec.S256().Params().N)
	}

	// Generate the padding, called "filler strings" in the paper.
	filler := generateHeaderPadding("rho", numHops, hopDataSize, hopSharedSecrets)

	// Allocate and initialize fields to zero-filled slices
	var mixHeader [routingInfoSize]byte
	var nextHmac [hmacSize]byte
        
        // Our starting packet needs to be filled out with random bytes, we
        // generate some deterministically using the session private key.
        paddingKey := generateKey("pad", sessionKey.Serialize())
        paddingBytes := generateCipherStream(paddingKey, routingInfoSize)
        copy(mixHeader[:], paddingBytes)

	// Compute the routing information for each hop along with a
	// MAC of the routing information using the shared key for that hop.
	for i := numHops - 1; i >= 0; i-- {
		rhoKey := generateKey("rho", hopSharedSecrets[i])
		muKey := generateKey("mu", hopSharedSecrets[i])

		hopsData[i].HMAC = nextHmac

		// Shift and obfuscate routing information
		streamBytes := generateCipherStream(rhoKey, numStreamBytes)

		rightShift(mixHeader[:], hopDataSize)
		buf := &bytes.Buffer{}
		hopsData[i].Encode(buf)
		copy(mixHeader[:], buf.Bytes())
		xor(mixHeader[:], mixHeader[:], streamBytes[:routingInfoSize])

		// These need to be overwritten, so every node generates a correct padding
		if i == numHops-1 {
			copy(mixHeader[len(mixHeader)-len(filler):], filler)
		}

		packet := append(mixHeader[:], assocData...)
		nextHmac = calcMac(muKey, packet)
	}

	packet := &OnionPacket{
		Version:      0x00,
		EphemeralKey: sessionKey.PubKey(),
		RoutingInfo:  mixHeader,
		HeaderMAC:    nextHmac,
	}
	return packet, nil
}
```

# オニオン復号

使用する `onion_packet` には 2 種類あります。

1. 支払い用の `update_add_htlc` に含まれる `onion_routing_packet`。`payload` TLV を含みます（[HTLC の追加](02-peer-protocol.md#adding-an-htlc-update_add_htlc) を参照）。
2. メッセージ用の `onion_message` に含まれる `onion_message_packet`。`onionmsg_tlv` TLV を含みます（[オニオンメッセージ](#onion-messages) を参照）。

これらのセクションでは、使用する `associated_data`、`path_key`（あれば）、取り出したペイロードの形式と扱い（次のピアを決定する方法を含む）、エラーの扱い方を規定します。処理自体は同一です。

## 要件

読み手は：

- `version` が 0 でない場合：
  - パケットの処理を中止し、失敗しなければならない。
- `public_key` が有効な公開鍵でない場合：
  - パケットの処理を中止し、失敗しなければならない。
- オニオンが支払い用である場合：
  - `hmac` を以前に受信していた場合：
    - プリイメージが既知ならば：
      - プリイメージを用いて HTLC をただちに償還してよい。
    - そうでなければ：
      - パケットの処理を中止し、失敗しなければならない。
- `path_key` が指定されている場合：
  - `blinding_ss` を ECDH(`path_key`, `node_privkey`) として計算する。
  - 次のいずれかを行う：
    - `public_key` に $`HMAC256(\text{"blinded\_node\_id"}, blinding\_ss)`$ を乗じて調整する。
  - もしくは（同等の操作として）：
    - 後述の自身の `node_privkey` に $`HMAC256(\text{"blinded\_node\_id"}, blinding\_ss)`$ を乗じて調整する。
- 共有秘密 `ss` を ECDH(`public_key`, `node_privkey`) として導出する（[共有秘密](#shared-secret) を参照）。
- `mu` を $`HMAC256(\text{"mu"}, ss)`$ として導出する（[鍵生成](#key-generation) を参照）。
- HMAC を $`HMAC256(mu, hop\_payloads || associated\_data)`$ として導出する。
- 計算した HMAC と `hmac` は定数時間で比較しなければならない。
- 計算した HMAC と `hmac` が一致しない場合：
  - パケットの処理を中止し、失敗しなければならない。
- `rho` を $`HMAC256(\text{"rho"}, ss)`$ として導出する（[鍵生成](#key-generation) を参照）。
- `rho` を用いて、`hop_payloads` の 2 倍の長さの `bytestream` を導出する（[疑似ランダムバイトストリーム](#pseudo-random-byte-stream) を参照）。
- `unwrapped_payloads` を `hop_payloads` と `bytestream` の XOR に設定する。
- `unwrapped_payloads` の先頭から `bigsize` を取り出し `payload_length` とする。形式が不正な場合：
  - パケットの処理を中止し、失敗しなければならない。
- `payload_length` が 2 未満の場合：
  - パケットの処理を中止し、失敗しなければならない。
- `unwrapped_payloads` に残るバイト数が `payload_length` 未満の場合：
  - パケットの処理を中止し、失敗しなければならない。
- `unwrapped_payloads` の先頭から `payload_length` バイトを取り出し、現在の `payload` とする。
- `unwrapped_payloads` に残るバイト数が 32 未満の場合：
  - パケットの処理を中止し、失敗しなければならない。
- `unwrapped_payloads` の先頭から 32 バイトを取り出し `next_hmac` とする。
- `unwrapped_payloads` が `hop_payloads` より短い場合：
  - パケットの処理を中止し、失敗しなければならない。
- `next_hmac` が全ゼロでない（最終ノードではない）場合：
  - `blinding_tweak` を $`SHA256(public\_key || ss)`$ として導出する（[一時オニオン鍵のブラインド化](#blinding-ephemeral-onion-keys) を参照）。
  - 次のピアに対し、以下の内容のオニオンを転送するべきである：
    - `version` を 0 に設定する。
    - `public_key` を、受信した `public_key` に `blinding_tweak` を乗じた値に設定する。
    - `hop_payloads` を、受信時の `hop_payloads` のサイズに切り詰めた `unwrapped_payloads` に設定する。
    - `hmac` を `next_hmac` に設定する。
  - 転送できない場合：
    - 失敗しなければならない。
- そうでない場合（`next_hmac` が全ゼロの場合）：
  - これはオニオンの最終目的地である。

## 根拠

ブラインドパスが使われる場合、送信者は実際には自分の `node_id` ではなく、調整済みのバージョンに対してこのオニオンを暗号化しています。オニオンと一緒に渡される `path_key` から使われた調整値を導き出せるので、ノードの秘密鍵を同じ要領で調整して復号するか、もしくは数学的に等価な方法としてオニオンの一時鍵のほうを調整することができます。

# フィラー生成

パケットを受信した処理ノードは、ルート情報と各ホップのペイロードから自分宛ての情報を取り出します。
取り出しは、フィールドの難読化を解いて左シフトすることで行います。
このままでは各ホップでフィールドが短くなっていき、攻撃者がルート長を推測できてしまいます。これを防ぐため、フィールドは転送前に事前にパディングされます。
パディングは HMAC の対象に含まれるため、起点ノードは各ホップで生成されるのと同じパディングを事前に作成し、各ホップの HMAC を正しく計算する必要があります。
選択されたルートが 1300 バイトより短い場合、フィラーはフィールド長を埋める用途にも使われます。

処理ノードは `hop_payloads` の難読化を解く前に、合計長が `2*1300` になるよう 1300 バイトの `0x00` を末尾に付加します。
次に同じ長さの疑似ランダムバイトストリームを生成し、`XOR` で `hop_payloads` に適用します。
これにより、自分宛て情報の難読化が解かれると同時に、末尾に追加した `0x00` バイトに難読化が施されます。

HMAC を正しく計算するために、起点ノードは各ホップの `hop_payloads` を事前に生成し、各ホップで段階的に難読化されていくパディングを含めなければなりません。この段階的に難読化されたパディングを `filler` と呼びます。

以下の Go の例は、フィラーの生成方法を示しています。

```Go
func generateFiller(key string, numHops int, hopSize int, sharedSecrets [][sharedSecretSize]byte) []byte {
	fillerSize := uint((numMaxHops + 1) * hopSize)
	filler := make([]byte, fillerSize)

	// The last hop does not obfuscate, it's not forwarding anymore.
	for i := 0; i < numHops-1; i++ {

		// Left-shift the field
		copy(filler[:], filler[hopSize:])

		// Zero-fill the last hop
		copy(filler[len(filler)-hopSize:], bytes.Repeat([]byte{0x00}, hopSize))

		// Generate pseudo-random byte stream
		streamKey := generateKey(key, sharedSecrets[i])
		streamBytes := generateCipherStream(streamKey, fillerSize)

		// Obfuscate
		xor(filler, filler, streamBytes)
	}

	// Cut filler down to the correct length (numHops+1)*hopSize
	// bytes will be prepended by the packet generation.
	return filler[(numMaxHops-numHops+2)*hopSize:]
}
```

この実装例はあくまでデモンストレーション目的であり、`filler` はもっと効率的に生成することもできます。
最終ホップはパケットをこれ以上転送しないため、`filler` を難読化する必要はなく、HMAC を取り出す必要もありません。

# エラーの返却

オニオンルーティングプロトコルには、暗号化されたエラーメッセージを起点ノードに返却する簡素な仕組みが含まれます。返されるエラーは、最終ノードを含む任意のホップから報告された失敗である可能性があります。
転送パケットの形式は返却経路には使えません。起点以外のホップは、それを生成するのに必要な情報を持っていないからです。
ホップ自体の失敗もあり得るためオンチェーンには載らず、これらのエラーメッセージは信頼できるとは限らない点に注意してください。

中継ホップは転送経路で得た共有秘密を保存しておき、各ホップで対応する返却パケットの難読化に再利用します。さらに、各ノードはルート上の自身の送信側ピアに関する情報をローカルに保持しており、最終的に返却パケットをどこへ返すべきかを把握しています。

## エラーを起こしたノード

エラーメッセージを生成するノード（_エラーを起こしたノード_）は、以下のフィールドからなる _返却パケット_ を組み立てます。

1. data:
   * [`32*byte`:`hmac`]
   * [`u16`:`failure_len`]
   * [`failure_len*byte`:`failuremsg`]
   * [`u16`:`pad_len`]
   * [`pad_len*byte`:`pad`]

ここで `hmac` はパケットの残りを認証する HMAC で、上述の手順で `um` 鍵タイプを使って生成した鍵を用います。`failuremsg` は後述で定義され、`pad` は長さを隠すための余分なバイトです。

エラーを起こしたノードは続いて、`ammag` 鍵タイプで新たな鍵を生成し、その鍵で疑似ランダムストリームを生成して、`XOR` でパケットに適用します。

この難読化ステップは返却経路上の各ホップで繰り返されます。各ホップは返却パケットを受信すると、自分の `ammag` を生成し、疑似ランダムバイトストリームを作って返却パケットに適用してから上流に転送します。

起点ノードは、対応する転送パケットの発信者であるため、返却メッセージの最終受信者が自分であることを検出できます。自分が開始した転送に対応するエラーメッセージを起点ノードが受け取った（これ以上返却転送できない）場合、ルート上の各ホップの `ammag` と `um` 鍵を生成します。次に各ホップの `ammag` 鍵で順番にエラーメッセージを復号し、`um` 鍵で HMAC を計算します。計算した HMAC と `hmac` フィールドが一致したホップが、エラーメッセージの送信者と判定できます。

転送パケットと返却パケットの対応付けは、本オニオンルーティングプロトコルの外側、例えば支払いチャネルの HTLC との関連付けで扱われます。

`path_key` を伴う HTLC のエラー処理は特に注意が必要です。実装（あるいはバージョン）の差がブラインドパスの構成要素のアンブラインドに利用されかねないためです。そこで、すべてのエラーを `invalid_onion_blinding` に変換し、導入点で通常のオニオンエラーへ変換し直すという方針を採っています。

### `attribution_data` の初期化

`attribution_data` のレイアウトについては [HTLC の削除: `update_fulfill_htlc`、`update_fail_htlc`、`update_fail_malformed_htlc`](02-peer-protocol.md#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc) を参照してください。

`htlc_hold_times` フィールドは、各ホップでの HTLC のホールド時間を 100 ミリ秒単位で表します。例えば値 3 は 300 ms を意味します。経路上のノードのうち正確な時刻情報を持たないものはゼロを報告して構いません。

エラーを起こしたノードは自身のホールド時間をこの配列の先頭に置き、残りはゼロで埋めます。フィールドサイズは、ルート上で対応する最大ホップ数（20）を基に決められています。

`truncated_hmacs` フィールドには、返却パケットの `hmac` と同じ `um` 鍵を使って計算された、ホップごとの認証コードを切り詰めた値が並びます。空間節約のため、本来 32 バイトの HMAC を先頭 4 バイトに切り詰めています。

理屈の上では、この切り詰めにより悪意あるノードが HMAC を当て推量する余地が生まれますが、誤った推量は今後の経路選択でペナルティとなるため、ゲーム理論的に攻撃者には不利です。

このフィールドのサイズは、ルートの最大ホップ数（20）と、切り詰め HMAC のサイズ（4 バイト）に基づきます。各ホップは、自身が経路上のどの位置にいる可能性があるかに対応する 20 個の HMAC を追加します。これは、各ホップの位置を知っているのが送信者だけだからです。

20 × 20 = 400 個の HMAC をすべて保持する必要はありません。位置 0 のノードは 20 通りの位置すべてに対する HMAC を保持する必要があります。位置 1 のノードでは、残りの最大ルート長は 19 なので 19 個の HMAC で済み、以下同様です。合計すると 20+19+18+...+1 = 210 個になります。

`hmacs` フィールドのレイアウトを以下に示します。実際のフォーマットはずっと長いですが、説明のため最大ルート長を 3 ホップに簡略化しています。

`hmac_0_2` | `hmac_0_1` | `hmac_0_0` | `hmac_1_1` | `hmac_1_0` | `hmac_2_0`

`hmac_x_y` はノード `x`（現在この失敗メッセージを処理しているノードから数える）が、自身がエラーを起こしたノードから `y` ホップ離れていると仮定して付与した HMAC を意味します。

各 HMAC は、以下の要素を指定の順序で連結したものから計算します。

* 疑似ランダムバイトストリームを適用する前の返却パケット。

* `htlc_hold_times` の先頭 `y+1` 個のホールド時間の連結。例えば `hmac_0_2` であれば 3 つすべてのホールド時間を対象とする。

* `x` から見た下流ノード位置に対応する `y` 個の下流 HMAC の連結。例えば `hmac_0_2` は `hmac_1_1` と `hmac_2_0` を対象とする。

エラーを起こしたノードは自身の 20 個の HMAC を配列の先頭に置き、残りはゼロで埋めます。厳密には、下流データを覆う必要がないため `hmac_0_0` だけを追加すれば足りますが、起点ノードでの検証効率のためにすべての HMAC を計算することを要求します。冗長な HMAC は、ゼロ初期化された部分を覆うことになります。

最後に、`ammagext` 鍵タイプを用いて新たな鍵を生成し、その鍵で疑似ランダムストリームを作って `attribution_data` フィールドに `XOR` で適用します。

### 要件

_エラーを起こしたノード_ は：
  - `failure_len` と `pad_len` の合計が少なくとも 256 になるように `pad` を設定しなければならない。
  - `failure_len` と `pad_len` の合計が 256 になるように `pad` を設定するべきである。これから外れると、古いノードが返却メッセージを解析できなくなることがある。
  - `option_attribution_data` を広告している場合：
    - 受信した `update_add_htlc` に `path_key` が設定されていない場合：
      - `attribution_data` を初期化し、`update_fail_htlc` に含めなければならない。

## 中継ノード

### 返却パケットの変換

ノードの `ammag` 鍵を生成し、疑似ランダムバイトストリームを生成して、その結果を返却パケットに適用して難読化します。これを `update_htlc_fail` メッセージの `reason` フィールドとして格納します。

  この難読化手順は、エラーを起こしたノードが行う難読化手順と同一です。

### `attribution_data` の変換

* 既存のホールド時間をすべて 4 バイトだけ右にシフトする。

* 既存の HMAC を全てシフトおよび剪定する。

  各上流方向の段階で、各ホップにつき 1 個の HMAC を剪定できます。20 通りの位置すべての HMAC が揃っていて、さらに上流にもう 1 ホップ存在することが分かった場合、直前のホップにより位置 21 に対応するようになった既存の HMAC は、いずれも不要になります。

  上で示した簡略化した 3 ホップのレイアウトに対するシフト/剪定操作は、次のような結果になります。

  `-` | `-` | `-` | `hmac_0'_1` | `hmac_0'_0` | `hmac_1'_0`

  かつての `hmac_x'_y` は `hmac_x+1_y` に相当します。各ホップの一番左の HMAC は破棄されます。

### `attribution_data` の更新

* `htlc_hold_times` の先頭にノード自身のホールド時間を入れる。前述のシフト操作によりそのスロットが空いている。

* 自身の 20 個の切り詰め HMAC を計算し、`hmacs` の先頭、新たに空いたスロットに入れる。

* ノードの `ammagext` 鍵を生成し、疑似ランダムバイトストリームを作って `attribution_data` フィールドに適用し難読化する。この難読化手順は、エラーを起こしたノードのものと同一である。

### 要件

- `option_attribution_data` を広告している場合：
  - 受信した `update_add_htlc` に `path_key` が設定されていない場合：
    - 下流から `attribution_data` を受信した場合：
      - 上記のとおり `attribution_data` を変換しなければならない。
    - そうでない場合：
      - 全ゼロの `attribution_data` ブロックをインスタンス化しなければならない。
    - 上記のとおり `attribution_data` を更新しなければならない。
- すべてのノードは：
  - 上記のとおり返却パケットを変換しなければならない。
  - `update_htlc_fail` メッセージを返却転送しなければならない。

## 起点ノード

起点ノードはもちろん対応する転送パケットの発信者なので、返却メッセージの最終受信者が自分であることを検出できます。自分が開始した転送に対応する `update_htlc_fail` メッセージを受け取った（これ以上返却転送できない）場合、ルート上の各ホップに対する `ammag`、`ammagext`、`um` 鍵を生成します。

次にメッセージを各ホップの `ammag` および `ammagext` 鍵で繰り返し復号します。各ホップで以下を行います。

`option_attribution_data` をサポートする起点ノードでは：

* `attribution_data` 内の、そのホップの位置に対応する HMAC を、ホップの `um` 鍵で検証する。HMAC が無効ならメッセージの処理を中止し、今後の経路選択でこのホップにペナルティを科すべきである。これが失敗を「帰属可能 (attributable)」たらしめる仕組みである。

  HMAC は下流ノードが追加した HMAC を含むすべてのデータをカバーするため、悪意あるノードが自分を露呈せずにメッセージを改ざんすることはできません。ただし責任の所在は依然として 2 ノードのペアにしか絞り込めません。送信者と受信者のどちらが書き換えたかは分からないからです（Lightning の他の失敗ケースでも同様です）。

  経路上のすべてのノードが `option_attribution_data` をサポートしているとは限らない場合、起点ノードは未対応の最初のノードまでのアトリビューションデータしか得られません。

* このホップから報告されたホールド時間を記録する。

  起点ノードはこの情報を使って、ノードを遅延の観点でスコアリングできます。ホールド時間がゼロで報告された場合、起点ノードは想定される遅延ペナルティを複数のノードに分散すべきです。これにより、経路上のノードはタイミングデータを提供して、他ノードの高遅延を肩代わりさせられないようにする動機を持ちます。

すべてのノードについて：

* ホップの `um` 鍵で返却パケットの HMAC を計算する。

* 計算した HMAC が返却パケットの `hmac` と一致したとき、その時点のホップが失敗の送信者であると分かる。続いて `failuremsg` を解析できる。

転送パケットと返却パケットの対応付けは本オニオンルーティングプロトコルの外側、例えば支払いチャネルの HTLC を介した関連付けで扱われます。

### 要件

_起点ノード_ は：
  - 返却メッセージを復号したら：
    - メッセージのコピーを保存するべきである。
    - ループを 27 回（TLV ペイロードタイプの最大ルート長）繰り返すまで復号を続けるべきである。
    - ルート長を隠すため、定数の `ammag` および `um` 鍵を使うべきである。
  - 失敗源を返却パケットから特定できず、かつ `attribution_data` が存在する場合：
    - `attribution_data` を用いて失敗源を特定するべきである。

### 根拠

_起点ノード_ への要件は、支払い送信者を隠す効果があります。エラーを発見したあともダミーの復号サイクルを 27 回まで続けることで、送信者が同じルートを複数回試行した場合でも、エラーを起こしたノードがタイミング解析でルート内の自分の相対位置を推測することはできません。

## 失敗メッセージ

`failuremsg` にカプセル化された失敗メッセージは、通常のメッセージと同一の形式を持ちます。すなわち、2 バイトの `failure_code` のあとに、その種別に応じたデータが続きます。メッセージデータの後にはオプションの [TLV ストリーム](01-messaging.md#type-length-value-format) が続きます。

以下に現在サポートされている `failure_code` の一覧と、その後に続く使用ケースの要件を示します。

`failure_code` は他の BOLT で定義されているメッセージタイプとは別の体系である点に注意してください。これらはトランスポート層で直接送受信されるのではなく返却パケットにラップされるため、`failure_code` の数値は他のメッセージタイプの値を流用しても衝突の懸念はありません。

`failure_code` の上位バイトはフラグの集合として読めます。

* 0x8000 (BADONION): 送信ピアによって暗号化された解析不能なオニオン
* 0x4000 (PERM): 恒久的な障害（それ以外は一時的）
* 0x2000 (NODE): ノードレベルの障害（それ以外はチャネルレベル）
* 0x1000 (UPDATE): チャネル転送パラメータが違反された

以下の `failure_code` が定義されています。

1. type: NODE|2 (`temporary_node_failure`)

処理ノードの一般的な一時的障害。

1. type: PERM|NODE|2 (`permanent_node_failure`)

処理ノードの一般的な恒久的障害。

1. type: PERM|NODE|3 (`required_node_feature_missing`)

処理ノードがこのオニオンに含まれていない必要機能を要求している。

1. type: BADONION|PERM|4 (`invalid_onion_version`)
2. data:
   * [`sha256`:`sha256_of_onion`]

`version` バイトが処理ノードに理解できなかった。

1. type: BADONION|PERM|5 (`invalid_onion_hmac`)
2. data:
   * [`sha256`:`sha256_of_onion`]

オニオンが処理ノードに到達した時点で HMAC が正しくなかった。

1. type: BADONION|PERM|6 (`invalid_onion_key`)
2. data:
   * [`sha256`:`sha256_of_onion`]

処理ノードで一時鍵を解析できなかった。

1. type: UPDATE|7 (`temporary_channel_failure`)
2. data:
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

処理ノードからのチャネルがこの HTLC を扱えなかったが、後でこの HTLC や別の HTLC を扱える可能性がある。

1. type: PERM|8 (`permanent_channel_failure`)

処理ノードからのチャネルが HTLC を扱えない。

1. type: PERM|9 (`required_channel_feature_missing`)

処理ノードからのチャネルがオニオンに含まれていない機能を要求している。

1. type: PERM|10 (`unknown_next_peer`)

オニオンが指定した `short_channel_id` が、処理ノードから出ているどのチャネルとも一致しない。

1. type: UPDATE|11 (`amount_below_minimum`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

HTLC の金額が処理ノードのチャネルにおける `htlc_minimum_msat` を下回っていた。

1. type: UPDATE|12 (`fee_insufficient`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

手数料が処理ノードのチャネルで要求される額を下回っていた。

1. type: UPDATE|13 (`incorrect_cltv_expiry`)
2. data:
   * [`u32`:`cltv_expiry`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

`cltv_expiry` が処理ノードのチャネルで要求される `cltv_expiry_delta` に従っておらず、以下の要件を満たしていない。

        cltv_expiry - cltv_expiry_delta >= outgoing_cltv_value

1. type: UPDATE|14 (`expiry_too_soon`)
2. data:
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

CLTV の期限が現在のブロック高に近すぎ、処理ノードでは安全に扱えない。

1. type: PERM|15 (`incorrect_or_unknown_payment_details`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u32`:`height`]

`payment_hash` が最終ノードにとって未知である、`payment_secret` が `payment_hash` に対応していない、その `payment_hash` に対する金額が低すぎる、HTLC の CLTV 期限が現在のブロック高に近すぎて安全に処理できない、または必要なはずの `payment_metadata` が存在しない、などの理由が該当する。

`htlc_msat` パラメータは冗長ですが、後方互換性のために残されています。`htlc_msat` の値は最終ホップのオニオンペイロードに指定された値以上である必要があるため、送信者にとって実質的な情報価値はありません（直前のノードが想定より低い手数料を取ったことを示唆する場合はあります）。直前のホップが HTLC に低すぎる金額や期限を設定した場合は、`final_incorrect_cltv_expiry` および `final_incorrect_htlc_amount` で扱われます。

`height` パラメータは、HTLC を受信した時点で最終ノードが把握している最新のブロック高に設定されます。送信者はこれを使い、誤った最終 CLTV 期限で支払いを送ったケースと、中継ホップが支払いを遅延させたために受信者の請求書 CLTV デルタ要件を満たせなくなったケースを区別できます。

注：本来、PERM|16 (`incorrect_payment_amount`) と 17 (`final_expiry_too_soon`) は、HTLC パラメータの誤りと支払いハッシュ未知を区別する目的で使われていました。残念ながら、この種の応答は、HTLC を転送するために受信したノードが、同一ハッシュで金額や期限の低い支払いを潜在的な宛先に送って応答を観測することで最終目的地を推測するプロービング攻撃を可能にしてしまいます。実装では、以前の非永続的な `final_expiry_too_soon` (17) と、現在 `incorrect_or_unknown_payment_details` (PERM|15) で表される他の永続的な失敗とを取り違えないよう注意が必要です。


1. type: 18 (`final_incorrect_cltv_expiry`)
2. data:
   * [`u32`:`cltv_expiry`]

HTLC の CLTV 有効期限がオニオン内の値より小さい。

1. type: 19 (`final_incorrect_htlc_amount`)
2. data:
   * [`u64`:`incoming_htlc_amt`]

HTLC の金額がオニオン内の値より小さい。

1. type: UPDATE|20 (`channel_disabled`)
2. data:
   * [`u16`:`disabled_flags`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

処理ノードからのチャネルが無効化されている。`disabled_flags` のフラグは現在定義されていないため、常にゼロバイト 2 個となる。

1. type: 21 (`expiry_too_far`)

HTLC の CLTV 有効期限が未来に大きく寄りすぎている。

1. type: PERM|22 (`invalid_onion_payload`)
2. data:
   * [`bigsize`:`type`]
   * [`u16`:`offset`]

復号後のオニオンのホップごとのペイロードが処理ノードに理解できない、または不完全である。エラーをペイロード内の特定の TLV 型に絞り込める場合、エラーを起こしたノードはその `type` と復号バイトストリーム内のバイト `offset` を含めてもよい。

1. type: 23 (`mpp_timeout`)

マルチパート支払いの全額が合理的な時間内に到着しなかった。

1. type: BADONION|PERM|24 (`invalid_onion_blinding`)
2. data:
   * [`sha256`:`sha256_of_onion`]

ブラインドパス内でエラーが発生した。

### 要件

_エラーを起こしたノード_ は：
  - 受信した `update_add_htlc` に `path_key` が設定されている場合：
    - `invalid_onion_blinding` エラーを返さなければならない。
  - オニオンペイロードに `current_path_key` が設定されており、自分が最終ノードでない場合：
    - `invalid_onion_blinding` エラーを返さなければならない。
  - それ以外の場合：
    - エラーメッセージを作成する際、上記のエラーコードのいずれかを選択しなければならない。
    - そのエラータイプに応じた適切なデータを含めなければならない。
    - 複数のエラーがある場合：
      - 上記リスト中、最初に遭遇したエラーを選ぶべきである。

_エラーを起こしたノード_ は以下を行ってよい：
  - オニオン内のホップごとのペイロードが無効（例：有効な TLV ストリームでない）か、必要な情報を欠いている（例：金額が指定されていない）場合：
    - `invalid_onion_payload` エラーを返してよい。
  - ノード全体に関する未指定の一時的エラーが発生した場合：
    - `temporary_node_failure` エラーを返してよい。
  - ノード全体に関する未指定の恒久的エラーが発生した場合：
    - `permanent_node_failure` エラーを返してよい。
  - ノードが `node_announcement` の `features` で広告している要件のうち、オニオンに含まれていないものがある場合：
    - `required_node_feature_missing` エラーを返してよい。

_転送ノード_ は以下を行わなければならない：
  - `update_add_htlc` 受信時に `path_key` が設定されている場合：
    - `invalid_onion_blinding` エラーを返す。
  - オニオンペイロードに `current_path_key` が設定されていて、自分が最終ノードでない場合：
    - `invalid_onion_blinding` エラーを返す。
  - それ以外の場合：
    - エラーメッセージを作成する際、上記のいずれかのエラーコードを選択する。

_転送ノード_ は以下を行ってよいが、_最終ノード_ は行ってはならない：
  - オニオンの `version` バイトが未知の場合：
    - `invalid_onion_version` エラーを返す。
  - オニオンの HMAC が誤っている場合：
    - `invalid_onion_hmac` エラーを返す。
  - オニオン内の一時鍵が解析不能な場合：
    - `invalid_onion_key` エラーを返す。
  - 受信ピアへの転送中に送信チャネルで未指定の一時的エラーが発生した場合（例：チャネル容量に達した、進行中の HTLC が多すぎる）：
    - `temporary_channel_failure` エラーを返す。
  - 受信ピアへの転送中に未指定の恒久的エラーが発生した場合（例：直近にチャネルが閉じられた）：
    - `permanent_channel_failure` エラーを返す。
  - 送信チャネルの `channel_announcement` の `features` で広告される要件がオニオンに含まれていない場合：
    - `required_channel_feature_missing` エラーを返す。
  - オニオンが指す受信ピアが未知である場合：
    - `unknown_next_peer` エラーを返す。
  - HTLC の金額が現行の最小値を下回る場合：
    - 送信 HTLC の金額と送信チャネルの現在の設定を報告する。
    - `amount_below_minimum` エラーを返す。
  - HTLC の手数料が不足している場合：
    - 受信 HTLC の金額と送信チャネルの現在の設定を報告する。
    - `fee_insufficient` エラーを返す。
  - 受信 `cltv_expiry` から `outgoing_cltv_value` を引いた値が送信チャネルの `cltv_expiry_delta` を下回る場合：
    - 送信 HTLC の `cltv_expiry` と送信チャネルの現在の設定を報告する。
    - `incorrect_cltv_expiry` エラーを返す。
  - `cltv_expiry` が現在に対して不当に近い場合：
    - 送信チャネルの現在の設定を報告する。
    - `expiry_too_soon` エラーを返す。
  - `cltv_expiry` が現時点から `max_htlc_cltv` より先の未来である場合：
    - `expiry_too_far` エラーを返す。
  - チャネルが無効化されている場合：
    - 送信チャネルの現在の設定を報告する。
    - `channel_disabled` エラーを返す。

_中継ホップ_ は行ってはならないが、_最終ノード_ は行う：

- 支払いハッシュがすでに支払われている場合：
  - 支払いハッシュを未知として扱ってよい。
  - HTLC を受け入れて成功させてよい。
- `payment_secret` がその `payment_hash` に対する期待値と異なる場合、または `payment_secret` が必要なのに存在しない場合：
  - HTLC を失敗させなければならない。
  - `incorrect_or_unknown_payment_details` エラーを返さなければならない。
- 支払われた金額が期待値より少ない場合：
  - HTLC を失敗させなければならない。
  - `incorrect_or_unknown_payment_details` エラーを返さなければならない。
- 支払いハッシュが未知の場合：
  - HTLC を失敗させなければならない。
  - `incorrect_or_unknown_payment_details` エラーを返さなければならない。
- 支払われた金額が期待値の 2 倍を超える場合：
  - HTLC を失敗させるべきである。
  - `incorrect_or_unknown_payment_details` エラーを返すべきである。
    - 注：これにより、起点ノードが情報漏えいを抑えるために金額を変動させつつ、誤って大幅に過剰支払いをすることを防げます。
- `cltv_expiry` 値が現在に対して不当に近い場合：
  - HTLC を失敗させなければならない。
  - `incorrect_or_unknown_payment_details` エラーを返さなければならない。
- 最終ノードの HTLC の `cltv_expiry` が `outgoing_cltv_value` を下回る場合：
  - `final_incorrect_cltv_expiry` エラーを返さなければならない。
- 最終ノードの HTLC の `amount_msat` が `amt_to_forward` を下回る場合：
  - `final_incorrect_htlc_amount` エラーを返さなければならない。
- `channel_update` を返す場合：
  - `short_channel_id` を、受信したオニオンで使われた `short_channel_id` に設定しなければならない。

### 根拠

`short_channel_id` のエイリアスが複数ある場合、`channel_update` の `short_channel_id` は、元の送信者が想定しているものを指すべきです。これは混乱を避け、他のエイリアスやチャネル UTXO の実位置に関する情報漏えいを抑えるためです。

`channel_update` フィールドは、`failure_code` に `UPDATE` フラグが含まれるメッセージで以前は必須でした。しかし、ノードがオニオン内の更新をゴシップデータに反映することは大きなフィンガープリンティング脆弱性につながるため、`channel_update` は必須ではなくなり、ノードはこれを含めない方向に移行することが期待されています。`channel_update` を含めないノードは、`channel_update` の `len` フィールドをゼロに設定することが期待されます。

それでも一部のノードは、同じ支払いの再試行のために `channel_update` を依然利用するかもしれません。

## 失敗コードの受信

### 要件

_起点ノード_ は：
  - `failuremsg` の余分なバイトを無視しなければならない。
  - _最終ノード_ がエラーを返している場合：
    - PERM ビットが立っている場合：
      - 支払いを失敗させるべきである。
    - そうでない場合：
      - エラーコードを理解でき有効である場合：
        - 支払いを再試行してよい。特に `final_expiry_too_soon` は送信後にブロック高が変わったときに起こり得、その場合 `temporary_node_failure` も数秒で解決することがある。
  - そうではなく中継ホップがエラーを返している場合：
    - NODE ビットが立っている場合：
      - エラーを起こしたノードに接続するすべてのチャネルを考慮対象から外すべきである。
    - PERM ビットが立っていない場合：
      - ピアから新しい `channel_update` を受け取ったらチャネルを復帰させるべきである。
    - それ以外の場合：
      - UPDATE が立っており、`channel_update` が有効で、支払い送信に使った `channel_update` より新しい場合：
        - 失敗した支払いの再試行ルートを計算する際、その `channel_update` を考慮してよい。
      - その他の文脈で `channel_update` を第三者に開示してはならない。これにはローカルネットワークグラフへの適用や、ピアへのゴシップ送信などが含まれる。
    - その後、ルーティングと支払い送信を再試行するべきである。
  - 各エラータイプに指定されたデータをデバッグ目的で利用してよい。

# 成功した支払いのホールド時間

`update_fulfill_htlc` メッセージにはオプションの `attribution_data` フィールドが含まれます。これは前述の失敗時のものと類似していますが、HMAC が対象とする失敗メッセージが存在しない点だけが異なります。アトリビューションデータを使うことで、送信者は支払い経路上の各ホップが報告し、コミットしたホールド時間を取得できます。

# オニオンメッセージ

オニオンメッセージは、ピアが既存の接続を介してインボイスを問い合わせることを可能にします（[BOLT 12](12-offer-encoding.md) を参照）。ゴシップメッセージと同様、特定のローカルチャネルに紐づきません。HTLC と同じように、エンドツーエンド暗号化のために [オニオンメッセージ](#onion-messages) プロトコルを使います。

オニオンメッセージは HTLC の `onion_packet` と同じ枠組みを使いますが、わずかに柔軟な形式です。1300 バイトのペイロードに固定するのではなく、ペイロード長は全体長（ヘッダーと末尾の 66 バイトを除いた長さ）から定まります。`onionmsg_payloads` 自体は `hop_payloads` の形式と同じですが、「レガシー」長は存在しません。`length` 0 は空の `onionmsg_payload` を意味します。

オニオンメッセージは信頼性が低いです。特に、処理が安価で転送にストレージを要しないように設計されているため、中継ノードからエラーは返されません。

一貫性のため、すべてのオニオンメッセージは [ルートブラインディング](#route-blinding) を使用します。

## `onion_message` メッセージ

1. type: 513 (`onion_message`) (`option_onion_messages`)
2. data:
    * [`point`:`path_key`]
    * [`u16`:`len`]
    * [`len*byte`:`onion_message_packet`]

1. type: `onion_message_packet`
2. data:
   * [`byte`:`version`]
   * [`point`:`public_key`]
   * [`...*byte`:`onionmsg_payloads`]
   * [`32*byte`:`hmac`]

1. type: `onionmsg_payloads`
2. data:
   * [`bigsize`:`length`]
   * [`length*u8`:`onionmsg_tlv`]
   * [`32*byte`:`hmac`]
   * ...
   * `filler`

`onionmsg_tlv` 自体も TLV です。中継ノードは `encrypted_recipient_data` を期待し、オニオンメッセージとともに渡される `path_key` を使ってそれを `encrypted_data_tlv` に復号します。

フィールド番号 64 以上は最終ホップ用ペイロードに予約されていますが、非最終ホップが明示的に拒否することはありません（もちろん偶数番号でない限り）。

1. `tlv_stream`: `onionmsg_tlv`
2. types:
    1. type: 2 (`reply_path`)
    2. data:
        * [`blinded_path`:`path`]
    1. type: 4 (`encrypted_recipient_data`)
    2. data:
        * [`...*byte`:`encrypted_recipient_data`]
    1. type: 64 (`invoice_request`)
    2. data:
        * [`tlv_invoice_request`:`invreq`]
    1. type: 66 (`invoice`)
    2. data:
        * [`tlv_invoice`:`inv`]
    1. type: 68 (`invoice_error`)
    2. data:
        * [`tlv_invoice_error`:`inverr`]

#### 要件

`encrypted_recipient_data` の作成者（通常はオニオンの受信者）は：

  - [ルートブラインディング](#route-blinding) で要求されるとおり、`encrypted_data_tlv` から `encrypted_recipient_data` を作成しなければならない。
  - いずれの `encrypted_data_tlv` にも `payment_relay` および `payment_constraints` を含めてはならない。
  - 各非最終ノード向けの `encrypted_data_tlv` には `next_node_id` または `short_channel_id` のいずれかを含めなければならない。

書き手は：

- `onion_message_packet` の `version` を 0 に設定しなければならない。
- Sphinx を用いて、上記の詳細に従って `onion_message_packet` の `onionmsg_payloads` を構築しなければならない。
- Sphinx の構築では `associated_data` を一切使用してはならない。
- `onion_message_packet` の `len` は 1366 または 32834 に設定するべきである。
- 返信を期待しているが合理的な期間内に受信できなかった場合は、別の経路で再試行するべきである。
- 非最終ノードの `onionmsg_tlv` については：
  - `encrypted_recipient_data` 以外のフィールドを設定してはならない。
- 最終ノードの `onionmsg_tlv` については：
  - 最終ノードに返信が許される場合：
    - `reply_path` の `path_key` を `first_node_id` 用の最初のパスキーに設定しなければならない。
    - `reply_path` の `first_node_id` を、返信パス最初のノードの非ブラインドなノード ID に設定しなければならない。
    - `reply_path` の各 `path` について：
      - `blinded_node_id` を、オニオンホップを暗号化するためのブラインドノード ID に設定しなければならない。
      - `encrypted_recipient_data` を、受取人が使用したときに `onionmsg_tlv` の要件を満たす、暗号化済みの有効な `encrypted_data_tlv` ストリームに設定しなければならない。
      - この `reply_path` の利用を認識できるよう、秘密を持たせるために `path_id` を使用してよい。
  - そうでない場合：
    - `reply_path` を設定してはならない。


読み手は：

- チャネルが確立されていないピアからのオニオンメッセージも受け入れるべきである。
- メッセージを破棄することでレート制限を行ってよい。
- [オニオン復号](04-onion-routing.md#onion-decryption) で説明されているとおり、空の `associated_data` と `path_key` を使って `onion_message_packet` を復号し、`onionmsg_tlv` を取り出さなければならない。
- 復号に失敗した、結果が有効な `onionmsg_tlv` でない、または未知の偶数型を含む場合：
  - メッセージを無視しなければならない。
- `encrypted_data_tlv` に `allowed_features` が含まれる場合：
  - 以下のいずれかに該当する場合はメッセージを無視しなければならない：
    - `encrypted_data_tlv.allowed_features.features` に未知の機能ビットが含まれている（奇数であっても）。
    - メッセージが `encrypted_data_tlv.allowed_features.features` に含まれない機能を使っている。
- オニオン暗号化の結果として自分が最終ノードでない場合：
  - `onionmsg_tlv` に `encrypted_recipient_data` 以外の TLV フィールドが含まれる場合：
    - メッセージを無視しなければならない。
  - `encrypted_data_tlv` に `path_id` が含まれる場合：
    - メッセージを無視しなければならない。
  - それ以外の場合：
    - `next_node_id` が存在する場合：
      - その node id を持つピアを *次のピア* とする。
    - そうでなく、`short_channel_id` が存在して、それが広告済みの short_channel_id またはチャネルのローカルエイリアスに対応する場合：
      - そのチャネルの反対側にいるピアを *次のピア* とする。
    - それ以外の場合：
      - メッセージを無視しなければならない。
    - *次のピア* に対し、メッセージを `onion_message` で転送するべきである。
    - 転送する場合：
      - 転送する `onion_message` の `path_key` を、[ルートブラインディング](#route-blinding) で計算した次の `path_key` に設定しなければならない。
- そうでない場合（自分が最終ノードである場合）：
  - `path_id` が設定されており、読み手が過去に `reply_path` で公開したパスに対応している場合：
    - そのオニオンメッセージが、当該の以前のオニオンへの返信でないならば：
      - オニオンメッセージを無視しなければならない。
  - それ以外の場合（未知または未設定の `path_id`）：
    - そのオニオンメッセージが、`path_id` を含むオニオンメッセージへの返信であった場合：
      - 自分が最初のオニオンメッセージを送らなかった場合とまったく同じように応答（または無応答）しなければならない。
  - `onionmsg_tlv` に複数のペイロードフィールドが含まれている場合：
    - メッセージを無視しなければならない。
  - 返信を送りたい場合：
    - `reply_path` を使ってオニオンメッセージを作成しなければならない。
    - 返信は `first_node_id` で示されるノードに `onion_message` で送信し、`reply_path` の `path_key` を使って `reply_path` の `path` に沿って届けなければならない。


#### 根拠

返信は、指定された正確な reply_path を介してのみ受け入れるよう注意が必要です。そうしないとプロービングが可能になってしまいます。双方向の整合、つまり「返信でないものは reply_path を使わない」「返信は常に reply_path を使う」を確かめる必要があります。

厳密には不要な `onionmsg_tlv` フィールドを含むメッセージを破棄する要件は、現在および将来の実装間で挙動の一貫性を保つために設けられています。奇数フィールドであっても、それを理解するノードはパースして拒否しうる一方、理解しないノードは無視するため、問題となり得ます。

すべてのオニオンメッセージはブラインド化されていますが、このオーバーヘッド（ここでは 33 バイトと、オニオン内の各 encrypted_data_tlv に対する 16 バイトの MAC）が常に必要なわけではありません。ブラインド化のおかげで、ノードは内容を知らずとも他者から提供されたパスを利用できます。これを普遍的に行うことで、実装はわずかに簡素化され、オニオンメッセージ同士の見分けも難しくなります。

`len` により HTLC オニオン標準の 1300 バイトより大きなメッセージを送れますが、匿名性集合を狭めるので控えめに使うべきです。HTLC オニオンに見えるサイズか、より大きいときは固定サイズに揃えることを推奨します。

オニオンメッセージは明示的にチャネルを要求しませんが、スパム抑制のためにノードがそのようなピアをレート制限することは可能です。特に転送を依頼されたメッセージについて顕著です。

## `max_htlc_cltv` の選択

`max_htlc_cltv` 値は、Lightning 実装の歴史的な運用値に基づき、2016 ブロックと定義されています。

# テストベクター

## エラーの返却

テストベクターでは以下のパラメータを使用します。

	pubkey[0] = 0x02eec7245d6b7d2ccb30380bfbe2a3648cd7a942653f5aa340edcea1f283686619
    htlc_hold_time[0] = 1

	pubkey[1] = 0x0324653eac434488002cc06bbfb7f10fe18991e35f9fe4302dbea6d2353dc0ab1c
    htlc_hold_time[1] = 2

	pubkey[2] = 0x027f31ebc5462c1fdce1b737ecff52d37d75dea43ce11c74d25aa297165faa2007
    htlc_hold_time[2] = 3

	pubkey[3] = 0x032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991
    htlc_hold_time[3] = 4

	pubkey[4] = 0x02edabbd16b41c8371b92ef2f04c1185b4f03b6dcd52ba9b78d9d7c89c8f221145
    htlc_hold_time[4] = 5

	nhops = 5
	sessionkey = 0x4141414141414141414141414141414141414141414141414141414141414141

	failure_source  = node 4
	failure_message = `incorrect_or_unknown_payment_details`
      htlc_msat = 100
      height    = 800000
      tlv data
        type  = 34001
        value = [128, 128, ..., 128] (300 bytes)

エラーメッセージの作成、各ホップにおける返却パケットおよび `attribution_data` の変換、そして起点ノードでの復号と検証についての完全なバイト列トレースは、英語版の同名セクション [Returning Errors](04-onion-routing.md#returning-errors-1) に記載されています。日本語版ではバイナリのテストベクターを再掲しません。実装者は英語版のバイト列をそのまま参照してください。


# References

[sphinx]: http://www.cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf
[RFC2104]: https://tools.ietf.org/html/rfc2104
[fips198]: http://csrc.nist.gov/publications/fips/fips198-1/FIPS-198-1_final.pdf
[sec2]: http://www.secg.org/sec2-v2.pdf
[rfc8439]: https://tools.ietf.org/html/rfc8439

# 著者

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
