# BOLT #8: 暗号化および認証されたトランスポート

Lightning ノード間のすべての通信は、ノード間でやり取りされる内容の機密性を提供するために暗号化されており、また悪意のある干渉を防ぐために認証されています。各ノードは、Bitcoin の `secp256k1` 曲線上の公開鍵である既知の長期識別子を持っています。この長期公開鍵は、ピアとの暗号化および認証された接続を確立するためにプロトコル内で使用され、さらにノードを代表して広告される情報の認証にも使用されます。

# 目次

  * [暗号メッセージングの概要](#cryptographic-messaging-overview)
    * [認証付き鍵合意ハンドシェイク](#authenticated-key-agreement-handshake)
    * [ハンドシェイクのバージョニング](#handshake-versioning)
    * [Noise プロトコルのインスタンス化](#noise-protocol-instantiation)
  * [認証付き鍵交換ハンドシェイクの仕様](#authenticated-key-exchange-handshake-specification)
    * [ハンドシェイク状態](#handshake-state)
    * [ハンドシェイク状態の初期化](#handshake-state-initialization)
    * [ハンドシェイク交換](#handshake-exchange)
  * [Lightning メッセージの仕様](#lightning-message-specification)
    * [メッセージの暗号化と送信](#encrypting-and-sending-messages)
    * [メッセージの受信と復号](#receiving-and-decrypting-messages)
  * [Lightning メッセージの鍵ローテーション](#lightning-message-key-rotation)
  * [セキュリティに関する考慮事項](#security-considerations)
  * [付録 A: トランスポートテストベクタ](#appendix-a-transport-test-vectors)
    * [イニシエータのテスト](#initiator-tests)
    * [レスポンダのテスト](#responder-tests)
    * [メッセージ暗号化のテスト](#message-encryption-tests)
  * [謝辞](#acknowledgments)
  * [参考文献](#references)
  * [著者](#authors)

## 暗号メッセージングの概要

Lightning メッセージを送信する前に、ノード間で送受信されるすべてのメッセージの暗号化と認証に用いる暗号セッション状態を、ノードはまず初期化しなければなりません。この暗号セッション状態の初期化は、内側のプロトコルメッセージのヘッダや慣習とは完全に独立しています。

2 つのノード間の通信は、明確に区別される 2 つのセグメントに分かれます。

1. 実際のデータ転送に先立って、両ノードは認証付き鍵合意ハンドシェイクを行います。これは Noise Protocol Framework<sup>[2](#reference-2)</sup> に基づいています。
2. 初期ハンドシェイクが成功すると、ノードは Lightning メッセージ交換フェーズに入ります。このフェーズでは、すべてのメッセージが関連データ付き認証暗号 (AEAD) の暗号文となります。

### 認証付き鍵合意ハンドシェイク

認証付き鍵交換に採用されるハンドシェイクは `Noise_XK` です。プレメッセージとして、イニシエータはレスポンダの ID 公開鍵を事前に知っている必要があります。これにより、レスポンダの静的鍵がハンドシェイク中に _一度も_ 送信されないため、レスポンダにある程度の ID 隠蔽性が提供されます。代わりに、認証は一連の楕円曲線ディフィー・ヘルマン (ECDH) 演算とそれに続く MAC チェックによって暗黙的に達成されます。

認証付き鍵合意 (`Noise_XK`) は、3 つの異なるステップ (act) で実行されます。ハンドシェイクの各 act では次の処理が行われます。すなわち、(暗号化されている可能性のある) 鍵素材が相手に送信され、どの act を実行しているかに応じた ECDH が行われ、その結果が現在の暗号化鍵セット (チェイニングキーである `ck` と暗号化鍵である `k`) に混合されます。そして、ゼロ長の暗号文を持つ AEAD ペイロードが送信されます。このペイロードには長さがないため、実際に送信されるのは MAC のみです。ECDH の出力をハッシュダイジェストへ混合していくことで、インクリメンタルな TripleDH ハンドシェイクが構成されます。

Noise プロトコルの記法を用いると、`e` と `s` (どちらも公開鍵で、`e` はエフェメラル鍵、`s` は静的鍵であり、本仕様では通常 `nodeid` に相当します) は暗号化されている可能性のある鍵素材を表し、`es`、`ee`、`se` はそれぞれ 2 つの鍵間での ECDH 演算を表します。ハンドシェイクは以下のように記述されます。
```
    Noise_XK(s, rs):
       <- s
       ...
       -> e, es
       <- e, ee
       -> s, se
```
鍵素材を含め、ハンドシェイクで送信されるすべてのデータは、セッション全体で共有される「ハンドシェイクダイジェスト」`h` にインクリメンタルにハッシュされていきます。ハンドシェイク状態 `h` 自体はハンドシェイク中に送信されることはない点に注意してください。代わりに、このダイジェストはゼロ長の AEAD メッセージにおける関連データ (Associated Data) として使用されます。

送信される各メッセージを認証することで、中間者攻撃 (MITM) によってハンドシェイク中のデータが改ざん・置換されていないことを保証します。改ざんされていれば、相手側で MAC チェックが失敗します。

受信者による MAC チェックが成功することは、その時点までのすべての認証が成功したことを暗黙的に示します。ハンドシェイク中に MAC チェックが失敗した場合は、接続を直ちに終了する必要があります。

### ハンドシェイクのバージョニング

初期ハンドシェイク中に送信される各メッセージは、現在のハンドシェイクで使用されるバージョンを示す 1 バイトの先頭バイトで始まります。バージョンが 0 の場合は変更が不要であることを示し、非ゼロのバージョンは、本ドキュメントで当初定められたプロトコルからクライアントが逸脱したことを示します。

クライアントは、未知のバージョンで開始されたハンドシェイクの試行を拒否しなければなりません。

### Noise プロトコルのインスタンス化

Noise プロトコルを具体的にインスタンス化するには、3 つの抽象的な暗号オブジェクト、すなわちハッシュ関数、楕円曲線、AEAD 暗号方式を定義する必要があります。Lightning では、ハッシュ関数として `SHA-256`、楕円曲線として `secp256k1`、AEAD 構成として `ChaChaPoly-1305` を採用しています。

使用される `ChaCha20` と `Poly1305` の組み合わせは、`RFC 8439`<sup>[1](#reference-1)</sup> に準拠しなければなりません。

Lightning における Noise の正式なプロトコル名は `Noise_XK_secp256k1_ChaChaPoly_SHA256` です。この値の ASCII 文字列表現をハッシュしたダイジェストが、開始時のハンドシェイク状態の初期化に使用されます。両エンドポイントのプロトコル名が異なる場合、ハンドシェイクは直ちに失敗します。

## 認証付き鍵交換ハンドシェイクの仕様

ハンドシェイクは 3 つの act で進行し、1.5 ラウンドトリップを要します。各ハンドシェイクメッセージは、ヘッダや追加のメタデータを持たない _固定長_ のペイロードです。各 act の正確なサイズは以下のとおりです。

   * **Act One**: 50 バイト
   * **Act Two**: 50 バイト
   * **Act Three**: 66 バイト

### ハンドシェイク状態

ハンドシェイクの過程を通じて、各側は次の変数を保持します。

 * `ck`: **チェイニングキー**。これまでに行われたすべての ECDH 出力の累積ハッシュです。ハンドシェイク終了時に、`ck` は Lightning メッセージ用の暗号化鍵を導出するために使用されます。

 * `h`: **ハンドシェイクハッシュ**。ハンドシェイクの過程でこれまでに送受信された _すべての_ ハンドシェイクデータの累積ハッシュです。

 * `temp_k1`, `temp_k2`, `temp_k3`: **中間鍵**。各ハンドシェイクメッセージの末尾にあるゼロ長 AEAD ペイロードの暗号化と復号に使用されます。

 * `e`: 当事者の **エフェメラル鍵ペア**。各セッションで、ノードは強力な暗号論的乱数を用いて新しいエフェメラル鍵を生成しなければなりません。

 * `s`: 当事者の **静的鍵ペア** (ローカルは `ls`、リモートは `rs`)。

以下の関数も参照されます。

  * `ECDH(k, rk)`: `k` (有効な `secp256k1` 秘密鍵) と `rk` (有効な公開鍵) を用いて楕円曲線ディフィー・ヘルマン演算を実行します。
      * 戻り値は、生成された点を圧縮形式で表したものを SHA256 でハッシュした値です。

  * `HKDF(salt,ikm)`: `RFC 5869`<sup>[3](#reference-3)</sup> で定義される関数で、`info` フィールドはゼロ長として評価します。
     * `HKDF` の呼び出しは常に、`HKDF` の extract-and-expand コンポーネントを用いて 64 バイトの暗号論的ランダム値を返すものとします。

  * `encryptWithAD(k, n, ad, plaintext)`: `encrypt(k, n, ad, plaintext)` を出力します。
     * ここで `encrypt` は、与えられた引数による `ChaCha20-Poly1305` (IETF バリアント) の評価です。ノンス `n` は 32 ビットのゼロビットに続く *リトルエンディアン* 64 ビット値としてエンコードされます。注: これは Lightning における通常のエンディアンではなく、Noise プロトコルの慣例に従います。

  * `decryptWithAD(k, n, ad, ciphertext)`: `decrypt(k, n, ad, ciphertext)` を出力します。
     * ここで `decrypt` は、与えられた引数による `ChaCha20-Poly1305` (IETF バリアント) の評価です。ノンス `n` は 32 ビットのゼロビットに続く *リトルエンディアン* 64 ビット値としてエンコードされます。

  * `generateKey()`: 新しい `secp256k1` 鍵ペアを生成して返します。
     * `generateKey` が返すオブジェクトには 2 つの属性があります。
         * `.pub` は公開鍵を表す抽象オブジェクトを返します。
         * `.priv` は、その公開鍵の生成に使われた秘密鍵を表します。
     * このオブジェクトは次の 1 つのメソッドも持ちます。
         * `.serializeCompressed()`

  * `a || b` は、2 つのバイト列 `a` と `b` の連結を表します。

### ハンドシェイク状態の初期化

Act One の開始前に、両側はセッションごとの状態を次のように初期化します。

 1. `h = SHA-256(protocolName)`
    * ここで `protocolName = "Noise_XK_secp256k1_ChaChaPoly_SHA256"` を ASCII 文字列としてエンコードしたものです。

 2. `ck = h`

 3. `h = SHA-256(h || prologue)`
    * ここで `prologue` は ASCII 文字列 `lightning` です。

最後の手順として、両側はレスポンダの公開鍵をハンドシェイクダイジェストに混ぜ込みます。

 * イニシエータ側のノードは、レスポンダ側ノードの静的公開鍵を Bitcoin の圧縮形式でシリアライズしたものを混ぜ込みます。
   * `h = SHA-256(h || rs.pub.serializeCompressed())`

 * レスポンダ側のノードは、自身のローカル静的公開鍵を Bitcoin の圧縮形式でシリアライズしたものを混ぜ込みます。
   * `h = SHA-256(h || ls.pub.serializeCompressed())`

### ハンドシェイク交換

#### Act One

```
    -> e, es
```

Act One はイニシエータからレスポンダに送信されます。Act One において、イニシエータはレスポンダによる暗黙のチャレンジを満たそうとします。このチャレンジを完了するには、イニシエータがレスポンダの静的公開鍵を知っている必要があります。

ハンドシェイクメッセージは _ちょうど_ 50 バイトです。内訳は、ハンドシェイクバージョンに 1 バイト、イニシエータの圧縮されたエフェメラル公開鍵に 33 バイト、`poly1305` タグに 16 バイトです。

**送信側のアクション:**

1. `e = generateKey()`
2. `h = SHA-256(h || e.pub.serializeCompressed())`
     * 新たに生成されたエフェメラル鍵が、進行中のハンドシェイクダイジェストに蓄積されます。
3. `es = ECDH(e.priv, rs)`
     * イニシエータは、新しく生成した自身のエフェメラル鍵とリモートノードの静的公開鍵との間で ECDH を行います。
4. `ck, temp_k1 = HKDF(ck, es)`
     * 認証用の MAC を生成するために用いる、新しい一時的な暗号化鍵が生成されます。
5. `c = encryptWithAD(temp_k1, 0, h, zero)`
     * ここで `zero` はゼロ長の平文です。
6. `h = SHA-256(h || c)`
     * 最後に、生成された暗号文が認証用ハンドシェイクダイジェストに蓄積されます。
7. `m = 0 || e.pub.serializeCompressed() || c` をネットワークバッファ経由でレスポンダに送信します。

**受信側のアクション:**

1. ネットワークバッファから _ちょうど_ 50 バイトを読み取ります。
2. 読み取ったメッセージ (`m`) を `v`、`re`、`c` にパースします。
    * `v` は `m` の _最初の_ 1 バイト、`re` は続く 33 バイト、`c` は最後の 16 バイトです。
    * リモート側のエフェメラル公開鍵 (`re`) の生バイト列は、その鍵のシリアライズされた合成形式に従い、アフィン座標で表される曲線上の点としてデシリアライズされなければなりません。
3. `v` が未知のハンドシェイクバージョンの場合、レスポンダは接続の試行を中止しなければなりません。
4. `h = SHA-256(h || re.serializeCompressed())`
    * レスポンダは、イニシエータのエフェメラル鍵を認証用ハンドシェイクダイジェストに蓄積します。
5. `es = ECDH(s.priv, re)`
    * レスポンダは、自身の静的秘密鍵とイニシエータのエフェメラル公開鍵との間で ECDH を行います。
6. `ck, temp_k1 = HKDF(ck, es)`
    * 認証用の MAC を直後に検証するために用いる、新しい一時的な暗号化鍵が生成されます。
7. `p = decryptWithAD(temp_k1, 0, h, c)`
    * この処理で MAC チェックが失敗した場合、イニシエータはレスポンダの静的公開鍵を知らないことになります。この場合、レスポンダはこれ以上メッセージを送らずに接続を終了しなければなりません。
8. `h = SHA-256(h || c)`
     * 受信した暗号文がハンドシェイクダイジェストに混ぜ込まれます。この手順は、ペイロードが MITM によって改ざんされていないことを保証する役割を持ちます。

#### Act Two

```
   <- e, ee
```

Act Two はレスポンダからイニシエータに送信されます。Act Two は Act One が成功した場合に _のみ_ 行われます。Act One の成功とは、Act One の末尾で送られたタグの MAC をレスポンダが正しく復号・検証できたことを意味します。

ハンドシェイクメッセージは _ちょうど_ 50 バイトです。内訳は、ハンドシェイクバージョンに 1 バイト、レスポンダの圧縮されたエフェメラル公開鍵に 33 バイト、`poly1305` タグに 16 バイトです。

**送信側のアクション:**

1. `e = generateKey()`
2. `h = SHA-256(h || e.pub.serializeCompressed())`
     * 新たに生成されたエフェメラル鍵が、進行中のハンドシェイクダイジェストに蓄積されます。
3. `ee = ECDH(e.priv, re)`
     * ここで `re` は、Act One で受信したイニシエータのエフェメラル鍵です。
4. `ck, temp_k2 = HKDF(ck, ee)`
     * 認証用の MAC を生成するために用いる、新しい一時的な暗号化鍵が生成されます。
5. `c = encryptWithAD(temp_k2, 0, h, zero)`
     * ここで `zero` はゼロ長の平文です。
6. `h = SHA-256(h || c)`
     * 最後に、生成された暗号文が認証用ハンドシェイクダイジェストに蓄積されます。
7. `m = 0 || e.pub.serializeCompressed() || c` をネットワークバッファ経由でイニシエータに送信します。

**受信側のアクション:**

1. ネットワークバッファから _ちょうど_ 50 バイトを読み取ります。
2. 読み取ったメッセージ (`m`) を `v`、`re`、`c` にパースします。
    * `v` は `m` の _最初の_ 1 バイト、`re` は続く 33 バイト、`c` は最後の 16 バイトです。
3. `v` が未知のハンドシェイクバージョンの場合、レスポンダは接続の試行を中止しなければなりません。
4. `h = SHA-256(h || re.serializeCompressed())`
5. `ee = ECDH(e.priv, re)`
    * ここで `re` はレスポンダのエフェメラル公開鍵です。
    * リモート側のエフェメラル公開鍵 (`re`) の生バイト列は、その鍵のシリアライズされた合成形式に従い、アフィン座標で表される曲線上の点としてデシリアライズされなければなりません。
6. `ck, temp_k2 = HKDF(ck, ee)`
     * 認証用の MAC を生成するために用いる、新しい一時的な暗号化鍵が生成されます。
7. `p = decryptWithAD(temp_k2, 0, h, c)`
    * この処理で MAC チェックが失敗した場合、イニシエータはこれ以上メッセージを送らずに接続を終了しなければなりません。
8. `h = SHA-256(h || c)`
     * 受信した暗号文がハンドシェイクダイジェストに混ぜ込まれます。この手順は、ペイロードが MITM によって改ざんされていないことを保証する役割を持ちます。

#### Act Three

```
   -> s, se
```

Act Three は、本節で説明する認証付き鍵合意の最終段階です。この act は、締めくくりのステップとしてイニシエータからレスポンダに送信されます。Act Three は、Act Two が成功した場合に _かつその場合のみ_ 実行されます。Act Three において、イニシエータは、ハンドシェイクのこの時点までに蓄積された `HKDF` 由来の秘密鍵を用いて、強い前方秘匿性のもとに自身の静的公開鍵を暗号化してレスポンダへ送信します。

ハンドシェイクメッセージは _ちょうど_ 66 バイトです。内訳は、ハンドシェイクバージョンに 1 バイト、`ChaCha20` ストリーム暗号で暗号化された静的公開鍵に 33 バイト、AEAD 構成によって生成される暗号化済み公開鍵のタグに 16 バイト、最終認証タグに 16 バイトです。

**送信側のアクション:**

1. `c = encryptWithAD(temp_k2, 1, h, s.pub.serializeCompressed())`
    * ここで `s` はイニシエータの静的公開鍵です。
2. `h = SHA-256(h || c)`
3. `se = ECDH(s.priv, re)`
    * ここで `re` はレスポンダのエフェメラル公開鍵です。
4. `ck, temp_k3 = HKDF(ck, se)`
    * 最後の中間共有秘密が、進行中のチェイニングキーに混ぜ込まれます。
5. `t = encryptWithAD(temp_k3, 0, h, zero)`
     * ここで `zero` はゼロ長の平文です。
6. `sk, rk = HKDF(ck, zero)`
     * ここで `zero` はゼロ長の平文であり、`sk` はイニシエータがレスポンダ宛てメッセージを暗号化するための鍵、`rk` はイニシエータがレスポンダから受信したメッセージを復号するための鍵です。
     * セッション中のメッセージ送受信に用いる、最終的な暗号化鍵が生成されます。
7. `rn = 0, sn = 0`
     * 送信ノンスと受信ノンスは 0 に初期化されます。
8. `rck = sck = ck`
     * 送信用と受信用のチェイニングキーは同じ値で初期化されます。
9. `m = 0 || c || t` をネットワークバッファ経由で送信します。

**受信側のアクション:**

1. ネットワークバッファから _ちょうど_ 66 バイトを読み取ります。
2. 読み取ったメッセージ (`m`) を `v`、`c`、`t` にパースします。
    * `v` は `m` の _最初の_ 1 バイト、`c` は続く 49 バイト、`t` は最後の 16 バイトです。
3. `v` が未知のハンドシェイクバージョンの場合、レスポンダは接続の試行を中止しなければなりません。
4. `rs = decryptWithAD(temp_k2, 1, h, c)`
     * この時点で、レスポンダはイニシエータの静的公開鍵を復元したことになります。
     * この処理で MAC チェックが失敗した場合、レスポンダはこれ以上メッセージを送らずに接続を終了しなければなりません。
5. `h = SHA-256(h || c)`
6. `se = ECDH(e.priv, rs)`
     * ここで `e` はレスポンダの元のエフェメラル鍵です。
7. `ck, temp_k3 = HKDF(ck, se)`
8. `p = decryptWithAD(temp_k3, 0, h, t)`
     * この処理で MAC チェックが失敗した場合、レスポンダはこれ以上メッセージを送らずに接続を終了しなければなりません。
9. `rk, sk = HKDF(ck, zero)`
     * ここで `zero` はゼロ長の平文であり、`rk` はレスポンダがイニシエータから受信したメッセージを復号するための鍵、`sk` はレスポンダがイニシエータ宛てメッセージを暗号化するための鍵です。
     * セッション中のメッセージ送受信に用いる、最終的な暗号化鍵が生成されます。
10. `rn = 0, sn = 0`
     * 送信ノンスと受信ノンスは 0 に初期化されます。
11. `rck = sck = ck`
     * 送信用と受信用のチェイニングキーは同じ値で初期化されます。

## Lightning メッセージの仕様

Act Three の終了時点で、両側はセッションの残り期間にわたってメッセージの暗号化と復号に用いる暗号化鍵を導出済みとなります。

実際の Lightning プロトコルメッセージは、AEAD 暗号文の中にカプセル化されます。各メッセージの直前には、続く Lightning メッセージの全長 (MAC を含まない) をエンコードした別の AEAD 暗号文がプレフィックスとして付加されます。

_いかなる_ Lightning メッセージも、その *最大* サイズが `65535` バイトを超えてはなりません。最大サイズを `65535` とすることで、テストが単純になり、メモリ管理が容易になり、メモリ枯渇攻撃の緩和にも役立ちます。

トラフィック解析を困難にするために、暗号化されたすべての Lightning メッセージの長さプレフィックスもまた暗号化されます。さらに、暗号化された長さプレフィックスには 16 バイトの `Poly-1305` タグが付加され、転送中にパケット長が改ざんされていないことを保証するとともに、復号オラクルの作成を防ぎます。

ワイヤ上のパケット構造は次のようになります。

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

プレフィックスとして付加されるメッセージ長は 2 バイトのビッグエンディアン整数としてエンコードされ、パケット全体の最大長は `2 + 16 + 65535 + 16` = `65569` バイトとなります。

### メッセージの暗号化と送信

送信用鍵 (`sk`) とノンス (`sn`) を用いて Lightning メッセージ (`m`) を暗号化しネットワークストリームへ送信するには、次の手順を踏みます。

1. `l = len(m)` とします。
    * ここで `len` は Lightning メッセージのバイト長を返します。
2. `l` を 2 バイトのビッグエンディアン整数としてシリアライズします。
3. `l` を `ChaChaPoly-1305`、`sn`、`sk` を用いて暗号化し、`lc` (18 バイト) を得ます。
    * ノンス `sn` は 96 ビットのリトルエンディアン値としてエンコードします。デコード後のノンスは 64 ビットなので、96 ビットのノンスは 32 ビットのゼロに続けて 64 ビット値を並べる形でエンコードされます。
        * この手順の後、ノンス `sn` をインクリメントしなければなりません。
    * 関連データ (AD) としてはゼロ長のバイトスライスを渡します。
4. 最後に、長さプレフィックスの暗号化と同じ手順でメッセージ本体 (`m`) を暗号化します。得られた暗号文を `c` とします。
    * この手順の後、ノンス `sn` をインクリメントしなければなりません。
5. `lc || c` をネットワークバッファに送出します。

### メッセージの受信と復号

ネットワークストリームから _次の_ メッセージを復号するには、次の手順を踏みます。

1. ネットワークバッファから _ちょうど_ 18 バイトを読み取ります。
2. その暗号化された長さプレフィックスを `lc` とします。
3. `lc` を `ChaCha20-Poly1305`、`rn`、`rk` を用いて復号し、暗号化されたパケットのサイズ `l` を得ます。
    * 関連データ (AD) としてはゼロ長のバイトスライスを渡します。
    * この手順の後、ノンス `rn` をインクリメントしなければなりません。
4. ネットワークバッファから _ちょうど_ `l+16` バイトを読み取り、これを `c` とします。
5. `c` を `ChaCha20-Poly1305`、`rn`、`rk` を用いて復号し、平文のパケット `p` を得ます。
    * この手順の後、ノンス `rn` をインクリメントしなければなりません。

## Lightning メッセージの鍵ローテーション

定期的に鍵を更新し古い鍵を破棄することは、後に鍵が漏洩した場合 (すなわち後方秘匿性の確保) に過去のメッセージが復号されるのを防ぐうえで有用です。

鍵ローテーションは _それぞれの_ 鍵 (`sk` と `rk`) について _個別に_ 行われ、それぞれ `sck` と `rck` を用いて実施されます。鍵は、ある当事者がそれを用いて 1000 回の暗号化または復号を行った後 (すなわち 500 メッセージごと) にローテーションされます。これは、その鍵に対応するノンスが 1000 に達した時点で鍵をローテーションすることで、適切に管理できます。

鍵 `k` のローテーションは次の手順で行います。

1. `ck` を対応するチェイニングキーとします (すなわち、`rk` には `rck`、`sk` には `sck`)。
2. `ck', k' = HKDF(ck, k)`
3. その鍵に対応するノンスを `n = 0` にリセットします。
4. `k = k'`
5. `ck = ck'`

# セキュリティに関する考慮事項

暗号化と復号には、既存のよく使われている検証済みのライブラリを利用することを強く推奨します。これにより、実装上の落とし穴を多く回避できます。

# 付録 A: トランスポートテストベクタ

再現可能なテスト用ハンドシェイクを行うため、以下では各側の `generateKey()` が返す値 (すなわち `e.priv` の値) を指定します。これは乱数性を要求する仕様への違反である点に注意してください。

## イニシエータのテスト

イニシエータは、以下の入力を与えられたとき、指定された出力を生成すべきです。コメントはデバッグ用に内部状態を示しています。

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

## レスポンダのテスト

レスポンダは、以下の入力を与えられたとき、指定された出力を生成すべきです。

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

## メッセージ暗号化のテスト

このテストでは、イニシエータが長さ 5 のメッセージ "hello" を 1001 回送信します。簡潔さの観点と、2 回の鍵ローテーションを確認する目的から、出力例は 6 件のみを示します。

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
