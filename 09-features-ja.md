# BOLT #9: 割り当てられた機能フラグ

このドキュメントは、`init` メッセージ（[BOLT #1](01-messaging.md)）内の `features` フラグの割り当て、および `channel_announcement` と `node_announcement` メッセージ（[BOLT #7](07-routing-gossip.md)）内の `features` フィールドの割り当てを追跡します。フラグは別々に追跡されます。なぜなら、新しいフラグが時間とともに追加される可能性があるからです。

いくつかの機能は導入され、非常に広く普及したため、すべてのノードによって `ASSUMED` とされ、無視しても安全です（その意味はこの仕様の以前の改訂版でのみ定義されています）。

フラグは最下位ビットから番号が付けられ、ビット 0 (すなわち 0x1、_偶数_ ビット) です。通常、フラグはペアで割り当てられ、機能をオプションとして導入し (_奇数_ ビット)、後に必須 (_偶数_ ビット) にアップグレードすることができます。これは古いノードによって拒否されます： [BOLT #1: The `init` Message](01-messaging.md#the-init-message) を参照してください。

いくつかの機能は、チャネルごとまたはノードごとに意味を持たないため、各機能はそれがどのようにそのコンテキストで提示されるかを定義します。いくつかの機能はチャネルを開くために必要かもしれませんが、チャネルの使用には必須ではないため、それらの機能の提示は機能自体に依存します。

コンテキスト列は次のように解釈されます：

* `I`: `init` メッセージで提示されます。
* `N`: `node_announcement` メッセージで提示されます。
* `C`: `channel_announcement` メッセージで提示されます。
* `C-`: `channel_announcement` メッセージで提示されますが、常に奇数 (オプション) です。
* `C+`: `channel_announcement` メッセージで提示されますが、常に偶数 (必須) です。
* `9`: [BOLT 11](11-payment-encoding.md) の請求書で提示されます。
* `B`: ブラインドパスの `allowed_features` フィールドで提示されます。

| Bits  | Name                              | Description                                               | Context  | Dependencies              | Link                                                                  |
|-------|-----------------------------------|-----------------------------------------------------------|----------|---------------------------|-----------------------------------------------------------------------|
| 0/1   | `option_data_loss_protect`        | ASSUMED                                                   |          |                           |                                                                       |
| 4/5   | `option_upfront_shutdown_script`  | チャネルを開くときにシャットダウン scriptpubkey にコミット | IN       |                           | [BOLT #2][bolt02-open]                                                |
| 6/7   | `gossip_queries`                  | ピアが有用なゴシップを共有する                             |          |                           |                                                                       |
| 8/9   | `var_onion_optin`                 | ASSUMED                                                   |          |                           |                                                                       |
| 10/11 | `gossip_queries_ex`               | ゴシップクエリが追加情報を含むことができる                 | IN       |                           | [BOLT #7][bolt07-query]                                               |
| 12/13 | `option_static_remotekey`         | ASSUMED                                                   |          |                           |                                                                       |
| 14/15 | `payment_secret`                  | ノードが `payment_secret` フィールドをサポート             | IN9      |                           | [Routing Onion Specification][bolt04]                                 |
| 16/17 | `basic_mpp`                       | ノードが基本的なマルチパート支払いを受け取ることができる   | IN9      | `payment_secret`          | [BOLT #4][bolt04-mpp]                                                 |
| 18/19 | `option_support_large_channel`    | 大きなチャネルを作成できる                                | IN       |                           | [BOLT #2](02-peer-protocol.md#the-open_channel-message)               |
| 22/23 | `option_anchors`                  | ゼロ手数料 HTLC トランザクションを持つアンカーコミットメントタイプ | IN       |                           | [BOLT #3][bolt03-htlc-tx], [lightning-dev][ml-sighash-single-harmful] |
| 24/25 | `option_route_blinding`           | ノードがブラインドパスをサポート                          | IN9      |                           | [BOLT #4][bolt04-route-blinding]                                      |
| 26/27 | `option_shutdown_anysegwit`       | `shutdown` で将来の segwit バージョンを許可               | IN       |                           | [BOLT #2][bolt02-shutdown]                                            |
| 28/29 | `option_dual_fund`                | チャネルオープンの v2 を使用し、デュアルファンディングを可能にする | IN       |                           | [BOLT #2](02-peer-protocol.md)                                        |
| 34/35 | `option_quiesce`                  | `stfu` メッセージのサポート                               | IN       |                           | [BOLT #2][bolt02-quiescence]                                          |
| 38/39 | `option_onion_messages`           | オニオンメッセージを転送できる                            | IN       |                           | [BOLT #7](04-onion-routing.md#onion-messages)                         |
| 44/45 | `option_channel_type`             | ノードが open/accept で `channel_type` フィールドをサポート | IN       |                           | [BOLT #2](02-peer-protocol.md#the-open_channel-message)               |
| 46/47 | `option_scid_alias`               | ルーティングのためのチャネルエイリアスを提供              | IN       |                           | [BOLT #2][bolt02-channel-ready]                                       |
| 48/49 | `option_payment_metadata`         | tlv レコード内の支払いメタデータ                          | 9        |                           | [BOLT #11](11-payment-encoding.md#tagged-fields)                      |
| 50/51 | `option_zeroconf`                 | ゼロコンフチャネルタイプを理解                             | IN       | `option_scid_alias`       | [BOLT #2][bolt02-channel-ready]                                       |

## Requirements

起点ノード：
  * 上記の機能をサポートする場合、対応する奇数ビットを、Context 列で示されたすべての機能フィールドに設定する「べき」です。ただし、偶数ビットを設定する必要があると示されている場合を除きます。
  * 上記の機能を必要とする場合、対応する偶数ビットを、Context 列で示されたすべての機能フィールドに設定する「必要があります」。ただし、奇数ビットを設定する必要があると示されている場合を除きます。
  * サポートしていない機能ビットを設定しては「なりません」。
  * 上記の表で指定されていないフィールドに機能ビットを設定しては「なりません」。
  * オプションビットと必須ビットの両方を設定しては「なりません」。
  * すべての推移的な機能依存関係を設定する「必要があります」。
  * 以下をサポートする「必要があります」：
    * `var_onion_optin`

受信ノード：
  * ペア内でオプションビットと必須ビットの両方が設定されている場合、その機能は必須として扱われるべきです。

特定のビットを受信するための要件は、上記の表のリンクされたセクションで定義されています。
上記で定義されていない機能ビットの要件は、[BOLT #1: The `init` Message](01-messaging.md#the-init-message) に記載されています。

## Rationale

`node_announcement` と [BOLT 11](11-payment-encoding.md) 請求書の両方のコンテキストで利用可能な機能フラグについては、[BOLT 11](11-payment-encoding.md) 請求書で設定された機能が `node_announcement` で設定されたものを上書きするべきです。これは、[BOLT 7](07-routing-gossip.md#the-node_announcement-message) で指定された未知の機能の動作と一貫性を保ちます。

起点は、適切に形成された機能ベクトルを作成するために、すべての推移的な機能依存関係を設定する必要があります。すべての既知の依存関係を事前に検証することで、単一の機能ビットに基づくロジックが簡素化されます。機能の依存関係が設定されていることがわかっており、各機能ゲートで検証する必要がありません。

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。


[bolt02-retransmit]: 02-peer-protocol.md#message-retransmission
[bolt02-open]: 02-peer-protocol.md#the-open_channel-message
[bolt03-htlc-tx]: 03-transactions.md#htlc-timeout-and-htlc-success-transactions
[bolt02-shutdown]: 02-peer-protocol.md#closing-initiation-shutdown
[bolt02-quiescence]: 02-peer-protocol.md#channel-quiescence
[bolt02-channel-ready]: 02-peer-protocol.md#the-channel_ready-message
[bolt04]: 04-onion-routing.md
[bolt07-sync]: 07-routing-gossip.md#initial-sync
[bolt07-query]: 07-routing-gossip.md#query-messages
[bolt04-mpp]: 04-onion-routing.md#basic-multi-part-payments
[bolt04-route-blinding]: 04-onion-routing.md#route-blinding
[ml-sighash-single-harmful]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-September/002796.html
