# BOLT #7: P2P ノードおよびチャネルの発見

この仕様書は、第三者に依存せずに情報を伝達するための、シンプルなノード発見、チャネル発見、およびチャネル更新のメカニズムについて説明します。

ノードとチャネルの発見は、異なる目的を持ちます：

- ノード発見は、ノードが自分の ID、ホスト、およびポートをブロードキャストし、他のノードが接続を開いて支払いチャネルを確立できるようにします。
- チャネル発見は、ネットワークのトポロジーのローカルビューを作成および維持し、ノードが目的の宛先へのルートを発見できるようにします。

チャネルとノードの発見をサポートするために、3 つの *ゴシップメッセージ* がサポートされています：

- ノード発見のために、ピアは `node_announcement` メッセージを交換し、ノードに関する追加情報を提供します。ノード情報を更新するために、複数の `node_announcement` メッセージが存在することがあります。

- チャネル発見のために、ネットワーク内のピアは、2 つのノード間の新しいチャネルに関する情報を含む `channel_announcement` メッセージを交換します。また、チャネルに関する情報を更新する `channel_update` メッセージを交換することもできます。どのチャネルに対しても有効な `channel_announcement` は 1 つだけですが、少なくとも 2 つの `channel_update` メッセージが期待されます。

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

`short_channel_id` は、資金調達トランザクションのユニークな記述です。以下のように構成されます：
  1. 最上位の 3 バイト：ブロックの高さを示します
  2. 次の 3 バイト：ブロック内のトランザクションインデックスを示します
  3. 最下位の 2 バイト：チャネルに支払う出力インデックスを示します。

`short_channel_id` の標準的な人間が読みやすい形式は、以下の要素を順に印刷することで作成されます：ブロックの高さ、トランザクションのインデックス、出力のインデックス。各要素は10進数で印刷され、小文字の `x` で区切られます。例えば、`short_channel_id` は `539268x845x1` と書かれることがあります。これは、ブロックの高さ 539268 のトランザクションのインデックス 845 の出力 1 にあるチャネルを示しています。

### Rationale

`short_channel_id` の人間が読みやすい形式は、ほとんどのシステムでダブルクリックやダブルタップすることで ID 全体を選択できるように設計されています。人間は数字を読むときに10進数を好むため、ID の要素は10進数で書かれています。小文字の `x` は、ほとんどのフォントで10進数の数字よりも視覚的に小さいため、ID の各要素を視覚的にグループ化しやすくするために使用されています。

## `announcement_signatures` メッセージ

これはチャネルの両端間の直接メッセージであり、チャネルをネットワークの他の部分に発表することを許可するオプトインメカニズムとして機能します。これは、`channel_announcement` メッセージを構築するために必要な署名を送信者によって含んでいます。

1. タイプ: 259 (`announcement_signatures`)
2. データ:
    * [`channel_id`:`channel_id`]
    * [`short_channel_id`:`short_channel_id`]
    * [`signature`:`node_signature`]
    * [`signature`:`bitcoin_signature`]

