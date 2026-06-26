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

再度検討中につき文言削除

## 8. Session Establishment

再度検討中につき文言削除

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
