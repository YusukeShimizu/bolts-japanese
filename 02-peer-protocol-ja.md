# BOLT #2: チャネル管理のためのピアプロトコル

ピアチャネルプロトコルには、確立、通常運用、閉鎖の三つのフェーズがあります。

# 目次

  * [チャネル](#channel)
    * [`channel_id` の定義](#definition-of-channel_id)
    * [インタラクティブなトランザクション構築](#interactive-transaction-construction)
      * [セットアップと用語](#set-up-and-vocabulary)
      * [手数料の責任](#fee-responsibility)
      * [概要](#overview)
      * [`tx_add_input` メッセージ](#the-tx_add_input-message)
      * [`tx_add_output` メッセージ](#the-tx_add_output-message)
      * [`tx_remove_input` と `tx_remove_output` メッセージ](#the-tx_remove_input-and-tx_remove_output-messages)
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
      * 資金署名の共有：`tx_signatures`](#sharing-funding-signatures-tx_signatures)
      * 手数料の引き上げ：`tx_init_rbf` と `tx_ack_rbf`](#fee-bumping-tx_init_rbf-and-tx_ack_rbf)
    * [チャネルの静止](#channel-quiescence)
    * [チャネルの閉鎖](#channel-close)
      * 閉鎖の開始：`shutdown`](#closing-initiation-shutdown)
      * 閉鎖交渉：`closing_signed`](#closing-negotiation-closing_signed)
    * [通常運用](#normal-operation)
      * HTLC の転送](#forwarding-htlcs)
      * `cltv_expiry_delta` の選択](#cltv_expiry_delta-selection)
      * HTLC の追加：`update_add_htlc`](#adding-an-htlc-update_add_htlc)
      * HTLC の削除：`update_fulfill_htlc`、`update_fail_htlc`、および `update_fail_malformed_htlc`](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * これまでの更新のコミット：`commitment_signed`](#committing-updates-so-far-commitment_signed)
      * 更新された状態への移行の完了：`revoke_and_ack`](#completing-the-transition-to-the-updated-state-revoke_and_ack)
      * 手数料の更新：`update_fee`](#updating-fees-update_fee)
    * メッセージの再送信：`channel_reestablish` メッセージ](#message-retransmission)
  * [著者](#authors)

# チャネル

## `channel_id` の定義

いくつかのメッセージは、チャネルを識別するために `channel_id` を使用します。これは、`funding_txid` と `funding_output_index` を組み合わせて、ビッグエンディアンの排他的論理和 (すなわち、`funding_output_index` が最後の 2 バイトを変更) を用いて資金調達トランザクションから導出されます。

チャネルの確立前には、ランダムなノンスである `temporary_channel_id` が使用されます。

異なるピアからの重複した `temporary_channel_id` が存在する可能性があることに注意してください。資金調達トランザクションが作成される前にチャネルをそのチャネル ID で参照する API は本質的に安全ではありません。資金調達が作成される前に交換された唯一のプロトコル提供のチャネル識別子は、(source_node_id, destination_node_id, temporary_channel_id) タプルです。資金調達トランザクションが確認される前にチャネルをそのチャネル ID で参照する API は永続的ではないことにも注意してください。資金調達出力に対応するスクリプト pubkey を知るまでは、重複するチャネル ID を防ぐものはありません。

### `channel_id`, v2

v2 プロトコルを使用して確立されたチャネルの場合、`channel_id` は `SHA256(lesser-revocation-basepoint || greater-revocation-basepoint)` です。ここで、lesser と greater はベースポイントの順序に基づいています。

`open_channel2` を送信する際、ピアの取り消しベースポイントは不明です。非イニシエータのためにゼロで埋められたベースポイントを使用して `temporary_channel_id` を計算する必要があります。

`accept_channel2` を送信する際、`open_channel2` からの `temporary_channel_id` を使用して、イニシエータがリクエストに対する応答を一致させることができるようにする必要があります。

#### 理論的根拠

取り消しベースポイントは、正しい操作のために両方のピアによって記憶される必要があります。最初のメッセージ交換後に知られるため、後続のメッセージで `temporary_channel_id` の必要性がなくなります。両側からの情報を混ぜることで、`channel_id` の衝突を避け、資金調達 txid への依存を排除します。

## インタラクティブなトランザクション構築

インタラクティブなトランザクション構築により、2 つのピアが共同でブロードキャスト用のトランザクションを構築することができます。このプロトコルは、デュアルファンドチャネル確立 (v2) の基盤です。

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

トランザクションの残りのバイトの手数料は、`tx_add_input` または `tx_add_output` を通じてそのインプットまたはアウトプットを提供したピアが、合意された `feerate` に基づいて負担します。

### 概要

*イニシエータ* は `tx_add_input` を用いてインタラクティブなトランザクション構築プロトコルを開始します。*非イニシエータ* は `tx_add_input`、`tx_add_output`、`tx_remove_input`、`tx_remove_output`、または `tx_complete` のいずれかで応答します。プロトコルは、両方のノードが連続した `tx_complete` を送受信するまで、インタラクティブなトランザクションプロトコルメッセージの同期交換を続けます。これはターンベースのプロトコルです。

ピアが連続した `tx_complete` を交換すると、インタラクティブなトランザクション構築プロトコルは終了したと見なされます。両方のピアはトランザクションを構築し、エラーが発見された場合は交渉を失敗させるべきです。

このプロトコルは、並行して複数のパーティが単一のトランザクションを共同で構築できるように明示的に設計されています。これにより、単一のトランザクションで複数のチャネルを開く能力が保持されます。`serial_id` は一般的にランダムに選ばれますが、すべてのピアセッションで一貫したトランザクション順序を維持するために、受信した `serial_id` を他のピアに転送する際に再利用し、必要に応じてパリティ要件を満たすために下位ビットを反転させるのが最も簡単です。

以下はいくつかの例となるやり取りです。

#### *initiator* のみ

A は *initiator* で、2 つのインプットと 1 つのアウトプット (ファンディングアウトプット) を持っています。B は *non-initiator* で、何も提供しません。

```
    +-------+                       +-------+
    |       |--(1)- tx_add_input -->|       |
    |       |<-(2)- tx_complete ----|       |
    |       |--(3)- tx_add_input -->|       |
    |   A   |<-(4)- tx_complete ----|   B   |
    |       |--(5)- tx_add_output ->|       |
    |       |<-(6)- tx_complete ----|       |
    |       |--(7)- tx_complete --->|       |
    +-------+                       +-------+
```

#### *initiator* と *non-initiator*

A は *initiator* で、2 つのインプットと 1 つのアウトプットを提供し、その後それを削除します。B は *non-initiator* で、1 つのインプットと 1 つのアウトプットを提供しますが、A が 2 番目のインプットを追加するまで待ちます。

A が 2 番目のインプットを送信しない場合、交渉は B の貢献なしに終了します。

```
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
```

### `tx_add_input` メッセージ

このメッセージはトランザクションインプットを含みます。

1. タイプ: 66 (`tx_add_input`)
2. データ:
    * [`channel_id`:`channel_id`]
    * [`u64`:`serial_id`]
    * [`u16`:`prevtx_len`]
    * [`prevtx_len*byte`:`prevtx`]
    * [`u32`:`prevtx_vout`]
    * [`u32`:`sequence`]

#### 要件

送信ノードは以下を行います：
  - 送信されたすべてのインプットをトランザクションに追加しなければなりません
  - 現在トランザクションに追加されている各インプットに対してユニークな `serial_id` を使用しなければなりません
  - `sequence` を 4294967293 (`0xFFFFFFFD`) 以下に設定しなければなりません
  - ピアから受信したインプットを再送信してはなりません
  - *initiator* の場合：
    - 偶数の `serial_id` を送信しなければなりません
  - *non-initiator* の場合：
    - 奇数の `serial_id` を送信しなければなりません

受信ノード：

- 受信したすべての入力をトランザクションに追加しなければなりません
- 以下の場合、交渉を失敗させなければなりません：
  - `sequence` が `0xFFFFFFFE` または `0xFFFFFFFF` に設定されている
  - `prevtx` と `prevtx_vout` が以前に追加された（削除されていない）入力と同一である
  - `prevtx` が有効なトランザクションでない
  - `prevtx_vout` が `prevtx` の出力数以上である
  - `prevtx` の `prevtx_vout` 出力の `scriptPubKey` が、1 バイトのプッシュオペコード（数値 `0` から `16`）に続いて 2 バイトから 40 バイトのデータプッシュでない
  - `serial_id` がすでにトランザクションに含まれている
  - `serial_id` のパリティが間違っている
  - この交渉中に 4096 の `tx_add_input` メッセージを受信した場合

#### 理論的根拠

各ノードはトランザクション入力のセットを知っていなければなりません。*非イニシエータ* はこのメッセージを省略してもかまいません。

`serial_id` はこの入力を一意に識別するランダムに選ばれた番号です。構築されたトランザクションの入力は `serial_id` によってソートされなければなりません。

`prevtx` はこの入力が消費する出力を含むシリアライズされたトランザクションです。入力が改ざんされていないことを確認するために使用されます。

`prevtx_vout` は消費される出力のインデックスです。

`sequence` はこの入力のシーケンス番号です：置換可能性を示さなければならず、オンチェーンのフィンガープリンティングを避けるために実装間で同じ値を使用するべきです。

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

1. subtype: `witness`
2. data:
    * [`u16`:`len`]
    * [`len*byte`:`witness_data`]

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

送信者：
  - `feerate` を以前に構築されたトランザクションの `feerate` の 25/24 倍以上に設定しなければなりません（切り捨て）。
  - トランザクションのファンディング出力に寄与する場合：
    - `funding_output_contribution` を設定しなければなりません。
  - 受信ノードに確認済みのインプットのみを使用することを要求する場合：
    - `require_confirmed_inputs` を設定しなければなりません。

受信者：
  - `tx_abort` または `tx_ack_rbf` で応答しなければなりません。
  - 以下の場合、`tx_abort` で応答しなければなりません：
    - `feerate` が最後に成功したトランザクションの `feerate` の 25/24 倍以上でない場合
  - 任意の理由で `tx_abort` を送信してもかまいません。
  - 以下の場合、交渉を失敗させなければなりません：
    - `require_confirmed_inputs` が設定されているが、確認済みのインプットを提供できない場合

#### 理論的根拠

`feerate` はこのトランザクションが支払う手数料率です。最後に使用された `feerate` よりも少なくとも 1/24 高くなければならず、進捗を確保するために最も近いサトシに切り捨てられます。

例えば、最後の `feerate` が 520 だった場合、次に送信される `feerate` は 541 でなければなりません（520 * 25 / 24 = 541.667、切り捨てて 541）。

RBF 試行の途中で以前のトランザクションが確認された場合、その試行は放棄しなければなりません。

`funding_output_contribution` は、このピアがトランザクションのファンディング出力に寄与するサトシの量です。この出力がある場合に限ります。以前に完了したトランザクションでの寄与とは異なる場合があります。省略された場合、送信者はファンディング出力に寄与していません。

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

送信者：
  - トランザクションの資金出力に寄与する場合：
    - `funding_output_contribution` を設定しなければなりません
  - 受信ノードが確認済みのインプットのみを使用することを要求する場合：
    - `require_confirmed_inputs` を設定しなければなりません

受信者：
  - `tx_abort` または `tx_add_input` メッセージで応答し、インタラクティブなトランザクション協力プロトコルを再開しなければなりません。
  - 交渉を失敗させなければなりません、もし：
    - `require_confirmed_inputs` が設定されているが、確認済みのインプットを提供できない場合

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
    - 交渉されたトランザクションのインプットが消費されるまで、チャネルを忘れてはなりません。
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

```
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
```

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
  - `open_channel_tlvs` を含める場合：
    - `upfront_shutdown_script` を含めなければなりません。
  - `option_channel_type` が交渉された場合：
    - `channel_type` を設定しなければなりません。
  - `channel_type` を含める場合：
    - 希望するタイプを表す定義されたタイプに設定しなければなりません。
    - チャネルタイプを表すために可能な限り小さいビットマップを使用しなければなりません。
    - 交渉されていない機能を含むタイプに設定してはなりません。
    - `announce_channel` が `true` (つまり `0` ではない) の場合：
      - `option_scid_alias` ビットが設定された `channel_type` を送信してはなりません。

送信ノードは以下を行う「べき」です (SHOULD)：
  - 受信者の不正行為があった場合に、送信者がコミットメントトランザクションの出力を不可逆的に使用できるようにするために、`to_self_delay` を十分に設定します。
  - トランザクションが即座にブロックに含まれると予想されるレート以上に `feerate_per_kw` を設定します。
  - コミットメントトランザクションが Bitcoin ネットワークを通じて伝播できるように、`dust_limit_satoshis` を十分な値に設定します。
  - このピアから受け入れる最小値 HTLC に `htlc_minimum_msat` を設定します。

受信ノードは以下を行わなければなりません (MUST)：
  - `channel_flags` の未定義ビットを無視します。
  - 前の `open_channel` を受信した後、`funding_created` メッセージを受信する前に接続が再確立された場合：
    - 新しい `open_channel` メッセージを受け入れます。
    - 前の `open_channel` メッセージを破棄します。
  - `option_dual_fund` が交渉された場合：
    - チャネルを失敗させます。

受信ノードは以下の場合にチャネルを失敗させる「かもしれません」(MAY)：
  - `option_channel_type` が交渉されたが、メッセージに `channel_type` が含まれていない。
  - `announce_channel` が `false` ( `0` ) であるが、チャネルを公に発表したい。
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

受信者は大きな `dust_limit_satoshis` を受け入れるべきではありません。これは、ピアが多くのダスト HTLC を含むコミットメントを公開し、それが実質的にマイナー手数料となるグリーフィング攻撃に利用される可能性があるためです。

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
  - `channel_type` が `open_channel` と `accept_channel` の両方に存在する場合:
    - これは `channel_type` です (上記で必要とされるように等しい必要があります)
  - そうでない場合:
    - `option_anchors` が交渉された場合:
      - `channel_type` は `option_anchors` と `option_static_remotekey` (ビット 22 と 12) です
    - そうでない場合:
      - `channel_type` は `option_static_remotekey` (ビット 12) です
  - すべてのコミットメントトランザクションにその `channel_type` を使用しなければなりません。

送信者は以下を設定しなければなりません:
  - `funding_created` メッセージからの `funding_txid` と `funding_output_index` の排他的論理和による `channel_id`。
  - 初期コミットメントトランザクションのために、その `funding_pubkey` を使用して、[BOLT #3](03-transactions.md#commitment-transaction) で定義された有効な署名に `signature` を設定。

受信者:
  - `signature` が間違っているか、LOW-S 標準ルール<sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup>に準拠していない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - 有効な `funding_signed` を受け取る前に資金調達トランザクションをブロードキャストしてはなりません。
  - 有効な `funding_signed` を受け取った場合:
    - 資金調達トランザクションをブロードキャストすることを推奨します。

#### 根拠

`option_static_remotekey` または `option_anchors` は、コミットメントトランザクションを初めて生成する際に決定します。現在の接続のために `init` メッセージ交換で伝達されたフィーチャービットが、チャネルの全期間にわたるチャネルコミットメント形式を決定します。後の再接続でこのパラメータが交渉されなくても、このチャネルは `option_static_remotekey` または `option_anchors` を使い続けます。ダウングレードはサポートしていません。

`option_anchors` は `option_static_remotekey` よりも優れていると考えられ、複数が交渉された場合は優れた方が優先されます。

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
このプロトコルは、`accept_channel2` ピア (受け入れ側/非イニシエータ) がインタラクティブなトランザクション構築プロトコルを通じて資金提供トランザクションに入力を提供できるように、以前のプロトコルを変更します。


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

ノードが `option_dual_fund` を交渉した場合：
  - 開始ノード：
    - `open_channel` を送信してはなりません

送信ノード：
  - `funding_feerate_perkw` をこのトランザクションの手数料率に設定しなければなりません
  - 受信ノードに確認済みのインプットのみを使用することを要求する場合：
    - `require_confirmed_inputs` を設定しなければなりません

受信ノード：
  - 交渉を失敗させてもよい場合：
    - `locktime` が受け入れられない
    - `funding_feerate_perkw` が受け入れられない
  - 交渉を失敗させなければならない場合：
    - `require_confirmed_inputs` が設定されているが、確認済みのインプットを提供できない

#### 根拠

`temporary_channel_id` は、ピアの取り消しベースポイントのゼロ化されたベースポイントを使用して導出しなければなりません。これにより、*受け入れ者* の取り消しベースポイントが知られる前に、ピアがチャネルに割り当て可能なエラーを返すことができます。

`funding_feerate_perkw` は、開設ノードが資金調達トランザクションのために支払う手数料率を 1000 ウェイトあたりのサトシで示します。詳細は [BOLT-3, Appendix F](03-transactions.md#appendix-f-dual-funded-transaction-test-vectors) に記載されています。

`locktime` は資金調達トランザクションのロックタイムです。

受信ノードは、`locktime` または `funding_feerate_perkw` が許容範囲外と見なされる場合、交渉を失敗させてもよいです。しかし、*受け入れ者* がチャネルの資金調達に参加せずにチャネル開設を進めることを許可することが推奨されます。

`open_channel` の `channel_reserve_satoshi` は省略されています。代わりに、チャネルリザーブは総チャネル残高（`open_channel2`.`funding_satoshis` + `accept_channel2`.`funding_satoshis`）の 1% に固定され、最も近いサトシ単位に切り捨てられるか、`dust_limit_satoshis` のいずれか大きい方になります。

`push_msat` が省略されていることに注意してください。

`second_per_commitment_point` は、実装の便宜のためにここ（および `channel_ready` で）送信されます。

送信ノードは、他の参加者が確認済みのインプットのみを使用するよう要求することがあります。これにより、送信ノードが他の参加者のインプットの未確認の低い手数料率の祖先の手数料を支払うことがないようにします。

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

受け入れるノード:
  - `open_channel2` メッセージの `temporary_channel_id` を使用しなければなりません。
  - `funding_satoshis` の値をゼロで応答してもかまいません。
  - 開始ノードに確認済みのインプットのみを使用するよう要求する場合:
    - `require_confirmed_inputs` を設定しなければなりません。

受信ノード:
  - 交渉を失敗させなければなりません:
    - `require_confirmed_inputs` が設定されているが、確認済みのインプットを提供できない場合

#### 理論

`funding_satoshis` は、*受け入れる側* がチャネルの資金調達トランザクションに貢献するビットコインのサトシ単位の量です。

`accept_channel` の `channel_reserve_satoshi` は省略されていることに注意してください。その代わりに、チャネルリザーブはチャネル全体の残高 (`open_channel2`.`funding_satoshis` + `accept_channel2`.`funding_satoshis`) の 1% に固定され、最も近いサトシ単位に切り下げられるか、`dust_limit_satoshis` のいずれか大きい方になります。

### 資金構成

チャネル確立 v2 の資金構成は、[インタラクティブトランザクション構築](#interactive-transaction-construction) プロトコルを利用し、以下の追加の注意点があります。

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

資金調達トランザクションがブロードキャストされた後、チャネルの確認を早めるために、より多くの手数料を支払うトランザクションに置き換えることができます。

#### 要件

`tx_init_rbf` の送信者：
  - *イニシエータ* でなければなりません。
  - `channel_ready` メッセージを送信または受信してはなりません。

受信者：
  - すでに `channel_ready` を送信または受信している場合、交渉を失敗させなければなりません。
  - 任意の理由で交渉を失敗させてもかまいません。

#### 理論的根拠

RBF 試行の途中で有効な `channel_ready` メッセージを受信した場合、その試行は放棄されなければなりません。

ピアは、`tx_init_rbf.funding_output_contribution` と `tx_ack_rbf.funding_output_contribution` において、`open_channel2` および `accept_channel2` で送信された金額とは異なる値を使用できます。資金調達出力にどれだけコミットしたいかを変更することが許可されています。

ピアは、大きな手数料率の変化により RBF 交渉を失敗させるのではなく、`sats` をゼロに設定し、チャネル資金調達へのさらなる参加を辞退することが推奨されます。貢献しないことで、無償で受信流動性を得ることができるかもしれません。

## チャネルの静止化

さまざまな基本的な変更、特にプロトコルのアップグレードは、両方のコミットメントトランザクションが一致し、保留中の更新がないチャネルで最も簡単に行えます。「何か基本的なことが進行中である」ことを示すことでチャネルを静止化するプロトコルを定義します。

### `stfu`

1. タイプ：2 (`stfu`)
2. データ：
    * [`channel_id`:`channel_id`]
    * [`u8`:`initiator`]

### 要件

`stfu` の送信者：

- `option_quiesce` が交渉されていない限り、`stfu` を送信してはなりません。
- 送信者の htlc 追加、htlc 削除、または手数料の更新がどちらのピアでも保留中の場合、`stfu` を送信してはなりません。
- `stfu` を二度送信してはなりません。
- `stfu` に返信している場合：
  - `initiator` を 0 に設定しなければなりません。
- それ以外の場合：
  - `initiator` を 1 に設定しなければなりません。
- `channel_id` を静止させるチャネルの ID に設定しなければなりません。
- チャネルが静止状態であると見なさなければなりません。
- `stfu` の後に更新メッセージを送信してはなりません。

`stfu` の受信者：

- `stfu` を送信した場合：
  - チャネルが静止状態であると見なさなければなりません。
- それ以外の場合：
  - これ以上更新メッセージを送信すべきではありません。
  - 可能になったら `stfu` で返信しなければなりません。

両方のノード：

- HTLC が保留中の場合、静止状態から 60 秒後に切断しなければなりません。

切断時：

- チャネルはもはや静止状態と見なされません。

依存プロトコル：

- 静止状態を終了させるすべての状態を指定しなければなりません。
  - 注記：これは静止状態に依存するプロトコルのバッチ実行を防ぎます。

### 理論的根拠

通常の使用法は、更新の送信を停止し、現在のすべての更新が両方のピアによって確認されるのを待ってから、静止状態を開始することです。いくつかのプロトコルでは、イニシエータを選ぶことが重要なので、このフラグが送信されます。

両側が同時に `stfu` を送信した場合、両方とも `initiator` を `1` に設定します。この場合、「イニシエータ」は任意にチャネルの資金提供者（`open_channel` の送信者）と見なされます。静止状態の効果は、一方が他方に返信した場合とまったく同じです。

依存プロトコルは、チャネルのトラフィックを再開するために切断が必要になるのを防ぐために、終了条件を指定しなければなりません。明示的な再開メッセージは、[検討されましたが却下されました](https://github.com/rustyrussell/lightning-rfc/pull/14)。これは、チャネル状態の双方向の合意を維持するのが著しく複雑になる多くのエッジケースを導入するためです。これにより、同じ静止セッションで複数の下流プロトコルをバッチ処理することが不可能であるという派生的な特性が導入されます。

## チャネルクローズ

ノードは接続の相互クローズを交渉できます。これは一方的なクローズとは異なり、資金に即座にアクセスでき、手数料を低く抑えて交渉することが可能です。

クローズは二段階で行われます：
1. 一方がチャネルをクリアしたいことを示し（新しい HTLC を受け入れない）
2. すべての HTLC が解決された後、最終的なチャネルクローズの交渉が始まります。

        +-------+                              +-------+
        |       |--(1)-----  shutdown  ------->|       |
        |       |<-(2)-----  shutdown  --------|       |
        |       |                              |       |
        |       | <complete all pending HTLCs> |       |
        |   A   |                 ...          |   B   |
        |       |                              |       |
        |       |--(3)-- closing_signed  F1--->|       |
        |       |<-(4)-- closing_signed  F2----|       |
        |       |              ...             |       |
        |       |--(?)-- closing_signed  Fn--->|       |
        |       |<-(?)-- closing_signed  Fn----|       |
        +-------+                              +-------+

### クローズの開始: `shutdown`

どちらのノードも（または両方）`shutdown` メッセージを送信してクローズを開始できます。これには支払いを希望する `scriptpubkey` が含まれます。

1. タイプ: 38 (`shutdown`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`scriptpubkey`]

#### 要件

送信ノード：
  - `funding_created`（ファンダーの場合）または `funding_signed`（ファンディーの場合）を送信していない場合：
    - `shutdown` を送信してはなりません。
  - `channel_ready` の前、つまりファンディングトランザクションが `minimum_depth` に達する前に `shutdown` を送信してもかまいません。
  - 受信ノードのコミットメントトランザクションに保留中の更新がある場合：
    - `shutdown` を送信してはなりません。
  - 複数の `shutdown` メッセージを送信してはなりません。
  - `shutdown` の後に `update_add_htlc` を送信してはなりません。
  - どちらのコミットメントトランザクションにも HTLC が残っていない場合（ダスト HTLC を含む）で、どちらの側にも送信する保留中の `revoke_and_ack` がない場合：
    - その時点以降に `update` メッセージを送信してはなりません。
  - `shutdown` を送信した後に追加された HTLC のルートを失敗させるべきです。
  - `open_channel` または `accept_channel` で非ゼロ長の `shutdown_scriptpubkey` を送信した場合：
    - `scriptpubkey` に同じ値を送信しなければなりません。
  - `scriptpubkey` を次の形式のいずれかに設定しなければなりません：


1. `OP_0` `20` 20バイト (バージョン 0 の witness pubkey hash への支払い)、または
2. `OP_0` `32` 32バイト (バージョン 0 の witness script hash への支払い)、または
3. `option_shutdown_anysegwit` が交渉された場合に限り：
   * `OP_1` から `OP_16` までのいずれかに続いて、2 から 40 バイトの単一プッシュ
     (witness プログラムバージョン 1 から 16)

受信ノード：
- `funding_signed` (資金提供者の場合) または `funding_created` (資金受領者の場合) を受信していない場合：
  - `error` を送信してチャネルを失敗させるべきです。
- `scriptpubkey` が上記の形式のいずれでもない場合：
  - `warning` を送信するべきです。
- まだ `channel_ready` を送信していない場合：
  - `shutdown` メッセージに `shutdown` で応答してもかまいません。
- ピアに未解決の更新がない場合、すでに `shutdown` を送信していない限り：
  - `shutdown` メッセージに `shutdown` で応答しなければなりません。
- 両方のノードが `option_upfront_shutdown_script` 機能を広告し、受信ノードが `open_channel` または `accept_channel` で非ゼロ長の `shutdown_scriptpubkey` を受信し、その `shutdown_scriptpubkey` が `scriptpubkey` と等しくない場合：
  - `warning` を送信してもかまいません。
  - 接続を失敗させなければなりません。

#### 理論的根拠

シャットダウンが開始されるときにチャネルの状態が常に「クリーン」(保留中の変更がない) である場合、そうでなかった場合の振る舞いの問題を避けることができます：送信者は常に最初に `commitment_signed` を送信します。

シャットダウンは終了の意図を示唆するため、新しい HTLC が追加または受け入れられることはありません。HTLC がクリアされると、取り消しが必要なコミットメントはなくなり、すべての更新が両方のコミットメントトランザクションに含まれるため、ピアはすぐに閉鎖交渉を開始できます。そのため、コミットメントトランザクションへのさらなる更新を禁止します (特に、`update_fee` はそうでなければ可能です)。ただし、コミットメントトランザクションに HTLC がある場合、イニシエータは未解決の HTLC がタイムアウトする可能性があるため、手数料率を上げることが望ましいかもしれません。

`scriptpubkey` の形式には、Bitcoin ネットワークによって受け入れられる標準的な segwit 形式のみが含まれており、結果として得られるトランザクションがマイナーに伝播することを保証します。ただし、古いノードは非 segwit スクリプトを送信することがあり、これは後方互換性のために受け入れられるかもしれません (この出力がダストリレー要件を満たさない場合は強制的に閉鎖するという注意付きで)。

`option_upfront_shutdown_script` 機能は、ノードが何らかの形で侵害された場合に `shutdown_scriptpubkey` に事前にコミットしたいという意図を示します。これは弱いコミットメントです（悪意のある実装はこのような仕様を無視しがちです）が、`scriptpubkey` を変更するために受信ノードの協力を必要とすることで、セキュリティを段階的に向上させます。

`shutdown` 応答の要件は、ノードが返信する前に未処理の変更をコミットするために `commitment_signed` を送信することを意味します。ただし、理論的には再接続することも可能であり、その場合は未コミットのすべての変更が単に消去されます。

### クローズ交渉: `closing_signed`

シャットダウンが完了し、チャネルが HTLC を持たず、取り消しが必要なコミットメントがなく、すべての更新が両方のコミットメントに含まれている場合、最終的な現在のコミットメントトランザクションには HTLC がなく、クローズ手数料の交渉が始まります。資金提供者は公正だと思う手数料を選び、`shutdown` メッセージの `scriptpubkey` フィールド（および選択した手数料）でクローズトランザクションに署名し、署名を送信します。その後、他のノードも同様に返信し、公正だと思う手数料を使用します。このやり取りは、両者が同じ手数料に合意するか、一方がチャネルを失敗させるまで続きます。

現代の方法では、資金提供者が許容可能な手数料範囲を送信し、非資金提供者がこの範囲内で手数料を選ぶ必要があります。非資金提供者が同じ値を選んだ場合、交渉は 2 つのメッセージで完了します。そうでない場合、資金提供者は同じ値で返信し、3 つのメッセージで完了します。

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

資金提供ノード：
  - `shutdown` が受信され、かつどちらのコミットメントトランザクションにも HTLC が残っていない場合：
    - `closing_signed` メッセージを送信する *べき* です。

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

- 閉鎖トランザクションの出力の一つが、その `scriptpubkey` のダスト制限を下回る場合（[BOLT 3](03-transactions.md#dust-limits)を参照）：
  - チャネルを失敗させなければなりません

#### 理論的根拠

`fee_range` が提供されていない場合、「厳密に間にある」という要件は、たとえ 1 サトシずつであっても前進が行われることを保証します。状態を保持せず、切断と再接続の間に手数料が変動した場合のコーナーケースを処理するために、再接続時に交渉が再開されます。

閉鎖トランザクションが遅延するリスクは限られていますが、非常に早くブロードキャストされるため、迅速な処理のためにプレミアムを支払う理由は通常ありません。

非資金提供者は手数料を支払わないため、最大手数料率を持つ理由はありません。ただし、トランザクションが伝播することを保証するために、最小手数料率を持ちたいかもしれません。必要に応じて後で CPFP を使用して確認を早めることができるため、その最小値は低くするべきです。

閉鎖トランザクションがビットコインのデフォルトのリレーポリシーを満たさないことがあるかもしれません（例えば、546 サトシ未満の出力に対して非セグウィットのシャットダウンスクリプトを使用する場合、`dust_limit_satoshis` が 546 サトシ未満である場合に可能です）。その場合、資金が危険にさらされることはありませんが、閉鎖トランザクションがマイナーに届くことはほぼないため、チャネルを強制的に閉鎖しなければなりません。

## 通常の操作

両方のノードが `channel_ready`（およびオプションで [`announcement_signatures`](07-routing-gossip.md#the-announcement_signatures-message)）を交換すると、チャネルはハッシュタイムロック契約を介して支払いを行うために使用できます。

変更はバッチで送信されます。`commitment_signed` メッセージの前に 1 つ以上の `update_` メッセージが送信されます。以下の図のように：

```
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
```

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

したがって、最悪のケースは `3R+2G+2S` です。ただし、`R` が少なくとも 1 であると仮定します。`R` が 2 以上の場合、他のノードがすべての再編成に勝つ可能性は低いです。高い手数料が使用されるため (HTLC 消費はほぼ任意の手数料を使用できます)、通常の運用中は `S` は小さいはずです。ただし、ブロック時間が不規則であるため、空のブロックが発生し続け、手数料が大きく変動する可能性があり、HTLC トランザクションの手数料を引き上げることはできないため、`S=12` を最低限と考えるべきです。`S` は攻撃下で最も変動する可能性のあるパラメータでもあるため、無視できない金額が危険にさらされている場合は、より高い値が望ましいかもしれません。猶予期間 `G` は低く (1 または 2)、ノードはできるだけ早くタイムアウトまたは完了する必要があります。ただし、`G` が低すぎると、ネットワーク遅延による不要なチャネル閉鎖のリスクが高まります。

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
    - `path_key` を設定しなければなりません（[ルートブラインディング](04-onion-routing.md#route-blinding)を参照）。


`id` は更新が完了した後（つまり `revoke_and_ack` が受信された後）に 0 にリセットしてはいけません。代わりに、継続してインクリメントし続ける必要があります。

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
    - `path_key`（指定されている場合）を使用しなければなりません。
    - `payment_hash` を `associated_data` として使用しなければなりません。
  - 復号に失敗した場合、結果が有効な `payload` TLV でない場合、または未知の偶数型を含む場合：
    - [Failure Messages](04-onion-routing.md#failure-messages) で詳述されているようにエラーで応答しなければなりません。
  - それ以外の場合：
    - [Payload Format](04-onion-routing.md#payload-format) の `payload` のリーダーに対する要件に従わなければなりません。

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

### これまでの更新のコミット: `commitment_signed`

ノードがリモートコミットメントに対して変更を持っている場合、それを適用し、結果として得られるトランザクションに署名し（[BOLT #3](03-transactions.md)で定義されています）、`commitment_signed` メッセージを送信できます。

1. タイプ: 132 (`commitment_signed`)
2. データ:
   * [`channel_id`:`channel_id`]
   * [`signature`:`signature`]
   * [`u16`:`num_htlcs`]
   * [`num_htlcs*signature`:`htlc_signature`]

#### 要件

送信ノード:
  - 更新を含まない `commitment_signed` メッセージを送信してはなりません。
  - 手数料のみを変更する `commitment_signed` メッセージを送信してもかまいません。
  - 新しい取り消し番号以外にコミットメントトランザクションを変更しない `commitment_signed` メッセージを送信してもかまいません（ダスト、同一 HTLC の置換、または重要でない複数の手数料変更による）。
  - コミットメントトランザクションの順序に対応するすべての HTLC トランザクションに対して 1 つの `htlc_signature` を含めなければなりません（[BOLT #3](03-transactions.md#transaction-input-and-output-ordering)を参照）。
  - 最近リモートノードからメッセージを受信していない場合:
      - `ping` を使用し、`commitment_signed` を送信する前に返信 `pong` を待つべきです。

受信ノードは以下のように動作します：

- 保留中のすべての更新が適用された後：
  - `signature` がローカルのコミットメントトランザクションに対して無効であるか、LOW-S 標準ルール <sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup> に準拠していない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
  - `num_htlcs` がローカルコミットメントトランザクションの HTLC 出力の数と等しくない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
- いずれかの `htlc_signature` が対応する HTLC トランザクションに対して無効であるか、LOW-S 標準ルール <sup>[LOWS](https://github.com/bitcoin/bitcoin/pull/6769)</sup> に準拠していない場合：
  - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させなければなりません。
- `revoke_and_ack` メッセージで応答しなければなりません。

#### 理論的根拠

スパム更新を提供することにはほとんど意味がありません。それはバグを示唆します。

`num_htlcs` フィールドは冗長ですが、パケット長のチェックを完全に自己完結させます。

最近のメッセージを要求することを推奨するのは、ネットワークが信頼できないという現実を認識しているからです。ノードは、`commitment_signed` を送信するまでピアがオフラインであることに気づかないかもしれません。`commitment_signed` が送信されると、送信者はそれらの HTLC に拘束されていると見なされ、出力 HTLC が完全に解決されるまで関連する受信 HTLC を失敗させることはできません。

`htlc_signature` は、提供された HTLC がタイムアウトした場合や受信した HTLC が消費された場合に、タイムロックメカニズムを暗黙的に強制します。これは、HTLC 出力にタイムロックを明示的に記載するよりも小さなスクリプトを作成することで手数料を削減するために行われます。

`option_anchors` は、HTLC トランザクションが他の入力と出力を添付することで「独自の手数料を持ち込む」ことを可能にし、したがって修正された署名フラグを使用します。

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

送信ノード：
  - `per_commitment_secret` を、前のコミットメントトランザクションのキーを生成するために使用した秘密に設定しなければなりません。
  - `next_per_commitment_point` を、次のコミットメントトランザクションの値に設定しなければなりません。

受信ノード：
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

送信ノードは：
  - `option_anchors` が交渉されていない場合：
    - `update_fee` が `feerate_per_kw` を増加させる場合：
      - 更新された `feerate_per_kw` でのリモートトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合：
        - `update_fee` を送信しない *かもしれません*
        - チャネルを失敗させる *かもしれません*
      - 更新された `feerate_per_kw` でのローカルトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合：
        - `update_fee` を送信しない *かもしれません*
        - チャネルを失敗させる *かもしれません*

受信ノードは：
  - `update_fee` が迅速な処理に対して低すぎる場合、または不当に大きい場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させる *必要があります* 。
  - 送信者が Bitcoin 手数料を支払う責任がない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させる *必要があります* 。
  - 送信者が受信ノードの現在のコミットメントトランザクションで新しい手数料率を負担できない場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させる *べき* です。
      - ただし、このチェックを `update_fee` がコミットされるまで遅らせる *かもしれません* 。
    - `option_anchors` が交渉されていない場合：
      - `update_fee` が `feerate_per_kw` を増加させる場合：
        - 更新された `feerate_per_kw` でのリモートトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合：
          - チャネルを失敗させる *かもしれません*
      - 更新された `feerate_per_kw` でのローカルトランザクションのダストバランスが `max_dust_htlc_exposure_msat` を超える場合：
          - チャネルを失敗させる *かもしれません*

#### 根拠

ビットコインの手数料は、一方的なクローズを効果的にするために必要です。`option_anchors` を使用することで、旧来のコミットメント形式ほど `feerate_per_kw` が確認を保証するために重要ではなくなりますが、それでもメモリプールに入るために十分である必要があります（最低中継手数料とメモリプールの最低手数料を満たす必要があります）。

旧来のコミットメント形式では、ブロードキャストするノードが親の手数料を子が支払うことで効果的な手数料を増やす一般的な方法はありません。

手数料の変動や、トランザクションが将来使用される可能性を考慮すると、手数料を支払う側が旧来のコミットメントトランザクションに対して十分な余裕（例えば、予想される手数料の 5 倍）を持つことが良い考えです。ただし、手数料の見積もり方法が異なるため、正確な値は指定されていません。

現在、手数料は一方的です（チャネル作成を要求した側が常にコミットメントトランザクションの手数料を支払います）。そのため、手数料レベルを設定するのは簡単ですが、同じ手数料率が HTLC トランザクションにも適用されるため、受信ノードも手数料の妥当性を考慮する必要があります。

オンチェーン手数料が増加し、コミットメントに多くの HTLC が含まれていて、更新された手数料率でトリムされる場合、設定された `max_dust_htlc_exposure_msat` を超える可能性があります。チャネルを事前にクローズするかどうかは、ノードのポリシーに委ねられています。

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
    1. type: 0 (`next_funding`)
    2. data:
        * [`sha256`:`next_funding_txid`]

`next_commitment_number`: コミットメント番号は各コミットメントトランザクションに対する 48 ビットのインクリメントカウンタです。カウンタはチャネル内の各ピアに対して独立しており、0 から始まります。再確立の場合を除いて、他のノードに明示的に中継されることはなく、それ以外の場合は暗黙的です。

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

送信ノード：

- `next_commitment_number` を、次に受信することを期待する `commitment_signed` のコミットメント番号に設定しなければなりません。
- `next_revocation_number` を、次に受信することを期待する `revoke_and_ack` メッセージのコミットメント番号に設定しなければなりません。
- `my_current_per_commitment_point` を有効なポイントに設定しなければなりません。
- `next_revocation_number` が 0 に等しい場合：
  - `your_last_per_commitment_secret` をすべてゼロに設定しなければなりません。
- それ以外の場合：
  - 受信した最後の `per_commitment_secret` に `your_last_per_commitment_secret` を設定しなければなりません。
- インタラクティブなトランザクション構築のために `commitment_signed` を送信したが、`tx_signatures` を受信していない場合：
  - そのインタラクティブなトランザクションの txid に `next_funding_txid` を設定しなければなりません。
- それ以外の場合：
  - `next_funding_txid` を設定してはなりません。

ノード：

- 送信および受信した `channel_reestablish` の両方で `next_commitment_number` が 1 の場合：
  - `channel_ready` を再送信しなければなりません。
- それ以外の場合：
  - `channel_ready` を再送信してはなりませんが、異なる `short_channel_id` の `alias` フィールドを持つ `channel_ready` を送信してもかまいません。
- 再接続時：
  - 受信した冗長な `channel_ready` を無視しなければなりません。
- `next_commitment_number` が受信ノードが最後に送信した `commitment_signed` メッセージのコミットメント番号に等しい場合：
  - 次の `commitment_signed` に同じコミットメント番号を再利用しなければなりません。
- それ以外の場合：
  - `next_commitment_number` が受信ノードが最後に送信した `commitment_signed` メッセージのコミットメント番号より 1 大きくない場合：
    - `error` を送信し、チャネルを失敗させるべきです。
  - `commitment_signed` を送信しておらず、かつ `next_commitment_number` が 1 に等しくない場合：
    - `error` を送信し、チャネルを失敗させるべきです。
- `next_revocation_number` が受信ノードが送信した最後の `revoke_and_ack` のコミットメント番号に等しく、かつ受信ノードがまだ `closing_signed` を受信していない場合：
  - `revoke_and_ack` を再送信しなければなりません。
  - 以前に再送信が必要な `commitment_signed` を送信している場合：
    - `revoke_and_ack` と `commitment_signed` を最初に送信したのと同じ相対順序で再送信しなければなりません。
- それ以外の場合：
  - `next_revocation_number` が受信ノードが送信した最後の `revoke_and_ack` のコミットメント番号より 1 大きくない場合：
    - `error` を送信し、チャネルを失敗させるべきです。
  - `revoke_and_ack` を送信しておらず、かつ `next_revocation_number` が 0 に等しくない場合：
    - `error` を送信し、チャネルを失敗させるべきです。


受信ノード：

- `my_current_per_commitment_point` を無視しなければなりませんが、有効なポイントであることを要求してもかまいません。
- `next_revocation_number` が上記で予想される値より大きく、かつ `your_last_per_commitment_secret` がその `next_revocation_number` マイナス 1 に対して正しい場合：
  - コミットメントトランザクションをブロードキャストしてはなりません。
  - ピアにチャネルを失敗させるよう要求する `error` を送信するべきです。
- それ以外の場合：
  - `your_last_per_commitment_secret` が予想される値と一致しない場合：
    - `error` を送信し、チャネルを失敗させるべきです。

受信ノード：

- `next_funding_txid` が設定されている場合：
  - `next_funding_txid` が最新のインタラクティブファンディングトランザクションと一致する場合：
    - そのファンディングトランザクションに対する `tx_signatures` をまだ受け取っていない場合：
      - そのファンディングトランザクションに対する `commitment_signed` を再送信しなければなりません。
      - すでに `commitment_signed` を受け取っており、[`tx_signatures` の要件](#the-tx_signatures-message)に従って最初に署名すべき場合：
        - そのファンディングトランザクションに対する `tx_signatures` を送信しなければなりません。
    - すでにそのファンディングトランザクションに対する `tx_signatures` を受け取っている場合：
      - そのファンディングトランザクションに対する `tx_signatures` を送信しなければなりません。
  - それ以外の場合：
    - 送信ノードにこのファンディングトランザクションを忘れることができることを知らせるために `tx_abort` を送信しなければなりません。

ノード：

- 以前に送信されたメッセージが失われたと仮定してはなりません。
  - 以前に `commitment_signed` メッセージを送信した場合：
    - 対応するコミットメントトランザクションが他方によっていつでもブロードキャストされる可能性がある場合を処理しなければなりません。
      - 注：これは特に、ノードが以前に送信した `update_` メッセージをそのまま再送信しない場合に重要です。
- 再接続時：
  - 以前に `shutdown` を送信した場合：
    - `shutdown` を再送信しなければなりません。

### 理論的根拠

上記の要件は、オープニングフェーズがほぼアトミックであることを保証します。完了しない場合は、再度開始します。唯一の例外は、`funding_signed` メッセージが送信されたが受信されなかった場合です。この場合、ファンダーはチャネルを忘れ、再接続時に新しいチャネルを開くと推測されます。一方、他のノードは、`channel_ready` を受信しないか、オンチェーンでファンディングトランザクションを確認しないため、最終的に元のチャネルを忘れることになります。

`error` には確認応答がないため、再接続が発生した場合には、再度切断する前に再送信するのが礼儀です。ただし、ノードがチャネルを完全に忘れてしまう場合もあるため、必須ではありません。

`closing_signed` も確認応答がないため、再接続時には再送信しなければなりません（ただし、再接続時には交渉が再開されるため、完全に同じ再送信である必要はありません）。`shutdown` の唯一の確認応答は `closing_signed` なので、どちらか一方を再送信する必要があります。

更新の処理も同様にアトミックです。コミットが確認されない（または送信されなかった）場合、更新は再送信されます。ただし、同一であることは求められません。異なる順序であったり、異なる手数料が関与したり、追加するには古すぎる HTLC が欠けている場合もあります。同一であることを要求すると、送信者が送信のたびにディスクに書き込むことを意味しますが、ここでのスキームは、送信または受信された各 `commitment_signed` に対して単一の永続的なディスクへの書き込みを推奨します。しかし、`commitment_signed` と `revoke_and_ack` の両方を再送信する必要がある場合、これら二つの相対的な順序は保持されなければなりません。さもないと、チャネルの閉鎖につながります。

`closing_signed` を受信した後に `revoke_and_ack` の再送信を要求されることは決してありません。これは、シャットダウンが完了したことを意味し、それはリモートノードが `revoke_and_ack` を受信した後にのみ発生するからです。

`next_commitment_number` は 1 から始まることに注意してください。コミットメント番号 0 はオープニング中に作成されます。
`next_revocation_number` は、コミットメント番号 1 の `commitment_signed` が送信され、コミットメント番号 0 の取り消しが受信されるまで 0 になります。

`channel_ready` は、通常の操作の開始によって暗黙的に確認されます。これは、`commitment_signed` が受信された後に開始されたことが知られているためです。したがって、`next_commitment_number` が 1 より大きいかどうかをテストします。

以前の草案では、資金提供者が「資金取引をブロードキャストした場合は覚えておかなければならない、そうでなければしてはならない」と主張していましたが、これは実際には不可能な要求でした。ノードは、まずディスクにコミットし、次にトランザクションをブロードキャストするか、その逆を行わなければなりません。新しい言語はこの現実を反映しています。ブロードキャストされていないチャネルを覚えておく方が、ブロードキャストされたチャネルを忘れるよりも確実に良いです。同様に、資金提供者の `funding_signed` メッセージについても、開かない（タイムアウトする）チャネルを覚えておく方が、資金提供者が開く間に資金提供者がそれを忘れてしまうよりも良いです。

ノードが何らかの理由で遅れてしまった場合（例えば、古いバックアップから復元された場合など）、遅れていることを検出することができます。遅れているノードは、自分の現在のコミットメントトランザクションをブロードキャストできないことを知っておく必要があります。これを行うと、リモートノードが取り消しプレイメージを知っていることを証明できるため、資金の全損につながります。遅れているノードから返される `error` は、他のノードが現在のコミットメントトランザクションをチェーンにドロップするように促すべきです。他のノードは、その `error` を待って、遅れているノードがまず状態を修正する機会を与えるべきです（例えば、異なるバックアップで再起動することによって）。

`next_funding_txid` は、ピアがインタラクティブなトランザクション構築の署名ステップを最終化することを可能にするか、またはピアの一方が署名していない場合にそのトランザクションを安全に中止することを可能にします。この場合、そのトランザクションはすでに状態から削除されています。

# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
