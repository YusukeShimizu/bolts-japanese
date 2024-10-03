# BOLT #12: Lightning Payments のための柔軟なプロトコル

# 目次

  * [BOLT 11 の制限](#limitations-of-bolt-11)
  * [支払いフローのシナリオ](#payment-flow-scenarios)
  * [エンコーディング](#encoding)
  * [署名の計算](#signature-calculation)
  * [オファー](#offers)
  * [請求書リクエスト](#invoice-requests)
  * [請求書](#invoices)
  * [請求書エラー](#invoice-errors)

# BOLT 11 の制限

BOLT 11 の請求書フォーマットは人気がありますが、いくつかの制限があります：

1. bech32 エンコーディングが絡んでいるため、他の形式（例：ライトニングネットワーク内）で送信するのが不便です。
2. 請求書全体に適用される署名により、請求書全体を公開せずに証明することが不可能です。
3. フィールドは一般的に外部で使用するために抽出できません：`h` フィールドは `d` フィールドのみの特別な抽出でした。
4. 「奇数であることは問題ない」というルールの欠如が後方互換性を難しくしています。
5. 金額を分ける「人間が読みやすい」という考えは問題が多いことが判明しました：`p` はしばしば誤って扱われ、ピコビットコインでの金額は現代のサトシベースの計算よりも難しいです。
6. 開発者は bech32 エンコーディングが拡張に問題があると感じたため、いずれにせよ置き換えるか廃棄したいと考えています。
7. 経路内の他のノードによるプロービングを防ぐために設計された `payment_secret` は、請求書が支払者と受取人の間で非公開のままである場合にのみ有用でした。
8. 請求書はユーザごとに発行されなければならず、同じユーザに対して2回の支払い試行が行われると非常に危険です。

# 支払いフローのシナリオ

ここでは、「ユーザ」を個々のユーザのライトニングノードの略、「商人」を何かを販売している、または販売した人のノードの略として使用します。

BOLT 12 でサポートされている基本的な支払いフローは2つあります：

一般的なユーザが商人に支払うフローは次のとおりです：
1. 商人がウェブページや QR コードなどで *オファー* を公開します。
2. 各ユーザが、オファーフィールドを含む *invoice_request* メッセージを使用してライトニングネットワーク上でユニークな *請求書* を要求します。
3. 商人が *請求書* で応答します。
4. ユーザが請求書に示された通りに商人に支払いを行います。


商人がユーザに支払うフロー (例：ATM や返金)：
1. 商人は、ユーザに送金したい金額を含む *invoice_request* を公開します。
2. ユーザは、(一時的な可能性のある) *invoice_node_id* を使用して、*invoice_request* の金額に対する *invoice* を lightning network 経由で送信します。
3. 商人は *invoice_node_id* を確認して、正しい相手に支払おうとしていることを確認し、請求書に対して支払いを行います。

## 支払い証明と支払者証明

通常の lightning の「支払い証明」は、請求書が支払われたことを示すことしかできません (`payment_hash` のプレイメージを示すことで)、誰が支払ったかを示すことはできません。商人は請求書が支払われたと主張でき、いったん明らかになると、誰でもその請求書を支払ったと主張することができます。[1]

*invoice_request* にキーを提供することで、支払者が請求書を要求したのが自分であることを証明できます。さらに、BOLT 12 の請求書署名のマークル構造により、ユーザは紛争が発生した場合に請求書のフィールドを選択的に公開できます。

# エンコーディング

ここで文書化されている各形式は、[TLV](01-messaging.md#type-length-value-format) 形式です。

サポートされている ASCII エンコーディングは、人間が読みやすいプレフィックスの後に `1` が続き、TLV のデータ文字列が順番に bech32 スタイルで続きます。必要に応じて `+` を挿入して追加データが続くことを示すことができます。bech32m とは異なり、チェックサムはありません。

## 要件

bolt12 文字列の作成者：
- 小文字か大文字のいずれかを使用しなければなりません。
- QR コードには大文字を使用することを推奨します。
- それ以外の場合は小文字を使用することを推奨します。
- 大きな bolt12 文字列を分割するために、オプションで空白を伴う `+` を使用してもかまいません。

bolt12 文字列の読み取り者：
- 小文字または大文字の文字列を処理しなければなりません。
- もし `+` が 2 つの bech32 文字の間に 0 個以上の空白文字を伴って現れた場合：
  - `+` と空白を削除しなければなりません。

## 理論的根拠

bech32 の使用は任意ですが、すでにビットコインの世界で存在しています。現在、6 文字の末尾のチェックサムは省略しています。QR コードには独自のチェックサムがあり、エラーは資金の損失ではなく、単に無効なオファー (または解析不能) につながるだけです。

`+` の使用（これは無視されます）は、Twitter のような制限されたテキストフィールドでの使用を可能にします：

```
lno1xxxxxxxx+

yyyyyyyyyyyy+

zzzzz
```

[format-string-test.json](bolt12/format-string-test.json) を参照してください。

# Signature Calculation

すべての署名は [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) に従って作成され、そこで推奨されているようにタグ付けされます。したがって、H(`tag`,`msg`) を SHA256(SHA256(`tag`) || SHA256(`tag`) || `msg`) と定義し、SIG(`tag`,`msg`,`key`) を `key` を使用して H(`tag`,`msg`) の署名とします。

各形式は 1 つ以上の *signature TLV elements*（TLV タイプ 240 から 1000 までを含む）を使用して署名されます。これらの場合、タグは "lightning" || `messagename` || `fieldname` であり、`msg` は Merkle ルートです。"lightning" はリテラルの 9 バイト ASCII 文字列で、`messagename` は署名される TLV ストリームの名前（例： "invoice_request" または "invoice"）であり、`fieldname` は署名を含む TLV フィールド（例： "signature"）です。

Merkle ツリーの構成は [BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki) で提案されているものと似ており、各 TLV リーフは隣接ノードを証明で明らかにしないようにノンスリーフとペアになっています。

Merkle ツリーのリーフは、各 TLV に対して TLV 昇順で次のようになります：
1. H("LnLeaf",tlv)。
2. H("LnNonce"||first-tlv,tlv-type) ここで first-tlv はストリーム内で数値的に最初の TLV エントリであり、tlv-type は現在の TLV の "type" フィールド（1-9 バイト）です。

Merkle ツリーの内部ノードは H("LnBranch", lesser-SHA256||greater-SHA256) です。この順序付けにより、証明がよりコンパクトになります。なぜなら、左/右が本質的に決定されるからです。

リーフが 2 のべき乗でない場合、ツリーの深さは不均一になり、最も低い順序のリーフで最も深くなります。

例：TLV0、TLV1、および TLV2（タイプ 0、1、および 2 それぞれ）を持つ `invoice` `signature` のエンコーディングを考えてみてください：

```
L1=H("LnLeaf",TLV0)
L1nonce=H("LnNonce"||TLV0,0)
L2=H("LnLeaf",TLV1)
L2nonce=H("LnNonce"||TLV0,1)
L3=H("LnLeaf",TLV2)
L3nonce=H("LnNonce"||TLV0,2)

Assume L1 < L1nonce, L2 > L2nonce and L3 > L3nonce.

   L1    L1nonce                      L2   L2nonce                L3   L3nonce
     \   /                             \   /                       \   /
      v v                               v v                         v v
L1A=H("LnBranch",L1||L1nonce) L2A=H("LnBranch",L2nonce||L2)  L3A=H("LnBranch",L3nonce||L3)
                 
Assume L1A < L2A:

       L1A   L2A                                 L3A=H("LnBranch",L3nonce||L3)
         \   /                                    |
          v v                                     v
  L1A2A=H("LnBranch",L1A||L2A)                   L3A=H("LnBranch",L3nonce||L3)
  
Assume L1A2A > L3A:

  L1A2A=H("LnBranch",L1A||L2A)          L3A
                          \            /
                           v          v
                Root=H("LnBranch",L3A||L1A2A)

Signature = SIG("lightninginvoicesignature", Root, nodekey)
```

# Offers

オファーは invoice_request の前段階です。読者はオファーに基づいて請求書（または複数）を要求します。オファーは特定の請求書よりも長期間存在する可能性があるため、いくつかの異なる特性を持っています。特に、金額は非ライトニング通貨である場合があります。また、QR コードに簡単に収まるようにコンパクトさを重視して設計されています。

署名以外の TLV 要素は、`invoice_request` と `invoice` メッセージに反映されるため、それぞれに特定の異なる TLV 範囲があります。

オファーの人間が読めるプレフィックスは `lno` です。

## オファーの TLV フィールド

1. `tlv_stream`: `offer`
2. タイプ:
    1. タイプ: 2 (`offer_chains`)
    2. データ:
        * [`...*chain_hash`:`chains`]
    1. タイプ: 4 (`offer_metadata`)
    2. データ:
        * [`...*byte`:`data`]
    1. タイプ: 6 (`offer_currency`)
    2. データ:
        * [`...*utf8`:`iso4217`]
    1. タイプ: 8 (`offer_amount`)
    2. データ:
        * [`tu64`:`amount`]
    1. タイプ: 10 (`offer_description`)
    2. データ:
        * [`...*utf8`:`description`]
    1. タイプ: 12 (`offer_features`)
    2. データ:
        * [`...*byte`:`features`]
    1. タイプ: 14 (`offer_absolute_expiry`)
    2. データ:
        * [`tu64`:`seconds_from_epoch`]
    1. タイプ: 16 (`offer_paths`)
    2. データ:
        * [`...*blinded_path`:`paths`]
    1. タイプ: 18 (`offer_issuer`)
    2. データ:
        * [`...*utf8`:`issuer`]
    1. タイプ: 20 (`offer_quantity_max`)
    2. データ:
        * [`tu64`:`max`]
    1. タイプ: 22 (`offer_issuer_id`)
    2. データ:
        * [`point`:`id`]

## オファーの要件

オファーの作成者は：
  - 包括的な範囲外の TLV フィールドを設定してはなりません：1 から 79 および 1000000000 から 1999999999。
  - インボイスのチェーンがビットコインのみでない場合：
    - オファーが有効な `offer_chains` を指定しなければなりません。
  - それ以外の場合：
    - ビットコインのみがチェーンであることを示唆する `offer_chains` を省略するべきです。
  - 支払い成功のために特定の最小 `offer_amount` が必要な場合：
    - 期待される金額（アイテムごと）に `offer_amount` を設定しなければなりません。
    - `offer_amount` の通貨が `chains` のすべてのエントリの通貨である場合：
      - 最小のライトニング支払い可能単位の倍数で `offer_amount` を指定しなければなりません
        （例：ビットコインの場合はミリサトシ）。
    - それ以外の場合：
      - ISO 4217 の 3 文字コードとして `offer_currency` `iso4217` を指定しなければなりません。
      - ISO 4217 の指数で調整された通貨単位で `offer_amount` を指定しなければなりません
        （例：USD セント）。
    - 支払いの目的を完全に説明する `offer_description` を設定しなければなりません。
  - それ以外の場合：
    - `offer_amount` を設定してはなりません
    - `offer_currency` を設定してはなりません
    - `offer_description` を設定してもかまいません
  - 自分用に `offer_metadata` を設定してもかまいません。
  - bolt12 オファー機能をサポートしている場合：
    - `offer_features`.`features` を bolt12 機能のビットマップに設定しなければなりません。
  - オファーが期限切れになる場合：
    - `offer_absolute_expiry` `seconds_from_epoch` を 1970 年 1 月 1 日午前 0 時 UTC からの秒数に設定しなければなりません。
      その後、`invoice_request` を試みてはなりません。
  - プライベートチャネルのみで接続されている場合：
    - 公開到達可能なノードからノードへの 1 つ以上のパスを含む `offer_paths` を含めなければなりません。
  - それ以外の場合：
    - `offer_paths` を含めてもかまいません。
  - `offer_paths` を含む場合：
    - `offer_issuer_id` を設定してもかまいません。
  - それ以外の場合：
    - インボイスを要求するノードの公開鍵に `offer_issuer_id` を設定しなければなりません。
  - `offer_issuer` を設定する場合：
    - インボイスの発行者を明確に識別するために設定するべきです。
    - ドメイン名を含む場合：
      - user@domain または domain のいずれかで始めるべきです
      - スペースと追加のテキストを続けてもかまいません
  - 単一のインボイスで複数のアイテムを供給できる場合：
    - 最大数量がわかっている場合：
      - その最大値を `offer_quantity_max` に設定しなければなりません。
      - `offer_quantity_max` を 0 に設定してはなりません。
    - それ以外の場合：
      - `offer_quantity_max` を 0 に設定しなければなりません。
  - それ以外の場合：
    - `offer_quantity_max` を設定してはなりません。

オファーのリーダー：

- オファーに含まれる TLV フィールドが、1 から 79 および 1000000000 から 1999999999 の範囲外の場合：
  - オファーに応答してはいけません。
- `offer_features` に未知の _奇数_ ビットが非ゼロで含まれている場合：
  - ビットを無視しなければなりません。
- `offer_features` に未知の _偶数_ ビットが非ゼロで含まれている場合：
  - オファーに応答してはいけません。
  - 未知のビットをユーザに示すべきです。
- `offer_chains` が設定されていない場合：
  - ノードがビットコインの請求書を受け入れない場合：
    - オファーに応答してはいけません。
- それ以外の場合：（`offer_chains` が設定されている場合）：
  - ノードが `chains` のいずれの請求書も受け入れない場合：
    - オファーに応答してはいけません。
- `offer_amount` が設定されていて `offer_description` が設定されていない場合：
  - オファーに応答してはいけません。
- `offer_currency` が設定されていて `offer_amount` が設定されていない場合：
  - オファーに応答してはいけません。
- `offer_issuer_id` と `offer_paths` のどちらも設定されていない場合：
  - オファーに応答してはいけません。
- `offer_paths` の `blinded_path` 内の `num_hops` が 0 の場合：
  - オファーに応答してはいけません。
- `offer_amount` を使用してユーザにコスト見積もりを提供する場合：
  - `offer_amount` の通貨単位を考慮しなければなりません：
    - `offer_currency` フィールドが設定されている場合
    - それ以外の場合、最小のライトニング支払い可能単位（例：ビットコインの場合はミリサトシ）。
  - 受け取った `invoice_amount` がその見積もりと大きく異なる場合、ユーザに警告しなければなりません。
- 現在の時刻が `offer_absolute_expiry` を過ぎている場合：
  - オファーに応答してはいけません。
- 請求書リクエストを送信することを選択した場合、オニオンメッセージを送信します：
  - `offer_paths` が設定されている場合：
    - そのパスの最終 `onion_msg_hop`.`blinded_node_id` に `offer_paths` 内の任意のパスを通じてオニオンメッセージを送信しなければなりません。
  - それ以外の場合：
    - `offer_issuer_id` にオニオンメッセージを送信しなければなりません。
  - 複数の請求書リクエストオニオンメッセージを同時に送信してもかまいません。

## 理論的根拠

オファー全体は請求書リクエストに反映されます。これは完全性のため（すべての情報が請求書に返されるようにするため）であり、オファーノードがステートレスであることを可能にするためです。これにより、`offer_metadata` は特に有用です。なぜなら、他のフィールドを検証するための認証クッキーを含むことができるからです。

オファーフィールドは請求書リクエスト（そして請求書）にコピーされるため、異なる範囲が必要です。範囲 1-79 は通常の範囲であり、もう 10 億は自己割り当ての実験的範囲です。

署名は不要であり、文字列を長くするだけです（低性能カメラでの QR コード使用を制限する可能性があります）。オファーにエラーがある場合、リクエストにはすべての非署名フィールドが含まれているため、請求書は発行されません。

`offer_paths` が設定されている場合、`offer_issuer_id` は省略可能です。なぜなら、パス内の各最終 `blinded_node_id` が宛先の有効な公開鍵として機能できるからです。

`offer_amount` は異なる通貨で指定できるため（`offer_currency` フィールドを使用）、単なるガイドです。発行者は請求書を生成する際に `invoice_amount` のミリサトシ数に変換します。または、請求書リクエストで `invreq_amount` に正確な金額を指定できますが、その場合、発行者が同意しない場合は拒否されることがあります。

`offer_quantity_max` は 1 に設定することが許可されています。これは無意味に思えますが、在庫に基づくシステムでは有用です。「残り 1 つだけ」のオファー生成を特別扱いするのは面倒です。

オファーは、見返りを期待せずに単にお金を送るために使用できます（チップ、称賛、寄付など）。この場合、説明フィールドはオプションです（`offer_issuer` フィールドはこのケースで非常に有用です！）。特定のものに対して料金を請求する場合、ユーザが何に対して支払ったのかを知るために説明は重要です。

# 請求書リクエスト

請求書リクエストは請求書のリクエストです。請求書リクエストの人間が読めるプレフィックスは `lnr` です。

請求書リクエストには、ワークフローの観点からはほぼ同じですが、ユーザの視点からはかなり異なる、2 つの似た用途があります。

1 つはオファーへの応答です。これには `offer_issuer_id` または `offer_paths` と他のすべてのオファー詳細が含まれ、通常はオニオンメッセージで受信されます。有効で既知のオファーを参照している場合、応答は通常、オニオンメッセージの `reply_path` フィールドを使用して `invoice` を返信することです。

第 2 のケースは、オファーなしで請求書リクエストを公開する場合です。例えば QR コードを介して行います。この場合、`offer_issuer_id` や `offer_paths` は含まれず、代わりに `invreq_payer_id`（おそらく `invreq_paths` も）を設定します。これは支払う側であるためです。他のオファーフィールドは `invoice_request` の作成者によって埋められ、一種の送金オファーを形成します。

注：`invreq_metadata` は 0 番号であり（他の invreq フィールドの 80-159 範囲には含まれません）、[署名計算](#signature-calculation) のための「数値的に最初の TLV エントリ」となります。これにより、マークルリーフが推測不可能になり、将来的なコンパクトな表現がフィールドを隠しつつも署名検証を可能にします。

## `invoice_request` の TLV フィールド

1. `tlv_stream`: `invoice_request`
2. types:
    1. type: 0 (`invreq_metadata`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 22 (`offer_issuer_id`)
    2. data:
        * [`point`:`id`]
    1. type: 80 (`invreq_chain`)
    2. data:
        * [`chain_hash`:`chain`]
    1. type: 82 (`invreq_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 84 (`invreq_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 86 (`invreq_quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 88 (`invreq_payer_id`)
    2. data:
        * [`point`:`key`]
    1. type: 89 (`invreq_payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 90 (`invreq_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]

## インボイスリクエストの要件

ライターは以下のようにします：

- オファーに応答する場合：
  - オファーからすべてのフィールドをコピーしなければなりません (未知のフィールドを含む)。
  - `offer_chains` が設定されている場合：
    - `invreq_chain` を `offer_chains` のいずれかに設定しなければなりません。ただし、そのチェーンがビットコインの場合は `invreq_chain` を省略するべきです。
  - それ以外の場合：
    - `invreq_chain` を設定する場合は、ビットコインに設定しなければなりません。
  - `signature`.`sig` を [Signature Calculation](#signature-calculation) に従って `invreq_payer_id` を使用して設定しなければなりません。
  - `offer_amount` が存在しない場合：
    - `invreq_amount` を指定しなければなりません。
  - それ以外の場合：
    - `invreq_amount` を省略してもかまいません。
    - `invreq_amount` を設定する場合：
      - `invreq_amount`.`msat` を `offer_amount` (および、存在する場合は `offer_currency` と `invreq_quantity`) によって期待される金額以上に指定しなければなりません。
  - `invreq_payer_id` を一時的な公開鍵に設定しなければなりません。
  - `invreq_payer_id` に対応する秘密鍵を記憶しなければなりません。
  - `offer_quantity_max` が存在する場合：
    - `invreq_quantity` をゼロより大きく設定しなければなりません。
    - `offer_quantity_max` がゼロでない場合：
      - `invreq_quantity` を `offer_quantity_max` 以下に設定しなければなりません。
  - それ以外の場合：
    - `invreq_quantity` を設定してはなりません。

- それ以外の場合 (オファーに応答しない場合)：
  - 支払いの目的を完全に説明する `offer_description` を設定しなければなりません。
  - `offer_absolute_expiry` と `offer_issuer` をオファーの場合と同様に設定 (または設定しない) しなければなりません。
  - `invreq_payer_id` を設定しなければなりません (オファーの場合の `offer_issuer_id` と同様に)。
  - `invreq_paths` をオファーの場合の `offer_paths` と同様に設定 (または設定しない) しなければなりません。
  - `signature`、`offer_metadata`、`offer_chains`、`offer_amount`、`offer_currency`、`offer_features`、`offer_quantity_max`、`offer_paths`、または `offer_issuer_id` を含めてはなりません。
  - インボイスのチェーンがビットコインのみでない場合：
    - オファーが有効な `invreq_chain` を指定しなければなりません。
  - `invreq_amount` を設定しなければなりません。

- 署名以外の TLV フィールドを 0 から 159 および 1000000000 から 2999999999 の範囲外に設定してはなりません。
- `invreq_metadata` を予測不可能なバイト列に設定しなければなりません。
- `invreq_amount` を設定する場合：
  - `msat` を `invreq_chain` (または `invreq_chain` がない場合はビットコイン) の最小ライトニング支払い単位 (例：ビットコインの場合はミリサトシ) の倍数で設定しなければなりません。
- bolt12 インボイスリクエスト機能をサポートする場合：
  - `invreq_features`.`features` を機能のビットマップに設定しなければなりません。

読者は以下の条件を満たす必要があります：

- `invreq_payer_id` または `invreq_metadata` が存在しない場合、請求書リクエストを拒否しなければなりません。
- 署名以外の TLV フィールドが 0 から 159 および 1000000000 から 2999999999 の範囲外にある場合、請求書リクエストを拒否しなければなりません。
- `invreq_features` に未知の _奇数_ ビットが非ゼロで含まれている場合：
  - ビットを無視しなければなりません。
- `invreq_features` に未知の _偶数_ ビットが非ゼロで含まれている場合：
  - 請求書リクエストを拒否しなければなりません。
- `signature` が `invreq_payer_id` を使用した [Signature Calculation](#signature-calculation) に記載された通りに正しくない場合、請求書リクエストを拒否しなければなりません。
- `invreq_paths` の任意の `blinded_path` で `num_hops` が 0 の場合：
  - 請求書リクエストを拒否しなければなりません。
- `offer_issuer_id` が存在し、`invreq_metadata` が以前の `invoice_request` と同一である場合：
  - 前の請求書で単に返信してもよいです。
- それ以外の場合：
  - 前の請求書で返信してはなりません。
- `offer_issuer_id` または `offer_paths` が存在する場合（オファーへの応答）：
  - オファーフィールドが有効で未期限切れのオファーと完全に一致しない場合、請求書リクエストを拒否しなければなりません。
  - `offer_paths` が存在する場合：
    - それらのパスのいずれかを経由して到着しなかった場合、請求書リクエストを無視しなければなりません。
  - それ以外の場合：
    - ブラインドパスを経由して到着した請求書リクエストを無視しなければなりません。
  - `offer_quantity_max` が存在する場合：
    - `invreq_quantity` フィールドがない場合、請求書リクエストを拒否しなければなりません。
    - `offer_quantity_max` が非ゼロの場合：
      - `invreq_quantity` がゼロまたは `offer_quantity_max` を超える場合、請求書リクエストを拒否しなければなりません。
  - それ以外の場合：
    - `invreq_quantity` フィールドがある場合、請求書リクエストを拒否しなければなりません。
  - `offer_amount` が存在する場合：
    - `offer_amount` を使用して *期待される金額* を計算しなければなりません：
      - `offer_currency` が `invreq_chain` 通貨でない場合、`invreq_chain` 通貨に変換します。
      - `invreq_quantity` が存在する場合、`invreq_quantity`.`quantity` で乗算します。
    - `invreq_amount` が存在する場合：
      - `invreq_amount`.`msat` が *期待される金額* より少ない場合、請求書リクエストを拒否しなければなりません。
      - `invreq_amount`.`msat` が *期待される金額* を大幅に超える場合、請求書リクエストを拒否してもよいです。
  - それ以外の場合（`offer_amount` がない場合）：
    - `invreq_amount` を含まない場合、請求書リクエストを拒否しなければなりません。
  - `onionmsg_tlv` `reply_path` を使用して請求書を送信することを推奨します。
- それ以外の場合（`offer_issuer_id` または `offer_paths` がない場合、私たちのオファーへの応答ではない場合）：
  - 以下のいずれかが存在する場合、請求書リクエストを拒否しなければなりません：
    - `offer_chains`、`offer_features` または `offer_quantity_max`。
  - `invreq_amount` が存在しない場合、請求書リクエストを拒否しなければなりません。
  - `offer_amount`（または `offer_currency`）をユーザへの情報表示に使用してもよいです。
  - 応答として請求書を送信する場合：
    - `invreq_paths` が存在する場合はそれを使用し、存在しない場合は `invreq_payer_id` をノード ID として送信しなければなりません。
- `invreq_chain` が存在しない場合：
  - ビットコインがサポートされているチェーンでない場合、請求書リクエストを拒否しなければなりません。
- それ以外の場合：
  - `invreq_chain`.`chain` がサポートされているチェーンでない場合、請求書リクエストを拒否しなければなりません。

## Rationale

`invreq_metadata` には通常、`invreq_payer_id` の導出に関する情報が含まれます。これは情報を漏らさないようにするべきです（例えば、単純な BIP-32 導出パスを使用しない）。有効なシステムとしては、ノードがベースの支払者キーを保持し、ここに 128 ビットの調整をエンコードすることが考えられます。支払者 ID は、SHA256(payer_base_pubkey || tweak) でベースキーを調整することによって導出されます。また、これは最初のエントリ（存在する場合）であり、ハッシュのための予測不可能なノンスを保証します。

`invreq_payer_note` は、請求書に対して褒め言葉や挑発、その他の落書きを刻むことができます。

ユーザは、請求書リクエストで `invreq_amount` を指定することで、チップを渡したり（または送信額を隠したり）することができます。オファーが `offer_amount` を指定している場合でもです。受取人は、請求書リクエストの金額が期待している金額（すなわち、通貨換算後の `offer_amount` に `invreq_quantity` を掛けたもの）を超える場合にのみこれを受け入れます。

現在、オファーに対する応答ではない請求書リクエストには、チェーン通貨で `invreq_amount` を明示的に記載する必要があります。そのため、`offer_amount` と `offer_currency` は冗長ですが、送信者が `invreq_amount` をどのように導出したかを支払者が知るための情報として役立つかもしれません。

`offer_paths` が存在する場合にそれを使用する必要があるという要件は、ノードが直接尋ねられた場合にオファーの発信元であることを明かさないようにするためです。同様に、オファーに対して正しいパスを使用する必要があるという要件は、他のオファーを作成した同じノードであることを明かさないようにするためです。

# Invoices

請求書は支払いリクエストであり、支払いが行われると、支払いのプレイメージを請求書と組み合わせて暗号学的な領収書を形成できます。

受取人は `invoice_request` に応じて `onion_message` の `invoice` フィールドを使用して `invoice` を送信します。

1. `tlv_stream`: `invoice`
2. types:
    1. type: 0 (`invreq_metadata`)
    2. data:
        * [`...*byte`:`blob`]
    1. type: 2 (`offer_chains`)
    2. data:
        * [`...*chain_hash`:`chains`]
    1. type: 4 (`offer_metadata`)
    2. data:
        * [`...*byte`:`data`]
    1. type: 6 (`offer_currency`)
    2. data:
        * [`...*utf8`:`iso4217`]
    1. type: 8 (`offer_amount`)
    2. data:
        * [`tu64`:`amount`]
    1. type: 10 (`offer_description`)
    2. data:
        * [`...*utf8`:`description`]
    1. type: 12 (`offer_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 14 (`offer_absolute_expiry`)
    2. data:
        * [`tu64`:`seconds_from_epoch`]
    1. type: 16 (`offer_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 18 (`offer_issuer`)
    2. data:
        * [`...*utf8`:`issuer`]
    1. type: 20 (`offer_quantity_max`)
    2. data:
        * [`tu64`:`max`]
    1. type: 22 (`offer_issuer_id`)
    2. data:
        * [`point`:`id`]
    1. type: 80 (`invreq_chain`)
    2. data:
        * [`chain_hash`:`chain`]
    1. type: 82 (`invreq_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 84 (`invreq_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 86 (`invreq_quantity`)
    2. data:
        * [`tu64`:`quantity`]
    1. type: 88 (`invreq_payer_id`)
    2. data:
        * [`point`:`key`]
    1. type: 89 (`invreq_payer_note`)
    2. data:
        * [`...*utf8`:`note`]
    1. type: 90 (`invreq_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 160 (`invoice_paths`)
    2. data:
        * [`...*blinded_path`:`paths`]
    1. type: 162 (`invoice_blindedpay`)
    2. data:
        * [`...*blinded_payinfo`:`payinfo`]
    1. type: 164 (`invoice_created_at`)
    2. data:
        * [`tu64`:`timestamp`]
    1. type: 166 (`invoice_relative_expiry`)
    2. data:
        * [`tu32`:`seconds_from_creation`]
    1. type: 168 (`invoice_payment_hash`)
    2. data:
        * [`sha256`:`payment_hash`]
    1. type: 170 (`invoice_amount`)
    2. data:
        * [`tu64`:`msat`]
    1. type: 172 (`invoice_fallbacks`)
    2. data:
        * [`...*fallback_address`:`fallbacks`]
    1. type: 174 (`invoice_features`)
    2. data:
        * [`...*byte`:`features`]
    1. type: 176 (`invoice_node_id`)
    2. data:
        * [`point`:`node_id`]
    1. type: 240 (`signature`)
    2. data:
        * [`bip340sig`:`sig`]


1. subtype: `blinded_payinfo`
2. data:
   * [`u32`:`fee_base_msat`]
   * [`u32`:`fee_proportional_millionths`]
   * [`u16`:`cltv_expiry_delta`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u64`:`htlc_maximum_msat`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]

1. subtype: `fallback_address`
2. data:
   * [`byte`:`version`]
   * [`u16`:`len`]
   * [`len*byte`:`address`]

## Invoice Features

| Bits | Description                      | Name           |
|------|----------------------------------|----------------|
| 16   | Multi-part-payment support       | MPP/compulsory |
| 17   | Multi-part-payment support       | MPP/optional   |

「MPP サポート」請求書機能は、支払者が請求書を支払うために複数の部分支払いを使用しなければならない (16) または使用してもよい (17) ことを示します。

一部の実装では MPP をサポートしない場合があります (例えば、小額の支払いの場合) 。または、単一のチャネルの容量制限により必要とされる場合があります。

## Requirements

請求書の作成者は：
  - 請求書が作成されたときの 1970 年 1 月 1 日午前 0 時 (UTC) からの秒数を `invoice_created_at` に設定しなければなりません。
  - `invoice_amount` を、`invreq_chain` に対して受け入れる最小の lightning 支払い単位 (例えば、ビットコインのミリサトシ) での最小金額に設定しなければなりません。
  - 請求書が `invoice_request` に対する応答である場合：
    - 請求書リクエストからすべての非署名フィールド (未知のフィールドを含む) をコピーしなければなりません。
    - `invreq_amount` が存在する場合：
      - `invoice_amount` を `invreq_amount` に設定しなければなりません。
    - そうでない場合：
      - `invoice_amount` を *期待される金額* に設定しなければなりません。
  - 支払いに対して返される `payment_preimage` の SHA256 ハッシュを `invoice_payment_hash` に設定しなければなりません。
  - `offer_issuer_id` が存在する場合：
    - `invoice_node_id` を `offer_issuer_id` に設定しなければなりません。
  - そうでない場合、`offer_paths` が存在する場合：
    - 請求書リクエストを受け取った経路の最終 `blinded_node_id` に `invoice_node_id` を設定しなければなりません。
  - 正確に 1 つの署名 TLV 要素：`signature` を指定しなければなりません。
    - [Signature Calculation](#signature-calculation) に記載されているように `invoice_node_id` を使用して署名を `sig` に設定しなければなりません。
  - 請求書を支払うために複数の部分が必要な場合：
    - `invoice_features`.`features` ビット `MPP/compulsory` を設定しなければなりません。
  - または、請求書を支払うために複数の部分を許可する場合：
    - `invoice_features`.`features` ビット `MPP/optional` を設定しなければなりません。
  - 支払いを受け入れる期限が `invoice_created_at` から 7200 秒後でない場合：
    - `invoice_relative_expiry`.`seconds_from_creation` を、請求書の支払いが試みられるべきでない `invoice_created_at` からの秒数に設定しなければなりません。
  - オンチェーン支払いを受け入れる場合：
    - `invoice_fallbacks` を指定してもよいです。
    - 優先順位がある場合、最も優先されるものから最も優先されないものの順に `invoice_fallbacks` を指定するべきです。
    - ビットコインチェーンの場合、各 `fallback_address` を有効なウィットネスバージョンとして `version` に設定し、有効なウィットネスプログラムとして `address` に設定しなければなりません。
  - ノードへの 1 つ以上の経路を含む `invoice_paths` を含めなければなりません。
    - 優先順位がある場合、最も優先されるものから最も優先されないものの順に `invoice_paths` を指定しなければなりません。
    - 各 `paths` の `blinded_path` に対して正確に 1 つの `blinded_payinfo` を含む `invoice_blindedpay` を含めなければなりません。
    - 各 `blinded_payinfo` の `features` を `encrypted_data_tlv`.`allowed_features` に一致させる (または `allowed_features` がない場合は空にする) ように設定しなければなりません。
    - 経路のいずれかを使用しない支払いを無視するべきです。

請求書のリーダー：

- `invoice_amount` が存在しない場合、請求書を拒否しなければなりません。
- `invoice_created_at` が存在しない場合、請求書を拒否しなければなりません。
- `invoice_payment_hash` が存在しない場合、請求書を拒否しなければなりません。
- `invoice_node_id` が存在しない場合、請求書を拒否しなければなりません。
- `invreq_chain` が存在しない場合：
  - ビットコインがサポートされているチェーンでない場合、請求書を拒否しなければなりません。
- それ以外の場合：
  - `invreq_chain`.`chain` がサポートされているチェーンでない場合、請求書を拒否しなければなりません。
- `invoice_features` に未知の _奇数_ ビットが非ゼロで含まれている場合：
  - ビットを無視しなければなりません。
- `invoice_features` に未知の _偶数_ ビットが非ゼロで含まれている場合：
  - 請求書を拒否しなければなりません。
- `invoice_relative_expiry` が存在する場合：
  - 現在の時刻が 1970-01-01 UTC から `invoice_created_at` に `seconds_from_creation` を加えた時間を超えている場合、請求書を拒否しなければなりません。
- それ以外の場合：
  - 現在の時刻が 1970-01-01 UTC から `invoice_created_at` に 7200 を加えた時間を超えている場合、請求書を拒否しなければなりません。
- `invoice_paths` が存在しないか空の場合、請求書を拒否しなければなりません。
- `invoice_paths` の任意の `blinded_path` で `num_hops` が 0 の場合、請求書を拒否しなければなりません。
- `invoice_blindedpay` が存在しない場合、請求書を拒否しなければなりません。
- `invoice_blindedpay` が `invoice_paths`.`blinded_path` ごとに正確に 1 つの `blinded_payinfo` を含まない場合、請求書を拒否しなければなりません。
- 各 `invoice_blindedpay`.`payinfo` に対して：
  - `payinfo`.`features` に未知の偶数ビットが設定されている場合、対応する `invoice_paths`.`path` を使用してはなりません。
  - これにより使用可能なパスがなくなる場合、請求書を拒否しなければなりません。
- 請求書が `invoice_request` に対する応答である場合：
  - 範囲 0 から 159 および 1000000000 から 2999999999（含む）のすべてのフィールドが請求書リクエストと正確に一致しない場合、請求書を拒否しなければなりません。
  - `offer_issuer_id` が存在する場合（オファーのための invoice_request）：
    - `invoice_node_id` が `offer_issuer_id` と等しくない場合、請求書を拒否しなければなりません。
  - それ以外の場合、`offer_paths` が存在する場合（ID のないオファーのための invoice_request）：
    - `invoice_node_id` が請求書リクエストを送信した最終 `blinded_node_id` と等しくない場合、請求書を拒否しなければなりません。
  - それ以外の場合（オファーのない invoice_request）：
    - `invoice_node_id` が正しいことをバンド外で確認できない場合、請求書を拒否してもかまいません。
- `signature` が [Signature Calculation](#signature-calculation) で説明されているように `invoice_node_id` を使用した有効な署名でない場合、請求書を拒否しなければなりません。
- 他に優先する理由がない場合、より早い `invoice_paths` を後のものより優先して使用するべきです。
- `invoice_features` に MPP/必須ビットが含まれている場合：
  - 複数の別々のブラインドパスを通じて請求書を支払わなければなりません。
- それ以外の場合、`invoice_features` に MPP/オプションビットが含まれている場合：
  - 複数の別々の支払いを通じて請求書を支払ってもかまいません。
- それ以外の場合：
  - 複数の部分を使用して請求書を支払ってはなりません。
- `invreq_amount` が存在する場合：
  - `invoice_amount` が `invreq_amount` と等しくない場合、請求書を拒否しなければなりません。
- それ以外の場合：
  - `invoice_amount`.`msat` が承認された金額範囲内にない場合、承認を確認するべきです。
- ビットコインチェーンの場合、請求書が `invoice_fallbacks` を指定している場合：
  - `version` が 16 を超える `fallback_address` を無視しなければなりません。
  - `address` が 2 未満または 40 バイトを超える `fallback_address` を無視しなければなりません。
  - 指定された `version` に対する既知の要件を満たさない `fallback_address` を無視しなければなりません。
- `invreq_paths` が存在する場合：
  - それらのパスのいずれかを経由して到着しなかった場合、請求書を拒否しなければなりません。
- それ以外の場合、`offer_issuer_id` も `offer_paths` も存在しない（オファーから派生していない）場合：
  - ブラインドパスを経由して到着した場合、請求書を拒否しなければなりません。
- それ以外の場合（オファーから派生している）：
  - 請求書リクエスト `onionmsg_tlv` `reply_path` を経由して到着しなかった場合、請求書を拒否しなければなりません。

## 理論的根拠

メッセージング層が信頼できないため、同じオファーに対する複数のリクエストを受け取ることが十分にあり得ます。`invreq_metadata` を予測不可能かつユニークにする責任は呼び出し側にあるため、書き手はすべてのフィールドが重複しているかどうかを確認せずに、単に以前のインボイスを返すことができます。このようなキャッシングは任意であり、例えば通貨変換が関与している場合やインボイスが期限切れの場合には慎重に制限する必要があります。

インボイスは、以前の invreq にコミットするのではなく、フィールドを重複させます。この平坦化された形式は、多少のスペースコストを伴いますが、支払者は返金や証明のためにインボイスだけを覚えておけばよいため、ストレージが簡素化されます。

インボイスの読み手は、インボイスが invreq フィールドを正しく反映していることを信頼できないため、それらが正しいことを確認する必要がありますが、単に未請求のインボイスを直接送信することも許可されています。

インボイスの受取人は、受け取ったオファーまたは送信した invreq から期待される金額を決定できるため、しばしば期待される金額の承認をすでに持っています。

デフォルトの `invoice_relative_expiry` は 7200 秒であり、新しいチャネルを開く必要がある場合でも、一般的に支払いに十分な時間です。

ブラインドパスは、BOLT 11 で使用される `payment_secret` と `payment_metadata` に相当するものを提供します。`invoice_node_id` や `invreq_payer_id` が公開されている場合でも、これらの機能を保持するためにブラインドパスの使用を強制します。受取人がブラインドパスによって提供される追加のプライバシーを気にしない場合、彼らは自分自身だけを含む長さ 1 のパスを作成できます。

ブラインドパス内の各ホップに対する詳細な per-hop-payinfo を提供する代わりに、手数料と CLTV デルタを集約します。これにより、パスを区別する可能性のある顕著な非均一性を明らかにすることを避けます。

オファーがない（単なるインボイスリクエスト）場合のインボイスでは、支払者はインボイスが意図した支払い受取人からのものであることを確認する必要があります。これが、この場合に `invoice_node_id` を確認することを提案する根拠です。


インボイスリクエストに基づかない生のインボイスは一般的にサポートされていませんが、実装によってはサポートすることが許可されており、将来的にその動作を定義する可能性があります。`invreq_chain` を明示的にチェックする冗長な要件はこれを考慮したものです。インボイスがインボイスリクエストへの応答である場合、そのフィールドはインボイスリクエストの要件により存在していたはずであり、ここでもそれを反映することを要求しています。

# インボイスエラー

情報を提供するエラーは、`invoice_request` または `invoice` に対して、オニオンメッセージの `invoice_error` フィールド（オニオン `reply_path` 経由）で返すことができます。

## `invoice_error` の TLV フィールド

1. `tlv_stream`: `invoice_error`
2. タイプ:
    1. タイプ: 1 (`erroneous_field`)
    2. データ:
        * [`tu64`:`tlv_fieldnum`]
    1. タイプ: 3 (`suggested_value`)
    2. データ:
        * [`...*byte`:`value`]
    1. タイプ: 5 (`error`)
    2. データ:
        * [`...*utf8`:`msg`]

## 要件

インボイスエラーの作成者:
  - `error` を説明的な文字列に設定しなければなりません。
  - 問題のあった `invoice` または `invoice_request` の特定のフィールド番号に `erroneous_field` を設定してもかまいません。
  - `erroneous_field` を設定する場合:
    - `suggested_value` を設定してもかまいません。
    - `suggested_value` を設定する場合:
      - その `tlv_fieldnum` に対して有効なフィールドに `suggested_value` を設定しなければなりません。
  - それ以外の場合:
    - `suggested_value` を設定してはなりません。

インボイスエラーの読み手:
   FIXME!

## 根拠

通常、エラーメッセージは診断に十分ですが、将来的な拡張により自動処理が有用になるかもしれません。

特に、非オファー応答の `invoice_request` が将来的に `invreq_amount` を省略し、オファーフィールドを使用して代替通貨を示すことを許可することができます。（「10セントを送ります！」）その場合、インボイスの送信者はそれが何ミリサトシかを推測しなければならず、受取人が変換に同意しなかった場合に `invoice_error` を使用して示し、送信者が新しいインボイスを送信できるようにします。

# FIXME: 将来の可能な拡張:

1. オファーは `invoice_request` に配達情報を要求できます。
2. オファーは更新可能です：`invoice_request` への応答は別のオファーであり、元の `offer_issuer_id` からの署名を含むかもしれません。
3. 空の TLV フィールドは、他の手段（つまり、トランスポート固有）で知られているはずの値を意味することができますが、署名のためにハッシュされます。
4. 一つの invreq とインボイスで複数のオファーを許可するようにアップグレードし、買い物リストを作成できます。
7. 全ゼロの offer_id は無償の支払いを意味します。
8. ストリーミングインボイス？
9. 繰り返しを再追加。
10. 証明をサポートするために `invreq_refund_for` を再追加。
11. （支払いが滞っている）インボイスを新しいものに置き換えるための `invoice_replace` を再追加。
12. 代替通貨を使用した非オファー `invoice_request` を許可？
13. 数量のステップを示す `offer_quantity_unit` を追加（例：100グラム）。

Please provide the Markdown file that you would like me to translate.
