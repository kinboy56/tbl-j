---
title: ソード・シールド
nav_order: 6
has_children: true
---

# ソード・シールド

ポケットモンスター ソード・シールドは、C++ のゲームコードが Protocol Buffers のメッセージを使い、
汎用のパブリッシュ/サブスクライブ型フレームワークを介して、Pia の上で直接動いています。

静的な読み取りは Shield 1.3.2 の EUR イメージ (`01008db008c2c000`、アップデート NCA、SDK 7.7.0.0) から行い、
計測はフランス語版の Sword 1.3.2 に対して行いました。両者はネットワークコードを共有しており、
差異がある箇所にはその旨を明記しています。

製品版の Sword は、両方の役割で pokeldn と交換したことがあります ([交換の主催](swsh_trade.md#hosting-a-trade))。

| ページ | 内容 |
|---|---|
| [カートリッジとセッション](swsh_session.md) | 鍵、イメージ、Pia 4、最初のアプリケーションデータへの紐付け |
| [同期フレームワーク](swsh_protocol.md) | メッセージ ID、内容、保持者、ルーティング、パーティのペイロード |
| [交換](swsh_trade.md) | 交換画面、ボックスの状態機械、確認の段階 |
| [ふしぎなおくりもの](swsh_gift.md) | ふしぎなおくりもののメニューのローカル無線の分岐 |
| [マックスレイドバトル](swsh_raid.md) | 巣穴のロビー、その告知、1 台または複数の本体が参加するレイドの主催、参加者が戦うバトル、ダイマックスアドベンチャー、退出 |

## 情報源

ID、オフセット、アドレスは、節でクライアントが明記されていない限り、Shield の `main` か電波上の通信から得ています。
`pokeldn/swsh/trade.py` の `SYNC_ANSWERS` は、本体同士の通信を取り込んだものを `andyjusa/nxldn-lab` が読み解いた結果です
(メッセージ 97 と 60000 は pokeldn とバイト単位で一致します)。LAN モードで通信するクライアントは 4 つあり、
`kwsch/PokePiaSWSH`、`lincoln-lm/swsh-lan-client`、`andyjusa/nxldn-lab`、`Slashcash/PSD` で、
Pia のステーションのハンドシェイク以降を扱います。いずれも LDN やホストの移行は扱いません。

## フィールドスクリプト

`bin/script/amx/*.amx` (1.3.2 では 953 個) は 64 ビットセルの Pawn 3.x で、マジックは `0xF1E1`、バージョンは 10、
フラグは `0x1C` (compact 形式のコード、sleep、検査なし) です。compact エンコーディングでは、1 バイトあたり 7 ビットを上位のグループから順に並べ、
符号は最初のバイトのビット 6 に入ります。分岐しない引数 1 個のオペコードは `(param << 32) | op` の形でも詰められ、
3.x の順で 162 から番号が振られています (`LOAD.pri` 162、`LOAD.S.pri` 164、`PUSH.C`
188、`PUSH.S` 190、`STACK` 191、`ADD.C` 197、`ZERO.S` 200、`EQ.C.pri` 201、`INC.S` 204、`HALT` 210、
`PUSH.ADR` 212)。オフセット 0 の `halt 0` が、この番号付けを確定させます。

ネイティブ関数は 12 バイトで、`u64 0` と `u32` の名前ハッシュからなり、`amx_Register` (`0x0066d970`) で解決されます。次のとおりです。

    h = 0; for each byte c: h = (h * 0x83) ^ c        (32-bit)

バインディングテーブルは、モジュールごとに `.data` 内の `(name, function)` の組です (`0x014aea00` などの取得関数があります)。名前は 777 個あり、513 個のハッシュのうち 512 個が使われています。
アイテム関連は `ItemAdd` (`0x014acea0`)、`ItemSub`
(`0x014acf40`)、`ItemGetNum`、`ItemAddCheck`、`ItemGetCategory`、`GetPocketNumberFromItemNumber_` です。
変数関連は `WorkGet`/`WorkSet` (ハッシュ化された 64 ビットキー) と、`TempWorkGet`/`TempWorkSet` (スクリプトが再開する前に UI が埋めるインデックス) です。
呼び出しは、引数ごとに右から左へ `PUSH` し、そのあと `SYSREQ.N native, bytes` を実行する形になります。

## 未解決

- [プレイヤープロファイル](swsh_protocol.md#the-player-profile): ポケモンキャンプのセッション (`StateCreateSession`、`StateConnect`) で、
  役割のサンプルの状態 3 と 4 が何を表すか。`a_wr0301` を冠の雪原の野生エリアとするのは番号から読み取ったもので、
  そこで取得したビーコンが 1 つあれば確定します。
- [バトルスタジアム](swsh_protocol.md#the-battle-stadium-block): 保存されたチームの記録を読むのにセーブデータのどのブロックを `0x01430670` が使うか。
  また、`v1/validate` の署名が `+0x36` を置き換える前に、セーブチームのディスクリプタが `+0x30..+0x135` に何を保持しているか
  (`0x015039f0` のあとの `[sp+0x28]` にある要素)。
- Sword と Shield の比較: バイナリの読み取りは Shield のもので、本体は Sword です。
  [セッションの定数](swsh_session.md#taking-a-seat) は両者で共通です。カードのバージョンマスクのビット 0 を Sword が検査していることは、
  Shield のコードと PKHeX の `RestrictVersion` (1 が Sword、2 が Shield、3 が両方) から推測したものです。Shield では検査が `1 << (v == 0x2D)` になっており、
  `0x007d4270` の 31 個の呼び出し箇所のうち 11 個が `0x2C` と `0x2D` の両方と比較しています。
  製品版 Sword の 2 つのスナップショットでは、MyStatus の `+0xA4` に `0x2C` が入っています。
  Sword 独自の `0x007d4270` と通信 ID の定数は、未解析です。
- [ふしぎなおくりもの](swsh_gift.md#what-the-menu-refuses): ゲームに存在しない種族の種類 1 のおくりものを、
  エミュレートした Shield がそうするように、製品版の本体が「Egg」という名前の不正なタマゴとして受け取るかどうか。
  また、そのようなタマゴが孵化時や交換でどうなるか。
