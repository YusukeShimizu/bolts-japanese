# BOLT #0: はじめにと索引

ようこそ、友よ！これらの Basis of Lightning Technology (BOLT) ドキュメントは、相互の協力によってオフチェーンでビットコインを送金するためのレイヤー2プロトコルを記述しています。必要な場合には、強制執行のためにオンチェーントランザクションに依拠します。

要件のなかには微妙なものもあります。ここに書かれている結果に至った動機や論拠を、できるだけ明確にするように努めました。それでも至らない部分はあるはずです。混乱する箇所や誤りを見つけたら、ぜひご連絡いただき、改善にご協力ください。

これはバージョン 0 です。

1. [BOLT #1](01-messaging.md): 基本プロトコル
2. [BOLT #2](02-peer-protocol.md): チャネル管理のためのピアプロトコル
3. [BOLT #3](03-transactions.md): ビットコイントランザクションとスクリプトのフォーマット
4. [BOLT #4](04-onion-routing.md): オニオンルーティングプロトコル
5. [BOLT #5](05-onchain.md): オンチェーントランザクション処理に関する推奨事項
7. [BOLT #7](07-routing-gossip.md): P2P ノードおよびチャネル発見
8. [BOLT #8](08-transport.md): 暗号化および認証付きトランスポート
9. [BOLT #9](09-features.md): 割り当て済みの機能フラグ
10. [BOLT #10](10-dns-bootstrap.md): DNS ブートストラップとノード位置特定の支援
11. [BOLT #11](11-payment-encoding.md): ライトニング支払いのためのインボイスプロトコル
12. [BOLT #12](12-offer-encoding.md): ライトニング支払いのためのネゴシエーションプロトコル

## The Spark: ライトニングの簡単な紹介

ライトニングは、チャネルのネットワークを利用して、ビットコインで高速な支払いを行うためのプロトコルです。

### チャネル

ライトニングは *チャネル* を確立することで動作します。2人の参加者は、ビットコインネットワーク上にロックした一定量のビットコイン（例: 0.1 ビットコイン）を保持するライトニング支払いチャネルを作成します。このビットコインは、両者の署名がそろったときにのみ使用できます。

最初は、それぞれが「すべてのビットコイン（例: 0.1 ビットコイン）を一方の当事者に戻す」内容のビットコイントランザクションを保持します。その後、これらの資金を別の比率で分配する新しいビットコイントランザクションに署名できます。たとえば、一方に 0.09 ビットコイン、もう一方に 0.01 ビットコインを割り当て、以前のビットコイントランザクションを無効化して使用されないようにします。

チャネル確立の詳細は [BOLT #2: Channel Establishment](02-peer-protocol.md#channel-establishment) を、チャネルを生成するビットコイントランザクションのフォーマットは [BOLT #3: Funding Transaction Output](03-transactions.md#funding-transaction-output) を参照してください。参加者の意見が一致しない場合や障害が発生した場合に、クロス署名済みのビットコイントランザクションを使用しなければならないときの要件については、[BOLT #5: Recommendations for On-chain Transaction Handling](05-onchain.md) を参照してください。

### 条件付き支払い

ライトニングのチャネルは 2 人の参加者の間でしか支払いを行えませんが、複数のチャネルを連結してネットワークを構成することで、ネットワーク内のすべての参加者の間で支払いが可能になります。これを実現するには、チャネルに追加できる条件付き支払いの仕組みが必要です。たとえば「6 時間以内に秘密を提示すれば 0.01 ビットコインを受け取れる」といった条件です。受取人が秘密を提示すると、そのビットコイントランザクションは、条件付き支払いを取り除き、その資金を受取人の出力に加えたものに置き換えられます。

参加者が条件付き支払いを追加するために使用するコマンドについては [BOLT #2: Adding an HTLC](02-peer-protocol.md#adding-an-htlc-update_add_htlc) を、ビットコイントランザクションの完全なフォーマットについては [BOLT #3: Commitment Transaction](03-transactions.md#commitment-transaction) を参照してください。

### 転送

このような条件付き支払いは、より短い時間制限を付けて別の参加者に安全に転送できます。たとえば「5 時間以内に秘密を提示すれば 0.01 ビットコインを受け取れる」といった具合です。これにより、仲介者を信頼することなく、複数のチャネルを連結してネットワークを構築できます。

支払いの転送に関する詳細は [BOLT #2: Forwarding HTLCs](02-peer-protocol.md#forwarding-htlcs) を、支払い指示がどのように運ばれるかは [BOLT #4: Packet Structure](04-onion-routing.md#packet-structure) を参照してください。

### ネットワークトポロジ

支払いを行うためには、参加者がどのチャネルを通じて送れるかを把握する必要があります。参加者同士は、チャネルやノードの作成、およびそれらの更新について情報を交換します。

通信プロトコルの詳細は [BOLT #7: P2P Node and Channel Discovery](07-routing-gossip.md) を、初期のネットワークブートストラップについては [BOLT #10: DNS Bootstrap and Assisted Node Location](10-dns-bootstrap.md) を参照してください。

### 支払いのインボイス

参加者は、どの支払いを行うべきかを示すインボイス（請求書）を受け取ります。

支払いの宛先と目的を記述し、支払者があとで支払い成功を証明できるようにするためのプロトコルについては、[BOLT #11: Invoice Protocol for Lightning Payments](11-payment-encoding.md) を参照してください。


## 用語集と用語ガイド

* #### *Announcement*:
   * *[peers](#peers)* 間で送られるゴシップメッセージで、*[channel](#channel)* または *[node](#node)* の発見を支援することを目的とします。

* #### `chain_hash`:
   * 対象ブロックチェーンを一意に識別するハッシュ（通常はジェネシスハッシュ）です。これにより、*[ノード](#node)* は複数のブロックチェーン上で *チャネル* を作成・参照できます。ノードは、自分が知らない `chain_hash` を参照するメッセージは無視します。`bitcoin-cli` とは異なり、ハッシュは反転させずにそのまま使用します。

     メインチェーンであるビットコインブロックチェーンの場合、`chain_hash` の値は次でなければならない（16 進数でエンコード）:
     `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`。

* #### *Channel*:
   * 2 つの *[ピア](#peers)* の間で行われる、高速なオフチェーンの相互交換手段です。
   資金をやり取りするために、ピアは署名を交換し、更新された *[コミットメントトランザクション](#commitment-transaction)* を作成します。
   * _クローズ方法を参照: [mutual close](#mutual-close), [revoked transaction close](#revoked-transaction-close), [unilateral close](#unilateral-close)_
   * _関連項目を参照: [route](#route)_

* #### *Closing transaction*:
   * *[mutual close](#mutual-close)* の一部として生成されるトランザクションです。クローズトランザクションは _コミットメントトランザクション_ に似ていますが、保留中の支払いは含まれません。
   * _関連項目を参照: [commitment transaction](#commitment-transaction), [funding transaction](#funding-transaction), [penalty transaction](#penalty-transaction)_

* #### *Commitment number*:
   * 各 *[コミットメントトランザクション](#commitment-transaction)* に付けられる 48 ビットのインクリメントカウンタです。カウンタは *チャネル* 内のピアごとに独立しており、0 から始まります。
   * _コンテナを参照: [commitment transaction](#commitment-transaction)_
   * _関連項目を参照: [closing transaction](#closing-transaction), [funding transaction](#funding-transaction), [penalty transaction](#penalty-transaction)_

* #### *Commitment revocation private key*:
   * 各 *[コミットメントトランザクション](#commitment-transaction)* には、相手 *ピア* がすべての出力を直ちに使用できるようにする、固有のコミットメント取り消し秘密鍵の値があります。この鍵を公開することが、古いコミットメントトランザクションを取り消す手段です。取り消しをサポートするため、コミットメントトランザクションの各出力はコミットメント取り消し公開鍵を参照します。
   * _コンテナを参照: [commitment transaction](#commitment-transaction)_
   * _派生元を参照: [per-commitment secret](#per-commitment-secret)_

* #### *Commitment transaction*:
   * *[資金調達トランザクション](#funding-transaction)* を消費するトランザクションです。
   各 *ピア* は相手ピアによるこのトランザクションへの署名を保持しているため、いつでも消費可能なコミットメントトランザクションを持っています。新しいコミットメントトランザクションが交渉されると、古いものは *取り消されます*。
   * _構成要素を参照: [commitment number](#commitment-number), [commitment revocation private key](#commitment-revocation-private-key), [HTLC](#HTLC-Hashed-Time-Locked-Contract), [per-commitment secret](#per-commitment-secret), [outpoint](#outpoint)_
   * _関連項目を参照: [closing transaction](#closing-transaction), [funding transaction](#funding-transaction), [penalty transaction](#penalty-transaction)_
   * _派生型を参照: [revoked commitment transaction](#revoked-commitment-transaction)_

* #### *Fail the channel*:
  * チャネルの強制クローズを意味します。ごく初期（オープン前）であれば、チャネルの存在を忘れるだけで済む場合もあります。通常は、最新のコミットメントトランザクションに署名してブロードキャストする必要がありますが、相互クローズの最中であれば、相互クローズトランザクションに署名してブロードキャストすることでも実行できます。詳細は [BOLT #5](05-onchain.md#failing-a-channel) を参照してください。

* #### *Close the connection*:
  * ピアとの通信（たとえば TCP ソケット）を閉じることを意味します。ピアとのチャネルを閉じることは含みませんが、チャネルを伴う接続については、未コミット状態の破棄を引き起こします。詳細は [BOLT #2](02-peer-protocol.md#message-retransmission) を参照してください。

* #### *Final node*:
   * *[origin node](#origin-node)* から複数の *[hop](#hop)* を経由して支払いをルーティングするパケットの最終受信者です。チェーンにおける最後の *[receiving peer](#receiving-peer)* でもあります。
   * _カテゴリを参照: [node](#node)_
   * _関連項目を参照: [origin node](#origin-node), [processing node](#processing-node)_

* #### *Funding transaction*:
   * *[チャネル](#channel)* 上の両 *[ピア](#peers)* に対して支払う、不可逆なオンチェーントランザクションです。両者の合意のもとでのみ消費できます。
   * _関連項目を参照: [closing transaction](#closing-transaction), [commitment transaction](#commitment-transaction), [penalty transaction](#penalty-transaction)_

* #### *Hop*:
   * *[ノード](#node)*。一般には、*[origin node](#origin-node)* と *[final node](#final-node)* の間に位置する中間ノードを指します。
   * _カテゴリを参照: [node](#node)_

* #### *HTLC*: Hashed Time Locked Contract（ハッシュタイムロック契約）。
   * 2 つの *[ピア](#peers)* 間の条件付き支払いです。受取人は、自身の署名と *payment preimage* を提示することで支払いを取得できます。そうでなければ、支払人は所定の時間が経過した後に契約を取り消し、支払いをキャンセルできます。HTLC は *[コミットメントトランザクション](#commitment-transaction)* の出力として実装されます。
   * _コンテナを参照: [commitment transaction](#commitment-transaction)_
   * _構成要素を参照: [Payment hash](#Payment-hash), [Payment preimage](#Payment-preimage)_

* #### *Invoice*: ライトニングネットワーク上での資金要求であり、支払いの種類、金額、有効期限、その他の情報を含むことがあります。これは、ビットコイン形式のアドレスを使う代わりに、ライトニングネットワーク上で支払いを行うための仕組みです。

* #### *It's ok to be odd*:
   * 一部の数値フィールドに適用されるルールで、機能サポートが任意か必須かを示します。偶数は、両エンドポイントが該当機能をサポートしなければならないことを示し、奇数は、相手側のエンドポイントがその機能を無視してもよいことを示します。

* #### *MSAT*:
   * ミリサトシのことで、フィールド名としてよく使われます。

* #### *Mutual close*:
   * *[チャネル](#channel)* の協調的なクローズで、各 *ピア* への出力を持つ *[資金調達トランザクション](#funding-transaction)* の無条件な使用をブロードキャストすることで達成されます（一方の出力が小さすぎる場合には、その出力は含められません）。
   * _関連項目を参照: [revoked transaction close](#revoked-transaction-close), [unilateral close](#unilateral-close)_

* #### *Node*:
   * ライトニングネットワークを構成するコンピュータやその他のデバイスです。
   * _関連項目を参照: [peers](#peers)_
   * _派生型を参照: [final node](#final-node), [hop](#hop), [origin node](#origin-node), [processing node](#processing-node), [receiving node](#receiving-node), [sending node](#sending-node)_

* #### *Origin node*:
   * 複数の [hop](#hop) を経由して *[final node](#final-node)* まで支払いをルーティングするパケットを発信する *[ノード](#node)* です。チェーンにおける最初の [sending peer](#sending-peer) でもあります。
   * _カテゴリを参照: [node](#node)_
   * _関連項目を参照: [final node](#final-node), [processing node](#processing-node)_

* #### *Outpoint*:
  * 未使用のトランザクション出力を一意に識別する、トランザクションハッシュと出力インデックスの組です。新しいトランザクションを入力として組み立てる際に必要です。
  * _関連項目を参照: [funding transaction](#funding-transaction), [commitment transaction](#commitment-transaction)_

* #### *Payment hash*:
   * *[HTLC](#HTLC-Hashed-Time-Locked-Contract)* に含まれるペイメントハッシュは、*[payment preimage](#Payment-preimage)* のハッシュです。
   * _コンテナを参照: [HTLC](#HTLC-Hashed-Time-Locked-Contract)_
   * _派生元を参照: [Payment preimage](#Payment-preimage)_

* #### *Payment preimage*:
   * 支払いが受領されたことの証拠で、最終受取人のみが保持する秘密です。最終受取人だけがこの秘密を知っており、資金を解放するためにプリイメージを公開します。ペイメントプリイメージは *[HTLC](#HTLC-Hashed-Time-Locked-Contract)* 内で *[payment hash](#Payment-hash)* としてハッシュ化されます。
   * _コンテナを参照: [HTLC](#HTLC-Hashed-Time-Locked-Contract)_
   * _派生先を参照: [payment hash](#Payment-hash)_

* #### *Peers*:
   * 互いに通信している 2 つの *[ノード](#node)* です。
      * 2 つのピアは、チャネルを設定する前にゴシップを交換することがあります。
      * 2 つのピアは、取引を行うための *[チャネル](#channel)* を確立することがあります。
   * _関連項目を参照: [node](#node)_

* #### *Penalty transaction*:
   * *[取り消されたコミットメントトランザクション](#revoked-commitment-transaction)* のすべての出力を、*commitment revocation private key* を用いて消費するトランザクションです。*[ピア](#peers)* は、相手ピアが *[取り消されたコミットメントトランザクション](#revoked-commitment-transaction)* をブロードキャストして「不正」を試みた場合に、これを使用します。
   * _関連項目を参照: [closing transaction](#closing-transaction), [commitment transaction](#commitment-transaction), [funding transaction](#funding-transaction)_

* #### *Per-commitment secret*:
   * 各 *[コミットメントトランザクション](#commitment-transaction)* は、その鍵を per-commitment secret から導出します。この秘密値は、過去のすべてのコミットメントに対する一連の per-commitment secret をコンパクトに保存できるように生成されます。
   * _コンテナを参照: [commitment transaction](#commitment-transaction)_
   * _派生先を参照: [commitment revocation private key](#commitment-revocation-private-key)_

* #### *Processing node*:
   * *[origin node](#origin-node)* から発信され、支払いをルーティングするために *[final node](#final-node)* に向けて送られているパケットを処理する *[ノード](#node)* です。メッセージを受信する際は *[receiving peer](#receiving-peer)* として、パケットを次に送る際は [sending peer](#sending-peer) として動作します。
   * _カテゴリを参照: [node](#node)_
   * _関連項目を参照: [final node](#final-node), [origin node](#origin-node)_

* #### *Receiving node*:
   * メッセージを受信している *[ノード](#node)* です。
   * _カテゴリを参照: [node](#node)_
   * _関連項目を参照: [sending node](#sending-node)_

* #### *Receiving peer*:
   * 直接接続されている *ピア* からメッセージを受信している *[ノード](#node)* です。
   * _カテゴリを参照: [peer](#Peers)_
   * _関連項目を参照: [sending peer](#sending-peer)_

* #### *Revoked commitment transaction*:
   * 新しいコミットメントトランザクションが交渉されたために取り消された、古い *[コミットメントトランザクション](#commitment-transaction)* です。
   * _カテゴリを参照: [commitment transaction](#commitment-transaction)_

* #### *Revoked transaction close*:
   * *[チャネル](#channel)* の不正なクローズで、*revoked commitment transaction* をブロードキャストすることで行われます。相手 *ピア* は *commitment revocation secret key* を知っているため、*[ペナルティトランザクション](#penalty-transaction)* を作成できます。
   * _関連項目を参照: [mutual close](#mutual-close), [unilateral close](#unilateral-close)_

* #### *Route*:
  * ライトニングネットワーク上の経路で、*origin node* から *[final node](#final-node)* への支払いを 1 つ以上の *[hop](#hop)* を介して可能にします。
  * _関連項目を参照: [channel](#channel)_

* #### *Sending node*:
   * メッセージを送信している *[ノード](#node)* です。
   * _カテゴリを参照: [node](#node)_
   * _関連項目を参照: [receiving node](#receiving-node)_

* #### *Sending peer*:
   * 直接接続されている *ピア* にメッセージを送信している *[ノード](#node)* です。
   * _カテゴリを参照: [peer](#Peers)_
   * _関連項目を参照: [receiving peer](#receiving-peer)_。

* #### *Unilateral close*:
   * *[チャネル](#channel)* の非協調的なクローズで、*[コミットメントトランザクション](#commitment-transaction)* をブロードキャストすることで達成されます。このトランザクションは *[クローズトランザクション](#closing-transaction)* よりもサイズが大きく（つまり効率が劣り）、コミットメントをブロードキャストした *[ピア](#peers)* は、事前に交渉された期間、自分の出力にアクセスできません。
   * _関連項目を参照: [mutual close](#mutual-close), [revoked transaction close](#revoked-transaction-close)_

## テーマソング

      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)

 
      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>

## 著者

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
この作品は [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) のもとでライセンスされています。
