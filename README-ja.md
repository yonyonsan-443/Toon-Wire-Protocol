# 透音プロトコル（Tôon Wire Protocol） v0.1

## 概要

Tôon Wire Protocolは、近距離に存在する既知のユーザー同士が、低電力無線通信（主にLoRa）を用いてメッセージを交換するための軽量P2P通信プロトコルである。
本プロトコルではインターネット全体のルーティングや名前解決を対象とせず、ローカル環境における直接通信のみを対象とする。

## Status of This Memo

この文書は、Tôon Wire Protocol v0.1 の初期草案である。

本仕様は正式な標準規格ではなく、実装の互換性や安全性を保証するものではない。  
現在の内容は設計途中であり、今後の検討、実装、レビューによって変更される可能性がある。

本プロトコルの設計に関する指摘、改善案、実装上の問題点、セキュリティ上の懸念などは歓迎される。  
Issue や Pull Request を通じて、より良い仕様へ改善できれば幸いである。

## Table of Contents
Abstract

Status of This Memo

Table of Contents

1. Introduction
2. Requirements Language
3. Terminology
4. Design Goals
5. Non-Goals
6. Architecture
7. Pairing
8. Session Establishment
9. Packet Format
10. Reliability and Fragmentation
11. Security Considerations
12. Regulatory Considerations
13. References
14. Acknowledgements

## 1. Introduction
「もしも自分の思考を、遠くにいる友達とやり取りできたら。

もしも意思疎通を、阿吽の呼吸のように見せることができたら。」

Tôon Wire Protocolは、2024年夏の深夜に思いついたそのような発想を出発点としている。
その後、通信、暗号、低電力無線、プロトコル設計に関する知識をもとに、「もしこのような通信を実現するとしたら、どのようなプロトコルが必要になるのか」を検討し、本仕様をGitHub上で公開することにした。

当初は、通信プロトコルだけでなく、デバイスやUIを含む全体設計を行うことも考えていた。
しかし、「脳内にGUIを表示し、意識することによって入力を行う」という構想は、現段階では実現困難である。
そのため、本仕様では思考の取得、解読、提示、入力方法そのものは扱わず、通信プロトコルの設計に範囲を限定する。

本プロトコルが扱うのは、そのような上位層のデバイスやUIが存在すると仮定したうえで、比較的近距離に存在する既知の相手同士が、低電力無線通信を用いてメッセージを交換するための通信仕様である。
このため、本仕様ではLoRaのような低電力・低帯域の無線通信を前提とし、中央サーバー、DNS、広域ルーティングに依存しないP2P通信を重視する。
また、通信時のヘッダー容量を削減するため、初回ペアリング時に交換する長期識別子であるPermanent-IDと、セッション中に使用する短期識別子であるCIDを分離する。

## 2. Requirements Language

この文書におけるキーワード "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", および "OPTIONAL" は、すべて大文字で記述された場合に限り、BCP 14、すなわち RFC 2119 および RFC 8174 に記述されている意味で解釈される。
これらのキーワードは、Tôon Wire Protocolの実装における要求レベルを示すために使用される。

## 3. Terminology

本仕様では、以下の用語を使用する。

- **Peer**: Tôon Wire Protocolにおいて通信相手となる端末またはユーザーを指す。
- **Permanent-ID**: 初回ペアリング時に交換される128bitの長期識別子である。通常のDATAフレームには含めない。
- **CID**: Session Establishment後に使用される短期識別子である。通常通信ではPermanent-IDの代わりにCIDを用いる。
- **OOB Channel**: 初回ペアリング時に使用される、LoRaとは異なる近距離通信路である。
- **Pair Secret**: 初回ペアリング時に導出されるPeerごとの長期秘密である。
- **Pair Hint**: Session Establishment時にPair Secretとsession nonceから導出される一時的な検索タグである。
- **Peer Registry**: 初回ペアリング済みのPeerに関する情報を保存するローカルな記録領域である。

