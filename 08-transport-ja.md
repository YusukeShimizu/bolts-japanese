# BOLT #8: 暗号化および認証されたトランスポート

Lightning ノード間のすべての通信は暗号化されており、ノード間のすべての通信内容の機密性を提供します。また、悪意のある干渉を避けるために認証されています。各ノードは、Bitcoin の `secp256k1` 曲線上の公開鍵である既知の長期識別子を持っています。この長期公開鍵は、ピアとの暗号化および認証された接続を確立するためにプロトコル内で使用され、ノードを代表して広告される情報を認証するためにも使用されます。

# 目次

  * [暗号メッセージの概要](#cryptographic-messaging-overview)
    * [認証された鍵合意ハンドシェイク](#authenticated-key-agreement-handshake)
    * [ハンドシェイクのバージョニング](#handshake-versioning)
    * [Noise プロトコルのインスタンス化](#noise-protocol-instantiation)
  * [認証された鍵交換ハンドシェイクの仕様](#authenticated-key-exchange-handshake-specification)
    * [ハンドシェイク状態](#handshake-state)
    * [ハンドシェイク状態の初期化](#handshake-state-initialization)
    * [ハンドシェイク交換](#handshake-exchange)
  * [Lightning メッセージの仕様](#lightning-message-specification)
    * [メッセージの暗号化と送信](#encrypting-and-sending-messages)
    * [メッセージの受信と復号](#receiving-and-decrypting-messages)
  * [Lightning メッセージの鍵ローテーション](#lightning-message-key-rotation)
  * [セキュリティに関する考慮事項](#security-considerations)
  * [付録 A: トランスポートテストベクトル](#appendix-a-transport-test-vectors)
    * [イニシエータテスト](#initiator-tests)
    * [レスポンダテスト](#responder-tests)
    * [メッセージ暗号化テスト](#message-encryption-tests)
  * [謝辞](#acknowledgments)
  * [参考文献](#references)
  * [著者](#authors)

## 暗号メッセージの概要

Lightning メッセージを送信する前に、ノードはまずノード間で送信されるすべてのメッセージを暗号化および認証するために使用される暗号セッション状態を初期化しなければなりません (MUST)。この暗号セッション状態の初期化は、内部プロトコルメッセージヘッダや慣習とは完全に別個のものです。

2 つのノード間の通信は、2 つの異なるセグメントに分かれています。

1. 実際のデータ転送の前に、両方のノードが認証された鍵合意ハンドシェイクに参加します。これは Noise Protocol Framework<sup>[2](#reference-2)</sup> に基づいています。
2. 初期ハンドシェイクが成功した場合、ノードは Lightning メッセージ交換フェーズに入ります。Lightning メッセージ交換フェーズでは、すべてのメッセージが認証付き暗号化 (AEAD) の暗号文です。

### 認証された鍵合意ハンドシェイク

認証された鍵交換のために選ばれたハンドシェイクは `Noise_XK` です。プレメッセージとして、イニシエータはレスポンダのアイデンティティ公開鍵を知っている必要があります。これにより、レスポンダの静的公開鍵がハンドシェイク中に _決して_ 送信されないため、レスポンダのアイデンティティをある程度隠すことができます。代わりに、認証は一連の楕円曲線ディフィー・ヘルマン (ECDH) 操作とその後の MAC チェックを通じて暗黙的に達成されます。

認証された鍵合意 (`Noise_XK`) は 3 つの異なるステップ (アクト) で実行されます。ハンドシェイクの各アクト中に以下が行われます。いくつかの (暗号化されている可能性のある) 鍵素材が他方に送信されます。実行されているアクトに基づいて ECDH が実行され、その結果が現在の暗号化鍵セット (`ck` チェイン鍵と `k` 暗号化鍵) に混合されます。そして、ゼロ長の暗号文を持つ AEAD ペイロードが送信されます。このペイロードには長さがないため、MAC のみが送信されます。ECDH 出力をハッシュダイジェストに混合することで、インクリメンタルな TripleDH ハンドシェイクが形成されます。

Noise Protocol の言語を使用すると、`e` と `s` (どちらも公開鍵で、`e` はエフェメラル鍵、`s` は静的鍵で、通常は `nodeid` です) は暗号化されている可能性のある鍵素材を示し、`es`、`ee`、`se` はそれぞれ 2 つの鍵間の ECDH 操作を示します。ハンドシェイクは次のように配置されます。
```
    Noise_XK(s, rs):
       <- s
       ...
       -> e, es
       <- e, ee
       -> s, se
```
送信されるすべてのハンドシェイクデータ、鍵素材を含む、はセッション全体の「ハンドシェイクダイジェスト」`h` にインクリメンタルにハッシュされます。ハンドシェイク状態 `h` はハンドシェイク中に送信されることはありません。代わりに、ダイジェストはゼロ長 AEAD メッセージ内の関連データとして使用されます。

送信される各メッセージを認証することで、中間者攻撃 (MITM) がハンドシェイクの一部として送信されたデータを改ざんまたは置換していないことを保証します。もしそうであれば、MAC チェックが相手側で失敗します。

受信者による MAC のチェックが成功した場合、それは暗黙的にその時点までのすべての認証が成功したことを示します。ハンドシェイクプロセス中に MAC チェックが失敗した場合、接続は直ちに終了されるべきです。

### ハンドシェイクのバージョニング

初期ハンドシェイク中に送信される各メッセージは、現在のハンドシェイクに使用されるバージョンを示す単一の先頭バイトで始まります。バージョンが 0 の場合、変更は必要ありませんが、非ゼロのバージョンはクライアントがこのドキュメント内で最初に指定されたプロトコルから逸脱したことを示します。

クライアントは未知のバージョンで開始されたハンドシェイクの試みを拒否しなければなりません。

### Noise プロトコルの具体化

Noise プロトコルの具体的な実装には、3 つの抽象的な暗号オブジェクトの定義が必要です。ハッシュ関数、楕円曲線、および AEAD 暗号スキームです。Lightning では、`SHA-256` がハッシュ関数として選ばれ、`secp256k1` が楕円曲線として選ばれ、`ChaChaPoly-1305` が AEAD 構造として選ばれています。

使用される `ChaCha20` と `Poly1305` の組み合わせは、`RFC 8439`<sup>[1](#reference-1)</sup> に準拠していなければなりません。

Noise の Lightning バリアントの公式プロトコル名は `Noise_XK_secp256k1_ChaChaPoly_SHA256` です。この値の ASCII 文字列表現は、開始ハンドシェイク状態を初期化するために使用されるダイジェストにハッシュされます。2 つのエンドポイントのプロトコル名が異なる場合、ハンドシェイクプロセスは直ちに失敗します。

## 認証付き鍵交換ハンドシェイク仕様

ハンドシェイクは 1.5 ラウンドトリップを要する 3 つのアクトで進行します。各ハンドシェイクはヘッダーや追加のメタデータが付いていない _固定_ サイズのペイロードです。各アクトの正確なサイズは以下の通りです：

   * **アクト 1**：50 バイト
   * **アクト 2**：50 バイト
   * **アクト 3**：66 バイト

### ハンドシェイク状態

ハンドシェイクプロセス全体を通じて、各側はこれらの変数を維持します：


 * `ck`: **チェインキー**。この値は、すべての以前の ECDH 出力の累積ハッシュです。ハンドシェイクの最後に、`ck` は Lightning メッセージの暗号化キーを導出するために使用されます。

 * `h`: **ハンドシェイクハッシュ**。この値は、ハンドシェイクプロセス中に送受信されたすべてのハンドシェイクデータの累積ハッシュです。

 * `temp_k1`, `temp_k2`, `temp_k3`: **中間キー**。これらは、各ハンドシェイクメッセージの最後にゼロ長の AEAD ペイロードを暗号化および復号化するために使用されます。

 * `e`: パーティの**エフェメラルキーペア**。各セッションごとに、ノードは強力な暗号ランダム性を持つ新しいエフェメラルキーを生成しなければなりません。

 * `s`: パーティの**静的キーペア**（`ls` はローカル、`rs` はリモート）

以下の関数も参照されます：

  * `ECDH(k, rk)`: `k`（有効な `secp256k1` 秘密鍵）と `rk`（有効な公開鍵）を使用して楕円曲線ディフィー・ヘルマン操作を実行します。
      * 返される値は、生成されたポイントの圧縮形式の SHA256 です。

  * `HKDF(salt,ikm)`: `RFC 5869`<sup>[3](#reference-3)</sup> で定義された関数で、ゼロ長の `info` フィールドで評価されます。
     * `HKDF` のすべての呼び出しは、`HKDF` の抽出および拡張コンポーネントを使用して 64 バイトの暗号ランダム性を暗黙的に返します。

  * `encryptWithAD(k, n, ad, plaintext)`: `encrypt(k, n, ad, plaintext)` を出力します。
     * ここで `encrypt` は、渡された引数で `ChaCha20-Poly1305`（IETF バリアント）を評価したもので、ノンス `n` は 32 ビットのゼロビットとしてエンコードされ、その後に *リトルエンディアン* の 64 ビット値が続きます。注：これは通常のエンディアンではなく、Noise プロトコルの慣例に従います。

  * `decryptWithAD(k, n, ad, ciphertext)`: `decrypt(k, n, ad, ciphertext)` を出力します。
     * ここで `decrypt` は、渡された引数で `ChaCha20-Poly1305`（IETF バリアント）を評価したもので、ノンス `n` は 32 ビットのゼロビットとしてエンコードされ、その後に *リトルエンディアン* の 64 ビット値が続きます。

  * `generateKey()`: 新しい `secp256k1` キーペアを生成して返します。
     * `generateKey` によって返されるオブジェクトには 2 つの属性があります：
         * `.pub` は公開鍵を表す抽象オブジェクトを返します。
         * `.priv` は公開鍵を生成するために使用される秘密鍵を表します。
     * オブジェクトには 1 つのメソッドもあります：
         * `.serializeCompressed()`


  * `a || b` は、2 つのバイト列 `a` と `b` の連結を表します。

### ハンドシェイク状態の初期化

Act One の開始前に、両側はセッションごとの状態を次のように初期化します。

 1. `h = SHA-256(protocolName)`
    * ここで `protocolName = "Noise_XK_secp256k1_ChaChaPoly_SHA256"` は
      ASCII 文字列としてエンコードされます。

 2. `ck = h`

 3. `h = SHA-256(h || prologue)`
    * ここで `prologue` は ASCII 文字列：`lightning` です。

最後のステップとして、両側は応答者の公開鍵をハンドシェイクダイジェストに混ぜ込みます。

 * 開始ノードは、応答ノードの静的公開鍵を Bitcoin の圧縮形式でシリアライズして混ぜ込みます。
   * `h = SHA-256(h || rs.pub.serializeCompressed())`

 * 応答ノードは、ローカルの静的公開鍵を Bitcoin の圧縮形式でシリアライズして混ぜ込みます。
   * `h = SHA-256(h || ls.pub.serializeCompressed())`

### ハンドシェイク交換

#### Act One

```
    -> e, es
```

Act One は開始者から応答者に送信されます。Act One の間、開始者は応答者による暗黙のチャレンジを満たそうとします。このチャレンジを完了するには、開始者は応答者の静的公開鍵を知っている必要があります。

ハンドシェイクメッセージは _正確に_ 50 バイトです。1 バイトはハンドシェイクバージョン、33 バイトは開始者の圧縮されたエフェメラル公開鍵、16 バイトは `poly1305` タグです。

**送信者のアクション：**

1. `e = generateKey()`
2. `h = SHA-256(h || e.pub.serializeCompressed())`
     * 新しく生成されたエフェメラルキーが進行中のハンドシェイクダイジェストに蓄積されます。
3. `es = ECDH(e.priv, rs)`
     * 開始者は、新しく生成されたエフェメラルキーとリモートノードの静的公開鍵の間で ECDH を実行します。
4. `ck, temp_k1 = HKDF(ck, es)`
     * 新しい一時的な暗号化キーが生成され、認証 MAC を生成するために使用されます。
5. `c = encryptWithAD(temp_k1, 0, h, zero)`
     * ここで `zero` はゼロ長の平文です。
6. `h = SHA-256(h || c)`
     * 最後に、生成された暗号文が認証ハンドシェイクダイジェストに蓄積されます。
7. `m = 0 || e.pub.serializeCompressed() || c` をネットワークバッファを介して応答者に送信します。

**受信者のアクション:**

1. ネットワークバッファから「正確に」50バイトを読み取ります。
2. 読み取ったメッセージ (`m`) を `v`、`re`、`c` に解析します：
    * `v` は `m` の「最初の」バイト、`re` は `m` の次の 33 バイト、`c` は `m` の最後の 16 バイトです
    * リモートパーティの一時的な公開鍵 (`re`) の生のバイトは、鍵のシリアライズされた構成フォーマットによってエンコードされたアフィン座標を使用して曲線上の点にデシリアライズされます。
3. `v` が認識されないハンドシェイクバージョンの場合、応答者は接続試行を中止しなければなりません。
4. `h = SHA-256(h || re.serializeCompressed())`
    * 応答者は、イニシエータの一時的な鍵を認証ハンドシェイクダイジェストに蓄積します。
5. `es = ECDH(s.priv, re)`
    * 応答者は、その静的秘密鍵とイニシエータの一時的な公開鍵の間で ECDH を実行します。
6. `ck, temp_k1 = HKDF(ck, es)`
    * 新しい一時的な暗号化鍵が生成され、これはすぐに認証 MAC を確認するために使用されます。
7. `p = decryptWithAD(temp_k1, 0, h, c)`
    * この操作で MAC チェックが失敗した場合、イニシエータは応答者の静的公開鍵を知らないことになります。この場合、応答者はこれ以上のメッセージを送信せずに接続を終了しなければなりません。
8. `h = SHA-256(h || c)`
    * 受信した暗号文がハンドシェイクダイジェストに混ぜられます。このステップは、ペイロードが MITM によって変更されていないことを保証するためのものです。

#### Act Two

```
   <- e, ee
```

Act Two は応答者からイニシエータに送信されます。Act Two は、Act One が成功した場合にのみ行われます。Act One が成功したのは、応答者が Act One の最後に送信されたタグの MAC を正しく復号し、確認できた場合です。

ハンドシェイクは「正確に」50バイトです：1 バイトのハンドシェイクバージョン、33 バイトの応答者の圧縮された一時的な公開鍵、16 バイトの `poly1305` タグです。

**送信者のアクション:**

1. `e = generateKey()`
2. `h = SHA-256(h || e.pub.serializeCompressed())`
     * 新しく生成された一時的な鍵が実行中のハンドシェイクダイジェストに蓄積されます。
3. `ee = ECDH(e.priv, re)`
     * ここで `re` は、Act One 中に受信されたイニシエータの一時的な鍵です
4. `ck, temp_k2 = HKDF(ck, ee)`
     * 新しい一時的な暗号化鍵が生成され、これは認証 MAC を生成するために使用されます。
5. `c = encryptWithAD(temp_k2, 0, h, zero)`
     * ここで `zero` はゼロ長の平文です
6. `h = SHA-256(h || c)`
     * 最後に、生成された暗号文が認証ハンドシェイクダイジェストに蓄積されます。
7. `m = 0 || e.pub.serializeCompressed() || c` をネットワークバッファを介してイニシエータに送信します。

**受信者のアクション:**

1. ネットワークバッファから「正確に」50バイトを読み取ります。
2. 読み取ったメッセージ (`m`) を `v`、`re`、`c` に解析します：
    * `v` は `m` の「最初の」バイト、`re` は `m` の次の 33 バイト、`c` は `m` の最後の 16 バイトです。
3. `v` が認識されないハンドシェイクバージョンの場合、応答者は接続試行を中止しなければなりません。
4. `h = SHA-256(h || re.serializeCompressed())`
5. `ee = ECDH(e.priv, re)`
    * `re` は応答者の一時的な公開鍵です
    * リモートパーティの一時的な公開鍵 (`re`) の生のバイトは、鍵のシリアライズされた構成フォーマットによってエンコードされたアフィン座標を使用して曲線上のポイントにデシリアライズされます。
6. `ck, temp_k2 = HKDF(ck, ee)`
     * 新しい一時的な暗号化キーが生成され、認証 MAC を生成するために使用されます。
7. `p = decryptWithAD(temp_k2, 0, h, c)`
    * この操作で MAC チェックが失敗した場合、開始者はさらなるメッセージなしに接続を終了しなければなりません。
8. `h = SHA-256(h || c)`
     * 受信した暗号文はハンドシェイクダイジェストに混ぜられます。このステップは、ペイロードが MITM によって改ざんされていないことを保証するためのものです。

#### アクトスリー

```
   -> s, se
```

アクトスリーは、このセクションで説明されている認証済み鍵合意の最終段階です。このアクトは、開始者から応答者に送信される締めくくりのステップです。アクトスリーは、アクトツーが成功した場合にのみ実行されます。アクトスリーの間、開始者はハンドシェイクのこの時点で蓄積された `HKDF` 派生秘密鍵を使用して、強力な前方秘匿性で暗号化された静的公開鍵を応答者に送信します。

ハンドシェイクは「正確に」66バイトです：1バイトはハンドシェイクバージョン、33バイトは `ChaCha20` ストリーム暗号で暗号化された静的公開鍵、16バイトは AEAD 構造によって生成された暗号化された公開鍵のタグ、16バイトは最終的な認証タグです。

**送信者のアクション:**

1. `c = encryptWithAD(temp_k2, 1, h, s.pub.serializeCompressed())`
    * `s` は開始者の静的公開鍵です
2. `h = SHA-256(h || c)`
3. `se = ECDH(s.priv, re)`
    * `re` は応答者の一時的な公開鍵です
4. `ck, temp_k3 = HKDF(ck, se)`
    * 最終的な中間共有秘密が実行中のチェーンキーに混ぜられます。
5. `t = encryptWithAD(temp_k3, 0, h, zero)`
     * `zero` はゼロ長のプレーンテキストです
6. `sk, rk = HKDF(ck, zero)`
     * `zero` はゼロ長のプレーンテキストであり、
       `sk` は開始者が応答者にメッセージを暗号化するために使用するキー、
       `rk` は開始者が応答者から送信されたメッセージを復号するために使用するキーです
     * セッションの期間中にメッセージを送受信するために使用される最終的な暗号化キーが生成されます。
7. `rn = 0, sn = 0`
     * 送信および受信ノンスは 0 に初期化されます。
8. `rck = sck = ck`
     * 送信および受信チェーンキーは同じように初期化されます。
9. `m = 0 || c || t` をネットワークバッファに送信します。

**受信者のアクション:**

1. ネットワークバッファから _正確に_ 66 バイトを読み取ります。
2. 読み取ったメッセージ (`m`) を `v`、`c`、`t` に解析します：
    * `v` は `m` の _最初の_ バイト、`c` は `m` の次の 49 バイト、`t` は `m` の最後の 16 バイトです。
3. `v` が認識されないハンドシェイクバージョンの場合、応答者は接続試行を中止しなければなりません。
4. `rs = decryptWithAD(temp_k2, 1, h, c)`
     * この時点で、応答者はイニシエータの静的公開鍵を復元しています。
     * この操作で MAC チェックが失敗した場合、応答者はこれ以上メッセージを送信せずに接続を終了しなければなりません。
5. `h = SHA-256(h || c)`
6. `se = ECDH(e.priv, rs)`
     * ここで `e` は応答者の元のエフェメラルキーです。
7. `ck, temp_k3 = HKDF(ck, se)`
8. `p = decryptWithAD(temp_k3, 0, h, t)`
     * この操作で MAC チェックが失敗した場合、応答者はこれ以上メッセージを送信せずに接続を終了しなければなりません。
9. `rk, sk = HKDF(ck, zero)`
     * ここで `zero` はゼロ長のプレーンテキストであり、
       `rk` はイニシエータが送信するメッセージを応答者が復号するために使用するキーで、
       `sk` は応答者がイニシエータにメッセージを暗号化するために使用するキーです。
     * セッションの期間中にメッセージの送受信に使用する最終的な暗号化キーが生成されます。
10. `rn = 0, sn = 0`
     * 送信および受信のノンスが 0 に初期化されます。
11. `rck = sck = ck`
     * 送信および受信のチェインキーが同じように初期化されます。

## Lightning メッセージ仕様

アクト 3 の終了時に、両側はセッションの残りの期間中にメッセージを暗号化および復号するために使用する暗号化キーを導出します。

実際の Lightning プロトコルメッセージは AEAD 暗号文内にカプセル化されます。各メッセージは、次の Lightning メッセージの総長をエンコードする別の AEAD 暗号文でプレフィックスされます（その MAC を含まない）。

_いかなる_ Lightning メッセージの*最大*サイズも `65535` バイトを超えてはなりません。最大サイズを `65535` にすることで、テストが簡単になり、メモリ管理が容易になり、メモリ枯渇攻撃を軽減するのに役立ちます。

トラフィック解析をより困難にするために、すべての暗号化された Lightning メッセージの長さプレフィックスも暗号化されます。さらに、16 バイトの `Poly-1305` タグが暗号化された長さプレフィックスに追加され、パケットの長さが転送中に変更されていないことを確認し、復号オラクルを作成しないようにします。

パケットの構造は次のようになります：

```
+-------------------------------
|2-byte encrypted message length|
+-------------------------------
|  16-byte MAC of the encrypted |
|        message length         |
+-------------------------------
|                               |
|                               |
|     encrypted Lightning       |
|            message            |
|                               |
+-------------------------------
|     16-byte MAC of the        |
|      Lightning message        |
+-------------------------------
```

プレフィックス付きメッセージの長さは 2 バイトのビッグエンディアン整数としてエンコードされ、パケットの最大長は `2 + 16 + 65535 + 16` = `65569` バイトです。

### メッセージの暗号化と送信

Lightning メッセージ ( `m` ) をネットワークストリームに暗号化して送信するために、送信キー ( `sk` ) とノンス ( `sn` ) が与えられた場合、次の手順を実行します：

1. `l = len(m)` とします。
    * ここで `len` は Lightning メッセージのバイト長を取得します
2. `l` を 2 バイトのビッグエンディアン整数としてシリアライズします。
3. `l` を暗号化して ( `ChaChaPoly-1305` 、 `sn` 、および `sk` を使用)、`lc` (18 バイト) を取得します。
    * ノンス `sn` は 96 ビットのリトルエンディアン数としてエンコードされます。デコードされたノンスは 64 ビットであるため、96 ビットのノンスは次のようにエンコードされます：32 ビットの先行する 0 と 64 ビットの値。
        * このステップの後、ノンス `sn` はインクリメントされなければなりません。
    * ゼロ長のバイトスライスを AD (関連データ) として渡します。
4. 最後に、メッセージ自体 ( `m` ) を長さプレフィックスを暗号化するのと同じ手順を使用して暗号化します。暗号化された暗号文を `c` とします。
    * このステップの後、ノンス `sn` はインクリメントされなければなりません。
5. `lc || c` をネットワークバッファに送信します。

### メッセージの受信と復号

ネットワークストリーム内の次のメッセージを復号するために、次の手順を実行します：

1. ネットワークバッファから正確に 18 バイトを読み取ります。
2. 暗号化された長さプレフィックスを `lc` とします。
3. `lc` を復号して ( `ChaCha20-Poly1305` 、 `rn` 、および `rk` を使用)、暗号化されたパケットのサイズ `l` を取得します。
    * ゼロ長のバイトスライスを AD (関連データ) として渡します。
    * このステップの後、ノンス `rn` はインクリメントされなければなりません。
4. ネットワークバッファから正確に `l+16` バイトを読み取り、バイトを `c` とします。
5. `c` を復号して ( `ChaCha20-Poly1305` 、 `rn` 、および `rk` を使用)、復号されたプレーンテキストパケット `p` を取得します。
    * このステップの後、ノンス `rn` はインクリメントされなければなりません。

## Lightning メッセージキーのローテーション

定期的にキーを変更し、以前のキーを忘れることは、後にキーが漏洩した場合（すなわち後方秘匿性）に古いメッセージの復号を防ぐのに役立ちます。

キーのローテーションは、_それぞれの_ キー（`sk` と `rk`）に対して _個別に_ 行われ、`sck` と `rck` をそれぞれ使用します。キーは、あるパーティがそれを使って 1000 回暗号化または復号した後（すなわち 500 メッセージごと）にローテーションされるべきです。これは、専用のノンスが 1000 に達したときにキーをローテーションすることで適切に管理できます。

キー `k` のローテーションは、次のステップに従って行われます：

1. `ck` をチェインキーとする（すなわち `rk` の場合は `rck`、`sk` の場合は `sck`）
2. `ck', k' = HKDF(ck, k)`
3. キーのノンスを `n = 0` にリセットします。
4. `k = k'`
5. `ck = ck'`

# セキュリティに関する考慮事項

暗号化と復号には、既存の一般的に使用されている検証済みのライブラリを使用することを強く推奨します。これにより、多くの実装上の落とし穴を避けることができます。

# 付録 A: トランスポートテストベクトル

繰り返し可能なテストハンドシェイクを行うために、以下では各サイドに対して `generateKey()` が返すもの（すなわち `e.priv` の値）を指定します。これはランダム性を要求する仕様の違反であることに注意してください。

## イニシエーターテスト

イニシエーターは、この入力を与えられたときに指定された出力を生成するべきです。コメントはデバッグ目的での内部状態を反映しています。

```
    name: transport-initiator successful handshake
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    # Act One
    # h=0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c
    # ss=0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3
    # HKDF(0x2640f52eebcd9e882958951c794250eedb28002c05d7dc2ea0f195406042caf1,0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3)
    # ck,temp_k1=0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f
    # encryptWithAD(0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f, 0x000000000000000000000000, 0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c, <empty>)
    # c=0df6086551151f58b8afe6c195782c6a
    # h=0x9d1ffbb639e7e20021d9259491dc7b160aab270fb1339ef135053f6f2cebe9ce
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # re=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # h=0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf
    # ss=0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47
    # HKDF(0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47)
    # ck,temp_k2=0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc
    # decryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000000000000000000, 0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf, 0x6e2470b93aac583c9ef6eafca3f730ae)
    # h=0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72
    # Act Three
    # encryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000100000000000000, 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72, 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa)
    # c=0xb9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c3822
    # h=0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660
    # ss=0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766
    # HKDF(0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766)
    # ck,temp_k3=0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520
    # encryptWithAD(0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520, 0x000000000000000000000000, 0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660, <empty>)
    # t=0x8dc68b1c466263b47fdf31e560e139ba
    output: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    # HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,zero)
    output: sk,rk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9,0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442

    name: transport-initiator act2 short read test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730
    output: ERROR (ACT2_READ_FAILED)

    name: transport-initiator act2 bad version test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0102466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    output: ERROR (ACT2_BAD_VERSION 1)

    name: transport-initiator act2 bad key serialization test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0004466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    output: ERROR (ACT2_BAD_PUBKEY)

    name: transport-initiator act2 bad MAC test
    rs.pub: 0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv: 0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub: 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv: 0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub: 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    output: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    input: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730af
    output: ERROR (ACT2_BAD_TAG)
```

## レスポンダーテスト

レスポンダーは、この入力を与えられたときに指定された出力を生成するべきです。

```
    name: transport-responder successful handshake
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # re=0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    # h=0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c
    # ss=0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3
    # HKDF(0x2640f52eebcd9e882958951c794250eedb28002c05d7dc2ea0f195406042caf1,0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3)
    # ck,temp_k1=0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f
    # decryptWithAD(0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f, 0x000000000000000000000000, 0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c, 0x0df6086551151f58b8afe6c195782c6a)
    # h=0x9d1ffbb639e7e20021d9259491dc7b160aab270fb1339ef135053f6f2cebe9ce
    # Act Two
    # e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27 e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    # h=0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf
    # ss=0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47
    # HKDF(0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f,0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47)
    # ck,temp_k2=0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc
    # encryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000000000000000000, 0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf, <empty>)
    # c=0x6e2470b93aac583c9ef6eafca3f730ae
    # h=0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    # decryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000100000000000000, 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72, 0xb9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c3822)
    # rs=0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    # h=0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660
    # ss=0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766
    # HKDF(0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba,0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766)
    # ck,temp_k3=0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520
    # decryptWithAD(0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520, 0x000000000000000000000000, 0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660, 0x8dc68b1c466263b47fdf31e560e139ba)
    # HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,zero)
    output: rk,sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9,0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442

    name: transport-responder act1 short read test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c
    output: ERROR (ACT1_READ_FAILED)

    name: transport-responder act1 bad version test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x01036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    output: ERROR (ACT1_BAD_VERSION)

    name: transport-responder act1 bad key serialization test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00046360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    output: ERROR (ACT1_BAD_PUBKEY)

    name: transport-responder act1 bad MAC test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6b
    output: ERROR (ACT1_BAD_TAG)

    name: transport-responder act3 bad version test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x01b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    output: ERROR (ACT3_BAD_VERSION 1)

    name: transport-responder act3 short read test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139
    output: ERROR (ACT3_READ_FAILED)

    name: transport-responder act3 bad MAC for ciphertext test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00c9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    output: ERROR (ACT3_BAD_CIPHERTEXT)

    name: transport-responder act3 bad rs test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00bfe3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa2235536ad09a8ee351870c2bb7f78b754a26c6cef79a98d25139c856d7efd252c2ae73c
    # decryptWithAD(0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc, 0x000000000000000000000001, 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72, 0xd7fedc211450dd9602b41081c9bd05328b8bf8c0238880f7b7cb8a34bb6d8354081e8d4b81887fae47a74fe8aab3008653)
    # rs=0x044f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    output: ERROR (ACT3_BAD_PUBKEY)

    name: transport-responder act3 bad MAC test
    ls.priv=2121212121212121212121212121212121212121212121212121212121212121
    ls.pub=028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv=0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub=0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    # Act One
    input: 0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    # Act Two
    output: 0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    # Act Three
    input: 0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139bb
    output: ERROR (ACT3_BAD_TAG)
```

## メッセージ暗号化テスト

このテストでは、イニシエーターが長さ 5 のメッセージ "hello" を 1001 回送信します。簡潔さと 2 回のキーのローテーションをテストするために、6 つの例のみを示します：

	name: transport-message test
    ck=0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01
	sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9
	rk=0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442
    # encrypt l: cleartext=0x0005, AD=NULL, sn=0x000000000000000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0xcf2b30ddf0cf3f80e7c35a6e6730b59fe802
    # encrypt m: cleartext=0x68656c6c6f, AD=NULL, sn=0x000000000100000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x473180f396d88a8fb0db8cbcf25d2f214cf9ea1d95
	output 0: 0xcf2b30ddf0cf3f80e7c35a6e6730b59fe802473180f396d88a8fb0db8cbcf25d2f214cf9ea1d95
    # encrypt l: cleartext=0x0005, AD=NULL, sn=0x000000000200000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x72887022101f0b6753e0c7de21657d35a4cb
    # encrypt m: cleartext=0x68656c6c6f, AD=NULL, sn=0x000000000300000000000000, sk=0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x2a1f5cde2650528bbc8f837d0f0d7ad833b1a256a1
	output 1: 0x72887022101f0b6753e0c7de21657d35a4cb2a1f5cde2650528bbc8f837d0f0d7ad833b1a256a1
    # 0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8 = HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01, 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9)
    # 0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8 = HKDF(0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01, 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9)
    output 500: 0x178cb9d7387190fa34db9c2d50027d21793c9bc2d40b1e14dcf30ebeeeb220f48364f7a4c68bf8
    output 501: 0x1b186c57d44eb6de4c057c49940d79bb838a145cb528d6e8fd26dbe50a60ca2c104b56b60e45bd
    # 0x728366ed68565dc17cf6dd97330a859a6a56e87e2beef3bd828a4c4a54d8df06, 0x9e0477f9850dca41e42db0e4d154e3a098e5a000d995e421849fcd5df27882bd = HKDF(0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8)
    # 0x728366ed68565dc17cf6dd97330a859a6a56e87e2beef3bd828a4c4a54d8df06, 0x9e0477f9850dca41e42db0e4d154e3a098e5a000d995e421849fcd5df27882bd = HKDF(0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31, 0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8)
    output 1000: 0x4a2f3cc3b5e78ddb83dcb426d9863d9d9a723b0337c89dd0b005d89f8d3c05c52b76b29b740f09
    output 1001: 0x2ecd8c8a5629d0d02ab457a0fdd0f7b90a192cd46be5ecb6ca570bfc5e268338b1a16cf4ef2d36


# 謝辞

TODO(roasbeef); fin

# 参考文献
1. <a id="reference-1">https://tools.ietf.org/html/rfc8439</a>
2. <a id="reference-2">http://noiseprotocol.org/noise.html</a>
3. <a id="reference-3">https://tools.ietf.org/html/rfc5869</a>

# 著者

FIXME

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/)の下にライセンスされています。
