# BOLT #9: 割り当てられた機能フラグ

このドキュメントは、`init` メッセージ（[BOLT #1](01-messaging.md)）内の `features` フラグの割り当て、および `channel_announcement` と `node_announcement` メッセージ（[BOLT #7](07-routing-gossip.md)）内の `features` フィールドの割り当てを記録します。新しいフラグは時間とともに追加されていく可能性があるため、これらのフラグはそれぞれ別々に管理されます。

一部の機能は導入後に広く普及し、すべてのノードに存在することが `ASSUMED`（前提）とされるようになりました。これらは無視しても安全で、その意味論は本仕様の以前の改訂版でのみ定義されています。

フラグは最下位ビット（ビット 0、すなわち 0x1、_偶数_ ビット）から番号が付けられます。通常はペアで割り当てられ、機能を最初はオプション（_奇数_ ビット）として導入し、後に必須（_偶数_ ビット）へ昇格できるようになっています。必須化されると古いノードでは拒否されます。詳細は [BOLT #1: The `init` Message](01-messaging.md#the-init-message) を参照してください。

機能の中にはチャネル単位やノード単位で扱うことが意味を持たないものもあるため、各機能はどのコンテキストでどのように提示されるかを定義します。また、チャネルを開く際には必要でも、開いた後の利用には必須ではない機能もあり、その提示方法は機能ごとに異なります。

Context 列は次のように解釈します。

* `I`: `init` メッセージで提示されます。
* `N`: `node_announcement` メッセージで提示されます。
* `C`: `channel_announcement` メッセージで提示されます。
* `C-`: `channel_announcement` メッセージで提示されますが、常に奇数（オプション）です。
* `C+`: `channel_announcement` メッセージで提示されますが、常に偶数（必須）です。
* `9`: [BOLT 11](11-payment-encoding.md) インボイスで提示されます。
* `B`: ブラインドパスの `allowed_features` フィールドで提示されます。
* `T`: [チャネルを開く際](02-peer-protocol.md#the-open_channel-message)の `channel_type` フィールドで使用されます。

| Bits  | Name                              | Description                                               | Context  | Dependencies                | Link                                                                  |
|-------|-----------------------------------|-----------------------------------------------------------|----------|-----------------------------|-----------------------------------------------------------------------|
| 0/1   | `option_data_loss_protect`        | ASSUMED                                                   |          |                             |                                                                       |
| 4/5   | `option_upfront_shutdown_script`  | チャネルを開く際にシャットダウン用 scriptpubkey にコミットする | IN       |                             | [BOLT #2][bolt02-open]                                                |
| 6/7   | `gossip_queries`                  | ピアが共有する価値のあるゴシップを持っている               |          |                             |                                                                       |
| 8/9   | `var_onion_optin`                 | ASSUMED                                                   |          |                             |                                                                       |
| 10/11 | `gossip_queries_ex`               | ゴシップクエリに追加情報を含められる                       | IN       |                             | [BOLT #7][bolt07-query]                                               |
| 12/13 | `option_static_remotekey`         | ASSUMED                                                   |          |                             |                                                                       |
| 14/15 | `payment_secret`                  | ASSUMED                                                   | IN9      |                             | [Routing Onion Specification][bolt04]                                 |
| 16/17 | `basic_mpp`                       | ノードが基本的なマルチパート支払いを受け取れる             | IN9      | `payment_secret`            | [BOLT #4][bolt04-mpp]                                                 |
| 18/19 | `option_support_large_channel`    | 大規模なチャネルを作成できる                               | IN       |                             | [BOLT #2](02-peer-protocol.md#the-open_channel-message)               |
| 22/23 | `option_anchors`                  | ゼロ手数料 HTLC トランザクションを伴うアンカーコミットメントタイプ | INT      |                             | [BOLT #3][bolt03-htlc-tx], [lightning-dev][ml-sighash-single-harmful] |
| 24/25 | `option_route_blinding`           | ノードがブラインドパスをサポートする                       | IN9      |                             | [BOLT #4][bolt04-route-blinding]                                      |
| 26/27 | `option_shutdown_anysegwit`       | `shutdown` で将来の segwit バージョンを許可する            | IN       |                             | [BOLT #2][bolt02-shutdown]                                            |
| 28/29 | `option_dual_fund`                | チャネルオープンの v2 を使用し、デュアルファンディングを有効化する | IN       |                             | [BOLT #2](02-peer-protocol.md)                                        |
| 34/35 | `option_quiesce`                  | `stfu` メッセージをサポートする                            | IN       |                             | [BOLT #2][bolt02-quiescence]                                          |
| 36/37 | `option_attribution_data`      | `update_fail_htlc` および `update_fulfill_htlc` で帰属データを生成・中継できる  | IN9      |                   | [BOLT #4][bolt04-attribution-data]   |
| 38/39 | `option_onion_messages`           | オニオンメッセージを転送できる                             | IN       |                             | [BOLT #7](04-onion-routing.md#onion-messages)                         |
| 42/43 | `option_provide_storage`          | 他のノードの暗号化バックアップデータを保管できる           | IN       |                             | [BOLT #1](01-messaging.md#peer-storage)                               |
| 44/45 | `option_channel_type`             | ASSUMED                                                   |          |                             |                                                                       |
| 46/47 | `option_scid_alias`               | ルーティング用のチャネルエイリアスを提供する               | INT      |                             | [BOLT #2][bolt02-channel-ready]                                       |
| 48/49 | `option_payment_metadata`         | tlv レコード内の支払いメタデータ                           | 9        |                             | [BOLT #11](11-payment-encoding.md#tagged-fields)                      |
| 50/51 | `option_zeroconf`                 | ゼロコンフィルメーションのチャネルタイプを理解する         | INT      | `option_scid_alias`         | [BOLT #2][bolt02-channel-ready]                                       |
| 60/61 | `option_simple_close`             | 簡素化されたクローズ交渉                                   | IN       | `option_shutdown_anysegwit` | [BOLT #2][bolt02-simple-close]                                        |
| 62/63 | `option_splice`                   | 資金調達トランザクションを新しいものに置き換えられる       | IN       |                             | [BOLT #2](02-peer-protocol.md#channel-splicing)                       |

## Requirements

起点ノードは:
  * 上記の機能をサポートする場合、Context 列で示されるすべての機能フィールドにおいて、対応する奇数ビットを設定すべきである。ただし、代わりに偶数の機能ビットを設定しなければならないと示されている場合を除く。
  * 上記の機能を必須とする場合、Context 列で示されるすべての機能フィールドにおいて、対応する偶数の機能ビットを設定しなければならない。ただし、代わりに奇数の機能ビットを設定しなければならないと示されている場合を除く。
  * サポートしていない機能ビットを設定してはならない。
  * 上記の表で指定されていないフィールドに機能ビットを設定してはならない。
  * オプションのビットと必須のビットを同時に設定してはならない。
  * 推移的な機能依存関係をすべて設定しなければならない。
  * 以下をサポートしなければならない:
    * `var_onion_optin`

受信ノードは:
  * ペア内でオプションのビットと必須のビットの両方が設定されている場合、その機能は必須として扱うべきである。

特定のビット受信時の要件は、上記の表でリンクされた各セクションで定義されています。
上記で定義されていない機能ビットの要件は、[BOLT #1: The `init` Message](01-messaging.md#the-init-message) に記載されています。

## Rationale

`node_announcement` と [BOLT 11](11-payment-encoding.md) インボイスの両方のコンテキストで利用可能な機能フラグについては、[BOLT 11](11-payment-encoding.md) インボイスで設定された機能が `node_announcement` で設定されたものを上書きすべきです。これは [BOLT 7](07-routing-gossip.md#the-node_announcement-message) で規定されている未知の機能の扱いとも整合しています。

起点は、整合の取れた機能ベクトルを構築するために、推移的な機能依存関係をすべて設定しなければなりません。既知の依存関係をあらかじめすべて検証しておくことで、単一の機能ビットに依存するロジックが簡潔になります。機能の依存関係が設定済みであることが保証されるため、各機能ゲートで都度検証する必要がなくなります。

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。

[bolt02-retransmit]: 02-peer-protocol.md#message-retransmission
[bolt02-open]: 02-peer-protocol.md#the-open_channel-message
[bolt02-simple-close]: 02-peer-protocol.md#closing-negotiation-closing_complete-and-closing_sig
[bolt03-htlc-tx]: 03-transactions.md#htlc-timeout-and-htlc-success-transactions
[bolt02-shutdown]: 02-peer-protocol.md#closing-initiation-shutdown
[bolt02-quiescence]: 02-peer-protocol.md#channel-quiescence
[bolt02-channel-ready]: 02-peer-protocol.md#the-channel_ready-message
[bolt04-attribution-data]: 04-onion-routing.md#returning-errors
[bolt07-sync]: 07-routing-gossip.md#initial-sync
[bolt07-query]: 07-routing-gossip.md#query-messages
[bolt04-mpp]: 04-onion-routing.md#basic-multi-part-payments
[bolt04-route-blinding]: 04-onion-routing.md#route-blinding
[bolt04-attributable-errors]: 04-onion-routing.md
[ml-sighash-single-harmful]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-September/002796.html
