# BOLT #3: ビットコインのトランザクションとスクリプト形式

これは、署名が有効であることを確認するために両者が合意する必要があるオンチェーントランザクションの正確な形式を詳述します。これには、資金調達トランザクションの出力スクリプト、コミットメントトランザクション、および HTLC トランザクションが含まれます。

# 目次

  * [トランザクション](#transactions)
    * [トランザクション出力の順序](#transaction-output-ordering)
    * [Segwit の使用](#use-of-segwit)
    * [資金調達トランザクションの出力](#funding-transaction-output)
    * [コミットメントトランザクション](#commitment-transaction)
        * [コミットメントトランザクションの出力](#commitment-transaction-outputs)
          * [`to_local` 出力](#to_local-output)
          * [`to_remote` 出力](#to_remote-output)
          * [`to_local_anchor` と `to_remote_anchor`](#to_local_anchor-and-to_remote_anchor-output-option_anchors)
          * [提供された HTLC 出力](#offered-htlc-outputs)
          * [受け取った HTLC 出力](#received-htlc-outputs)
        * [トリムされた出力](#trimmed-outputs)
    * [HTLC-timeout と HTLC-success トランザクション](#htlc-timeout-and-htlc-success-transactions)
	* [クローズトランザクション](#closing-transaction)
    * [手数料](#fees)
        * [手数料の計算](#fee-calculation)
        * [手数料の支払い](#fee-payment)
        * [共同トランザクションの手数料計算](#calculating-fees-for-collaborative-transactions)
    * [ダスト制限](#dust-limits)
    * [コミットメントトランザクションの構築](#commitment-transaction-construction)
  * [鍵](#keys)
    * [鍵の導出](#key-derivation)
        * [`localpubkey`, `remotepubkey`, `local_htlcpubkey`, `remote_htlcpubkey`, `local_delayedpubkey`, および `remote_delayedpubkey` の導出](#localpubkey-remotepubkey-local_htlcpubkey-remote_htlcpubkey-local_delayedpubkey-and-remote_delayedpubkey-derivation)
        * [`revocationpubkey` の導出](#revocationpubkey-derivation)
        * [コミットメントごとの秘密の要件](#per-commitment-secret-requirements)
    * [効率的なコミットメントごとの秘密の保存](#efficient-per-commitment-secret-storage)
  * [付録 A: 期待される重み](#appendix-a-expected-weights)
      * [資金調達トランザクションの期待される重み (v2 チャネル確立)](#expected-weight-of-the-funding-transaction-v2-channel-establishment)
      * [コミットメントトランザクションの期待される重み](#expected-weight-of-the-commitment-transaction)
      * [HTLC-timeout と HTLC-success トランザクションの期待される重み](#expected-weight-of-htlc-timeout-and-htlc-success-transactions)
  * [付録 B: 資金調達トランザクションのテストベクトル](#appendix-b-funding-transaction-test-vectors)
  * [付録 C: コミットメントと HTLC トランザクションのテストベクトル](#appendix-c-commitment-and-htlc-transaction-test-vectors)
  * [付録 D: コミットメントごとの秘密生成のテストベクトル](#appendix-d-per-commitment-secret-generation-test-vectors)
    * [生成テスト](#generation-tests)
    * [保存テスト](#storage-tests)
  * [付録 E: 鍵導出のテストベクトル](#appendix-e-key-derivation-test-vectors)
  * [付録 F: コミットメントと HTLC トランザクションのテストベクトル (アンカー)](#appendix-f-commitment-and-htlc-transaction-test-vectors-anchors)
  * [付録 G: デュアルファンドトランザクションのテストベクトル](#appendix-g-dual-funded-transaction-test-vectors)
  * [参考文献](#references)
  * [著者](#authors)

# トランザクション

## トランザクション出力の順序

トランザクションの出力は常に以下の順序でソートされます：
 * まずその値に基づいて、小さいものから（全サトシ単位で、HTLC 出力の場合は millisatoshi 部分を無視します）
 * 次に `scriptpubkey` に基づき、共通の長さのプレフィックスを `memcmp` のように辞書順で比較し、長さが異なる場合は短いスクリプトを選択します。
 * 最後に、HTLC 出力の場合は `cltv_expiry` の昇順で。

## 理論的根拠

同じ `amount`（`amount_msat` から丸められた）と `payment_hash` を持つ 2 つの提供された HTLC は、`cltv_expiry` が異なっていても同一の出力を持ちます。これは、`htlc_signatures` を送信する際に同じ順序が使用され、HTLC トランザクション自体が異なるため、このケースの標準的な順序に両方のピアが同意する必要があるためにのみ重要です。

## Segwit の使用

ここで使用されるほとんどのトランザクション出力は、pay-to-witness-script-hash<sup>[BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#witness-program)</sup> (P2WSH) 出力です：Segwit バージョンの P2SH です。このような出力を使用するには、ウィットネススタックの最後の項目が、使用されている P2WSH 出力を生成するために使用された実際のスクリプトでなければなりません。この最後の項目は、このドキュメントの残りの部分では簡潔さのために省略されています。

`<>` は、MINIMALIF 標準ルールに準拠するために必要な空のベクターを示します。<sup>[MINIMALIF](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2016-August/013014.html)</sup>

## 資金調達トランザクション出力

* 資金調達出力スクリプトは次の P2WSH です：

`2 <pubkey1> <pubkey2> 2 OP_CHECKMULTISIG`

* ここで `pubkey1` は圧縮形式の 2 つの `funding_pubkey` のうち辞書順で小さい方であり、`pubkey2` は辞書順で大きい方です。

## コミットメントトランザクション

* バージョン：2
* ロックタイム：上位 8 ビットは 0x20、下位 24 ビットは隠されたコミットメント番号の下位 24 ビット
* txin カウント：1
   * `txin[0]` アウトポイント：`funding_created` メッセージからの `txid` と `output_index`
   * `txin[0]` シーケンス：上位 8 ビットは 0x80、下位 24 ビットは隠されたコミットメント番号の上位 24 ビット
   * `txin[0]` スクリプトバイト：0
   * `txin[0]` ウィットネス：`0 <signature_for_pubkey1> <signature_for_pubkey2>`

48 ビットのコミットメント番号は、次の下位 48 ビットと `XOR` することで隠されます。

    SHA256(payment_basepoint from open_channel || payment_basepoint from accept_channel)

これは、一方的なクローズの場合にチャネル上で行われたコミットメントの数を隠しますが、両ノード（`payment_basepoint` を知っている）が取り消されたコミットメントトランザクションを迅速に見つけるための有用なインデックスを提供します。

### コミットメントトランザクションのアウトプット

取り消されたコミットメントトランザクションの場合にペナルティトランザクションの機会を与えるために、コミットメントトランザクションの所有者（いわゆる「ローカルノード」）に資金を返すすべてのアウトプットは `to_self_delay` ブロックの間遅延されなければなりません。HTLC に対するこの遅延は、第二段階の HTLC トランザクション（ローカルノードが受け入れた HTLC の場合は HTLC-success、ローカルノードが提供した HTLC の場合は HTLC-timeout）で行われます。

HTLC アウトプットのための別のトランザクション段階がある理由は、HTLC が `to_self_delay` 遅延内にあってもタイムアウトしたり履行されたりできるようにするためです。そうでなければ、HTLC に必要な最小タイムアウトがこの遅延によって長くなり、ネットワークを通過する HTLC のタイムアウトが長くなります。

各アウトプットの金額は、必ず切り捨てて全サトシ単位にしなければなりません。この金額から HTLC トランザクションの手数料を引いた額が、コミットメントトランザクションの所有者によって設定された `dust_limit_satoshis` より少ない場合、アウトプットは生成されてはなりません（したがって資金は手数料に加算されます）。

#### `to_local` アウトプット

このアウトプットは、このコミットメントトランザクションの所有者に資金を返すため、`OP_CHECKSEQUENCEVERIFY` を使用してタイムロックされなければなりません。取り消し秘密鍵を知っている場合、他の当事者が遅延なしで請求できます。アウトプットはバージョン 0 の P2WSH で、ウィットネススクリプトは次のとおりです。

    OP_IF
        # ペナルティトランザクション
        <revocationpubkey>
    OP_ELSE
        `to_self_delay`
        OP_CHECKSEQUENCEVERIFY
        OP_DROP
        <local_delayedpubkey>
    OP_ENDIF
    OP_CHECKSIG

アウトプットは、`nSequence` フィールドが `to_self_delay` に設定されたインプットとウィットネスによって消費されます（この期間が経過した後でのみ有効になります）。

    <local_delayedsig> <>

もし取り消されたコミットメントトランザクションが公開された場合、相手側は以下のウィットネスを使ってこのアウトプットを即座に使用できます。

    <revocation_sig> 1

#### `to_remote` アウトプット

コミットメントトランザクションに `option_anchors` が適用される場合、`to_remote` アウトプットは 1 ブロックの CSV ロックで制約されます。

    <remotepubkey> OP_CHECKSIGVERIFY 1 OP_CHECKSEQUENCEVERIFY

このアウトプットは `nSequence` フィールドが `1` に設定されたインプットとウィットネスで使用されます。

    <remote_sig>

それ以外の場合、このアウトプットは `remotepubkey` への単純な P2WPKH です。注意：リモートのコミットメントトランザクションは、あなたの `localpubkey` を彼らの `to_remote` アウトプットとして使用します。

#### `to_local_anchor` と `to_remote_anchor` アウトプット (option_anchors)

このアウトプットは、子が親の手数料を支払うことでトランザクションをマイニングするインセンティブを提供するために、ローカルおよびリモートノードによってそれぞれ使用されます。アンカーアウトプットは常に追加されますが、HTLC がなく、片方の当事者がダスト制限を下回るコミットメントアウトプットを持っている場合を除きます。その場合、実際に生成されるコミットメントアウトプットにのみアンカーが追加されます。これは通常、開始者が開設直後にクローズする場合に発生します（`to_remote` アウトプットなし）。

    <local_funding_pubkey/remote_funding_pubkey> OP_CHECKSIG OP_IFDUP
    OP_NOTIF
        OP_16 OP_CHECKSEQUENCEVERIFY
    OP_ENDIF

各当事者は、自分のファンディングキーにロックされた独自のアンカーアウトプットを持っています。これは、悪意のあるピアが低い手数料密度の子トランザクションをアンカーに付けて、被害者がコミットメントトランザクションを時間内に確認できないようにするのを防ぐためです。この防御は、Bitcoin Core 0.19 の変更によってサポートされています：https://github.com/bitcoin/bitcoin/pull/15681。これが、コミットメントトランザクションのすべての非アンカーアウトプットが CSV ロックされている理由でもあります。

UTXO セットの汚染を防ぐため、未使用のまま残ったアンカーは、コミットメントトランザクションが確認された後、誰でも使用できます。これが、アンカーアウトプットをファンディングキーにロックする理由でもあります。第三者はこのキーを観察し、コミットメントアウトプットが使用されない場合でも、支払いスクリプトを再構築できます。これは、手数料市場がアンカーを掃除するのが経済的になるレベルまで下がることを前提としています。

出力の金額は 330 サトシに固定されており、これは P2WSH のデフォルトのダスト制限です。

出力を使用するには、次のウィットネスが必要です。

    <local_sig/remote_sig>

16 ブロック後、誰でも次のウィットネスを使ってアンカーをスイープできます。

    <>

#### 提供された HTLC 出力

この出力は、HTLC タイムアウト後に HTLC タイムアウトトランザクションに資金を送るか、支払いプレイメージまたは取り消しキーを使用してリモートノードに資金を送ります。出力は P2WSH であり、ウィットネススクリプトは以下の通りです（`option_anchors` がない場合）。

    # 取り消しキーを使用してリモートノードへ
    OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
    OP_IF
        OP_CHECKSIG
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_NOTIF
            # HTLC タイムアウトトランザクションを介してローカルノードへ（タイムロック）。
            OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # プレイメージを使用してリモートノードへ。
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF

または、`option_anchors` がある場合：

    # 取り消しキーを使用してリモートノードへ
    OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
    OP_IF
        OP_CHECKSIG
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_NOTIF
            # HTLC タイムアウトトランザクションを介してローカルノードへ（タイムロック）。
            OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # プレイメージを使用してリモートノードへ。
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            OP_CHECKSIG
        OP_ENDIF
        1 OP_CHECKSEQUENCEVERIFY OP_DROP
    OP_ENDIF

リモートノードは次のウィットネスを使って HTLC を引き出せます。

    <remotehtlcsig> <payment_preimage>

`option_anchors` が適用される場合、支出入力の nSequence フィールドは `1` でなければなりません。

取り消されたコミットメントトランザクションが公開された場合、リモートノードは次のウィットネスを使ってこの出力を即座に使用できます。

    <revocation_sig> <revocationpubkey>

送信ノードは、HTLC が期限切れになった後に HTLC タイムアウトトランザクションを使用して HTLC をタイムアウトできます。これはローカルノードが HTLC をタイムアウトできる唯一の方法であり、この分岐には `<remotehtlcsig>` が必要です。これにより、HTLC タイムアウトトランザクションが `cltv_expiry` を指定された `locktime` として持っているため、ローカルノードが HTLC を早期にタイムアウトすることはできません。ローカルノードはこれらの資金にアクセスする前に `to_self_delay` を待つ必要があり、トランザクションが取り消された場合にリモートノードがこれらの資金を請求できるようにします。

#### 受信 HTLC 出力

この出力は、HTLC タイムアウト後または取り消しキーを使用してリモートノードに資金を送るか、成功した支払いプレイメージを持つ HTLC 成功トランザクションに送ります。出力は P2WSH で、ウィットネススクリプトは以下の通りです（`option_anchors` がない場合）：

    # 取り消しキーを使用してリモートノードへ
    OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
    OP_IF
        OP_CHECKSIG
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_IF
            # HTLC 成功トランザクションを介してローカルノードへ。
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # タイムアウト後にリモートノードへ。
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF

または、`option_anchors` がある場合：

    # 取り消しキーを使用してリモートノードへ
    OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
    OP_IF
        OP_CHECKSIG
    OP_ELSE
        <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_IF
            # HTLC 成功トランザクションを介してローカルノードへ。
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            # タイムアウト後にリモートノードへ。
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            OP_CHECKSIG
        OP_ENDIF
        1 OP_CHECKSEQUENCEVERIFY OP_DROP
    OP_ENDIF

HTLC をタイムアウトさせるために、リモートノードは以下のウィットネスでそれを消費します：

    <remotehtlcsig> <>

`option_anchors` が適用される場合、消費入力の nSequence フィールドは `1` でなければならないことに注意してください。

取り消されたコミットメントトランザクションが公開された場合、リモートノードは以下のウィットネスを使用してこの出力を即座に消費できます：

    <revocation_sig> <revocationpubkey>

HTLC を引き出すためには、以下に詳述する HTLC 成功トランザクションが使用されます。これはローカルノードが HTLC を消費できる唯一の方法です。この分岐は `<remotehtlcsig>` を必要とし、これによりローカルノードはこれらの資金にアクセスする前に `to_self_delay` を待たなければならず、トランザクションが取り消された場合にリモートノードがこれらの資金を請求できるようにします。

### トリムされたアウトプット

各ピアは `dust_limit_satoshis` を指定し、それ以下のアウトプットは生成しないようにします。これら生成されないアウトプットは「トリムされた」と呼ばれます。トリムされたアウトプットは、生成する価値がないほど小さいと見なされ、その代わりにコミットメントトランザクションの手数料に加えられます。HTLC に関しては、第二段階の HTLC トランザクションもこの制限を下回る可能性があることを考慮する必要があります。

#### 要件

基本手数料とアンカーアウトプットの値：
  - コミットメントトランザクションのアウトプットが決定される前に：
    - [手数料計算](#fee-calculation)で指定されているように、`to_local` または `to_remote` アウトプットから差し引かなければなりません。

コミットメントトランザクション：
  - コミットメントトランザクションの `to_local` アウトプットの金額が、トランザクション所有者によって設定された `dust_limit_satoshis` より少ない場合：
    - そのアウトプットを含めてはなりません。
  - それ以外の場合：
    - [`to_local` アウトプット](#to_local-output)で指定されているように生成しなければなりません。
  - コミットメントトランザクションの `to_remote` アウトプットの金額が、トランザクション所有者によって設定された `dust_limit_satoshis` より少ない場合：
    - そのアウトプットを含めてはなりません。
  - それ以外の場合：
    - [`to_remote` アウトプット](#to_remote-output)で指定されているように生成しなければなりません。
  - 提供された各 HTLC に対して：
    - HTLC 金額から HTLC-タイムアウト手数料を引いた額が、トランザクション所有者によって設定された `dust_limit_satoshis` より少ない場合：
      - そのアウトプットを含めてはなりません。
    - それ以外の場合：
      - [提供された HTLC アウトプット](#offered-htlc-outputs)で指定されているように生成しなければなりません。
  - 受け取った各 HTLC に対して：
    - HTLC 金額から HTLC-成功手数料を引いた額が、トランザクション所有者によって設定された `dust_limit_satoshis` より少ない場合：
      - そのアウトプットを含めてはなりません。
    - それ以外の場合：
      - [受け取った HTLC アウトプット](#received-htlc-outputs)で指定されているように生成しなければなりません。

## HTLC-タイムアウトおよび HTLC-成功トランザクション

これらの HTLC トランザクションはほぼ同一ですが、HTLC-タイムアウトトランザクションはタイムロックされています。HTLC-タイムアウト/HTLC-成功トランザクションは、有効なペナルティトランザクションによって消費されることができます。


* version: 2
* locktime: HTLC-success の場合は `0`、HTLC-timeout の場合は `cltv_expiry`
* txin count: 1
   * `txin[0]` outpoint: コミットメントトランザクションの `txid` と、HTLC トランザクションに対応する HTLC 出力の `output_index`
   * `txin[0]` sequence: `0`（`option_anchors` の場合は `1` に設定）
   * `txin[0]` script bytes: `0`
   * `txin[0]` witness stack: HTLC-success の場合は `0 <remotehtlcsig> <localhtlcsig> <payment_preimage>`、HTLC-timeout の場合は `0 <remotehtlcsig> <localhtlcsig> <>`
* txout count: 1
   * `txout[0]` amount: HTLC の `amount_msat` を 1000 で割った値（切り捨て）から手数料を引いたサトシ（詳細は [Fee Calculation](#fee-calculation) を参照）
   * `txout[0]` script: 以下に示す witness script を持つバージョン 0 の P2WSH
* このコミットメントトランザクションに `option_anchors` が適用される場合、`SIGHASH_SINGLE|SIGHASH_ANYONECANPAY` が [BOLT #5](05-onchain.md#generation-of-htlc-transactions) に記載されているように使用されます。

出力の witness script は以下の通りです：

    OP_IF
        # ペナルティトランザクション
        <revocationpubkey>
    OP_ELSE
        `to_self_delay`
        OP_CHECKSEQUENCEVERIFY
        OP_DROP
        <local_delayedpubkey>
    OP_ENDIF
    OP_CHECKSIG

ペナルティを通じてこれを使用するには、リモートノードは witness stack `<revocationsig> 1` を使用し、出力を回収するには、ローカルノードは nSequence `to_self_delay` と witness stack `<local_delayedsig> 0` を使用します。

## Closing Transaction

各ノードには 2 つの可能なバリアントがあることに注意してください。

* version: 2
* locktime: 0
* txin count: 1
   * `txin[0]` outpoint: `funding_created` メッセージからの `txid` と `output_index`
   * `txin[0]` sequence: 0xFFFFFFFF
   * `txin[0]` script bytes: 0
   * `txin[0]` witness: `0 <signature_for_pubkey1> <signature_for_pubkey2>`
* txout count: 1 または 2
   * `txout` amount: 一方のノードに支払われる最終残高（このピアがチャネルを資金提供した場合、`closing_signed` からの `fee_satoshis` を差し引く）
   * `txout` script: そのノードの `shutdown` メッセージ内の `scriptpubkey` で指定されたもの

### 要件

署名を提供する各ノードは：
  - 各出力をサトシ単位に切り捨てなければなりません。
  - `fee_satoshis` によって与えられた手数料を資金提供者への出力から差し引かなければなりません。
  - 自身の `dust_limit_satoshis` 未満の出力を削除しなければなりません。
  - 自身の出力を削除してもかまいません。

### 理論的根拠

一方のノードが他方の出力をビットコインネットワーク上で伝播させるには小さすぎる（いわゆる「ダスト」）と考え、他方のノードがその出力を捨てるには価値がありすぎると考える場合、クローズ時に修復不可能な違いが生じる可能性があります。このため、各側が独自の `dust_limit_satoshis` を使用し、クローズトランザクションの見た目について意見が一致しない場合、署名検証の失敗が起こる可能性があります。

しかし、一方が自分の出力を削除することを選択した場合、他方がクローズプロトコルを失敗させる理由はありません。したがって、これは明示的に許可されています。署名はどのバリアントが使用されたかを示します。

資金額が `dust_limit_satoshis` の2倍を超える場合、少なくとも1つの出力が存在します。

## 手数料

### 手数料の計算

コミットメントトランザクションと HTLC トランザクションの手数料計算は、現在の `feerate_per_kw` とトランザクションの*期待される重量*に基づいています。

実際の重量と期待される重量は、いくつかの理由で異なります：

* ビットコインは DER エンコードされた署名を使用しており、サイズが変動します。
* ビットコインは可変長整数も使用しているため、多くの出力がある場合、エンコードに3バイトを要することがあります。
* `to_remote` 出力がダスト制限を下回る可能性があります。
* 手数料が引かれると `to_local` 出力がダスト制限を下回る可能性があります。

したがって、*期待される重量*の簡略化された公式が使用され、以下を仮定します：

* 署名は73バイト（最大長）です。
* 出力の数が少ない（したがって1バイトでカウントされる）です。
* 常に `to_local` 出力と `to_remote` 出力があります。
* （`option_anchors` がある場合）常に `to_local_anchor` と `to_remote_anchor` 出力があります。

これにより、以下の*期待される重量*が得られます（計算の詳細は[付録 A](#appendix-a-expected-weights)にあります）：

    コミットメント重量（option_anchors なし）：   724 + 172 * num-untrimmed-htlc-outputs
    コミットメント重量（option_anchors あり）：  1124 + 172 * num-untrimmed-htlc-outputs
    HTLC-timeout 重量（option_anchors なし）： 663
    HTLC-timeout 重量（option_anchors あり）： 666
    HTLC-success 重量（option_anchors なし）： 703
    HTLC-success 重量（option_anchors あり）： 706

コミットメントトランザクションの「基本手数料」への言及に注意してください。これは資金提供者が支払うものです。実際の手数料は、四捨五入や出力のトリミングにより、ここで計算された金額よりも高くなる場合があります。

#### 要件

HTLCタイムアウトトランザクションの手数料：
  - `option_anchors` が適用される場合：
    1. 0でなければなりません。
  - それ以外の場合、以下に一致するように計算しなければなりません：
    1. `feerate_per_kw` に 663 を掛け、1000 で割ります（切り捨て）。

HTLC成功トランザクションの手数料：
  - `option_anchors` が適用される場合：
    1. 0でなければなりません。
  - それ以外の場合、以下に一致するように計算しなければなりません：
    1. `feerate_per_kw` に 703 を掛け、1000 で割ります（切り捨て）。

コミットメントトランザクションの基本手数料：
  - 以下に一致するように計算しなければなりません：
    1. `weight` を 724（`option_anchors` が適用される場合は 1124）で開始します。
    2. 各コミットされた HTLC について、その出力が [Trimmed Outputs](#trimmed-outputs) で指定されたようにトリミングされていない場合、`weight` に 172 を加えます。
    3. `feerate_per_kw` に `weight` を掛け、1000 で割ります（切り捨て）。

#### 例

例えば、`feerate_per_kw` が 5000、`dust_limit_satoshis` が 546 satoshis の場合、次のようなコミットメントトランザクションがあります：
* 5000000 ミリサトシ（5000 satoshis）と 1000000 ミリサトシ（1000 satoshis）の 2 つの提供された HTLC
* 7000000 ミリサトシ（7000 satoshis）と 800000 ミリサトシ（800 satoshis）の 2 つの受け取った HTLC

HTLCタイムアウトトランザクションの `weight` は 663 で、したがって手数料は 3315 satoshis です。
HTLC成功トランザクションの `weight` は 703 で、したがって手数料は 3515 satoshis です。

コミットメントトランザクションの `weight` は次のように計算されます：

* `weight` は 724 で開始します。

* 5000 satoshis の提供された HTLC は 546 + 3315 を超えており、次の結果をもたらします：
  * コミットメントトランザクションで 5000 satoshi の出力
  * この出力を消費する 5000 - 3315 satoshis の HTLCタイムアウトトランザクション
  * `weight` は 896 に増加

* 1000 satoshis の提供された HTLC は 546 + 3315 を下回っているため、トリミングされます。

* 7000 satoshis の受け取った HTLC は 546 + 3515 を超えており、次の結果をもたらします：
  * コミットメントトランザクションで 7000 satoshi の出力
  * この出力を消費する 7000 - 3515 satoshis の HTLC成功トランザクション
  * `weight` は 1068 に増加


* 受け取った 800 サトシの HTLC は 546 + 3515 を下回るため、トリムされます。

基本的なコミットメントトランザクションの手数料は 5340 サトシです。実際の手数料（ダスト出力を作る 1000 および 800 サトシの HTLC を追加する）は 7140 サトシです。`to_local` または `to_remote` 出力が `dust_limit_satoshis` を下回る場合、最終的な手数料はさらに高くなる可能性があります。

### 手数料の支払い

基本的なコミットメントトランザクションの手数料と `to_local_anchor` および `to_remote_anchor` 出力の金額は、資金提供者の金額から差し引かれます。資金提供者に関するコミットメントトランザクション出力の制限は、[BOLT #2](02-peer-protocol.md) に記載されているように、チャネルリザーブに関連して適用されます。

手数料が資金提供者の出力から差し引かれた後、その出力が `dust_limit_satoshis` を下回る可能性があることに注意してください。そのため、手数料にも寄与します。

ノードは以下を行います：
  - 結果として得られる手数料率が低すぎる場合：
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させることができます（MAY）。

### 協調トランザクションの手数料計算

[インタラクティブプロトコル](02-peer-protocol.md#interactive-transaction-construction)を使用して構築されたトランザクションの場合、手数料はトランザクションの各当事者によって支払われ、開始時に決定された `feerate` で、共通のトランザクションフィールドの手数料はイニシエータが負担します。

## ダスト制限

`dust_limit_satoshis` パラメータは、ノードがオンチェーントランザクション出力を生成しない閾値を設定するために使用されます。

ビットコインには、ダスト閾値を下回る出力を無効または使用不可能にするコンセンサスルールはありませんが、一般的な実装のポリシールールは、そのような出力を含むトランザクションの中継を防ぎます。

Bitcoin Core は以下のダスト閾値を定義しています：

- 公開鍵ハッシュへの支払い (p2pkh)：546 サトシ
- スクリプトハッシュへの支払い (p2sh)：540 サトシ
- 公開鍵ハッシュへの証人支払い (p2wpkh)：294 サトシ
- スクリプトハッシュへの証人支払い (p2wsh)：330 サトシ
- 不明な Segwit バージョン：354 サトシ

この計算の理論的根拠（[こちら](https://github.com/bitcoin/bitcoin/blob/2aff9a36c352640a263e8b5de469710f7e80eb54/src/policy/policy.cpp#L28)で実装されています）は、以下のセクションで説明されています。

これらのセクションでは、Bitcoin Core の実装に従って、3000 sat/kB の手数料レートで計算を行います。

### Pay to pubkey hash (p2pkh)

p2pkh 出力は 34 バイトです：

- 出力額に 8 バイト
- スクリプトの長さに 1 バイト
- スクリプトに 25 バイト（`OP_DUP` `OP_HASH160` `20` 20 バイト `OP_EQUALVERIFY` `OP_CHECKSIG`）

p2pkh 入力は少なくとも 148 バイトです：

- 前の出力に 36 バイト（32 バイトのハッシュ + 4 バイトのインデックス）
- シーケンスに 4 バイト
- スクリプトシグの長さに 1 バイト
- スクリプトシグに 107 バイト：
  - アイテム数に 1 バイト
  - 署名の長さに 1 バイト
  - 署名に 71 バイト
  - 公開鍵の長さに 1 バイト
  - 公開鍵に 33 バイト

したがって、p2pkh のダスト閾値は `(34 + 148) * 3000 / 1000 = 546 satoshis` です。

### Pay to script hash (p2sh)

p2sh 出力は 32 バイトです：

- 出力額に 8 バイト
- スクリプトの長さに 1 バイト
- スクリプトに 23 バイト（`OP_HASH160` `20` 20 バイト `OP_EQUAL`）

p2sh 入力は基礎となるスクリプトに依存するため、固定サイズではありませんが、下限として 148 バイトを使用します。

したがって、p2sh のダスト閾値は `(32 + 148) * 3000 / 1000 = 540 satoshis` です。

### Pay to witness pubkey hash (p2wpkh)

p2wpkh 出力は 31 バイトです：

- 出力額に 8 バイト
- スクリプトの長さに 1 バイト
- スクリプトに 22 バイト（`OP_0` `20` 20 バイト）

p2wpkh 入力は少なくとも 67 バイトです（署名の長さに依存）：

- 前の出力に 36 バイト（32 バイトのハッシュ + 4 バイトのインデックス）
- シーケンスに 4 バイト
- スクリプトシグの長さに 1 バイト
- 証拠に 26 バイト（26.75 から切り下げ、75% の segwit 割引適用）：
  - アイテム数に 1 バイト
  - 署名の長さに 1 バイト
  - 署名に 71 バイト
  - 公開鍵の長さに 1 バイト
  - 公開鍵に 33 バイト

したがって、p2wpkh のダスト閾値は `(31 + 67) * 3000 / 1000 = 294 satoshis` です。

### Pay to witness script hash (p2wsh)

p2wsh 出力は 43 バイトです：

- 出力額に 8 バイト
- スクリプトの長さに 1 バイト
- スクリプトに 34 バイト（`OP_0` `32` 32 バイト）

p2wsh のインプットは、基礎となるスクリプトに依存するため固定サイズではありません。そのため、67 バイトを下限として使用します。

p2wsh のダスト閾値は次のようになります：`(43 + 67) * 3000 / 1000 = 330 satoshis`

### 不明な segwit バージョン

不明な segwit アウトプットは最大で 51 バイトです：

- アウトプット量に 8 バイト
- スクリプト長に 1 バイト
- スクリプトに 42 バイト（`OP_1` から `OP_16` までを含み、2 から 40 バイトの単一プッシュが続く）

インプットは基礎となるスクリプトに依存するため固定サイズではありません。そのため、67 バイトを下限として使用します。

不明な segwit バージョンのダスト閾値は次のようになります：`(51 + 67) * 3000 / 1000 = 354 satoshis`

## コミットメントトランザクションの構築

このセクションでは、前のセクションをまとめて、あるピアのコミットメントトランザクションを構築するアルゴリズムを詳述します。以下の情報が与えられた場合です：そのピアの `dust_limit_satoshis`、現在の `feerate_per_kw`、各ピアに支払われる金額（`to_local` と `to_remote`）、およびすべてのコミットされた HTLC：

1. [コミットメントトランザクション](#commitment-transaction)で指定されたように、コミットメントトランザクションのインプットとロックタイムを初期化します。
2. コミットされた HTLC のうち、どれがトリムされる必要があるかを計算します（[トリムされたアウトプット](#trimmed-outputs)を参照）。
3. 基本の[コミットメントトランザクション手数料](#fee-calculation)を計算します。
4. この基本手数料をファンダー（`to_local` または `to_remote` のいずれか）から差し引きます。`option_anchors` がコミットメントトランザクションに適用される場合は、固定アンカーサイズの 330 sats を 2 回ファンダー（`to_local` または `to_remote` のいずれか）から差し引きます。
5. 提供されたすべての HTLC について、トリムされていない場合は、[提供された HTLC アウトプット](#offered-htlc-outputs)を追加します。
6. 受け取ったすべての HTLC について、トリムされていない場合は、[受け取った HTLC アウトプット](#received-htlc-outputs)を追加します。
7. `to_local` の金額が `dust_limit_satoshis` 以上である場合、[`to_local` アウトプット](#to_local-output)を追加します。
8. `to_remote` の金額が `dust_limit_satoshis` 以上である場合、[`to_remote` アウトプット](#to_remote-output)を追加します。
9. `option_anchors` がコミットメントトランザクションに適用される場合：
   * `to_local` が存在するか、トリムされていない HTLC がある場合、[`to_local_anchor` アウトプット](#to_local_anchor-and-to_remote_anchor-output-option_anchors)を追加します。
   * `to_remote` が存在するか、トリムされていない HTLC がある場合、[`to_remote_anchor` アウトプット](#to_local_anchor-and-to_remote_anchor-output-option_anchors)を追加します。
10. アウトプットを [BIP 69+CLTV 順序](#transaction-output-ordering)に並べ替えます。

# 鍵

## 鍵の導出

各コミットメントトランザクションは一意の `localpubkey` を使用します。HTLC-success トランザクションと HTLC-timeout トランザクションは `local_delayedpubkey` と `revocationpubkey` を使用します。これらは `per_commitment_point` に基づいて各トランザクションごとに変更されます。

鍵を変更する理由は、取り消されたトランザクションを信頼せずに監視することを外部に委託できるようにするためです。このような _ウォッチャー_ は、コミットメントトランザクションの内容を特定できないようにする必要があります。たとえ _ウォッチャー_ が監視すべきトランザクション ID を知っていて、どの HTLC や残高が含まれているかを合理的に推測できたとしてもです。それでも、すべてのコミットメントトランザクションを保存することを避けるために、_ウォッチャー_ に `per_commitment_secret` の値（コンパクトに保存可能）と、ペナルティトランザクションに必要なスクリプトを再生成するために使用される `revocation_basepoint` と `delayed_payment_basepoint` を渡すことができます。したがって、_ウォッチャー_ は各ペナルティ入力の署名だけを受け取り（そして保存し）さえすればよいのです。

毎回 `localpubkey` を変更することで、コミットメントトランザクション ID は、`to_local` 出力がないという自明な場合を除いて推測されることはありません。すべてのコミットメントトランザクションはその出力スクリプトに ID を使用します。ペナルティトランザクションに必要な `local_delayedpubkey` を分割することで、`localpubkey` を明かすことなく _ウォッチャー_ と共有することができます。たとえ両方のピアが同じ _ウォッチャー_ を使用しても、何も明かされません。

最後に、通常の一方的なクローズの場合でも、HTLC-success および/または HTLC-timeout トランザクションは _ウォッチャー_ に何も明かしません。なぜなら、対応する `per_commitment_secret` を知らず、`local_delayedpubkey` や `revocationpubkey` をその基点と関連付けることができないからです。

効率のために、鍵は単一のシードから生成される一連の per-commitment secrets から生成されます。これにより、受信者はそれらをコンパクトに保存できます（[下記](#efficient-per-commitment-secret-storage) を参照）。

### `localpubkey`、`local_htlcpubkey`、`remote_htlcpubkey`、`local_delayedpubkey`、および `remote_delayedpubkey` の導出

これらの pubkey は、基点からの加算によって単純に生成されます。

	pubkey = basepoint + SHA256(per_commitment_point || basepoint) * G

- `localpubkey` はローカルノードの `payment_basepoint` を使用します。
- `local_htlcpubkey` はローカルノードの `htlc_basepoint` を使用します。
- `remote_htlcpubkey` はリモートノードの `htlc_basepoint` を使用します。
- `local_delayedpubkey` はローカルノードの `delayed_payment_basepoint` を使用します。
- `remote_delayedpubkey` はリモートノードの `delayed_payment_basepoint` を使用します。

対応する秘密鍵も、基点の秘密がわかっている場合（つまり、`localpubkey`、`local_htlcpubkey`、および `local_delayedpubkey` に対応する秘密鍵のみ）には同様に導出できます。

    privkey = basepoint_secret + SHA256(per_commitment_point || basepoint)

### `remotepubkey` の導出

`remotepubkey` は単にリモートノードの `payment_basepoint` です。

これは、ノードがデータを失い対応する `per_commitment_point` を知らなくても、コミットメントトランザクションを消費できることを意味します。ウォッチタワーは、オンチェーンで `to_remote` 出力のみを持つトランザクションを見た場合に、それに与えられたトランザクションを関連付けることができますが、そのようなトランザクションは強制を必要とせず、ウォッチタワーに渡すべきではありません。

### `revocationpubkey` の導出

`revocationpubkey` はブラインドキーです。ローカルノードがリモートノードのために新しいコミットメントを作成したいとき、自身の `revocation_basepoint` とリモートノードの `per_commitment_point` を使用して、コミットメントのための新しい `revocationpubkey` を導出します。リモートノードが使用された `per_commitment_secret` を公開した後（これによりそのコミットメントが取り消されます）、ローカルノードは `revocationprivkey` を導出できます。これは、キーを導出するために必要な 2 つの秘密（`revocation_basepoint_secret` と `per_commitment_secret`）を知っているためです。

`per_commitment_point` は楕円曲線乗算を使用して生成されます。

	per_commitment_point = per_commitment_secret * G

そして、これはリモートノードの `revocation_basepoint` から revocation pubkey を導出するために使用されます。

	revocationpubkey = revocation_basepoint * SHA256(revocation_basepoint || per_commitment_point) + per_commitment_point * SHA256(per_commitment_point || revocation_basepoint)

この構造により、ベースポイントを提供するノードも `per_commitment_point` を提供するノードも、他のノードの秘密なしに秘密鍵を知ることができないようにしています。

対応する秘密鍵は、`per_commitment_secret` が分かれば導出できます：

    revocationprivkey = revocation_basepoint_secret * SHA256(revocation_basepoint || per_commitment_point) + per_commitment_secret * SHA256(per_commitment_point || revocation_basepoint)

### Per-commitment Secret の要件

ノードは：
  - 各接続に対して推測不可能な 256 ビットのシードを選ばなければならない、
  - シードを公開してはならない。

最大で (2^48 - 1) 個の per-commitment secret を生成できます。

最初に使用するシークレット：
  - インデックス 281474976710655 でなければならない、
    - そこからインデックスは減少します。

I 番目のシークレット P：
  - このアルゴリズムの出力と一致しなければならない：
```
generate_from_seed(seed, I):
    P = seed
    for B in 47 down to 0:
        if B set in I:
            flip(B) in P
            P = SHA256(P)
    return P
```

ここで「flip(B)」は、値の (B div 8) バイトの (B mod 8) ビットを交互に切り替えます。したがって、「e3b0...」の「flip(0)」は「e2b0...」であり、「e3b0...」の「flip(10)」は「e3b4...」です。

受信ノードは：
  - すべての以前の per-commitment secret を保存してもよい。
  - 以下に説明するように、コンパクトな表現から計算してもよい。

## 効率的な Per-commitment Secret の保存

一連のシークレットの受信者は、49 の (value, index) ペアの配列にコンパクトに保存できます。なぜなら、2^X 境界上の特定のシークレットに対して、次の 2^X 境界までのすべてのシークレットを導出できるからです。そして、シークレットは常に `0xFFFFFFFFFFFF` から始まる降順で受信されます。

バイナリでは、任意のインデックスを *プレフィックス* と、それに続くいくつかの末尾の 0 として考えると便利です。この *プレフィックス* に一致する任意のインデックスのシークレットを導出できます。

例えば、シークレット `0xFFFFFFFFFFF0` は、`0xFFFFFFFFFFF1` から `0xFFFFFFFFFFFF` までのシークレットを導出することができます。そして、シークレット `0xFFFFFFFFFF08` は、`0xFFFFFFFFFF09` から `0xFFFFFFFFFF0F` までのシークレットを導出することができます。

これは、上記の `generate_from_seed` のわずかな一般化を使用して行います：

    # インデックスのビット..47 が同じであるベースシークレットから I 番目のシークレットを返します。
    derive_secret(base, bits, I):
        P = base
        for B in bits - 1 down to 0:
            if B set in I:
                flip(B) in P
                P = SHA256(P)
        return P


各ユニークなプレフィックスに対して保存する秘密は一つだけでよく、実際には末尾の 0 の数を数え、それによってストレージ配列のどこに秘密を保存するかを決定します：

    # 別名：末尾の 0 を数える
    where_to_put_secret(I):
        for B in 0 to 47:
            if testbit(I) in B == 1:
                return B
        # I = 0、これはシードです。
        return 48

すべての以前の秘密が正しく導出されることを確認するためのダブルチェックが必要です。このチェックが失敗した場合、秘密は同じシードから生成されていません：

    insert_secret(secret, I):
        B = where_to_put_secret(I)

        # これはトラバーサル全体で各バケット内の秘密のインデックスを追跡します。
        for b in 0 to B:
            if derive_secret(secret, B, known[b].index) != known[b].secret:
                error The secret for I is incorrect
                return

        # これが自動的に必要に応じて known[] を拡張すると仮定します。
        known[B].index = I
        known[B].secret = secret

最後に、インデックス `I` で未知の秘密を導出する必要がある場合、どの既知の秘密を使用してそれを導出できるかを発見しなければなりません。最も簡単な方法は、すべての既知の秘密を反復し、それぞれが未知の秘密を導出するために使用できるかどうかをテストすることです：

    derive_old_secret(I):
        for b in 0 to len(secrets):
            # インデックスの非ゼロプレフィックスをマスクします。
            MASK = ~((1 << b) - 1)
            if (I & MASK) == secrets[b].index:
                return derive_secret(known, i, I)
        error Index 'I' hasn't been received yet.

これは複雑に見えますが、エントリ `b` のインデックスには `b` 個の末尾の 0 があることを覚えておいてください。マスクと比較は、各バケットのインデックスが目的のインデックスのプレフィックスであるかどうかを単にチェックします。

# Appendix A: Expected Weights

## Expected Weight of the Funding Transaction (v2 Channel Establishment)

資金調達トランザクションの*期待される重み*は次のように計算されます：

      inputs: 41 bytes
        - previous_out_point: 36 bytes
          - hash: 32 bytes
          - index: 4 bytes
        - var_int: 1 byte
        - script_sig: 0 bytes
        - witness <---- 「witness」データのコストは別途計算されます。
        - sequence: 4 bytes


      non_funding_outputs: 9 バイト + `scriptlen`
        - value: 8 バイト
        - var_int: 1 バイト <---- 標準的な出力スクリプトを仮定
        - script: `scriptlen`

      funding_output: 43 バイト
        - value: 8 バイト
        - var_int: 1 バイト
        - script: 34 バイト
          - OP_0: 1 バイト
          - PUSHDATA(32-byte-hash): 33 バイト

非証人データを 4 倍すると、重みは次のようになります：

      // transaction_fields = 10 (version, input count, output count, locktime)
      // segwit_fields = 2 (marker + flag)
      // funding_transaction = 43 + num_inputs * 41 + num_outputs * 9 + sum(scriptlen)
      funding_transaction_weight = 4 * (funding_transaction + transaction_fields) + segwit_fields

      witness_weight = sum(witness_len)

      overall_weight = funding_transaction_weight + witness_weight


## コミットメントトランザクションの期待される重み

コミットメントトランザクションの*期待される重み*は次のように計算されます：

	p2wsh: 34 バイト
		- OP_0: 1 バイト
		- OP_DATA: 1 バイト (witness_script_SHA256 の長さ)
		- witness_script_SHA256: 32 バイト

	p2wpkh: 22 バイト
		- OP_0: 1 バイト
		- OP_DATA: 1 バイト (public_key_HASH160 の長さ)
		- public_key_HASH160: 20 バイト

	multi_sig: 71 バイト
		- OP_2: 1 バイト
		- OP_DATA: 1 バイト (pub_key_alice の長さ)
		- pub_key_alice: 33 バイト
		- OP_DATA: 1 バイト (pub_key_bob の長さ)
		- pub_key_bob: 33 バイト
		- OP_2: 1 バイト
		- OP_CHECKMULTISIG: 1 バイト

	witness: 222 バイト
		- number_of_witness_elements: 1 バイト
		- nil_length: 1 バイト
		- sig_alice_length: 1 バイト
		- sig_alice: 73 バイト
		- sig_bob_length: 1 バイト
		- sig_bob: 73 バイト
		- witness_script_length: 1 バイト
		- witness_script (multi_sig)

	funding_input: 41 バイト
		- previous_out_point: 36 バイト
			- hash: 32 バイト
			- index: 4 バイト
		- var_int: 1 バイト (script_sig の長さ)
		- script_sig: 0 バイト
		- witness <---- トランザクションの検証には "witness" が "script_sig" の代わりに使用されます。ただし、"witness" は別に保存され、そのサイズのコストは小さくなります。したがって、通常のデータの計算は witness データから分離されています。
		- sequence: 4 バイト


	output_paying_to_local: 43 バイト
		- value: 8 バイト
		- var_int: 1 バイト (pk_script length)
		- pk_script (p2wsh): 34 バイト

	output_paying_to_remote (no option_anchors): 31 バイト
		- value: 8 バイト
		- var_int: 1 バイト (pk_script length)
		- pk_script (p2wpkh): 22 バイト

	output_paying_to_remote (option_anchors): 43 バイト
		- value: 8 バイト
		- var_int: 1 バイト (pk_script length)
		- pk_script (p2wsh): 34 バイト

	output_anchor (option_anchors): 43 バイト
		- value: 8 バイト
		- var_int: 1 バイト (pk_script length)
		- pk_script (p2wsh): 34 バイト

    htlc_output: 43 バイト
		- value: 8 バイト
		- var_int: 1 バイト (pk_script length)
		- pk_script (p2wsh): 34 バイト

	 witness_header: 2 バイト
		- flag: 1 バイト
		- marker: 1 バイト

	 commitment_transaction (no option_anchors): 125 + 43 * num-htlc-outputs バイト
		- version: 4 バイト
		- witness_header <---- part of the witness data
		- count_tx_in: 1 バイト
		- tx_in: 41 バイト
			funding_input
		- count_tx_out: 1 バイト
		- tx_out: 74 + 43 * num-htlc-outputs バイト
			output_paying_to_remote,
			output_paying_to_local,
			....htlc_output's...
		- lock_time: 4 バイト

	 commitment_transaction (option_anchors): 225 + 43 * num-htlc-outputs バイト
		- version: 4 バイト
		- witness_header <---- part of the witness data
		- count_tx_in: 1 バイト
		- tx_in: 41 バイト
			funding_input
		- count_tx_out: 3 バイト
		- tx_out: 172 + 43 * num-htlc-outputs バイト
			output_paying_to_remote,
			output_paying_to_local,
			output_anchor,
			output_anchor,
			....htlc_output's...
		- lock_time: 4 バイト

非ウィットネスデータを 4 倍すると、重みは次のようになります：

	// 500 + 172 * num-htlc-outputs weight (no option_anchors)
	// 900 + 172 * num-htlc-outputs weight (option_anchors)
	commitment_transaction_weight = 4 * commitment_transaction

	// 224 weight
	witness_weight = witness_header + witness

	overall_weight (no option_anchors) = 500 + 172 * num-htlc-outputs + 224 weight
	overall_weight (option_anchors) = 900 + 172 * num-htlc-outputs + 224 weight

## HTLC-timeout と HTLC-success トランザクションの期待される重み

HTLC トランザクションの*期待される重量*は次のように計算されます：

    accepted_htlc_script: 140 bytes (option_anchors を使用すると 143 bytes)
        - OP_DUP: 1 byte
        - OP_HASH160: 1 byte
        - OP_DATA: 1 byte (RIPEMD160(SHA256(revocationpubkey)) の長さ)
        - RIPEMD160(SHA256(revocationpubkey)): 20 bytes
        - OP_EQUAL: 1 byte
        - OP_IF: 1 byte
        - OP_CHECKSIG: 1 byte
        - OP_ELSE: 1 byte
        - OP_DATA: 1 byte (remotepubkey の長さ)
        - remotepubkey: 33 bytes
        - OP_SWAP: 1 byte
        - OP_SIZE: 1 byte
        - OP_DATA: 1 byte (32 の長さ)
        - 32: 1 byte
        - OP_EQUAL: 1 byte
        - OP_IF: 1 byte
        - OP_HASH160: 1 byte
        - OP_DATA: 1 byte (RIPEMD160(payment_hash) の長さ)
        - RIPEMD160(payment_hash): 20 bytes
        - OP_EQUALVERIFY: 1 byte
        - 2: 1 byte
        - OP_SWAP: 1 byte
        - OP_DATA: 1 byte (localpubkey の長さ)
        - localpubkey: 33 bytes
        - 2: 1 byte
        - OP_CHECKMULTISIG: 1 byte
        - OP_ELSE: 1 byte
        - OP_DROP: 1 byte
        - OP_DATA: 1 byte (cltv_expiry の長さ)
        - cltv_expiry: 4 bytes
        - OP_CHECKLOCKTIMEVERIFY: 1 byte
        - OP_DROP: 1 byte
        - OP_CHECKSIG: 1 byte
        - OP_ENDIF: 1 byte
        - OP_1: 1 byte (option_anchors)
        - OP_CHECKSEQUENCEVERIFY: 1 byte (option_anchors)
        - OP_DROP: 1 byte (option_anchors)
        - OP_ENDIF: 1 byte

    offered_htlc_script: 133 bytes (option_anchors を使用すると 136 bytes)
        - OP_DUP: 1 byte
        - OP_HASH160: 1 byte
        - OP_DATA: 1 byte (RIPEMD160(SHA256(revocationpubkey)) の長さ)
        - RIPEMD160(SHA256(revocationpubkey)): 20 bytes
        - OP_EQUAL: 1 byte
        - OP_IF: 1 byte
        - OP_CHECKSIG: 1 byte
        - OP_ELSE: 1 byte
        - OP_DATA: 1 byte (remotepubkey の長さ)
        - remotepubkey: 33 bytes
        - OP_SWAP: 1 byte
        - OP_SIZE: 1 byte
        - OP_DATA: 1 byte (32 の長さ)
        - 32: 1 byte
        - OP_EQUAL: 1 byte
        - OP_NOTIF: 1 byte
        - OP_DROP: 1 byte
        - 2: 1 byte
        - OP_SWAP: 1 byte
        - OP_DATA: 1 byte (localpubkey の長さ)
        - localpubkey: 33 bytes
        - 2: 1 byte
        - OP_CHECKMULTISIG: 1 byte
        - OP_ELSE: 1 byte
        - OP_HASH160: 1 byte
        - OP_DATA: 1 byte (RIPEMD160(payment_hash) の長さ)
        - RIPEMD160(payment_hash): 20 bytes
        - OP_EQUALVERIFY: 1 byte
        - OP_CHECKSIG: 1 byte
        - OP_ENDIF: 1 byte
        - OP_1: 1 byte (option_anchors)
        - OP_CHECKSEQUENCEVERIFY: 1 byte (option_anchors)
        - OP_DROP: 1 byte (option_anchors)
        - OP_ENDIF: 1 byte


    timeout_witness: 285 バイト (option_anchors を使用すると 288 バイト)
		- number_of_witness_elements: 1 バイト
		- nil_length: 1 バイト
		- sig_alice_length: 1 バイト
		- sig_alice: 73 バイト
		- sig_bob_length: 1 バイト
		- sig_bob: 73 バイト
		- nil_length: 1 バイト
		- witness_script_length: 1 バイト
		- witness_script (offered_htlc_script)

    success_witness: 324 バイト (option_anchors を使用すると 327 バイト)
		- number_of_witness_elements: 1 バイト
		- nil_length: 1 バイト
		- sig_alice_length: 1 バイト
		- sig_alice: 73 バイト
		- sig_bob_length: 1 バイト
		- sig_bob: 73 バイト
		- preimage_length: 1 バイト
		- preimage: 32 バイト
		- witness_script_length: 1 バイト
		- witness_script (accepted_htlc_script)

    commitment_input: 41 バイト
		- previous_out_point: 36 バイト
			- hash: 32 バイト
			- index: 4 バイト
		- var_int: 1 バイト (script_sig length)
		- script_sig: 0 バイト
		- witness (success_witness または timeout_witness)
		- sequence: 4 バイト

    htlc_output: 43 バイト
		- value: 8 バイト
		- var_int: 1 バイト (pk_script length)
		- pk_script (p2wsh): 34 バイト

	htlc_transaction:
		- version: 4 バイト
		- witness_header <---- witness データの一部
		- count_tx_in: 1 バイト
		- tx_in: 41 バイト
			commitment_input
		- count_tx_out: 1 バイト
		- tx_out: 43
			htlc_output
		- lock_time: 4 バイト

非 witness データを 4 倍すると、重みは 376 になります。それに各ケースの witness データを加えると (HTLC-timeout の場合は 285 または 288 + 2、HTLC-success の場合は 324 または 327 + 2)、重みは次のようになります：

	663 (HTLC-timeout) (option_anchors を使用すると 666)
	703 (HTLC-success) (option_anchors を使用すると 706)
                - (実際には 702 と 705 ですが、歴史的な理由でこれらの数字を使用します)

# Appendix B: Funding Transaction Test Vectors

以下において：
 - *local* が資金提供者であると仮定します。
 - 秘密鍵は 32 バイトに末尾の 1 を加えた形式で表示します (Bitcoin の「圧縮」秘密鍵の慣習、つまり公開鍵が圧縮されている鍵)。
 - トランザクション署名はすべて RFC6979 (HMAC-SHA256 を使用) に基づいて決定的です。有効な署名は、明示的に述べられていない限り、関連するトランザクションのすべての入力と出力に署名しなければなりません (つまり、`SIGHASH_ALL` [signature hash](https://bitcoin.org/en/glossary/signature-hash) を使用して作成されなければなりません)。クライアントは署名をコンパクトエンコーディングで送信し、Bitcoin-script 形式では送信してはならないことに注意してください。したがって、署名ハッシュバイトは送信されません。

資金調達トランザクションの入力は、以下の最初の 2 ブロックを持つテストチェーンを使用して作成されました。2 番目のブロックには使用可能なコインベースが含まれています（このような P2PKH 入力は推奨されませんが、[BOLT #2](02-peer-protocol.md#the-funding_created-message) に詳述されているように、最も簡単な例を提供します）。

    Block 0 (genesis): 0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4adae5494dffff7f20020000000101000000010000000000000000000000000000000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac00000000
    Block 1: 0000002006226e46111a0b59caaf126043eb5bbf28c34f3a5e332a1fc7b2b73cf188910fadbb20ea41a8423ea937e76e8151636bf6093b70eaff942930d20576600521fdc30f9858ffff7f20000000000101000000010000000000000000000000000000000000000000000000000000000000000000ffffffff03510101ffffffff0100f2052a010000001976a9143ca33c2e4446f4a305f23c80df8ad1afdcf652f988ac00000000
    Block 1 coinbase transaction: 01000000010000000000000000000000000000000000000000000000000000000000000000ffffffff03510101ffffffff0100f2052a010000001976a9143ca33c2e4446f4a305f23c80df8ad1afdcf652f988ac00000000
    Block 1 coinbase privkey: 6bd078650fcee8444e4e09825227b801a1ca928debb750eb36e6d56124bb20e801
    # privkey in base58: cRCH7YNcarfvaiY1GWUKQrRGmoezvfAiqHtdRvxe16shzbd7LDMz
    # pubkey in base68: mm3aPLSv9fBrbS68JzurAMp4xGoddJ6pSf

資金調達トランザクションは、以下の公開鍵に支払われます。

    local_funding_pubkey: 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb
    remote_funding_pubkey: 030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c1
    # funding witness script = 5221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae

ファンディングトランザクションには単一の入力とおつりの出力があります（この場合、順序は BIP69 によって決定されます）：

    input txid: fd2105607605d2302994ffea703b09f66b6351816ee737a93e42a841ea20bbad
    input index: 0
    input satoshis: 5000000000
    funding satoshis: 10000000
    # funding witness script = 5221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae
    # feerate_per_kw: 15000
    change satoshis: 4989986080
    funding output: 0

生成されたファンディングトランザクションは次のとおりです：

    funding tx: 0200000001adbb20ea41a8423ea937e76e8151636bf6093b70eaff942930d20576600521fd000000006b48304502210090587b6201e166ad6af0227d3036a9454223d49a1f11839c1a362184340ef0240220577f7cd5cca78719405cbf1de7414ac027f0239ef6e214c90fcaab0454d84b3b012103535b32d5eb0a6ed0982a0479bbadc9868d9836f6ba94dd5a63be16d875069184ffffffff028096980000000000220020c015c4a6be010e21657068fc2e6a9d02b27ebe4d490a25846f7237f104d1a3cd20256d29010000001600143ca33c2e4446f4a305f23c80df8ad1afdcf652f900000000
    # txid: 8984484a580b825b9972d7adb15050b3ab624ccd731946b3eeddb92f4e7ef6be

# Appendix C: Commitment and HTLC Transaction Test Vectors

以下において：
 - *local* トランザクションが考慮されます。これは、すべての支払いが *local* に遅延することを意味します。
 - *local* がオープナーであると仮定します。
 - 秘密鍵は 32 バイトに末尾の 1 を加えたものとして表示されます（Bitcoin の「圧縮」秘密鍵の慣習、すなわち公開鍵が圧縮されている鍵）。
 - トランザクションの署名はすべて RFC6979 を使用して決定的です（HMAC-SHA256 を使用）。

まず、各テストベクトルの共通の基本パラメータを定義します。HTLC は最初の「HTLC のない単純なコミットメントトランザクション」テストでは使用されず、HTLC 5 と 6 は「同じ金額とプレイメージ」テストでのみ使用されます。

    funding_tx_id: 8984484a580b825b9972d7adb15050b3ab624ccd731946b3eeddb92f4e7ef6be
    funding_output_index: 0
    funding_amount_satoshi: 10000000
    commitment_number: 42
    local_delay: 144
    local_dust_limit_satoshi: 546
    htlc 0 direction: remote->local
    htlc 0 amount_msat: 1000000
    htlc 0 expiry: 500
    htlc 0 payment_preimage: 0000000000000000000000000000000000000000000000000000000000000000
    htlc 1 direction: remote->local
    htlc 1 amount_msat: 2000000
    htlc 1 expiry: 501
    htlc 1 payment_preimage: 0101010101010101010101010101010101010101010101010101010101010101
    htlc 2 direction: local->remote
    htlc 2 amount_msat: 2000000
    htlc 2 expiry: 502
    htlc 2 payment_preimage: 0202020202020202020202020202020202020202020202020202020202020202
    htlc 3 direction: local->remote
    htlc 3 amount_msat: 3000000
    htlc 3 expiry: 503
    htlc 3 payment_preimage: 0303030303030303030303030303030303030303030303030303030303030303
    htlc 4 direction: remote->local
    htlc 4 amount_msat: 4000000
    htlc 4 expiry: 504
    htlc 4 payment_preimage: 0404040404040404040404040404040404040404040404040404040404040404
    htlc 5 direction: local->remote
    htlc 5 amount_msat: 5000000
    htlc 5 expiry: 506
    htlc 5 payment_preimage: 0505050505050505050505050505050505050505050505050505050505050505
    htlc 6 direction: local->remote
    htlc 6 amount_msat: 5000001
    htlc 6 expiry: 505
    htlc 6 payment_preimage: 0505050505050505050505050505050505050505050505050505050505050505

<!-- テストベクターの値は、キー導出に従って導出されていますが、このテストには必須ではありません。完全性のためにここに含まれており、誰かがテストベクターを自分で再現したい場合に備えています：

INTERNAL: remote_funding_privkey: 1552dfba4f6cf29a62a0af13c8d6981d36d0ef8d61ba10fb0fe90da7634d7e1301
INTERNAL: local_payment_basepoint_secret: 111111111111111111111111111111111111111111111111111111111111111101
INTERNAL: remote_revocation_basepoint_secret: 222222222222222222222222222222222222222222222222222222222222222201
INTERNAL: local_delayed_payment_basepoint_secret: 333333333333333333333333333333333333333333333333333333333333333301
INTERNAL: remote_payment_basepoint_secret: 444444444444444444444444444444444444444444444444444444444444444401
x_local_per_commitment_secret: 1f1e1d1c1b1a191817161514131211100f0e0d0c0b0a0908070605040302010001
# From remote_revocation_basepoint_secret
INTERNAL: remote_revocation_basepoint: 02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
# From local_delayed_payment_basepoint_secret
INTERNAL: local_delayed_payment_basepoint: 023c72addb4fdf09af94f0c94d7fe92a386a7e70cf8a1d85916386bb2535c7b1b1
INTERNAL: local_per_commitment_point: 025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486
INTERNAL: remote_privkey: 8deba327a7cc6d638ab0eb025770400a6184afcba6713c210d8d10e199ff2fda01
# From local_delayed_payment_basepoint_secret, local_per_commitment_point and local_delayed_payment_basepoint
INTERNAL: local_delayed_privkey: adf3464ce9c2f230fd2582fda4c6965e4993ca5524e8c9580e3df0cf226981ad01
-->

コミットメント番号の隠蔽因子を導出するために使用されるポイントは以下の通りです：

    local_payment_basepoint: 034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    remote_payment_basepoint: 032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991
    # obscured commitment number = 0x2bb038521914 ^ 42

HTLC ベースポイントには同じ値を使用します：

    local_htlc_basepoint: 034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    remote_htlc_basepoint: 032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991


そして、トランザクションを作成するために必要な鍵は以下の通りです：

    local_funding_privkey: 30ff4956bbdd3222d44cc5e8a1261dab1e07957bdac5ae88fe3261ef321f374901
    local_funding_pubkey: 023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb
    remote_funding_pubkey: 030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c1
    local_privkey: bb13b121cdc357cd2e608b0aea294afca36e2b34cf958e2e6451a2f27469449101
    local_htlcpubkey: 030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e7
    remote_htlcpubkey: 0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b
    local_delayedpubkey: 03fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c
    local_revocation_pubkey: 0212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b19
    # funding wscript = 5221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae

そして、テストベクトル自体は以下の通りです：

    name: simple commitment tx with no HTLCs
    to_local_msat: 7000000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 15000
    # base commitment transaction fee = 10860
    # actual commitment transaction fee = 10860
    # to_local amount 6989140 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
    remote_signature = 3045022100c3127b33dcc741dd6b05b1e63cbd1a9a7d816f37af9b6756fa2376b056f032370220408b96279808fe57eb7e463710804cdf4f108388bc5cf722d8c848d2c7f9f3b0
    # local_signature = 30440220616210b2cc4d3afb601013c373bbd8aac54febd9f15400379a8cb65ce7deca60022034236c010991beb7ff770510561ae8dc885b8d38d1947248c38f2ae055647142
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8002c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e48454a56a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e04004730440220616210b2cc4d3afb601013c373bbd8aac54febd9f15400379a8cb65ce7deca60022034236c010991beb7ff770510561ae8dc885b8d38d1947248c38f2ae05564714201483045022100c3127b33dcc741dd6b05b1e63cbd1a9a7d816f37af9b6756fa2376b056f032370220408b96279808fe57eb7e463710804cdf4f108388bc5cf722d8c848d2c7f9f3b001475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

```markdown
name: すべての 5 つの HTLC がトリムされていないコミットメント tx (最小手数料率)
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 0
# 基本コミットメントトランザクション手数料 = 0
# 実際のコミットメントトランザクション手数料 = 0
# HTLC #2 提供額 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
# HTLC #3 提供額 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
# HTLC #0 受取額 1000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac6868
# HTLC #1 受取額 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
# HTLC #4 受取額 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
# to_local 金額 6988000 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
# to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 3044022009b048187705a8cbc9ad73adbe5af148c3d012e1f067961486c822c7af08158c022006d66f3704cfab3eb2dc49dae24e4aa22a6910fc9b424007583204e3621af2e5
# local_signature = 304402206fc2d1f10ea59951eefac0b4b7c396a3c3d87b71ff0b019796ef4535beaf36f902201765b0181e514d04f4c8ad75659d7037be26cdb3f8bb6f78fe61decef484c3ea
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8007e80300000000000022002052bfef0479d7b293c27e0f1eb294bea154c63a3294ef092c19af51409bce0e2ad007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484e0a06a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402206fc2d1f10ea59951eefac0b4b7c396a3c3d87b71ff0b019796ef4535beaf36f902201765b0181e514d04f4c8ad75659d7037be26cdb3f8bb6f78fe61decef484c3ea01473044022009b048187705a8cbc9ad73adbe5af148c3d012e1f067961486c822c7af08158c022006d66f3704cfab3eb2dc49dae24e4aa22a6910fc9b424007583204e3621af2e501475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 5
# 出力 #0 の署名 (htlc-success for htlc #0)
remote_htlc_signature = 3045022100d9e29616b8f3959f1d3d7f7ce893ffedcdc407717d0de8e37d808c91d3a7c50d022078c3033f6d00095c8720a4bc943c1b45727818c082e4e3ddbc6d3116435b624b
# local_htlc_signature = 30440220636de5682ef0c5b61f124ec74e8aa2461a69777521d6998295dcea36bc3338110220165285594b23c50b28b82df200234566628a27bcd17f7f14404bd865354eb3ce
htlc_success_tx (htlc #0): 02000000000101ab84ff284f162cfbfef241f853b47d4368d171f9e2a1445160cd591c4c7d882b00000000000000000001e8030000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100d9e29616b8f3959f1d3d7f7ce893ffedcdc407717d0de8e37d808c91d3a7c50d022078c3033f6d00095c8720a4bc943c1b45727818c082e4e3ddbc6d3116435b624b014730440220636de5682ef0c5b61f124ec74e8aa2461a69777521d6998295dcea36bc3338110220165285594b23c50b28b82df200234566628a27bcd17f7f14404bd865354eb3ce012000000000000000000000000000000000000000000000000000000000000000008a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac686800000000
# 出力 #1 の署名 (htlc-timeout for htlc #2)
remote_htlc_signature = 30440220649fe8b20e67e46cbb0d09b4acea87dbec001b39b08dee7bdd0b1f03922a8640022037c462dff79df501cecfdb12ea7f4de91f99230bb544726f6e04527b1f896004
# local_htlc_signature = 3045022100803159dee7935dba4a1d36a61055ce8fd62caa528573cc221ae288515405a252022029c59e7cffce374fe860100a4a63787e105c3cf5156d40b12dd53ff55ac8cf3f
htlc_timeout_tx (htlc #2): 02000000000101ab84ff284f162cfbfef241f853b47d4368d171f9e2a1445160cd591c4c7d882b01000000000000000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220649fe8b20e67e46cbb0d09b4acea87dbec001b39b08dee7bdd0b1f03922a8640022037c462dff79df501cecfdb12ea7f4de91f99230bb544726f6e04527b1f89600401483045022100803159dee7935dba4a1d36a61055ce8fd62caa528573cc221ae288515405a252022029c59e7cffce374fe860100a4a63787e105c3cf5156d40b12dd53ff55ac8cf3f01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
# 出力 #2 の署名 (htlc-success for htlc #1)
remote_htlc_signature = 30440220770fc321e97a19f38985f2e7732dd9fe08d16a2efa4bcbc0429400a447faf49102204d40b417f3113e1b0944ae0986f517564ab4acd3d190503faf97a6e420d43352
# local_htlc_signature = 3045022100a437cc2ce77400ecde441b3398fea3c3ad8bdad8132be818227fe3c5b8345989022069d45e7fa0ae551ec37240845e2c561ceb2567eacf3076a6a43a502d05865faa
htlc_success_tx (htlc #1): 02000000000101ab84ff284f162cfbfef241f853b47d4368d171f9e2a1445160cd591c4c7d882b02000000000000000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220770fc321e97a19f38985f2e7732dd9fe08d16a2efa4bcbc0429400a447faf49102204d40b417f3113e1b0944ae0986f517564ab4acd3d190503faf97a6e420d4335201483045022100a437cc2ce77400ecde441b3398fea3c3ad8bdad8132be818227fe3c5b8345989022069d45e7fa0ae551ec37240845e2c561ceb2567eacf3076a6a43a502d05865faa012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
# 出力 #3 の署名 (htlc-timeout for htlc #3)
remote_htlc_signature = 304402207bcbf4f60a9829b05d2dbab84ed593e0291836be715dc7db6b72a64caf646af802201e489a5a84f7c5cc130398b841d138d031a5137ac8f4c49c770a4959dc3c1363
# local_htlc_signature = 304402203121d9b9c055f354304b016a36662ee99e1110d9501cb271b087ddb6f382c2c80220549882f3f3b78d9c492de47543cb9a697cecc493174726146536c5954dac7487
htlc_timeout_tx (htlc #3): 02000000000101ab84ff284f162cfbfef241f853b47d4368d171f9e2a1445160cd591c4c7d882b03000000000000000001b80b0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402207bcbf4f60a9829b05d2dbab84ed593e0291836be715dc7db6b72a64caf646af802201e489a5a84f7c5cc130398b841d138d031a5137ac8f4c49c770a4959dc3c13630147304402203121d9b9c055f354304b016a36662ee99e1110d9501cb271b087ddb6f382c2c80220549882f3f3b78d9c492de47543cb9a697cecc493174726146536c5954dac748701008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
# 出力 #4 の署名 (htlc-success for htlc #4)
remote_htlc_signature = 3044022076dca5cb81ba7e466e349b7128cdba216d4d01659e29b96025b9524aaf0d1899022060de85697b88b21c749702b7d2cfa7dfeaa1f472c8f1d7d9c23f2bf968464b87
# local_htlc_signature = 3045022100d9080f103cc92bac15ec42464a95f070c7fb6925014e673ee2ea1374d36a7f7502200c65294d22eb20d48564954d5afe04a385551919d8b2ddb4ae2459daaeee1d95
htlc_success_tx (htlc #4): 02000000000101ab84ff284f162cfbfef241f853b47d4368d171f9e2a1445160cd591c4c7d882b04000000000000000001a00f0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022076dca5cb81ba7e466e349b7128cdba216d4d01659e29b96025b9524aaf0d1899022060de85697b88b21c749702b7d2cfa7dfeaa1f472c8f1d7d9c23f2bf968464b8701483045022100d9080f103cc92bac15ec42464a95f070c7fb6925014e673ee2ea1374d36a7f7502200c65294d22eb20d48564954d5afe04a385551919d8b2ddb4ae2459daaeee1d95012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000
```

```
name: commitment tx with seven outputs untrimmed (maximum feerate)
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 647
# base commitment transaction fee = 1024
# actual commitment transaction fee = 1024
# HTLC #2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
# HTLC #3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
# HTLC #0 received amount 1000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac6868
# HTLC #1 received amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
# HTLC #4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
# to_local amount 6986976 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
# to_remote amount 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 3045022100a135f9e8a5ed25f7277446c67956b00ce6f610ead2bdec2c2f686155b7814772022059f1f6e1a8b336a68efcc1af3fe4d422d4827332b5b067501b099c47b7b5b5ee
# local_signature = 30450221009ec15c687898bb4da8b3a833e5ab8bfc51ec6e9202aaa8e66611edfd4a85ed1102203d7183e45078b9735c93450bc3415d3e5a8c576141a711ec6ddcb4a893926bb7
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8007e80300000000000022002052bfef0479d7b293c27e0f1eb294bea154c63a3294ef092c19af51409bce0e2ad007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484e09c6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e04004830450221009ec15c687898bb4da8b3a833e5ab8bfc51ec6e9202aaa8e66611edfd4a85ed1102203d7183e45078b9735c93450bc3415d3e5a8c576141a711ec6ddcb4a893926bb701483045022100a135f9e8a5ed25f7277446c67956b00ce6f610ead2bdec2c2f686155b7814772022059f1f6e1a8b336a68efcc1af3fe4d422d4827332b5b067501b099c47b7b5b5ee01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 5
# signature for output #0 (htlc-success for htlc #0)
remote_htlc_signature = 30450221008437627f9ad84ac67052e2a414a4367b8556fd1f94d8b02590f89f50525cd33502205b9c21ff6e7fc864f2352746ad8ba59182510819acb644e25b8a12fc37bbf24f
# local_htlc_signature = 30440220344b0deb055230d01703e6c7acd45853c4af2328b49b5d8af4f88a060733406602202ea64f2a43d5751edfe75503cbc35a62e3141b5ed032fa03360faf4ca66f670b
htlc_success_tx (htlc #0): 020000000001012cfb3e4788c206881d38f2996b6cb2109b5935acb527d14bdaa7b908afa9b2fe0000000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004830450221008437627f9ad84ac67052e2a414a4367b8556fd1f94d8b02590f89f50525cd33502205b9c21ff6e7fc864f2352746ad8ba59182510819acb644e25b8a12fc37bbf24f014730440220344b0deb055230d01703e6c7acd45853c4af2328b49b5d8af4f88a060733406602202ea64f2a43d5751edfe75503cbc35a62e3141b5ed032fa03360faf4ca66f670b012000000000000000000000000000000000000000000000000000000000000000008a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac686800000000
# signature for output #1 (htlc-timeout for htlc #2)
remote_htlc_signature = 304402205a67f92bf6845cf2892b48d874ac1daf88a36495cf8a06f93d83180d930a6f75022031da1621d95c3f335cc06a3056cf960199dae600b7cf89088f65fc53cdbef28c
# local_htlc_signature = 30450221009e5e3822b0185c6799a95288c597b671d6cc69ab80f43740f00c6c3d0752bdda02206da947a74bd98f3175324dc56fdba86cc783703a120a6f0297537e60632f4c7f
htlc_timeout_tx (htlc #2): 020000000001012cfb3e4788c206881d38f2996b6cb2109b5935acb527d14bdaa7b908afa9b2fe0100000000000000000124060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402205a67f92bf6845cf2892b48d874ac1daf88a36495cf8a06f93d83180d930a6f75022031da1621d95c3f335cc06a3056cf960199dae600b7cf89088f65fc53cdbef28c014830450221009e5e3822b0185c6799a95288c597b671d6cc69ab80f43740f00c6c3d0752bdda02206da947a74bd98f3175324dc56fdba86cc783703a120a6f0297537e60632f4c7f01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
# signature for output #2 (htlc-success for htlc #1)
remote_htlc_signature = 30440220437e21766054a3eef7f65690c5bcfa9920babbc5af92b819f772f6ea96df6c7402207173622024bd97328cfb26c6665e25c2f5d67c319443ccdc60c903217005d8c8
# local_htlc_signature = 3045022100fcfc47e36b712624677626cef3dc1d67f6583bd46926a6398fe6b00b0c9a37760220525788257b187fc775c6370d04eadf34d06f3650a63f8df851cee0ecb47a1673
htlc_success_tx (htlc #1): 020000000001012cfb3e4788c206881d38f2996b6cb2109b5935acb527d14bdaa7b908afa9b2fe020000000000000000010a060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220437e21766054a3eef7f65690c5bcfa9920babbc5af92b819f772f6ea96df6c7402207173622024bd97328cfb26c6665e25c2f5d67c319443ccdc60c903217005d8c801483045022100fcfc47e36b712624677626cef3dc1d67f6583bd46926a6398fe6b00b0c9a37760220525788257b187fc775c6370d04eadf34d06f3650a63f8df851cee0ecb47a1673012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
# signature for output #3 (htlc-timeout for htlc #3)
remote_htlc_signature = 304402207436e10737e4df499fc051686d3e11a5bb2310e4d1f1e691d287cef66514791202207cb58e71a6b7a42dd001b7e3ae672ea4f71ea3e1cd412b742e9124abb0739c64
# local_htlc_signature = 3045022100e78211b8409afb7255ffe37337da87f38646f1faebbdd61bc1920d69e3ead67a02201a626305adfcd16bfb7e9340928d9b6305464eab4aa4c4a3af6646e9b9f69dee
htlc_timeout_tx (htlc #3): 020000000001012cfb3e4788c206881d38f2996b6cb2109b5935acb527d14bdaa7b908afa9b2fe030000000000000000010c0a0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402207436e10737e4df499fc051686d3e11a5bb2310e4d1f1e691d287cef66514791202207cb58e71a6b7a42dd001b7e3ae672ea4f71ea3e1cd412b742e9124abb0739c6401483045022100e78211b8409afb7255ffe37337da87f38646f1faebbdd61bc1920d69e3ead67a02201a626305adfcd16bfb7e9340928d9b6305464eab4aa4c4a3af6646e9b9f69dee01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
# signature for output #4 (htlc-success for htlc #4)
remote_htlc_signature = 30450221009acd6a827a76bfee50806178dfe0495cd4e1d9c58279c194c7b01520fe68cb8d022024d439047c368883e570997a7d40f0b430cb5a742f507965e7d3063ae3feccca
# local_htlc_signature = 3044022048762cf546bbfe474f1536365ea7c416e3c0389d60558bc9412cb148fb6ab68202207215d7083b75c96ff9d2b08c59c34e287b66820f530b486a9aa4cdd9c347d5b9
htlc_success_tx (htlc #4): 020000000001012cfb3e4788c206881d38f2996b6cb2109b5935acb527d14bdaa7b908afa9b2fe04000000000000000001da0d0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004830450221009acd6a827a76bfee50806178dfe0495cd4e1d9c58279c194c7b01520fe68cb8d022024d439047c368883e570997a7d40f0b430cb5a742f507965e7d3063ae3feccca01473044022048762cf546bbfe474f1536365ea7c416e3c0389d60558bc9412cb148fb6ab68202207215d7083b75c96ff9d2b08c59c34e287b66820f530b486a9aa4cdd9c347d5b9012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000
```


    name: 6つの出力がトリムされていないコミットメントトランザクション (最小手数料率)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 648
    # 基本コミットメントトランザクション手数料 = 914
    # 実際のコミットメントトランザクション手数料 = 1914
    # HTLC #2 提供額 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
    # HTLC #3 提供額 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC #1 受領額 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
    # HTLC #4 受領額 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local 金額 6987086 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
    remote_signature = 304402203948f900a5506b8de36a4d8502f94f21dd84fd9c2314ab427d52feaa7a0a19f2022059b6a37a4adaa2c5419dc8aea63c6e2a2ec4c4bde46207f6dc1fcd22152fc6e5
    # local_signature = 3045022100b15f72908ba3382a34ca5b32519240a22300cc6015b6f9418635fb41f3d01d8802207adb331b9ed1575383dca0f2355e86c173802feecf8298fbea53b9d4610583e9
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8006d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e4844e9d6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100b15f72908ba3382a34ca5b32519240a22300cc6015b6f9418635fb41f3d01d8802207adb331b9ed1575383dca0f2355e86c173802feecf8298fbea53b9d4610583e90147304402203948f900a5506b8de36a4d8502f94f21dd84fd9c2314ab427d52feaa7a0a19f2022059b6a37a4adaa2c5419dc8aea63c6e2a2ec4c4bde46207f6dc1fcd22152fc6e501475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 4
    # 出力 #0 の署名 (htlc #2 の htlc-timeout)
    remote_htlc_signature = 3045022100a031202f3be94678f0e998622ee95ebb6ada8da1e9a5110228b5e04a747351e4022010ca6a21e18314ed53cfaae3b1f51998552a61a468e596368829a50ce40110e0
    # local_htlc_signature = 304502210097e1873b57267730154595187a34949d3744f52933070c74757005e61ce2112e02204ecfba2aa42d4f14bdf8bad4206bb97217b702e6c433e0e1b0ce6587e6d46ec6
    htlc_timeout_tx (htlc #2): 020000000001010f44041fdfba175987cf4e6135ba2a154e3b7fb96483dc0ed5efc0678e5b6bf10000000000000000000123060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100a031202f3be94678f0e998622ee95ebb6ada8da1e9a5110228b5e04a747351e4022010ca6a21e18314ed53cfaae3b1f51998552a61a468e596368829a50ce40110e00148304502210097e1873b57267730154595187a34949d3744f52933070c74757005e61ce2112e02204ecfba2aa42d4f14bdf8bad4206bb97217b702e6c433e0e1b0ce6587e6d46ec601008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
    # 出力 #1 の署名 (htlc #1 の htlc-success)
    remote_htlc_signature = 304402202361012a634aee7835c5ecdd6413dcffa8f404b7e77364c792cff984e4ee71e90220715c5e90baa08daa45a7439b1ee4fa4843ed77b19c058240b69406606d384124
    # local_htlc_signature = 3044022019de73b00f1d818fb388e83b2c8c31f6bce35ac624e215bc12f88f9dc33edf48022006ff814bb9f700ee6abc3294e146fac3efd4f13f0005236b41c0a946ee00c9ae
    htlc_success_tx (htlc #1): 020000000001010f44041fdfba175987cf4e6135ba2a154e3b7fb96483dc0ed5efc0678e5b6bf10100000000000000000109060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402202361012a634aee7835c5ecdd6413dcffa8f404b7e77364c792cff984e4ee71e90220715c5e90baa08daa45a7439b1ee4fa4843ed77b19c058240b69406606d38412401473044022019de73b00f1d818fb388e83b2c8c31f6bce35ac624e215bc12f88f9dc33edf48022006ff814bb9f700ee6abc3294e146fac3efd4f13f0005236b41c0a946ee00c9ae012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
    # 出力 #2 の署名 (htlc #3 の htlc-timeout)
    remote_htlc_signature = 304402207e8e82cd71ed4febeb593732c260456836e97d81896153ecd2b3cf320ca6861702202dd4a30f68f98ced7cc56a36369ac1fdd978248c5ff4ed204fc00cc625532989
    # local_htlc_signature = 3045022100bd0be6100c4fd8f102ec220e1b053e4c4e2ecca25615490150007b40d314dc3902201a1e0ea266965b43164d9e6576f58fa6726d42883dd1c3996d2925c2e2260796
    htlc_timeout_tx (htlc #3): 020000000001010f44041fdfba175987cf4e6135ba2a154e3b7fb96483dc0ed5efc0678e5b6bf1020000000000000000010b0a0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402207e8e82cd71ed4febeb593732c260456836e97d81896153ecd2b3cf320ca6861702202dd4a30f68f98ced7cc56a36369ac1fdd978248c5ff4ed204fc00cc62553298901483045022100bd0be6100c4fd8f102ec220e1b053e4c4e2ecca25615490150007b40d314dc3902201a1e0ea266965b43164d9e6576f58fa6726d42883dd1c3996d2925c2e226079601008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # 出力 #3 の署名 (htlc #4 の htlc-success)
    remote_htlc_signature = 3044022024cd52e4198c8ae0e414a86d86b5a65ea7450f2eb4e783096736d93395eca5ce022078f0094745b45be4d4b2b04dd5978c9e66ba49109e5704403e84aaf5f387d6be
    # local_htlc_signature = 3045022100bbfb9d0a946d420807c86e985d636cceb16e71c3694ed186316251a00cbd807202207773223f9a337e145f64673825be9b30d07ef1542c82188b264bedcf7cda78c6
    htlc_success_tx (htlc #4): 020000000001010f44041fdfba175987cf4e6135ba2a154e3b7fb96483dc0ed5efc0678e5b6bf103000000000000000001d90d0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022024cd52e4198c8ae0e414a86d86b5a65ea7450f2eb4e783096736d93395eca5ce022078f0094745b45be4d4b2b04dd5978c9e66ba49109e5704403e84aaf5f387d6be01483045022100bbfb9d0a946d420807c86e985d636cceb16e71c3694ed186316251a00cbd807202207773223f9a337e145f64673825be9b30d07ef1542c82188b264bedcf7cda78c6012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

```
name: commitment tx with six outputs untrimmed (maximum feerate)
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 2069
# base commitment transaction fee = 2921
# actual commitment transaction fee = 3921
# HTLC #2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
# HTLC #3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
# HTLC #1 received amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
# HTLC #4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
# to_local amount 6985079 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
# to_remote amount 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 304502210090b96a2498ce0c0f2fadbec2aab278fed54c1a7838df793ec4d2c78d96ec096202204fdd439c50f90d483baa7b68feeef4bd33bc277695405447bcd0bfb2ca34d7bc
# local_signature = 3045022100ad9a9bbbb75d506ca3b716b336ee3cf975dd7834fcf129d7dd188146eb58a8b4022061a759ee417339f7fe2ea1e8deb83abb6a74db31a09b7648a932a639cda23e33
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8006d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e48477956a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100ad9a9bbbb75d506ca3b716b336ee3cf975dd7834fcf129d7dd188146eb58a8b4022061a759ee417339f7fe2ea1e8deb83abb6a74db31a09b7648a932a639cda23e330148304502210090b96a2498ce0c0f2fadbec2aab278fed54c1a7838df793ec4d2c78d96ec096202204fdd439c50f90d483baa7b68feeef4bd33bc277695405447bcd0bfb2ca34d7bc01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 4
# signature for output #0 (htlc-timeout for htlc #2)
remote_htlc_signature = 3045022100f33513ee38abf1c582876f921f8fddc06acff48e04515532a32d3938de938ffd02203aa308a2c1863b7d6fdf53159a1465bf2e115c13152546cc5d74483ceaa7f699
# local_htlc_signature = 3045022100a637902a5d4c9ba9e7c472a225337d5aac9e2e3f6744f76e237132e7619ba0400220035c60d784a031c0d9f6df66b7eab8726a5c25397399ee4aa960842059eb3f9d
htlc_timeout_tx (htlc #2): 02000000000101adbe717a63fb658add30ada1e6e12ed257637581898abe475c11d7bbcd65bd4d0000000000000000000175020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100f33513ee38abf1c582876f921f8fddc06acff48e04515532a32d3938de938ffd02203aa308a2c1863b7d6fdf53159a1465bf2e115c13152546cc5d74483ceaa7f69901483045022100a637902a5d4c9ba9e7c472a225337d5aac9e2e3f6744f76e237132e7619ba0400220035c60d784a031c0d9f6df66b7eab8726a5c25397399ee4aa960842059eb3f9d01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
# signature for output #1 (htlc-success for htlc #1)
remote_htlc_signature = 3045022100ce07682cf4b90093c22dc2d9ab2a77ad6803526b655ef857221cc96af5c9e0bf02200f501cee22e7a268af40b555d15a8237c9f36ad67ef1841daf9f6a0267b1e6df
# local_htlc_signature = 3045022100e57e46234f8782d3ff7aa593b4f7446fb5316c842e693dc63ee324fd49f6a1c302204a2f7b44c48bd26e1554422afae13153eb94b29d3687b733d18930615fb2db61
htlc_success_tx (htlc #1): 02000000000101adbe717a63fb658add30ada1e6e12ed257637581898abe475c11d7bbcd65bd4d0100000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100ce07682cf4b90093c22dc2d9ab2a77ad6803526b655ef857221cc96af5c9e0bf02200f501cee22e7a268af40b555d15a8237c9f36ad67ef1841daf9f6a0267b1e6df01483045022100e57e46234f8782d3ff7aa593b4f7446fb5316c842e693dc63ee324fd49f6a1c302204a2f7b44c48bd26e1554422afae13153eb94b29d3687b733d18930615fb2db61012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
# signature for output #2 (htlc-timeout for htlc #3)
remote_htlc_signature = 3045022100e3e35492e55f82ec0bc2f317ffd7a486d1f7024330fe9743c3559fc39f32ef0c02203d1d4db651fc388a91d5ad8ecdd8e83673063bc8eefe27cfd8c189090e3a23e0
# local_htlc_signature = 3044022068613fb1b98eb3aec7f44c5b115b12343c2f066c4277c82b5f873dfe68f37f50022028109b4650f3f528ca4bfe9a467aff2e3e43893b61b5159157119d5d95cf1c18
htlc_timeout_tx (htlc #3): 02000000000101adbe717a63fb658add30ada1e6e12ed257637581898abe475c11d7bbcd65bd4d020000000000000000015d060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100e3e35492e55f82ec0bc2f317ffd7a486d1f7024330fe9743c3559fc39f32ef0c02203d1d4db651fc388a91d5ad8ecdd8e83673063bc8eefe27cfd8c189090e3a23e001473044022068613fb1b98eb3aec7f44c5b115b12343c2f066c4277c82b5f873dfe68f37f50022028109b4650f3f528ca4bfe9a467aff2e3e43893b61b5159157119d5d95cf1c1801008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
# signature for output #3 (htlc-success for htlc #4)
remote_htlc_signature = 304402207475aeb0212ef9bf5130b60937817ad88c9a87976988ef1f323f026148cc4a850220739fea17ad3257dcad72e509c73eebe86bee30b178467b9fdab213d631b109df
# local_htlc_signature = 3045022100d315522e09e7d53d2a659a79cb67fef56d6c4bddf3f46df6772d0d20a7beb7c8022070bcc17e288607b6a72be0bd83368bb6d53488db266c1cdb4d72214e4f02ac33
htlc_success_tx (htlc #4): 02000000000101adbe717a63fb658add30ada1e6e12ed257637581898abe475c11d7bbcd65bd4d03000000000000000001f2090000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402207475aeb0212ef9bf5130b60937817ad88c9a87976988ef1f323f026148cc4a850220739fea17ad3257dcad72e509c73eebe86bee30b178467b9fdab213d631b109df01483045022100d315522e09e7d53d2a659a79cb67fef56d6c4bddf3f46df6772d0d20a7beb7c8022070bcc17e288607b6a72be0bd83368bb6d53488db266c1cdb4d72214e4f02ac33012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000
```

```markdown
name: 5つの出力を持つコミットメントトランザクション（トリムなし、最小手数料率）
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 2070
# 基本コミットメントトランザクション手数料 = 2566
# 実際のコミットメントトランザクション手数料 = 5566
# HTLC #2 提供額 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
# HTLC #3 提供額 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
# HTLC #4 受領額 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
# to_local 金額 6985434 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
# to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 304402204ca1ba260dee913d318271d86e10ca0f5883026fb5653155cff600fb40895223022037b145204b7054a40e08bb1fefbd826f827b40838d3e501423bcc57924bcb50c
# local_signature = 3044022001014419b5ba00e083ac4e0a85f19afc848aacac2d483b4b525d15e2ae5adbfe022015ebddad6ee1e72b47cb09f3e78459da5be01ccccd95dceca0e056a00cc773c1
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8005d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5b80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484da966a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400473044022001014419b5ba00e083ac4e0a85f19afc848aacac2d483b4b525d15e2ae5adbfe022015ebddad6ee1e72b47cb09f3e78459da5be01ccccd95dceca0e056a00cc773c10147304402204ca1ba260dee913d318271d86e10ca0f5883026fb5653155cff600fb40895223022037b145204b7054a40e08bb1fefbd826f827b40838d3e501423bcc57924bcb50c01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 3
# 出力 #0 の署名 (htlc #2 の htlc-timeout)
remote_htlc_signature = 304402205f6b6d12d8d2529fb24f4445630566cf4abbd0f9330ab6c2bdb94222d6a2a0c502202f556258ae6f05b193749e4c541dfcc13b525a5422f6291f073f15617ba8579b
# local_htlc_signature = 30440220150b11069454da70caf2492ded9e0065c9a57f25ac2a4c52657b1d15b6c6ed85022068a38833b603c8892717206383611bad210f1cbb4b1f87ea29c6c65b9e1cb3e5
htlc_timeout_tx (htlc #2): 02000000000101403ad7602b43293497a3a2235a12ecefda4f3a1f1d06e49b1786d945685de1ff0000000000000000000174020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402205f6b6d12d8d2529fb24f4445630566cf4abbd0f9330ab6c2bdb94222d6a2a0c502202f556258ae6f05b193749e4c541dfcc13b525a5422f6291f073f15617ba8579b014730440220150b11069454da70caf2492ded9e0065c9a57f25ac2a4c52657b1d15b6c6ed85022068a38833b603c8892717206383611bad210f1cbb4b1f87ea29c6c65b9e1cb3e501008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
# 出力 #1 の署名 (htlc #3 の htlc-timeout)
remote_htlc_signature = 3045022100f960dfb1c9aee7ce1437efa65b523e399383e8149790e05d8fed27ff6e42fe0002202fe8613e062ffe0b0c518cc4101fba1c6de70f64a5bcc7ae663f2efae43b8546
# local_htlc_signature = 30450221009a6ed18e6873bc3644332a6ee21c152a5b102821865350df7a8c74451a51f9f2022050d801fb4895d7d7fbf452824c0168347f5c0cbe821cf6a97a63af5b8b2563c6
htlc_timeout_tx (htlc #3): 02000000000101403ad7602b43293497a3a2235a12ecefda4f3a1f1d06e49b1786d945685de1ff010000000000000000015c060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100f960dfb1c9aee7ce1437efa65b523e399383e8149790e05d8fed27ff6e42fe0002202fe8613e062ffe0b0c518cc4101fba1c6de70f64a5bcc7ae663f2efae43b8546014830450221009a6ed18e6873bc3644332a6ee21c152a5b102821865350df7a8c74451a51f9f2022050d801fb4895d7d7fbf452824c0168347f5c0cbe821cf6a97a63af5b8b2563c601008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
# 出力 #2 の署名 (htlc #4 の htlc-success)
remote_htlc_signature = 3045022100ae5fc7717ae684bc1fcf9020854e5dbe9842c9e7472879ac06ff95ac2bb10e4e022057728ada4c00083a3e65493fb5d50a232165948a1a0f530ef63185c2c8c56504
# local_htlc_signature = 30440220408ad3009827a8fccf774cb285587686bfb2ed041f89a89453c311ce9c8ee0f902203c7392d9f8306d3a46522a66bd2723a7eb2628cb2d9b34d4c104f1766bf37502
htlc_success_tx (htlc #4): 02000000000101403ad7602b43293497a3a2235a12ecefda4f3a1f1d06e49b1786d945685de1ff02000000000000000001f1090000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100ae5fc7717ae684bc1fcf9020854e5dbe9842c9e7472879ac06ff95ac2bb10e4e022057728ada4c00083a3e65493fb5d50a232165948a1a0f530ef63185c2c8c56504014730440220408ad3009827a8fccf774cb285587686bfb2ed041f89a89453c311ce9c8ee0f902203c7392d9f8306d3a46522a66bd2723a7eb2628cb2d9b34d4c104f1766bf37502012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000
```

```markdown
name: commitment tx with five outputs untrimmed (maximum feerate)
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 2194
# base commitment transaction fee = 2720
# actual commitment transaction fee = 5720
# HTLC #2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
# HTLC #3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
# HTLC #4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
# to_local amount 6985280 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
# to_remote amount 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 304402204bb3d6e279d71d9da414c82de42f1f954267c762b2e2eb8b76bc3be4ea07d4b0022014febc009c5edc8c3fc5d94015de163200f780046f1c293bfed8568f08b70fb3
# local_signature = 3044022072c2e2b1c899b2242656a537dde2892fa3801be0d6df0a87836c550137acde8302201654aa1974d37a829083c3ba15088689f30b56d6a4f6cb14c7bad0ee3116d398
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8005d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5b80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e48440966a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400473044022072c2e2b1c899b2242656a537dde2892fa3801be0d6df0a87836c550137acde8302201654aa1974d37a829083c3ba15088689f30b56d6a4f6cb14c7bad0ee3116d3980147304402204bb3d6e279d71d9da414c82de42f1f954267c762b2e2eb8b76bc3be4ea07d4b0022014febc009c5edc8c3fc5d94015de163200f780046f1c293bfed8568f08b70fb301475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 3
# signature for output #0 (htlc-timeout for htlc #2)
remote_htlc_signature = 3045022100939726680351a7856c1bc386d4a1f422c7d29bd7b56afc139570f508474e6c40022023175a799ccf44c017fbaadb924c40b2a12115a5b7d0dfd3228df803a2de8450
# local_htlc_signature = 304502210099c98c2edeeee6ec0fb5f3bea8b79bb016a2717afa9b5072370f34382de281d302206f5e2980a995e045cf90a547f0752a7ee99d48547bc135258fe7bc07e0154301
htlc_timeout_tx (htlc #2): 02000000000101153cd825fdb3aa624bfe513e8031d5d08c5e582fb3d1d1fe8faf27d3eed410cd0000000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100939726680351a7856c1bc386d4a1f422c7d29bd7b56afc139570f508474e6c40022023175a799ccf44c017fbaadb924c40b2a12115a5b7d0dfd3228df803a2de84500148304502210099c98c2edeeee6ec0fb5f3bea8b79bb016a2717afa9b5072370f34382de281d302206f5e2980a995e045cf90a547f0752a7ee99d48547bc135258fe7bc07e015430101008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
# signature for output #1 (htlc-timeout for htlc #3)
remote_htlc_signature = 3044022021bb883bf324553d085ba2e821cad80c28ef8b303dbead8f98e548783c02d1600220638f9ef2a9bba25869afc923f4b5dc38be3bb459f9efa5d869392d5f7779a4a0
# local_htlc_signature = 3045022100fd85bd7697b89c08ec12acc8ba89b23090637d83abd26ca37e01ae93e67c367302202b551fe69386116c47f984aab9c8dfd25d864dcde5d3389cfbef2447a85c4b77
htlc_timeout_tx (htlc #3): 02000000000101153cd825fdb3aa624bfe513e8031d5d08c5e582fb3d1d1fe8faf27d3eed410cd010000000000000000010a060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022021bb883bf324553d085ba2e821cad80c28ef8b303dbead8f98e548783c02d1600220638f9ef2a9bba25869afc923f4b5dc38be3bb459f9efa5d869392d5f7779a4a001483045022100fd85bd7697b89c08ec12acc8ba89b23090637d83abd26ca37e01ae93e67c367302202b551fe69386116c47f984aab9c8dfd25d864dcde5d3389cfbef2447a85c4b7701008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
# signature for output #2 (htlc-success for htlc #4)
remote_htlc_signature = 3045022100c9e6f0454aa598b905a35e641a70cc9f67b5f38cc4b00843a041238c4a9f1c4a0220260a2822a62da97e44583e837245995ca2e36781769c52f19e498efbdcca262b
# local_htlc_signature = 30450221008a9f2ea24cd455c2b64c1472a5fa83865b0a5f49a62b661801e884cf2849af8302204d44180e50bf6adfcf1c1e581d75af91aba4e28681ce4a5ee5f3cbf65eca10f3
htlc_success_tx (htlc #4): 02000000000101153cd825fdb3aa624bfe513e8031d5d08c5e582fb3d1d1fe8faf27d3eed410cd020000000000000000019a090000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100c9e6f0454aa598b905a35e641a70cc9f67b5f38cc4b00843a041238c4a9f1c4a0220260a2822a62da97e44583e837245995ca2e36781769c52f19e498efbdcca262b014830450221008a9f2ea24cd455c2b64c1472a5fa83865b0a5f49a62b661801e884cf2849af8302204d44180e50bf6adfcf1c1e581d75af91aba4e28681ce4a5ee5f3cbf65eca10f3012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000
```

申し訳ありませんが、コードブロック内の内容は翻訳しないでください。その他の部分については、具体的なテキストを提供していただければ翻訳を行います。

申し訳ありませんが、コードブロック内の内容は翻訳しないというルールに従って、翻訳を行うことができません。コードブロックの外にあるテキストを提供していただければ、それを翻訳いたします。


    name: 3つの出力がトリムされていないコミットメントトランザクション (最小手数料率)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 3703
    # 基本コミットメントトランザクション手数料 = 3317
    # 実際のコミットメントトランザクション手数料 = 11317
    # HTLC #4 受取額 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local 金額 6984683 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
    remote_signature = 3045022100b495d239772a237ff2cf354b1b11be152fd852704cb184e7356d13f2fb1e5e430220723db5cdb9cbd6ead7bfd3deb419cf41053a932418cbb22a67b581f40bc1f13e
    # local_signature = 304402201b736d1773a124c745586217a75bed5f66c05716fbe8c7db4fdb3c3069741cdd02205083f39c321c1bcadfc8d97e3c791a66273d936abac0c6a2fde2ed46019508e1
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8003a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484eb936a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402201b736d1773a124c745586217a75bed5f66c05716fbe8c7db4fdb3c3069741cdd02205083f39c321c1bcadfc8d97e3c791a66273d936abac0c6a2fde2ed46019508e101483045022100b495d239772a237ff2cf354b1b11be152fd852704cb184e7356d13f2fb1e5e430220723db5cdb9cbd6ead7bfd3deb419cf41053a932418cbb22a67b581f40bc1f13e01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 1
    # 出力 #0 の署名 (htlc #4 の htlc-success)
    remote_htlc_signature = 3045022100c34c61735f93f2e324cc873c3b248111ccf8f6db15d5969583757010d4ad2b4602207867bb919b2ddd6387873e425345c9b7fd18d1d66aba41f3607bc2896ef3c30a
    # local_htlc_signature = 3045022100988c143e2110067117d2321bdd4bd16ca1734c98b29290d129384af0962b634e02206c1b02478878c5f547018b833986578f90c3e9be669fe5788ad0072a55acbb05
    htlc_success_tx (htlc #4): 0200000000010120060e4a29579d429f0f27c17ee5f1ee282f20d706d6f90b63d35946d8f3029a0000000000000000000175050000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100c34c61735f93f2e324cc873c3b248111ccf8f6db15d5969583757010d4ad2b4602207867bb919b2ddd6387873e425345c9b7fd18d1d66aba41f3607bc2896ef3c30a01483045022100988c143e2110067117d2321bdd4bd16ca1734c98b29290d129384af0962b634e02206c1b02478878c5f547018b833986578f90c3e9be669fe5788ad0072a55acbb05012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

```markdown
name: commitment tx with three outputs untrimmed (maximum feerate)
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 4914
# base commitment transaction fee = 4402
# actual commitment transaction fee = 12402
# HTLC #4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
# to_local amount 6983598 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
# to_remote amount 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 3045022100b4b16d5f8cc9fc4c1aff48831e832a0d8990e133978a66e302c133550954a44d022073573ce127e2200d316f6b612803a5c0c97b8d20e1e44dbe2ac0dd2fb8c95244
# local_signature = 3045022100d72638bc6308b88bb6d45861aae83e5b9ff6e10986546e13bce769c70036e2620220320be7c6d66d22f30b9fcd52af66531505b1310ca3b848c19285b38d8a1a8c19
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8003a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484ae8f6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100d72638bc6308b88bb6d45861aae83e5b9ff6e10986546e13bce769c70036e2620220320be7c6d66d22f30b9fcd52af66531505b1310ca3b848c19285b38d8a1a8c1901483045022100b4b16d5f8cc9fc4c1aff48831e832a0d8990e133978a66e302c133550954a44d022073573ce127e2200d316f6b612803a5c0c97b8d20e1e44dbe2ac0dd2fb8c9524401475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 1
# signature for output #0 (htlc-success for htlc #4)
remote_htlc_signature = 3045022100f43591c156038ba217756006bb3c55f7d113a325cdd7d9303c82115372858d68022016355b5aadf222bc8d12e426c75f4a03423917b2443a103eb2a498a3a2234374
# local_htlc_signature = 30440220585dee80fafa264beac535c3c0bb5838ac348b156fdc982f86adc08dfc9bfd250220130abb82f9f295cc9ef423dcfef772fde2acd85d9df48cc538981d26a10a9c10
htlc_success_tx (htlc #4): 02000000000101a9172908eace869cc35128c31fc2ab502f72e4dff31aab23e0244c4b04b11ab00000000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100f43591c156038ba217756006bb3c55f7d113a325cdd7d9303c82115372858d68022016355b5aadf222bc8d12e426c75f4a03423917b2443a103eb2a498a3a2234374014730440220585dee80fafa264beac535c3c0bb5838ac348b156fdc982f86adc08dfc9bfd250220130abb82f9f295cc9ef423dcfef772fde2acd85d9df48cc538981d26a10a9c10012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000
```


    name: 2つの出力がトリムされていないコミットメントトランザクション (最小手数料率)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 4915
    # 基本コミットメントトランザクション手数料 = 3558
    # 実際のコミットメントトランザクション手数料 = 15558
    # to_local 金額 6984442 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
    remote_signature = 304402203a286936e74870ca1459c700c71202af0381910a6bfab687ef494ef1bc3e02c902202506c362d0e3bee15e802aa729bf378e051644648253513f1c085b264cc2a720
    # local_signature = 30450221008a953551f4d67cb4df3037207fc082ddaf6be84d417b0bd14c80aab66f1b01a402207508796dc75034b2dee876fe01dc05a08b019f3e5d689ac8842ade2f1befccf5
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8002c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484fa926a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e04004830450221008a953551f4d67cb4df3037207fc082ddaf6be84d417b0bd14c80aab66f1b01a402207508796dc75034b2dee876fe01dc05a08b019f3e5d689ac8842ade2f1befccf50147304402203a286936e74870ca1459c700c71202af0381910a6bfab687ef494ef1bc3e02c902202506c362d0e3bee15e802aa729bf378e051644648253513f1c085b264cc2a72001475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

    name: 2つの出力がトリムされていないコミットメントトランザクション (最大手数料率)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 9651180
    # 基本コミットメントトランザクション手数料 = 6987454
    # 実際のコミットメントトランザクション手数料 = 6999454
    # to_local 金額 546 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
    remote_signature = 304402200a8544eba1d216f5c5e530597665fa9bec56943c0f66d98fc3d028df52d84f7002201e45fa5c6bc3a506cc2553e7d1c0043a9811313fc39c954692c0d47cfce2bbd3
    # local_signature = 3045022100e11b638c05c650c2f63a421d36ef8756c5ce82f2184278643520311cdf50aa200220259565fb9c8e4a87ccaf17f27a3b9ca4f20625754a0920d9c6c239d8156a11de
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b800222020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80ec0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e4840400483045022100e11b638c05c650c2f63a421d36ef8756c5ce82f2184278643520311cdf50aa200220259565fb9c8e4a87ccaf17f27a3b9ca4f20625754a0920d9c6c239d8156a11de0147304402200a8544eba1d216f5c5e530597665fa9bec56943c0f66d98fc3d028df52d84f7002201e45fa5c6bc3a506cc2553e7d1c0043a9811313fc39c954692c0d47cfce2bbd301475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

```markdown
name: 出力がトリムされていないコミットメントトランザクション (最小手数料率)
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 9651181
# 基本コミットメントトランザクション手数料 = 6987455
# 実際のコミットメントトランザクション手数料 = 7000000
# to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 304402202ade0142008309eb376736575ad58d03e5b115499709c6db0b46e36ff394b492022037b63d78d66404d6504d4c4ac13be346f3d1802928a6d3ad95a6a944227161a2
# local_signature = 304402207e8d51e0c570a5868a78414f4e0cbfaed1106b171b9581542c30718ee4eb95ba02203af84194c97adf98898c9afe2f2ed4a7f8dba05a2dfab28ac9d9c604aa49a379
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8001c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484040047304402207e8d51e0c570a5868a78414f4e0cbfaed1106b171b9581542c30718ee4eb95ba02203af84194c97adf98898c9afe2f2ed4a7f8dba05a2dfab28ac9d9c604aa49a3790147304402202ade0142008309eb376736575ad58d03e5b115499709c6db0b46e36ff394b492022037b63d78d66404d6504d4c4ac13be346f3d1802928a6d3ad95a6a944227161a201475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 0

name: 資金提供者の金額を超える手数料のコミットメントトランザクション
to_local_msat: 6988000000
to_remote_msat: 3000000000
local_feerate_per_kw: 9651936
# 基本コミットメントトランザクション手数料 = 6988001
# 実際のコミットメントトランザクション手数料 = 7000000
# to_remote 金額 3000000 P2WPKH(032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991)
remote_signature = 304402202ade0142008309eb376736575ad58d03e5b115499709c6db0b46e36ff394b492022037b63d78d66404d6504d4c4ac13be346f3d1802928a6d3ad95a6a944227161a2
# local_signature = 304402207e8d51e0c570a5868a78414f4e0cbfaed1106b171b9581542c30718ee4eb95ba02203af84194c97adf98898c9afe2f2ed4a7f8dba05a2dfab28ac9d9c604aa49a379
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8001c0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484040047304402207e8d51e0c570a5868a78414f4e0cbfaed1106b171b9581542c30718ee4eb95ba02203af84194c97adf98898c9afe2f2ed4a7f8dba05a2dfab28ac9d9c604aa49a3790147304402202ade0142008309eb376736575ad58d03e5b115499709c6db0b46e36ff394b492022037b63d78d66404d6504d4c4ac13be346f3d1802928a6d3ad95a6a944227161a201475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 0
```

```markdown
name: 3 つの HTLC 出力を持つコミットメントトランザクション、2 つのオファーが同じ金額とプレイメージを持つ
to_local_msat: 6987999999
to_remote_msat: 3000000000
local_feerate_per_kw: 253
# HTLC #1 受信額 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
# HTLC #5 提供額 5000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9142002cc93ebefbb1b73f0af055dcc27a0b504ad7688ac6868
# HTLC #6 提供額 5000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9142002cc93ebefbb1b73f0af055dcc27a0b504ad7688ac6868
# HTLC #5 と #6 はそれぞれ CLTV 506 と 505 を持ち、プレイメージ 0505050505050505050505050505050505050505050505050505050505050505
remote_signature = 304402207d0870964530f97b62497b11153c551dca0a1e226815ef0a336651158da0f82402200f5378beee0e77759147b8a0a284decd11bfd2bc55c8fafa41c134fe996d43c8
# local_signature = 304402200d10bf5bc5397fc59d7188ae438d80c77575595a2d488e41bd6363a810cc8d72022012b57e714fbbfdf7a28c47d5b370cb8ac37c8545f596216e5b21e9b236ef457c
output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8005d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2d8813000000000000220020305c12e1a0bc21e283c131cea1c66d68857d28b7b2fce0a6fbc40c164852121b8813000000000000220020305c12e1a0bc21e283c131cea1c66d68857d28b7b2fce0a6fbc40c164852121bc0c62d0000000000160014cc1b07838e387deacd0e5232e1e8b49f4c29e484a69f6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402200d10bf5bc5397fc59d7188ae438d80c77575595a2d488e41bd6363a810cc8d72022012b57e714fbbfdf7a28c47d5b370cb8ac37c8545f596216e5b21e9b236ef457c0147304402207d0870964530f97b62497b11153c551dca0a1e226815ef0a336651158da0f82402200f5378beee0e77759147b8a0a284decd11bfd2bc55c8fafa41c134fe996d43c801475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
num_htlcs: 3
# 出力 #0 の署名 (htlc-success for htlc #1)
remote_htlc_signature = 3045022100b470fe12e5b7fea9eccb8cbff1972cea4f96758041898982a02bcc7f9d56d50b0220338a75b2afaab4ec00cdd2d9273c68c7581ff5a28bcbb40c4d138b81f1d45ce5
# local_htlc_signature = 3044022017b90c65207522a907fb6a137f9dd528b3389465a8ae72308d9e1d564f512cf402204fc917b4f0e88604a3e994f85bfae7c7c1f9d9e9f78e8cd112e0889720d9405b
htlc_success_tx (htlc #1): 020000000001014bdccf28653066a2c554cafeffdfe1e678e64a69b056684deb0c4fba909423ec000000000000000000011f070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100b470fe12e5b7fea9eccb8cbff1972cea4f96758041898982a02bcc7f9d56d50b0220338a75b2afaab4ec00cdd2d9273c68c7581ff5a28bcbb40c4d138b81f1d45ce501473044022017b90c65207522a907fb6a137f9dd528b3389465a8ae72308d9e1d564f512cf402204fc917b4f0e88604a3e994f85bfae7c7c1f9d9e9f78e8cd112e0889720d9405b012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
# 出力 #1 の署名 (htlc-timeout for htlc #6)
remote_htlc_signature = 3045022100b575379f6d8743cb0087648f81cfd82d17a97fbf8f67e058c65ce8b9d25df9500220554a210d65b02d9f36c6adf0f639430ca8293196ba5089bf67cc3a9813b7b00a
# local_htlc_signature = 3045022100ee2e16b90930a479b13f8823a7f14b600198c838161160b9436ed086d3fc57e002202a66fa2324f342a17129949c640bfe934cbc73a869ba7c06aa25c5a3d0bfb53d
htlc_timeout_tx (htlc #6): 020000000001014bdccf28653066a2c554cafeffdfe1e678e64a69b056684deb0c4fba909423ec01000000000000000001e1120000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100b575379f6d8743cb0087648f81cfd82d17a97fbf8f67e058c65ce8b9d25df9500220554a210d65b02d9f36c6adf0f639430ca8293196ba5089bf67cc3a9813b7b00a01483045022100ee2e16b90930a479b13f8823a7f14b600198c838161160b9436ed086d3fc57e002202a66fa2324f342a17129949c640bfe934cbc73a869ba7c06aa25c5a3d0bfb53d01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9142002cc93ebefbb1b73f0af055dcc27a0b504ad7688ac6868f9010000
# 出力 #2 の署名 (htlc-timeout for htlc #5)
remote_htlc_signature = 30440220471c9f3ad92e49b13b7b8059f43ecf8f7887b0dccbb9fdb54bfe23d62a8ae332022024bd22fae0740e86a44228c35330da9526fd7306dffb2b9dc362d5e78abef7cc
# local_htlc_signature = 304402207157f452f2506d73c315192311893800cfb3cc235cc1185b1cfcc136b55230db022014be242dbc6c5da141fec4034e7f387f74d6ff1899453d72ba957467540e1ecb
htlc_timeout_tx (htlc #5): 020000000001014bdccf28653066a2c554cafeffdfe1e678e64a69b056684deb0c4fba909423ec02000000000000000001e1120000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220471c9f3ad92e49b13b7b8059f43ecf8f7887b0dccbb9fdb54bfe23d62a8ae332022024bd22fae0740e86a44228c35330da9526fd7306dffb2b9dc362d5e78abef7cc0147304402207157f452f2506d73c315192311893800cfb3cc235cc1185b1cfcc136b55230db022014be242dbc6c5da141fec4034e7f387f74d6ff1899453d72ba957467540e1ecb01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9142002cc93ebefbb1b73f0af055dcc27a0b504ad7688ac6868fa010000
```


# 付録 D: コミットメントごとの秘密生成テストベクトル

これらは、すべてのノードが使用する生成アルゴリズムをテストします。

## 生成テスト

    name: generate_from_seed 0 final node
	seed: 0x0000000000000000000000000000000000000000000000000000000000000000
	I: 281474976710655
	output: 0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148

	name: generate_from_seed FF final node
	seed: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
	I: 281474976710655
	output: 0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc

	name: generate_from_seed FF alternate bits 1
	seed: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
	I: 0xaaaaaaaaaaa
	output: 0x56f4008fb007ca9acf0e15b054d5c9fd12ee06cea347914ddbaed70d1c13a528

	name: generate_from_seed FF alternate bits 2
	seed: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
	I: 0x555555555555
	output: 0x9015daaeb06dba4ccc05b91b2f73bd54405f2be9f217fbacd3c5ac2e62327d31

	name: generate_from_seed 01 last nontrivial node
	seed: 0x0101010101010101010101010101010101010101010101010101010101010101
	I: 1
	output: 0x915c75942a26bb3a433a8ce2cb0427c29ec6c1775cfc78328b57f6ba7bfeaa9c

## ストレージテスト

これらはオプションのコンパクトストレージシステムをテストします。多くの場合、誤ったエントリはその親が明らかになるまで判別できません。エントリは特に破損しており、その子もすべて破損しています。

これらのテストでは `0xFFF...FF` のシードが使用され、誤ったエントリには `0x000...00` が使用されます。

    name: insert_secret correct sequence
	I: 281474976710655
	secret: 0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
	output: OK
	I: 281474976710654
	secret: 0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
	output: OK
	I: 281474976710653
	secret: 0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
	output: OK
	I: 281474976710652
	secret: 0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
	output: OK
	I: 281474976710651
	secret: 0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
	output: OK
	I: 281474976710650
	secret: 0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
	output: OK
	I: 281474976710649
	secret: 0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
	output: OK
	I: 281474976710648
	secret: 0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
	output: OK

```
name: insert_secret #1 incorrect
I: 281474976710655
secret: 0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148
output: OK
I: 281474976710654
secret: 0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
output: ERROR

name: insert_secret #2 incorrect (#1 derived from incorrect)
I: 281474976710655
secret: 0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148
output: OK
I: 281474976710654
secret: 0xdddc3a8d14fddf2b68fa8c7fbad2748274937479dd0f8930d5ebb4ab6bd866a3
output: OK
I: 281474976710653
secret: 0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
output: OK
I: 281474976710652
secret: 0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
output: ERROR

name: insert_secret #3 incorrect
I: 281474976710655
secret: 0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
output: OK
I: 281474976710654
secret: 0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
output: OK
I: 281474976710653
secret: 0xc51a18b13e8527e579ec56365482c62f180b7d5760b46e9477dae59e87ed423a
output: OK
I: 281474976710652
secret: 0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
output: ERROR

name: insert_secret #4 incorrect (1,2,3 derived from incorrect)
I: 281474976710655
secret: 0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148
output: OK
I: 281474976710654
secret: 0xdddc3a8d14fddf2b68fa8c7fbad2748274937479dd0f8930d5ebb4ab6bd866a3
output: OK
I: 281474976710653
secret: 0xc51a18b13e8527e579ec56365482c62f180b7d5760b46e9477dae59e87ed423a
output: OK
I: 281474976710652
secret: 0xba65d7b0ef55a3ba300d4e87af29868f394f8f138d78a7011669c79b37b936f4
output: OK
I: 281474976710651
secret: 0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
output: OK
I: 281474976710650
secret: 0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
output: OK
I: 281474976710649
secret: 0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
output: OK
I: 281474976710648
secret: 0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
output: ERROR
```


    name: insert_secret #5 incorrect
	I: 281474976710655
	secret: 0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
	output: OK
	I: 281474976710654
	secret: 0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
	output: OK
	I: 281474976710653
	secret: 0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
	output: OK
	I: 281474976710652
	secret: 0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
	output: OK
	I: 281474976710651
	secret: 0x631373ad5f9ef654bb3dade742d09504c567edd24320d2fcd68e3cc47e2ff6a6
	output: OK
	I: 281474976710650
	secret: 0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
	output: ERROR

    name: insert_secret #6 incorrect (5 derived from incorrect)
	I: 281474976710655
	secret: 0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
	output: OK
	I: 281474976710654
	secret: 0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
	output: OK
	I: 281474976710653
	secret: 0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
	output: OK
	I: 281474976710652
	secret: 0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
	output: OK
	I: 281474976710651
	secret: 0x631373ad5f9ef654bb3dade742d09504c567edd24320d2fcd68e3cc47e2ff6a6
	output: OK
	I: 281474976710650
	secret: 0xb7e76a83668bde38b373970155c868a653304308f9896692f904a23731224bb1
	output: OK
	I: 281474976710649
	secret: 0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
	output: OK
	I: 281474976710648
	secret: 0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
	output: ERROR

    name: insert_secret #7 incorrect
	I: 281474976710655
	secret: 0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
	output: OK
	I: 281474976710654
	secret: 0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
	output: OK
	I: 281474976710653
	secret: 0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
	output: OK
	I: 281474976710652
	secret: 0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
	output: OK
	I: 281474976710651
	secret: 0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
	output: OK
	I: 281474976710650
	secret: 0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
	output: OK
	I: 281474976710649
	secret: 0xe7971de736e01da8ed58b94c2fc216cb1dca9e326f3a96e7194fe8ea8af6c0a3
	output: OK
	I: 281474976710648
	secret: 0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
	output: ERROR


    name: insert_secret #8 incorrect
	I: 281474976710655
	secret: 0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
	output: OK
	I: 281474976710654
	secret: 0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
	output: OK
	I: 281474976710653
	secret: 0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
	output: OK
	I: 281474976710652
	secret: 0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
	output: OK
	I: 281474976710651
	secret: 0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
	output: OK
	I: 281474976710650
	secret: 0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
	output: OK
	I: 281474976710649
	secret: 0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
	output: OK
	I: 281474976710648
	secret: 0xa7efbc61aac46d34f77778bac22c8a20c6a46ca460addc49009bda875ec88fa4
	output: ERROR

# Appendix E: Key Derivation Test Vectors

これらは `localpubkey`、`remotepubkey`、`local_htlcpubkey`、`remote_htlcpubkey`、`local_delayedpubkey`、および `remote_delayedpubkey`（同じ式を使用します）の導出をテストします。また、`revocationpubkey` も含まれます。

すべて以下の秘密（および導出されたポイント）を使用します：

    base_secret: 0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
    per_commitment_secret: 0x1f1e1d1c1b1a191817161514131211100f0e0d0c0b0a09080706050403020100
    base_point: 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2
    per_commitment_point: 0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486

    name: derivation of pubkey from basepoint and per_commitment_point
    # SHA256(per_commitment_point || basepoint)
    # => SHA256(0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2)
    # = 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
    # + basepoint (0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2)
    # = 0x0235f2dbfaa89b57ec7b055afe29849ef7ddfeb1cefdb9ebdc43f5494984db29e5
    localpubkey: 0x0235f2dbfaa89b57ec7b055afe29849ef7ddfeb1cefdb9ebdc43f5494984db29e5

```markdown
name: basepoint secret と per_commitment_secret からのプライベートキーの導出
# SHA256(per_commitment_point || basepoint)
# => SHA256(0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2)
# = 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
# + basepoint_secret (0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f)
# = 0xcbced912d3b21bf196a766651e436aff192362621ce317704ea2f75d87e7be0f
localprivkey: 0xcbced912d3b21bf196a766651e436aff192362621ce317704ea2f75d87e7be0f

name: basepoint と per_commitment_point からの取り消し公開鍵の導出
# SHA256(revocation_basepoint || per_commitment_point)
# => SHA256(0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2 || 0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486)
# = 0xefbf7ba5a074276701798376950a64a90f698997cce0dff4d24a6d2785d20963
# x revocation_basepoint = 0x02c00c4aadc536290422a807250824a8d87f19d18da9d610d45621df22510db8ce
# SHA256(per_commitment_point || revocation_basepoint)
# => SHA256(0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2)
# = 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
# x per_commitment_point = 0x0325ee7d3323ce52c4b33d4e0a73ab637711057dd8866e3b51202a04112f054c43
# 0x02c00c4aadc536290422a807250824a8d87f19d18da9d610d45621df22510db8ce + 0x0325ee7d3323ce52c4b33d4e0a73ab637711057dd8866e3b51202a04112f054c43 => 0x02916e326636d19c33f13e8c0c3a03dd157f332f3e99c317c141dd865eb01f8ff0
revocationpubkey: 0x02916e326636d19c33f13e8c0c3a03dd157f332f3e99c317c141dd865eb01f8ff0

name: basepoint_secret と per_commitment_secret からの取り消し秘密鍵の導出
# SHA256(revocation_basepoint || per_commitment_point)
# => SHA256(0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2 || 0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486)
# = 0xefbf7ba5a074276701798376950a64a90f698997cce0dff4d24a6d2785d20963
# * revocation_basepoint_secret (0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f)# = 0x44bfd55f845f885b8e60b2dca4b30272d5343be048d79ce87879d9863dedc842
# SHA256(per_commitment_point || revocation_basepoint)
# => SHA256(0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2)
# = 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
# * per_commitment_secret (0x1f1e1d1c1b1a191817161514131211100f0e0d0c0b0a09080706050403020100)# = 0x8be02a96a97b9a3c1c9f59ebb718401128b72ec009d85ee1656319b52319b8ce
# => 0xd09ffff62ddb2297ab000cc85bcb4283fdeb6aa052affbc9dddcf33b61078110
revocationprivkey: 0xd09ffff62ddb2297ab000cc85bcb4283fdeb6aa052affbc9dddcf33b61078110
```


# Appendix F: コミットメントと HTLC トランザクションのテストベクトル (アンカー)

アンカーのテストベクトルは、付録 C で定義されているテストケースに基づいています。付録 C では、`to_local_msat` と `to_remote_msat` は以下を差し引く前の残高です：
* コミット手数料 (資金提供者のみ)
* アンカー出力 (資金提供者のみ)
* 進行中の HTLC

```yaml
[
    {
        "Name": "simple commitment tx with no HTLCs",
        "LocalBalance": 7000000000,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 546,
        "FeePerKw": 15000,
        "UseTestHtlcs": false,
        "HtlcDescs": [],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80044a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f4a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994c0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994a508b6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e04004830450221008266ac6db5ea71aac3c95d97b0e172ff596844851a3216eb88382a8dddfd33d2022050e240974cfd5d708708b4365574517c18e7ae535ef732a3484d43d0d82be9f701483045022100f89034eba16b2be0e5581f750a0a6309192b75cce0f202f0ee2b4ec0cc394850022076c65dc507fe42276152b7a3d90e961e678adbe966e916ecfe85e64d430e75f301475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "3045022100f89034eba16b2be0e5581f750a0a6309192b75cce0f202f0ee2b4ec0cc394850022076c65dc507fe42276152b7a3d90e961e678adbe966e916ecfe85e64d430e75f3"
    },
    {
        "Name": "simple commitment tx with no HTLCs and single anchor",
        "LocalBalance": 10000000000,
        "RemoteBalance": 0,
        "DustLimitSatoshis": 546,
        "FeePerKw": 15000,
        "UseTestHtlcs": false,
        "HtlcDescs": [],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80024a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f10529800000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400473044022007cf6b405e9c9b4f527b0ecad9d8bb661fabb8b12abf7d1c0b3ad1855db3ed490220616d5c1eeadccc63bd775a131149455d62d95a42c2a1b01cc7821fc42dce7778014730440220655bf909fb6fa81d086f1336ac72c97906dce29d1b166e305c99152d810e26e1022051f577faa46412c46707aaac46b65d50053550a66334e00a44af2706f27a865801475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "30440220655bf909fb6fa81d086f1336ac72c97906dce29d1b166e305c99152d810e26e1022051f577faa46412c46707aaac46b65d50053550a66334e00a44af2706f27a8658"
    },
    {
        "Name": "commitment tx with seven outputs untrimmed",
        "LocalBalance": 6988000000,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 546,
        "FeePerKw": 644,
        "UseTestHtlcs": true,
        "HtlcDescs": [
            {
                "RemoteSigHex": "30440220746dc89a593e1b50915db63359b50c3c404f8324f78075c87708c866ccefda2502202a11062012dc8607b17e8c46ea4eefdfcc6b89a35440de99432ba12788d7bb17",
                "ResolutionTxHex": "02000000000101b8cefef62ea66f5178b9361b2371be0759cbc8c689bcfa7a8e6746d497ec221a02000000000100000001e8030000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220746dc89a593e1b50915db63359b50c3c404f8324f78075c87708c866ccefda2502202a11062012dc8607b17e8c46ea4eefdfcc6b89a35440de99432ba12788d7bb1783473044022036f77f88b49bd03dc16d3a015efcc7acffc2a93b213324035d77a716f79013ff02203c218eb882a40402a8bb05dbb751a442345a4e56307be76b214b6d4db9ed3c92012000000000000000000000000000000000000000000000000000000000000000008d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac6851b2756800000000"
            },
            {
                "RemoteSigHex": "3045022100a847bc3b8cf2441013725ae32dc589c419835d069afd6d7f3d9834d8be7cea6e02204ce9d35ec7f0788da80e4d62e810c4ccd13617e58fa10832e38fb7ae87bdf338",
                "ResolutionTxHex": "02000000000101b8cefef62ea66f5178b9361b2371be0759cbc8c689bcfa7a8e6746d497ec221a03000000000100000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100a847bc3b8cf2441013725ae32dc589c419835d069afd6d7f3d9834d8be7cea6e02204ce9d35ec7f0788da80e4d62e810c4ccd13617e58fa10832e38fb7ae87bdf33883473044022026739b1adbfa34c485bf0e5a19e0cf7532f64545bcc5c95b95643ccc2d351e1902201743bb1b34f00e031cc15f6eff0f8ab764508d2681ff965ba62e76e540bca1e801008876a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6851b27568f6010000"
            },
            {
                "RemoteSigHex": "3044022035977f2aa6d6ae12f1dae3366440faa17558e9c88c3188bfc8df7e276c6c65410220659f1f9070c725d9d46e43b99a36fc6d711d36069704add2d57fc1fa2818cf12",
                "ResolutionTxHex": "02000000000101b8cefef62ea66f5178b9361b2371be0759cbc8c689bcfa7a8e6746d497ec221a04000000000100000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022035977f2aa6d6ae12f1dae3366440faa17558e9c88c3188bfc8df7e276c6c65410220659f1f9070c725d9d46e43b99a36fc6d711d36069704add2d57fc1fa2818cf12834730440220191ba44e57b1601a59d99f85971d4801b286d428de275487c87ceeb8df1a4811022002215875d92833df0c9615c9096cf97152f87139ed9f82718bbb5b8b3d312524012001010101010101010101010101010101010101010101010101010101010101018d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6851b2756800000000"
            },
            {
                "RemoteSigHex": "3044022002f3ff9a31270092c214d3d5b8b4f826599404bba64b87f7536ef6324d41551b022079dc4cb25f7ecd84b49f5cce03eae7e2655b59f39d543765daef0d4f11c93fa2",
                "ResolutionTxHex": "02000000000101b8cefef62ea66f5178b9361b2371be0759cbc8c689bcfa7a8e6746d497ec221a05000000000100000001b80b0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022002f3ff9a31270092c214d3d5b8b4f826599404bba64b87f7536ef6324d41551b022079dc4cb25f7ecd84b49f5cce03eae7e2655b59f39d543765daef0d4f11c93fa283483045022100f03047e38bc0aae2d80d53424b8c1d1b8139120e2bf09ad31a2803978745e6e102205b74c0eef0b472710b98c77e619ee9d0cc47a9dd786f4f214a564f44d79f9b9a01008876a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6851b27568f7010000"
            },
            {
                "RemoteSigHex": "30440220547937564288f64fb3e3c945a1348b912471f0c1c4cc7dc8ceca15a4cbd299b6022053c4f8e30832b13dfbe31e4091e313428625e0b5ac61eecba93f8f11c1e26225",
                "ResolutionTxHex": "02000000000101b8cefef62ea66f5178b9361b2371be0759cbc8c689bcfa7a8e6746d497ec221a06000000000100000001a00f0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220547937564288f64fb3e3c945a1348b912471f0c1c4cc7dc8ceca15a4cbd299b6022053c4f8e30832b13dfbe31e4091e313428625e0b5ac61eecba93f8f11c1e2622583473044022022604660234aef9bd21284598ec50f070ac82a3a0152e0af5e98a02cd6e8976f022042b0b9112ee00806b856dff6de52a82c98b036a4fe14bb5fd2926725e2fc8191012004040404040404040404040404040404040404040404040404040404040404048d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6851b2756800000000"
            }
        ],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80094a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f4a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994e80300000000000022002010f88bf09e56f14fb4543fd26e47b0db50ea5de9cf3fc46434792471082621aed0070000000000002200203e68115ae0b15b8de75b6c6bc9af5ac9f01391544e0870dae443a1e8fe7837ead007000000000000220020fe0598d74fee2205cc3672e6e6647706b4f3099713b4661b62482c3addd04a5eb80b000000000000220020f96d0334feb64a4f40eb272031d07afcb038db56aa57446d60308c9f8ccadef9a00f000000000000220020ce6e751274836ff59622a0d1e07f8831d80bd6730bd48581398bfadd2bb8da9ac0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994a4f996a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100ef82a405364bfc4007e63a7cc82925a513d79065bdbc216d60b6a4223a323f8a02200716730b8561f3c6d362eaf47f202e99fb30d0557b61b92b5f9134f8e2de368101483045022100e0106830467a558c07544a3de7715610c1147062e7d091deeebe8b5c661cda9402202ad049c1a6d04834317a78483f723c205c9f638d17222aafc620800cc1b6ae3501475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "3045022100e0106830467a558c07544a3de7715610c1147062e7d091deeebe8b5c661cda9402202ad049c1a6d04834317a78483f723c205c9f638d17222aafc620800cc1b6ae35"
    },
    {
        "Name": "commitment tx with six outputs untrimmed (minimum dust limit)",
        "LocalBalance": 6988000000,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 1001,
        "FeePerKw": 645,
        "UseTestHtlcs": true,
        "HtlcDescs": [
            {
                "RemoteSigHex": "3045022100e04d160a326432659fe9fb127304c1d348dfeaba840081bdc57d8efd902a48d8022008a824e7cf5492b97e4d9e03c06a09f822775a44f6b5b2533a2088904abfc282",
                "ResolutionTxHex": "02000000000101104f394af4c4fad78337f95e3e9f802f4c0d86ab231853af09b285348561320002000000000100000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100e04d160a326432659fe9fb127304c1d348dfeaba840081bdc57d8efd902a48d8022008a824e7cf5492b97e4d9e03c06a09f822775a44f6b5b2533a2088904abfc28283483045022100b7c49846466b13b190ff739bbe3005c105482fc55539e55b1c561f76b6982b6c02200e5c35808619cf543c8405cff9fedd25f333a4a2f6f6d5e8af8150090c40ef0901008876a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6851b27568f6010000"
            },
            {
                "RemoteSigHex": "3045022100fbdc3c367ce3bf30796025cc590ee1f2ce0e72ae1ac19f5986d6d0a4fc76211f02207e45ae9267e8e820d188569604f71d1abd11bd385d58853dd7dc034cdb3e9a6e",
                "ResolutionTxHex": "02000000000101104f394af4c4fad78337f95e3e9f802f4c0d86ab231853af09b285348561320003000000000100000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100fbdc3c367ce3bf30796025cc590ee1f2ce0e72ae1ac19f5986d6d0a4fc76211f02207e45ae9267e8e820d188569604f71d1abd11bd385d58853dd7dc034cdb3e9a6e83483045022100d29330f24db213b262068706099b39c15fa7e070c3fcdf8836c09723fc4d365602203ce57d01e9f28601e461a0b5c4a50119b270bde8b70148d133a6849c70b115ac012001010101010101010101010101010101010101010101010101010101010101018d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6851b2756800000000"
            },
            {
                "RemoteSigHex": "3044022066c5ef625cee3ddd2bc7b6bfb354b5834cf1cc6d52dd972fb41b7b225437ae4a022066cb85647df65c6b87a54e416dcdcca778a776c36a9643d2b5dc793c9b29f4c1",
                "ResolutionTxHex": "02000000000101104f394af4c4fad78337f95e3e9f802f4c0d86ab231853af09b285348561320004000000000100000001b80b0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022066c5ef625cee3ddd2bc7b6bfb354b5834cf1cc6d52dd972fb41b7b225437ae4a022066cb85647df65c6b87a54e416dcdcca778a776c36a9643d2b5dc793c9b29f4c18347304402202d4ce515cd9000ec37575972d70b8d24f73909fb7012e8ebd8c2066ef6fe187902202830b53e64ea565fecd0f398100691da6bb2a5cf9bb0d1926f1d71d05828a11e01008876a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6851b27568f7010000"
            },
            {
                "RemoteSigHex": "3044022022c7e11595c53ee89a57ca76baf0aed730da035952d6ab3fe6459f5eff3b337a022075e10cc5f5fd724a35ce4087a5d03cd616698626c69814032132b50bb97dc615",
                "ResolutionTxHex": "02000000000101104f394af4c4fad78337f95e3e9f802f4c0d86ab231853af09b285348561320005000000000100000001a00f0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022022c7e11595c53ee89a57ca76baf0aed730da035952d6ab3fe6459f5eff3b337a022075e10cc5f5fd724a35ce4087a5d03cd616698626c69814032132b50bb97dc61583483045022100b20cd63e0587d1711beaebda4730775c4ac8b8b2ec78fe18a0c44c3f168c25230220079abb7fc4924e2fca5950842e5b9e416735585026914570078c4ef62f286226012004040404040404040404040404040404040404040404040404040404040404048d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6851b2756800000000"
            }
        ],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80084a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f4a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994d0070000000000002200203e68115ae0b15b8de75b6c6bc9af5ac9f01391544e0870dae443a1e8fe7837ead007000000000000220020fe0598d74fee2205cc3672e6e6647706b4f3099713b4661b62482c3addd04a5eb80b000000000000220020f96d0334feb64a4f40eb272031d07afcb038db56aa57446d60308c9f8ccadef9a00f000000000000220020ce6e751274836ff59622a0d1e07f8831d80bd6730bd48581398bfadd2bb8da9ac0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994abc996a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100d57697c707b6f6d053febf24b98e8989f186eea42e37e9e91663ec2c70bb8f70022079b0715a472118f262f43016a674f59c015d9cafccec885968e76d9d9c5d005101473044022025d97466c8049e955a5afce28e322f4b34d2561118e52332fb400f9b908cc0a402205dc6fba3a0d67ee142c428c535580cd1f2ff42e2f89b47e0c8a01847caffc31201475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "3044022025d97466c8049e955a5afce28e322f4b34d2561118e52332fb400f9b908cc0a402205dc6fba3a0d67ee142c428c535580cd1f2ff42e2f89b47e0c8a01847caffc312"
    },
    {
        "Name": "commitment tx with four outputs untrimmed (minimum dust limit)",
        "LocalBalance": 6988000000,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 2001,
        "FeePerKw": 2185,
        "UseTestHtlcs": true,
        "HtlcDescs": [
            {
                "RemoteSigHex": "304402206870514a72ad6e723ff7f1e0370d7a33c1cd2a0b9272674143ebaf6a1d02dee102205bd953c34faf5e7322e9a1c0103581cb090280fda4f1039ee8552668afa90ebb",
                "ResolutionTxHex": "02000000000101ac13a7715f80b8e52dda43c6929cade5521bdced3a405da02b443f1ffb1e33cc02000000000100000001b80b0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402206870514a72ad6e723ff7f1e0370d7a33c1cd2a0b9272674143ebaf6a1d02dee102205bd953c34faf5e7322e9a1c0103581cb090280fda4f1039ee8552668afa90ebb834730440220669de9ca7910eff65a7773ebd14a9fc371fe88cde5b8e2a81609d85c87ac939b02201ac29472fa4067322e92d75b624942d60be5050139b20bb363db75be79eb946f01008876a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6851b27568f7010000"
            },
            {
                "RemoteSigHex": "3045022100949e8dd938da56445b1cdfdebe1b7efea086edd05d89910d205a1e2e033ce47102202cbd68b5262ab144d9ec12653f87dfb0bb6bd05d1f58ae1e523f028eaefd7271",
                "ResolutionTxHex": "02000000000101ac13a7715f80b8e52dda43c6929cade5521bdced3a405da02b443f1ffb1e33cc03000000000100000001a00f0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100949e8dd938da56445b1cdfdebe1b7efea086edd05d89910d205a1e2e033ce47102202cbd68b5262ab144d9ec12653f87dfb0bb6bd05d1f58ae1e523f028eaefd727183483045022100e3104ed8b239f8019e5f0a1a73d7782a94a8c36e7984f476c3a0b3cb0e62e27902207e3d52884600985f8a2098e53a5c30dd6a5e857733acfaa07ab2162421ed2688012004040404040404040404040404040404040404040404040404040404040404048d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6851b2756800000000"
            }
        ],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80064a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f4a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994b80b000000000000220020f96d0334feb64a4f40eb272031d07afcb038db56aa57446d60308c9f8ccadef9a00f000000000000220020ce6e751274836ff59622a0d1e07f8831d80bd6730bd48581398bfadd2bb8da9ac0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994ac5916a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100cd8479cfe1edb1e5a1d487391e0451a469c7171e51e680183f19eb4321f20e9b02204eab7d5a6384b1b08e03baa6e4d9748dfd2b5ab2bae7e39604a0d0055bbffdd501473044022040f63a16148cf35c8d3d41827f5ae7f7c3746885bb64d4d1b895892a83812b3e02202fcf95c2bf02c466163b3fa3ced6a24926fbb4035095a96842ef516e86ba54c001475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "3044022040f63a16148cf35c8d3d41827f5ae7f7c3746885bb64d4d1b895892a83812b3e02202fcf95c2bf02c466163b3fa3ced6a24926fbb4035095a96842ef516e86ba54c0"
    },
    {
        "Name": "commitment tx with three outputs untrimmed (minimum dust limit)",
        "LocalBalance": 6988000000,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 3001,
        "FeePerKw": 3687,
        "UseTestHtlcs": true,
        "HtlcDescs": [
            {
                "RemoteSigHex": "3044022017b558a3cf5f0cb94269e2e927b29ed22bd2416abb8a7ce6de4d1256f359b93602202e9ca2b1a23ea3e69f433c704e327739e219804b8c188b1d52f74fd5a9de954c",
                "ResolutionTxHex": "02000000000101542562b326c08e3a076d9cfca2be175041366591da334d8d513ff1686fd95a6002000000000100000001a00f0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022017b558a3cf5f0cb94269e2e927b29ed22bd2416abb8a7ce6de4d1256f359b93602202e9ca2b1a23ea3e69f433c704e327739e219804b8c188b1d52f74fd5a9de954c83483045022100af7a8b7c7ff2080c68995254cb66d64d9954edcc5baac3bb4f27ed2d29aaa6120220421c27da7a60574a9263f271e0f3bd34594ec6011095190022b3b54596ea03de012004040404040404040404040404040404040404040404040404040404040404048d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6851b2756800000000"
            }
        ],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80054a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f4a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994a00f000000000000220020ce6e751274836ff59622a0d1e07f8831d80bd6730bd48581398bfadd2bb8da9ac0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994aa28b6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100c970799bcb33f43179eb43b3378a0a61991cf2923f69b36ef12548c3df0e6d500220413dc27d2e39ee583093adfcb7799be680141738babb31cc7b0669a777a31f5d01483045022100ad6c71569856b2d7ff42e838b4abe74a713426b37f22fa667a195a4c88908c6902202b37272b02a42dc6d9f4f82cab3eaf84ac882d9ed762859e1e75455c2c22837701475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "3045022100ad6c71569856b2d7ff42e838b4abe74a713426b37f22fa667a195a4c88908c6902202b37272b02a42dc6d9f4f82cab3eaf84ac882d9ed762859e1e75455c2c228377"
    },
    {
        "Name": "commitment tx with two outputs untrimmed (minimum dust limit)",
        "LocalBalance": 6988000000,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 4001,
        "FeePerKw": 4894,
        "UseTestHtlcs": true,
        "HtlcDescs": [],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80044a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f4a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994c0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994ad0886a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e04004830450221009f16ac85d232e4eddb3fcd750a68ebf0b58e3356eaada45d3513ede7e817bf4c02207c2b043b4e5f971261975406cb955219fa56bffe5d834a833694b5abc1ce4cfd01483045022100e784a66b1588575801e237d35e510fd92a81ae3a4a2a1b90c031ad803d07b3f3022021bc5f16501f167607d63b681442da193eb0a76b4b7fd25c2ed4f8b28fd35b9501475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "3045022100e784a66b1588575801e237d35e510fd92a81ae3a4a2a1b90c031ad803d07b3f3022021bc5f16501f167607d63b681442da193eb0a76b4b7fd25c2ed4f8b28fd35b95"
    },
    {
        "Name": "commitment tx with one output untrimmed (minimum dust limit)",
        "LocalBalance": 6988000000,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 4001,
        "FeePerKw": 6216010,
        "UseTestHtlcs": true,
        "HtlcDescs": [],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80024a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994c0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994a04004830450221009ad80792e3038fe6968d12ff23e6888a565c3ddd065037f357445f01675d63f3022018384915e5f1f4ae157e15debf4f49b61c8d9d2b073c7d6f97c4a68caa3ed4c1014830450221008fd5dbff02e4b59020d4cd23a3c30d3e287065fda75a0a09b402980adf68ccda022001e0b8b620cd915ddff11f1de32addf23d81d51b90e6841b2cb8dcaf3faa5ecf01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "30450221008fd5dbff02e4b59020d4cd23a3c30d3e287065fda75a0a09b402980adf68ccda022001e0b8b620cd915ddff11f1de32addf23d81d51b90e6841b2cb8dcaf3faa5ecf"
    },
    {
        "Name": "commitment tx with 3 htlc outputs, 2 offered having the same amount and preimage",
        "LocalBalance": 6987999999,
        "RemoteBalance": 3000000000,
        "DustLimitSatoshis": 546,
        "FeePerKw": 253,
        "UseTestHtlcs": true,
        "HtlcDescs": [
            {
                "RemoteSigHex": "30440220078fe5343dab88c348a3a8a9c1a9293259dbf35507ae971702cc39dd623ea9af022011ed0c0f35243cd0bb4d9ca3c772379b2b5f4af93140e9fdc5600dfec1cdb0c2",
                "ResolutionTxHex": "020000000001013d060d0305c9616eaabc21d41fae85bcb5477b5d7f1c92aa429cf15339bbe1c402000000000100000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220078fe5343dab88c348a3a8a9c1a9293259dbf35507ae971702cc39dd623ea9af022011ed0c0f35243cd0bb4d9ca3c772379b2b5f4af93140e9fdc5600dfec1cdb0c28347304402205df665e2908c7690d2d33eb70e6e119958c28febe141a94ed0dd9a55ce7c8cfc0220364d02663a5d019af35c5cd5fda9465d985d85bbd12db207738d61163449a424012001010101010101010101010101010101010101010101010101010101010101018d76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6851b2756800000000"
            },
            {
                "RemoteSigHex": "304402202df6bf0f98a42cfd0172a16bded7d1b16c14f5f42ba23f5c54648c14b647531302200fe1508626817f23925bb56951d5e4b2654c751743ab6db48a6cce7dda17c01c",
                "ResolutionTxHex": "020000000001013d060d0305c9616eaabc21d41fae85bcb5477b5d7f1c92aa429cf15339bbe1c40300000000010000000188130000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402202df6bf0f98a42cfd0172a16bded7d1b16c14f5f42ba23f5c54648c14b647531302200fe1508626817f23925bb56951d5e4b2654c751743ab6db48a6cce7dda17c01c8347304402203f99ec05cdd89558a23683b471c1dcce8f6a92295f1fff3b0b5d21be4d4f97ea022019d29070690fc2c126fe27cc4ab2f503f289d362721b2efa7418e7fddb939a5b01008876a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9142002cc93ebefbb1b73f0af055dcc27a0b504ad7688ac6851b27568f9010000"
            },
            {
                "RemoteSigHex": "3045022100bd206b420c495f3aa714d3ea4766cbe95441deacb5d2f737f1913349aee7c2ae02200249d2c950dd3b15326bf378ae5d2b871d33d6737f5d70735f3de8383140f2a1",
                "ResolutionTxHex": "020000000001013d060d0305c9616eaabc21d41fae85bcb5477b5d7f1c92aa429cf15339bbe1c40400000000010000000188130000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100bd206b420c495f3aa714d3ea4766cbe95441deacb5d2f737f1913349aee7c2ae02200249d2c950dd3b15326bf378ae5d2b871d33d6737f5d70735f3de8383140f2a183483045022100f2cd35e385b9b7e15b92a5d78d120b6b2c5af4e974bc01e884c5facb3bb5966c0220706e0506477ce809a40022d6de8e041e9ef13136c45abee9c36f58a01fdb188b01008876a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9142002cc93ebefbb1b73f0af055dcc27a0b504ad7688ac6851b27568fa010000"
            }
        ],
        "ExpectedCommitmentTxHex": "02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b80074a010000000000002200202b1b5854183c12d3316565972c4668929d314d81c5dcdbb21cb45fe8a9a8114f4a01000000000000220020e9e86e4823faa62e222ebc858a226636856158f07e69898da3b0d1af0ddb3994d007000000000000220020fe0598d74fee2205cc3672e6e6647706b4f3099713b4661b62482c3addd04a5e881300000000000022002018e40f9072c44350f134bdc887bab4d9bdfc8aa468a25616c80e21757ba5dac7881300000000000022002018e40f9072c44350f134bdc887bab4d9bdfc8aa468a25616c80e21757ba5dac7c0c62d0000000000220020f3394e1e619b0eca1f91be2fb5ab4dfc59ba5b84ebe014ad1d43a564d012994aad9c6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100b4014970d9d7962853f3f85196144671d7d5d87426250f0a5fdaf9a55292e92502205360910c9abb397467e19dbd63d081deb4a3240903114c98cec0a23591b79b7601473044022027b38dfb654c34032ffb70bb43022981652fce923cbbe3cbe7394e2ade8b34230220584195b78da6e25c2e8da6b4308d9db25b65b64975db9266163ef592abb7c72501475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220",
        "RemoteSigHex": "3044022027b38dfb654c34032ffb70bb43022981652fce923cbbe3cbe7394e2ade8b34230220584195b78da6e25c2e8da6b4308d9db25b65b64975db9266163ef592abb7c725"
    }
]
```

# Appendix G: デュアルファンドトランザクションのテストベクトル

## ファンディングトランザクションの構築
### 前提条件：

```
Genesis block 0:
0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4adae5494dffff7f20020000000101000000010000000000000000000000000000000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac00000000

Block 1:0000002006226e46111a0b59caaf126043eb5bbf28c34f3a5e332a1fc7b2b73cf188910ff86fd1d0db3ac5a72df968622f31e6b5e6566a09e29206d7c7a55df90e181de8be86815cffff7f200000000001020000000001010000000000000000000000000000000000000000000000000000000000000000ffffffff03510101ffffffff0200f2052a0100000017a914113ca7e584fe1575b6fc39abae991529f66eda58870000000000000000266a24aa21a9ede2f61c3f71d1defd3fa999dfa36953755c690689799962b48bebd836974e8cf90120000000000000000000000000000000000000000000000000000000000000000000000000
Coinbase address pubkey: 2MtpN8zCxTp8AWSg7VBjBX7vU6x73bVCKP8
Coinbase address privkey: cPxFtfE1w3ptFnsZvvFeWji21kTArYa9GXwMkYsoQHdaJKrjUTek

Parent transaction (spends coinbase of block 1):
02000000000101f86fd1d0db3ac5a72df968622f31e6b5e6566a09e29206d7c7a55df90e181de800000000171600141fb9623ffd0d422eacc450fd1e967efc477b83ccffffffff0580b2e60e00000000220020fd89acf65485df89797d9ba7ba7a33624ac4452f00db08107f34257d33e5b94680b2e60e0000000017a9146a235d064786b49e7043e4a042d4cc429f7eb6948780b2e60e00000000160014fbb4db9d85fba5e301f4399e3038928e44e37d3280b2e60e0000000017a9147ecd1b519326bc13b0ec716e469b58ed02b112a087f0006bee0000000017a914f856a70093da3a5b5c4302ade033d4c2171705d387024730440220696f6cee2929f1feb3fd6adf024ca0f9aa2f4920ed6d35fb9ec5b78c8408475302201641afae11242160101c6f9932aeb4fcd1f13a9c6df5d1386def000ea259a35001210381d7d5b1bc0d7600565d827242576d9cb793bfe0754334af82289ee8b65d137600000000
```

### ファンディングトランザクション (親の出力を消費)：

ロックタイム：120
手数料率：253 sat/kiloweight
オープナーの `funding_satoshi`：2 0000 0000 sat
アクセプターの `funding_satoshi`：2 0000 0000 sat

インプット：
```
4303ca8ff10c6c345b9299672a66f111c5b81ae027cc5b0d4d39d09c66b032b9 0
	witness_data:
	  preimage: 20 68656c6c6f2074686572652c2074686973206973206120626974636f6e212121
	  witness_script: 27 82012088a820add57dfe5277079d069ca4ad4893c96de91f88ffb981fdc6a2a34d5336c66aff87
	scriptPubKey: 0020fd89acf65485df89797d9ba7ba7a33624ac4452f00db08107f34257d33e5b946
	address: bcrt1qlky6eaj5sh0cj7tanwnm573nvf9vg3f0qrdssyrlxsjh6vl9h9rql40v2g

4303ca8ff10c6c345b9299672a66f111c5b81ae027cc5b0d4d39d09c66b032b9 2
	pubkey: 034695f5b7864c580bf11f9f8cb1a94eb336f2ce9ef872d2ae1a90ee276c772484
	privkey: cUM8Dr33wK4uFmw3Tz8sbQ7BiBNgX5BthRurU7RkgXVvNUPcWrJf
	witness_program: fbb4db9d85fba5e301f4399e3038928e44e37d32
	scriptPubKey: 0014fbb4db9d85fba5e301f4399e3038928e44e37d32
	address: bcrt1qlw6dh8v9lwj7xq058x0rqwyj3ezwxlfjxsy7er

```

### 期待されるオープナーの `tx_add_input` (上記のインプット 0)：
```
  num_inputs: 1
  tx_add_input:[
    {
      channel_id: "xxx",
      serial_id: 20,
      prevtx_len: 353,
      prevtx: "02000000000101f86fd1d0db3ac5a72df968622f31e6b5e6566a09e29206d7c7a55df90e181de800000000171600141fb9623ffd0d422eacc450fd1e967efc477b83ccffffffff0580b2e60e00000000220020fd89acf65485df89797d9ba7ba7a33624ac4452f00db08107f34257d33e5b94680b2e60e0000000017a9146a235d064786b49e7043e4a042d4cc429f7eb6948780b2e60e00000000160014fbb4db9d85fba5e301f4399e3038928e44e37d3280b2e60e0000000017a9147ecd1b519326bc13b0ec716e469b58ed02b112a087f0006bee0000000017a914f856a70093da3a5b5c4302ade033d4c2171705d387024730440220696f6cee2929f1feb3fd6adf024ca0f9aa2f4920ed6d35fb9ec5b78c8408475302201641afae11242160101c6f9932aeb4fcd1f13a9c6df5d1386def000ea259a35001210381d7d5b1bc0d7600565d827242576d9cb793bfe0754334af82289ee8b65d137600000000",
      prev_vout: 0,
      sequence: 4294967293
    }
  ]
```

### 期待されるアクセプターの `tx_add_input` (上記のインプット 2)：
```
  num_inputs: 1
  tx_add_input:[
    {
      channel_id: "xxx",
      serial_id: 11,
      prevtx_len: 353,
      prevtx: "02000000000101f86fd1d0db3ac5a72df968622f31e6b5e6566a09e29206d7c7a55df90e181de800000000171600141fb9623ffd0d422eacc450fd1e967efc477b83ccffffffff0580b2e60e00000000220020fd89acf65485df89797d9ba7ba7a33624ac4452f00db08107f34257d33e5b94680b2e60e0000000017a9146a235d064786b49e7043e4a042d4cc429f7eb6948780b2e60e00000000160014fbb4db9d85fba5e301f4399e3038928e44e37d3280b2e60e0000000017a9147ecd1b519326bc13b0ec716e469b58ed02b112a087f0006bee0000000017a914f856a70093da3a5b5c4302ade033d4c2171705d387024730440220696f6cee2929f1feb3fd6adf024ca0f9aa2f4920ed6d35fb9ec5b78c8408475302201641afae11242160101c6f9932aeb4fcd1f13a9c6df5d1386def000ea259a35001210381d7d5b1bc0d7600565d827242576d9cb793bfe0754334af82289ee8b65d137600000000",
      prev_vout: 2,
      sequence: 4294967293
    }
  ]
```

### 出力：(scriptPubKeys)
```
# opener's change address
pubkey: 0206e626a4c6d4392d4030bc78bd93f728d1ba61214a77c63adc17d71e32ded3df
# privkey: cSpC1KYEV1vsUFBwTdcuRkncbwfipY1m5zuQ9CjgAYwiVvbQ4fc1
scriptPubKey: 00141ca1cca8855bad6bc1ea5436edd8cff10b7e448b
address: bcrt1qrjsue2y9twkkhs022smwmkx07y9hu3ytshgjmj

# accepter's change address
pubkey: 028f3978c211f4c0bf4d20674f345ae14e08871b25b2c957b4bdbd42e9726278fc
privkey: cQ1HXnbAE4wGhuB2b9rJEydV5ayeEmMqxf1dvHPZmyMTPkwvZJyg
scriptPubKey: 001444cb0c39f93ecc372b5851725bd29d865d333b10
address: bcrt1qgn9scw0e8mxrw26c29e9h55asewnxwcsdxdp50

# the 2-of-2s
pubkey1: 0292edb5f7bbf9e900f7e024be1c1339c6d149c11930e613af3a983d2565f4e41e
pubkey2: 02e16172a41e928cbd78f761bd1c657c4afc7495a1244f7f30166b654fbf7661e3
script_def: multi(2,0292edb5f7bbf9e900f7e024be1c1339c6d149c11930e613af3a983d2565f4e41e,02e16172a41e928cbd78f761bd1c657c4afc7495a1244f7f30166b654fbf7661e3)
script: 52210292edb5f7bbf9e900f7e024be1c1339c6d149c11930e613af3a983d2565f4e41e2102e16172a41e928cbd78f761bd1c657c4afc7495a1244f7f30166b654fbf7661e352ae
scriptPubKey: 0020297b92c238163e820b82486084634b4846b86a3c658d87b9384192e6bea98ec5
address: bcrt1q99ae9s3czclgyzuzfpsggc6tfprts63uvkxc0wfcgxfwd04f3mzs3asq6l
```

### 期待されるオープナーの `tx_add_output`：

```
  num_outputs: 2
  tx_add_output[
    {
      channel_id:"xxx",
      serial_id: 30,
      sats: 49999845,
      scriptlen: 22,
      script: "1600141ca1cca8855bad6bc1ea5436edd8cff10b7e448b"
    },{
      channel_id: "xxx",
      serial_id: 44,
      sats: 400000000,
      scriptlen: 34,
      script: "220020297b92c238163e820b82486084634b4846b86a3c658d87b9384192e6bea98ec5"
    }
  ]
```

### 期待されるアクセプターの `tx_add_output`：

```
  num_outputs: 1
  tx_add_output[
    {
      channel_id: xxx,
      serial_id: 33,
      sats: 49999900,
      scriptlen: 22,
      script: 16001444cb0c39f93ecc372b5851725bd29d865d333b10
    }
  ]
```

### 期待される手数料の計算：

オープナーの手数料とお釣り：
```
    initiator_weight = (input_count, output_count, version, locktime) * 4
                     + segwit_fields (marker + flag)
                     + inputs (txid, vout, scriptSig, sequence) * number initiator inputs * 4
                     + funding_output * 4
                     + change_output * 4
                     + max(number initiator inputs
			       * minimum witness weight,
			   actual witness weight of all inputs)

    initiator_weight = (1 + 1 + 4 + 4) * 4
                     + 2
                     + (32 + 4 + 1 + 4) * 1 * 4
                     + 43 * 4
                     + 31 * 4
                     + max(1 * 107, 71)

    initiator_weight = 609

    initiator_fees = initiator_weight * feerate
    initiator_fees = 609 * 253 / 1000
    initiator_fees = 155 sats

    change = total_funding
           - funding_sats
           - fees

    change = 2 5000 0000
           - 2 0000 0000
           -         155

    change =   4999 9845

    as hex: e5effa0200000000
```

### アクセプターの手数料とお釣り：
```
    contributor_weight = inputs(txid, vout, scriptSig, sequence)
			  * number contributor inputs * 4
                       + change_output * 4
		       + max(number contributor inputs
			         * minimum witness weight,
			     actual witness weight of all inputs)

    contributor_weight = (32 + 4 + 1 + 4) * 1 *  4
                       + 31 * 4
                       + max(1 * 107, 99)

    contributor_weight = 395

    contributor_fees = contributor_weight * feerate
    contributor_fees = 398 * 253 / 1000
    contributor_fees = 100 sats

    change = total_funding
           - funding_sats
           - fees

    change = 2 5000 0000
           - 2 0000 0000
           -         100

    change =   4999 9900

    as hex: 1cf0fa0200000000
```

### 署名なしファンディングトランザクション：

```
0200000002b932b0669cd0394d0d5bcc27e01ab8c511f1662a6799925b346c0cf18fca03430200000000fdffffffb932b0669cd0394d0d5bcc27e01ab8c511f1662a6799925b346c0cf18fca03430000000000fdffffff03e5effa02000000001600141ca1cca8855bad6bc1ea5436edd8cff10b7e448b1cf0fa020000000016001444cb0c39f93ecc372b5851725bd29d865d333b100084d71700000000220020297b92c238163e820b82486084634b4846b86a3c658d87b9384192e6bea98ec578000000
```

### 期待されるオープナーの `tx_signatures`：

```
  tx_signatures{
      channel_id: xxx,
      txid: "5ca4e657c1aa9d069ea4a5d712045d233a7d7c52738cb02993637289e6386057",
      num_witnesses: 1,
      witness[{
        len: 74,
        witness_data: "022068656c6c6f2074686572652c2074686973206973206120626974636f6e2121212782012088a820add57dfe5277079d069ca4ad4893c96de91f88ffb981fdc6a2a34d5336c66aff87"
      }]
  }
```

### 期待されるアクセプターの `tx_signatures`：

```
  tx_signatures{
      channel_id: xxx,
      txid: "5ca4e657c1aa9d069ea4a5d712045d233a7d7c52738cb02993637289e6386057",
      num_witnesses: 1,
      witness[{
        len: 107,
        witness_data: "0247304402207de9ba56bb9f641372e805782575ee840a899e61021c8b1572b3ec1d5b5950e9022069e9ba998915dae193d3c25cb89b5e64370e6a3a7755e7f31cf6d7cbc2a49f6d0121034695f5b7864c580bf11f9f8cb1a94eb336f2ce9ef872d2ae1a90ee276c772484"
      }]
  }
```

#### 署名済みファンディングトランザクション：

ロックタイムは 120 に設定されています。

```
02000000000102b932b0669cd0394d0d5bcc27e01ab8c511f1662a6799925b346c0cf18fca03430200000000fdffffffb932b0669cd0394d0d5bcc27e01ab8c511f1662a6799925b346c0cf18fca03430000000000fdffffff03e5effa02000000001600141ca1cca8855bad6bc1ea5436edd8cff10b7e448b1cf0fa020000000016001444cb0c39f93ecc372b5851725bd29d865d333b100084d71700000000220020297b92c238163e820b82486084634b4846b86a3c658d87b9384192e6bea98ec50247304402207de9ba56bb9f641372e805782575ee840a899e61021c8b1572b3ec1d5b5950e9022069e9ba998915dae193d3c25cb89b5e64370e6a3a7755e7f31cf6d7cbc2a49f6d0121034695f5b7864c580bf11f9f8cb1a94eb336f2ce9ef872d2ae1a90ee276c772484022068656c6c6f2074686572652c2074686973206973206120626974636f6e2121212782012088a820add57dfe5277079d069ca4ad4893c96de91f88ffb981fdc6a2a34d5336c66aff8778000000
```


# 参考文献

# 著者

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