チャネルを発表するという開始ノードの意志は、チャネルの開設時に `channel_flags` の `announce_channel` ビットを設定することで示されます（[BOLT #2](02-peer-protocol.md#the-open_channel-message) を参照）。

### Requirements

`announcement_signatures` メッセージは、新たに確立されたチャネルに対応する `channel_announcement` メッセージを構築し、それをエンドポイントの `node_id` と `bitcoin_key` に対応する秘密で署名することで作成されます。署名された後、`announcement_signatures` メッセージを送信することができます。

ノードは：
  - `open_channel` メッセージに `announce_channel` ビットが設定されていて、`shutdown` メッセージが送信されていない場合：
    - `announcement_signatures` メッセージを送信しなければなりません。
      - `channel_ready` が送信および受信され、資金調達トランザクションが少なくとも6回の確認を受けるまで、`announcement_signatures` メッセージを送信してはなりません。
  - それ以外の場合：
    - `announcement_signatures` メッセージを送信してはなりません。
  - 再接続時（上記のタイミング要件が満たされた後）：
    - 最初の `announcement_signatures` メッセージに対して自分の `announcement_signatures` メッセージで応答しなければなりません。
    - `announcement_signatures` メッセージを受信していない場合：
      - `announcement_signatures` メッセージを再送信することを推奨します。

受信ノード：

- `short_channel_id` が正しくない場合：
  - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させるべきです。
- `node_signature` または `bitcoin_signature` が正しくない場合：
  - `warning` を送信して接続を閉じるか、`error` を送信してチャネルを失敗させてもかまいません。
- 有効な `announcement_signatures` メッセージを送信し、かつ受信した場合：
  - ピアに対して `channel_announcement` メッセージをキューに入れるべきです。
- `channel_ready` を送信していない場合：
  - `channel_ready` を送信した後まで announcement_signatures の処理を延期してもかまいません。
  - そうでない場合：
    - 無視しなければなりません。

### 理論的根拠

早期の announcement_signatures の延期を許可する理由は、以前の仕様では funding locked の受信を待つ必要がなかったためです。無視するのではなく延期することで、この動作との互換性を保つことができます。

## `channel_announcement` メッセージ

このゴシップメッセージは、チャネルに関する所有情報を含みます。各オンチェーンの Bitcoin キーを関連する Lightning ノードキーに結びつけ、その逆も同様です。少なくとも一方が `channel_update` を使用して手数料レベルと有効期限を発表するまで、チャネルは実際には使用できません。

`node_1` と `node_2` 間のチャネルの存在を証明するには、以下が必要です：

1. 資金調達トランザクションが `bitcoin_key_1` と `bitcoin_key_2` に支払われることを証明する
2. `node_1` が `bitcoin_key_1` を所有していることを証明する
3. `node_2` が `bitcoin_key_2` を所有していることを証明する

すべてのノードが未使用のトランザクション出力を知っていると仮定すると、最初の証明は `short_channel_id` によって与えられた出力を見つけ、それが [BOLT #3](03-transactions.md#funding-transaction-output) で指定されたキーの P2WSH 資金調達トランザクション出力であることを確認することで達成されます。

最後の二つの証明は明示的な署名によって達成されます：`bitcoin_signature_1` と `bitcoin_signature_2` は各 `bitcoin_key` に対して生成され、それぞれの対応する `node_id` が署名されます。

また、`node_1` と `node_2` がアナウンスメントメッセージに同意していることを証明する必要があります。これは、各 `node_id` (`node_signature_1` と `node_signature_2`) からの署名をメッセージに署名することで達成されます。


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

### 要件

オリジンノード：
  - `chain_hash` を、チャネルが開かれたチェーンを一意に識別する 32 バイトのハッシュに設定しなければなりません：
    - _ビットコインブロックチェーン_ の場合：
      - `chain_hash` の値（16進数でエンコードされたもの）を `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000` に設定しなければなりません。
  - `short_channel_id` を、[BOLT #2](02-peer-protocol.md#the-channel_ready-message) で指定されたように、確認済みの資金調達トランザクションを参照するように設定しなければなりません。
    - 注：対応する出力は、[BOLT #3](03-transactions.md#funding-transaction-output) で説明されているように P2WSH でなければなりません。
  - `node_id_1` と `node_id_2` を、チャネルを運営する 2 つのノードの公開鍵に設定し、`node_id_1` が圧縮キーのうち辞書順で小さい方になるように、昇順でソートされた 2 つの圧縮キーのうち辞書順で小さい方に設定しなければなりません。
  - `bitcoin_key_1` と `bitcoin_key_2` を、それぞれ `node_id_1` と `node_id_2` の `funding_pubkey` に設定しなければなりません。
  - メッセージのオフセット 256 からメッセージの終わりまでのダブル SHA256 ハッシュ `h` を計算しなければなりません。
    - 注：ハッシュは 4 つの署名をスキップしますが、メッセージの残りの部分をハッシュし、将来追加されるフィールドも含みます。
  - `node_signature_1` と `node_signature_2` を、ハッシュ `h` の有効な署名に設定しなければなりません（それぞれ `node_id_1` と `node_id_2` の秘密を使用）。
  - `bitcoin_signature_1` と `bitcoin_signature_2` を、ハッシュ `h` の有効な署名に設定しなければなりません（それぞれ `bitcoin_key_1` と `bitcoin_key_2` の秘密を使用）。
  - `features` を、このチャネルのために交渉された機能に基づいて設定しなければなりません。[BOLT #9](09-features.md#assigned-features-flags) に従ってください。
  - `len` を、設定された `features` ビットを保持するために必要な最小長に設定しなければなりません。

受信ノード：

- メッセージの署名を検証することで、メッセージの整合性と真正性を確認しなければなりません。
- `features` フィールドに未知の偶数ビットがある場合：
  - チャネルを通じてメッセージをルートしようとしてはなりません。
- `short_channel_id` の出力が P2WSH に対応していない場合（`bitcoin_key_1` と `bitcoin_key_2` を使用し、[BOLT #3](03-transactions.md#funding-transaction-output) で指定されている）または出力が消費されている場合：
  - メッセージを無視しなければなりません。
- 指定された `chain_hash` が受信者にとって未知の場合：
  - メッセージを無視しなければなりません。
- それ以外の場合：
  - `bitcoin_signature_1`、`bitcoin_signature_2`、`node_signature_1` または `node_signature_2` が無効または正しくない場合：
    - `warning` を送信するべきです。
    - 接続を閉じるかもしれません。
    - メッセージを無視しなければなりません。
  - それ以外の場合：
    - `node_id_1` または `node_id_2` がブラックリストに載っている場合：
      - メッセージを無視するべきです。
    - それ以外の場合：
      - 参照されたトランザクションが以前にチャネルとして発表されていない場合：
        - メッセージを再放送のためにキューに入れるべきです。
        - 最小期待長を超えるメッセージについては、そうしないことを選ぶかもしれません。
    - 同じトランザクションで、同じブロック内で、異なる `node_id_1` または `node_id_2` の有効な `channel_announcement` を以前に受信している場合：
      - 前のメッセージの `node_id_1` と `node_id_2`、およびこの `node_id_1` と `node_id_2` をブラックリストに載せ、それらに接続されたチャネルを忘れるべきです。
    - それ以外の場合：
      - この `channel_announcement` を保存するべきです。
- 資金出力が消費されたか再編成された場合：
  - 12 ブロックの遅延後にチャネルを忘れるべきです。

### 理論的根拠

両方のノードは、このチャネルを介して他の支払いをルートする意思があることを示すために署名する必要があります（すなわち、パブリックネットワークの一部であること）。彼らの Bitcoin 署名を要求することは、彼らがチャネルを制御していることを証明します。

競合するノードのブラックリスト化は、複数の異なるアナウンスを禁止します。このような競合するアナウンスは、キーが漏洩したことを意味するため、どのノードによっても放送されるべきではありません。

チャネルは十分に深くなるまで広告されるべきではありませんが、トランザクションが異なるブロックに移動していない場合にのみ、再放送に対する要件が適用されます。

過度に大きなメッセージを保存することを避けつつ、将来の合理的な拡張を可能にするために、ノードは再放送を制限することが許可されています（おそらく統計的に）。

将来的には新しいチャネル機能が可能です。後方互換性のある（またはオプションの）機能は_奇数_の機能ビットを持ち、互換性のない機能は_偶数_の機能ビットを持ちます（["It's OK to be odd!"](00-introduction.md#glossary-and-terminology-guide)）。

資金出力の消費に関するチャネルを忘れる際には、12 ブロックの遅延が使用されます。これは、このチャネルがスプライスされたことを示す新しい `channel_announcement` が伝播することを許可するためです。

## The `node_announcement` Message

このゴシップメッセージは、ノードが公開鍵に加えてそれに関連する追加データを示すことを可能にします。トリビアルなサービス拒否攻撃を避けるために、既に知られているチャネルに関連付けられていないノードは無視されます。

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

`timestamp` は、複数のアナウンスメントがある場合にメッセージの順序を決定するために使用されます。`rgb_color` と `alias` は、情報機関がノードに黒のような色や 'IRATEMONK' や 'WISTFULTOLL' のようなクールなニックネームを割り当てることを可能にします。

`addresses` は、ノードがネットワーク接続を受け入れる意思を示すことを可能にします。これは、ノードに接続するための一連の `address descriptor` を含みます。最初のバイトはアドレスタイプを示し、そのタイプに適したバイト数が続きます。

以下の `address descriptor` タイプが定義されています：

   * `1`: ipv4; data = `[4:ipv4_addr][2:port]` (長さ 6)
   * `2`: ipv6; data = `[16:ipv6_addr][2:port]` (長さ 18)
   * `3`: 廃止 (長さ 12)。Tor v2 オニオンサービスを含むために使用されました。
   * `4`: Tor v3 オニオンサービス; data = `[35:onion_addr][2:port]` (長さ 37)
       * バージョン 3 ([prop224](https://gitweb.torproject.org/torspec.git/tree/proposals/224-rend-spec-ng.txt))
         オニオンサービスアドレス。エンコード：
         `[32:32_byte_ed25519_pubkey] || [2:checksum] || [1:version]`、ここで
         `checksum = sha3(".onion checksum" || pubkey || version)[:2]`。
   * `5`: DNS ホスト名; data = `[1:hostname_len][hostname_len:hostname][2:port]` (長さ 最大 258)
       * `hostname` バイトは ASCII 文字でなければなりません。
       * 非 ASCII 文字は Punycode を使用してエンコードされなければなりません：
         https://en.wikipedia.org/wiki/Punycode

### 要件

オリジンノード：
  - `timestamp` を、それ以前に作成した `node_announcement` のいずれよりも大きく設定しなければなりません。
    - UNIXタイムスタンプに基づいてもかまいません。
  - `signature` を、`signature` 以降の残りのパケット全体のダブルSHA256の署名に設定しなければなりません（`node_id` によって与えられたキーを使用します）。
  - `alias` および `rgb_color` を設定して、マップやグラフでの外観をカスタマイズしてもかまいません。
    - 注：`rgb_color` の最初のバイトは赤の値、2番目のバイトは緑の値、最後のバイトは青の値です。
  - `alias` を有効なUTF-8文字列に設定し、`alias` の末尾のバイトは0にしなければなりません。
  - `addresses` を、着信接続を期待する各パブリックネットワークアドレスのアドレス記述子で埋めるべきです。
  - `addrlen` を `addresses` のバイト数に設定しなければなりません。
  - アドレス記述子を昇順に配置しなければなりません。
  - ゼロ型のアドレス記述子をどこにも配置すべきではありません。
  - `addresses` に続くフィールドを整列させるためだけに配置を使用すべきです。
  - `port` が0である `type 1`、`type 2`、または `type 5` のアドレス記述子を作成してはなりません。
  - `ipv4_addr` および `ipv6_addr` がルーティング可能なアドレスであることを確認すべきです。
  - `features` を [BOLT #9](09-features.md#assigned-features-flags) に従って設定しなければなりません。
  - `flen` を設定した `features` ビットを保持するのに必要な最小長に設定すべきです。
  - Tor v2 オニオンサービスをアナウンスすべきではありません。
  - `type 5` DNSホスト名を1つ以上アナウンスしてはなりません。

受信ノード：
  - `node_id` が有効な圧縮公開鍵でない場合：
    - `warning` を送信すべきです。
    - 接続を閉じてもかまいません。
    - メッセージをさらに処理してはなりません。
  - `signature` が（`signature` フィールドに続くメッセージ全体のダブルSHA256を使用して）有効な署名でない場合：
    - `warning` を送信すべきです。
    - 接続を閉じてもかまいません。
    - メッセージをさらに処理してはなりません。
  - `features` フィールドに未知の偶数ビットが含まれている場合：
    - ノードに接続すべきではありません。
    - 同じビットが設定されていない [BOLT #11](11-payment-encoding.md) 請求書を支払う場合を除き、ノードに支払いを送信しようとしてはなりません。
    - ノードを経由して支払いをルーティングしてはなりません。
  - 上記で定義されたタイプに一致しない最初の `address descriptor` を無視すべきです。
  - `addrlen` が既知のタイプのアドレス記述子を保持するのに不十分な場合：
    - `warning` を送信すべきです。
    - 接続を閉じてもかまいません。
  - `port` が0の場合：
    - `ipv6_addr` または `ipv4_addr` または `hostname` を無視すべきです。
  - `node_id` が `channel_announcement` メッセージから以前に知られていない場合、または `timestamp` がこの `node_id` から最後に受信した `node_announcement` よりも大きくない場合：
    - メッセージを無視すべきです。
  - それ以外の場合：
    - `timestamp` がこの `node_id` から最後に受信した `node_announcement` よりも大きい場合：
      - メッセージを再放送のためにキューに入れるべきです。
      - 最小期待長よりも長いメッセージをキューに入れないことを選んでもかまいません。
  - インターフェースでノードを参照するために `rgb_color` および `alias` を使用してもかまいません。
    - それらの自己署名の起源を示唆すべきです。
  - Tor v2 オニオンサービスを無視すべきです。
  - 1つ以上の `type 5` アドレスがアナウンスされた場合：
    - 追加のデータを無視すべきです。
    - `node_announcement` を転送してはなりません。

### 理論的根拠

将来的に新しいノード機能が可能になるかもしれません。後方互換性のある（またはオプションの）ものは _奇数_ の `feature` _ビット_ を持ち、互換性のないものは _偶数_ の `feature` _ビット_ を持ちます。これらは通常通り伝播されます。ここでの互換性のないフィーチャービットはノードに関するものであり、`node_announcement` メッセージ自体に関するものではありません。

将来的に新しいアドレスタイプが追加される可能性があります。アドレス記述子は昇順に並べられる必要があるため、未知のものは安全に無視できます。`addresses` を超える追加フィールドも将来的に追加される可能性があります。特定のアライメントが必要な場合は、`addresses` 内にオプションのパディングを含めることができます。

### ノードエイリアスのセキュリティに関する考慮事項

ノードエイリアスはユーザが定義するものであり、レンダリングや永続化の過程でインジェクション攻撃の可能性があります。

ノードエイリアスは、HTML/JavaScript コンテキストやその他の動的に解釈されるレンダリングフレームワークで表示される前に必ずサニタイズされるべきです。同様に、インジェクションの脆弱性や SQL やその他の動的に解釈されるクエリ言語をサポートする永続化エンジンから保護するために、準備されたステートメント、入力検証、およびエスケープを使用することを検討してください。

* [保存型および反射型 XSS 防止](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)
* [DOM ベースの XSS 防止](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL インジェクション防止](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

[Little Bobby Tables](https://xkcd.com/327/) の学校のようにならないでください。

## `channel_update` メッセージ

チャネルが最初にアナウンスされた後、各側はこのチャネルを通じて HTLC を中継するために必要な手数料と最小有効期限デルタを独立してアナウンスします。各側は `channel_announcement` と一致する 8 バイトのチャネル shortid と、チャネルのどちら側にいるかを示す 1 ビットの `channel_flags` フィールドを使用します（起点または終点）。ノードは手数料を変更するためにこれを複数回行うことができます。

`channel_update` ゴシップメッセージは、*支払いを送信する* のではなく、*支払いを中継する* 文脈でのみ有用です。支払いを `A` -> `B` -> `C` -> `D` と行う場合、`B` -> `C`（`B` によってアナウンスされた）および `C` -> `D`（`C` によってアナウンスされた）に関連する `channel_update` のみが関与します。ルートを構築する際には、HTLC の金額と有効期限を宛先から送信元に向かって逆に計算する必要があります。ルートの最後の HTLC に使用する `amount_msat` の正確な初期値と `cltv_expiry` の最小値は、支払い要求で提供されます（[BOLT #11](11-payment-encoding.md#tagged-fields) を参照）。


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

`channel_flags` ビットフィールドは、チャネルの方向を示すために使用されます。これは、この更新がどのノードから発信されたかを識別し、チャネルに関するさまざまなオプションを示します。以下の表は、その個々のビットの意味を示しています：

| ビット位置  | 名前        | 意味                          |
| ----------- | ----------- | ----------------------------- |
| 0           | `direction` | この更新が指す方向。          |
| 1           | `disable`   | チャネルを無効にする。        |

`message_flags` ビットフィールドは、メッセージに関する追加の詳細を提供するために使用されます：

| ビット位置  | 名前           |
| ----------- | ---------------|
| 0           | `must_be_one`  |
| 1           | `dont_forward` |

署名の検証に使用する `node_id` は、対応する `channel_announcement` から取得されます。フラグの最下位ビットが 0 の場合は `node_id_1`、それ以外の場合は `node_id_2` です。

### 要件

発信ノード：
  - `channel_ready` を受信する前に作成した `channel_update` を送信してはなりません。
  - チャネルがまだ発表されていない場合でも（つまり、`announce_channel` ビットが設定されていないか、ピアが [announcement signatures](#the-announcement_signatures-message) を交換する前に `channel_update` が送信された場合）、チャネルパラメータをチャネルピアに伝えるために `channel_update` を作成してもかまいません。
    - `short_channel_id` をピアから受け取った `alias` または実際のチャネル `short_channel_id` に設定しなければなりません。
    - `message_flags` の `dont_forward` を 1 に設定しなければなりません。
    - プライバシーの理由から、そのような `channel_update` を他のピアに転送してはなりません。
    - 注：そのような `channel_update` は、`channel_announcement` に先行されていない場合、他のピアにとって無効であり、破棄されます。
  - `signature` を、`signature` の後の残りのパケット全体のダブル-SHA256 の署名に設定しなければなりません。自身の `node_id` を使用します。
  - `chain_hash` と `short_channel_id` を、`channel_announcement` メッセージで指定されたチャネルを一意に識別する 32 バイトのハッシュと 8 バイトのチャネル ID に一致させなければなりません。
  - メッセージ内で発信ノードが `node_id_1` の場合：
    - `channel_flags` の `direction` ビットを 0 に設定しなければなりません。
  - それ以外の場合：
    - `channel_flags` の `direction` ビットを 1 に設定しなければなりません。
  - `htlc_maximum_msat` を、このチャネルを通じて単一の HTLC に対して送信する最大値に設定しなければなりません。
    - これをチャネル容量以下に設定しなければなりません。
    - これをピアから受け取った `max_htlc_value_in_flight_msat` 以下に設定しなければなりません。
    - これを `htlc_minimum_msat` 以上に設定しなければなりません。
  - `message_flags` の `must_be_one` を 1 に設定しなければなりません。
  - 意味が割り当てられていない `channel_flags` と `message_flags` のビットを 0 に設定しなければなりません。
  - チャネルの一時的な利用不可（例：接続の喪失）または恒久的な利用不可（例：オンチェーン決済前）を示すために、`disable` ビットを 1 に設定した `channel_update` を作成して送信してもかまいません。
    - チャネルを再有効化するために、`disable` ビットを 0 に設定した後続の `channel_update` を送信してもかまいません。
  - `timestamp` を 0 より大きく、かつこの `short_channel_id` に対して以前に送信された `channel_update` より大きく設定しなければなりません。
    - `timestamp` を UNIX タイムスタンプに基づいて設定することを推奨します。
  - `cltv_expiry_delta` を、受信 HTLC の `cltv_expiry` から差し引くブロック数に設定しなければなりません。
  - `htlc_minimum_msat` を、チャネルピアが受け入れる最小 HTLC 値（ミリサトシ単位）に設定しなければなりません。
    - `htlc_minimum_msat` を `htlc_maximum_msat` 以下に設定しなければなりません。
  - `fee_base_msat` を、任意の HTLC に対して請求する基本料金（ミリサトシ単位）に設定しなければなりません。
  - `fee_proportional_millionths` を、転送されたサトシごとに請求する金額（サトシの百万分の一単位）に設定しなければなりません。
  - 冗長な `channel_update` を作成しないことを推奨します。
  - チャネルパラメータを更新した新しい `channel_update` を作成する場合：
    - 以前のチャネルパラメータを 10 分間受け入れ続けることを推奨します。

受信ノード：

- `short_channel_id` が以前の `channel_announcement` と一致しない場合、またはその間にチャネルが閉じられた場合：
  - 自分のチャネルに対応しない `channel_update` を無視しなければなりません。
- 自分のチャネルに対する `channel_update` を受け入れるべきです (非公開であっても)、関連するオリジンノードの転送パラメータを学ぶためです。
- `signature` が無効な署名である場合、`signature` フィールドに続くメッセージ全体のダブル-SHA256 を `node_id` を使って検証する (未知のフィールドが `fee_proportional_millionths` の後に続く場合を含む)：
  - `warning` を送信し、接続を閉じるべきです。
  - メッセージをそれ以上処理してはなりません。
- 指定された `chain_hash` 値が未知である場合 (指定されたチェーンでアクティブでないことを意味する)：
  - チャネル更新を無視しなければなりません。
- `timestamp` がこの `short_channel_id` および `node_id` に対して最後に受信した `channel_update` と等しい場合：
  - `timestamp` 以下のフィールドが異なる場合：
    - この `node_id` をブラックリストに載せてもよいです。
    - それに関連するすべてのチャネルを忘れてもよいです。
  - `timestamp` 以下のフィールドが等しい場合：
    - このメッセージを無視するべきです。
- `timestamp` がこの `short_channel_id` および `node_id` に対して最後に受信した `channel_update` よりも低い場合：
  - メッセージを無視するべきです。
- それ以外の場合：
  - `timestamp` が将来に不合理に遠い場合：
    - `channel_update` を破棄してもよいです。
  - それ以外の場合：
    - メッセージを再放送のためにキューに入れるべきです。
    - 最小期待長より長いメッセージを選択しないかもしれません。
- `htlc_maximum_msat` < `htlc_minimum_msat` の場合：
  - 経路の考慮中にこのチャネルを無視するべきです。
- `htlc_maximum_msat` がチャネル容量を超える場合：
  - この `node_id` をブラックリストに載せてもよいです。
  - 経路の考慮中にこのチャネルを無視するべきです。
- それ以外の場合：
  - 経路選択時に `htlc_maximum_msat` を考慮するべきです。

### 理論的根拠

`timestamp` フィールドは、将来に不合理に遠いか、2週間更新されていない `channel_update` をプルーニングするためにノードによって使用されます。そのため、UNIX タイムスタンプ (すなわち UTC 1970-01-01 からの秒数) であることが理にかなっています。しかし、1秒以内に2つの `channel_update` がある可能性を考慮すると、これは厳格な要件にはなり得ません。

複数の `channel_update` メッセージが同じ秒内にチャンネルパラメータを変更する場合、DoS 攻撃の試みと見なされる可能性があり、そのようなメッセージに署名するノードはブラックリストに登録されることがあります。ただし、ノードは異なる署名（署名の署名時にノンスを変更する）で同じ `channel_update` メッセージを送信することができるため、署名以外のフィールドを確認して、同じタイムスタンプでチャンネルパラメータが変更されたかどうかを確認します。また、ECDSA 署名は可鍛性があることに注意することも重要です。そのため、`channel_update` メッセージを受け取った中間ノードは、署名の `s` コンポーネントを `-s` に変更するだけで再放送することができます。しかし、これによりメッセージが発信された `node_id` がブラックリストに登録されるべきではありません。

冗長な `channel_update` に対する推奨事項は、ネットワークのスパムを最小限に抑えますが、時には避けられないこともあります。たとえば、到達不能なピアとのチャンネルは、最終的にチャンネルが無効であることを示す `channel_update` を引き起こし、ピアが再接続したときにチャンネルを再有効化する別の更新が行われます。ゴシップメッセージはバッチ処理され、以前のものを置き換えるため、結果として単一の一見冗長な更新が行われることがあります。

ノードがチャンネルパラメータを変更するために新しい `channel_update` を作成すると、それがネットワーク全体に伝播するまでに時間がかかり、支払者は古いパラメータを使用する可能性があります。支払いの遅延と信頼性を向上させるために、少なくとも 10 分間は古いパラメータを受け入れ続けることが推奨されます。

`message_flags` の `must_be_one` フィールドは以前、`htlc_maximum_msat` フィールドの存在を示すために使用されていました。このフィールドは現在常に存在しなければならないため、`must_be_one` は定数値であり、受信者によって無視されます。

## Query Messages

メッセージの理解はかつて `gossip_queries` 機能ビットで示されていましたが、現在これらのメッセージは普遍的にサポートされており、その機能はわずかに再利用されています。この機能を提供しないことは、ノードがゴシップの問い合わせに値しないことを意味します。つまり、彼らはゴシップマップ全体を保存していないか、単一のピア（このノード）にのみ接続しているかのいずれかです。

いくつかのメッセージには、`short_channel_id` の長い配列（`encoded_short_ids` と呼ばれる）が含まれています。そのため、将来的に異なるエンコーディング方式を定義できるように、エンコーディングバイトを含めています。これにより、利点がある場合に対応できます。

エンコーディングタイプ：
* `0`: 昇順に並んだ `short_channel_id` タイプの非圧縮配列。
* `1`: 以前は zlib 圧縮に使用されていましたが、このエンコーディングは使用してはいけません。

このエンコーディングは他のタイプ（タイムスタンプ、フラグなど）の配列にも使用され、`encoded_` プレフィックスで指定されます。例えば、`encoded_timestamps` は `0` プレフィックスのタイムスタンプの配列です。

クエリメッセージは、ルーティングテーブルを同期するために必要なメッセージの数を減らすのに役立つオプションフィールドで拡張できます。これにより、以下が可能になります：

- `channel_update` メッセージのタイムスタンプベースのフィルタリング：すでに持っているものより新しい `channel_update` メッセージのみを要求します。
- `channel_update` メッセージのチェックサムベースのフィルタリング：すでに持っているものと異なる情報を持つ `channel_update` メッセージのみを要求します。

ノードは、`gossip_queries_ex` 機能ビットを使用して拡張ゴシップクエリをサポートしていることを示すことができます。

### `query_short_channel_ids`/`reply_short_channel_ids_end` メッセージ

1. タイプ: 261 (`query_short_channel_ids`)
2. データ:
    * [`chain_hash`:`chain_hash`]
    * [`u16`:`len`]
    * [`len*byte`:`encoded_short_ids`]
    * [`query_short_channel_ids_tlvs`:`tlvs`]

1. `tlv_stream`: `query_short_channel_ids_tlvs`
2. タイプ:
    1. タイプ: 1 (`query_flags`)
    2. データ:
        * [`byte`:`encoding_type`]
        * [`...*byte`:`encoded_query_flags`]

`encoded_query_flags` はビットフィールドの配列で、ビットフィールドごとに一つの bigsize があり、各 `short_channel_id` に対して一つのビットフィールドがあります。ビットの意味は以下の通りです：

| ビット位置   | 意味                                      |
| ------------- | ---------------------------------------- |
| 0             | 送信者は `channel_announcement` を希望   |
| 1             | 送信者はノード 1 の `channel_update` を希望 |
| 2             | 送信者はノード 2 の `channel_update` を希望 |
| 3             | 送信者はノード 1 の `node_announcement` を希望 |
| 4             | 送信者はノード 2 の `node_announcement` を希望 |

クエリフラグは最小限にエンコードされる必要があり、1 つのフラグは 1 バイトでエンコードされます。

1. タイプ: 262 (`reply_short_channel_ids_end`)
2. データ:
    * [`chain_hash`:`chain_hash`]
    * [`byte`:`full_information`]

これは、特定のチャネル（`short_channel_id` によって識別される）に対する `channel_announcement` および `channel_update` メッセージをノードがクエリするための一般的なメカニズムです。これは通常、ノードが `channel_announcement` を持たない `channel_update` を見た場合や、`reply_channel_range` から以前は知られていなかった `short_channel_id` を取得した場合に使用されます。

#### 要件

送信者:
  - `gossip_queries` を提供しないピアにこれを送信してはなりません。
  - 以前にこのピアに `query_short_channel_ids` を送信し、`reply_short_channel_ids_end` を受け取っていない場合、`query_short_channel_ids` を送信してはなりません。
  - `chain_hash` を、`short_channel_id` が参照するチェーンを一意に識別する 32 バイトのハッシュに設定しなければなりません。
  - `encoded_short_ids` の最初のバイトをエンコーディングタイプに設定しなければなりません。
  - `short_channel_id` の整数個を `encoded_short_ids` にエンコードしなければなりません。
  - `channel_announcement` を持たない `short_channel_id` に対する `channel_update` を受け取った場合、これを送信してもかまいません。
  - 参照されるチャネルが未使用の出力でない場合、これを送信してはなりません。
  - オプションの `query_flags` を含めてもかまいません。その場合:
    - `encoded_short_ids` と同様に `encoding_type` を設定しなければなりません。
    - 各クエリフラグは最小限にエンコードされた bigsize です。
    - 各 `short_channel_id` ごとに 1 つのクエリフラグをエンコードしなければなりません。

受信者:
  - `encoded_short_ids` の最初のバイトが既知のエンコーディングタイプでない場合:
    - `warning` を送信してもかまいません。
    - 接続を閉じてもかまいません。
  - `encoded_short_ids` が整数個の `short_channel_id` にデコードされない場合:
    - `warning` を送信してもかまいません。
    - 接続を閉じてもかまいません。
  - この送信者から以前に受け取った `query_short_channel_ids` に対して `reply_short_channel_ids_end` を送信していない場合:
    - `warning` を送信してもかまいません。
    - 接続を閉じてもかまいません。
  - 受信メッセージに `query_short_channel_ids_tlvs` が含まれている場合:
    - `encoding_type` が既知のエンコーディングタイプでない場合:
      - `warning` を送信してもかまいません。
      - 接続を閉じてもかまいません。
    - `encoded_query_flags` が `short_channel_id` ごとに正確に 1 つのフラグにデコードされない場合:
      - `warning` を送信してもかまいません。
      - 接続を閉じてもかまいません。
  - 各既知の `short_channel_id` に応答しなければなりません:
    - 受信メッセージに `encoded_query_flags` が含まれていない場合:
      - 各エンドの `channel_announcement` と最新の `channel_update` で応答しなければなりません。
      - 各 `channel_announcement` に対する `node_announcement` を続けなければなりません。
    - それ以外の場合:
      - `encoded_short_ids` の N 番目の `short_channel_id` に対する `query_flag` を、デコードされた `encoded_query_flags` の N 番目の bigsize と定義します。
      - `query_flag` のビット 0 がセットされている場合:
        - `channel_announcement` で応答しなければなりません。
      - `query_flag` のビット 1 がセットされており、`node_id_1` から `channel_update` を受け取っている場合:
        - `node_id_1` の最新の `channel_update` で応答しなければなりません。
      - `query_flag` のビット 2 がセットされており、`node_id_2` から `channel_update` を受け取っている場合:
        - `node_id_2` の最新の `channel_update` で応答しなければなりません。
      - `query_flag` のビット 3 がセットされており、`node_id_1` から `node_announcement` を受け取っている場合:
        - `node_id_1` の最新の `node_announcement` で応答しなければなりません。
      - `query_flag` のビット 4 がセットされており、`node_id_2` から `node_announcement` を受け取っている場合:
        - `node_id_2` の最新の `node_announcement` で応答しなければなりません。
    - これらを送信するために次の送信ゴシップフラッシュを待つべきではありません。
  - 単一の `query_short_channel_ids` に対する応答で重複した `node_announcements` を送信することを避けるべきです。
  - これらの応答に `reply_short_channel_ids_end` を続けなければなりません。
  - `chain_hash` に対する最新のチャネル情報を維持していない場合:
    - `full_information` を 0 に設定しなければなりません。
  - それ以外の場合:
    - `full_information` を 1 に設定するべきです。

#### 理由

将来のノードは完全な情報を持たないかもしれません。特に未知の `chain_hash` チェーンについては完全な情報を持たないでしょう。この `full_information` フィールド（以前は混乱を招くことから `complete` と呼ばれていました）は信頼できませんが、0 であれば送信者が他の場所で追加のデータを探すべきであることを示します。

明示的な `reply_short_channel_ids_end` メッセージにより、受信者が何も知らないことを示すことができ、送信者はタイムアウトに依存する必要がなくなります。また、クエリの自然なレート制限を引き起こします。

### `query_channel_range` と `reply_channel_range` メッセージ

1. タイプ: 263 (`query_channel_range`)
2. データ:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_blocknum`]
    * [`u32`:`number_of_blocks`]
    * [`query_channel_range_tlvs`:`tlvs`]

1. `tlv_stream`: `query_channel_range_tlvs`
2. タイプ:
    1. タイプ: 1 (`query_option`)
    2. データ:
        * [`bigsize`:`query_option_flags`]

`query_option_flags` は最小限にエンコードされた bigsize として表現されるビットフィールドです。ビットの意味は次のとおりです：

| ビット位置   | 意味                     |
| ------------- | ----------------------- |
| 0             | 送信者はタイムスタンプを希望 |
| 1             | 送信者はチェックサムを希望 |

チェックサムのみを要求することは可能ですが、タイムスタンプも要求しないとあまり役に立ちません。受信ノードが異なるチェックサムを持つ古い `channel_update` を持っているかもしれず、それを要求しても無意味です。また、`channel_update` のチェックサムが実際に 0 である場合（これは非常にありそうにないことですが）、それはクエリされません。

1. タイプ: 264 (`reply_channel_range`)
2. データ:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_blocknum`]
    * [`u32`:`number_of_blocks`]
    * [`byte`:`sync_complete`]
    * [`u16`:`len`]
    * [`len*byte`:`encoded_short_ids`]
    * [`reply_channel_range_tlvs`:`tlvs`]

1. `tlv_stream`: `reply_channel_range_tlvs`
2. タイプ:
    1. タイプ: 1 (`timestamps_tlv`)
    2. データ:
        * [`byte`:`encoding_type`]
        * [`...*byte`:`encoded_timestamps`]
    1. タイプ: 3 (`checksums_tlv`)
    2. データ:
        * [`...*channel_update_checksums`:`checksums`]

単一の `channel_update` に対して、タイムスタンプは次のようにエンコードされます：

1. サブタイプ: `channel_update_timestamps`
2. データ:
    * [`u32`:`timestamp_node_id_1`]
    * [`u32`:`timestamp_node_id_2`]

ここで：
* `timestamp_node_id_1` は `node_id_1` の `channel_update` のタイムスタンプであり、そのノードからの `channel_update` がない場合は 0 です。
* `timestamp_node_id_2` は `node_id_2` の `channel_update` のタイムスタンプであり、そのノードからの `channel_update` がない場合は 0 です。

単一の `channel_update` に対して、チェックサムは次のようにエンコードされます：

1. サブタイプ: `channel_update_checksums`
2. データ:
    * [`u32`:`checksum_node_id_1`]
    * [`u32`:`checksum_node_id_2`]

ここで：
* `checksum_node_id_1` は `node_id_1` の `channel_update` のチェックサムであり、そのノードからの `channel_update` がない場合は 0 です。
* `checksum_node_id_2` は `node_id_2` の `channel_update` のチェックサムであり、そのノードからの `channel_update` がない場合は 0 です。

`channel_update` のチェックサムは、`signature` と `timestamp` フィールドを除いたこの `channel_update` の CRC32C チェックサムです。詳細は [RFC3720](https://tools.ietf.org/html/rfc3720#appendix-B.4) を参照してください。

これにより、特定のブロック内のチャネルをクエリすることができます。

#### 要件

`query_channel_range` の送信者：
  - `gossip_queries` を提供しないピアにこれを送信してはなりません。
  - 以前にこのピアに `query_channel_range` を送信し、すべての `reply_channel_range` 応答を受け取っていない場合、これを送信してはなりません。
  - `chain_hash` を、`reply_channel_range` が参照するチェーンを一意に識別する 32 バイトのハッシュに設定しなければなりません。
  - `first_blocknum` を、チャネルを知りたい最初のブロックに設定しなければなりません。
  - `number_of_blocks` を 1 以上に設定しなければなりません。
  - 受け取りたい拡張情報の種類を指定する追加の `query_channel_range_tlv` を追加してもかまいません。

`query_channel_range` の受信者：
  - 以前にこの送信者から受け取った `query_channel_range` に対してすべての `reply_channel_range` を送信していない場合：
    - `warning` を送信してもかまいません。
    - 接続を閉じてもかまいません。
  - 1 つ以上の `reply_channel_range` で応答しなければなりません：
    - `chain_hash` を `query_channel_range` と同じに設定しなければなりません。
    - `number_of_blocks` を `encoded_short_ids` に収まる最大のブロック数に制限しなければなりません。
    - ブロックの内容を複数の `reply_channel_range` に分割してもかまいません。
    - 最初の `reply_channel_range` メッセージ：
      - `first_blocknum` を `query_channel_range` の `first_blocknum` 以下に設定しなければなりません。
      - `first_blocknum` に `number_of_blocks` を加えた値を `query_channel_range` の `first_blocknum` より大きくしなければなりません。
    - 続く `reply_channel_range` メッセージ：
      - `first_blocknum` を前の `first_blocknum` 以上にしなければなりません。
    - これは最終の `reply_channel_range` でない場合、`sync_complete` を `false` に設定しなければなりません。
    - 最終の `reply_channel_range` メッセージ：
      - `first_blocknum` に `number_of_blocks` を加えた値を `query_channel_range` の `first_blocknum` に `number_of_blocks` を加えた値以上にしなければなりません。
    - `sync_complete` を `true` に設定しなければなりません。

もし受信メッセージに `query_option` が含まれている場合、受信者は返信に追加情報を付加してもかまいません：

- `query_option_flags` のビット 0 が設定されている場合、受信者は `encoded_short_ids` 内のすべての `short_channel_id` に対する `channel_update` タイムスタンプを含む `timestamps_tlv` を付加してもかまいません。
- `query_option_flags` のビット 1 が設定されている場合、受信者は `encoded_short_ids` 内のすべての `short_channel_id` に対する `channel_update` チェックサムを含む `checksums_tlv` を付加してもかまいません。

#### Rationale

単一の応答が単一のパケットには大きすぎる場合があるため、複数の返信が必要になることがあります。ピアが（例えば）1000 ブロック範囲の結果を保存できるようにしたいので、返信は要求された範囲を超えてもかまいません。ただし、各返信が関連性を持つこと（要求された範囲と重なること）が必要です。

返信が増加順であることを主張することで、受信者は返信が完了したかどうかを簡単に判断できます。単に `first_blocknum` に `number_of_blocks` を加えた値が、要求した `first_blocknum` に `number_of_blocks` を加えた値に等しいか、またはそれを超えているかを確認するだけです。

タイムスタンプとチェックサムフィールドの追加により、ピアは冗長な更新のクエリを省略できます。

### The `gossip_timestamp_filter` Message

1. type: 265 (`gossip_timestamp_filter`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_timestamp`]
    * [`u32`:`timestamp_range`]

このメッセージは、ノードが将来のゴシップメッセージを特定の範囲に制限することを可能にします。ゴシップメッセージを受け取りたいノードはこれを送信する必要があります。そうでなければ、ゴシップメッセージは受信されません。

このフィルタは以前のものを置き換えるため、ピアからのゴシップを変更するために複数回使用できます。

#### Requirements

送信者：
  - `chain_hash` を、ゴシップが参照するチェーンを一意に識別する 32 バイトのハッシュに設定しなければなりません。
  - 受信者が `gossip_queries` を提供しない場合：
    - `first_timestamp` を 0xFFFFFFFF に、`timestamp_range` を 0 に設定するべきです。

受信者：
  - `timestamp` が `first_timestamp` 以上で、`first_timestamp` に `timestamp_range` を加えた値未満のすべてのゴシップメッセージを送信するべきです。
    - これらを送信するために次の送信ゴシップフラッシュを待ってもかまいません。
  - `timestamp` に関係なく、生成したゴシップメッセージを送信するべきです。
  - それ以外の場合（中継されたゴシップ）：
    - 将来のゴシップメッセージを `timestamp` が `first_timestamp` 以上で、`first_timestamp` に `timestamp_range` を加えた値未満のものに制限するべきです。
  - `channel_announcement` に対応する `channel_update` がない場合：
    - `channel_announcement` を送信してはなりません。
  - それ以外の場合：
    - 対応する `channel_update` の `timestamp` を `channel_announcement` の `timestamp` と見なさなければなりません。
    - 最初の対応する `channel_update` を受信した後に `channel_announcement` を送信するかどうかを考慮しなければなりません。
  - `channel_announcement` が送信される場合：
    - 対応する `channel_update` および `node_announcement` より前に `channel_announcement` を送信しなければなりません。

#### 理由

`channel_announcement` にはタイムスタンプがないため、適切なものを生成します。`channel_update` がない場合、それは全く送信されません。これは、剪定されたチャネルの場合に最も可能性が高いです。

通常、`channel_announcement` は `channel_update` によってすぐに続けられます。理想的には、最初（最も古い）`channel_update` のタイムスタンプを `channel_announcement` の時間として使用することを指定したいところですが、ネットワーク上の新しいノードはこれを持っておらず、さらに最初の `channel_update` タイムスタンプを保存する必要があります。代わりに、任意の更新を使用できるようにしており、これは実装が簡単です。

それでも `channel_announcement` が見逃された場合は、`query_short_channel_ids` を使用して取得できます。

ノードは `timestamp_filter` を使用して、多くのピアを持つ場合にゴシップ負荷を軽減できます（例：伝播が十分であると仮定して、最初の数ピアの後に `first_timestamp` を `0xFFFFFFFF` に設定）。この十分な伝播の仮定は、ノード自身が直接生成したゴシップメッセージには適用されないため、フィルタを無視するべきです。

### 要件

ノードは：
  - 自分で生成していないゴシップメッセージを、明示的に要求されない限り中継してはなりません。

## 再放送

### 要件

受信ノードは：
  - 新しい `channel_announcement` または `channel_update` または更新された `timestamp` を持つ `node_announcement` を受信した場合：
    - ネットワークのトポロジーに関するローカルビューを適宜更新するべきです。
  - アナウンスからの変更を適用した後：
    - 対応するオリジンノードに関連付けられたチャネルがない場合：
      - オリジンノードを既知のノードセットから削除してもかまいません。
    - それ以外の場合：
      - 適切なメタデータを更新し、アナウンスに関連付けられた署名を保存するべきです。
        - 注：これは後でノードがピアのためにアナウンスを再構築できるようにします。

ノードは：
  - `gossip_timestamp_filter` を受信するまで、自分で生成していないゴシップを送信してはなりません。
  - メッセージの到着時間に関係なく、60秒ごとに一度、送信ゴシップメッセージをフラッシュするべきです。
    - 注：これは重複しないユニークなアナウンスをもたらします。
    - `init` で `networks` を送信し、このゴシップメッセージの `chain_hash` を指定しなかったピアにゴシップメッセージを転送してはなりません。
  - 定期的に自分のチャネルを再アナウンスしてもかまいません。
    - 注：リソース要件を低く保つために、これは推奨されません。

### 理論的根拠

ゴシップメッセージが処理された後、それは処理ノードのピアに向けて送信されるメッセージのリストに追加され、発信ノードからの古い更新を置き換えます。このゴシップメッセージのリストは定期的にフラッシュされます。このようなストア・アンド・ディレイ・フォワードのブロードキャストは、_段階的ブロードキャスト_ と呼ばれます。また、このようなバッチ処理は、低オーバーヘッドで自然なレート制限を形成します。

## HTLC手数料

### 要件

発信ノード：
  - 次の手数料以上を支払うHTLCを受け入れるべきです（SHOULD）：
    - fee_base_msat + ( amount_to_forward * fee_proportional_millionths / 1000000 )
  - `channel_update`を送信した後、しばらくの間、古い手数料を支払うHTLCを受け入れるべきです（SHOULD）。
    - 注：これは伝播遅延を考慮したものです。

## ネットワークビューの剪定

### 要件

ノード：
  - ブロックチェーン内の資金調達トランザクションを監視して、閉じられているチャネルを特定するべきです（SHOULD）。
  - チャネルの資金調達出力が消費され、12ブロックの確認を受けた場合：
    - ローカルネットワークビューから削除されるべきです（SHOULD）し、閉じられたと見なされるべきです（SHOULD）。
  - 発表されたノードに関連するオープンチャネルがもうない場合：
    - `node_announcement`メッセージを通じて追加されたノードをローカルビューから剪定してもよいです（MAY）。
      - 注：これは、`node_announcement`が`channel_announcement`に先行するという依存関係の直接の結果です。

### 古いエントリの剪定に関する推奨事項

#### 要件

ノード：
  - 最新の`channel_update`の`timestamp`がどちらの方向でも2週間（1209600秒）以上前の場合：
    - チャネルを剪定してもよいです（MAY）。
    - チャネルを無視してもよいです（MAY）。
    - 注：これは個々のノードポリシーであり、転送ピアによって強制されるべきではありません（MUST NOT）、例えば、古いゴシップメッセージを受け取ったときにチャネルを閉じることによって。

#### 理論的根拠

いくつかのシナリオでは、チャネルが使用不能になり、そのエンドポイントがこれらのチャネルの更新を送信できなくなることがあります。例えば、両方のエンドポイントがプライベートキーへのアクセスを失い、`channel_update`に署名できず、オンチェーンでチャネルを閉じることもできない場合です。この場合、チャネルはネットワークの他の部分から分離されるため、計算されたルートの一部になる可能性は低いですが、ローカルネットワークビューに残り、他のピアに無期限に転送され続けます。

最も古い `channel_update` は、チャネルを剪定するために使用されます。チャネルが使用可能であるためには、両側がアクティブである必要があります。このようにすることで、一方のノードが新しい `channel_update` を送り続けていても、もう一方のノードが消えてしまった場合にはチャネルが剪定されます。

## ルーティングに関する推奨事項

HTLC のルートを計算する際には、`cltv_expiry_delta` と手数料の両方を考慮する必要があります。`cltv_expiry_delta` は、最悪の場合の失敗時に資金が利用できなくなる時間に寄与します。これら二つの属性の関係は、関与するノードの信頼性に依存するため不明確です。

単に意図した受取人にルーティングし、`cltv_expiry_delta` を合計することでルートを計算すると、中間ノードがルート内での自分の位置を推測することが可能になります。HTLC の CLTV、周囲のネットワークトポロジー、および `cltv_expiry_delta` を知ることで、攻撃者は意図した受取人を推測する方法を得ることになります。したがって、意図した受取人が受け取る CLTV にランダムなオフセットを追加することが非常に望ましいです。これにより、ルート全体の CLTV が上昇します。

もっともらしいオフセットを作成するために、起点ノードは意図した受取人から始めてグラフ上で制限されたランダムウォークを開始し、`cltv_expiry_delta` を合計し、その結果の合計をオフセットとして使用することができます。これにより、実際のルートに対する _シャドウルート拡張_ が効果的に作成され、単にランダムなオフセットを選ぶよりもこの攻撃ベクトルに対する保護が向上します。

他のより高度な考慮事項には、ルート選択の多様化が含まれ、単一障害点や検出を避け、ローカルチャネルのバランスを取ることが含まれます。

### ルーティングの例

4 つのノードを考えます：

```
   B
  / \
 /   \
A     C
 \   /
  \ /
   D
```

各ノードは、各チャネルの端で次の `cltv_expiry_delta` を広告しています：

1. A: 10 ブロック
2. B: 20 ブロック
3. C: 30 ブロック
4. D: 40 ブロック

C はまた、支払いを要求する際に `min_final_cltv_expiry_delta` として 18（デフォルト）を使用します。

また、各ノードは各チャネルに対して使用する設定された手数料スキームを持っています：

1. A: 100 ベース + 1000 ミリオンズ
2. B: 200 ベース + 2000 ミリオンズ
3. C: 300 ベース + 3000 ミリオンズ
4. D: 400 ベース + 4000 ミリオンズ

ネットワークは 8 つの `channel_update` メッセージを受け取ります。

1. A->B: `cltv_expiry_delta` = 10, `fee_base_msat` = 100, `fee_proportional_millionths` = 1000
1. A->D: `cltv_expiry_delta` = 10, `fee_base_msat` = 100, `fee_proportional_millionths` = 1000
1. B->A: `cltv_expiry_delta` = 20, `fee_base_msat` = 200, `fee_proportional_millionths` = 2000
1. D->A: `cltv_expiry_delta` = 40, `fee_base_msat` = 400, `fee_proportional_millionths` = 4000
1. B->C: `cltv_expiry_delta` = 20, `fee_base_msat` = 200, `fee_proportional_millionths` = 2000
1. D->C: `cltv_expiry_delta` = 40, `fee_base_msat` = 400, `fee_proportional_millionths` = 4000
1. C->B: `cltv_expiry_delta` = 30, `fee_base_msat` = 300, `fee_proportional_millionths` = 3000
1. C->D: `cltv_expiry_delta` = 30, `fee_base_msat` = 300, `fee_proportional_millionths` = 3000

**B->C.** B が C に直接 4,999,999 ミリサトシを送る場合、自分自身に手数料を課すことも、自分の `cltv_expiry_delta` を追加することもないので、C が要求する `min_final_cltv_expiry_delta` の 18 を使用します。おそらく、追加の CLTV として 42 を与える _シャドールート_ も追加するでしょう。さらに、これらの値は最小値を表すため、他のホップで追加の CLTV デルタを追加することもできますが、ここでは簡単のためにそうしないことを選びます。

   * `amount_msat`: 4999999
   * `cltv_expiry`: current-block-height + 18 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 18 + 42

**A->B->C.** A が B 経由で C に 4,999,999 ミリサトシを送る場合、B->C の `channel_update` で指定された手数料を B に支払う必要があります。これは [HTLC Fees](#htlc-fees) に従って計算されます。

        fee_base_msat + ( amount_to_forward * fee_proportional_millionths / 1000000 )

        200 + ( 4999999 * 2000 / 1000000 ) = 10199

同様に、B->C の `channel_update` の `cltv_expiry_delta` (20)、C が要求する `min_final_cltv_expiry_delta` (18)、および _シャドールート_ のコスト (42) を追加する必要があります。したがって、A->B の `update_add_htlc` メッセージは次のようになります。

   * `amount_msat`: 5010198
   * `cltv_expiry`: current-block-height + 20 + 18 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 18 + 42

B->C の `update_add_htlc` は、上記の B->C の直接支払いと同じになります。

**A->D->C.** 最後に、何らかの理由で A が D 経由のより高価なルートを選んだ場合、A->D の `update_add_htlc` メッセージは次のようになります：

   * `amount_msat`: 5020398
   * `cltv_expiry`: 現在のブロック高 + 40 + 18 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = 現在のブロック高 + 18 + 42

そして D->C の `update_add_htlc` も、上記の B->C の直接支払いと同じになります。


![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) の下でライセンスされています。