## 4. Design goals
Tôon Wire Protocolは、広域通信網や中央サーバーに依存しない、近距離・低電力・既知相手向けの直接通信を重視する。
本プロトコルは、不特定多数との通信やユーザー探索を目的とせず、あらかじめ登録された相手、または対面などの帯域外手段によって確認された相手との通信を前提とする。
設計上の主な方針は以下の通りである。

- 近距離通信を前提とする
- 既知の相手との通信を前提とする
- 中央サーバーに依存しない
- DNSのような名前解決を使用しない
- 低電力無線通信に適した小さなパケットを使用する
- Permanent-IDとCIDを分離し、通信中のヘッダーを小さく保つ
- 正常時は過剰なACKを返さず、必要に応じてNACKによる再送要求を行う
- OSI参照モデルにおける主に第1層から第5層までを対象とする

本仕様は、メッセージをどのように取得するか、どのように人間へ提示するか、またその内容をどのように解釈するかについては規定しない。

## 5. Non-goals

Tôon Wire Protocolは以下を対象としない。

- インターネット全体を対象とした通信網
- DNSのような名前解決システム
- 中央サーバーを利用したメッセージ中継
- 匿名通信ネットワーク
- LoRaWAN上で動作するアプリケーションとしての実装
- LoRaWANゲートウェイ、Network Server、Join Server、Application Serverへの依存
- 人間の思考の取得や解読
- 脳への情報提示手法
- UI/UXの設計
- 自然言語の意味解析
- SNS機能の実装
- 音声通話および映像通話

## 6. Architecture

### 6.1 Reference Device Model

本節では、Tôon Wire Protocolの設計時に想定する参考デバイスモデルを示す。
本節の内容は、実装の一例を説明するものであり、特に明記されない限り、すべての実装に対する必須要件ではない。

Tôon Wire Protocolは、主に身体装着型または携帯型の小型端末上で動作することを想定する。
想定される端末は、以下のような構成要素を備える。

- LoRa PHYによる送受信機能
- 初回ペアリングのための近距離OOBチャネル
- Pair Secret、秘密鍵、およびPeer Registryを保護するための保護ストレージ
- ECDHE、HKDF、HMAC、および認証付き暗号を処理できる計算資源
- 地域ごとの無線規制に従う送信制御機能
- 上位層アプリケーションまたはユーザーインターフェースとの接続点

初期実装では、OOBチャネルとしてBluetooth Low Energyを使用してもよい。
BLEをOOBチャネルとして使用する実装は、LE Secure Connectionsまたは同等以上の安全性を持つ認証付きペアリング方式を使用するべきである（SHOULD）。

実装は、Pair Secretおよび長期秘密鍵を通常の平文ストレージに保存するべきではない（SHOULD NOT）。
可能であれば、TPM 2.0、Secure Element、または同等のhardware-backed protected storageを使用するべきである（SHOULD）。

本仕様は、特定の筐体形状、電池方式、入力方式、表示方式、またはニューロインターフェースの実装方法を規定しない。
これらはTôon Wire Protocolの上位層または実装依存の設計事項である。

## 7. Pairing

### 7.1 Overview

初回ペアリング手順は、物理的に同一空間に存在する2つの端末間で、長期的な信頼関係を確立するために実行される。

この手順の目的は、Permanent-IDの交換、人間による相互認証、および後続のLoRa通信に使用する暗号学的な鍵素材の生成を行い、Tôon Wire Protocolにおける通常通信の基盤を構築することである。

Tôon Wire Protocolは、携帯電話網、インターネットメッセンジャー、または既存のSNSを置き換えることを目的としない。
本仕様は、将来の思考通信を想定した上位層が存在する場合に、その下位通信路としてどのような低電力P2P通信プロトコルが考えられるかを検討する実験的草案である。

### 7.2 Physical Co-presence

ペアリングを行うユーザー同士は、物理的に同一空間に存在している状態でペアリングを行わなければならない（MUST）。

