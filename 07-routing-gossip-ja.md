# BOLT #7: P2P ノードおよびチャネルの発見

この仕様書では、第三者に依存せずに情報を伝達するための、シンプルなノード発見、チャネル発見、およびチャネル更新の仕組みについて説明します。

ノード発見とチャネル発見は、それぞれ異なる目的を持ちます。

- ノード発見では、ノードが自身の ID、ホスト、およびポートをブロードキャストすることで、他のノードが接続を開いて支払いチャネルを確立できるようにします。
- チャネル発見では、ネットワークトポロジーのローカルビューを作成・維持することで、ノードが目的の宛先へのルートを発見できるようにします。

チャネル発見とノード発見をサポートするために、3 種類の *ゴシップメッセージ* が定義されています。

- ノード発見のために、ピアは `node_announcement` メッセージを交換し、ノードに関する追加情報を提供します。ノード情報を更新するために、複数の `node_announcement` メッセージが存在することがあります。

- チャネル発見のために、ネットワーク内のピアは、2 つのノード間の新しいチャネルに関する情報を含む `channel_announcement` メッセージを交換します。また、チャネルに関する情報を更新する `channel_update` メッセージを交換することもあります。任意のチャネルに対して有効な `channel_announcement` は 1 つだけですが、少なくとも 2 つの `channel_update` メッセージが期待されます。

