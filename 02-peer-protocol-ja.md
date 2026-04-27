# BOLT #2: チャネル管理のためのピアプロトコル

ピアチャネルプロトコルには、確立、通常運用、クローズの 3 つのフェーズがあります。

# 目次

  * [チャネル](#channel)
    * [`channel_id` の定義](#definition-of-channel_id)
    * [interactive-tx によるトランザクション構築](#interactive-transaction-construction)
      * [セットアップと用語](#set-up-and-vocabulary)
      * [手数料の責任](#fee-responsibility)
      * [概要](#overview)
      * [`tx_add_input` メッセージ](#the-tx_add_input-message)
      * [`tx_add_output` メッセージ](#the-tx_add_output-message)
      * [`tx_remove_input` および `tx_remove_output` メッセージ](#the-tx_remove_input-and-tx_remove_output-messages)
      * [`tx_complete` メッセージ](#the-tx_complete-message)
      * [`tx_signatures` メッセージ](#the-tx_signatures-message)
      * [`tx_init_rbf` メッセージ](#the-tx_init_rbf-message)
      * [`tx_ack_rbf` メッセージ](#the-tx_ack_rbf-message)
      * [`tx_abort` メッセージ](#the-tx_abort-message)
    * [チャネル確立 v1](#channel-establishment-v1)
      * [`open_channel` メッセージ](#the-open_channel-message)
      * [`accept_channel` メッセージ](#the-accept_channel-message)
      * [`funding_created` メッセージ](#the-funding_created-message)
      * [`funding_signed` メッセージ](#the-funding_signed-message)
      * [`channel_ready` メッセージ](#the-channel_ready-message)
    * [チャネル確立 v2](#channel-establishment-v2)
      * [`open_channel2` メッセージ](#the-open_channel2-message)
      * [`accept_channel2` メッセージ](#the-accept_channel2-message)
      * [資金構成](#funding-composition)
      * [`commitment_signed` メッセージ](#the-commitment_signed-message)
      * [資金調達署名の共有：`tx_signatures`](#sharing-funding-signatures-tx_signatures)
      * [手数料の引き上げ：`tx_init_rbf` と `tx_ack_rbf`](#fee-bumping-tx_init_rbf-and-tx_ack_rbf)
    * [チャネルのクワイエセンス](#channel-quiescence)
    * [チャネルスプライシング](#channel-splicing)
      * [`splice_init` メッセージ](#the-splice_init-message)
      * [`splice_ack` メッセージ](#the-splice_ack-message)
      * [スプライストランザクションの構築](#splice-transaction-construction)
      * [スプライスの完了](#splice-completion)
    * [チャネルクローズ](#channel-close)
      * [クローズの開始：`shutdown`](#closing-initiation-shutdown)
      * [クローズ交渉：`closing_complete` と `closing_sig`](#closing-negotiation-closing_complete-and-closing_sig)
      * [レガシークローズ交渉：`closing_signed`](#legacy-closing-negotiation-closing_signed)
    * [通常運用](#normal-operation)
      * [HTLC の転送](#forwarding-htlcs)
      * [`cltv_expiry_delta` の選択](#cltv_expiry_delta-selection)
      * [HTLC の追加：`update_add_htlc`](#adding-an-htlc-update_add_htlc)
      * [HTLC の削除：`update_fulfill_htlc`、`update_fail_htlc`、および `update_fail_malformed_htlc`](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * [チャネルメッセージのバッチ処理](#batching-channel-messages)
      * [これまでの更新のコミット：`commitment_signed`](#committing-updates-so-far-commitment_signed)
      * [更新された状態への移行の完了：`revoke_and_ack`](#completing-the-transition-to-the-updated-state-revoke_and_ack)
      * [手数料の更新：`update_fee`](#updating-fees-update_fee)
    * [メッセージの再送信：`channel_reestablish` メッセージ](#message-retransmission)
  * [著者](#authors)

# チャネル

## `channel_id` の定義

いくつかのメッセージはチャネルを識別するために `channel_id` を使用します。これは資金調達トランザクションから導出され、`funding_txid` と `funding_output_index` をビッグエンディアンで排他的論理和することで得られます (つまり `funding_output_index` が末尾 2 バイトを変更します)。

チャネル確立前は、ランダムなノンスである `temporary_channel_id` が使用されます。

異なるピアから重複する `temporary_channel_id` が存在する可能性がある点に注意してください。そのため、資金調達トランザクションが作成される前にチャネル ID でチャネルを参照する API は本質的に安全ではありません。資金調達トランザクション作成前に交換される唯一のプロトコル提供チャネル識別子は (source_node_id, destination_node_id, temporary_channel_id) のタプルです。また、資金調達トランザクションが確認される前にチャネル ID でチャネルを参照する API は永続的でもありません。資金調達出力に対応する scriptpubkey が判明するまで、重複したチャネル ID の発生を防ぐ手段はないためです。

### `channel_id`, v2

v2 プロトコルで確立されたチャネルの `channel_id` は `SHA256(lesser-revocation-basepoint || greater-revocation-basepoint)` です。ここで lesser と greater はベースポイントの順序に基づきます。

`open_channel2` を送信する時点ではピアの取り消しベースポイントは未知です。そのため、非イニシエータのベースポイントとしてゼロ埋めされた値を用いて `temporary_channel_id` を計算しなければなりません。

`accept_channel2` を送信する際は、イニシエータがリクエストと応答を対応付けられるよう、`open_channel2` の `temporary_channel_id` をそのまま使用しなければなりません。

#### 根拠

取り消しベースポイントは、正しい動作のために両方のピアが記憶しておく必要があります。最初のメッセージ交換後にこれが分かるため、それ以降のメッセージでは `temporary_channel_id` を使う必要がなくなります。両側の情報を混ぜることで `channel_id` の衝突を避け、資金調達 txid への依存も排除できます。

## interactive-tx によるトランザクション構築

interactive-tx (インタラクティブなトランザクション構築) により、2 つのピアが協力してブロードキャスト用トランザクションを構築できます。このプロトコルはデュアルファンドチャネル確立 (v2) の基盤です。

### セットアップと用語

トランザクション構築には二つの当事者がいます：*イニシエータ* と *非イニシエータ* です。
*イニシエータ* はプロトコルを開始するピアであり、例えばチャネル確立 v2 では `open_channel2` を送信するピアが *イニシエータ* となります。

プロトコルは以下の仮定をしています：

- トランザクションの `feerate` が既知である。
- トランザクションの `dust_limit` が既知である。
- トランザクションの `nLocktime` が既知である。
- トランザクションの `nVersion` が既知である。

### 手数料の責任

*イニシエータ* は以下のフィールドに対する手数料を支払う責任があります。これらは `common fields` と呼ばれます。

  - version
  - segwit marker + flag
  - input count
  - output count
  - locktime

トランザクションの残りのバイトの手数料は、`tx_add_input` または `tx_add_output` を通じてその入力または出力を提供したピアが、合意された `feerate` に基づいて負担します。

### 概要

*イニシエータ* は `tx_add_input` で interactive-tx 構築プロトコルを開始します。*非イニシエータ* は `tx_add_input`、`tx_add_output`、`tx_remove_input`、`tx_remove_output`、または `tx_complete` のいずれかで応答します。プロトコルは、両ノードが連続して `tx_complete` を送受信するまで interactive-tx プロトコルメッセージの同期的な交換を続けます。これはターン制のプロトコルです。

両ピアが連続して `tx_complete` を交換した時点で、interactive-tx 構築プロトコルは完了したとみなします。両ピアはトランザクションを構築すべきで、エラーがあれば交渉を失敗させるべきです。

このプロトコルは、並行して複数のパーティが単一のトランザクションを共同で構築できるように明示的に設計されています。これにより、単一のトランザクションで複数のチャネルを開く能力が保持されます。`serial_id` は一般的にランダムに選ばれますが、すべてのピアセッションで一貫したトランザクション順序を維持するために、受信した `serial_id` を他のピアに転送する際に再利用し、必要に応じてパリティ要件を満たすために下位ビットを反転させるのが最も簡単です。

以下はいくつかの例となるやり取りです。

#### *initiator* のみ

A は *initiator* で、2 つの入力と 1 つの出力 (資金調達出力) を持っています。B は *non-initiator* で、何も提供しません。

        +-------+                       +-------+
        |       |--(1)- tx_add_input -->|       |
        |       |<-(2)- tx_complete ----|       |
        |       |--(3)- tx_add_input -->|       |
        |   A   |<-(4)- tx_complete ----|   B   |
        |       |--(5)- tx_add_output ->|       |
        |       |<-(6)- tx_complete ----|       |
        |       |--(7)- tx_complete --->|       |
        +-------+                       +-------+

#### *initiator* と *non-initiator*

A は *initiator* で、2 つの入力と 1 つの出力を提供し、その後それを削除します。B は *non-initiator* で、1 つの入力と 1 つの出力を提供しますが、A が 2 番目の入力を追加するまで待ちます。

A が 2 番目の入力を送信しない場合、交渉は B の貢献なしに終了します。

        +-------+                         +-------+
        |       |--(1)- tx_add_input ---->|       |
        |       |<-(2)- tx_complete ------|       |
        |       |--(3)- tx_add_output --->|       |
        |       |<-(4)- tx_complete ------|       |
        |       |--(5)- tx_add_input ---->|       |
        |   A   |<-(6)- tx_add_input -----|   B   |
        |       |--(7)- tx_remove_output >|       |
        |       |<-(8)- tx_add_output ----|       |
        |       |--(9)- tx_complete ----->|       |
        |       |<-(10) tx_complete ------|       |
        +-------+                         +-------+

### `tx_add_input` メッセージ

このメッセージはトランザクションの入力を 1 つ含みます。

1. type: 66 (`tx_add_input`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`u64`:`serial_id`]
    * [`u16`:`prevtx_len`]
    * [`prevtx_len*byte`:`prevtx`]
    * [`u32`:`prevtx_vout`]
    * [`u32`:`sequence`]
    * [`tx_add_input_tlvs`:`tlvs`]

1. `tlv_stream`: `tx_add_input_tlvs`
2. types:
   1. type: 0 (`shared_input_txid`)
   2. data:
     * [`sha256`:`funding_txid`]

#### 要件

送信ノード:
  - 送信したすべての入力をトランザクションに追加しなければなりません。
  - 現在トランザクションに追加されている各入力に対し、一意な `serial_id` を使用しなければなりません。
  - `sequence` を 4294967293 (`0xFFFFFFFD`) 以下に設定しなければなりません。
  - ピアから受信した入力を再送信してはなりません。
  - *initiator* の場合:
    - 偶数の `serial_id` を送信しなければなりません。
  - *non-initiator* の場合:
    - 奇数の `serial_id` を送信しなければなりません。

受信ノード:

- 受信したすべての入力をトランザクションに追加しなければなりません。
- 以下のいずれかに該当する場合、交渉を失敗させなければなりません:
  - `sequence` が `0xFFFFFFFE` または `0xFFFFFFFF` に設定されている。
  - `prevtx_len` が `0` の場合:
    - `shared_input_txid` が設定されていない。
    - `shared_input_txid` および `prevtx_vout` が以前の資金調達出力と一致しない。
    - `shared_input_txid` が設定された入力がすでに追加されている (かつ削除されていない)。
  - `prevtx_len` が `0` でない場合:
    - `prevtx` と `prevtx_vout` が以前に追加された (削除されていない) 入力と同一である。
    - `prevtx` が有効なトランザクションでない。
    - `prevtx_vout` が `prevtx` の出力数以上である。
    - `prevtx` の `prevtx_vout` 出力の `scriptPubKey` が「1 バイトのプッシュオペコード (数値 `0` から `16`) のあとに 2 から 40 バイトのデータプッシュが続く形」になっていない。
  - `serial_id` がすでにトランザクションに含まれている。
  - `serial_id` のパリティが誤っている。
  - この交渉中に 4096 個の `tx_add_input` メッセージを受信している。

#### 根拠

各ノードはトランザクションの入力集合を把握しなければなりません。*非イニシエータ* はこのメッセージを省略してもよいです。

`serial_id` はこの入力を一意に識別するためにランダムに選ばれる番号です。構築後のトランザクション内の入力は `serial_id` でソートされなければなりません。

`prevtx` は、この入力が消費する出力を含むシリアライズ済みトランザクションです。入力が改ざんされていない (non-malleable) ことを検証するために使用します。両ピアがその入力が non-malleable であると既に分かっている場合 (例えば前回の資金調達出力である場合) は、`prevtx_len` を `0` にして `prevtx` を省略できます。

`prevtx_vout` は消費する出力のインデックスです。

`sequence` はこの入力のシーケンス番号です。置換可能性 (replaceability) を示す値でなければならず、オンチェーンでのフィンガープリンティングを避けるために実装間で同じ値を使うべきです。

#### 流動性グリーフィング

`tx_add_input` を送信する際、送信者はリモートノードがプロトコルを迅速に完了する保証がありません。悪意のあるリモートノードはメッセージを遅延させたり応答を停止したりする可能性があり、これにより正直なノードがブロードキャストできない部分的に作成されたトランザクションが発生することがあります。正直なノードがこのリモートノードのために対応する UTXO を専有的にロックしている場合、これは正直なノードの流動性をロックするために悪用される可能性があります。

したがって、実装は UTXO をロックせず、同時セッションで積極的に再利用することを推奨します。これにより、正直なノードで作成されたトランザクションが悪意のあるノードとの保留中のトランザクションを二重支出し、正直なノードに追加のコストをかけずに済むことが保証されます。

残念ながら、これにより正直なノードとの同時セッション間で競合が発生することもあります。しかし、以下の理由からこれは合理的なトレードオフです。

* オンチェーンでの資金調達の試みは比較的まれな操作です。
* 正直なノードはプロトコルを迅速に完了するため、競合のリスクが低減されます。
* 失敗した試みはコストなしで再試行できます。

### `tx_add_output` メッセージ

このメッセージはトランザクションの出力を追加します。

1. type: 67 (`tx_add_output`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`u64`:`serial_id`]
    * [`u64`:`sats`]
    * [`u16`:`scriptlen`]
    * [`scriptlen*byte`:`script`]

#### 要件

どちらのノードも：
  - このメッセージを省略してもよい（MAY omit this message）

送信ノードは：
  - 送信されたすべての出力をトランザクションに追加しなければならない（MUST add all sent outputs to the transaction）
  - もし *initiator* である場合：
    - 偶数の `serial_id` を送信しなければならない（MUST send even `serial_id`s）
  - もし *non-initiator* である場合：
    - 奇数の `serial_id` を送信しなければならない（MUST send odd `serial_id`s）

受信ノードは：
  - 受信したすべての出力をトランザクションに追加しなければならない（MUST add all received outputs to the transaction）
  - P2WSH、P2WPKH、P2TR `script` を受け入れなければならない（MUST accept P2WSH, P2WPKH, P2TR `script`s）
  - `script` が非標準である場合、交渉を失敗させてもよい（MAY fail the negotiation if `script` is non-standard）
  - 以下の場合、交渉を失敗させなければならない（MUST fail the negotiation if）：
    - `serial_id` がすでにトランザクションに含まれている場合
    - `serial_id` のパリティが間違っている場合
    - この交渉中に 4096 の `tx_add_output` メッセージを受信した場合
    - `sats` の金額が `dust_limit` より少ない場合
    - `sats` の金額が 2,100,000,000,000,000 (`MAX_MONEY`) を超える場合

#### 理論的根拠

各ノードはトランザクション出力のセットを知っていなければなりません。

`serial_id` はこの出力を一意に識別するランダムに選ばれた番号です。
構築されたトランザクション内の出力は `serial_id` によってソートされなければなりません。

`sats` は出力のサトシ値です。

`script` は出力の scriptPubKey です（その長さは省略されます）。
`script` は標準性ルールに従う必要はありません。`OP_RETURN` のような非標準スクリプトは受け入れられるかもしれませんが、対応するトランザクションはネットワーク全体での中継に失敗する可能性があります。

### `tx_remove_input` と `tx_remove_output` メッセージ

このメッセージはトランザクションから入力を削除します。

1. type: 68 (`tx_remove_input`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`u64`:`serial_id`]

このメッセージはトランザクションから出力を削除します。

1. type: 69 (`tx_remove_output`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`u64`:`serial_id`]

#### 要件

送信ノード：
  - トランザクションに追加していない、またはすでに削除した `serial_id` を持つ `tx_remove` を送信してはなりません

受信ノード：
  - 指定された入力または出力をトランザクションから削除しなければなりません
  - 以下の場合、交渉を失敗させなければなりません：
    - `serial_id` で識別される入力または出力が送信者によって追加されていない場合
    - `serial_id` が現在追加されている入力（または出力）に対応していない場合

### `tx_complete` メッセージ

このメッセージはピアのトランザクションへの貢献の完了を示します。

1. type: 70 (`tx_complete`)
2. data:
    * [`channel_id`:`channel_id`]

#### 要件

ノード：
  - このプロトコルを完了するために、このメッセージを連続して送信しなければなりません

受信ノード：
  - 交渉された入力と出力を使用してトランザクションを構築しなければなりません
  - 以下の場合、交渉を失敗させなければなりません：
    - ピアの合計入力サトシがその出力より少ない場合。この要件の遵守を確認する際には、ピアの資金出力の部分を考慮しなければなりません。
    - ピアの支払った手数料率が合意された `feerate`（`minimum fee` に基づく）を満たしていない場合
    - 非イニシエータの場合：
      - イニシエータの手数料が `common` フィールドをカバーしていない場合
    - 252 を超える入力がある場合
    - 252 を超える出力がある場合
    - トランザクションの推定重量が 400,000 (`MAX_STANDARD_TX_WEIGHT`) を超える場合

#### 理論的根拠

トランザクションの入力と出力の交換の完了を示すためです。

`tx_complete` メッセージの交換が成功した場合、両方のノードはトランザクションを構築し、プロトコルの次の部分に進むべきです。チャネル確立 v2 の場合、コミットメントトランザクションの交換です。

`minimum fee` の計算については [BOLT #3](03-transactions.md#calculating-fees-for-collaborative-transaction-construction) を参照してください。

最大入力と出力は 252 に制限されています。これにより、トランザクションの入力と出力のカウントのバイトサイズが実質的に 1 に固定されます。

### `tx_signatures` メッセージ

1. type: 71 (`tx_signatures`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`sha256`:`txid`]
    * [`u16`:`num_witnesses`]
    * [`num_witnesses*witness`:`witnesses`]
    * [`tx_signatures_tlvs`:`tlvs`]

1. subtype: `witness`
2. data:
    * [`u16`:`len`]
    * [`len*byte`:`witness_data`]

1. `tlv_stream`: `tx_signatures_tlvs`
2. types:
   1. type: 0 (`shared_input_signature`)
   2. data:
     * [`signature`:`signature`]

#### 要件

送信ノード：
  - 合計 `tx_add_input` 値によって定義される、最も低い合計サトシを寄与している場合、または両方のピアが同等の金額を寄与しているが、`node_id` が辞書順で最も低い場合：
    - 最初に `tx_signatures` を送信しなければなりません
  - `witnesses` をそれに対応する入力の `serial_id` によって順序付けしなければなりません
  - `num_witnesses` は追加した入力の数と等しくなければなりません
  - 各署名に `SIGHASH_ALL` (0x01) フラグを使用しなければなりません

受信ノード：
  - 交渉を失敗させなければなりません、もし：
    - メッセージが空の `witness` を含む場合
    - `witnesses` の数が送信ノードによって追加された入力の数と等しくない場合
    - `txid` がトランザクションの txid と一致しない場合
    - `witnesses` が非標準である場合
    - 署名が `SIGHASH_ALL` (0x01) でないフラグを使用している場合
  - トランザクションに `witnesses` を適用し、それをブロードキャストすることを推奨します
  - まだ送信していない場合、自分の `tx_signatures` を返信しなければなりません

#### 理論的根拠

どのピアが最初に `tx_signatures` を送信するかを決定するために厳密な順序付けが使用されます。これにより、各ピアが他のピアが `tx_signatures` を送信するのを待っているデッドロックを防ぎ、マルチパーティトランザクションの協力を可能にします。

`witness_data` はビットコインのワイヤープロトコルに従ってエンコードされます（CompactSize 数の要素、それぞれの要素は CompactSize 長とその後に続くバイト数）。

`minimum fee` は `tx_complete` の結論で計算および検証されますが、交換された witness データの手数料が不足している可能性があります。必要な手数料を正しく計算するのは送信ピアの責任です。

### `tx_init_rbf` メッセージ

このメッセージは、トランザクションが完了した後にその置換を開始します。


1. type: 72 (`tx_init_rbf`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u32`:`locktime`]
   * [`u32`:`feerate`]
   * [`tx_init_rbf_tlvs`:`tlvs`]

1. `tlv_stream`: `tx_init_rbf_tlvs`
2. types:
    1. type: 0 (`funding_output_contribution`)
    2. data:
        * [`s64`:`satoshis`]
   1. type: 2 (`require_confirmed_inputs`)

#### 要件

送信者:
  - `feerate` を、以前に構築されたトランザクションの `feerate` の 25/24 倍以上 (端数切り捨て) に設定しなければなりません。
  - トランザクションの資金調達出力に寄与する場合:
    - `funding_output_contribution` を設定しなければなりません。
  - 受信ノードに確認済みの入力のみの使用を要求する場合:
    - `require_confirmed_inputs` を設定しなければなりません。
  - 以前のトランザクションに寄与していた場合:
    - 各以前のトランザクション構築試行から少なくとも 1 つの入力を `tx_add_input` で送信し、新しいトランザクションが他のすべての試行を二重支出することを保証しなければなりません。

受信者:
  - `tx_abort` または `tx_ack_rbf` のいずれかで応答しなければなりません。
  - 以下の場合は `tx_abort` で応答しなければなりません:
    - `feerate` が、最後に正常に構築されたトランザクションの `feerate` の 25/24 倍以上でない場合。
  - 任意の理由で `tx_abort` を送信してよいです。
  - 以下の場合、交渉を失敗させなければなりません:
    - `require_confirmed_inputs` が設定されているにもかかわらず、確認済みの入力を提供できない場合。

#### 根拠

`feerate` はこのトランザクションが支払う手数料率です。前回使用した `feerate` より少なくとも 1/24 高くなければならず、進捗を保証するためにサトシ単位に切り捨てられます。

例えば、前回の `feerate` が 520 であれば、次に送る `feerate` は 541 でなければなりません (520 * 25 / 24 = 541.667 → 切り捨てで 541)。

RBF の試行中に以前のトランザクションが確認された場合、その RBF 試行は放棄しなければなりません。

`funding_output_contribution` は、資金調達出力が存在する場合に、このピアがその出力に寄与するサトシ量です。以前に完了したトランザクションでの寄与額と異なっていてもかまいません。省略された場合、送信者は資金調達出力に寄与しないことを意味します。

### `tx_ack_rbf` メッセージ

1. type: 73 (`tx_ack_rbf`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`tx_ack_rbf_tlvs`:`tlvs`]

1. `tlv_stream`: `tx_ack_rbf_tlvs`
2. types:
    1. type: 0 (`funding_output_contribution`)
    2. data:
        * [`s64`:`satoshis`]
   1. type: 2 (`require_confirmed_inputs`)

#### 要件

送信者:
  - トランザクションの資金調達出力に寄与する場合:
    - `funding_output_contribution` を設定しなければなりません。
  - 受信ノードに確認済みの入力のみの使用を要求する場合:
    - `require_confirmed_inputs` を設定しなければなりません。
  - 以前のトランザクションに寄与していた場合:
    - 各以前のトランザクション構築試行から少なくとも 1 つの入力を `tx_add_input` で送信し、新しいトランザクションが他のすべての試行を二重支出することを保証しなければなりません。

受信者:
  - `tx_abort` または `tx_add_input` のいずれかで応答し、interactive-tx の協調プロトコルを再開しなければなりません。
  - 以下の場合、交渉を失敗させなければなりません:
    - `require_confirmed_inputs` が設定されているにもかかわらず、確認済みの入力を提供できない場合。

#### 理論的根拠

`funding_output_contribution` は、このピアがトランザクションの資金出力に寄与するサトシの量です。この出力がある場合に限ります。以前に完了したトランザクションでの寄与とは異なる場合があります。省略された場合、送信者は資金出力に寄与していません。

ピアは、手数料率の大幅な変更により RBF 交渉を失敗させるのではなく、資金出力への寄与を停止し、トランザクションへのさらなる参加を辞退することを推奨します（寄与しないことで、コストなしで受信流動性を得ることができます）。

### `tx_abort` メッセージ

1. タイプ: 74 (`tx_abort`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`data`]

#### 要件

送信ノード：
  - すでに `tx_signatures` を送信してはなりません
  - 現在の交渉を忘れ、状態をリセットするべきです。
  - 空の `data` フィールドを送信してもかまいません。
  - 無効な署名チェックが原因で失敗した場合：
    - `tx_signatures` または `commitment_signed` メッセージに応じて、生の16進エンコードされたトランザクションを含めるべきです。

受信ノード：
  - すでにピアに `tx_signatures` を送信している場合：
    - 交渉されたトランザクションの入力が消費されるまで、チャネルを忘れてはなりません。
  - `tx_signatures` を送信していない場合：
    - 現在の交渉を忘れ、状態をリセットするべきです。
  - `tx_abort` を送信していない場合：
    - `tx_abort` をエコーバックしなければなりません
  - `data` が印刷可能な ASCII 文字のみで構成されていない場合（参考までに：印刷可能な文字セットには、バイト値 32 から 126 までが含まれます）：
    - `data` をそのまま印刷すべきではありません。

#### 理由

受信ノードは、すでに `tx_signatures` を送信している場合、トランザクションが署名されてピアによって公開されない保証がありません。トランザクションとチャネル（該当する場合）を、トランザクションが消費されなくなるまで（すなわち、任意の入力が別のトランザクションで消費された場合）記憶しておく必要があります。

`tx_abort` メッセージは、進行中の交渉をキャンセルし、初期の開始状態に戻ることを可能にします。これはチャネルを閉じる `error` メッセージとは異なります。

`tx_abort` をエコーバックすることで、ピアが中止メッセージを確認したことを確認し、発信元のピアが古いメッセージを心配することなく進行中のプロセスを終了できるようにします。

## チャネル確立 v1

認証と接続の初期化を行った後（[BOLT #8](08-transport.md) および [BOLT #1](01-messaging.md#the-init-message) 参照）、チャネルの確立が始まります。

チャネルを確立するための方法は2つあります。ここで示されるレガシーバージョンと、もう一つのバージョン（[下記](#channel-establishment-v2)）です。使用可能なチャネル確立プロトコルは `init` メッセージで交渉されます。

これは、資金提供ノード（ファンダー）が `open_channel` メッセージを送信し、それに続いて応答ノード（ファンディー）が `accept_channel` を送信することで構成されます。チャネルパラメータが確定すると、ファンダーは資金提供トランザクションとコミットメントトランザクションの両バージョンを作成できます。これについては [BOLT #3](03-transactions.md#bolt-3-bitcoin-transaction-and-script-formats) で説明されています。ファンダーはその後、`funding_created` メッセージとともに資金提供出力のアウトポイントと、ファンディーのバージョンのコミットメントトランザクションの署名を送信します。ファンディーが資金提供アウトポイントを知ると、ファンダーのバージョンのコミットメントトランザクションの署名を生成し、`funding_signed` メッセージを使用して送信できます。

チャネルファンダーが `funding_signed` メッセージを受け取ると、ビットコインネットワークに資金提供トランザクションをブロードキャストしなければなりません。`funding_signed` メッセージが送信/受信された後、両側は資金提供トランザクションがブロックチェーンに入り、指定された深さ（確認数）に達するのを待つべきです。両側が `channel_ready` メッセージを送信した後、チャネルは確立され、通常の操作を開始できます。`channel_ready` メッセージには、チャネル認証証明を構築するために使用される情報が含まれています。

        +-------+                              +-------+
        |       |--(1)---  open_channel  ----->|       |
        |       |<-(2)--  accept_channel  -----|       |
        |       |                              |       |
        |   A   |--(3)--  funding_created  --->|   B   |
        |       |<-(4)--  funding_signed  -----|       |
        |       |                              |       |
        |       |--(5)---  channel_ready  ---->|       |
        |       |<-(6)---  channel_ready  -----|       |
        +-------+                              +-------+

        - ここでノード A は「funder」であり、ノード B は「fundee」です。

これがどの段階でも失敗した場合、または一方のノードが他方のノードによって提示されたチャネル条件が適切でないと判断した場合、チャネルの確立は失敗します。

複数のチャネルが並行して動作できることに注意してください。すべてのチャネルメッセージは、`temporary_channel_id`（資金調達トランザクションが作成される前）または `channel_id`（資金調達トランザクションから派生）によって識別されます。

### `open_channel` メッセージ

このメッセージはノードに関する情報を含み、新しいチャネルを設定したいという意図を示します。これは資金調達トランザクションと、コミットメントトランザクションの両バージョンを作成するための最初のステップです。

1. タイプ: 32 (`open_channel`)
2. データ:
   * [`chain_hash`:`chain_hash`]
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`funding_satoshis`]
   * [`u64`:`push_msat`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`channel_reserve_satoshis`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u32`:`feerate_per_kw`]
   * [`u16`:`to_self_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`revocation_basepoint`]
   * [`point`:`payment_basepoint`]
   * [`point`:`delayed_payment_basepoint`]
   * [`point`:`htlc_basepoint`]
   * [`point`:`first_per_commitment_point`]
   * [`byte`:`channel_flags`]
   * [`open_channel_tlvs`:`tlvs`]

1. `tlv_stream`: `open_channel_tlvs`
2. タイプ:
    1. タイプ: 0 (`upfront_shutdown_script`)
    2. データ:
        * [`...*byte`:`shutdown_scriptpubkey`]
    1. タイプ: 1 (`channel_type`)
    2. データ:
        * [`...*byte`:`type`]

`chain_hash` の値は、開かれるチャネルがどのブロックチェーンに属するかを示します。これは通常、該当するブロックチェーンのジェネシスハッシュです。`chain_hash` の存在により、ノードは複数の異なるブロックチェーンにわたってチャネルを開くことができ、同じピアに対して複数のブロックチェーン内にチャネルを開くことも可能です（対象のチェーンをサポートしている場合）。

`temporary_channel_id` は、資金調達トランザクションが確立されるまで、このチャネルをピアごとに識別するために使用されます。その後、資金調達トランザクションから派生した `channel_id` に置き換えられます。

`funding_satoshis` は、送信者がチャネルに投入する金額です。`push_msat` は、送信者が受信者に無条件で与える初期資金の額です。`dust_limit_satoshis` は、このノードのコミットメントまたは HTLC トランザクションに対して生成されるべきでない出力の閾値です（つまり、この金額以下の HTLC と HTLC トランザクション手数料はオンチェーンで強制されません）。これは、小さな出力が標準トランザクションと見なされず、Bitcoin ネットワークを通じて伝播しないという現実を反映しています。`channel_reserve_satoshis` は、他のノードが直接支払いとして保持するべき最小額です。`htlc_minimum_msat` は、このノードが受け入れる最小値の HTLC を示します。

`max_htlc_value_in_flight_msat` は、リモートノードが提供する未決済 HTLC の総価値の上限であり、ローカルノードが HTLC に対するリスクを制限できるようにします。同様に、`max_accepted_htlcs` は、リモートノードが提供できる未決済 HTLC の数を制限します。

`feerate_per_kw` は、コミットメントおよび HTLC トランザクションに対してこの側が支払う初期手数料率をサトシ／1000-weight で示します（通常使用される 'サトシ／1000 vbytes' の 1/4 です）。これは [BOLT #3](03-transactions.md#fee-calculation) で説明されているように、後で `update_fee` メッセージで調整できます。

`to_self_delay` は、他のノードの to-self 出力が遅延されるブロック数を示し、`OP_CHECKSEQUENCEVERIFY` 遅延を使用します。これは、故障が発生した場合に自分の資金を引き出す前に待たなければならない期間です。


`funding_pubkey` は、資金調達トランザクションの出力における 2-of-2 マルチシグスクリプトの公開鍵です。

さまざまな `_basepoint` フィールドは、各コミットメントトランザクションのために [BOLT #3](03-transactions.md#key-derivation) で説明されているようにユニークな鍵を導出するために使用されます。これらの鍵を変化させることで、外部の観察者に対して各コミットメントトランザクションのトランザクション ID が予測不可能になります。これは、第三者にペナルティトランザクションをアウトソースする際にプライバシーを保護するために非常に有用です。

`first_per_commitment_point` は、最初のコミットメントトランザクションに使用される per-commitment ポイントです。

`channel_flags` の最下位ビットのみが現在定義されています：`announce_channel`。これは、資金調達フローの開始者がこのチャネルをネットワークに公開することを希望するかどうかを示します。詳細は [BOLT #7](07-routing-gossip.md#bolt-7-p2p-node-and-channel-discovery) に記載されています。

`shutdown_scriptpubkey` は、相互クローズ時に資金がどこに行くかを送信ノードがコミットすることを可能にし、リモートノードは後でノードが侵害された場合でもこれを強制するべきです。

`option_support_large_channel` は、このノードが 2^24 以上の `funding_satoshis` を受け入れることを知らせるための機能です。これは `node_announcement` メッセージでブロードキャストされるため、他のノードは `init` メッセージを交換する前に大きなチャネルを受け入れる意思のあるピアを特定するために使用できます。

#### 定義されたチャネルタイプ

チャネルタイプは明示的な列挙です：将来の定義の便宜のために偶数の機能ビットを再利用しますが、任意の組み合わせではありません（チャネルの操作に影響を与える永続的な機能を表します）。

現在定義されている基本タイプは以下の通りです：
  - `option_static_remotekey` (bit 12)
  - `option_anchors` および `option_static_remotekey` (bits 22 and 12)

各基本タイプには以下のバリエーションが許可されています：
  - `option_scid_alias` (bit 46)
  - `option_zeroconf` (bit 50)

#### 要件

送信ノード：
  - `chain_hash` 値がチャネルを開きたいチェーンを識別することを保証しなければなりません。
  - `temporary_channel_id` が同じピアとの他のチャネル ID と異なることを保証しなければなりません。
  - 両方のノードが `option_support_large_channel` を広告している場合：
    - `funding_satoshis` を 2^24 サトシ以上に設定してもよいです。
  - それ以外の場合：
    - `funding_satoshis` を 2^24 サトシ未満に設定しなければなりません。
  - `push_msat` を 1000 * `funding_satoshis` 以下に設定しなければなりません。
  - `funding_pubkey`、`revocation_basepoint`、`htlc_basepoint`、`payment_basepoint`、および `delayed_payment_basepoint` を圧縮形式の有効な secp256k1 公開鍵に設定しなければなりません。
  - `first_per_commitment_point` を、[BOLT #3](03-transactions.md#per-commitment-secret-requirements) で指定されたように導出された、初期コミットメントトランザクションに使用される per-commitment ポイントに設定しなければなりません。
  - `channel_reserve_satoshis` を `dust_limit_satoshis` 以上に設定しなければなりません。
  - `channel_flags` の未定義ビットを 0 に設定しなければなりません。
  - 両方のノードが `option_upfront_shutdown_script` 機能を広告している場合：
    - `shutdown` `scriptpubkey` によって要求される有効な `shutdown_scriptpubkey` またはゼロ長の `shutdown_scriptpubkey` (つまり `0x0000`) のいずれかを持つ `upfront_shutdown_script` を含めなければなりません。
  - それ以外の場合：
    - `upfront_shutdown_script` を含めてもよいです。
  - `open_channel_tlvs` を含める場合:
    - `upfront_shutdown_script` を含めなければなりません。
    - `channel_type` を設定しなければなりません:
      - 希望するタイプを表す定義済みのタイプに設定しなければなりません。
      - チャネルタイプを表すには、可能な限り小さいビットマップを使用しなければなりません。
      - 交渉されていない機能を含むタイプを設定すべきではありません。
      - `announce_channel` が `true` (`0` 以外) の場合:
        - `option_scid_alias` ビットが立った `channel_type` を送信してはなりません。

送信ノードは以下を行う「べき」です (SHOULD)：
  - 受信者の不正行為があった場合に、送信者がコミットメントトランザクションの出力を不可逆的に使用できるようにするために、`to_self_delay` を十分に設定します。
  - トランザクションが即座にブロックに含まれると予想されるレート以上に `feerate_per_kw` を設定します。
  - コミットメントトランザクションが Bitcoin ネットワークを通じて伝播できるように、`dust_limit_satoshis` を十分な値に設定します。
  - このピアから受け入れる最小値 HTLC に `htlc_minimum_msat` を設定します。

受信ノードは以下を行わなければなりません (MUST):
  - `channel_flags` の未定義ビットを無視します。
  - メッセージに `channel_type` が含まれていない場合:
    - チャネルを失敗させます。
  - 直前の `open_channel` を受信した後、`funding_created` メッセージを受信する前に接続が再確立された場合:
    - 新しい `open_channel` メッセージを受け入れます。
    - 直前の `open_channel` メッセージを破棄します。
  - `option_dual_fund` が交渉されている場合:
    - チャネルを失敗させます。

受信ノードは以下の場合にチャネルを失敗させてもよい (MAY):
  - `announce_channel` が `false` (`0`) なのに、チャネルを公に告知したい場合。
  - `funding_satoshis` が小さすぎる。
  - `htlc_minimum_msat` が大きすぎると考える。
  - `max_htlc_value_in_flight_msat` が小さすぎると考える。
  - `channel_reserve_satoshis` が大きすぎると考える。
  - `max_accepted_htlcs` が小さすぎると考える。
  - `dust_limit_satoshis` が大きすぎると考える。

受信ノードは以下の場合にチャネルを失敗させなければなりません (MUST)：
  - `chain_hash` 値が受信者にとって未知のチェーンのハッシュに設定されている。
  - `push_msat` が `funding_satoshis` * 1000 より大きい。
  - `to_self_delay` が不当に大きい。
  - `max_accepted_htlcs` が 483 より大きい。
  - `feerate_per_kw` がタイムリーな処理に対して小さすぎるか、不当に大きいと考える。
  - `funding_pubkey`、`revocation_basepoint`、`htlc_basepoint`、`payment_basepoint`、または `delayed_payment_basepoint` が圧縮形式の有効な secp256k1 公開鍵でない。
  - `dust_limit_satoshis` が `channel_reserve_satoshis` より大きい。
  - `dust_limit_satoshis` が `354 satoshis` より小さい (詳細は [BOLT 3](03-transactions.md#dust-limits) を参照)。
  - 初期コミットメントトランザクションの資金提供者の金額が完全な [手数料支払い](03-transactions.md#fee-payment) に十分でない。
  - 初期コミットメントトランザクションの `to_local` と `to_remote` の金額が `channel_reserve_satoshis` 以下である (詳細は [BOLT 3](03-transactions.md#commitment-transaction-outputs) を参照)。
  - `funding_satoshis` が 2^24 以上であり、受信者が `option_support_large_channel` をサポートしていない。
  - `channel_type` をサポートし、`channel_type` が設定されている場合：
    - `type` が適切でない場合。
    - `type` に `option_zeroconf` が含まれており、未確認のチャネルを開くために送信者を信頼していない場合。

受信ノードは以下を行ってはなりません：
  - `push_msat` を使用して受け取った資金を、資金取引が十分な深さに達するまで受け取ったと見なすこと。

#### 根拠

`funding_satoshis` が 2^24 サトシ未満であるという要件は、実装がまだ安定していない間の一時的な自己制限でしたが、`option_support_large_channel` を広告することで解除できます。

*チャネルリザーブ* はピアの `channel_reserve_satoshis` によって指定されます。チャネル全体の 1% が推奨されます。チャネルの各側はこのリザーブを維持し、古い取り消されたコミットメントトランザクションを放送しようとした場合に常に失うものがあるようにします。最初は、このリザーブが満たされない場合がありますが、プロトコルはこのリザーブを満たす方向に常に進展があることを保証し、一度満たされると維持されます。

送信者は非ゼロの `push_msat` を使用して受信者に初期資金を無条件に与えることができますが、この場合でも資金提供者が手数料を支払うのに十分な残りの資金を持ち、一方が使える金額を持っていることを確認します（これにより少なくとも一つの非ダスト出力があることも意味します）。他のオンチェーントランザクションと同様に、この支払いは資金取引が十分に確認されるまで確実ではなく（この間に二重支払いの危険があります）、オンチェーン確認を通じて支払いを証明する別の方法が必要な場合があります。

`feerate_per_kw` は一般的に手数料を支払う送信者のみが関心を持ちますが、HTLC トランザクションによって支払われる手数料率もあります。したがって、不合理に大きな手数料率は受信者にもペナルティを課す可能性があります。

`htlc_basepoint` を `payment_basepoint` から分離することでセキュリティが向上します。ノードはプロトコルのために HTLC 署名を生成するために `htlc_basepoint` に関連する秘密が必要ですが、`payment_basepoint` の秘密はコールドストレージに置くことができます。

`channel_reserve_satoshis` が `dust_limit_satoshis` に従ってダストと見なされないという要件は、すべての出力がダストとして排除されるケースを排除します。`accept_channel` における類似の要件は、両側の `channel_reserve_satoshis` が `dust_limit_satoshis` を上回ることを保証します。

受信者は大きな `dust_limit_satoshis` を受け入れるべきではありません。これは、ピアが多くのダスト HTLC を含むコミットメントを公開し、それが実質的にマイナー手数料に化けるグリーフィング攻撃に悪用される可能性があるためです。一方で、HTLC 出力は現在のオンチェーン手数料率に見合う第 2 段階トランザクションで使用される必要があるため、Bitcoin Core の標準ダストリミットより高い値も許容しなければなりません。

チャネル障害の処理方法の詳細は [BOLT 5:Failing a Channel](05-onchain.md#failing-a-channel) に記載されています。

### `accept_channel` メッセージ

このメッセージはノードに関する情報を含み、新しいチャネルの受け入れを示します。これは、資金調達トランザクションと両方のバージョンのコミットメントトランザクションを作成するための第二段階です。

1. タイプ: 33 (`accept_channel`)
2. データ:
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`channel_reserve_satoshis`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u32`:`minimum_depth`]
   * [`u16`:`to_self_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`revocation_basepoint`]
   * [`point`:`payment_basepoint`]
   * [`point`:`delayed_payment_basepoint`]
   * [`point`:`htlc_basepoint`]
   * [`point`:`first_per_commitment_point`]
   * [`accept_channel_tlvs`:`tlvs`]

1. `tlv_stream`: `accept_channel_tlvs`
2. タイプ:
    1. タイプ: 0 (`upfront_shutdown_script`)
    2. データ:
        * [`...*byte`:`shutdown_scriptpubkey`]
    1. タイプ: 1 (`channel_type`)
    2. データ:
        * [`...*byte`:`type`]

#### 要件

`temporary_channel_id` は `open_channel` メッセージの `temporary_channel_id` と同じでなければなりません。

送信者:
  - `channel_type` が `option_zeroconf` を含む場合:
    - `minimum_depth` をゼロに設定しなければなりません。
  - それ以外の場合:
    - 資金調達トランザクションの二重支出を避けるために合理的と考えるブロック数に `minimum_depth` を設定するべきです。
  - `channel_reserve_satoshis` を `open_channel` メッセージの `dust_limit_satoshis` 以上に設定しなければなりません。
  - `dust_limit_satoshis` を `open_channel` メッセージの `channel_reserve_satoshis` 以下に設定しなければなりません。
  - `option_channel_type` が交渉された場合:
    - `channel_type` を `open_channel` の `channel_type` に設定しなければなりません。

受信者：

- `minimum_depth` が不合理に大きい場合：
  - チャネルを失敗させても構いません (MAY)。
- `open_channel` メッセージ内の `channel_reserve_satoshis` が `dust_limit_satoshis` より少ない場合：
  - チャネルを失敗させなければなりません (MUST)。
- `open_channel` メッセージからの `channel_reserve_satoshis` が `dust_limit_satoshis` より少ない場合：
  - チャネルを失敗させなければなりません (MUST)。
- `channel_type` が設定されており、`open_channel` で `channel_type` が設定されていて、それらが等しくないタイプの場合：
  - チャネルを失敗させなければなりません (MUST)。
- `option_channel_type` が交渉されたが、メッセージに `channel_type` が含まれていない場合：
  - チャネルを失敗させても構いません (MAY)。

他のフィールドは `open_channel` の対応するフィールドと同じ要件があります。

### `funding_created` メッセージ {#funding_created}

このメッセージは、資金提供者が初期コミットメントトランザクションのために作成したアウトポイントを説明します。ピアの署名を `funding_signed` 経由で受け取った後、資金提供トランザクションをブロードキャストします。

1. タイプ: 34 (`funding_created`)
2. データ：
    * [`32*byte`:`temporary_channel_id`]
    * [`sha256`:`funding_txid`]
    * [`u16`:`funding_output_index`]
    * [`signature`:`signature`]

#### 要件

送信者は以下を設定しなければなりません (MUST)：

- `temporary_channel_id` を `open_channel` メッセージの `temporary_channel_id` と同じにします。
- `funding_txid` を非可鍛性トランザクションのトランザクション ID に設定し、
  - このトランザクションをブロードキャストしてはなりません (MUST NOT)。
- `funding_output_index` を [BOLT #3](03-transactions.md#funding-transaction-output) で定義されている資金提供トランザクション出力に対応するトランザクションの出力番号に設定します。
- `signature` を [BOLT #3](03-transactions.md#commitment-transaction) で定義されている初期コミットメントトランザクションのための `funding_pubkey` を使用した有効な署名に設定します。

送信者：

- 資金提供トランザクションを作成する際：
  - BIP141 (Segregated Witness) 入力のみを使用するべきです (SHOULD)。
  - 資金提供トランザクションが次の 2016 ブロックで確認されることを確実にするべきです (SHOULD)。

受信者：

- `signature` が不正確または LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合：
  - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません (MUST)。

#### 理由

`funding_output_index` は 2 バイトしか使用できません。これは `channel_id` にパックされ、ゴシッププロトコル全体で使用されるためです。65535 の出力の制限は過度に負担になるべきではありません。

すべての Segregated Witness 入力を持つトランザクションは改ざんされないため、資金調達トランザクションの推奨となります。

資金提供者は、資金調達トランザクションが 2016 ブロック以内に確認されるように、変更出力に CPFP を使用することができます。そうでない場合、資金受領者はそのチャネルを忘れてしまうかもしれません。

### `funding_signed` メッセージ

このメッセージは、資金提供者が最初のコミットメントトランザクションに必要な署名を提供します。これにより、必要に応じて資金が引き出せることを確認してトランザクションをブロードキャストできます。

このメッセージは、チャネルを識別するための `channel_id` を導入します。これは、`funding_txid` と `funding_output_index` を組み合わせて、ビッグエンディアンの排他的論理和 (すなわち、`funding_output_index` が最後の 2 バイトを変更) を使用して資金調達トランザクションから導出されます。

1. タイプ: 35 (`funding_signed`)
2. データ:
    * [`channel_id`:`channel_id`]
    * [`signature`:`signature`]

#### 要件

両方のピア:
  - 交渉された `channel_type` をすべてのコミットメントトランザクションで使用しなければなりません。

送信者は以下を設定しなければなりません:
  - `channel_id`: `funding_created` メッセージの `funding_txid` と `funding_output_index` の排他的論理和。
  - `signature`: `funding_pubkey` を用いた、初期コミットメントトランザクションに対する有効な署名 ([BOLT #3](03-transactions.md#commitment-transaction) で定義)。

受信者:
  - `signature` が無効、または LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 有効な `funding_signed` を受け取る前に、資金調達トランザクションをブロードキャストしてはなりません。
  - 有効な `funding_signed` を受け取った場合:
    - 資金調達トランザクションをブロードキャストすべきです。

#### 根拠

ここで `open_channel` と `accept_channel` で伝えられた `channel_type` を用いてコミットメントトランザクションを生成します。この `channel_type` がチャネル全期間にわたるコミットメント形式を決定します。

### `channel_ready` メッセージ

このメッセージ（以前は `funding_locked` と呼ばれていました）は、資金調達トランザクションがチャネル使用に十分な確認を得たことを示します。両方のノードがこれを送信すると、チャネルは通常の動作モードに入ります。

オープナーはいつでもこのメッセージを送信することができます（自分自身を信頼していると仮定されるため）が、アセプターは通常、資金が `accept_channel` で要求された `minimum_depth` に達するまで待ちます。

1. タイプ: 36 (`channel_ready`)
2. データ:
    * [`channel_id`:`channel_id`]
    * [`point`:`second_per_commitment_point`]
    * [`channel_ready_tlvs`:`tlvs`]

1. `tlv_stream`: `channel_ready_tlvs`
2. タイプ:
    1. タイプ: 1 (`short_channel_id`)
    2. データ:
        * [`short_channel_id`:`alias`]

#### 要件

送信者:
  - `funding_created` メッセージで与えられた `funding_txid` と `funding_output_index` によって指定されたアウトポイントが、[BOLT #3](03-transactions.md#funding-transaction-output) で指定された scriptpubkey に正確に `funding_satoshis` を支払う場合を除き、`channel_ready` を送信してはなりません。
  - チャネルを開いているノードでない場合:
    - このメッセージを送信する前に、資金調達トランザクションが `minimum_depth` に達するまで待つべきです。
  - `second_per_commitment_point` を、[BOLT #3](03-transactions.md#per-commitment-secret-requirements) で指定されたように導出されたコミットメントトランザクション #1 に使用する per-commitment point に設定しなければなりません。
  - `option_scid_alias` が交渉された場合:
    - `short_channel_id` `alias` を設定しなければなりません。
  - それ以外の場合:
    - `short_channel_id` `alias` を設定してもかまいません。
  - `alias` を設定する場合:
    - `open_channel` で `announce_channel` ビットが設定されていた場合:
      - 初めに `alias` を実際の `short_channel_id` に関連しない値に設定すべきです。
    - それ以外の場合:
      - `alias` を実際の `short_channel_id` に関連しない値に設定しなければなりません。
    - 複数のピアに対して同じ `alias` を送信したり、同じノード上のチャネルの `short_channel_id` と衝突するエイリアスを使用してはなりません。
    - このチャネルへの着信 HTLC に対して `alias` を常に `short_channel_id` として認識しなければなりません。
    - `channel_type` に `option_scid_alias` が設定されている場合:
      - 実際の `short_channel_id` を使用してこのチャネルへの着信 HTLC を許可してはなりません。
    - 異なる `alias` 値で同じピアに複数の `channel_ready` メッセージを送信してもかまいません。
  - それ以外の場合:
    - このメッセージを送信する前に、資金調達トランザクションが `minimum_depth` に達するまで待たなければなりません。

送信者：

資金提供を行わないノード (fundee)：
  - タイムアウトとして 2016 ブロック経過後に正しい資金提供トランザクションが見られない場合、チャネルを忘れるべきです。

受信者：
  - BOLT 11 の `r` フィールドで受け取った `alias` のいずれかを使用してもかまいません。
  - `channel_type` に `option_scid_alias` が設定されている場合：
    - BOLT 11 の `r` フィールドで実際の `short_channel_id` を使用してはなりません。

`channel_ready` を待っている時点から、どちらのノードも、合理的なタイムアウト後に他のノードから必要な応答を受け取らない場合、`error` を送信してチャネルを失敗させてもかまいません。

#### 理論的根拠

資金提供を行わないノードは、資金が危険にさらされることがないため、チャネルが存在したことを単に忘れることができます。fundee がチャネルを永遠に記憶していると、サービス拒否のリスクが生じるため、忘れることが推奨されます（たとえ `push_msat` の約束が重要であっても）。

fundee がチャネルが確認される前に忘れてしまった場合、funder は資金を取り戻すためにコミットメントトランザクションをブロードキャストし、新しいチャネルを開く必要があります。これを避けるために、funder は資金提供トランザクションが次の 2016 ブロックで確認されることを確実にするべきです。

ここでの `alias` は、2 つの異なる使用ケースのために必要です。1 つ目は、まだ確認されていないチャネルを通じて支払いをルーティングするためです（確認されるまで実際の `short_channel_id` は不明です）。2 つ目は、プライベートチャネルで使用する 1 つ以上のエイリアスを提供するためです（実際の `short_channel_id` が利用可能になった後でも）。

ノードは複数の `alias` を送信できますが、送信したすべてのエイリアスを記憶しておく必要があります。受信者は、BOLT 11 の請求書の `r` ルートヒントで使用するために 1 つだけ記憶しておけばよいです。

`channel_ready` メッセージが交換されているときに RBF 交渉が進行中の場合、交渉は中止されなければなりません。

## チャネル確立 v2

これはチャネル確立プロトコルの改訂版です。
このプロトコルは、`accept_channel2` ピア (アクセプター/非イニシエータ) が interactive-tx 構築プロトコルを通じて資金調達トランザクションに入力を提供できるように、従来のプロトコルを変更したものです。


        +-------+                              +-------+
        |       |--(1)--- open_channel2  ----->|       |
        |       |<-(2)--- accept_channel2 -----|       |
        |       |                              |       |
    --->|       |      <tx collaboration>      |       |
    |   |       |                              |       |
    |   |       |--(3)--  commitment_signed -->|       |
    |   |       |<-(4)--  commitment_signed ---|       |
    |   |   A   |                              |   B   |
    |   |       |<-(5)--  tx_signatures -------|       |
    |   |       |--(6)--  tx_signatures ------>|       |
    |   |       |                              |       |
    |   |       |--(a)--- tx_init_rbf -------->|       |
    ----|       |<-(b)--- tx_ack_rbf ----------|       |
        |       |                              |       |
        |       |    <tx rbf collaboration>    |       |
        |       |                              |       |
        |       |--(c)--  commitment_signed -->|       |
        |       |<-(d)--  commitment_signed ---|       |
        |       |                              |       |
        |       |<-(e)--  tx_signatures -------|       |
        |       |--(f)--  tx_signatures ------>|       |
        |       |                              |       |
        |       |--(7)--- channel_ready  ----->|       |
        |       |<-(8)--- channel_ready  ------|       |
        +-------+                              +-------+

        - ここでノード A は *opener*/*initiator* であり、ノード B は
          *accepter*/*non-initiator* です。

### `open_channel2` メッセージ

このメッセージは v2 チャネル確立ワークフローを開始します。

1. type: 64 (`open_channel2`)
2. data:
   * [`chain_hash`:`chain_hash`]
   * [`channel_id`:`temporary_channel_id`]
   * [`u32`:`funding_feerate_perkw`]
   * [`u32`:`commitment_feerate_perkw`]
   * [`u64`:`funding_satoshis`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u16`:`to_self_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`u32`:`locktime`]
   * [`point`:`funding_pubkey`]
   * [`point`:`revocation_basepoint`]
   * [`point`:`payment_basepoint`]
   * [`point`:`delayed_payment_basepoint`]
   * [`point`:`htlc_basepoint`]
   * [`point`:`first_per_commitment_point`]
   * [`point`:`second_per_commitment_point`]
   * [`byte`:`channel_flags`]
   * [`opening_tlvs`:`tlvs`]


1. `tlv_stream`: `opening_tlvs`
2. 種類:
   1. 種類: 0 (`upfront_shutdown_script`)
   2. データ:
       * [`...*byte`:`shutdown_scriptpubkey`]
   1. 種類: 1 (`channel_type`)
   2. データ:
        * [`...*byte`:`type`]
   1. 種類: 2 (`require_confirmed_inputs`)

根拠と要件は [`open_channel`](#the-open_channel-message) と同じですが、以下の追加があります。

#### 要件

ノードが `option_dual_fund` を交渉している場合:
  - opener ノード:
    - `open_channel` を送信してはなりません。

送信ノード:
  - `channel_type` を設定しなければなりません。
  - `funding_feerate_perkw` をこのトランザクションの手数料率に設定しなければなりません。
  - 受信ノードに確認済みの入力のみの使用を要求する場合:
    - `require_confirmed_inputs` を設定しなければなりません。

受信ノード:
  - 以下の場合、交渉を失敗させてもよい (MAY):
    - `locktime` が受け入れられない値である場合。
    - `funding_feerate_perkw` が受け入れられない値である場合。
  - 以下の場合、交渉を失敗させなければならない (MUST):
    - `require_confirmed_inputs` が設定されているが、確認済みの入力を提供できない場合。
    - `channel_type` が設定されていない場合。

#### 根拠

`temporary_channel_id` は、ピアの取り消しベースポイントのゼロ化されたベースポイントを使用して導出しなければなりません。これにより、*受け入れ者* の取り消しベースポイントが知られる前に、ピアがチャネルに割り当て可能なエラーを返すことができます。

`funding_feerate_perkw` は、開設ノードが資金調達トランザクションのために支払う手数料率を 1000 ウェイトあたりのサトシで示します。詳細は [BOLT-3, Appendix F](03-transactions.md#appendix-f-dual-funded-transaction-test-vectors) に記載されています。

`locktime` は資金調達トランザクションのロックタイムです。

受信ノードは、`locktime` または `funding_feerate_perkw` が許容範囲外と見なされる場合、交渉を失敗させてもよいです。しかし、*受け入れ者* がチャネルの資金調達に参加せずにチャネル開設を進めることを許可することが推奨されます。

`open_channel` の `channel_reserve_satoshi` は省略されています。代わりに、チャネルリザーブは総チャネル残高（`open_channel2`.`funding_satoshis` + `accept_channel2`.`funding_satoshis`）の 1% に固定され、最も近いサトシ単位に切り捨てられるか、`dust_limit_satoshis` のいずれか大きい方になります。

`push_msat` が省略されていることに注意してください。

`second_per_commitment_point` は、実装の便宜のためにここ（および `channel_ready` で）送信されます。

送信ノードは、他の参加者が確認済みの入力のみを使用するよう要求することがあります。これにより、送信ノードが他の参加者の入力の未確認の低い手数料率の祖先の手数料を支払うことがないようにします。

### `accept_channel2` メッセージ

このメッセージはノードに関する情報を含み、新しいチャネルの受け入れを示します。

1. タイプ: 65 (`accept_channel2`)
2. データ:
    * [`channel_id`:`temporary_channel_id`]
    * [`u64`:`funding_satoshis`]
    * [`u64`:`dust_limit_satoshis`]
    * [`u64`:`max_htlc_value_in_flight_msat`]
    * [`u64`:`htlc_minimum_msat`]
    * [`u32`:`minimum_depth`]
    * [`u16`:`to_self_delay`]
    * [`u16`:`max_accepted_htlcs`]
    * [`point`:`funding_pubkey`]
    * [`point`:`revocation_basepoint`]
    * [`point`:`payment_basepoint`]
    * [`point`:`delayed_payment_basepoint`]
    * [`point`:`htlc_basepoint`]
    * [`point`:`first_per_commitment_point`]
    * [`point`:`second_per_commitment_point`]
    * [`accept_tlvs`:`tlvs`]

1. `tlv_stream`: `accept_tlvs`
2. タイプ:
   1. タイプ: 0 (`upfront_shutdown_script`)
   2. データ:
       * [`...*byte`:`shutdown_scriptpubkey`]
   1. タイプ: 1 (`channel_type`)
   2. データ:
        * [`...*byte`:`type`]
   1. タイプ: 2 (`require_confirmed_inputs`)

理論と要件は、以下の追加を除き、上記の [`accept_channel`](#the-accept_channel-message) に記載されているものと同じです。

#### 要件

受諾するノード (acceptor):
  - `open_channel2` メッセージの `temporary_channel_id` を使用しなければなりません。
  - `channel_type` を `open_channel2` の `channel_type` に設定しなければなりません。
  - `funding_satoshis` をゼロで応答してもよいです。
  - 開始ノード (opener) に確認済みの入力のみの使用を要求する場合:
    - `require_confirmed_inputs` を設定しなければなりません。

受信ノード:
  - 以下のいずれかに該当する場合、交渉を失敗させなければなりません:
    - `require_confirmed_inputs` が設定されているにもかかわらず、確認済みの入力を提供できない場合。
    - `channel_type` が設定されていない場合。

#### 理論

`funding_satoshis` は、*受け入れる側* がチャネルの資金調達トランザクションに貢献するビットコインのサトシ単位の量です。

`accept_channel` の `channel_reserve_satoshi` は省略されていることに注意してください。その代わりに、チャネルリザーブはチャネル全体の残高 (`open_channel2`.`funding_satoshis` + `accept_channel2`.`funding_satoshis`) の 1% に固定され、最も近いサトシ単位に切り下げられるか、`dust_limit_satoshis` のいずれか大きい方になります。

### 資金構成

チャネル確立 v2 の資金構成は、[interactive-tx によるトランザクション構築](#interactive-transaction-construction) プロトコルを利用しますが、以下の追加の注意点があります。

#### `tx_add_input` メッセージ

##### 要件

送信ノード：
  - 受信者が `open_channel2`、`accept_channel2`、`tx_init_rbf` または `tx_ack_rbf` で `require_confirmed_inputs` を設定した場合：
    - 未確認の入力を含む `tx_add_input` を送信してはなりません

#### `tx_add_output` メッセージ

##### 要件

送信ノード：
  - *オープナー* である場合：
    - チャネルの資金出力を含む少なくとも 1 つの `tx_add_output` を送信しなければなりません

##### 理論的根拠

チャネルの資金出力は *オープナー* によって追加され、その手数料を支払います。

#### `tx_complete` メッセージ

連続する `tx_complete` を受信した場合、受信ノード：
  - *アクセプター* である場合：
    - 交渉を失敗させなければなりません：
      - 資金出力が受信されなかった場合
      - 資金出力の値が `open_channel2`.`funding_satoshis` と `accept_channel2`.`funding_satoshis` の合計に等しくない場合
      - 資金出力の値が `dust_limit` より小さい場合
  - これが RBF 試行である場合：
    - 交渉を失敗させなければなりません：
      - トランザクションの合計手数料が最後に成功した交渉トランザクションの手数料より少ない場合
      - トランザクションが各以前の資金トランザクションと少なくとも 1 つの入力を共有していない場合
  - `open_channel2`、`accept_channel2`、`tx_init_rbf` または `tx_ack_rbf` で `require_confirmed_inputs` を送信した場合：
    - 交渉を失敗させなければなりません：
      - 他のピアによって追加された入力のうちの 1 つが未確認である場合

### `commitment_signed` メッセージ

このメッセージは両方のピアによって交換されます。最初のコミットメントトランザクションの署名を含んでいます。

根拠と要件は、以下に示す [`commitment_signed`](#committing-updates-so-far-commitment_signed) と同じですが、以下の追加があります。

#### 要件

送信ノード：
  - MUST で HTLC をゼロにする。
  - この資金調達トランザクションの詳細を記憶しておく必要があります。

受信ノード：
  - メッセージに 1 つ以上の HTLC がある場合：
    - 交渉を失敗させる必要があります
  - まだ `commitment_signed` を送信していない場合：
    - `commitment_signed` を送信する必要があります
  - それ以外の場合：
    - 最初に署名する必要がある場合は、[`tx_signatures` の要件](#the-tx_signatures-message) に指定されているように `tx_signatures` を送信する必要があります

#### 根拠

最初のコミットメントトランザクションには HTLC がありません。

ピアがコミットメント署名を交換する準備ができたら、切断が発生した場合に署名交換を再開できるように、資金調達トランザクションの詳細を記憶しておく必要があります。

### 資金調達署名の共有：`tx_signatures`

ピアから有効な `commitment_signed` を受信し、`commitment_signed` を送信した後、ピアは：
  - 資金調達トランザクションの署名を含む `tx_signatures` を、[`tx_signatures` の要件](#the-tx_signatures-message) に指定された順序に従って送信する必要があります

#### 要件

送信ノード：
  - ピアから有効なコミットメント署名を受け取ったことを確認する必要があります
  - この資金調達トランザクションの詳細を記憶しておく必要があります
  - 有効な `commitment_signed` メッセージを受信していない場合：
    - `tx_signatures` メッセージを送信してはなりません

受信ノード：
  - このチャネルに対してすでに `channel_ready` メッセージを送信または受信している場合：
    - このメッセージを無視する必要があります
  - `witness` の重みが資金調達トランザクションの *opener* の手数料率を下回り、受信ノードによってトランザクションが迅速に確認されるのに不十分であると判断された場合：
    - コミットメントトランザクションをブロードキャストし、チャネルを閉じるべきです
    - 生産的な機会がある場合にはチャネル入力を二重支出し、このチャネルオープンを実質的にキャンセルするべきです
  - 資金調達トランザクションに `witnesses` を適用し、それをブロードキャストするべきです

#### 理論的根拠

ピアは、有効な `commitment_signed` メッセージを受信した後、[`tx_signatures` セクション](#the-tx_signatures-message)で指定された順序に従って `tx_signatures` を送信します。

ピアが提供する有効な証人データが、支払った手数料率を `open_channel2.funding_feerate_perkw` 以下にする場合、そのチャネルは失敗と見なされ、実行可能な機会があるときに二重支出されるべきです。これにより、ピアが手数料を過少に支払うことを抑制することが期待されます。

### 手数料の引き上げ：`tx_init_rbf` と `tx_ack_rbf`

資金調達トランザクションがブロードキャストされた後は、チャネル確認を早めるために、より多くの手数料を支払うトランザクションに置き換えられます。

#### 要件

`tx_init_rbf` の送信者:
  - *イニシエータ* または *アクセプター* のいずれでもよい。
    - 送信者がアクセプターの場合、`interactive-tx` セッションのイニシエータとなります。したがって:
      - チャネル出力の `tx_add_output` を送信しなければなりません。
      - 共通フィールドの手数料を支払わなければなりません。
  - `channel_ready` メッセージを送信または受信していてはなりません。

受信者:
  - すでに `channel_ready` を送信または受信している場合、交渉を失敗させなければなりません。
  - 任意の理由で交渉を失敗させてもかまいません。

#### 根拠

RBF 試行の途中で有効な `channel_ready` メッセージが受信された場合、その試行は放棄しなければなりません。

ピアは `tx_init_rbf.funding_output_contribution` および `tx_ack_rbf.funding_output_contribution` に、`open_channel2` や `accept_channel2`、または以前の RBF 試行で送った金額と異なる値を設定できます。資金調達出力へのコミット量を変更してよいということです。

ピアは、大きな手数料率変化により RBF 交渉を失敗させるよりも、`sats` をゼロに設定してチャネル資金調達への参加を辞退するほうが推奨されます。寄与しないことで、無償で受信流動性を得られる可能性があるためです。

両ノードのいずれも RBF を開始できるようにしているのは、最初の資金調達トランザクションの確認を待たずに、追加資金をチャネルに投入したいと考えるかもしれないためです。

## チャネルのクワイエセンス

各種の基本的な変更、特にプロトコルアップグレードは、両ピアのコミットメントトランザクションが一致しており、保留中の更新もないチャネル上で行うのが最も簡単です。本仕様では「基本的な何かが進行中である」と示すことでチャネルを静止化 (クワイエセンス) するプロトコルを定義します。

### `stfu`

1. タイプ：2 (`stfu`)
2. データ：
    * [`channel_id`:`channel_id`]
    * [`u8`:`initiator`]

### 要件

`stfu` の送信者:

- `option_quiesce` が交渉されていない限り、`stfu` を送信してはなりません。
- 送信者の HTLC 追加・削除、または手数料更新がいずれかのピアで保留中の場合、`stfu` を送信してはなりません。
- `stfu` を 2 回送信してはなりません。
- `stfu` への返信である場合:
  - `initiator` を 0 に設定しなければなりません。
- それ以外の場合:
  - `initiator` を 1 に設定しなければなりません。
- `channel_id` を、クワイエセンスさせるチャネルの ID に設定しなければなりません。
- 以後、チャネルはクワイエセンス中であると見なさなければなりません。
- `stfu` のあとに更新メッセージを送信してはなりません。

`stfu` の受信者:

- すでに `stfu` を送信していた場合:
  - 以後、チャネルはクワイエセンスであると見なさなければなりません。
- そうでない場合:
  - これ以上の更新メッセージを送るべきではありません。
  - 可能になったら `stfu` で返信しなければなりません。

両方のノード:

- HTLC が保留中の場合、クワイエセンス状態が 60 秒続いた時点で切断しなければなりません。

切断時:

- チャネルはもはやクワイエセンス状態とは見なされません。

依存プロトコル:

- クワイエセンスを終了させるすべての状態を指定しなければなりません。
  - 注: これにより、クワイエセンスに依存する複数のプロトコルをまとめて実行することはできなくなります。

### 根拠

通常の利用法は、更新の送信を止めて、現在のすべての更新が両ピアで確認済みになるのを待ってから、クワイエセンスを開始することです。プロトコルによってはイニシエータを選ぶことが重要なので、そのためにこのフラグが送られます。

両側が同時に `stfu` を送信した場合、双方とも `initiator` を `1` に設定します。その場合、「イニシエータ」は任意にチャネルのファンダー (`open_channel` の送信者) として扱います。クワイエセンスの効果は、一方が他方に返信した場合とまったく同じです。

依存プロトコルは、チャネルトラフィックを再開するために切断が必要になることを避けるため、終了条件を指定しなければなりません。明示的な再開メッセージは [検討されたものの却下されました](https://github.com/rustyrussell/lightning-rfc/pull/14)。チャネル状態の双方向の合意を維持するのが著しく複雑になるエッジケースが多いためです。その派生的な性質として、同じクワイエセンスセッション内で複数の下流プロトコルをまとめて実行することはできなくなります。

## チャネルスプライシング

スプライシングとは、資金調達トランザクションを新しいものに置き換える操作の総称です。簡略化のため、スプライシングはチャネルが [クワイエセンス](#channel-quiescence) 状態にある間に行われます。

スプライストランザクションが署名されると (どれかが確認されるのを待つ間)、チャネルは通常運用に戻ります。この時点でチャネルはクワイエセンス状態ではなくなります。

両側が `splice_locked` を送信し、いずれかのスプライストランザクションが許容できる深さに達したことが示されると、最終的にスプライスが終了します。

        +-------+                               +-------+
        |       |--- splice_init -------------->|       |
        |   A   |<--------------- splice_ack ---|   B   |
        |       |                               |       |
        |       |--- tx_add_input ------------->|       |
        |       |<------------- tx_add_input ---|       |
        |       |--- tx_add_input ------------->|       |
        |       |<------------ tx_add_output ---|       |
        |       |--- tx_add_output ------------>|       |
        |       |<-------------- tx_complete ---|       |
        |       |--- tx_add_output ------------>|       |
        |       |<-------------- tx_complete ---|       |
        |       |--- tx_complete -------------->|       |
        |       |                               |       |
        |       |--- commit_sig --------------->|       |
        |       |<--------------- commit_sig ---|       |
        |       |--- tx_signatures ------------>|       |
        |       |<------------ tx_signatures ---|       |
        |       |                               |       |
        |       |       <RESUME CHANNEL>        |       |
        |       |                               |       |
        |       |--- update_add_htlc ---------->|       |
        |       |--- commit_sig --------------->|       |
        |       |--- commit_sig --------------->|       |
        |       |<----------- revoke_and_ack ---|       |
        |       |<--------------- commit_sig ---|       |
        |       |<--------------- commit_sig ---|       |
        |       |--- revoke_and_ack ----------->|       |
        |       |                               |       |
        |       |             <RBF>             |       |
        |       |                               |       |
        |       |<-------------- tx_init_rbf ---|       |
        |       |--- tx_ack_rbf --------------->|       |
        |       |<------------- tx_add_input ---|       |
        |       |--- tx_add_input ------------->|       |
        |       |<------------- tx_add_input ---|       |
        |       |--- tx_add_output ------------>|       |
        |       |<------------ tx_add_output ---|       |
        |       |--- tx_complete -------------->|       |
        |       |<------------ tx_add_output ---|       |
        |       |--- tx_complete -------------->|       |
        |       |<-------------- tx_complete ---|       |
        |       |                               |       |
        |       |<--------------- commit_sig ---|       |
        |       |--- commit_sig --------------->|       |
        |       |--- tx_signatures ------------>|       |
        |       |<------------ tx_signatures ---|       |
        |       |                               |       |
        |       |       <RESUME CHANNEL>        |       |
        |       |                               |       |
        |       |--- update_add_htlc ---------->|       |
        |       |--- commit_sig --------------->|       |
        |       |--- commit_sig --------------->|       |
        |       |--- commit_sig --------------->|       |
        |       |<----------- revoke_and_ack ---|       |
        |       |<--------------- commit_sig ---|       |
        |       |<--------------- commit_sig ---|       |
        |       |<--------------- commit_sig ---|       |
        |       |--- revoke_and_ack ----------->|       |
        |       |                               |       |
        |       |      <SPLICE COMPLETION>      |       |
        |       |                               |       |
        |       |--- splice_locked ------------>|       |
        |       |<------------ splice_locked ---|       |
        |       |                               |       |
        |       |       <RESUME CHANNEL>        |       |
        |       |                               |       |
        |       |--- update_add_htlc ---------->|       |
        |       |--- commit_sig --------------->|       |
        |       |<----------- revoke_and_ack ---|       |
        |       |<--------------- commit_sig ---|       |
        |       |--- revoke_and_ack ----------->|       |
        |       |                               |       |
        +-------+                               +-------+

### `splice_init` メッセージ

1. type: 80 (`splice_init`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`s64`:`funding_contribution_satoshis`]
    * [`u32`:`funding_feerate_perkw`]
    * [`u32`:`locktime`]
    * [`point`:`funding_pubkey`]
    * [`splice_init_tlvs`:`tlvs`]

1. `tlv_stream`: `splice_init_tlvs`
2. types:
   1. type: 2 (`require_confirmed_inputs`)

`funding_contribution_satoshis` は、送信者が自身のチャネル残高に追加する金額 (splice-in) または減算する金額 (splice-out) を表します。

#### 要件

送信ノード:
  - チャネルがクワイエセンス状態でない場合、`splice_init` を送信してはなりません。
  - クワイエセンスのイニシエータでない場合、`splice_init` を送信してはなりません。
  - `channel_ready` の送受信が完了するまで、`splice_init` を送信してはなりません。
  - 別のスプライス交渉が進行中の場合、`splice_init` を送信してはなりません。
  - 別のスプライスが交渉済みでも `splice_locked` の送受信が完了していない場合、`splice_init` を送信してはなりません。
  - すでに `shutdown` を送信済みの場合、`splice_init` を送信してはなりません。
  - `funding_feerate_perkw` をスプライストランザクションの手数料率に設定しなければなりません。
  - チャネルから資金を引き出す (splice-out) 場合:
    - `funding_contribution_satoshis` を、現在のチャネル残高から減らす量に等しい負の値に設定しなければなりません。
  - チャネルに資金を追加する (splice-in) 場合:
    - `funding_contribution_satoshis` を、現在のチャネル残高に加える量に等しい正の値に設定しなければなりません。
  - 受信ノードに確認済みの入力のみの使用を要求する場合:
    - `require_confirmed_inputs` を設定しなければなりません。
  - 以前の資金調達トランザクションで使用したものとは異なる `funding_pubkey` を使用すべきです。

受信ノード:
  - チャネルがクワイエセンス状態でない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 送信ノードがクワイエセンスのイニシエータでない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 別のスプライスがすでに交渉中の場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 別のスプライスが交渉済みでも、まだロックされていない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - すでに `shutdown` を受信している場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `funding_feerate_perkw` が受け入れられない場合:
    - `tx_abort` で応答しなければなりません。
  - `funding_contribution_satoshis` が負の値で、その絶対値が送信ノードの現在のチャネル残高を超えている場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - スプライス試行を受け入れる場合:
    - `splice_ack` で応答しなければなりません。
  - そうでない (スプライスを拒否する) 場合:
    - `tx_abort` で応答しなければなりません。

### `splice_ack` メッセージ

1. type: 81 (`splice_ack`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`s64`:`funding_contribution_satoshis`]
    * [`point`:`funding_pubkey`]
    * [`splice_ack_tlvs`:`tlvs`]

1. `tlv_stream`: `splice_ack_tlvs`
2. types:
   1. type: 2 (`require_confirmed_inputs`)

#### 要件

送信ノード:
  - 以前の資金調達トランザクションで使用したものとは異なる `funding_pubkey` を使用すべきです。
  - スプライスに寄与したくない場合、`funding_contribution_satoshis` を `0` に設定してもよいです。
  - 受信ノードに確認済みの入力のみの使用を要求する場合:
    - `require_confirmed_inputs` を設定しなければなりません。

受信ノード:
  - `splice_init` を送信済みの場合:
    - `funding_contribution_satoshis` が負の値で、その絶対値が送信ノードの現在のチャネル残高を超える場合:
      - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
    - スプライス試行を受け入れる場合:
      - スプライストランザクションを作成するため、`interactive-tx` セッションを開始しなければなりません。
    - そうでない場合:
      - `tx_abort` を送信してスプライス試行を拒否しなければなりません。
  - そうでない (`splice_init` を送信していない) 場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。

### スプライストランザクションの構築

スプライストランザクションは [interactive-tx によるトランザクション構築](#interactive-transaction-construction) プロトコルを用いて作成しますが、以下の追加要件があります。

#### `tx_add_input` メッセージ

##### 要件

送信ノード:
  - スプライスのイニシエータの場合:
    - `tx_add_input` を、`shared_input_txid` に直前の資金調達トランザクションの `txid` を入れて送信し、現在のチャネル入力をスプライストランザクションに追加しなければなりません。
      - その共有入力に対しては `prevtx` を含めてはなりません。
      - `prevtx_vout` を、直前の資金調達出力のインデックスに設定しなければなりません。
  - 受信ノードが `splice_init`、`splice_ack`、`tx_init_rbf`、`tx_ack_rbf` のいずれかで `require_confirmed_inputs` を設定している場合:
    - 未確認入力を含む `tx_add_input` を送信してはなりません。

受信ノード:
  - `shared_input_txid` が設定されている場合:
    - 直前の資金調達トランザクションの `txid` と一致しない場合:
      - `tx_abort` で交渉を失敗させなければなりません。
    - `prevtx_vout` が直前の資金調達出力のインデックスと一致しない場合:
      - `tx_abort` で交渉を失敗させなければなりません。

##### 根拠

スプライストランザクションは、現在のチャネル資金調達出力を必ず消費します。スプライスのイニシエータがその入力をトランザクションに追加し、その重み分の手数料を支払います。直前の資金調達トランザクション全体を `prevtx` で送るのは無駄であり、65kB を超える資金調達トランザクションでは送信することすらできません。そのため `shared_input_txid` を使って `txid` のみを伝えます。

#### `tx_add_output` メッセージ

##### 要件

送信ノード:
  - スプライスのイニシエータの場合:
    - `splice_init` および `splice_ack` の `funding_pubkey` を用いた、新しいチャネルの資金調達出力を含む `tx_add_output` を、少なくとも 1 つ送信しなければなりません。
      - その `tx_add_output` の金額は、以前のチャネル容量に `splice_init` および `splice_ack` の `funding_contribution_satoshis` を反映した値に設定しなければなりません。

##### 根拠

スプライスのイニシエータが、新しいチャネル資金調達出力を追加し、その重み分の手数料を支払います。

#### `tx_complete` メッセージ

##### 要件

受信ノード:
  - 各側のチャネル残高は、それぞれの `funding_contribution_satoshis` を以前のチャネル残高に加算して計算しなければなりません。
  - 以下のいずれかに該当する場合、`tx_abort` で交渉を失敗させなければなりません:
    - 現在の資金調達トランザクションを消費する入力が、ちょうど 1 つ存在しない場合。
    - `splice_init` および `splice_ack` の資金調達公開鍵と寄与額を用いたチャネル資金調達出力が、ちょうど 1 つ存在しない場合。
    - これが RBF 試行で、トランザクションの合計手数料が、最後に正常に交渉されたスプライストランザクションの手数料より少ない場合。
    - いずれかの側がチャネル資金調達出力以外の出力を追加しており、その側の残高が、新しいチャネル容量に対応するチャネルリザーブを下回る場合。

##### 根拠

ある側がリザーブ要件を満たさないこと自体は問題ありませんが、その側がチャネルから資金を引き出す場合は、リザーブを満たさなければなりません。ピアが多額の資金をチャネルに追加してきても、こちらがスプライスに寄与する意思がない限り、リザーブを増やす必要はありません (もし途中でこの状況になった場合、`tx_remove_output` や `tx_remove_input` を使えます)。

#### `commitment_signed` メッセージ

`tx_complete` を交換した後、両ピアは `commitment_signed` を送信し、新しいチャネル資金調達出力を消費するコミットメントトランザクションを作成して、スプライストランザクションにコミットします。

通常の [`commitment_signed`](#committing-updates-so-far-commitment_signed) の要件に加えて、次のものが適用されます。

##### 要件

送信ノード:
  - スプライス資金調達出力を消費するコミットメントトランザクションを作成し、以下を満たさなければなりません:
    - `splice_init` および `splice_ack` の `funding_contribution_satoshis` を、それぞれの送信者のメイン残高に加算する。
    - 既存のコミットメントトランザクションと同じ手数料率を使用する。
    - 既存のコミットメントトランザクションと同じ `commitment_number` を使用する。
  - 保留中の HTLC 用の署名を送信しなければなりません。
  - このスプライストランザクションの詳細を記憶しておかなければなりません。

受信ノード:
  - `revoke_and_ack` で応答してはなりません。
  - まだ自身の `commitment_signed` を送信していない場合:
    - `commitment_signed` を送信しなければなりません。
  - [`tx_signatures` の要件](#the-tx_signatures-message) に従って先に署名すべき場合:
    - `tx_signatures` を送信しなければなりません。
    - イニシエータが共有入力 (直前のチャネル出力に対応) の `tx_add_input` を送信するため、最初に `tx_signatures` を送るのは誰かを判定する際には、各ノードの過去残高ではなく、以前のチャネル容量の 100% がイニシエータに帰属するものとして扱う点に注意してください。

再接続時:
  - `next_funding` がスプライストランザクションと一致する場合:
    - `commitment_signed` を再送信しなければなりません。

##### 根拠

ピアがコミットメント署名を交換できる状態に達したら、切断時に署名交換を再開できるよう、スプライストランザクションの詳細を記憶しておく必要があります。

#### `tx_signatures` メッセージ

##### 要件

送信ノード:
  - `shared_input_signature` には、この入力に対応する `funding_pubkey` を用いて、直前のチャネル資金調達出力を消費する `tx_add_input` に対する有効な ECDSA 署名を設定しなければなりません。

受信ノード:
  - `shared_input_signature` が設定されていない場合:
    - `error` を送信してチャネルを失敗させなければなりません。
  - `shared_input_signature` が無効、または LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合:
    - `error` を送信してチャネルを失敗させなければなりません。
  - チャネルはもはやクワイエセンスではないと見なさなければなりません。

再接続時:
  - `next_funding` がスプライストランザクションと一致する場合:
    - `tx_signatures` を再送信しなければなりません。

##### 根拠

チャネル資金調達出力を消費するには両ピアの署名が必要です。各ピアが自分の署名を送信することで、追加メッセージなしに共有入力に対する有効な witness を構築できます。

`tx_signatures` の交換が完了すれば、スプライストランザクションをブロードキャストできます。チャネルはクワイエセンスではなくなり、トランザクションの確認と `splice_locked` の交換を待つ間、通常運用を再開できます。

#### `tx_init_rbf` メッセージ

##### 要件

送信ノード:
  - チャネルがクワイエセンス状態でない場合、`tx_init_rbf` を送信してはなりません。
  - クワイエセンスのイニシエータでない場合、`tx_init_rbf` を送信してはなりません。
  - スプライスのイニシエータでなくても、`tx_init_rbf` を送信してよいです。
  - 保留中の RBF 試行が 10 件を超えている場合:
    - 迅速な確認を保証するために十分高い `feerate` を設定しなければなりません。
  - すでに `splice_locked` を送信済みの場合、`tx_init_rbf` を送信してはなりません。
  - `option_zeroconf` が交渉されている場合、`tx_init_rbf` を送信してはなりません。
  - `funding_output_contribution` を、`splice_init`、`splice_ack`、または以前の RBF 試行で使った `funding_contribution_satoshis` と異なる値に設定してもよいです。

受信ノード:
  - チャネルがクワイエセンス状態でない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 送信ノードがクワイエセンスのイニシエータでない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 直近に別の RBF 試行が作られていた場合:
    - `tx_abort` を送り、この RBF 試行を拒否し、以前の試行が確認されるのを待つべきです。
  - 保留中の RBF 試行が 10 件を超えており、`feerate` が迅速な確認を保証するのに十分高くない場合:
    - `tx_abort` を送り RBF 試行を拒否すべきです。
  - 送信者が以前に `splice_locked` を送信していた場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `option_zeroconf` が交渉されている場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `funding_output_contribution` が負の値で、その絶対値が送信ノードの現在のチャネル残高を超える場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。

##### 根拠

スプライストランザクションは、メモリプールの手数料変動に応じて RBF できます。両ノードが RBF を開始できるようにしているのは、最初のスプライストランザクションの確認を待たずに、追加でチャネルへスプライスインまたはスプライスアウトしたい場合があるためです。

保留中の RBF 試行数を制限しているのは、[`start_batch`](#batching-channel-messages) で定義された `batch_size` 上限に達するのを防ぐためです。多数の RBF 試行をすでに作っている場合は十分高い手数料率を要求し、また RBF 試行の間に間隔を空けて、以前の試行が確認される機会を与えます。

スプライストランザクションは常に現在のチャネル資金調達出力を消費するため、RBF 試行同士は自動的に二重支出関係になります。`option_zeroconf` が交渉されている場合は資金喪失リスクがあるため、RBF を禁止しています。

#### `tx_ack_rbf` メッセージ

##### 要件

送信ノード:
  - `funding_output_contribution` を、`splice_init`、`splice_ack`、または以前の RBF 試行で使った `funding_contribution_satoshis` と異なる値に設定してもよいです。

受信ノード:
  - `funding_output_contribution` が負の値で、その絶対値が送信ノードの現在のチャネル残高を超える場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。

### スプライスの完了

スプライストランザクションが署名済みでも、まだ許容できる深さに達していない間は、チャネル運用は通常に戻り、HTLC を交換できます。ただし、支払いはすべてのスプライストランザクションについて妥当でなければなりません。

ノードは複数のコミットメントトランザクション (現在の資金調達トランザクション用と各スプライストランザクション用) を追跡し、それぞれのコミットメントトランザクションに対する署名を交換します。

```
+------------+        +-----------+
| Funding Tx |---+--->| Commit Tx |
+------------+   |    +-----------+
                 |    +-----------+            +-----------+
                 +--->| Splice Tx |----------->| Commit Tx |
                 |    +-----------+            +-----------+
                 |    +---------------+        +-----------+
                 +--->| Splice RBF #1 |------->| Commit Tx |
                 |    +---------------+        +-----------+
                 |    +---------------+        +-----------+
                 +--->| Splice RBF #2 |------->| Commit Tx |
                      +---------------+        +-----------+
```

スプライスは `splice_locked` メッセージの交換で完了し、その時点でロックされたトランザクションが直前の資金調達トランザクションを置き換えます。

#### `splice_locked` メッセージ

1. type: 77 (`splice_locked`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`sha256`:`splice_txid`]

##### 要件

各ノード:
  - いずれかのスプライストランザクションが許容できる深さに達した場合:
    - そのトランザクションの `txid` を入れた `splice_locked` を送信しなければなりません。
  - `option_zeroconf` が交渉されている場合:
    - `tx_signatures` の交換直後に `splice_locked` を送信すべきです。

受信ノード:
  - `splice_txid` が、保留中のいずれのスプライストランザクションとも一致しない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。

`splice_locked` の送受信が完了した後:
  - `splice_txid` が一致する場合:
    - このスプライストランザクションの RBF 試行および祖先トランザクションについて、`commitment_signed` の送信を停止しなければなりません。
    - RBF 試行および祖先トランザクションを破棄してもよいです。
    - このチャネルの `announce_channel` が設定されている場合:
      - このスプライストランザクションに対応する `short_channel_id` を含む `announcement_signatures` を送信しなければなりません。
  - `splice_txid` が異なる RBF 候補を指している場合:
    - メッセージを無視すべきです。
    - `error` を送信してチャネルを失敗させてもよいです。

##### 根拠

ノード同士が異なるブロックチェーンのフォーク上にいる場合、どの RBF 試行が確認されたかについて見解が分かれることがあります。その場合、ノードはチャネルを閉じるか、`splice_locked` を無視して、いずれかのフォークがもう一方を置き換えるのを待てばよいです。最終的には両ノードが同じ RBF 試行が確認されたことに合意し、同じ `splice_txid` で `splice_locked` を交換してスプライスを完了できます。

## チャネルクローズ

ノードは接続の相互クローズを交渉できます。これは一方的クローズとは異なり、資金にすぐアクセスでき、より低い手数料で交渉できます。

クローズは 2 段階で進みます:
1. 一方がチャネルをクリアしたいことを示します (これにより新しい HTLC を受け付けなくなります)。
2. すべての HTLC が解決されたあと、最終的なチャネルクローズ交渉が始まります。

        +-------+                                                          +-------+
        |       | shutdown(scriptA1)                                       |       |
        |       |--------------------------------------------------------->|       |
        |       |                                       shutdown(scriptB1) |       |
        |       |<---------------------------------------------------------|       |
        |       |                                                          |       |
        |       |               <complete all pending HTLCs>               |       |
        |   A   |                           ....                           |   B   |
        |       |                                                          |       |
        |       | closing_complete(scriptA1, scriptB1, 1000 sat)           |       |
        |       |--------------------------------------------------------->|       |
        |       |            closing_complete(scriptB1, scriptA1, 750 sat) |       |
        |       |<---------------------------------------------------------|       |
        |       |                closing_sig(scriptA1, scriptB1, 1000 sat) |       |
        |       |<---------------------------------------------------------|       |
        |       | closing_sig(scriptB1, scriptA1, 750 sat)                 |       |
        |       |--------------------------------------------------------->|       |
        +-------+                                                          +-------+

### クローズの開始: `shutdown`

どちらのノードも（または両方）`shutdown` メッセージを送信してクローズを開始できます。これには支払いを希望する `scriptpubkey` が含まれます。

1. タイプ: 38 (`shutdown`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`scriptpubkey`]

#### 要件

送信ノード:
  - `funding_created` (ファンダーの場合) または `funding_signed` (ファンディーの場合) を送信していない場合:
    - `shutdown` を送信してはなりません。
  - `channel_ready` の前、つまり資金調達トランザクションが `minimum_depth` に達する前に `shutdown` を送信してもよいです。
  - 受信ノードのコミットメントトランザクションに保留中の更新がある場合:
    - `shutdown` を送信してはなりません。
  - 複数の `shutdown` メッセージを送信してはなりません。
  - まだロックされていないスプライストランザクションがある場合、`shutdown` を送信してはなりません。
  - `shutdown` の後に `update_add_htlc` を送信してはなりません。
  - どちらのコミットメントトランザクションにも HTLC が残っておらず (ダスト HTLC を含む)、どちらの側にも送信すべき `revoke_and_ack` がない場合:
    - その時点以降、`update` メッセージを送信してはなりません。
  - `shutdown` 送信後に追加された HTLC のルートは失敗させるべきです。
  - `open_channel` または `accept_channel` でゼロ長でない `shutdown_scriptpubkey` を送信していた場合:
    - `scriptpubkey` には同じ値を送信しなければなりません。
  - `scriptpubkey` は次のいずれかの形式に設定しなければなりません:

    1. `OP_0` `20` 20 バイト (witness pubkey hash バージョン 0 への支払い)、または
    2. `OP_0` `32` 32 バイト (witness script hash バージョン 0 への支払い)、または
    3. `option_shutdown_anysegwit` が交渉された場合に限り:
       * `OP_1` から `OP_16` のいずれか、続いて 2 から 40 バイトの単一プッシュ (witness プログラムバージョン 1 から 16)。
    4. `option_simple_close` が交渉された場合に限り:
       * `OP_RETURN` の後に以下のいずれか:
         * `6` から `75` の値、続いてその値ぶんのバイト
         * `76`、続いて `76` から `80` の値、続いてその値ぶんのバイト

受信ノード:
- `funding_signed` (ファンダーの場合) または `funding_created` (ファンディーの場合) を受信していない場合:
  - `error` を送信してチャネルを失敗させるべきです。
- `scriptpubkey` が上記の形式のいずれにも該当しない場合:
  - `warning` を送信すべきです。
- まだ `channel_ready` を送信していない場合:
  - `shutdown` メッセージに対して `shutdown` で返信してもよいです。
- ピアに未解決の更新がなくなり、かつ自身がまだ `shutdown` を送信していない場合:
  - `shutdown` メッセージに対して `shutdown` で返信しなければなりません。
- 両方のノードが `option_upfront_shutdown_script` を広告しており、受信ノードが `open_channel` または `accept_channel` でゼロ長でない `shutdown_scriptpubkey` を受信していて、その `shutdown_scriptpubkey` が `scriptpubkey` と一致しない場合:
  - `warning` を送信してもよいです。
  - 接続を失敗させなければなりません。

#### 根拠

シャットダウン開始時にチャネル状態が常に「クリーン」(保留中の変更なし) であるようにすれば、そうでない場合の挙動を考えなくて済みます。送信者は常に先に `commitment_signed` を送ります。

シャットダウンはチャネル終了の意図を示すため、新しい HTLC は追加・受け入れされません。HTLC がクリアされれば、取り消しが必要なコミットメントは残らず、すべての更新が両方のコミットメントトランザクションに含まれるので、ピアはすぐにクローズ交渉を開始できます。このため、コミットメントトランザクションへのこれ以上の更新は禁止します (特に `update_fee` は許してしまうため)。ただし、コミットメントトランザクションに HTLC が残っている間は、HTLC のタイムアウトに備えて、イニシエータが手数料率を上げることが望ましい場合があります。

`scriptpubkey` の形式には、Bitcoin ネットワークが受け入れる標準的な segwit 形式のみを含めることで、結果として得られるトランザクションがマイナーへ確実に伝播することを保証します。ただし、後方互換性のために、古いノードが送ってくる非 segwit スクリプトを受け入れることがあるかもしれません (この出力がダストリレー要件を満たさない場合には強制クローズが必要となる旨に注意が必要です)。

`option_upfront_shutdown_script` 機能は、ノードが侵害された場合に備えて `shutdown_scriptpubkey` に事前にコミットしたいという意図を表します。これは弱いコミットメント (悪意ある実装はこの種の仕様を無視しがちです) ですが、`scriptpubkey` の変更に受信ノードの協力を必要とすることで、セキュリティを段階的に高めます。

`shutdown` への返信要件は、返信する前に未処理の変更をコミットするため `commitment_signed` を送ることを意味します。理論的には代わりに再接続することもでき、その場合は未コミットの変更がすべて消去されます。

`OP_RETURN` は、PUSH オペコードのみが続き、スクリプト全体が 83 バイト以下である場合のみ標準とみなされます。本仕様では、それを少し厳しくして単一の PUSH のみを許容しています。これにはスクリプト上 2 つの形式があり、1 つは最大 75 バイトをプッシュする形式、もう 1 つは 76〜80 バイトに必要な長めの形式 (`OP_PUSHDATA1`) です。

### クローズ交渉: `closing_complete` と `closing_sig`

シャットダウンが完了し、チャネルから HTLC がなくなり、取り消し待ちのコミットメントもなく、すべての更新が両方のコミットメントに反映されると、最終的な現行コミットメントトランザクションには HTLC がない状態になります。

`option_simple_close` が交渉されていない場合は、下記の [レガシークローズ交渉](#legacy-closing-negotiation-closing_signed) を参照してください。

各ピアは自身が手数料を支払うクローズトランザクションを作り、そのトランザクションの詳細を含めて `closing_complete` を相手ピアに送ります。受け取ったピアはそのトランザクションに署名し、`closing_sig` を返します。これにより各ピアが独立に `closing_complete` を送って `closing_sig` を受け取り、独立した (互いに競合する) 2 つのクローズトランザクションが作られます。

支払い額の少ない側は (もしいれば)、自分の出力をクローズトランザクションから省略してもよいです。

このプロセスは `closing_complete` を再送することで何度でも繰り返せ、手数料の増加や出力スクリプトの変更を行えます。

1. type: 40 (`closing_complete`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`closer_scriptpubkey_len`]
   * [`closer_scriptpubkey_len*byte`:`closer_scriptpubkey`]
   * [`u16`:`closee_scriptpubkey_len`]
   * [`closee_scriptpubkey_len*byte`:`closee_scriptpubkey`]
   * [`u64`:`fee_satoshis`]
   * [`u32`:`locktime`]
   * [`closing_tlvs`:`tlvs`]

1. type: 41 (`closing_sig`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`closer_scriptpubkey_len`]
   * [`closer_scriptpubkey_len*byte`:`closer_scriptpubkey`]
   * [`u16`:`closee_scriptpubkey_len`]
   * [`closee_scriptpubkey_len*byte`:`closee_scriptpubkey`]
   * [`u64`:`fee_satoshis`]
   * [`u32`:`locktime`]
   * [`closing_tlvs`:`tlvs`]

1. `tlv_stream`: `closing_tlvs`
2. types:
    1. type: 1 (`closer_output_only`)
    2. data:
        * [`signature`:`sig`]
    1. type: 2 (`closee_output_only`)
    2. data:
        * [`signature`:`sig`]
    1. type: 3 (`closer_and_closee_outputs`)
    2. data:
        * [`signature`:`sig`]

#### 要件

注: 署名対象トランザクションの詳細と要件は [BOLT 3](03-transactions.md#closing-transaction) を参照してください。

ある出力が [Bitcoin Core のダスト閾値](03-transactions.md#dust-limits) より小さい場合、その出力は *ダスト* と見なされます。

注: ここに書かれた要件は `option_simple_close` が交渉された場合のみ適用されます。それ以外の場合の要件は [レガシークローズ交渉](#legacy-closing-negotiation-closing_signed) にあります。

両ノード:
  - `shutdown` を送受信した後で、どちらのコミットメントトランザクションにも HTLC が残っていない場合:
    - `closing_complete` を送信すべきです。

`closing_complete` の送信者 (「クローザー」):
  - `fee_satoshis` を、自身の残高以下の額にサトシ単位で切り下げて設定しなければなりません。
  - 少なくとも 1 つの出力がダストにならないように `fee_satoshis` を設定しなければなりません。
  - `closer_scriptpubkey` を、自身が望む出力スクリプトに設定しなければなりません。
  - `closee_scriptpubkey` を、ピアから直近に受信したスクリプト (`closing_complete` から、または最初の `shutdown` から) に設定しなければなりません。
  - `locktime` を、クローズトランザクションの希望する `nLockTime` に設定しなければなりません。
  - 自身の残高 (millisatoshi) がリモート残高より少ない場合:
    - `closer_output_only` を設定してはなりません。
    - 自身の出力金額がダストの場合、`closee_output_only` を設定しなければなりません。
    - 自身の出力金額が経済的でないと判断され、かつ `closer_scriptpubkey` が `OP_RETURN` でない場合、`closee_output_only` を設定してもよいです。
  - そうでない場合 (残高が少ない側ではないため、自分の出力を取り除けない):
    - `closee_output_only` を設定してはなりません。
    - 自身の出力金額が経済的でないと判断する場合:
      - 有効な `OP_RETURN` スクリプトを `closer_scriptpubkey` として送信してもよいです。
      - その場合、出力金額はゼロに設定し、すべての資金が手数料に回るようにしなければなりません ([BOLT #3](03-transactions.md#closing-transaction) を参照)。
    - クローズイーの出力金額がダストの場合:
      - `closer_output_only` を設定しなければなりません。
      - `closer_and_closee_outputs` を設定してはなりません。
    - そうでない場合:
      - `closer_output_only` と `closer_and_closee_outputs` の両方を設定しなければなりません。
  - クローズトランザクションは [BOLT #3](03-transactions.md#closing-transaction) に従って生成しなければなりません。
  - `signature` フィールドには、自身の `funding_pubkey` を用いた以下の有効な署名を設定しなければなりません:
    - `closer_output_only`: ローカル ("クローザー") 出力のみのクローズトランザクション。
    - `closee_output_only`: リモート ("クローズイー") 出力のみのクローズトランザクション。
    - `closer_and_closee_outputs`: クローザーとクローズイー両方の出力を持つクローズトランザクション。
  - 別の `closing_complete` を送りたい場合 (例: 別の `fee_satoshis` や `closer_scriptpubkey` で):
    - まず `closing_sig` を受け取るまで待たなければなりません。
    - `closing_sig` を受信できない場合、接続を閉じるべきです。

`closing_complete` の受信者 (「クローズイー」):
  - `fee_satoshis` がクローザーの残高を超える場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `closee_scriptpubkey` が、自身が直近に送信したスクリプト (`closing_complete` から、または最初の `shutdown` から) と一致しない場合:
    - `closing_complete` を無視すべきです。
    - `warning` を送信すべきです。
    - 接続を閉じるべきです。
  - `closer_scriptpubkey` が無効な場合 ([`shutdown` の要件](#closing-initiation-shutdown) を参照):
    - `closing_complete` を無視すべきです。
    - `warning` を送信すべきです。
    - 接続を閉じるべきです。
  - `closer_scriptpubkey` が有効な `OP_RETURN` スクリプトの場合:
    - クローザー出力の金額をゼロに設定し、すべての資金が手数料に回るようにしなければなりません ([BOLT #3](03-transactions.md#closing-transaction) を参照)。
  - リモートクローズトランザクションを [BOLT #3](03-transactions.md#closing-transaction) に従って生成しなければなりません。
  - 検証する署名を以下のように選択します:
    - 自身の出力金額がダストの場合:
      - `closer_output_only` を使用しなければなりません。
    - そうでなく、自身の出力金額が経済的でないと判断され、かつ `closee_scriptpubkey` が `OP_RETURN` でない場合:
      - `closer_output_only` を使用しなければなりません。
    - そうでなく、`closer_and_closee_outputs` が含まれる場合:
      - `closer_and_closee_outputs` を使用しなければなりません。
    - それ以外の場合:
      - `closee_output_only` を使用しなければなりません。
  - 選択した署名フィールドが存在しない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 該当するクローズトランザクションに対して署名フィールドが無効な場合 ([BOLT #3](03-transactions.md#closing-transaction) を参照):
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 署名フィールドが LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 該当するクローズトランザクションに署名してブロードキャストしなければなりません。
  - `closing_sig` には、`closing_complete` と同じ TLV フィールドに有効な単一の署名を入れて送信しなければなりません。
  - 自分が今後送る `closing_complete` メッセージでは `closer_scriptpubkey` を使用しなければなりません。

`closing_sig` の受信者:
  - `closer_scriptpubkey`、`closee_scriptpubkey`、`fee_satoshis`、または `locktime` が `closing_complete` で送ったものと一致しない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `tlvs` にちょうど 1 つの署名が含まれていない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `tlvs` に `closing_complete` で送った TLV フィールドが含まれていない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 該当するクローズトランザクションに対して署名フィールドが無効な場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 署名フィールドが LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - そうでない場合:
    - 該当するクローズトランザクションをブロードキャストしなければなりません。
  - 別の `fee_satoshis` や `closer_scriptpubkey` で `closing_complete` を再度送ってもよいです。

### 根拠

クローズプロトコルは、各側が自身の希望する手数料を支払う形にすることで、手数料合意の不一致による失敗シナリオを避ける設計になっています。

一方の残高が他方より少ない場合は、自分の出力を省略することを選んでもよいですが、その場合は得られたトランザクションがブロードキャスト可能となるようにダストを必ず省略しなければなりません。

両方の出力がダストになるほど手数料が高い場合のコーナーケースには 2 通りの対応があります: 低い手数料を払って問題を回避するか、`OP_RETURN` を使うか (これは「ダスト」になり得ません) です。一方が `OP_RETURN` 出力を選んだ場合、ブロードキャストできるよう金額は 0 でなければなりません。

通常、迅速処理のために高い手数料を払う理由はありません。緊急の子トランザクションがクローズトランザクションの代わりに手数料を払えるためです。CPFP が使えず、迅速処理が望まれる場合、クローザーは `closing_complete` を再送して、以前のクローズトランザクションを RBF できます。

新しい `closing_complete` メッセージは以前のものを上書きするため、再交渉も可能です (`upfront_shutdown_script` が交渉されていなければ出力アドレスも変更可能)。両ノードが同時に `closer_scriptpubkey` を変更しようとして `closing_complete` を送ったときには、稀ですがレース条件が起きます。受信した `closing_complete` は相手が以前の出力スクリプトを使っていることになるため、対応するトランザクションには署名すべきではありません。この場合は再接続するだけでよく、それにより両ノードが `shutdown` で最新の出力スクリプトを送り直し、署名フローを再開する機会が得られます。`closing_sig` にもクローザー/クローズイーのスクリプトを含めることで、相手がスクリプト不一致を検出して署名を正しく無視できるようにし、レース条件のデバッグも助けます。

クローザーがリレーされないトランザクション (出力がダストである、または手数料率が低すぎる) を提案しても、クローズイーが署名すること自体に害はありません。

同様に、クローザーが高い手数料を提案しても、クローザーが支払うのですからクローズイーが署名しても害はありません。

各側は手数料を相手に押し付けたいと考え、最小限の手数料を提案する弱いゲームが発生します。どちらの側もリレーされる手数料を提案しなかった場合、再交渉するか、最終的なコミットメントトランザクションを使うことになります。実際にはオープナー側がコミットメントトランザクションの手数料を負担し、その消費にもさらに手数料がかかるため、合理的なクローズ手数料を提示するインセンティブがあります。

### レガシークローズ交渉: `closing_signed`

シャットダウンが完了し、チャネルから HTLC がなくなり、取り消し待ちのコミットメントもなく、すべての更新が両方のコミットメントに含まれた状態になると、最終的な現行コミットメントトランザクションには HTLC が無くなり、クローズ手数料の交渉が始まります。`option_simple_close` が交渉されている場合は前節が適用され、それ以外の場合に本節が適用されます。

ファンダーは公平と判断する手数料を選び、`shutdown` メッセージの `scriptpubkey` フィールド (および選択した手数料) でクローズトランザクションに署名し、署名を送信します。次にもう一方のノードも同様に、自分が公平と思う手数料で返答します。このやり取りは両者が同じ手数料に合意するか、一方がチャネルを失敗させるまで続きます。

現代的な方式では、ファンダーが許容できる手数料の範囲を送り、非ファンダーがその範囲内から手数料を選びます。非ファンダーが同じ値を選んだ場合は 2 メッセージで交渉が完了し、そうでない場合はファンダーが同じ値で返答するため 3 メッセージで完了します。

1. type: 39 (`closing_signed`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`fee_satoshis`]
   * [`signature`:`signature`]
   * [`closing_signed_tlvs`:`tlvs`]

1. `tlv_stream`: `closing_signed_tlvs`
2. types:
    1. type: 1 (`fee_range`)
    2. data:
        * [`u64`:`min_fee_satoshis`]
        * [`u64`:`max_fee_satoshis`]

#### 要件

注: ここの要件は `option_simple_close` が交渉されていない場合のみ適用されます。それ以外の場合は [クローズ交渉: `closing_complete` と `closing_sig`](#closing-negotiation-closing_complete-and-closing_sig) の要件が適用されます。

ファンディングノード:
  - `shutdown` が受信されており、どちらのコミットメントトランザクションにも HTLC が残っていない場合:
    - `closing_signed` メッセージを送信すべきです。

送信ノード：

- ブロックに含まれるコストの見積もりに基づいて、初期の `fee_satoshis` を設定するべきです。
- クローズトランザクションに支払う準備がある最小および最大手数料に基づいて `fee_range` を設定するべきです。
- 合理的な時間が経過しても `closing_signed` の応答を受け取らない場合：
  - チャネルを失敗させなければなりません。
- 資金提供者でない場合：
  - 受け取った `max_fee_satoshis` 以上に `max_fee_satoshis` を設定するべきです。
  - `min_fee_satoshis` をかなり低い値に設定するべきです。
- [BOLT #3](03-transactions.md#closing-transaction) で指定されているように、クローズトランザクションの Bitcoin 署名に `signature` を設定しなければなりません。

受信ノード：

- [BOLT #3](03-transactions.md#closing-transaction) で指定されたクローズトランザクションのいずれかのバリアントに対して `signature` が有効でない場合、または LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合：
  - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
- `fee_satoshis` が以前に送信した `fee_satoshis` と等しい場合：
  - 最終的なクローズトランザクションに署名してブロードキャストするべきです。
  - 接続を閉じてもかまいません。
- `fee_satoshis` が以前に送信した `fee_range` と一致する場合：
  - `fee_satoshis` を使用して最終的なクローズトランザクションに署名してブロードキャストするべきです。
  - 以前に送信した `fee_satoshis` と異なる場合は、同じ `fee_satoshis` 値で `closing_signed` を返信するべきです。
  - 接続を閉じてもかまいません。
- メッセージに `fee_range` が含まれている場合：
  - それと自身の `fee_range` に重なりがない場合：
    - 警告を送信するべきです。
    - 合理的な時間が経過しても満足のいく `fee_range` を受け取らない場合、チャネルを失敗させなければなりません。
  - それ以外の場合：
    - 資金提供者である場合：
      - 送信および受信した `fee_range` の重なりに `fee_satoshis` がない場合：
        - チャネルを失敗させなければなりません。
      - それ以外の場合：
        - 同じ `fee_satoshis` で返信しなければなりません。
    - それ以外の場合（資金提供者でない場合）：
      - すでに `closing_signed` を送信している場合：
        - 送信した値と `fee_satoshis` が異なる場合：
          - チャネルを失敗させなければなりません。
      - それ以外の場合：
        - 受信した `fee_range` と（送信予定の）`fee_range` の重なりに `fee_satoshis` を提案しなければなりません。
- それ以外の場合、`fee_satoshis` が最後に送信した `fee_satoshis` と以前に受信した `fee_satoshis` の間に厳密にない場合、再接続していない限り：
  - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させるべきです。
- それ以外の場合、受信者が手数料に同意する場合：
  - 同じ `fee_satoshis` 値で `closing_signed` を返信するべきです。
- それ以外の場合：
  - 受信した `fee_satoshis` と以前に送信した `fee_satoshis` の間に「厳密に」ある値を提案しなければなりません。

受信ノード：

- クローズトランザクションの出力のいずれかが、その `scriptpubkey` のダストリミットを下回る場合 ([BOLT 3](03-transactions.md#dust-limits) を参照):
  - チャネルを失敗させなければなりません。

#### 理論的根拠

`fee_range` が提供されていない場合、「厳密に間にある」という要件は、たとえ 1 サトシずつであっても前進が行われることを保証します。状態を保持せず、切断と再接続の間に手数料が変動した場合のコーナーケースを処理するために、再接続時に交渉が再開されます。

クローズトランザクションが遅延するリスクは限定的で、すぐにブロードキャストされるため、通常は迅速処理のためにプレミアムを支払う理由はありません。

非資金提供者は手数料を支払わないため、最大手数料率を持つ理由はありません。ただし、トランザクションが伝播することを保証するために、最小手数料率を持ちたいかもしれません。必要に応じて後で CPFP を使用して確認を早めることができるため、その最小値は低くするべきです。

クローズトランザクションが Bitcoin のデフォルトのリレーポリシーを満たさないことがあります (例えば 546 サトシ未満の出力に非 segwit のシャットダウンスクリプトを用いる場合。`dust_limit_satoshis` が 546 サトシ未満なら起こり得ます)。資金が危険にさらされることはありませんが、クローズトランザクションがマイナーまで届く可能性はほぼないため、チャネルを強制クローズしなければなりません。

## 通常の操作

両方のノードが `channel_ready`（およびオプションで [`announcement_signatures`](07-routing-gossip.md#the-announcement_signatures-message)）を交換すると、チャネルはハッシュタイムロック契約を介して支払いを行うために使用できます。

変更はバッチで送信されます。`commitment_signed` メッセージの前に 1 つ以上の `update_` メッセージが送信されます。以下の図のように：

        +-------+                               +-------+
        |       |--(1)---- update_add_htlc ---->|       |
        |       |--(2)---- update_add_htlc ---->|       |
        |       |<-(3)---- update_add_htlc -----|       |
        |       |                               |       |
        |       |--(4)--- commitment_signed --->|       |
        |   A   |<-(5)---- revoke_and_ack ------|   B   |
        |       |                               |       |
        |       |<-(6)--- commitment_signed ----|       |
        |       |--(7)---- revoke_and_ack ----->|       |
        |       |                               |       |
        |       |--(8)--- commitment_signed --->|       |
        |       |<-(9)---- revoke_and_ack ------|       |
        +-------+                               +-------+

逆説的に言えば、これらの更新は*他のノードの*コミットメントトランザクションに適用されます。ノードは、リモートノードが `revoke_and_ack` を通じてそれらを適用したことを確認したときにのみ、自分のコミットメントトランザクションにそれらの更新を追加します。

したがって、各更新は次の状態を経て進行します。

1. 受信者で保留中
2. 受信者の最新のコミットメントトランザクションに含まれる
3. ... そして受信者の以前のコミットメントトランザクションが取り消され、更新が送信者で保留中
4. ... そして送信者の最新のコミットメントトランザクションに含まれる
5. ... そして送信者の以前のコミットメントトランザクションが取り消される

2つのノードの更新は独立しているため、2つのコミットメントトランザクションは無期限に同期していない可能性があります。これは問題ではありません。重要なのは、両方の側が特定の更新に対して取り消し不能なコミットメントを持っているかどうかです（上記の最終状態）。

### HTLC の転送

一般的に、ノードは自分自身の支払いを開始するため、または他のノードの支払いを転送するために HTLC を提供します。転送の場合、*送信側*の HTLC が*受信側*の HTLC を引き換えられる場合にのみ引き換えられるように注意する必要があります。以下の要件は、これが常に真であることを保証します。

HTLC のそれぞれの**追加/削除**は、次の場合に*取り消し不能なコミットメント*と見なされます。

1. それを含む/含まないコミットメントトランザクションが両方のノードによってコミットされ、以前のそれを含まない/含むコミットメントトランザクションが取り消された場合、または
2. それを含む/含まないコミットメントトランザクションがブロックチェーンに不可逆的にコミットされた場合。

#### 要件

ノードは：
  - 受信 HTLC が取り消し不能なコミットメントになるまで：
    - その受信 HTLC に応じて対応する送信 HTLC (`update_add_htlc`) を提供してはなりません。
  - 送信 HTLC の削除が取り消し不能なコミットメントになるまで、または送信オンチェーン HTLC 出力が HTLC-タイムアウトトランザクションを介して（十分な深さで）消費されるまで：
    - その送信 HTLC に対応する受信 HTLC (`update_fail_htlc`) を失敗させてはなりません。
  - 受信 HTLC の `cltv_expiry` に達した場合、または対応する送信 HTLC の `cltv_expiry` マイナス `current_height` が `cltv_expiry_delta` より小さい場合：
    - その受信 HTLC (`update_fail_htlc`) を失敗させなければなりません。
  - 受信 HTLC の `cltv_expiry` が不当に遠い将来である場合：
    - その受信 HTLC (`update_fail_htlc`) を失敗させるべきです。
  - 送信 HTLC に対する `update_fulfill_htlc` を受信した場合、またはオンチェーン HTLC 消費から `payment_preimage` を発見した場合：
    - その送信 HTLC に対応する受信 HTLC を履行しなければなりません。

#### 根拠

一般的に、交換の片方はもう片方より先に処理する必要があります。HTLC（Hashed Time-Locked Contract）の履行は異なります。プレイメージの知識は定義上、取り消し不可能であり、遅延を減らすために受信した HTLC はできるだけ早く履行されるべきです。

不合理に長い期限の HTLC はサービス拒否のベクトルとなるため、許可されません。「不合理」の正確な値は現在不明であり、ネットワークトポロジーに依存する可能性があります。

### `cltv_expiry_delta` の選択

HTLC がタイムアウトした場合、それは履行されるかタイムアウトされるかのいずれかです。この移行には注意が必要で、提供された HTLC と受信した HTLC の両方に対して考慮が必要です。

次のシナリオを考えてみましょう。A が B に HTLC を送り、B が C に転送し、C が支払いを受け取るとすぐに商品を届ける場合です。

1. C は、B が応答しなくなった場合でも、B からの HTLC がタイムアウトできないことを確認する必要があります。つまり、C は B がオンチェーンでタイムアウトする前に、オンチェーンで受信した HTLC を履行できます。

2. B は、C が B からの HTLC を履行した場合、A からの受信 HTLC を履行できることを確認する必要があります。つまり、B は C からプレイメージを取得し、A がオンチェーンでタイムアウトする前に、オンチェーンで受信した HTLC を履行できます。

ここでの重要な設定は、[BOLT #7](07-routing-gossip.md#the-channel_update-message) の `cltv_expiry_delta` と、関連する [BOLT #11](11-payment-encoding.md#tagged-fields) の `min_final_cltv_expiry_delta` です。`cltv_expiry_delta` は、転送ケース (B) における HTLC CLTV タイムアウトの最小差です。`min_final_cltv_expiry_delta` は、ターミナルケース (C) における HTLC CLTV タイムアウトと現在のブロック高の最小差です。

ノードが一つのチャネルで HTLC を受け入れ、別のチャネルで CLTV タイムアウトの差が小さすぎる HTLC を提供する場合、リスクがあります。このため、ノードを横断するデルタとして *送信* チャネルの `cltv_expiry_delta` が使用されます。

送信 HTLC の解決と受信 HTLC の解決の間の最悪のケースのブロック数は、いくつかの仮定に基づいて導出できます：

* 最悪のケースの再編成深度 `R` ブロック
* 応答しないピアを諦めてチェーンに落とす前の HTLC タイムアウト後の猶予期間 `G` ブロック
* トランザクションのブロードキャストとブロックに含まれるまでのブロック数 `S`

最悪のケースは、転送ノード (B) が送信先の HTLC 完了を見つけるのに最も長い時間を要し、さらにそれをオンチェーンで引き換えるのにも最も長い時間を要する場合です。

1. B->C の HTLC がブロック `N` でタイムアウトし、B は C を待つのを諦めるまで `G` ブロック待ちます。B または C がブロックチェーンにコミットし、B が HTLC を消費しますが、これには `S` ブロックかかります。
2. 悪いケース：C が競争に勝ち (ぎりぎりで) HTLC を完了し、B はブロック `N+G+S+1` でそのトランザクションを初めて見ます。
3. 最悪のケース：C が勝ち、完了する `R` 深の再編成があります。B は `N+G+S+R` でトランザクションを初めて見ます。
4. B は今度は受信側の A->B HTLC を完了する必要がありますが、A が応答しません。B は A を待つのを諦めるまでさらに `G` ブロック待ちます。A または B がブロックチェーンにコミットします。
5. 悪いケース：B がブロック `N+G+S+R+G+1` で A のコミットメントトランザクションを見て、HTLC 出力を消費する必要があり、これには `S` ブロックかかります。
6. 最悪のケース：A がコミットメントトランザクションを消費するために使用する `R` 深の再編成があり、B はブロック `N+G+S+R+G+R` で A のコミットメントトランザクションを見て、HTLC 出力を消費する必要があり、これには `S` ブロックかかります。
7. B の HTLC 消費はタイムアウトする前に少なくとも `R` 深である必要があります。そうでないと、別の再編成により A がトランザクションをタイムアウトさせることができる可能性があります。

したがって最悪ケースは `3R+2G+2S` です (ただし `R` は少なくとも 1 と仮定)。`R` が 2 以上のとき、他ノードが連続するすべての再編成に勝つ可能性は低くなります。HTLC の消費にはほぼ任意の手数料を付けられるため通常運用時の `S` は小さいはずですが、ブロック時間が不規則で空ブロックも生じ得て手数料は大きく変動し、HTLC トランザクションの手数料を引き上げる手段はないため、`S=12` を最小値と考えるべきです。`S` は攻撃下で最も変動しやすいパラメータでもあり、無視できない額が危険にさらされる場合はより大きな値が望まれることがあります。猶予期間 `G` は小さく (1 または 2) し、ノードはできるだけ早くタイムアウトまたは履行を行うべきです。ただし `G` が小さすぎると、ネットワーク遅延による不要なチャネルクローズのリスクが高まります。

以下の 4 つの値を導出する必要があります。

1. チャネルの `cltv_expiry_delta`、`3R+2G+2S`：不明な場合は、少なくとも 34 の `cltv_expiry_delta` が妥当です (R=2, G=2, S=12)。

2. 提供された HTLC の期限：チャネルが失敗し、オンチェーンでタイムアウトするまでの期限です。これは HTLC の `cltv_expiry` の後 `G` ブロックです。1 または 2 ブロックが妥当です。

3. このノードが履行した受信 HTLC の期限：チャネルが失敗し、`cltv_expiry` の前にオンチェーンで HTLC が履行されるまでの期限です。上記のステップ 4-7 を参照してください。これは `cltv_expiry` の前 `2R+G+S` ブロックを意味します。18 ブロックが妥当です。

4. 終端支払いに対して受け入れられる最小 `cltv_expiry`：終端ノード C の最悪のケースは `2R+G+S` ブロックです (再び、上記のステップ 1-3 は適用されません)。[BOLT #11](11-payment-encoding.md) のデフォルトは 18 であり、この計算と一致します。

#### 要件

提供ノード：
  - 提供する各 HTLC のタイムアウト期限を推定しなければなりません。
  - `cltv_expiry` の前にタイムアウト期限がある HTLC を提供してはなりません。
  - 提供した HTLC がどちらかのノードの現在のコミットメントトランザクションに含まれており、このタイムアウト期限を過ぎている場合：
    - 受信ピアに `error` を送信するべきです (接続されている場合)。
    - チャネルを失敗させなければなりません。

履行ノード：
  - 履行しようとしている各 HTLC に対して：
    - 履行期限を推定しなければなりません。
  - 履行期限がすでに過ぎている HTLC を失敗させ (転送せず)、なければなりません。
  - 履行した HTLC がどちらかのノードの現在のコミットメントトランザクションに含まれており、この履行期限を過ぎている場合：
    - 提供ピアに `error` を送信するべきです (接続されている場合)。
    - チャネルを失敗させなければなりません。

### トリムされたインフライト HTLC への露出の制限：`max_dust_htlc_exposure_msat`

チャネル内の HTLC が [BOLT3 #3](03-transactions.md) の「トリムされた」しきい値を下回る場合、HTLC はオンチェーンで請求できず、どちらかの当事者がチャネルを一方的に閉じた場合に追加のマイナー手数料に変わります。しきい値は HTLC ごとに設定されているため、チャネルが強制的に閉じられたときに多くのダスト HTLC がコミットされている場合、これらの HTLC への総露出はかなりのものになる可能性があります。

これは、悪意のある主体が <sup>[mining capabilities](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-May/002714.html)</sup> を獲得した場合、グリーフィング攻撃や miner-extractable-value 攻撃で悪用される可能性があります。

総露出は次の簡易計算によって示されます：

	remote `max_accepted_htlcs` * (`HTLC-success-kiloweight` * `feerate_per_kw` + remote `dust_limit_satoshis`)
		+ local `max_accepted_htlcs` * (`HTLC-timeout-kiloweight` * `feerate_per_kw` + remote `dust_limit_satoshis`)

このシナリオを緩和するために、`max_dust_htlc_exposure_msat` の閾値を HTLC を送信、転送、受信する際に適用することができます。

ノードは次のように動作します：
  - HTLC を受信する際：
    - HTLC の `amount_msat` が remote `dust_limit_satoshis` と `feerate_per_kw` での HTLC-timeout 手数料を足したものより小さい場合：
      - `amount_msat` と remote トランザクションのダストバランスを足したものが `max_dust_htlc_exposure_msat` を超える場合：
        - この HTLC をコミットしたら失敗させるべきです
        - この HTLC のプレイメージを公開すべきではありません
    - HTLC の `amount_msat` が local `dust_limit_satoshis` と `feerate_per_kw` での HTLC-success 手数料を足したものより小さい場合：
      - `amount_msat` と local トランザクションのダストバランスを足したものが `max_dust_htlc_exposure_msat` を超える場合：
        - この HTLC をコミットしたら失敗させるべきです
        - この HTLC のプレイメージを公開すべきではありません
  - HTLC を提供する際：
    - HTLC の `amount_msat` が remote `dust_limit_satoshis` と `feerate_per_kw` での HTLC-success 手数料を足したものより小さい場合：
      - `amount_msat` と remote トランザクションのダストバランスを足したものが `max_dust_htlc_exposure_msat` を超える場合：
        - この HTLC を送信すべきではありません
        - 対応する受信 HTLC（ある場合）を失敗させるべきです
    - HTLC の `amount_msat` がホルダーの `dust_limit_satoshis` と `feerate_per_kw` での HTLC-timeout 手数料を足したものより小さい場合：
      - `amount_msat` と local トランザクションのダストバランスを足したものが `max_dust_htlc_exposure_msat` を超える場合：
        - この HTLC を送信すべきではありません
        - 対応する受信 HTLC（ある場合）��失敗させるべきです

`max_dust_htlc_exposure_msat` は、ダストエクスポージャーからのトリムされた残高の上限です。使用される正確な値はノードポリシーによります。

`option_anchors` を使用しないチャネルでは、`feerate_per_kw` の増加により、コミットメントトランザクションから複数の HTLC がトリムされる可能性があり、これがダストエクスポージャーの大幅な増加を引き起こすことがあります。

### HTLC の追加: `update_add_htlc`

どちらのノードも `update_add_htlc` を送信して、HTLC を相手に提供できます。これは支払いプレイメージと引き換えに償還可能です。金額は millisatoshi で表されますが、オンチェーンでの強制執行はダスト制限を超える全サトシ金額に対してのみ可能です（コミットメントトランザクションでは [BOLT #3](03-transactions.md) に指定されているように切り捨てられます）。

支払いの宛先を示す `onion_routing_packet` 部分のフォーマットは [BOLT #4](04-onion-routing.md) に記載されています。

1. タイプ: 128 (`update_add_htlc`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`u64`:`amount_msat`]
   * [`sha256`:`payment_hash`]
   * [`u32`:`cltv_expiry`]
   * [`1366*byte`:`onion_routing_packet`]

1. `tlv_stream`: `update_add_htlc_tlvs`
2. タイプ:
    1. タイプ: 0 (`blinded_path`)
    2. データ:
        * [`point`:`path_key`]

#### 要件

送信ノード:
  - ビットコイン手数料の支払いに責任がある場合:
    - 自分のコミットメントトランザクションにその HTLC を追加した後、現在の `feerate_per_kw` でローカルまたはリモートのコミットメントトランザクションの手数料を支払いながらチャネルリザーブを維持できない場合、`amount_msat` を提供してはなりません（[手数料の更新](#updating-fees-update_fee)を参照）。
    - `option_anchors` がこのコミットメントトランザクションに適用され、送信ノードがファンダーである場合:
      - リザーブを超えて `to_local_anchor` と `to_remote_anchor` の支払いもできなければなりません。
    - 自分のコミットメントトランザクションにその HTLC を追加した後、将来の追加の非ダスト HTLC を受け取るまたは送信する際にコミットメントトランザクションの手数料を支払うための残高が残らない場合、`amount_msat` を提供すべきではありません。この「手数料スパイクバッファー」は、実装間の予測可能性を確保するために、現在の `feerate_per_kw` の 2 倍を処理できることが推奨されます。
  - ビットコイン手数料の支払いに責任がない場合:
    - リモートノードがその HTLC をコミットメントトランザクションに追加した後、現在の `feerate_per_kw` で更新されたローカルまたはリモートトランザクションの手数料を支払いながらチャネルリザーブを維持できない場合、`amount_msat` を提供すべきではありません。
  - `amount_msat` を 0 より大きく提供しなければなりません。
  - 受信ノードの `htlc_minimum_msat` 未満の `amount_msat` を提供してはなりません。
  - `cltv_expiry` を 500000000 未満に設定しなければなりません。
  - リモートの `max_accepted_htlcs` HTLC を超える場合、リモートコミットメントトランザクションで:
    - HTLC を追加してはなりません。
  - 提供された HTLC の総価値がリモートの `max_htlc_value_in_flight_msat` を超える場合:
    - HTLC を追加してはなりません。
  - 最初に提供する HTLC について:
    - `id` を 0 に設定しなければなりません。
  - 各連続するオファーごとに `id` の値を 1 増やさなければなりません。
  - ブラインドルート内で支払いを中継している場合:
    - `path_key` を設定しなければなりません ([ルートブラインディング](04-onion-routing.md#route-blinding) を参照)。
  - スプライスが保留中の場合:
    - すべてのコミットメントトランザクションについて要件を満たすことを保証しなければなりません。

`id` は更新が完了したあと (`revoke_and_ack` を受信したあと) も 0 にリセットしてはいけません。代わりにインクリメントし続けなければなりません。

受信ノード：
  - `amount_msat` が 0 に等しい、または自身の `htlc_minimum_msat` より少ない場合を受信した場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させるべきです。
  - 送信ノードが現在の `feerate_per_kw` で負担できない `amount_msat` を受信した場合（チャネルリザーブおよび `to_local_anchor` と `to_remote_anchor` のコストを維持しながら）：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させるべきです。
  - 送信ノードが受信者の `max_accepted_htlcs` を超える HTLC をローカルコミットメントトランザクションに追加した場合、または受信者の `max_htlc_value_in_flight_msat` を超える価値のある HTLC をローカルコミットメントトランザクションに追加した場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させるべきです。
  - 送信ノードが `cltv_expiry` を 500000000 以上に設定した場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させるべきです。
  - 同じ `payment_hash` を持つ複数の HTLC を許可しなければなりません。
  - 送信者がその HTLC のコミットメントを以前に承認していない場合：
    - 再接続後に繰り返される `id` 値を無視しなければなりません。
  - 他の `id` 違反が発生した場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させてもかまいません。
  - `onion_routing_packet` を [Onion Decryption](04-onion-routing.md#onion-decryption) で説明されているように復号して `payload` を抽出しなければなりません。
    - `path_key` (指定されている場合) を使用しなければなりません。
    - `payment_hash` を `associated_data` として使用しなければなりません。
  - 復号に失敗した、結果が有効な `payload` TLV でない、または未知の偶数型を含む場合:
    - [Failure Messages](04-onion-routing.md#failure-messages) に詳述されたエラーで応答しなければなりません。
  - それ以外の場合:
    - [Payload Format](04-onion-routing.md#payload-format) のリーダー要件に従わなければなりません。
  - スプライスが保留中の場合:
    - すべてのコミットメントトランザクションについて要件を満たすことを保証しなければなりません。

`onion_routing_packet` には、経路に沿った各ホップのための指示とホップのリストが難読化された状態で含まれています。これは `payment_hash` を関連データとして設定することで HTLC にコミットします。つまり、HMAC の計算に `payment_hash` を含めます。これにより、異なる `payment_hash` で以前の `onion_routing_packet` を再利用するリプレイ攻撃を防ぎます。

#### 理論的根拠

無効な金額は明らかなプロトコル違反であり、破綻を示します。

もしノードが同じ payment hash を持つ複数の HTLC を受け入れなかった場合、攻撃者はノードに既存の HTLC があるかどうかを調べることができてしまいます。この重複を処理するための要件は、別の識別子の使用につながります。64 ビットのカウンタがオーバーフローしないと仮定しています。

未承認の更新の再送信は、再接続の目的で明示的に許可されています。他の時点でそれを許可することは、受信者のコードを簡素化します（ただし、厳密なチェックはデバッグに役立つかもしれません）。

`max_accepted_htlcs` は 483 に制限されており、両側が最大数の HTLC を送信した場合でも、`commitment_signed` メッセージが最大メッセージサイズを超えないようにしています。また、単一のペナルティトランザクションが、[BOLT #5](05-onchain.md#penalty-transaction-weight-calculation) で計算されるように、コミットメントトランザクション全体を消費できることを保証します。

`cltv_expiry` 値が 500000000 以上である場合、秒単位の時間を示すことになり、プロトコルはブロック単位の有効期限のみをサポートします。

ビットコイン手数料を支払う責任のあるノードは、将来の手数料の増加に対応するために、リザーブの上に「手数料スパイクバッファ」を維持する必要があります。このバッファがないと、ビットコイン手数料を支払う責任のあるノードは、チャネルリザーブを維持しながら非ダスト HTLC を送受信できない状態に陥る可能性があります（コミットメントトランザクションの重みが増加するため）、結果としてチャネルが劣化します。詳細は [#728](https://github.com/lightningnetwork/lightning-rfc/issues/728) を参照してください。

### HTLC の削除: `update_fulfill_htlc`、`update_fail_htlc`、および `update_fail_malformed_htlc`

簡単のため、ノードは他のノードによって追加された HTLC のみを削除できます。HTLC を削除する理由は 4 つあります。支払いプレイメージが提供された場合、タイムアウトした場合、ルートに失敗した場合、または不正な形式である場合です。

プリイメージを提供するには：

1. type: 130 (`update_fulfill_htlc`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`32*byte`:`payment_preimage`]

タイムアウトまたはルート失敗した HTLC の場合：

1. type: 131 (`update_fail_htlc`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`u16`:`len`]
   * [`len*byte`:`reason`]

`reason` フィールドは、元の HTLC イニシエータの利益のために暗号化された不透明なデータです。これは [BOLT #4](04-onion-routing.md) で定義されています。ただし、ピアがそれを解析できなかった場合の特別な不正な失敗バリアントがあります。この場合、現在のノードは代わりに行動を起こし、`update_fail_htlc` に暗号化して中継します。

解析できない HTLC の場合：

1. type: 135 (`update_fail_malformed_htlc`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`sha256`:`sha256_of_onion`]
   * [`u16`:`failure_code`]

#### 要件

ノードは：
  - できるだけ早く HTLC を削除するべきです。
  - タイムアウトした HTLC を失敗させるべきです。
  - 対応する HTLC が両側のコミットメントトランザクションで取り消し不能にコミットされるまで：
    - `update_fulfill_htlc`、`update_fail_htlc`、または `update_fail_malformed_htlc` を送信してはなりません。
  - 受信 HTLC を失敗させるとき：
    - `current_path_key` がオニオンペイロードに設定されており、それが最終ノードでない場合：
      - 任意のローカルまたは下流のエラーに対して `invalid_onion_blinding` 失敗コードを使用して `update_fail_htlc` エラーを送信しなければなりません。
      - 受信したオニオンの `sha256_of_onion` を使用するべきです。
      - 全てゼロの `sha256_of_onion` を使用してもよいです。
      - `update_fail_htlc` を送信する前にランダムな遅延を追加するべきです。
    - 受信した `update_add_htlc` に `path_key` が設定されている場合：
      - 任意のローカルまたは下流のエラーに対して `invalid_onion_blinding` 失敗コードを使用して `update_fail_malformed_htlc` エラーを送信しなければなりません。
      - 受信したオニオンの `sha256_of_onion` を使用するべきです。
      - 全てゼロの `sha256_of_onion` を使用してもよいです。

受信ノードは：
  - `id` が現在のコミットメントトランザクション内の HTLC に対応していない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `update_fulfill_htlc` の `payment_preimage` 値が対応する HTLC `payment_hash` に SHA256 ハッシュされない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `update_fail_malformed_htlc` の `failure_code` に `BADONION` ビットが設定されていない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `update_fail_malformed_htlc` の `sha256_of_onion` が送信したオニオンと一致せず、全てゼロでない場合：
    - 再試行するか、別のエラーレスポンスを選択してもよいです。
  - それ以外の場合、`update_fail_malformed_htlc` によって送信されたアウトゴーイング HTLC がキャンセルされた受信ノードは：
    - 元々 HTLC を送信したリンクに送信する `update_fail_htlc` において、与えられた `failure_code` を使用し、データを `sha256_of_onion` に設定してエラーを返さなければなりません。

#### 理論的根拠

HTLC のタイムアウトを設定しないノードは、チャネルの失敗を招くリスクがあります（[`cltv_expiry_delta` の選択](#cltv_expiry_delta-selection)を参照）。

送信者より先に `update_fulfill_htlc` を送信するノードも、HTLC にコミットしており、資金を失うリスクがあります。

オニオンが不正な場合、上流ノードはレスポンスを生成するための共有キーを抽出できません。そのため、このノードがそれを行う特別な失敗メッセージが必要です。

ノードは、上流が問題としている SHA256 が送信したオニオンと一致するかどうかを確認できます。これによりランダムなビットエラーを検出できるかもしれません。しかし、実際に送信された暗号化パケットを再確認しない限り、エラーが自分のものかリモートのものかはわかりません。そのため、このような検出はオプションとして残されています。

ブラインドルート内のノードは、`invalid_onion_blinding` を使用して、ブラインドルートをプローブしようとする送信者に情報を漏らさないようにする必要があります。

### チャネルメッセージのバッチ処理

複数のチャネルメッセージは、`start_batch` メッセージを使うことで 1 つの論理メッセージとしてまとめて扱えます。

1. type: 127 (`start_batch`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`batch_size`]

1. `tlv_stream`: `start_batch_tlvs`
2. types:
   1. type: 1 (`message_type`)
   2. data:
     * [`u16`:`message_type`]

#### 要件

送信ノード:
  - `batch_size` を 1 より大きい値に設定しなければなりません。
  - `batch_size` を 20 以下の値に設定しなければなりません。
  - `message_type` を `132` (すなわち [BOLT 1](./01-messaging.md#lightning-message-format) で定義された `commitment_signed` メッセージ型) に設定しなければなりません。
  - `start_batch` を送信した後:
    - 同じ `channel_id` の `commitment_signed` メッセージを `batch_size` 個、間に無関係なメッセージを挟まずに送信しなければなりません。

受信ノード:
  - `batch_size` が 1 より大きくない場合:
    - `start_batch` メッセージを無視しなければなりません。
    - `warning` を送信すべきです。
  - `batch_size` が 20 を超える場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 続く `batch_size` 個のメッセージをグループ化し、まとめて処理しなければなりません。
  - そのうちのいずれかが指定された `channel_id` 向けでない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `message_type` が指定されていない、または `commitment_signed` の型に設定されていない場合:
    - `start_batch` メッセージを無視し、続くメッセージは順次処理しなければなりません。

#### 根拠

`start_batch` メッセージは現状、スプライシング中に複数の `commitment_signed` を送信する場合にのみ用いられます。そのためここではこの特定のシナリオに合わせて要件を絞っています。将来 `start_batch` を他の機能に使う場合には、この要件は緩和されてよいです。

`batch_size` を 20 に制限しているのは、過剰なキューイングを悪用した受信ノードへの DoS を防ぐためです。`start_batch` は今のところスプライス RBF 試行のみで使われ、トランザクションを確認させるのにそれほど多くの試行は不要です。

### これまでの更新のコミット: `commitment_signed`

ノードがリモートコミットメントに変更を持っている場合、それを適用し ([BOLT #3](03-transactions.md) で定義された) 結果のトランザクションに署名して `commitment_signed` メッセージを送信できます。

1. type: 132 (`commitment_signed`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`signature`:`signature`]
   * [`u16`:`num_htlcs`]
   * [`num_htlcs*signature`:`htlc_signature`]
   * [`commitment_signed_tlvs`:`tlvs`]

1. `tlv_stream`: `commitment_signed_tlvs`
2. types:
   1. type: 1 (`funding_txid`)
   2. data:
     * [`sha256`:`funding_txid`]

#### 要件

送信ノード:
  - 更新を 1 つも含まない `commitment_signed` メッセージを送信してはなりません。
  - 手数料のみを変更する `commitment_signed` メッセージを送信してもよいです。
  - 新しい取り消し番号以外にコミットメントトランザクションを変更しない `commitment_signed` メッセージを送信してもよいです (ダスト化、同一 HTLC の差し替え、または無視できる手数料変更などによる)。
  - コミットメントトランザクションの並びに対応する HTLC トランザクションごとに、対応する `htlc_signature` を 1 つ含めなければなりません ([BOLT #3](03-transactions.md#transaction-input-and-output-ordering) を参照)。
  - `funding_txid` を、このコミットメントトランザクションが消費する資金調達トランザクションに設定しなければなりません。
  - 最近リモートノードからメッセージを受信していない場合:
    - `commitment_signed` を送信する前に `ping` を使い、`pong` の返信を待つべきです。
  - 保留中のスプライストランザクションが `N` 個 (`N` > 0) ある場合:
    - 先に `start_batch` を、`batch_size` を `N + 1` に、`message_type` を `132` (`commitment_signed`) に設定して送信しなければなりません。
    - 現在のチャネル資金調達出力に対する `commitment_signed` を送信しなければなりません。
    - 各スプライストランザクションに対して `commitment_signed` を送信しなければなりません。
    - 各 `commitment_signed` メッセージの `funding_txid` を、対応するコミットメントトランザクションが消費する資金調達トランザクションに設定しなければなりません。
    - 一連の `commitment_signed` を送り終えるまで、他のメッセージを送信してはなりません。

受信ノード:
  - 保留中のすべての更新が適用されたあと:
    - `signature` がローカルコミットメントトランザクションに対して無効、または LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合:
      - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
    - `num_htlcs` がローカルコミットメントトランザクションの HTLC 出力数と一致しない場合:
      - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - いずれかの `htlc_signature` が対応する HTLC トランザクションに対して無効、または LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 保留中のスプライストランザクションがあるにもかかわらず、送信ノードが `start_batch` を送らずにそのバッチを送ってこなかった場合:
    - `error` を送信してチャネルを失敗させなければなりません。
  - 送信ノードが `start_batch` を送り、`commitment_signed` のバッチを処理している場合:
    - いずれかの `commitment_signed` メッセージで `funding_txid` が欠けている場合:
      - `error` を送信してチャネルを失敗させなければなりません。
    - 保留中のスプライストランザクションがある場合:
      - 各 `commitment_signed` を `funding_txid` に基づいて検証しなければなりません。
      - ある資金調達トランザクションに対する `commitment_signed` が欠けている場合:
        - `error` を送信してチャネルを失敗させなければなりません。
      - そうでない場合:
        - `revoke_and_ack` メッセージで応答しなければなりません。
    - そうでない (保留中のスプライストランザクションがない) 場合:
      - `funding_txid` が現在の資金調達トランザクションと一致しない `commitment_signed` は無視しなければなりません。
      - 現在の資金調達トランザクションに対する `commitment_signed` が欠けている場合:
        - `error` を送信してチャネルを失敗させなければなりません。
      - そうでない場合:
        - `revoke_and_ack` メッセージで応答しなければなりません。
  - それ以外の場合:
    - `revoke_and_ack` メッセージで応答しなければなりません。

#### 根拠

スパム的な更新を送ることにはほとんど意味がなく、バグを示唆します。

`num_htlcs` フィールドは冗長ですが、パケット長のチェックを完全に自己完結させるために使われます。

最近のメッセージ受信を要求するのは、ネットワークが信頼できないという現実への対応です。ノードは `commitment_signed` を送るまでピアがオフラインであることに気づかないかもしれません。`commitment_signed` が送信されると、送信者は HTLC に拘束されたとみなされ、出力 HTLC が完全に解決されるまで対応する受信 HTLC を失敗させられなくなります。

`htlc_signature` は、提供 HTLC のタイムアウトや受信 HTLC の消費時に、タイムロックの仕組みを暗黙に強制します。これにより、HTLC 出力にタイムロックを明示的に書き込むよりもスクリプトを小さくでき、手数料を抑えられます。

`option_anchors` は、HTLC トランザクションが他の入力・出力を追加して「自分の手数料を持ち込む」ことを許すため、署名フラグを修正したものを使います。

スプライシングでは追加の署名を送受信する必要があります。どのスプライストランザクションが新しいチャネル資金調達トランザクションになるか不明だからです。`start_batch` を用いて、保留中の各スプライストランザクションと現在の資金調達トランザクションに対する `commitment_signed` のバッチを送ります。`splice_locked` を送ったあとには、ピアがそれを受信する前に送り始めた古い `commitment_signed` のバッチが届く可能性がありますが、`funding_txid` でフィルタすることで安全に無視できます。

### 更新された状態への移行の完了：`revoke_and_ack`

`commitment_signed` の受信者が署名を確認し、有効な新しいコミットメントトランザクションを持っていることを確認したら、`revoke_and_ack` メッセージで前のコミットメントトランザクションのコミットメントプレイメージを返信します。

このメッセージは、`commitment_signed` の受領確認としても暗黙的に機能します。したがって、`commitment_signed` の送信者が、その `commitment_signed` の前に送信した保留中の更新を自分のコミットメントに適用するのに適したタイミングです。

キー導出の説明は [BOLT #3](03-transactions.md#key-derivation) にあります。

1. タイプ: 133 (`revoke_and_ack`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`32*byte`:`per_commitment_secret`]
   * [`point`:`next_per_commitment_point`]

#### 要件

送信ノード:
  - `per_commitment_secret` を、前のコミットメントトランザクションの鍵を生成するのに使用した秘密に設定しなければなりません。
  - `next_per_commitment_point` を、次のコミットメントトランザクション用の値に設定しなければなりません。
  - `commitment_signed` のバッチに応答する場合でも、単一の `revoke_and_ack` メッセージのみを送信しなければなりません。

受信ノード:
  - `per_commitment_secret` が有効な秘密鍵でない場合、または前の `per_commitment_point` を生成しない場合：
    - `error` を送信し、チャネルを失敗させなければなりません。
  - `per_commitment_secret` が [BOLT #3](03-transactions.md#per-commitment-secret-requirements) のプロトコルによって生成されていない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させてもかまいません。

ノード：
  - 古い（取り消された）コミットメントトランザクションをブロードキャストしてはなりません、
    - 注：これを行うと、他のノードがすべてのチャネル資金を差し押さえることができます。
  - コミットメントトランザクションに署名すべきではありません、失敗した接続のためにそれらをブロードキャストしようとしている場合を除いて、
    - 注：これは上記のリスクを軽減するためです。

### 手数料の更新：`update_fee`

`update_fee` メッセージは、ビットコイン手数料を支払うノードによって送信されます。すべての更新と同様に、最初に受信者のコミットメントトランザクションにコミットされ、その後（確認されると）送信者のコミットメントにコミットされます。HTLC とは異なり、`update_fee` は決して閉じられず、単に置き換えられます。

受信者が `update_fee` を受け取る前に新しい HTLC を追加できるため、競合の可能性があります。この状況下では、受信者によって `update_fee` が最終的に確認されると、送信者は自分のコミットメントトランザクションの手数料を負担できないかもしれません。この場合、手数料は [BOLT #3](03-transactions.md#fee-payment) に記載されているように、手数料率よりも低くなります。

手数料率から手数料を導出するために使用される正確な計算は、[BOLT #3](03-transactions.md#fee-calculation) に記載されています。

1. タイプ: 134 (`update_fee`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`u32`:`feerate_per_kw`]

#### 要件

Bitcoin 手数料を支払う責任があるノードは：
  - コミットメントトランザクションの迅速な処理に十分な（大幅な余裕を持った）現在の手数料率を確保するために `update_fee` を送信する *べき* です。

Bitcoin 手数料を支払う責任がないノードは：
  - `update_fee` を送信しては *いけません* 。

送信ノード:
  - `option_anchors` が交渉されていない場合:
    - `update_fee` が `feerate_per_kw` を増加させる場合:
      - 更新後の `feerate_per_kw` でリモートトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合:
        - `update_fee` を送信しなくてもよい (MAY NOT)。
        - チャネルを失敗させてもよい (MAY)。
      - 更新後の `feerate_per_kw` でローカルトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合:
        - `update_fee` を送信しなくてもよい (MAY NOT)。
        - チャネルを失敗させてもよい (MAY)。
  - スプライスが保留中の場合:
    - すべてのコミットメントトランザクションについて要件を満たすことを保証しなければなりません。

受信ノード:
  - `update_fee` がタイムリーな処理に対して低すぎる、または不当に大きい場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 送信者が Bitcoin 手数料を支払う責任がない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 送信者が、受信ノードの現在のコミットメントトランザクション上で新しい手数料率を負担できない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させるべきです。
      - ただし、このチェックを `update_fee` がコミットされるまで遅延させてもよいです。
    - `option_anchors` が交渉されていない場合:
      - `update_fee` が `feerate_per_kw` を増加させる場合:
        - 更新後の `feerate_per_kw` でリモートトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合:
          - チャネルを失敗させてもよい (MAY)。
      - 更新後の `feerate_per_kw` でローカルトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合:
          - チャネルを失敗させてもよい (MAY)。
  - スプライスが保留中の場合:
    - すべてのコミットメントトランザクションについて要件を満たすことを保証しなければなりません。

#### 根拠

ビットコインの手数料は、一方的なクローズを効果的にするために必要です。`option_anchors` を使用することで、旧来のコミットメント形式ほど `feerate_per_kw` が確認を保証するために重要ではなくなりますが、それでもメモリプールに入るために十分である必要があります（最低中継手数料とメモリプールの最低手数料を満たす必要があります）。

旧来のコミットメント形式では、ブロードキャストするノードが親の手数料を子が支払うことで効果的な手数料を増やす一般的な方法はありません。

手数料の変動や、トランザクションが将来使用される可能性を考慮すると、手数料を支払う側が旧来のコミットメントトランザクションに対して十分な余裕（例えば、予想される手数料の 5 倍）を持つことが良い考えです。ただし、手数料の見積もり方法が異なるため、正確な値は指定されていません。

現在、手数料は一方的です（チャネル作成を要求した側が常にコミットメントトランザクションの手数料を支払います）。そのため、手数料レベルを設定するのは簡単ですが、同じ手数料率が HTLC トランザクションにも適用されるため、受信ノードも手数料の妥当性を考慮する必要があります。

オンチェーン手数料が増加し、コミットメントに多くの HTLC が含まれていて、更新された手数料率でトリムされる場合、設定された `max_dust_htlc_exposure_msat` を超える可能性があります。チャネルを事前にクローズするかどうかは、ノードのポリシーに委ねられています。

スプライシングがサポートされている場合、コミットメントトランザクションが同時に複数存在することがあります。提案する変更はそれらすべてに対して妥当でなければなりません。

## メッセージの再送信

通信トランスポートは信頼性が低く、時折再確立が必要になることがあるため、トランスポートの設計はプロトコルから明示的に分離されています。

それでも、トランスポートは順序付けされ、信頼性があると仮定されています。再接続は、何が受信されたかについての疑念を生じさせるため、その時点で明示的な確認があります。

チャネルの確立とクローズの場合、メッセージには明示的な順序があるため、これは比較的簡単です。しかし、通常の操作中は、`commitment_signed` / `revoke_and_ack` の交換まで更新の確認が遅れるため、更新が受信されたと仮定することはできません。これはまた、受信ノードが `commitment_signed` を受け取った時点でのみ更新を保存する必要があることを意味します。

[BOLT #7](07-routing-gossip.md) で説明されているメッセージは特定のチャネルに依存しないことに注意してください。これらのメッセージの送信要件はそこでカバーされており、`init` の後に送信されること（すべてのメッセージと同様）以外は、ここでの要件に依存しません。

1. type: 136 (`channel_reestablish`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`next_commitment_number`]
   * [`u64`:`next_revocation_number`]
   * [`32*byte`:`your_last_per_commitment_secret`]
   * [`point`:`my_current_per_commitment_point`]

1. `tlv_stream`: `channel_reestablish_tlvs`
2. types:
    1. type: 1 (`next_funding`)
    2. data:
        * [`sha256`:`next_funding_txid`]
        * [`byte`:`retransmit_flags`]
    1. type: 5 (`my_current_funding_locked`)
    2. data:
        * [`sha256`:`my_current_funding_locked_txid`]
        * [`byte`:`retransmit_flags`]

`next_commitment_number`: コミットメント番号は、各コミットメントトランザクションに対する 48 ビットのインクリメントカウンタです。カウンタはチャネル内のピアごとに独立しており、0 から始まります。再確立時を除き、もう一方のノードに明示的に伝えられることはなく、それ以外は暗黙的です。

`next_funding.retransmit_flags` ビットフィールドは、再接続後に対応する `next_funding_txid` に対してピアが再送すべきメッセージを示します:

| ビット位置 | 名前                |
| ---------- | ------------------- |
| 0          | `commitment_signed` |

`my_current_funding_locked.retransmit_flags` ビットフィールドは、再接続後にピアに再送してほしいメッセージを示します:

| ビット位置 | 名前                       |
| ---------- | -------------------------- |
| 0          | `announcement_signatures`  |

### 要件

資金提供ノード：
  - 切断時：
    - 資金提供トランザクションをブロードキャストした場合：
      - 再接続のためにチャネルを記憶しなければなりません。
    - それ以外の場合：
      - 再接続のためにチャネルを記憶すべきではありません。

非資金提供ノード：
  - 切断時：
    - `funding_signed` メッセージを送信した場合：
      - 再接続のためにチャネルを記憶しなければなりません。
    - それ以外の場合：
      - 再接続のためにチャネルを記憶すべきではありません。

ノード：
  - 新しい暗号化されたトランスポートで以前のチャネルの継続を処理しなければなりません。
  - 切断時：
    - 反対側から送信された未コミットの更新をすべて逆転しなければなりません（つまり、`commitment_signed` を受信していないすべての `update_` で始まるメッセージ）。
      - 注：ノードはすでに `update_fulfill_htlc` からの `payment_preimage` 値を使用している可能性があるため、`update_fulfill_htlc` の効果は完全には逆転されません。
  - 再接続時：
    - チャネルがエラー状態にある場合：
      - エラーパケットを再送信し、そのチャネルの他のパケットを無視すべきです。
    - それ以外の場合：
      - 各チャネルに対して `channel_reestablish` を送信しなければなりません。
      - そのチャネルの他のメッセージを送信する前に、他のノードの `channel_reestablish` メッセージを受信するのを待たなければなりません。

送信ノード:

- `next_commitment_number` を、次に受信を期待する `commitment_signed` のコミットメント番号に設定しなければなりません。
- `next_revocation_number` を、次に受信を期待する `revoke_and_ack` メッセージのコミットメント番号に設定しなければなりません。
- `my_current_per_commitment_point` を有効なポイントに設定しなければなりません。
- `next_revocation_number` が 0 に等しい場合:
  - `your_last_per_commitment_secret` をすべてゼロに設定しなければなりません。
- それ以外の場合:
  - `your_last_per_commitment_secret` を、自身が受信した最後の `per_commitment_secret` に設定しなければなりません。
- interactive-tx 構築のために `commitment_signed` を送信したが、`tx_signatures` を受信していない場合:
  - `next_funding` TLV を含めなければなりません。
  - `next_funding_txid` をその interactive-tx の txid に設定しなければなりません。
  - その `next_funding_txid` に対する `commitment_signed` をまだ受信していない場合:
    - `retransmit_flags` の `commitment_signed` ビットを立てなければなりません。
- それ以外の場合:
  - `next_funding` TLV を含めてはなりません。
- `option_splice` が交渉されている場合:
  - 切断中にスプライストランザクションが許容できる深さに達した場合:
    - その最新のトランザクションの txid を入れた `my_current_funding_locked` を含めなければなりません。
  - そうでなく、すでに何らかのトランザクションについて `splice_locked` を送信していた場合:
    - 直近に送信した `splice_locked` の txid を入れた `my_current_funding_locked` を含めなければなりません。
  - そうでなく、すでに `channel_ready` を送信している場合:
    - チャネルの資金調達トランザクションの txid を入れた `my_current_funding_locked` を含めなければなりません。
  - そうでない (まだ `channel_ready` も `splice_locked` も送信していない) 場合:
    - `my_current_funding_locked` を含めてはなりません。
  - `my_current_funding_locked` を含める場合:
    - このチャネルで `announce_channel` が設定されている場合:
      - 該当トランザクションに対する `announcement_signatures` をまだ受信していない場合:
        - `retransmit_flags` の `announcement_signatures` ビットを `1` にしなければなりません。
    - そうでない場合:
      - `retransmit_flags` の `announcement_signatures` ビットを `0` にしなければなりません。

ノード:

- `next_commitment_number` が 0 の場合:
  - 直ちにチャネルを失敗させ、関連する最新コミットメントトランザクションをブロードキャストしなければなりません。
- 送信した `channel_reestablish` および受信した `channel_reestablish` の両方で `next_commitment_number` が 1 であり、いずれの `channel_reestablish` にもスプライストランザクションに対する `my_current_funding_locked` または `next_funding` が含まれていない場合:
  - `channel_ready` を再送しなければなりません。
- それ以外の場合:
  - `channel_ready` を再送してはなりませんが、異なる `short_channel_id` `alias` フィールドを持つ `channel_ready` を送信してもよいです。
- 再接続時:
  - 冗長な `channel_ready` を受信した場合は無視しなければなりません。
- `next_commitment_number` が受信ノードが最後に送信した `commitment_signed` メッセージのコミットメント番号と等しい場合:
  - 次の `commitment_signed` には同じコミットメント番号を再利用しなければなりません。
- それ以外の場合:
  - `next_commitment_number` が、受信ノードが次に送る `commitment_signed` のコミットメント番号と等しくない場合:
    - `error` を送信してチャネルを失敗させるべきです。
- `next_revocation_number` が受信ノードが送信した最後の `revoke_and_ack` のコミットメント番号と等しく、かつ受信ノードがまだ `closing_signed` を受信していない場合:
  - `revoke_and_ack` を再送しなければなりません。
  - 以前に再送が必要な `commitment_signed` を送信している場合:
    - `revoke_and_ack` と `commitment_signed` を、最初に送信したのと同じ相対順序で再送しなければなりません。
- それ以外の場合:
  - `next_revocation_number` が、受信ノードが送信した最後の `revoke_and_ack` のコミットメント番号より 1 大きくない場合:
    - `error` を送信してチャネルを失敗させるべきです。
  - `revoke_and_ack` を送信しておらず、かつ `next_revocation_number` が 0 と等しくない場合:
    - `error` を送信してチャネルを失敗させるべきです。

受信ノード:

- `my_current_per_commitment_point` は無視しなければなりませんが、有効なポイントであることを要求してもよいです。
- `next_revocation_number` が上記の期待値より大きく、かつ `your_last_per_commitment_secret` がその `next_revocation_number` から 1 を引いた値に対して正しい場合:
  - 自身のコミットメントトランザクションをブロードキャストしてはなりません。
  - ピアにチャネルを失敗させるよう要求する `error` を送信すべきです。
- それ以外の場合:
  - `your_last_per_commitment_secret` が期待値と一致しない場合:
    - `error` を送信してチャネルを失敗させるべきです。

受信ノード:

- `next_funding` TLV が設定されている場合:
  - `next_funding_txid` が最新の interactive-tx 資金調達トランザクションと一致する場合:
    - そのトランザクションに対する `tx_signatures` をまだ受信していない場合:
      - `retransmit_flags` の `commitment_signed` ビットが立っている場合:
        - そのトランザクションに対する `commitment_signed` を再送しなければなりません。
      - すでに `commitment_signed` を受信しており、[`tx_signatures` の要件](#the-tx_signatures-message) に従って先に署名すべき場合:
        - そのトランザクションに対する `tx_signatures` を送信しなければなりません。
    - すでにそのトランザクションに対する `tx_signatures` を受信している場合:
      - そのトランザクションに対する `tx_signatures` を送信しなければなりません。
  - 自身も `channel_reestablish` で `next_funding` を設定したが、値が一致しない場合:
    - `error` を送信してチャネルを失敗させなければなりません。
  - それ以外の場合:
    - 送信ノードがこのトランザクションを忘れてよいと知らせるため、`tx_abort` を送信しなければなりません。

受信ノード:

- 保留中のスプライストランザクションがあり、`my_current_funding_locked` がそのいずれかと一致しており、まだそのトランザクションに対する `splice_locked` を受信していない場合:
  - その `txid` に対する `splice_locked` を受信したかのように `my_current_funding_locked` を処理しなければなりません。
- `my_current_funding_locked` が含まれており、`retransmit_flags` の `announcement_signatures` ビットが立っている場合:
  - このチャネルで `announce_channel` が設定されており、対応するスプライストランザクションに対する `announcement_signatures` を送信できる状態の場合:
    - `announcement_signatures` を再送しなければなりません。

ノード：

- 以前に送信されたメッセージが失われたと仮定してはなりません。
  - 以前に `commitment_signed` メッセージを送信した場合：
    - 対応するコミットメントトランザクションが他方によっていつでもブロードキャストされる可能性がある場合を処理しなければなりません。
      - 注：これは特に、ノードが以前に送信した `update_` メッセージをそのまま再送信しない場合に重要です。
- 再接続時：
  - 以前に `shutdown` を送信した場合：
    - `shutdown` を再送信しなければなりません。

### 理論的根拠

上記の要件は、オープニングフェーズがほぼアトミックであることを保証します。完了しない場合は、再度開始します。唯一の例外は、`funding_signed` メッセージが送信されたが受信されなかった場合です。この場合、ファンダーはチャネルを忘れ、再接続時に新しいチャネルを開くと推測されます。一方、他のノードは、`channel_ready` を受信しないか、オンチェーンで資金調達トランザクションを確認しないため、最終的に元のチャネルを忘れることになります。

`error` には確認応答がないため、再接続が発生した場合には、再度切断する前に再送信するのが礼儀です。ただし、ノードがチャネルを完全に忘れてしまう場合もあるため、必須ではありません。

`closing_signed` も確認応答がないため、再接続時には再送信しなければなりません（ただし、再接続時には交渉が再開されるため、完全に同じ再送信である必要はありません）。`shutdown` の唯一の確認応答は `closing_signed` なので、どちらか一方を再送信する必要があります。

更新の取り扱いも同様にアトミックです。コミットが確認されない (または送られなかった) 場合、更新は再送されます。ただし同一である必要はなく、異なる順序であったり、別の手数料が伴ったり、追加するには古すぎる HTLC が欠けていたりしてもかまいません。同一性を要求してしまうと、送信のたびに送信者のディスク書き込みが必要になりますが、本仕様の方式は送受信される各 `commitment_signed` ごとに 1 回の永続化書き込みで済ませることを意図しています。ただし `commitment_signed` と `revoke_and_ack` を両方とも再送する必要がある場合は、両者の相対順序を保たなければなりません。さもないとチャネルクローズに繋がります。

`closing_signed` を受信した後に `revoke_and_ack` の再送信を要求されることは決してありません。これは、シャットダウンが完了したことを意味し、それはリモートノードが `revoke_and_ack` を受信した後にのみ発生するからです。

`next_commitment_number` は 1 から始まることに注意してください。コミットメント番号 0 はオープニング中に作成されます。
`next_revocation_number` は、コミットメント番号 1 の `commitment_signed` が送信され、コミットメント番号 0 の取り消しが受信されるまで 0 になります。

`channel_ready` は、通常の操作の開始によって暗黙的に確認されます。これは、`commitment_signed` が受信された後に開始されたことが知られているためです。したがって、`next_commitment_number` が 1 より大きいかどうかをテストします。

以前の草案では、資金提供者が「資金取引をブロードキャストした場合は覚えておかなければならない、そうでなければしてはならない」と主張していましたが、これは実際には不可能な要求でした。ノードは、まずディスクにコミットし、次にトランザクションをブロードキャストするか、その逆を行わなければなりません。新しい言語はこの現実を反映しています。ブロードキャストされていないチャネルを覚えておく方が、ブロードキャストされたチャネルを忘れるよりも確実に良いです。同様に、資金提供者の `funding_signed` メッセージについても、開かない（タイムアウトする）チャネルを覚えておく方が、資金提供者が開く間に資金提供者がそれを忘れてしまうよりも良いです。

ノードが何らかの理由で遅れてしまった場合（例えば、古いバックアップから復元された場合など）、遅れていることを検出することができます。遅れているノードは、自分の現在のコミットメントトランザクションをブロードキャストできないことを知っておく必要があります。これを行うと、リモートノードが取り消しプレイメージを知っていることを証明できるため、資金の全損につながります。遅れているノードから返される `error` は、他のノードが現在のコミットメントトランザクションをチェーンにドロップするように促すべきです。他のノードは、その `error` を待って、遅れているノードがまず状態を修正する機会を与えるべきです（例えば、異なるバックアップで再起動することによって）。

`next_funding` TLV は、ピアが interactive-tx 構築の署名ステップを最終化したり、いずれかのピアがそのトランザクションを署名せずに状態から既に削除している場合に安全に中止したりすることを可能にします。

`my_current_funding_locked` は `splice_locked` の送信と等価ですが、`channel_reestablish` の中でアトミックに処理されます (`splice_locked` メッセージの再送を要求する代わりです)。これはチャネル更新とのレース条件を避けるのに役立ちます (詳しい例は [この例](./bolt02/splicing-test.md#disconnection-with-concurrent-splice_locked) を参照してください)。`splice_locked` メッセージが切断中に失われた場合や、ピアの切断中にスプライストランザクションが許容できる深さに達した場合にも対応できます。また、最新のスプライストランザクションに対する `announcement_signatures` メッセージが切断前に届いていなかった場合に再送を要求することもできます。

# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