これは、本プロトコルにおける「既知の相手同士による通信」という設計思想に基づく。

### 7.3 Out-of-Band Channel

初回ペアリング時、端末はLoRaとは異なる近距離無線チャネルを使用して、Permanent-IDの交換および人間による相互認証を行わなければならない（MUST）。

本仕様では、この近距離チャネルをOut-of-Band Channel、以後OOBチャネルと呼ぶ。

初期実装におけるOOBチャネルとして、Bluetooth Low Energy（BLE）を使用することを想定する。
BLEを用いる実装は、LE Secure Connectionsに対応しなければならない（MUST）。

BLEは以下の理由から、初回ペアリング用のOOBチャネルとして適している。

- 近距離無線である
- 双方向通信が可能である
- 人間の関与を前提とした認証方式を利用できる
- 低電力であり、耳掛けデバイスに適する

### 7.4 Human Authentication

初回ペアリング時、端末は人間が相手を確認できる認証手段を提供すべきである（SHOULD）。

BLEをOOBチャネルとして使用する実装では、LE Secure ConnectionsのNumeric Comparisonを使用することを推奨する（SHOULD）。

Numeric Comparisonを使用する場合、両端末は同一の確認値をユーザーに提示しなければならない（MUST）。
ユーザーは、提示された確認値が両端末で一致することを確認した場合にのみ、ペアリングを承認するべきである（SHOULD）。

確認値の提示は、画面表示、音声読み上げ、脳内GUI、またはその他の上位層インターフェースによって行われてもよい（MAY）。
実装は、確認値の一致をユーザーが十分に判断できない場合、ペアリングを中止できる手段を提供するべきである（SHOULD）。

確認値が一致しない場合、またはユーザーが一致を確認できない場合、端末はペアリングを完了してはならない（MUST NOT）。

この手順は、初回ペアリング時の中間者攻撃（MITM）を防ぐことを目的とする。


### 7.5 Key Establishment

#### 7.5.1 Pair Secret

初回ペアリング時、端末はBLE LE Secure Connections、または同等以上の安全性を持つ手順を用いて、Permanent-IDおよび後続通信に必要な情報の交換を保護すべきである（SHOULD）。

本仕様では、初回ペアリングの結果として、Peerごとの長期秘密であるPair Secretを導出する。

Pair Secretは、後続のSession Establishmentにおいて、Pair Hintの導出、ハンドシェイクの認証、およびセッション鍵導出に使用される鍵素材である。

Pair Secretは、無線通信上で送信してはならない（MUST NOT）。
Pair Secretは、初回ペアリング時に実行される鍵交換手順、およびペアリング時に交換された識別情報をもとに、両端末がそれぞれ導出する。

実装は、Pair SecretをPermanent-ID、Display Name、またはその他の公開情報のみから導出してはならない（MUST NOT）。

実装は、Pair SecretをTPM 2.0、Secure Element、OS KeyStore、または同等のprotected storageに保存するべきである（SHOULD）。

#### 7.5.2 Pair Secret Derivation

初回ペアリング時、端末は後続のSession Establishmentで使用するPeerごとの長期秘密として、Pair Secretを導出する。

初期実装では、OOBチャネル上でTôon Wire Protocol用のECDHE鍵交換を行い、その共有秘密をPair Secret導出の主材料として使用することを想定する。

概念的には、Pair Secretは以下のように導出される。

```text
pairing_shared_secret = ECDHE(pairing_private_key, peer_pairing_public_key)

pair_secret = HKDF-SHA256(
  input = pairing_shared_secret,
  salt = pairing_nonce_i || pairing_nonce_r,
  info = pairing_transcript || "TWP Pair Secret v1",
  length = 32
)
```

`pairing_nonce_i` および `pairing_nonce_r` は、初回ペアリング時に各端末が生成するランダム値である。

`pairing_transcript` は、初回ペアリング時に交換されたPermanent-ID、Negotiated TWP Version、Selected Crypto Suite、ECDHE公開鍵、およびその他の合意対象情報を含む正規化されたバイト列である。