# 目次

  * [`short_channel_id` の定義](#definition-of-short_channel_id)
  * [`announcement_signatures` メッセージ](#the-announcement_signatures-message)
  * [`channel_announcement` メッセージ](#the-channel_announcement-message)
  * [`node_announcement` メッセージ](#the-node_announcement-message)
  * [`channel_update` メッセージ](#the-channel_update-message)
  * [クエリメッセージ](#query-messages)
  * [再放送](#rebroadcasting)
  * [HTLC 手数料](#htlc-fees)
  * [ネットワークビューの剪定](#pruning-the-network-view)
  * [ルーティングの推奨事項](#recommendations-for-routing)
  * [参考文献](#references)

## `short_channel_id` の定義

`short_channel_id` は、資金調達トランザクションを一意に表す識別子であり、次の構造を持ちます。
  1. 最上位 3 バイト: ブロックの高さ
  2. 次の 3 バイト: ブロック内のトランザクションインデックス
  3. 最下位 2 バイト: チャネルへ支払う出力インデックス

`short_channel_id` の標準的な人間可読形式は、ブロックの高さ、トランザクションインデックス、出力インデックスをこの順に並べ、各要素を 10 進数で表記し、小文字の `x` で区切って作成します。例えば `539268x845x1` は、ブロック高 539268・トランザクションインデックス 845・出力 1 にあるチャネルを示します。

### Rationale

`short_channel_id` の人間可読形式は、ほとんどのシステムでダブルクリックやダブルタップにより ID 全体を選択できるように設計されています。人間は数字を 10 進数で読むのを好むため、各要素は 10 進数で表記されます。小文字の `x` は、多くのフォントで 10 進数の数字より視覚的に小さく見えるため、ID の各要素を視覚的にグループ化しやすくする目的で採用されています。

## `announcement_signatures` メッセージ

これはチャネルの両端点間で直接やり取りされるメッセージであり、チャネルをネットワーク全体にアナウンスすることを許可するオプトイン機構として機能します。送信者による、`channel_announcement` メッセージの構築に必要な署名を含みます。

1. type: 259 (`announcement_signatures`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`short_channel_id`:`short_channel_id`]
    * [`signature`:`node_signature`]
    * [`signature`:`bitcoin_signature`]

開始ノードがチャネルをアナウンスする意思は、チャネル開設時に `channel_flags` の `announce_channel` ビットを設定することで示されます ([BOLT #2](02-peer-protocol.md#the-open_channel-message) を参照)。

### Requirements

`announcement_signatures` メッセージは、新たに確立されたチャネルに対応する `channel_announcement` メッセージを構築し、それを各エンドポイントの `node_id` および `bitcoin_key` に対応する秘密鍵で署名することで作成されます。署名後に `announcement_signatures` メッセージを送信できます。

ノードは:
  - `open_channel` メッセージで `announce_channel` ビットが設定されており、かつ `shutdown` メッセージがまだ送信されていない場合:
    - `channel_ready` を送受信し、かつ資金調達トランザクションが再編成されない程度の十分な確認を得た後で:
      - 当該資金調達トランザクションに対する `announcement_signatures` を送信しなければならない。
    - `splice_locked` を送受信し、かつスプライストランザクションが再編成されない程度の十分な確認を得た後で:
      - 対応するスプライストランザクションに対する `announcement_signatures` を送信しなければならない。
  - それ以外の場合:
    - `announcement_signatures` メッセージを送信してはならない。
  - 再接続時 (上記のタイミング要件が満たされた後):
    - 資金調達トランザクションに対する `announcement_signatures` をまだ受信していない場合:
      - 自身の `announcement_signatures` メッセージを送信しなければならない。
    - 資金調達トランザクションに対する `announcement_signatures` を受信した場合:
      - 自身の `announcement_signatures` メッセージで応答しなければならない。
    - スプライストランザクションに対する `announcement_signatures` をまだ受信していない場合:
      - `my_current_funding_locked` の `retransmit_flags` 内の `announcement_signatures` ビットを設定しなければならない。
    - *リモート* の `retransmit_flags` で `announcement_signatures` ビットが設定されている場合:
      - 自身の `announcement_signatures` メッセージを再送信しなければならない。

受信ノードは:
  - `short_channel_id` が自身の資金調達トランザクションのいずれにも一致しない場合:
    - `warning` を送信すべきである。
  - `node_signature` または `bitcoin_signature` が正しくない場合:
    - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させてよい。
  - 有効な `announcement_signatures` メッセージを送受信した場合:
    - 資金調達トランザクションが少なくとも 6 回の確認を得ているならば:
      - `channel_announcement` メッセージをピア向けにキューイングすべきである。
  - `channel_ready` をまだ送信していない場合:
    - `channel_ready` を送信した後まで `announcement_signatures` の処理を延期すべきである。
  - この `short_channel_id` に一致するトランザクションに対する `splice_locked` をまだ送信していない場合:
    - `splice_locked` を送信した後まで `announcement_signatures` の処理を延期すべきである。

### Rationale

ブロックチェーンの再編成が発生すると `short_channel_id` が無効になってしまうため、資金調達トランザクションが十分な確認を得るまでチャネルをアナウンスしてはなりません。

スプライシングが用いられる場合、双方が `splice_locked` を送信した時点で各スプライストランザクションごとに `channel_announcement` が生成されます。これによって、現在アクティブなチャネルを消費するトランザクションがクローズではなくスプライスであることをネットワークに通知でき、当該チャネルは更新後の `short_channel_id` のもとで引き続き利用できます。

## `channel_announcement` メッセージ

このゴシップメッセージは、チャネルの所有権に関する情報を含みます。各オンチェーンの Bitcoin 鍵を、それに対応する Lightning ノードの鍵と結び付け、また逆方向にも結び付けます。少なくとも一方が `channel_update` を用いて手数料レベルや有効期限をアナウンスするまでは、チャネルは実用上使用できません。

`node_1` と `node_2` の間にチャネルが存在することを証明するには、次の事項が必要です。

1. 資金調達トランザクションが `bitcoin_key_1` と `bitcoin_key_2` に支払うことの証明
2. `node_1` が `bitcoin_key_1` を所有していることの証明
3. `node_2` が `bitcoin_key_2` を所有していることの証明

すべてのノードが未使用トランザクション出力を把握していると仮定すれば、最初の証明は `short_channel_id` で示される出力を見つけ、それが [BOLT #3](03-transactions.md#funding-transaction-output) で規定されたキーに対する P2WSH 資金調達トランザクション出力であることを確認することで達成されます。

残り 2 つの証明は明示的な署名によって達成されます。すなわち、各 `bitcoin_key` について `bitcoin_signature_1` と `bitcoin_signature_2` を生成し、それぞれ対応する `node_id` に署名します。

さらに、`node_1` と `node_2` がアナウンスメントメッセージの内容に合意していることも証明する必要があります。これは、各 `node_id` による署名 (`node_signature_1` と `node_signature_2`) をメッセージに付与することで達成されます。

1. type: 256 (`channel_announcement`)
2. data:
    * [`signature`:`node_signature_1`]
    * [`signature`:`node_signature_2`]
    * [`signature`:`bitcoin_signature_1`]
    * [`signature`:`bitcoin_signature_2`]
    * [`u16`:`len`]
    * [`len*byte`:`features`]
    * [`chain_hash`:`chain_hash`]
    * [`short_channel_id`:`short_channel_id`]
    * [`point`:`node_id_1`]
    * [`point`:`node_id_2`]
    * [`point`:`bitcoin_key_1`]
    * [`point`:`bitcoin_key_2`]

### Requirements

オリジンノードは:
  - `chain_hash` を、チャネルが開設されたチェーンを一意に識別する 32 バイトのハッシュに設定しなければならない:
    - _Bitcoin ブロックチェーン_ の場合:
      - `chain_hash` の値 (16 進数でエンコードしたもの) を `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000` に設定しなければならない。
  - チャネル作成をアナウンスする場合:
    - `short_channel_id` を、[BOLT #2](02-peer-protocol.md#the-channel_ready-message) で指定された確認済みの資金調達トランザクションを参照するように設定しなければならない。
  - スプライストランザクションをアナウンスする場合:
    - `short_channel_id` を、[BOLT #2](02-peer-protocol.md#the-splice_locked-message) で指定されたとおり、`splice_locked` を送受信した確認済みのスプライストランザクションを参照するように設定しなければならない。
    - 直前の `channel_announcement` の `short_channel_id` を用いた支払いの中継も、引き続き行うべきである。
    - 最新の `channel_announcement` に一致する `short_channel_id` を用いた新しい `channel_update` を送信すべきである。
  - 注: 対応する出力は、[BOLT #3](03-transactions.md#funding-transaction-output) に記載のとおり P2WSH でなければならない。
  - `node_id_1` と `node_id_2` を、チャネルを運営する 2 つのノードの公開鍵に設定しなければならない。このとき、2 つの圧縮鍵を昇順 (辞書順) に並べたうえで、辞書順で小さい方を `node_id_1` としなければならない。
  - `bitcoin_key_1` と `bitcoin_key_2` を、それぞれ `node_id_1`、`node_id_2` の `funding_pubkey` に設定しなければならない。
  - メッセージのオフセット 256 からメッセージ末尾までのダブル SHA256 ハッシュ `h` を計算しなければならない。
    - 注: このハッシュは 4 つの署名をスキップするが、メッセージの残りの部分をハッシュ対象とし、将来末尾に追加されるフィールドも含む。
  - `node_signature_1` と `node_signature_2` を、ハッシュ `h` に対する有効な署名 (それぞれ `node_id_1`、`node_id_2` に対応する秘密鍵で生成) に設定しなければならない。
  - `bitcoin_signature_1` と `bitcoin_signature_2` を、ハッシュ `h` に対する有効な署名 (それぞれ `bitcoin_key_1`、`bitcoin_key_2` に対応する秘密鍵で生成) に設定しなければならない。
  - `features` を、このチャネルに対して交渉された機能に応じて、[BOLT #9](09-features.md#assigned-features-flags) に従って設定しなければならない。
  - `len` を、設定する `features` ビットを保持するために必要な最小長に設定しなければならない。
  - 資金調達トランザクションの確認数が 6 未満の場合:
    - `channel_announcement` を送信してはならない。

受信ノードは:
  - 署名を検証することで、メッセージの完全性と真正性を確認しなければならない。
  - `features` フィールドに未知の偶数ビットがある場合:
    - そのチャネルを通じてメッセージをルーティングしようとしてはならない。
  - `short_channel_id` の出力が、`bitcoin_key_1` と `bitcoin_key_2` を用いて [BOLT #3](03-transactions.md#funding-transaction-output) に指定された P2WSH に対応しない場合、または出力が消費済みである場合:
    - メッセージを無視しなければならない。
  - 指定された `chain_hash` が受信者にとって未知の場合:
    - メッセージを無視しなければならない。
  - `short_channel_id` の出力が確認数 6 を満たしていない場合:
    - 受信ノードがまだ最新ブロックを受け取っていない可能性を考慮し、確認数が 6 に近ければメッセージを受け入れてよい。
    - そうでなければ:
      - メッセージを無視すべきである。
  - それ以外の場合:
    - `bitcoin_signature_1`、`bitcoin_signature_2`、`node_signature_1`、`node_signature_2` のいずれかが無効または正しくない場合:
      - `warning` を送信すべきである。
      - 接続を閉じてよい。
      - メッセージを無視しなければならない。
    - それ以外の場合:
      - `node_id_1` または `node_id_2` がブラックリストに載っている場合:
        - メッセージを無視すべきである。
      - それ以外の場合:
        - 参照されたトランザクションが以前にチャネルとしてアナウンスされていない場合:
          - メッセージを再放送用のキューに入れるべきである。
          - 期待される最小長より長いメッセージについては、キューに入れないことを選んでよい。
      - 同一トランザクションかつ同一ブロックで、異なる `node_id_1` または `node_id_2` を持つ有効な `channel_announcement` を以前に受信している場合:
        - 前のメッセージの `node_id_1` と `node_id_2`、および今回の `node_id_1` と `node_id_2` をブラックリストに登録し、これらに紐付くチャネルを忘れるべきである。
      - それ以外の場合:
        - この `channel_announcement` を保存すべきである。
  - 資金調達出力が消費された、または再編成された場合:
    - 72 ブロックの遅延を経てチャネルを忘れるべきである。
    - この `channel_announcement` をピアに再放送すべきでない。

### Rationale

両方のノードが署名を要求されるのは、このチャネルを通じて他の支払いを中継する意思 (すなわちパブリックネットワークの一部であること) を示すためです。Bitcoin 署名を要求することで、両者がチャネルを実際に制御していることが証明されます。

競合するノードをブラックリストに載せることで、複数の異なるアナウンスを排除できます。このような競合するアナウンスは鍵の漏洩を意味するため、どのノードもブロードキャストすべきではありません。

チャネルは十分な確認を得るまでアドバタイズすべきではありません。ただし再放送に関する要件は、トランザクションが別のブロックに移動していない場合にのみ適用されます。

過度に大きなメッセージを保存することを避けつつ、将来の妥当な拡張を許容するため、ノードは (たとえば統計的に) 再放送を制限してよいことになっています。

将来的に新しいチャネル機能が追加される可能性があります。後方互換性のある (またはオプションの) 機能は _奇数_ の機能ビットを持ち、互換性のない機能は _偶数_ の機能ビットを持ちます (["It's OK to be odd!"](00-introduction.md#glossary-and-terminology-guide))。

資金調達出力が消費されたあとにチャネルを忘れるまで 72 ブロックの遅延を設けるのは、このチャネルがクローズではなくスプライスされたことを示す新たな `channel_announcement` が伝搬する余地を確保するためです。この遅延のおかげで、スプライストランザクションが十分な確認を得るまでの間も、当該チャネルでの支払い中継を継続できます。

## `node_announcement` メッセージ

このゴシップメッセージは、ノードが公開鍵に加えて自身に関する追加データを示すことを可能にします。単純なサービス拒否攻撃を避けるため、既知のチャネルに関連付けられていないノードは無視されます。

1. type: 257 (`node_announcement`)
2. data:
   * [`signature`:`signature`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]
   * [`u32`:`timestamp`]
   * [`point`:`node_id`]
   * [`3*byte`:`rgb_color`]
   * [`32*byte`:`alias`]
   * [`u16`:`addrlen`]
   * [`addrlen*byte`:`addresses`]

`timestamp` は、複数のアナウンスメントが存在する場合にメッセージの順序付けに用いられます。`rgb_color` と `alias` を用いると、情報機関がノードに黒のような色や 'IRATEMONK'・'WISTFULTOLL' のようなクールなニックネームを割り当てることもできます。

`addresses` は、ノードがネットワーク接続を受け入れる意思を表明するためのフィールドです。ノードへ接続するための一連の `address descriptor` が含まれます。最初のバイトはアドレスタイプを示し、そのタイプに応じたバイト列が続きます。

定義されている `address descriptor` タイプは以下のとおりです。

   * `1`: ipv4; data = `[4:ipv4_addr][2:port]` (長さ 6)
   * `2`: ipv6; data = `[16:ipv6_addr][2:port]` (長さ 18)
   * `3`: 廃止 (長さ 12)。以前は Tor v2 オニオンサービスのために使用されていました。
   * `4`: Tor v3 オニオンサービス; data = `[35:onion_addr][2:port]` (長さ 37)
       * バージョン 3 ([prop224](https://gitweb.torproject.org/torspec.git/tree/proposals/224-rend-spec-ng.txt)) のオニオンサービスアドレス。エンコード:
         `[32:32_byte_ed25519_pubkey] || [2:checksum] || [1:version]` で、
         `checksum = sha3(".onion checksum" || pubkey || version)[:2]`。
   * `5`: DNS ホスト名; data = `[1:hostname_len][hostname_len:hostname][2:port]` (長さは最大 258)
       * `hostname` のバイト列は ASCII 文字でなければなりません。
       * 非 ASCII 文字は Punycode を用いてエンコードしなければなりません:
         https://en.wikipedia.org/wiki/Punycode

### Requirements

オリジンノードは:
  - `timestamp` を、それ以前に自身が作成した `node_announcement` のいずれよりも大きい値に設定しなければならない。
    - UNIX タイムスタンプを基準にしてもよい。
  - `signature` を、`signature` 以降のパケット全体のダブル SHA256 に対する署名 (`node_id` で示される鍵を用いて生成) に設定しなければならない。
  - 地図やグラフ上での外観をカスタマイズするために、`alias` および `rgb_color` を設定してもよい。
    - 注: `rgb_color` の 1 バイト目が赤、2 バイト目が緑、3 バイト目が青の値である。
  - `alias` を有効な UTF-8 文字列に設定しなければならず、`alias` の末尾の余ったバイトは 0 にしなければならない。
  - 着信接続を受け付ける各パブリックネットワークアドレスについて、`addresses` をアドレス記述子で埋めるべきである。
  - `addrlen` を `addresses` のバイト数に設定しなければならない。
  - アドレス記述子は昇順に並べなければならない。
  - 型がゼロのアドレス記述子をどこにも配置すべきでない。
  - パディングは、`addresses` の後に続くフィールドの整列のためにのみ使用すべきである。
  - `port` が 0 である `type 1`、`type 2`、`type 5` のアドレス記述子を生成してはならない。
  - `ipv4_addr` および `ipv6_addr` はルーティング可能なアドレスであることを保証すべきである。
  - `features` を [BOLT #9](09-features.md#assigned-features-flags) に従って設定しなければならない。
  - `flen` を、設定する `features` ビットを保持するために必要な最小長に設定すべきである。
  - Tor v2 オニオンサービスはアナウンスすべきでない。
  - `type 5` の DNS ホスト名を 2 つ以上アナウンスしてはならない。

受信ノードは:
  - `node_id` が有効な圧縮公開鍵でない場合:
    - `warning` を送信すべきである。
    - 接続を閉じてよい。
    - メッセージをこれ以上処理してはならない。
  - `signature` が、`signature` フィールド以降のメッセージ全体のダブル SHA256 に対する有効な署名 (`node_id` で検証) でない場合:
    - `warning` を送信すべきである。
    - 接続を閉じてよい。
    - メッセージをこれ以上処理してはならない。
  - `features` フィールドに未知の偶数ビットが含まれている場合:
    - そのノードに接続すべきでない。
    - 同じビットが設定されていない [BOLT #11](11-payment-encoding.md) インボイスを支払う場合を除き、そのノード _宛て_ の支払いを送信しようとしてはならない。
    - そのノードを _経由して_ 支払いをルーティングしてはならない。
  - 上記で定義されたタイプに一致しない最初の `address descriptor` 以降は無視すべきである。
  - `addrlen` が既知タイプのアドレス記述子を保持するのに不足している場合:
    - `warning` を送信すべきである。
    - 接続を閉じてよい。
  - `port` が 0 の場合:
    - `ipv6_addr`、`ipv4_addr`、または `hostname` を無視すべきである。
  - `node_id` が `channel_announcement` メッセージにより既知でない場合、または `timestamp` がこの `node_id` から最後に受信した `node_announcement` を上回らない場合:
    - メッセージを無視すべきである。
  - それ以外の場合:
    - `timestamp` がこの `node_id` から最後に受信した `node_announcement` より大きい場合:
      - メッセージを再放送用のキューに入れるべきである。
      - 期待される最小長より長いメッセージはキューに入れないことを選んでよい。
  - インタフェース上でノードを参照する目的で `rgb_color` および `alias` を使用してよい。
    - それらが自己署名された情報であることを示唆すべきである。
  - Tor v2 オニオンサービスは無視すべきである。
  - `type 5` のアドレスが 2 つ以上アナウンスされている場合:
    - 余分なデータは無視すべきである。
    - その `node_announcement` を転送してはならない。

### Rationale

将来的に新しいノード機能が追加される可能性があります。後方互換性のある (またはオプションの) ものは _奇数_ の `feature` _ビット_ を持ち、互換性のないものは _偶数_ の `feature` _ビット_ を持ちます。これらは通常通り伝搬されます。ここでの互換性のない機能ビットは、ノード自体に関するものであって `node_announcement` メッセージそのものに関するものではない点に注意してください。

将来的に新しいアドレスタイプが追加される可能性があります。アドレス記述子は昇順に並べる必要があるため、未知のタイプは安全に無視できます。また `addresses` の後に追加のフィールドが将来加わる可能性があり、特定のアライメントが必要な場合には `addresses` 内にオプションのパディングを含めることができます。

### ノードエイリアスのセキュリティに関する考慮事項

ノードエイリアスはユーザが自由に定義できるため、レンダリングおよび永続化の過程でインジェクション攻撃の入口となりうる点に注意が必要です。

ノードエイリアスは、HTML/JavaScript コンテキストや動的に解釈されるその他のレンダリングフレームワークで表示する前に必ずサニタイズすべきです。同様に、SQL などの動的に解釈されるクエリ言語をサポートする永続化エンジンに対するインジェクションの脆弱性から保護するため、プリペアドステートメント、入力検証、エスケープの使用を検討してください。

* [保存型および反射型 XSS 防止](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)
* [DOM ベースの XSS 防止](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL インジェクション防止](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

[Little Bobby Tables](https://xkcd.com/327/) の学校のようにならないようにしましょう。

## `channel_update` メッセージ

チャネルが最初にアナウンスされた後、各側は独立して、このチャネルを通じて HTLC を中継するために必要な手数料と最小の有効期限デルタをアナウンスします。各側は `channel_announcement` と一致する 8 バイトの channel shortid と、自身がチャネルのどちら側 (起点側か終点側か) かを示す 1 ビットの `channel_flags` を使用します。ノードは手数料を変更するために、これを複数回行うことができます。

`channel_update` ゴシップメッセージは、支払いを *送信する* 場面ではなく、支払いを *中継する* 場面でのみ意味を持ちます。`A` -> `B` -> `C` -> `D` と支払いを行う場合、関係するのは `B` -> `C` (`B` がアナウンス) と `C` -> `D` (`C` がアナウンス) の `channel_update` のみです。ルートを構築する際には、HTLC の金額と有効期限を宛先から送信元へ向かって逆向きに計算する必要があります。ルートの最後の HTLC に用いる `amount_msat` の正確な初期値と `cltv_expiry` の最小値は支払い要求で提供されます ([BOLT #11](11-payment-encoding.md#tagged-fields) を参照)。

1. type: 258 (`channel_update`)
2. data:
    * [`signature`:`signature`]
    * [`chain_hash`:`chain_hash`]
    * [`short_channel_id`:`short_channel_id`]
    * [`u32`:`timestamp`]
    * [`byte`:`message_flags`]
    * [`byte`:`channel_flags`]
    * [`u16`:`cltv_expiry_delta`]
    * [`u64`:`htlc_minimum_msat`]
    * [`u32`:`fee_base_msat`]
    * [`u32`:`fee_proportional_millionths`]
    * [`u64`:`htlc_maximum_msat`]

`channel_flags` ビットフィールドは、チャネルの方向を示すとともに、この更新がどちらのノード起点であるかを識別し、チャネルに関する各種オプションを伝達します。各ビットの意味は次の表のとおりです。

| ビット位置  | 名前        | 意味                          |
| ----------- | ----------- | ----------------------------- |
| 0           | `direction` | この更新が指す方向。          |
| 1           | `disable`   | チャネルを無効にする。        |

`message_flags` ビットフィールドは、メッセージに関する追加情報を伝えます。

| ビット位置  | 名前           |
| ----------- | ---------------|
| 0           | `must_be_one`  |
| 1           | `dont_forward` |

署名検証に用いる `node_id` は対応する `channel_announcement` から取得します。`channel_flags` の最下位ビットが 0 の場合は `node_id_1`、それ以外の場合は `node_id_2` を使用します。

### Requirements

オリジンノードは:
  - `channel_ready` を受信する前に作成した `channel_update` を送信してはならない。
  - チャネルがまだアナウンスされていない場合 (すなわち `announce_channel` ビットが設定されていない場合や、ピア間で [announcement signatures](#the-announcement_signatures-message) が交換される前に `channel_update` が送信される場合) でも、チャネルパラメータをチャネルピアに伝えるために `channel_update` を作成してよい。
    - `short_channel_id` を、ピアから受け取った `alias`、または実際のチャネル `short_channel_id` のいずれかに設定しなければならない。
    - `message_flags` の `dont_forward` を 1 に設定しなければならない。
    - プライバシー保護のため、このような `channel_update` を他のピアに転送してはならない。
    - 注: `channel_announcement` を伴わない `channel_update` は他のピアにとって無効であり破棄される。
  - `signature` を、`signature` 以降のパケット全体のダブル SHA256 に対する、自身の `node_id` で生成した署名に設定しなければならない。
  - `chain_hash` および `short_channel_id` を、`channel_announcement` メッセージで指定されたチャネルを一意に識別する 32 バイトのハッシュおよび 8 バイトのチャネル ID と一致させなければならない。
  - メッセージ内でオリジンノードが `node_id_1` である場合:
    - `channel_flags` の `direction` ビットを 0 に設定しなければならない。
  - それ以外の場合:
    - `channel_flags` の `direction` ビットを 1 に設定しなければならない。
  - `htlc_maximum_msat` を、このチャネルを通じて単一の HTLC として送信する最大値に設定しなければならない。
    - チャネル容量以下に設定しなければならない。
    - ピアから受け取った `max_htlc_value_in_flight_msat` 以下に設定しなければならない。
    - `htlc_minimum_msat` 以上に設定しなければならない。
  - `message_flags` の `must_be_one` を 1 に設定しなければならない。
  - `channel_flags` および `message_flags` のうち、意味が割り当てられていないビットを 0 に設定しなければならない。
  - チャネルの一時的な利用不可 (例: 接続喪失) や恒久的な利用不可 (例: オンチェーン決済前) を示すために、`disable` ビットを 1 に設定した `channel_update` を作成・送信してよい。
    - チャネルを再有効化するために、`disable` ビットを 0 に設定した後続の `channel_update` を送信してよい。
  - `timestamp` を 0 より大きく、かつこの `short_channel_id` に対して以前送信した `channel_update` より大きい値に設定しなければならない。
    - `timestamp` は UNIX タイムスタンプを基準にすべきである。
  - `cltv_expiry_delta` を、受信 HTLC の `cltv_expiry` から差し引くブロック数に設定しなければならない。
  - `htlc_minimum_msat` を、チャネルピアが受け入れる最小 HTLC 値 (ミリサトシ単位) に設定しなければならない。
    - `htlc_minimum_msat` を `htlc_maximum_msat` 以下に設定しなければならない。
  - `fee_base_msat` を、任意の HTLC に対して請求する基本手数料 (ミリサトシ単位) に設定しなければならない。
  - `fee_proportional_millionths` を、中継するサトシあたりに請求する額 (サトシの百万分の一単位) に設定しなければならない。
  - 冗長な `channel_update` を作成すべきでない。
  - 更新されたチャネルパラメータで新たな `channel_update` を作成する場合:
    - 以前のチャネルパラメータを 10 分間は受け入れ続けるべきである。

受信ノードは:
  - `short_channel_id` が以前の `channel_announcement` と一致しない場合:
    - 自身のチャネルに対応しない `channel_update` は無視しなければならない。
  - チャネルの出力が消費済みの場合:
    - `disable` ビットが 1 に設定された `channel_update` を除き、`channel_update` を無視しなければならない。
    - `disable` ビットが 1 に設定された `channel_update` を除き、ピアに対して `channel_update` を再放送すべきでない。
  - 自身のチャネルに対する `channel_update` は、非公開のものであっても、関連するオリジンノードの転送パラメータを把握するために受け入れるべきである。
  - `signature` が、`signature` フィールド以降のメッセージ全体のダブル SHA256 に対する `node_id` での有効な署名でない場合 (`fee_proportional_millionths` の後に未知のフィールドが続く場合も含む):
    - `warning` を送信して接続を閉じるべきである。
    - メッセージをこれ以上処理してはならない。
  - 指定された `chain_hash` 値が未知である (指定チェーン上でアクティブでないことを意味する) 場合:
    - その `channel_update` を無視しなければならない。
  - この `short_channel_id` および `node_id` について最後に受信した `channel_update` と `timestamp` が等しい場合:
    - `timestamp` 以下のフィールドが異なるとき:
      - この `node_id` をブラックリストに載せてよい。
      - それに関連付けられるすべてのチャネルを忘れてよい。
    - `timestamp` 以下のフィールドが等しいとき:
      - このメッセージを無視すべきである。
  - この `short_channel_id` および `node_id` について最後に受信した `channel_update` より `timestamp` が小さい場合:
    - メッセージを無視すべきである。
  - それ以外の場合:
    - `timestamp` が将来に向けて不合理に遠い場合:
      - `channel_update` を破棄してよい。
    - それ以外の場合:
      - メッセージを再放送用のキューに入れるべきである。
      - 期待される最小長より長いメッセージはキューに入れないことを選んでよい。
  - `htlc_maximum_msat` < `htlc_minimum_msat` の場合:
    - 経路選択時にこのチャネルを無視すべきである。
  - `htlc_maximum_msat` がチャネル容量を超える場合:
    - この `node_id` をブラックリストに載せてよい。
    - 経路選択時にこのチャネルを無視すべきである。
  - それ以外の場合:
    - 経路選択時に `htlc_maximum_msat` を考慮すべきである。

### Rationale

`timestamp` フィールドは、将来に向けて不合理に遠い、または 2 週間更新されていない `channel_update` を剪定するために用いられるため、UNIX タイムスタンプ (UTC 1970-01-01 からの秒数) として扱うのが合理的です。ただし、1 秒以内に 2 つの `channel_update` が発生し得るため、これを厳密な要件にすることはできません。

同一秒内にチャネルパラメータを変更する `channel_update` メッセージが複数発生する場合は DoS 攻撃の試みとみなしうるため、そのようなメッセージに署名したノードはブラックリストに載る可能性があります。一方で、ノードは署名以外を変えずに署名のみを変更したメッセージを送信することもありうるため (署名生成時にノンスが変わるため)、同一タイムスタンプでチャネルパラメータが実際に変更されているかは署名以外のフィールドで確認します。また、ECDSA 署名はマリアブルであるため、`channel_update` を受け取った中間ノードが署名の `s` 成分を `-s` に変えるだけで再放送することも可能ですが、これだけを理由にメッセージ送信元の `node_id` をブラックリスト化すべきではありません。

冗長な `channel_update` を避けるよう推奨することでネットワークのスパムを最小化できますが、避けられない場合もあります。たとえば、到達不能なピアとのチャネルでは、最終的にチャネルを無効化する `channel_update` が発生し、ピアが再接続した際に再有効化する別の更新が続くことになります。ゴシップメッセージはバッチ処理され、以前のものを置き換えるため、結果として一見冗長な単一の更新が残ることがあります。

ノードがチャネルパラメータを変更する新しい `channel_update` を作成すると、それがネットワーク全体に伝搬するまでには時間を要し、支払者が古いパラメータを使う可能性があります。支払いのレイテンシと信頼性を高めるため、古いパラメータも少なくとも 10 分間は受け入れ続けることが推奨されます。

`message_flags` の `must_be_one` フィールドは、以前は `htlc_maximum_msat` フィールドの存在を示すために用いられていました。同フィールドは現在常に存在しなければならないため、`must_be_one` は定数値となり、受信者には無視されます。

## Query Messages

これらのメッセージへの対応は、かつては `gossip_queries` 機能ビットで示されていましたが、現在は普遍的にサポートされており、この機能ビットは少し意味が変わっています。この機能を提供しないことは、そのノードがゴシップの問い合わせに値しない (すなわちゴシップマップ全体を保持していないか、ただ 1 つのピア (このノード自身) にのみ接続している) ことを示します。

`short_channel_id` の長い配列 (`encoded_short_ids` と呼ばれる) を含むメッセージがいくつかあります。将来的に有用な別のエンコーディング方式を導入できるよう、エンコーディングバイトをメッセージ内に含めるようにしています。

エンコーディングタイプ:
* `0`: 昇順に並んだ非圧縮の `short_channel_id` 配列。
* `1`: 以前は zlib 圧縮に用いられていたが、このエンコーディングは使用してはならない。

このエンコーディングは他の型 (タイムスタンプ、フラグなど) の配列にも使用され、`encoded_` プレフィックス付きで参照されます。例えば `encoded_timestamps` は、先頭バイトが `0` のタイムスタンプ配列です。

クエリメッセージはオプションフィールドで拡張でき、これによりルーティングテーブル同期に必要なメッセージ数を削減できます。具体的には次のことが可能になります。

- `channel_update` メッセージのタイムスタンプベースのフィルタリング: 自身が保持しているものより新しい `channel_update` だけを要求する。
- `channel_update` メッセージのチェックサムベースのフィルタリング: 自身が保持しているものと内容が異なる `channel_update` だけを要求する。

ノードは `gossip_queries_ex` 機能ビットによって、拡張ゴシップクエリのサポートを示すことができます。

### `query_short_channel_ids` および `reply_short_channel_ids_end` メッセージ

1. type: 261 (`query_short_channel_ids`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u16`:`len`]
    * [`len*byte`:`encoded_short_ids`]
    * [`query_short_channel_ids_tlvs`:`tlvs`]

1. `tlv_stream`: `query_short_channel_ids_tlvs`
2. types:
    1. type: 1 (`query_flags`)
    2. data:
        * [`byte`:`encoding_type`]
        * [`...*byte`:`encoded_query_flags`]

`encoded_query_flags` はビットフィールドの配列で、各ビットフィールドは 1 つの bigsize で表され、`short_channel_id` ごとに 1 つのビットフィールドが対応します。ビットの意味は以下のとおりです。

| ビット位置   | 意味                                      |
| ------------- | ---------------------------------------- |
| 0             | 送信者は `channel_announcement` を希望   |
| 1             | 送信者はノード 1 の `channel_update` を希望 |
| 2             | 送信者はノード 2 の `channel_update` を希望 |
| 3             | 送信者はノード 1 の `node_announcement` を希望 |
| 4             | 送信者はノード 2 の `node_announcement` を希望 |

クエリフラグは最小エンコードでなければならず、1 つのフラグは 1 バイトにエンコードされます。

1. type: 262 (`reply_short_channel_ids_end`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`byte`:`full_information`]

これは、`short_channel_id` で識別される特定のチャネルに対する `channel_announcement` および `channel_update` メッセージをノードがクエリするための汎用的な仕組みです。通常は、対応する `channel_announcement` を持たない `channel_update` を見つけた場合や、`reply_channel_range` から未知の `short_channel_id` を取得した場合に使用されます。

#### Requirements

送信者は:
  - `gossip_queries` を提供しないピアに対してこれを送信すべきでない。
  - 以前にこのピアへ送った `query_short_channel_ids` に対する `reply_short_channel_ids_end` をまだ受け取っていない場合、`query_short_channel_ids` を送信してはならない。
  - `chain_hash` を、`short_channel_id` が参照するチェーンを一意に識別する 32 バイトのハッシュに設定しなければならない。
  - `encoded_short_ids` の先頭バイトをエンコーディングタイプに設定しなければならない。
  - `encoded_short_ids` には整数個の `short_channel_id` をエンコードしなければならない。
  - 対応する `channel_announcement` を持たない `short_channel_id` の `channel_update` を受け取った場合に、本メッセージを送信してよい。
  - 参照されるチャネルが未使用出力でない場合、本メッセージを送信すべきでない。
  - オプションの `query_flags` を含めてよい。その場合:
    - `encoded_short_ids` と同様に `encoding_type` を設定しなければならない。
    - 各クエリフラグは最小エンコードの bigsize でなければならない。
    - `short_channel_id` ごとに 1 つのクエリフラグをエンコードしなければならない。

受信者は:
  - `encoded_short_ids` の先頭バイトが既知のエンコーディングタイプでない場合:
    - `warning` を送信してよい。
    - 接続を閉じてよい。
  - `encoded_short_ids` が整数個の `short_channel_id` にデコードできない場合:
    - `warning` を送信してよい。
    - 接続を閉じてよい。
  - この送信者から以前に受け取った `query_short_channel_ids` に対して `reply_short_channel_ids_end` をまだ送信していない場合:
    - `warning` を送信してよい。
    - 接続を閉じてよい。
  - 受信メッセージに `query_short_channel_ids_tlvs` が含まれる場合:
    - `encoding_type` が既知のエンコーディングタイプでない場合:
      - `warning` を送信してよい。
      - 接続を閉じてよい。
    - `encoded_query_flags` が `short_channel_id` ごとにちょうど 1 つのフラグにデコードできない場合:
      - `warning` を送信してよい。
      - 接続を閉じてよい。
  - 既知の各 `short_channel_id` に対して応答しなければならない:
    - 受信メッセージに `encoded_query_flags` が含まれていない場合:
      - 各方向の `channel_announcement` と最新の `channel_update` で応答しなければならない。
      - 各 `channel_announcement` に続けて、対応する `node_announcement` も送らなければならない。
    - それ以外の場合:
      - `encoded_short_ids` の N 番目の `short_channel_id` に対する `query_flag` を、デコードされた `encoded_query_flags` の N 番目の bigsize と定義する。
      - `query_flag` のビット 0 が設定されている場合:
        - `channel_announcement` で応答しなければならない。
      - `query_flag` のビット 1 が設定されており、`node_id_1` から `channel_update` を受信済みの場合:
        - `node_id_1` の最新の `channel_update` で応答しなければならない。
      - `query_flag` のビット 2 が設定されており、`node_id_2` から `channel_update` を受信済みの場合:
        - `node_id_2` の最新の `channel_update` で応答しなければならない。
      - `query_flag` のビット 3 が設定されており、`node_id_1` から `node_announcement` を受信済みの場合:
        - `node_id_1` の最新の `node_announcement` で応答しなければならない。
      - `query_flag` のビット 4 が設定されており、`node_id_2` から `node_announcement` を受信済みの場合:
        - `node_id_2` の最新の `node_announcement` で応答しなければならない。
    - これらの送信のために次の送信ゴシップフラッシュを待つべきでない。
  - 単一の `query_short_channel_ids` に対する応答で同一の `node_announcement` を重複して送信することを避けるべきである。
  - これらの応答の最後に `reply_short_channel_ids_end` を送信しなければならない。
  - `chain_hash` に対する最新のチャネル情報を維持していない場合:
    - `full_information` を 0 に設定しなければならない。
  - それ以外の場合:
    - `full_information` を 1 に設定すべきである。

#### Rationale

将来のノードは完全な情報を保持していない可能性があり、特に未知の `chain_hash` のチェーンについては顕著です。この `full_information` フィールド (以前は紛らわしく `complete` と呼ばれていました) を完全に信頼することはできませんが、値が 0 であれば送信者は他の場所で追加データを探すべき、という指針が得られます。

`reply_short_channel_ids_end` メッセージを明示することで、受信者は何も知らない旨を伝えることができ、送信者はタイムアウトに依存する必要がなくなります。さらに、これによりクエリの自然なレートリミットが生じます。

### `query_channel_range` および `reply_channel_range` メッセージ

1. type: 263 (`query_channel_range`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_blocknum`]
    * [`u32`:`number_of_blocks`]
    * [`query_channel_range_tlvs`:`tlvs`]

1. `tlv_stream`: `query_channel_range_tlvs`
2. types:
    1. type: 1 (`query_option`)
    2. data:
        * [`bigsize`:`query_option_flags`]

`query_option_flags` は、最小エンコードの bigsize で表現されるビットフィールドです。各ビットの意味は以下のとおりです。

| ビット位置   | 意味                     |
| ------------- | ----------------------- |
| 0             | 送信者はタイムスタンプを希望 |
| 1             | 送信者はチェックサムを希望 |

チェックサムのみを要求することは可能ですが、タイムスタンプも併せて要求しなければあまり有用ではありません。受信側ノードが異なるチェックサムを持つ古い `channel_update` を保有している可能性があり、それを要求しても意味がないからです。また、`channel_update` のチェックサムが偶然 0 になる場合 (極めて稀ですが) には、そのチャネルはクエリされません。

1. type: 264 (`reply_channel_range`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_blocknum`]
    * [`u32`:`number_of_blocks`]
    * [`byte`:`sync_complete`]
    * [`u16`:`len`]
    * [`len*byte`:`encoded_short_ids`]
    * [`reply_channel_range_tlvs`:`tlvs`]

1. `tlv_stream`: `reply_channel_range_tlvs`
2. types:
    1. type: 1 (`timestamps_tlv`)
    2. data:
        * [`byte`:`encoding_type`]
        * [`...*byte`:`encoded_timestamps`]
    1. type: 3 (`checksums_tlv`)
    2. data:
        * [`...*channel_update_checksums`:`checksums`]

単一の `channel_update` に対するタイムスタンプは、次の形式でエンコードされます。

1. subtype: `channel_update_timestamps`
2. data:
    * [`u32`:`timestamp_node_id_1`]
    * [`u32`:`timestamp_node_id_2`]

ここで:
* `timestamp_node_id_1` は `node_id_1` の `channel_update` のタイムスタンプ。当該ノードからの `channel_update` がない場合は 0。
* `timestamp_node_id_2` は `node_id_2` の `channel_update` のタイムスタンプ。当該ノードからの `channel_update` がない場合は 0。

単一の `channel_update` に対するチェックサムは、次の形式でエンコードされます。

1. subtype: `channel_update_checksums`
2. data:
    * [`u32`:`checksum_node_id_1`]
    * [`u32`:`checksum_node_id_2`]

ここで:
* `checksum_node_id_1` は `node_id_1` の `channel_update` のチェックサム。当該ノードからの `channel_update` がない場合は 0。
* `checksum_node_id_2` は `node_id_2` の `channel_update` のチェックサム。当該ノードからの `channel_update` がない場合は 0。

`channel_update` のチェックサムは、`signature` と `timestamp` フィールドを除いた当該 `channel_update` に対する CRC32C チェックサムです。詳細は [RFC3720](https://tools.ietf.org/html/rfc3720#appendix-B.4) を参照してください。

これにより、特定のブロック範囲内のチャネルをクエリできます。

#### Requirements

`query_channel_range` の送信者は:
  - `gossip_queries` を提供しないピアに対してこれを送信すべきでない。
  - 以前にこのピアへ送った `query_channel_range` に対するすべての `reply_channel_range` 応答をまだ受け取っていない場合、本メッセージを送信してはならない。
  - `chain_hash` を、`reply_channel_range` が参照するチェーンを一意に識別する 32 バイトのハッシュに設定しなければならない。
  - `first_blocknum` を、チャネル情報を取得したい最初のブロックに設定しなければならない。
  - `number_of_blocks` を 1 以上に設定しなければならない。
  - 受け取りたい拡張情報の種類を指定する追加の `query_channel_range_tlv` を付加してよい。

`query_channel_range` の受信者は:
  - この送信者から以前受信した `query_channel_range` に対するすべての `reply_channel_range` をまだ送信していない場合:
    - `warning` を送信してよい。
    - 接続を閉じてよい。
  - 1 つ以上の `reply_channel_range` で応答しなければならない:
    - `chain_hash` を `query_channel_range` と同じ値に設定しなければならない。
    - `number_of_blocks` を、結果が `encoded_short_ids` に収まる最大のブロック数に制限しなければならない。
    - ブロックの内容を複数の `reply_channel_range` に分割してよい。
    - 最初の `reply_channel_range` メッセージ:
      - `first_blocknum` を `query_channel_range` の `first_blocknum` 以下に設定しなければならない。
      - `first_blocknum` に `number_of_blocks` を加えた値を `query_channel_range` の `first_blocknum` より大きくしなければならない。
    - 後続の `reply_channel_range` メッセージ:
      - `first_blocknum` を直前の `first_blocknum` 以上にしなければならない。
    - 最終の `reply_channel_range` でない場合、`sync_complete` を `false` に設定しなければならない。
    - 最終の `reply_channel_range` メッセージ:
      - `first_blocknum` に `number_of_blocks` を加えた値を、`query_channel_range` の `first_blocknum` に `number_of_blocks` を加えた値以上にしなければならない。
    - `sync_complete` を `true` に設定しなければならない。

受信メッセージに `query_option` が含まれる場合、受信者は応答に追加情報を付加してよい。

- `query_option_flags` のビット 0 が設定されている場合、受信者は `encoded_short_ids` 内の全 `short_channel_id` の `channel_update` タイムスタンプを含む `timestamps_tlv` を付加してよい。
- `query_option_flags` のビット 1 が設定されている場合、受信者は `encoded_short_ids` 内の全 `short_channel_id` の `channel_update` チェックサムを含む `checksums_tlv` を付加してよい。

#### Rationale

単一の応答が 1 パケットに収まらない可能性があるため、複数の応答が必要になる場合があります。ピアが (例えば) 1000 ブロック範囲分の結果を事前計算しておけるよう、応答は要求範囲を超えてもよい設計にしています。ただし、各応答は要求範囲と重なる関連性のあるものでなければなりません。

応答を昇順に並べる要件により、受信者は応答が完了したかどうかを簡単に判定できます。すなわち、`first_blocknum + number_of_blocks` が要求した `first_blocknum + number_of_blocks` 以上であるかを確認するだけです。

タイムスタンプおよびチェックサムフィールドの追加により、ピアは冗長な更新のクエリを省略できます。

### `gossip_timestamp_filter` メッセージ

1. type: 265 (`gossip_timestamp_filter`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_timestamp`]
    * [`u32`:`timestamp_range`]

このメッセージにより、ノードは今後受け取るゴシップメッセージを特定の範囲に制限できます。ゴシップメッセージを受信したいノードは、必ずこれを送信する必要があります。送信しなければ、ゴシップメッセージは届きません。

このフィルタは以前のものを置き換えるため、ピアから受け取るゴシップを変更するために複数回送信できます。

#### Requirements

送信者は:
  - `chain_hash` を、ゴシップが参照するチェーンを一意に識別する 32 バイトのハッシュに設定しなければならない。
  - 受信者が `gossip_queries` を提供しない場合:
    - `first_timestamp` を 0xFFFFFFFF に、`timestamp_range` を 0 に設定すべきである。

受信者は:
  - `timestamp` が `first_timestamp` 以上、かつ `first_timestamp + timestamp_range` 未満のすべてのゴシップメッセージを送信すべきである。
    - 送信のために次の送信ゴシップフラッシュを待ってよい。
  - 自身が生成したゴシップメッセージは、`timestamp` にかかわらず送信すべきである。
  - それ以外の場合 (中継されるゴシップ):
    - 今後のゴシップメッセージを、`timestamp` が `first_timestamp` 以上、かつ `first_timestamp + timestamp_range` 未満のものに制限すべきである。
  - `channel_announcement` に対応する `channel_update` がない場合:
    - その `channel_announcement` を送信してはならない。
  - `channel_announcement` の資金調達出力が消費されている場合:
    - その `channel_announcement` を送信すべきでない。
  - それ以外の場合:
    - 対応する `channel_update` の `timestamp` を `channel_announcement` の `timestamp` と見なさなければならない。
    - 対応する最初の `channel_update` を受信した後に、`channel_announcement` を送信するかどうかを判断しなければならない。
  - `channel_announcement` を送信する場合:
    - 対応する `channel_update` および `node_announcement` より前に `channel_announcement` を送信しなければならない。

#### Rationale

`channel_announcement` 自体にはタイムスタンプがないため、適切なタイムスタンプを擬似的に生成します。対応する `channel_update` が存在しない場合は送信しません。これは、剪定済みチャネルの場合に最も起こりやすい状況です。

通常、`channel_announcement` の直後に `channel_update` が続きます。理想的には最初 (最古) の `channel_update` のタイムスタンプを `channel_announcement` の時刻として用いると規定したいところですが、ネットワーク上の新しいノードはそれを保持しておらず、さらに最初の `channel_update` のタイムスタンプを保存する必要が生じます。代わりに任意の更新のタイムスタンプを使用してよいとしており、こちらの方が実装が簡単です。

それでも `channel_announcement` を取りこぼした場合は、`query_short_channel_ids` で取得できます。

ノードは多くのピアを持つ場合に、`timestamp_filter` を用いてゴシップ負荷を軽減できます (例: 伝搬は十分であると仮定し、最初の数ピア以降は `first_timestamp` を `0xFFFFFFFF` に設定する)。この「伝搬は十分である」という仮定は、ノード自身が直接生成したゴシップには当てはまらないため、それらに対してはフィルタを無視すべきです。

### Requirements

ノードは:
  - 明示的に要求されない限り、自身が生成していないゴシップメッセージを中継してはならない。

## 再放送

### Requirements

受信ノードは:
  - 新しい `channel_announcement`、新しい `channel_update`、または `timestamp` が更新された `node_announcement` を受信した場合:
    - ネットワークトポロジーに関するローカルビューを適切に更新すべきである。
  - アナウンスメントによる変更を適用した後:
    - 対応するオリジンノードに関連付けられたチャネルが存在しない場合:
      - 既知のノード集合からそのオリジンノードを除外してよい。
    - それ以外の場合:
      - 該当するメタデータを更新し、アナウンスメントに付随する署名を保存すべきである。
        - 注: これによりノードは後でピア向けにアナウンスを再構築できる。

ノードは:
  - `gossip_timestamp_filter` を受信するまで、自身が生成していないゴシップを送信してはならない。
  - メッセージの到着タイミングに関わらず、60 秒ごとに 1 回、送出待ちのゴシップメッセージをフラッシュすべきである。
    - 注: これによりアナウンスは時差をもって配信され、重複が避けられる。
    - `init` で `networks` を送信し、このゴシップメッセージの `chain_hash` を指定しなかったピアに対しては、ゴシップメッセージを転送すべきでない。
  - 自身のチャネルを定期的に再アナウンスしてよい。
    - 注: リソース要件を低く保つため、これは推奨されない。

### Rationale

ゴシップメッセージは処理された後、当該ノードのピア向けの送出メッセージリストに追加され、同じオリジンノードからの古い更新を置き換えます。このゴシップメッセージリストは定期的にフラッシュされます。このようにストアして遅延フォワードする方式のブロードキャストは _時差ブロードキャスト_ と呼ばれます。また、このようなバッチ処理は低オーバーヘッドな自然のレートリミットを形成します。

## HTLC 手数料

### Requirements

オリジンノードは:
  - 次の額以上の手数料を支払う HTLC を受け入れるべきである:
    - fee_base_msat + ( amount_to_forward * fee_proportional_millionths / 1000000 )
  - `channel_update` を送信した後も、一定の合理的な期間は古い手数料を支払う HTLC を受け入れるべきである。
    - 注: これは伝搬遅延を許容するためである。

## ネットワークビューの剪定

### Requirements

ノードは:
  - クローズされつつあるチャネルを把握するため、ブロックチェーン上の資金調達トランザクションを監視すべきである。
  - チャネルの資金調達出力が消費され、72 ブロックの確認を得た場合:
    - 当該チャネルをローカルネットワークビューから削除し、クローズしたとみなすべきである。
  - アナウンスされたノードに関連するオープンチャネルが存在しなくなった場合:
    - `node_announcement` メッセージで追加されたそのノードをローカルビューから剪定してよい。
      - 注: これは、`node_announcement` が `channel_announcement` に先行されるという依存関係の直接的な帰結である。

### 古いエントリの剪定に関する推奨事項

#### Requirements

ノードは:
  - 最新の `channel_update` の `timestamp` がどちらの方向についても 2 週間 (1209600 秒) より古い場合:
    - チャネルを剪定してよい。
    - チャネルを無視してよい。
    - 注: これは各ノード固有のポリシーであり、たとえば古いゴシップメッセージを受け取った際にチャネルをクローズするなどの形で、転送ピアによって強制されてはならない。

#### Rationale

いくつかのシナリオではチャネルが使用不能となり、エンドポイントが当該チャネルの更新を送信できなくなることがあります。例えば、両エンドポイントが秘密鍵へのアクセスを失い、`channel_update` への署名もオンチェーンでのチャネルクローズもできなくなる場合です。このようなチャネルはネットワークから分断されているため計算されたルートに含まれる可能性は低いものの、ローカルネットワークビューに残り続け、他のピアに無期限に転送され続けてしまいます。

剪定の判定にはより古い側の `channel_update` を使用します。チャネルが利用可能であるためには両側がアクティブである必要があるため、こうすることで、一方のノードが新しい `channel_update` を送り続けていても、もう一方が消えていれば剪定されるようになります。

## ルーティングに関する推奨事項

HTLC のルート計算では、`cltv_expiry_delta` と手数料の両方を考慮する必要があります。`cltv_expiry_delta` は、最悪のケースで失敗が発生した際に資金が利用できなくなる時間に寄与します。両者の関係は、関与するノードの信頼性に依存するため一概には決められません。

意図した受取人へ単純にルーティングし、`cltv_expiry_delta` を合計するだけでルートを構築すると、中間ノードがルート上での自身の位置を推測できてしまいます。HTLC の CLTV、周辺のネットワークトポロジー、`cltv_expiry_delta` の値を組み合わせれば、攻撃者は意図された受取人を推定する手がかりを得てしまいます。そのため、意図した受取人が受け取る CLTV にランダムなオフセットを追加することが強く望まれます。これにより、ルート全体の CLTV が一律に押し上げられます。

もっともらしいオフセットを作成するために、起点ノードは意図した受取人を起点としてグラフ上で制限付きのランダムウォークを行い、`cltv_expiry_delta` を合計し、その合計値をオフセットとして用いてよいでしょう。これにより、実際のルートに対する _シャドウルート拡張_ が効果的に得られ、単に乱数オフセットを選ぶよりもこの攻撃ベクトルに対する保護が高まります。

他のより高度な考慮事項としては、単一障害点や検出回避のためのルート選択の多様化、およびローカルチャネルのバランス調整などが挙げられます。

### ルーティングの例

4 つのノードを考えます。

```
   B
  / \
 /   \
A     C
 \   /
  \ /
   D
```

各ノードは、自身が持つ各チャネルの端で次の `cltv_expiry_delta` をアドバタイズします。

1. A: 10 ブロック
2. B: 20 ブロック
3. C: 30 ブロック
4. D: 40 ブロック

C はさらに、支払いを要求する際の `min_final_cltv_expiry_delta` として 18 (デフォルト値) を用います。

また、各ノードはチャネルごとに使用する手数料スキームを次のように設定しています。

1. A: 100 base + 1000 millionths
2. B: 200 base + 2000 millionths
3. C: 300 base + 3000 millionths
4. D: 400 base + 4000 millionths

ネットワークは 8 つの `channel_update` メッセージを受け取ります。

1. A->B: `cltv_expiry_delta` = 10, `fee_base_msat` = 100, `fee_proportional_millionths` = 1000
1. A->D: `cltv_expiry_delta` = 10, `fee_base_msat` = 100, `fee_proportional_millionths` = 1000
1. B->A: `cltv_expiry_delta` = 20, `fee_base_msat` = 200, `fee_proportional_millionths` = 2000
1. D->A: `cltv_expiry_delta` = 40, `fee_base_msat` = 400, `fee_proportional_millionths` = 4000
1. B->C: `cltv_expiry_delta` = 20, `fee_base_msat` = 200, `fee_proportional_millionths` = 2000
1. D->C: `cltv_expiry_delta` = 40, `fee_base_msat` = 400, `fee_proportional_millionths` = 4000
1. C->B: `cltv_expiry_delta` = 30, `fee_base_msat` = 300, `fee_proportional_millionths` = 3000
1. C->D: `cltv_expiry_delta` = 30, `fee_base_msat` = 300, `fee_proportional_millionths` = 3000

**B->C.** B が C に直接 4,999,999 ミリサトシを送る場合、自分自身に手数料を課すことも自身の `cltv_expiry_delta` を加えることもしないため、C が要求する `min_final_cltv_expiry_delta` の 18 を使用します。さらに、おそらく 42 ブロック分の追加 CLTV を与える _シャドウルート_ も加えるでしょう。これらの値は最小値を表すため、他のホップでも追加の CLTV デルタを上乗せできますが、ここでは簡略化のためそうしません。

   * `amount_msat`: 4999999
   * `cltv_expiry`: current-block-height + 18 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 18 + 42

**A->B->C.** A が B 経由で C に 4,999,999 ミリサトシを送る場合、B->C の `channel_update` で指定された手数料を B に支払う必要があります。これは [HTLC 手数料](#htlc-fees) に従って次のように計算します。

        fee_base_msat + ( amount_to_forward * fee_proportional_millionths / 1000000 )

        200 + ( 4999999 * 2000 / 1000000 ) = 10199

同様に、B->C の `channel_update` の `cltv_expiry_delta` (20)、C が要求する `min_final_cltv_expiry_delta` (18)、および _シャドウルート_ のコスト (42) を加算する必要があります。したがって、A->B の `update_add_htlc` メッセージは次のようになります。

   * `amount_msat`: 5010198
   * `cltv_expiry`: current-block-height + 20 + 18 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 18 + 42

B->C の `update_add_htlc` は、前述の B->C の直接支払いと同じ内容になります。

**A->D->C.** 最後に、何らかの理由で A が D 経由のより高価なルートを選んだ場合、A->D の `update_add_htlc` メッセージは次のようになります。

   * `amount_msat`: 5020398
   * `cltv_expiry`: 現在のブロック高 + 40 + 18 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = 現在のブロック高 + 18 + 42

そして D->C の `update_add_htlc` も、前述の B->C の直接支払いと同じ内容になります。


![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
