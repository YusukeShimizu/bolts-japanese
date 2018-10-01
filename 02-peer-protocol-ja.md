# BOLT #2: チャネル管理のためのピア間プロトコル

ピア間チャネルプロトコルは、確立、通常取引、クローズの３つのフェイズで構成されます。

# 目次
  * [チャネル](#channel)
    * [チャネル確立](#channel-establishment)
      * [ `open_channel` メッセージ](#the-open_channel-message)
      * [ `accept_channel` メッセージ](#the-accept_channel-message)
      * [ `funding_created` メッセージ](#the-funding_created-message)
      * [ `funding_signed` メッセージ](#the-funding_signed-message)
      * [ `funding_locked` メッセージ](#the-funding_locked-message)
    * [チャネルクローズ](#channel-close)
      * [クローズ開始: `shutdown` メッセージ](#closing-initiation-shutdown)
      * [クローズ調整todo: `closing_signed` メッセージ](#closing-negotiation-closing_signed)
    * [通常取引](#normal-operation)
      * [HTLC送付](#forwarding-htlcs)
      * [HTLCタイムアウトのリスク](#risks-with-htlc-timeouts)
      * [HTLC追加: `update_add_htlc` メッセージ](#adding-an-htlc-update_add_htlc)
      * [HTLC削除: `update_fulfill_htlc` メッセージ、 `update_fail_htlc` メッセージ、 `update_fail_malformed_htlc` メッセージ](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * [今までの更新内容のコミット: `commitment_signed` メッセージ](#committing-updates-so-far-commitment_signed)
      * [更新された状態へ遷移完了: `revoke_and_ack` メッセージ](#completing-the-transition-to-the-updated-state-revoke_and_ack)
      * [手数料の更新: `update_fee` メッセージ](#updating-fees-update_fee)
    * [メッセージの再送付](#message-retransmission)
  * [著者](#authors)

# チャネル

## チャネルの確立

チャネルの確立は認証のあとすぐに始まります。
チャネルの確立は `open_channel` メッセージを送るファンディングノードと、それに対して `accept_channel` メッセージを返答するノードで構成されます。
チャネルパラメータを決めた上で、ファンディングノードはファンディングトランザクションとコミットメントトランザクションの両ノードのバージョンを作ることができます。
このバージョンは、[BOLT 03](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#bolt-3-bitcoin-transaction-and-script-formats) に記述されているものです。
ファンディングノードはその後、`funding_created` メッセージを使ってファンディングトランザクションアウトプットのアウトポイントを送ります。
この際、返答ノードのコミットメントトランザクションバージョンに合った署名も送られます。
一度返答ノードがファンディングアウトポイントを受け取るとコミットメントトランザクションに対する初期コミットメントを生成できるようになり、 `funding_signed` メッセージを使ってこれをファンディングノードに返送します。

一度チャネルファンディングノードが `funding_signed` メッセージを受け取ると、彼らはファンディングトランザクションをBitcoinネットワークにブロードキャストしなければいけません。
`funding_signed` メッセージを送受したあと、両サイドのノードは、ファンディングトランザクションがブロックチェーンに入り指定された深さ(承認数)に達するまで待つべきです。
`funding_locked` メッセージが両サイドに送られると、チャネルは確立され通常取引を始めることができるようになります。
`funding_locked` メッセージにはチャネル認証証明を作るために必要な情報が含まれています。

        +-------+                              +-------+
        |       |--(1)---  open_channel  ----->|       |
        |       |<-(2)--  accept_channel  -----|       |
        |       |                              |       |
        |   A   |--(3)--  funding_created  --->|   B   |
        |       |<-(4)--  funding_signed  -----|       |
        |       |                              |       |
        |       |--(5)--- funding_locked  ---->|       |
        |       |<-(6)--- funding_locked  -----|       |
        +-------+                              +-------+


もしどこか段階で失敗する、または相手ノードから提供されるチャネル期間をノードが適切なものでないと判断した場合は、チャネル確立が異常終了します。

ここで、２つのノード間で複数のチャネルを平行で動作させることができる点を強調しておきます。
というのは、全てのチャネルメッセージは `temporary-channel-id` メッセージ(ファンディングトランザクションが作られる前であれば)またはファンディングトランザクションから作られる `channel-id` メッセージによって一意に指定され、２つのノードidとは切り話されているためです。

### `open_channel`メッセージ

このメッセージはノードに関する情報を含んでおり、新しくチャネルの開設要望を示しています。

1. type: 32 (`open_channel`)
2. data:
   * [`32`:`chain_hash`]
   * [`32`:`temporary_channel_id`]
   * [`8`:`funding_satoshis`]
   * [`8`:`push_msat`]
   * [`8`:`dust_limit_satoshis`]
   * [`8`:`max_htlc_value_in_flight_msat`]
   * [`8`:`channel_reserve_satoshis`]
   * [`4`:`htlc_minimum_msat`]
   * [`4`:`feerate_per_kw`]
   * [`2`:`to_self_delay`]
   * [`2`:`max_accepted_htlcs`]
   * [`33`:`funding_pubkey`]
   * [`33`:`revocation_basepoint`]
   * [`33`:`payment_basepoint`]
   * [`33`:`delayed_payment_basepoint`]
   * [`33`:`first_per_commitment_point`]

chain-hashの値はオープンしたチャネルがあるブロックチェーンを示しています。
これは通常それぞれのブロックチェーンのジェネシスブロックのハッシュ値になります。
chain-hashがあることで、ノードは複数のブロックチェーンにチャネルを持てるだけでなく、多くの別々のブロックチェーンをまたがるようなチャネルを開くこともできるようになります(todo)。

`temporary-channel-id` フィールドは、ファンディングトランザクションが確立するまでの間にチャネルを指定するidとして使われ
ます。
`funding-satoshis` フィールドはファンディングノードが最初にチャネルに置いた金額です。
`dust-limit-satoshis` フィールドは、このノードのコミットメントまたはHTLCトランザクションに対して生成されるアウトプットが持つ金額下限値で、例えば、この金額とHTLCトランザクション手数料の和よりも小さいHTLCはオンチェーンに入ることができません。
これは、極小金額を持つアウトプットが標準トランザクションとはみなされず、Bitcoinネットワークを伝搬できないということに対応しています。

`max-htlc-value-in-flight-msat` フィールドは未払いHTLCの総和に対する上限値です。
これは未払いHTLCに拠出できる金額を制限できるようにしています。
同様に `max-accepted-htlcs` フィールドは相手のノードが要求できる未払いHTLCの上限数になっています。
`channel-reserve-satoshis`フィールドは相手のノードが自身に必ずキープしておかなければいけない最小金額です。
`htlc-minimum-msat` フィールドはこのノードが受け取れる最も小さいHTLC金額を示しています。


`feerate-per-kw` フィールドは、1000ウェイトごと(つまり、通常使われる'キロバイト単位の手数料率'の1/4)の初期手数料率を示しています。
この初期手数料は、 [BOLT #3](03-transactions.md#fee-calculation) で書かれているようなコミットメントトランザクションやHTLCトランザクションに対して支払われる手数料です(のちほど `update_fee` メッセージによって微調整できます)。
`to-self-delay` フィールドは `OP_CHECKSEQUENCEVERIFY` を使って行われる相手ノード自身へのアウトプットをどれくらい遅延させておかないといけないかを指定するブロック数です。
これは、相手ノードがチャネルをbreakdownした際に、相手ノードの自分の資産を得るのにどれくらいの時間待たなければいけないかを与えています。

`funding-pubkey` フィールドは、ファンディングトランザクションアウトプットの2-of-2マルチシグscriptにある公開鍵です。
`revocation-basepoint` フィールドは、取り消しpreimageと紐づくもので、このコミットメントトランザクションがこのコミットメ
ントトランザクションに対する一意な取り消し鍵を生成する基準になるものです。
`payment-basepoint` フィールドと `delayed-payment-basepoint` フィールドは、同様にこのノードへの支払いに使う鍵の列を生成
するために使います。
`delayed-payment-basepoint` フィールドは、遅延によって妨げられる支払いに対して使われます(todo)。
これらの鍵を変えることで、たとえ１個のコミットメントトランザクションがわかってもそれぞれのコミットメントトランザクションのtxidは外部から予測できなくなります。
これにより、第三者にペナルティートランザクションをアウトソースするときでもプライバシーを守ることができます。

#### 要件

open_channelメッセージを送付するノードは、chain-hashの値が開くチャネルのブロックチェーンを一意に指定していることを保障しなければいけません。
Bitcoinブロックチェーンに対してであれば、chain-hashの値は(16進数で) `000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f` です。

メッセージ送付ノードは、 `temporary-channel-id` フィールドが同じピア内で他で使われているチャネルidとは別の一意の値になっていることを保障しなければいけません。
メッセージ送付ノードは、 `funding-satoshis` フィールドを2^24satoshiよりも小さい値にしなければいけません。
メッセージ送付ノードは、 `push-msat` フィールドを `funding-satoshis` フィールド * 1000より小さいまたは等しい値にしなければいけません。
メッセージ送付ノードは、メッセージ受信ノードによる不正行為が置きた場合に送付ノードがコミットメントトランザクションを使用できるように十分な値を `to-self-delay` フィールドを設定するべきです。
`funding-pubkey` フィールド、 `revocation-basepoint` フィールド、 `payment-basepoint` フィールド、 `delayed-payment-basepoint` フィールドは、DER形式の有効な圧縮secp256k1公開鍵でなければいけません。
メッセージ送付ノードは、トランザクションがすぐにブロックに取り込まれると見積もれるような最小の値を `feerate-per-kw` フィールドに設定するべきです。

メッセージ送付ノードは `dust-limit-satoshis` フィールドをコミットメントトランザクションがBitcoinネットワークを伝搬できるような十分な値にするべきです。
`htlc-minimum-msat` フィールドには相手ピアが受け入れてくれるようなHTLCに対する最低金額を設定すべきです。

もし前の `open-channel` メッセージを受け取ったのち `funding-created` メッセージではなく新しい `open-channel` メッセージを受け取った場合は、受信ノードは新しい `open-channel` メッセージを受け入れなければいけません。
この場合、受信ノードは前の `open-channel` メッセージを破棄しなければいけません。

受信ノードは、もし `to-self-delay` フィールドの値が極端に大きくなっていた場合、チャネルを異常終了しなければいけません。
受信ノードは、もし `funding-satoshis` フィールドの値が小さすぎている場合、チャネルを異常終了するかもしれません。
受信ノードは、もし `push-msat` フィールドの値が `funding-amount` * 1000よりも大きくなっている場合、チャネルを異常終了しなければいけません。
受信ノードは、もし `htlc-minimum-msat` フィールド、 `channel-reserve-satoshis` フィールドの値が大きすぎる、または `max-htlc-value-in-flight` フィールド、 `max-accepted-htlcs` フィールドの値が小さすぎる場合はチャネルを異常終了しなければいけません。
受信ノードは、もし `max-accepted-htlcs` フィールドの値が483より大きい場合、チャネルを異常終了しなければいけません。

受信ノードは、もし `feerate-per-kw` フィールドの値が迅速な処理に対して小さすぎる、または極端に大きすぎる場合、チャネルを異常終了しなければいけません。
受信ノードは、もし `funding-pubkey` フィールド、 `revocation-basepoint` フィールド、 `payment-basepoint` フィールド、 `delayed-payment-basepoint` フィールドの値が、DER形式の有効な圧縮secp256k1公開鍵でない場合、チャネルを異常終了しなければいけません。

受信ノードは、ファンディングトランザクションの承認数が十分な数になるまで `push-msat` を使って受け取った資金を受け取り済
みとして考えてはいけません。

#### 合理性

*channel reserve(チャネル予備金)* はピアの `channel-reserve-satoshis` フィールドの値によって指定されています。例としては、チャネルに置いてある総金額の1%です。
相手が古いトランザクションをブロードキャストしようとした場合に常に何かを失うように、チャネルのそれぞれの側がこのreserveを保ちます。
最初に、このreserveは満たされていないかもしれません。
というのは、片方の側だけが資金を持っているからです。
しかし、プロトコルはreserveを保つような方向で動き、一度満たされればそれを保つようにします。

メッセージ送信ノードは、無条件に最低でも `push-msat` の値の初期資金を受信ノードに送ります。
これは、通常のreserverメカニズムが適用されていない１つの場合です。
しかし、他のオンチェーントランザクションのように、この支払いはファンディングトランザクションが十分な数だけ承認されてからでなければ確かなものにはなりません(二重支払いの可能性があるため)。
そして、支払いがオンチェーンでの承認を受けたものかどうかを確認するためには他の方法が必要である可能性があります。

`feerate-per-kw` フィールドは、一般にメッセージ送信ノード側のみが気にすることです(つまり、誰が手数料を払うのか)。
しかし、このフィールドの値はまたHTLCによって支払われる手数料率でもあります。
このため、高すぎる手数料率は受金者を不利な立場に置くことになります。

#### 将来

### `accept_channel` メッセージ

このメッセージはノードについての情報を持ち、新しいチャネルの了承を示します。


1. type: 33 (`accept_channel`)
2. data:
   * [`32`:`temporary_channel_id`]
   * [`8`:`dust_limit_satoshis`]
   * [`8`:`max_htlc_value_in_flight_msat`]
   * [`8`:`channel_reserve_satoshis`]
   * [`4`:`minimum_depth`]
   * [`4`:`htlc_minimum_msat`]
   * [`2`:`to_self_delay`]
   * [`2`:`max_accepted_htlcs`]
   * [`33`:`funding_pubkey`]
   * [`33`:`revocation_basepoint`]
   * [`33`:`payment_basepoint`]
   * [`33`:`delayed_payment_basepoint`]
   * [`33`:`first_per_commitment_point`]


#### 要件

### `funding_created` メッセージ

このメッセージは、ファンディングノードが初期コミットメントトランザクションのために作るアウトポイントを示しています。
ピアが署名を受け取ったのちにこのファンディングトランザクションがブロードキャストされます。

1. type: 34 (`funding_created`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`funding_txid`]
    * [`2`:`funding_output_index`]
    * [`64`:`signature`]

#### 要件

#### 合理性

`funding-output-index` フィールドは２バイト長です。
というのは、このフィールドの値はゴシッププロトコルを通して使われるchannel-idに詰め込まれるものであるためです。
65535個のアウトプット制限は過度な制限というわけではないです。


インプット全てがSegregated Witnessインプットになっているトランザクションには展性はないため、ファンディングトランザクションに推奨されています。

### The `funding_signed` message

このメッセージは、ファンディングノードに最初のコミットメントトランザクションに必要な署名を与えます。
この署名があるので、必要なときにファンディングノードが資金を取り戻せるようなトランザクションをブロードキャストできます。

このメッセージにはチャネルを指定する `channel-id` を入れています。
このidは、ビッグエンディアンによる排他的OR(つまり `funding-output-index` は最後の２バイトだけになります)を使って `funding-txid` と `funding-output-index` を加え合わせることで作られたファンディングトランザクションから導かれるものです。todo

1. type: 35 (`funding_signed`)
2. data:
    * [`32`:`channel_id`]
    * [`64`:`signature`]


#### 要件


### `funding_locked` メッセージ


このメッセージは、ファンディングトランザクションが `accept_channel` メッセージの中で通知された `minimum-depth` フィールドの値分だけの承認数が確保されたことを示しています。
一度両方のノードがこれを送ると、チャネルは通常取引モードに移行します。

1. type: 36 (`funding_locked`)
2. data:
    * [`32`:`channel_id`]
    * [`33`:`next_per_commitment_point`]


#### 要件

#### Rationale

#### 将来

のちほどSPV proofやroute block hashを別のメッセージに追加するかもしれません。

## チャネルクローズ

ノードはコネクションの相互クローズを試みることができます。
これはunilateral close(一方的なクローズ)とは違うもので、両端ノードは直ちにそれぞれの資金にアクセスできるようになり、より低い手数料で行うことができます。

クロージングは２つのステージを経ます。
最初は、片方のノードがチャネルを清算(つまり、もう新しいHTLCを受け付けない)し、一度全てのHTLCを解決するとチャネルクローズの最終調整が始まります。

        +-------+                              +-------+
        |       |--(1)-----  shutdown  ------->|       |
        |       |<-(2)-----  shutdown  --------|       |
        |       |                              |       |
        |       | <complete all pending HTLCs> |       |
        |   A   |                 ...          |   B   |
        |       |                              |       |
        |       |<-(3)-- closing_signed  F1----|       |
        |       |--(4)-- closing_signed  F2--->|       |
        |       |              ...             |       |
        |       |--(?)-- closing_signed  Fn--->|       |
        |       |<-(?)-- closing_signed  Fn----|       |
        +-------+                              +-------+


### クロージング開始: `shutdown`

片方のノード(または両方)がクロージングを開始するために `shutdown` メッセージを送ります。
これにはscriptpubkeyが含まれます。


1. type: 38 (`shutdown`)
2. data:
   * [`32`:`channel_id`]
   * [`2`:`len`]
   * [`len`:`scriptpubkey`]


#### 要件


#### 合理性

もしシャットダウンを開始した時点でチャネルの状態が"きれい"(ペンディング状態の更新がない)であれば、きれいではない場合にどのようにするのかという点を避けることができます。
ファンディングノードは常に `commitment_signed` メッセージを最初に送るようにします。todo

シャットダウンはチャネルの停止要求を意味しているため、もう新しいHTLCは追加も了承もされません。

`scriptpubkey` はBitcoinネットワークに受け入れられる標準的な形式のみを含むようにしておき、生成されるトランザクションが確実にマイナーに伝搬するようにしておきます。

`shutdown` メッセージに返答するということは、未払いの変更を引き受けるためにこのノードが `commitment_signed` メッセージを返答前に送っているというです。
todo

### Closing negotiation: `closing_signed`

Once shutdown is complete and the channel is empty of HTLCs, the final
current commitment transactions will have no HTLCs, and closing fee
negotiation begins.  Each node chooses a fee it thinks is fair, and
signs the close transaction with the `scriptpubkey` fields from the
`shutdown` messages and that fee, and sends the signature.  The
process terminates when both agree on the same fee, or one side fails
the channel.

1. type: 39 (`closing_signed`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`fee_satoshis`]
   * [`64`:`signature`]

#### Requirements

Nodes SHOULD send a `closing_signed` message after `shutdown` has
been received and no HTLCs remain in either commitment transaction.

A sending node MUST set `fee_satoshis` lower than or equal to the
fee of the final commitment transaction.

The sender SHOULD set the initial `fee_satoshis` according to its
estimate of cost of inclusion in a block.

The sender MUST set `signature` to the Bitcoin signature of the close
transaction with the node responsible for paying the bitcoin fee
paying `fee_satoshis`, without populating any output which is below
its own `dust_limit_satoshis`. The sender MAY also eliminate its own
output from the mutual close transaction.

The receiver MUST check `signature` is valid for either the close
transaction with the given `fee_satoshis` as detailed above and its
own `dust_limit_satoshis` OR that same transaction with the sender's
output eliminated, and MUST fail the connection if it is not.

If the receiver agrees with the fee, it SHOULD reply with a
`closing_signed` with the same `fee_satoshis` value, otherwise it
SHOULD propose a value strictly between the received `fee_satoshis`
and its previously-sent `fee_satoshis`.

Once a node has sent or received a `closing_signed` with matching
`fee_satoshis` it SHOULD close the connection and SHOULD sign and
broadcast the final closing transaction.

#### Rationale

There is a possibility of irreparable differences on closing if one
node considers the other's output too small to allow propagation on
the bitcoin network (aka "dust"), and that other node instead
considers that output to be too valuable to discard.  This is why each
side uses its own `dust_limit_satoshis`, and the result can be a
signature validation failure, if they disagree on what the closing
transaction should look like.

However, if one side chooses to eliminate its own output, there's no
reason for the other side to fail the closing protocol, so this is
explicitly allowed.

Note that there is limited risk if the closing transaction is
delayed, and it will be broadcast very soon, so there is usually no
reason to pay a premium for rapid processing.

## Normal Operation

Once both nodes have exchanged `funding_locked` (and optionally `announcement_signatures`), the channel can be used to make payments via Hash TimeLocked Contracts.

Changes are sent in batches: one or more `update_` messages are sent before a
`commitment_signed` message, as in the following diagram:

        +-------+                            +-------+
        |       |--(1)---- add_htlc   ------>|       |
        |       |--(2)---- add_htlc   ------>|       |
        |       |<-(3)---- add_htlc   -------|       |
        |       |                            |       |
        |       |--(4)----   commit   ------>|       |
        |   A   |                            |   B   |
        |       |<-(5)--- revoke_and_ack-----|       |
        |       |<-(6)----   commit   -------|       |
        |       |                            |       |
        |       |--(7)--- revoke_and_ack---->|       |
        +-------+                            +-------+


Counterintuitively, these updates apply to the *other node's*
commitment transaction; the node only adds those updates to its own
commitment transaction when the remote node acknowledges it has
applied them via `revoke_and_ack`.

Thus each update traverses through the following states:

1. Pending on the receiver
2. In the receiver's latest commitment transaction,
3. ... and the receiver's previous commitment transaction has been revoked,
   and the HTLC is pending on the sender.
4. ... and in the sender's latest commitment transaction
5. ... and the sender's previous commitment transaction has been revoked


As the two nodes updates are independent, the two commitment
transactions may be out of sync indefinitely.  This is not concerning:
what matters is whether both sides have irrevocably committed to a
particular HTLC or not (the final state, above).

### Forwarding HTLCs

In general, a node offers HTLCs for two reasons: to initiate a payment of its own,
or to forward a payment coming from another node. In the forwarding case, care must
be taken to ensure that the *outgoing* HTLC cannot be redeemed unless the *incoming*
HTLC can be redeemed; these requirements ensure that is always true.

The addition/removal of an HTLC is considered *irrevocably committed* when:

1. the commitment transaction with/without it it is committed by both nodes, and any
previous commitment transaction which without/with it has been revoked, OR
2. the commitment transaction with/without it has been irreversibly committed to
the blockchain.

#### Requirements

A node MUST NOT offer an HTLC (`update_add_htlc`) in response to an incoming HTLC until
the incoming HTLC has been irrevocably committed.

A node MUST NOT fail an incoming HTLC (`update_fail_htlc`) for which it has committed
to an outgoing HTLC, until the removal of the outgoing HTLC is irrevocably committed.

A node SHOULD fulfill an incoming HTLC for which it has committed to an outgoing HTLC,
as soon as it receives `update_fulfill_htlc` for the outgoing HTLC.

#### Rationale

In general, we need to complete one side of the exchange before dealing with the other.
Fulfilling an HTLC is different: knowledge of the preimage is by definition irrevocable,
so we should fulfill the incoming HTLC as soon as we can to reduce latency.


### Risks With HTLC Timeouts


Once an HTLC has timed out where it could either be fulfilled or timed-out;
care must be taken around this transition both for offered and received HTLCs.

As a result of forwarding an HTLC from node A to node C, B will end up having an incoming
HTLC from A and an outgoing HTLC to C. B will make sure that the incoming HTLC has a greater
timeout than the outgoing HTLC, so that B can get refunded from C sooner than it has to refund
 A if the payment does not complete.

For example, node A might offer node B an HTLC with a timeout of 3 days, and node B might
offer node C the same HTLC with a timeout of 2 days:

```
    3 days timeout        2 days timeout
A ------------------> B ------------------> C
```

The difference in timeouts is called `cltv_expiry_delta` in
[BOLT #7](07-routing-gossip.md).

This difference is important: after 2 days B can try to
remove the offer to C even if C is unresponsive, by broadcasting the
commitment transaction it has with C and spending the HTLC output.
Even though C might race to try to use its payment preimage at that point to
also spend the HTLC, it should be resolved well before the 3 day
deadline so B can either redeem the HTLC off A or close it.


If the timing is too close, there is a risk of "one-sided redemption",
where the payment preimage received from an offered HTLC is too late
to be used for an incoming HTLC, leaving the node with unexpected
liability.


Thus the effective timeout of the HTLC is the `cltv_expiry`, plus some
additional delay for the transaction which redeems the HTLC output to
be irreversibly committed to the blockchain.

The fulfillment risk is similar: if a node C fulfills an HTLC after
its timeout, B might broadcast the commitment transaction and
immediately broadcast the HTLC timeout transaction.  In this scenario,
B would gain knowledge of the preimage without paying C.

#### Requirements

A node MUST estimate the deadline for successful redemption for each
HTLC.  A node MUST NOT offer a HTLC after this deadline, and
MUST fail the channel if an HTLC which it offered is in either node's
current commitment transaction past this deadline.

A node MUST NOT fulfill an HTLC after this deadline, and MUST fail the
connection if a HTLC it has fulfilled is in either node's current
commitment transaction past this deadline.

### Adding an HTLC: `update_add_htlc`


Either node can send `update_add_htlc` to offer a HTLC to the other,
which is redeemable in return for a payment preimage.  Amounts are in
millisatoshi, though on-chain enforcement is only possible for whole
satoshi amounts greater than the dust limit: in commitment transactions these are rounded down as
specified in [BOLT #3](03-transactions.md).


The format of the `onion_routing_packet` portion, which indicates where the payment
is destined, is described in [BOLT #4](04-onion-routing.md).


1. type: 128 (`update_add_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`4`:`amount_msat`]
   * [`4`:`cltv_expiry`]
   * [`32`:`payment_hash`]
   * [`1366`:`onion_routing_packet`]


#### Requirements

A sending node MUST NOT offer `amount_msat` it cannot pay for in the
remote commitment transaction at the current `feerate_per_kw` (see "Updating
Fees") while maintaining its channel reserve, MUST offer
`amount_msat` greater than 0, MUST NOT offer `amount_msat` below
the receiving node's `htlc_minimum_msat`, and MUST set `cltv_expiry` less
than 500000000.

A sending node MUST NOT add an HTLC if it would result in it offering
more than the remote's `max_accepted_htlcs` HTLCs in the remote commitment
transaction, or if the sum of total offered HTLCs would exceed the remote's
`max_htlc_value_in_flight_msat`.

A sending node MUST set `id` to 0 for the first HTLC it offers, and
increase the value by 1 for each successive offer.

A receiving node SHOULD fail the channel if it receives an
`amount_msat` equal to zero, below its own `htlc_minimum_msat`, or
which the sending node cannot afford at the current `feerate_per_kw` while
maintaining its channel reserve.  A receiving node SHOULD fail the
channel if a sending node adds more than its `max_accepted_htlcs` HTLCs to
its local commitment transaction, or adds more than its `max_htlc_value_in_flight_msat` worth of offered HTLCs to its local commitment transaction, or
sets `cltv_expiry` to greater or equal to 500000000.

A receiving node MUST allow multiple HTLCs with the same payment hash.

A receiving node MUST ignore a repeated `id` value after a
reconnection if the sender did not previously acknowledge the
commitment of that HTLC.  A receiving node MAY fail the channel if
other `id` violations occur.

The `onion_routing_packet` contains an obfuscated list of hops and instructions for each hop along the path.
It commits to the HTLC by setting the `payment_hash` as associated data, i.e., including the `payment_hash` in the computation of HMACs.
This prevents replay attacks that'd reuse a previous `onion_routing_packet` with a different `payment_hash`.

#### Rationale


Invalid amounts are a clear protocol violation and indicate a
breakdown.


If a node did not accept multiple HTLCs with the same payment hash, an
attacker could probe to see if a node had an existing HTLC.  This
requirement deal with duplicates leads us to using a separate
identifier; we assume a 64 bit counter never wraps.


Retransmissions of unacknowledged updates are explicitly allowed for
reconnection purposes; allowing them at other times simplifies the
recipient code, though strict checking may help debugging.

`max_accepted_htlcs` is limited to 483, to ensure that even if both
sides send the maximum number of HTLCs, the `commitment_signed` message will
still be under the maximum message size.  It also ensures that
a single penalty transaction can spend the entire commitment transaction,
as calculated in [BOLT #5](05-onchain.md#penalty-transaction-weight-calculation).

`cltv_expiry` values equal or above 500000000 would indicate a time in
seconds, and the protocol only supports an expiry in blocks.

### Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc` and `update_fail_malformed_htlc`

For simplicity, a node can only remove HTLCs added by the other node.
There are three reasons for removing an HTLC: it has timed out, it has
failed to route, or the payment preimage is supplied.

The `reason` field is an opaque encrypted blob for the benefit of the
original HTLC initiator as defined in [BOLT #4](04-onion-routing.md),
but there's a special malformed failure variant for the case where
our peer couldn't parse it; in this case the current node encrypts
it into a `update_fail_htlc` for relaying.

1. type: 130 (`update_fulfill_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`32`:`payment_preimage`]

For a timed out or route-failed HTLC:

1. type: 131 (`update_fail_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`2`:`len`]
   * [`len`:`reason`]

For a unparsable HTLC:

1. type: 135 (`update_fail_malformed_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`32`:`sha256_of_onion`]
   * [`2`:`failure_code`]

#### Requirements

A node SHOULD remove an HTLC as soon as it can; in particular, a node
SHOULD fail an HTLC which has timed out.

A node MUST NOT send `update_fulfill_htlc` until an HTLC is
irrevocably committed in both sides' commitment transactions.

A receiving node MUST check that `id` corresponds to an HTLC in its
current commitment transaction, and MUST fail the channel if it does
not.

A receiving node MUST check that the `payment_preimage` value in
`update_fulfill_htlc` SHA256 hashes to the corresponding HTLC
`payment_hash`, and MUST fail the channel if it does not.

A receiving node MUST fail the channel if the `BADONION` bit in
`failure_code` is not set for `update_fail_malformed_htlc`.

A receiving node MAY check the `sha256_of_onion` in
`update_fail_malformed_htlc` and MAY retry or choose an alternate
error response if it does not match the onion it sent.

Otherwise, a receiving node which has an outgoing HTLC canceled by
`update_fail_malformed_htlc` MUST return an error in the
`update_fail_htlc` sent to the link which originally sent the HTLC
using the `failure_code` given and setting the data to
`sha256_of_onion`.

#### Rationale

A node which doesn't time out HTLCs risks channel failure (see
"Risks With HTLC Timeouts").

A node which sends `update_fulfill_htlc` before the sender is also
committed to the HTLC risks losing funds.

If the onion is malformed, the upstream node won't be able to extract
a key to generate a response, hence the special failure message which
makes this node do it.

The node can check that the SHA256 the upstream is complaining about
does match the onion it sent, which may allow it to detect random bit
errors.  Without re-checking the actual encrypted packet sent, however,
it won't know whether the error was its own or on the remote side, so
such detection is left as an option.

### Committing Updates So Far: `commitment_signed`


When a node has changes for the remote commitment, it can apply them,
sign the resulting transaction as defined in [BOLT #3](03-transactions.md) and send a
`commitment_signed` message.


1. type: 132 (`commitment_signed`)
2. data:
   * [`32`:`channel_id`]
   * [`64`:`signature`]
   * [`2`:`num_htlcs`]
   * [`num_htlcs*64`:`htlc_signature`]

#### Requirements


A node MUST NOT send a `commitment_signed` message which does not include any
updates.  Note that a node MAY send a `commitment_signed` message which only
alters the fee, and MAY send a `commitment_signed` message which doesn't
change the commitment transaction other than the new revocation hash
(due to dust, identical HTLC replacement, or insignificant or multiple
fee changes).  A node MUST include one `htlc_signature` for every HTLC
transaction corresponding to BIP69 lexicographic ordering of the commitment
transaction.


A receiving node MUST fail the channel if `signature` is not valid for
its local commitment transaction once all pending updates are applied.
A receiving node MUST fail the channel if `num_htlcs` is not equal to
the number of HTLC outputs in the local commitment transaction once all
pending updates are applied.  A receiving node MUST fail the channel if
any `htlc_signature` is not valid for the corresponding HTLC transaction.


A receiving node MUST respond with a `revoke_and_ack` message.


#### Rationale


There's little point offering spam updates; it implies a bug.


The `num_htlcs` field is redundant, but makes the packet length check fully self-contained.


### Completing the transition to the updated state: `revoke_and_ack`


Once the recipient of `commitment_signed` checks the signature, it knows that
it has a valid new commitment transaction, replies with the commitment
preimage for the previous commitment transaction in a `revoke_and_ack`
message.


This message also implicitly serves as an acknowledgment of receipt
of the `commitment_signed`, so it's a logical time for the `commitment_signed` sender
to apply to its own commitment, any pending updates it sent before
that `commitment_signed`.


The description of key derivation is in [BOLT #3](03-transactions.md#key-derivation).


1. type: 133 (`revoke_and_ack`)
2. data:
   * [`32`:`channel_id`]
   * [`32`:`per_commitment_secret`]
   * [`33`:`next_per_commitment_point`]

#### Requirements


A sending node MUST set `per_commitment_secret` to the secret used to generate keys for the
previous commitment transaction, MUST set `next_per_commitment_point` to the values for its next commitment transaction.

A receiving node MUST check that `per_commitment_secret` generates the previous `per_commitment_point`, and MUST fail if it does not. A receiving node MAY fail if the `per_commitment_secret` was not generated by the protocol in [BOLT #3](03-transactions.md#per-commitment-secret-requirements).

Nodes MUST NOT broadcast old (revoked) commitment transactions; doing
so will allow the other node to seize all the funds.  Nodes SHOULD NOT
sign commitment transactions unless it is about to broadcast them (due
to a failed connection), to reduce this risk.

### Updating Fees: `update_fee`

An `update_fee` message is sent by the node which is paying the
bitcoin fee.  Like any update, it is first committed to the receiver's
commitment transaction, then (once acknowledged) committed to the
sender's.  Unlike an HTLC, `update_fee` is never closed, simply
replaced.

There is a possibility of a race: the recipient can add new HTLCs
before it receives the `update_fee`, and the sender may not be able to
afford the fee on its own commitment transaction once the `update_fee`
is acknowledged by the recipient.  In this case, the fee will be less
than the fee rate, as described in [BOLT #3](03-transactions.md#fee-payment).

The exact calculation used for deriving the fee from the fee rate is
given in [BOLT #3](03-transactions.md#fee-calculation).


1. type: 134 (`update_fee`)
2. data:
   * [`32`:`channel_id`]
   * [`4`:`feerate_per_kw`]

#### Requirements

The node which is responsible for paying the bitcoin fee SHOULD send
`update_fee` to ensure the current fee rate is sufficient for
timely processing of the commitment transaction by a significant
margin.

The node which is not responsible for paying the bitcoin fee MUST NOT
send `update_fee`.

A receiving node SHOULD fail the channel if the `update_fee` is too
low for timely processing, or unreasonably large.

A receiving node MUST fail the channel if the sender is not
responsible for paying the bitcoin fee.

A receiving node SHOULD fail the channel if the sender cannot afford
the new fee rate on the receiving node's current commitment
transaction, but it MAY delay this check until the `update_fee` is
committed.

#### Rationale

Bitcoin fees are required for unilateral closes to be effective,
particularly since there is no general method for the node which
broadcasts it to use child-pays-for-parent to increase its effective
fee.

Given the variance in fees, and the fact that the transaction may be
spent in the future, it's a good idea for the fee payer to keep a good
margin, say 5x the expected fee requirement, but differing methods of
fee estimation mean we don't specify an exact value.

Since the fees are currently one-sided (the party which requested the
channel creation always pays the fees for the commitment transaction),
it is simplest to only allow them to set fee levels, but as the same
fee rate applies to HTLC transactions, the receiving node must also
care about the reasonableness of the fee.

## Message Retransmission

Because communication transports are unreliable and may need to be
re-established from time to time, the design of the transport has been
explicitly separated from the protocol.

Nonetheless, we assume that our transport is ordered and reliable;
reconnection introduces doubt as to what has been received, so we
retransmit any channel messages which may not have been.

This is fairly straightforward in the case of channel establishment
and close where messages have an explicit order, but in normal
operation acknowledgments of updates are delayed until the
`commitment_signed` / `revoke_and_ack` exchange, so we cannot assume
the updates have been received.  This also means that the receiving
node only needs to store updates upon receipt of `commitment_signed`.

Note that messages described in [BOLT #7](07-routing-gossip.md) are
independent of particular channels; their transmission requirements
are covered there, and other than being transmitted after `init` (like
any message), they are independent of requirements here.

### Requirements

A node MUST handle continuing a previous channel on a new encrypted
transport.

On disconnection, the funder MUST remember the channel for
reconnection if it has broadcast the funding transaction, otherwise it
MUST NOT.

On disconnection, the non-funding node MUST remember the channel for
reconnection if it has sent the `funding_signed` message, otherwise
it MUST NOT.

On disconnection, a node MUST reverse any uncommitted updates sent by
the other side (ie. all messages beginning with `update_` for which no
`commitment_signed` has been received).  Note that a node MAY have
already use the `payment_preimage` value from the `update_fulfill_htlc`,
so the effects of `update_fulfill_htlc` is not completely reversed.

On reconnection, if a channel is in an error state, the node SHOULD
retransmit the error packet and ignore any other packets for that
channel, or if the channel has entered closing negotiation, the node
MUST retransmit the last `closing_signed`.

Otherwise, on reconnection, a node MUST retransmit old messages after `funding_signed` which may not
have been received, and MUST NOT retransmit old messages which have
been explicitly or implicitly acknowledged.  The following table
lists the acknowledgment conditions for each message:

* `funding_locked`: acknowledged by `update_` messages, `commitment_signed`, `revoke_and_ack` or `shutdown` messages.
* `update_` messages: acknowledged by `revoke_and_ack`.
* `commitment_signed`: acknowledged by `revoke_and_ack`.
* `revoke_and_ack`: acknowledged by `commitment_signed` or `closing_signed`
* `shutdown`: acknowledged by `closing_signed`.

Before retransmitting `commitment_signed`, the node MUST send
appropriate `update_` messages (the other node will have forgotten
them, as required above).

A node MAY simply retransmit messages which are identical to the
previous transmission.  A node MUST not assume that
previously-transmitted messages were lost: in particular, if it has
sent a previous `commitment_signed` message, a node MUST handle the
case where the corresponding commitment transaction is broadcast by
the other side at any time.  This is particularly important if a node
does not simply retransmit the exact same `update_` messages as
previously sent.

A receiving node MAY ignore spurious message retransmission, or MAY
fail the channel if they occur.

### Rationale

The effect of requirements above are that the opening phase is almost
atomic: if it doesn't complete, it starts again.  The only exception
is where the `funding_signed` message is sent and not received: in
this case, the funder will forget the channel and presumably open
a new one on reconnect; the other node will eventually forget the
original channel due to never receiving `funding_locked` or seeing
the funding transaction on-chain.

There's no acknowledgment for `error`, so if a reconnect occurs it's
polite to retransmit before disconnecting again, but it's not a MUST
because there are also occasions where a node can simply forget the
channel altogether.

There is similarly no acknowledgment for `closing_signed`, so it
is also retransmitted on reconnection.

# Authors

FIXME

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