`pairing_transcript` に含まれる情報の順序は、両端末で同一になるように定義されなければならない（MUST）。
たとえば、initiator側の情報を先に置き、responder側の情報を後に置くなど、実装間で曖昧にならない順序を使用する。

Pair Secretの導出に使用した一時秘密鍵、pairing_shared_secret、およびその他の一時的な鍵素材は、Pair Secret導出後に破棄されるべきである（SHOULD）。

### 7.6 Information Exchange

OOBチャネル上で認証済み暗号通信が確立された後、端末同士は後続の通信に必要な情報を交換しなければならない（MUST）。

交換される情報には、少なくとも以下を含む。

* Permanent-ID
* 対応するTôon Wire Protocolのバージョン
* 対応する暗号方式または暗号スイート
* 対応するフレームサイズ
* 対応するLoRa通信パラメータ

実装は、ユーザー表示のためにDisplay Nameを交換してもよい（MAY）。

Display Nameは、相手端末が提示するローカル表示名である。
受信側の実装は、相手端末から受け取ったDisplay Nameを初期表示名として使用してもよい（MAY）。

ユーザーは、ローカル端末上でDisplay Nameを後から変更できるべきである（SHOULD）。
Display NameはPeerの認証に使用してはならない（MUST NOT）。

Pair Secret、秘密鍵、またはPair Secretを参照するための実装固有のhandleは、Information Exchangeに含めてはならない（MUST NOT）。

### 7.7 Peer Registry Entry

初回ペアリングが完了した端末は、相手Peerに関する情報をPeer Registryに保存する。

Peer Registry Entryは、後続のSession Establishmentにおいて、接続要求の照合、暗号方式の確認、Pair Hintの検証、およびセッション鍵導出に使用される。

Peer Registry Entryは、少なくとも以下の情報を含むべきである（SHOULD）。

* Peer Permanent-ID
* Negotiated TWP Version
* Selected Crypto Suite
* Trust State
* Pair Secret or Pair Secret Handle

実装は、ユーザー表示のためにDisplay Nameを保存してもよい（MAY）。

Display Nameが存在しない場合、実装はPeer Permanent-IDの短縮表記などを用いて、ユーザーがPeerを識別できる代替表示を提供するべきである（SHOULD）。

Trust Stateは、そのPeerとの通信を現在許可するかどうかを示す。
実装は、少なくともpaired、blocked、revokedに相当する状態を扱うべきである（SHOULD）。

Pair Secretは、Peerごとの長期秘密であり、通常のLoRa通信において送信されない。
Pair Secretは、Session Establishment時にPair Hint、ハンドシェイク認証値、およびセッション鍵を導出するために使用される。

`pairing_nonce_i` および `pairing_nonce_r` は、Pair Secret導出時の一時的な入力であり、Pair Secret導出後にPeer Registry Entryへ保存することを要求しない。

Peer Registry Entryは、Pair Secret本体の代わりに、Pair Secretを参照するための実装固有のhandleを保持してもよい（MAY）。
このhandleはローカル実装上の参照情報であり、無線通信上で送信してはならない（MUST NOT）。

実装は、Pair Secretの導出に使用したKDF、暗号スイート、および正規化されたペアリングtranscriptのハッシュ値を保存してもよい（MAY）。

* Pair Secret KDF ID
* Pairing Transcript Hash

Pairing Transcript Hashは、デバッグ、監査、再ペアリング判定、または将来の鍵更新手順のために使用されてもよい（MAY）。

複数のTWPバージョンまたは複数の暗号スイートをサポートする実装は、ダウングレード攻撃を防ぐため、合意されたバージョンおよび暗号スイートをペアリング時の認証対象に含めるべきである（SHOULD）。

### 7.8 Scope of BLE Usage

BLEまたはその他のOOBチャネルは、初回ペアリングのために使用される。

ペアリング完了後、通常のTôon Wire通信はLoRaによって行われなければならない（MUST）。

実装は、BLEを通常メッセージ配送のための恒常的な通信路として使用してはならない（MUST NOT）。
ただし、再ペアリング、鍵更新、または明示的なメンテナンス操作のために、OOBチャネルを再度使用してもよい（MAY）。

## 8. Session Establishment

### 8.1 Pair Hint Derivation

Session Establishmentを開始する端末は、SESSION_INITフレームを送信する前に、セッションごとのランダム値である `session_nonce_i` を生成する。

送信側は、対象PeerとのPair Secretおよび `session_nonce_i` を用いて、Pair Hintを導出する。

```text
Pair Hint = Truncate64(
  HMAC-SHA256(pair_secret, session_nonce_i || "TWP Pair Hint v1")
)
```

`Truncate64` は、HMAC-SHA256の出力の先頭64bit、すなわち8 bytesを取り出す操作を意味する。

Pair Hintは、受信側がSESSION_INITの対象Peerを特定するための一時的な検索タグである。
Pair HintはPermanent-IDの短縮表現ではない。
実装は、Permanent-IDのみからPair Hintを導出してはならない（MUST NOT）。

Pair Hintは固定値として保存されない。
同じPeerに対するSession Establishmentであっても、`session_nonce_i` が変わるため、Pair Hintは毎回異なる値になる。

`session_nonce_i` は、暗号学的に安全な乱数生成器によって生成されるべきである（SHOULD）。
実装は、同一Peerに対して同じ `session_nonce_i` を意図的に再利用してはならない（MUST NOT）。

SESSION_INITを受信した端末は、Peer Registry内の各Pair Secretを用いてPair Hintを検証する。
一致するPeer Registry Entryが1つだけ存在する場合、そのSESSION_INITは当該Peerからの接続要求として扱われる。

一致するPeerが存在しない場合、実装はそのSESSION_INITを破棄しなければならない（MUST）。
複数のPeerが同一のPair Hintに一致した場合、実装は安全のためそのSESSION_INITを破棄するべきである（SHOULD）。

Trust StateがblockedまたはrevokedであるPeerに一致した場合、実装はそのSESSION_INITに応答してはならない（MUST NOT）。

Pair Hintの一致は、SESSION_INITの対象Peerを特定するための手段であり、Session Establishmentの完了を意味しない。
実装は、後続のハンドシェイク認証およびセッション鍵導出が完了するまで、通常DATAフレームの送受信を開始してはならない（MUST NOT）。
## 13. References

### 13.1 Normative References

[RFC2119]
Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, BCP 14, RFC 2119, March 1997.

[RFC8174]
Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, BCP 14, RFC 8174, May 2017.

### 13.2 Informative References

[RFC8376]
Farrell, S., Ed., “Low-Power Wide Area Network (LPWAN) Overview”, RFC 8376, May 2018.

[RFC5869]
Krawczyk, H. and P. Eronen, “HMAC-based Extract-and-Expand Key Derivation Function (HKDF)”, RFC 5869, May 2010.

[RFC7748]
Langley, A., Hamburg, M., and S. Turner, “Elliptic Curves for Security”, RFC 7748, January 2016.

[RFC8439]
Nir, Y. and A. Langley, “ChaCha20 and Poly1305 for IETF Protocols”, RFC 8439, June 2018.

## 14. Acknowledgements

著者は、本仕様の構成および記述に大きな影響を与えたIETFコミュニティ、ならびに既存のインターネットプロトコルの設計者およびRFC著者に謝意を表する。

本仕様は、ChatGPTの支援を受けて作成された。ChatGPTは、設計方針の検討、文章の推敲、章構成の整理、およびプロトコル設計上の論点整理に使用された。最終的な設計判断、用語の選択、および本文の内容に関する責任は著者にある。

また、本実験的仕様に対して、将来的にIssue、Pull Request、レビュー、実装上の知見、またはSecurity Considerationsを提供するすべての読者および実装者に感謝する。
