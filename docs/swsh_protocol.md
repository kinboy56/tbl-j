---
title: 同期フレームワーク
parent: Sword and Shield
nav_order: 2
---

# メッセージ、コンテンツ、ルーティング

Pia の上位にある publish/subscribe フレームワークです。メッセージは u16 LE の id、判別子バイト、0 バイト、
protobuf の本体からなります。

## メッセージ id の出どころ

下位の id は、`0x01BBFFA0` にある 838 件の 24 バイトレコードの表から得られます。レコードは
`{u64 handler slot, u32 0x402, u32 id, u64 0}` の形で、id は 1..880 です（97 は ping ホルダー。110、120、130 もこの範囲です）。
上位の id は基点にオフセットを足したものです。20030 や 40040 は定数として一度も現れず、60000 は基点 + 0 です（41 か所で MOVZ）。

## コンテンツとホルダー

各コンテンツは 3 つのホルダーを登録します（登録関数は、コンテンツ 30 が `0x010ccd90`、40 が `0x010da7d0`、
50 が `0x010d5150`）。コンテンツのオフセット `ldrh [content+0x372]` に基点を足し、各ホルダーを
`0x006daeb0(manager, &holder, flag)` に渡します:

    mov w9, #0x2710    id = 10000 + offset   holder 0x010dd910   flag 1
    mov w9, #0x4e20    id = 20000 + offset   holder 0x010d85f0   flag 0
    mov w9, #0x7530    id = 30000 + offset   holder 0x010d0980   flag 0

40000 系は一段下の層で作られ、40000 は MOVN として生成されます:

    ldrb w20, [x0, #0x28]          the content offset, 30 / 40 / 50
    mov  w9, #-0x63c0              w9 = 0xffff9c40
    add  w24, w20, w9
    strh w24, [x20, #0x160]        40030 / 40040 / 40050

リスナーのないホルダーは、メッセージを黙って捨てます（`0x010d81d0`: `ldr x8,[x0,#0x168]; cbz x8, out`）。
登録箇所: コンテンツ 30 は `0x010ccc94`、`0x010ccca4`、`0x010ccf7c`。コンテンツ 40 は `0x010da6d0`、
`0x010da9bc`（解除は `0x010dab10`）。コンテンツ 50 は `0x010d50ac`、`0x010d533c`（解除は `0x010d5660`）。
40 と 50 の 20000 基点ホルダーには、どれも到達しません。20040 と 20050 は機能しません。

### 30000 のホルダー

30000 + オフセットのホルダー（`0x010d0980`、vtable GOT `0x02618620` = `0x02528100`、RTTI なし）は
`gflnet.p2p.framework.pb.SequenceDataHolder` を持ちます:

    SequenceDataHolder    oneof message {
                            1 CancelAccepted        cancelAccepted
                            2 RequestCancel         requestCancel
                            3 RequestCancelAll      requestCancelAll
                            4 RequestForcedProceed  requestForcedProceed }
    CancelAccepted        { 1 int32 currentSeqNo; 2 bool  isForced    }
    RequestCancel         { 1 int32 currentSeqNo }
    RequestCancelAll      { 1 int32 currentSeqNo }
    RequestForcedProceed  { 1 int32 currentSeqNo; 2 int32 targetSeqNo }

パーサーは `0x006a7580`（タグ `0x0a`..`0x22`）です。RequestForcedProceed の `0x006a65c0` は、フィールドを
`+0x14`、`+0x18` に格納します。受信処理（スロット 8、`0x008b9590`）は、`0x0204c430` を経由して
`[holder+0x168]` のリスナースロット（case - 1）を呼び出します。

Shield 1.3.2 が作るのは CancelAccepted だけです。protobuf コード（`0x006a3000..0x006a8400`）の外では、
他のコンストラクタ（`0x006a4b50`、`0x006a5730`、`0x006a6330`）もそのアクセサも呼ばれません。
`0x006a3db0` には外部からの呼び出しが 21 件あります。送信ラッパー `0x008b7420` と、10 個の同期コンテンツそれぞれの
2 つの返信ジョブです（`0x008b71a0`、`0x008b7700`、`0x008cfb10`、`0x008cfbe0`、
`0x00c06420`、`0x00c064f0`、`0x010763a0`、`0x01076470`、`0x010a2090`、`0x010a2160`、`0x010cf460`、
`0x010cf530`、`0x010d7760`、`0x010d7830`、`0x010dd0c0`、`0x010dd190`、`0x012a8390`、`0x012a8460`、
`0x012bec80`、`0x012bed50`）。小売の交換では case 2 から 4 が現れませんが、コンソールはそれらにも動作します（
[コンテンツ 40](swsh_trade.md#the-cancel-and-proceed-messages)）。コンテンツ 40 のインスタンス 30040（ポート 0）は次のとおりです:

    RequestForcedProceed{c, c+1}   58750000 2204 08<c> 10<c+1>     c=0: 58750000 2204 0800 1001

### 3 つの交換コンテンツ

`0x010c9280` がセッションを組み立てます:

    +0x120   content 30, the box exchange       ctor 0x010cca10, registers offset 30 at 0x010ce4f0
    +0x148   content 40, SyncSaveDataHolder     ctor 0x010da3f0
    +0x2b0   content 50, PokemonTradeDataHolder ctor 0x010d4d40
    +0x60    an event source, listeners in a vector at its own +0x60
    +0x68    own Pokemon        +0x70  theirs
    +0x90    a delegate to +0xb0: thunk 0x010cc250 -> box callback 0x010ca800(session, code, payload)
    +0x140   the trade state, 1..10            +0x144  the error code
    +0x418   send command 3 on the first frame  +0x419  the role bit that suppresses it

各コンテンツの init は、ホルダーを登録して 40000 の id を生成します。コンテンツ 50 は交換の状態 1（`0x010d4d90`）で、
コンテンツ 40 は状態 6（`0x010da470`）で生成されます。これは交換（状態 3）と状態 4 の判定 `0x01109320` の後です。

## ルーティングの経路

    0x006a9a20   the sync pump, from the trade session update
    0x006db3b0   the poll: per registered entry, drain both streams of its kind (byte +8)
    0x006a8490   stream A: mesh port [pia+0xd0+kind*4], Pia protocol 0x7C
    0x006a84f0   stream B: mesh port [pia+0xd8+kind*4], Pia protocol 0x80
    0x006db9c0   drain: slot 0x78 fills (sender*, length) and the buffer at manager+0xf0
    0x006db620   dispatch: match the entry, call the holder's slot 8
    0x010d81d0   content 50's 10000-base holder: parse, call the listener's slot 0
    0x010d5e40   the listener: sender to station index, memcpy 0x158, invoke

### アプリケーションヘッダー

ペイロードは `struct.pack("<HBB", id, discriminator, 0)` の後に本体を続けたものです。`manager+0x240f0` で
`0x006db840` が組み立てます（`strh w3`、判別子は `manager+0x480f0` から、`strb wzr`）。解析は `0x006db620` が行います。
判別子は世代カウンタです。`[content+0x370] = (+1) mod 255`（`0x008b6670`）で進み、`[manager+0x480f0]` に格納されます。
強制 CancelAccepted が来ると、コンテンツ 40 でこれが 1 つ進みます（`0x006db470`、`0x010dcea4..0x010dcec8`）。
登録関数は `0x010d53a0` でこれを 0 にします。強制 CancelAccepted のない 1 回の交換の 107 個のペイロードは、すべて 0 でした。

### 登録内容をライブで読む

    manager = read_u64(read_u64(main + 0x02616750))     0x006a9a70, the poll's caller
    count   = read_u64(manager + 0xD8)                  0x006db3dc
    array   = read_u64(manager + 0xD0)                  0x006db3e0
    entry i = array + i * 0x10                          0x006db3e4
    id      = holder's vtable slot 7, called at 0x006db740 (`ldr x8, [x8, #0x38]`)

ここではスロット 7 が `[holder+0x160]` を返します。ほかの箇所では、その `ldrh` を読み解いてください。待機画面では空です。
着席したリンク交換では 2 件が追加されます（着席から 2 秒以内に確認）。ふしぎなおくりものでは追加されません。

### ディスパッチの条件、ポート、送信元

エントリは 16 バイトです。ホルダー、+8 の kind（`w2`）、+9 のフラグ（`w3`、10000 + オフセットの場合だけ 1）からなります:

    id == holder->slot7()
    entry[8] == the drain's kind
    entry[9] ? header[2] == [manager+0x480f0] : no check

kind はメッシュポートです。コンテンツのホルダー（20030、10050、ping）は 0、40000 系は 1 です（`0x006daf40(manager, &holder, 1, 0)`）。
送信元は、`0x006db9c0` の `[sp+0x28]` にあるトランスポートポインタです。これが `0x010d5e40` に渡され、
`0x006b5850`（`mesh->GetStationIndex`、失敗時 0xfd）で解決されます。ループバックは `[[0x2616a30]]+0xf0` を渡します。
10050 では `Data.ownerId` は関係ありません。

## `Data` エンベロープ

`gflnet.p2p.sync.pb`、`data.proto`:

    1  uint32  syncId        the content offset       30 / 40 / 50
    2  uint32  elementId     the base                 10000 / 20000
    3  uint64  ownerId       the sender
    4  uint64  clock
    5  bytes   body          four bytes in a pair; a 344-byte PK8 in a 40050

40000 のホルダーのリスナーは `element+0x18` です。そのスロット 0（`0x006d59f0`）が次のように振り分けます:

    [Data+0x14] == [listener+0x10]        syncId, the element's offset
    walk [listener+0x28] .. [+0x30]       sub-elements, 0x90 bytes each
      [Data+0x18] == [sub+0x62]           elementId, u16
      [Data+0x20] == [sub+0x68]           ownerId, u64
    sub->vtable at 0x48 (sub, body, len, [Data+0x28])     body and clock
    otherwise: ret                        dropped silently

### サブ要素の種類

| 種類 | コンストラクタ | vtable | 生成時の `+0x88` | スロット 9、受信: サイズ確認、`+0x88` に格納 |
|---|---|---|---|---|
| 32 ビット | `0x006d5c00` | `0x2512bc8` | 0 (`0x006d5ce0`) | `0x006d5f20`: 4、`ldr w8` / `str w8` |
| pair（フェーズ） | `0x006d6160` | `0x2512c90` | `0xfc18fc18` (`0x006d6230`) | `0x006d6490`: 4、`ldr w8` / `str w8` |
| 16 ビット | `0x006d66d0` | `0x2512d58` | 0 (`0x006d67b0`) | `0x006d69f0`: 2、`ldrh w8` / `strh w8` |

受信後は `[sub+0x78] = clock` と `strh 0x0100 -> [sub+0x60]` を設定します（ready バイトは `+0x61`）。
スロット 7 はハッシュ値で、clock を返します。サブ要素は 0x90 バイトです。コンストラクタは、要素 `x0`（`+0x80`）、
elementId `w1`（`+0x62`）、ownerId `x2`（`+0x68`）、フラグ `w3`（`+0x70`）を受け取り、`x8` 経由で返します。

同期要素（`0x006d3f80`）は、同期コンテンツごとに 1 回作られます: `0x008b5c60`、`0x008cecb0`、
`0x00c05690`、`0x01075610`、`0x010a1230`、`0x010ce604`（30）、`0x010d6904`（50）、`0x010dc264`（40）、
`0x012a7530`、`0x012bdef0`。vtable は `0x2512b28` から始まります:

    element+0x00  0x2512b38   primary, 6 slots; 2, 3, 4 are the three constructors with w3 = 1
    element+0x08  0x2512b78   interface A: the 32-bit kind (thunk 0x006d5ab0)
    element+0x10  0x2512b90   interface B: the pair (0x006d5ac0), the 16-bit kind (0x006d5ad0)
    element+0x18  0x2512bb0   the router listener, slot 0 = 0x006d59f0

生成関数 `0x006d44e0` は、`[element+0x70..+0x78]` の全局を加えて 4 つのチャンネルを作ります:

| フィールド | 作成者 | サイズ | 保持するもの |
|---|---|---|---|
| `+0xb0` | `0x006cd890` | 0x130 | |
| `+0xd0` | `0x006d9ca0`、A | 0x38 | 局ごとに 32 ビットのサブ要素（elementId 10000、`0x006d9d90`）。クォーラムハッシュ |
| `+0xf0` | `0x006d29b0`、B | 0x40 | `+0x38` に 16 ビットのサブ要素（elementId 20000、owner 0）: [共有値](swsh_trade.md#the-phase-is-the-elements-field)。局ごとに pair（elementId 20000、`0x006d2aa0`）を、16 バイトの `{stationId, sub}` エントリで持つ |
| `+0x110` | `0x006d7980` | 0x30 | クォーラムハッシュが走査するリスト |

### クォーラムハッシュ

elementId 10000 の 4 バイトは、`element+0xa8` の `0x006d7b40` が作ります:

    h = 0
    for sub in subs:
        if (!sub[+0x61]) { h = 0; break }        any station not ready -> zero
        h = crc32(le32(h + sub->slot7()))        slot 7 is the clock

0 以外なら、全局が ready です。`0x0065de30` は `zlib.crc32` です（unicorn で確認）。`0x0065df04`（多項式 `0x8005`）は CRC-16 です。
走査されるリスト `element+0x110` は、`0x006d48b0` が `+0x40` をコピーしたもので、`+0x68` が 0 のエントリだけを残します。
交換におけるその長さは未確認です。

チェーンは、送信元の elementId 0 と 1（owner なし）を経て、自身の elementId 20000 までを対象とします。
ホストの Sword が送った 0 以外のハッシュはすべて一致します。例としてコンテンツ 50 を挙げます:

    elementId 0, no owner      clock 2304   the host's own Pokemon (344 bytes)
    elementId 1, no owner      clock 2313   the partner's Pokemon, sent back
    elementId 20000, host      clock 2278   then 2338 after its pair moved
    crc32 chain                8ffa0f2e     then b3615e90

clock は、メッセージを送る前に変わることがあります。ホストのハッシュをそのまま返す参加者は、ホストの Sword に対して交換を完了させます。

## プロトコル 0x84 のパーティペイロード

`ReliableBroadcastProtocol` は、3456 バイトの交換スナップショットを 3 つの断片に分け、ack を受けるまで運びます。
3 つ目は zlib（Pia フラグ 0x10）です。1404 + 1404 + 157 バイト（生）で、最後の断片は展開すると 648 バイトになります。
`trade_payload.py` は、それ以外の合計を拒否します。レイアウトは次のとおりです（`kwsch/PokePiaSWSH`、
`lincoln-lm/swsh-lan-client` と同じ）:

    0x000  six PK8 records, party form, 0x158 each          -> 0x810
    0x810  u32   party count
    0x814  MyStatus, 272 bytes      TID/SID at 0xA0, trainer name at 0xB0
    0x924  TrainerCard, 456 bytes   trainer name at 0x00, save start date at 0x170
    0xAEC  the player profile, 266 bytes                     -> 0xBF6
    0xBF6  392 bytes, the Battle Stadium block; zero for Link Trade
    0xD7E  2 bytes of padding                                -> 0xD80 = 3456

MyStatus と TrainerCard は PKHeX のセーブブロックです（`Saves/Substructures/Gen8/SWSH/`）。

組み立て関数 `0x0110c180` は、`0x00784f90`（パーティ）、`0x01424f10`（MyStatus）、0x1C8 の memcpy（トレーナーカード）、
`0x01124fa0`（プロフィール）、任意ブロックの 0x188 の memcpy または memset を行います。`0x010fcff0` の呼び出し元のうち、
通信交換（`0x010967f0`）とパスワードマッチング（`0x00bd80f4`、`ChikaMatchingStateSession`）はブロックを渡しません。
バトルスタジアム（`0x00b2d7d0`、`StateBtlSpot*Battle`、Casual、Rank、Comp）は 1 つ渡します。
受信側 `0x0110cff0` は、局ごとに両方の領域をコピーします。

### バトルスタジアムのブロック

データは 0x180 バイトで、長さは u64 です。生成関数 `0x00b2eb60` が任意項目（フラグ `+0`、データ `+8`）を埋めます。
スナップショットは、そのホルダーの `+0x60`（`0x0110c378`）から 0x188 をコピーします。読み取り側 `0x00b2d9fc` は長さ 0x118 を要求します。
0xBF6 から始まります:

    0x000  u32    CRC-16 (0x8005) of the 0x22fc-byte regulation core   0x008ff9e0:
                  0x0065dd70([reg+0x180]+0x60, 0x22fc); regulation_preset_core_%d.bin (0x008feb34)
    0x004  0x100  the team's signature: +0x36 of the 0x136-byte team descriptor at match+0x98
                  (0x00b2ef24)
    0x104  4      zero
    0x108  u64    the manager's optional (value +0x1ab8, flag +0x1ab0, 0x00adc8d0), zero unless
                  match type 3                                         0x00b2f100
    0x110  u8     the manager's byte +0x191c (0x00adbbe0) when 0x00adb920 yields an object and
                  0x00b1ce10(0) == 2                                   0x00b2efc4
    0x111  u32, u16, u8   team descriptor +0, +4, +6
    0x118  0x68   zero
    0x180  u64    length, 0x118

受信側 `0x00b2e590` は、相手の CRC を自分のもの（`0x00b2e760`）と比較します。`0x008fc470` と合わせ、両方が成り立つとき結果は 9、
そうでなければ 10 です（`cinc` `0x00b2e778`）。`0x010719f0`、`0x01071a98`、`0x01071c00` は、それと
`+0x188` にある 0x640 バイトの CRC（`0x008ffa10`）を比較します。記述子は 18 か所で丸ごとコピーされ、
そのうち 4 か所は `StateDownloadTeamMenu`（`0x013e660c` から `0x013e7c58`）です。

match+0x70 は任意項目です（フラグ `+0x70`、値は `+0x78` から: u64 のキー、ポインタ、u64、ポインタ、そして `+0x98` の記述子）。
書き込み元は `0x00ad9cf0`（`0x00adeaf0` の memcpy）だけです。バトルスタジアムのマッチング更新 `0x00ad5750` が、
`[state+0xb8]+0x20` から `0x00ad6a2c` で埋めます。`0x00ad6a68` と `0x00b1ea88` がこれを消去します。
そのリクエストオブジェクトは、チーム取得ジョブ（`0x014f7fd0`..`0x014f809c`、完了フラグ `+0x18`）が、
`[job+0x88]` の結果ホルダーから埋めます。記述子の `+0x00..+0x2f` は `+0x38` からの u64 6 個、`+0x30..+0x35` は
`+0x68` からのバイト 6 個、`+0x36..+0x135` は `+0x6e` からです。その後、`v1/validate` の署名（`0x014f8094`）で上書きされます。
ホルダーは、キーがリストの選択中のキーと一致するチームリストのエントリ（0x158 バイト）を丸ごと受け取ります
（選択中のキーは `0x01503f10`、`0x014fa804` でコピーされます）。リスト `0x0150320c` は 2 種類のエントリを持ちます:

| キー | エントリ |
|---|---|
| (1, team) | セーブ内のバトルチーム: メンバーごとに u64 の `box \| slot << 32` を 6 個。`save+0x525 + team*12 + i*2` から（`0x0140a5c0`、`0x0140a5f0`）。名前は `save+0x4a0 + team*0x16` |
| (2, i) | 保存済みレコード: `0x01430670` が読み込む 0x87f バイトのレコードの、0x26..0x15b のバイト |

署名は RSA-2048、PKCS#1 v1.5、SHA-256 で、`0x011aedf0(team, v, sig)` が検証します（呼び出し元は `0x00b2f19c`、
`0x00b2f1e0`、`0x00b2f2e0`、`0x0109eb68`）。`nn::crypto::detail::BigNum::ModExp`（PLT `0x018fff50`、GOT `0x0260fb38`）の
唯一の呼び出し元です:

    0x011aee90  count 0x148-byte stored PK8s        bl 0x7664c0, add w26,#0x148
    0x011aeeac  v as BE u16, then 00 01             rev w8,w22; lsr #16; strh 0x100
    0x011aef98  Sha256Impl::Initialize
    0x011aefa8  BigNum::Set(modulus, .., 0x100), Set(exponent, ..)
    0x011aeffc  Sha256Impl::Update(message)
    0x011af01c  BigNum::ModExp(out, sig, exponent, 0x100, ..)
    0x011af06c  cmp x8,#0xca: the 00 01 FF..FF 00 padding
    0x011af090  memcmp(.., Sha256Generator::Asn1ObjectIdentifier, 0x13)
    0x011af128  Sha256Impl::GetHash, memcmp(.., 0x20)

コンソールが持つのは公開鍵だけです。バイナリイメージには
`https://v3-lp1.vp.n.srv.nintendo.net/v1/public_key`（`0x01bd7d41`）と `.../v1/validate`
（`0x01c11a93`）が含まれるため、署名するのは Nintendo のサーバーだと思われます。通信交換はこの検査に到達しません。

`v1/validate`（`0x011a2a70`）は、NUL 終端の文字列（`v1/public_key` の本文すべて）、鍵のバージョン（BE u16、鍵ホルダーの `+0x68`。
`v1/public_key` の応答から `0x0144fb90` が設定）、コンソールのバージョン `0x007d4270() = 0x2D`（Shield）を BE u16 で、
`00 01`、BE の件数、0x148 バイトのレコード（`0x007664c0`）を送ります。署名対象のメッセージには、同じレコード、バージョン、
`00 01` が入ります。そこでの `v` は、相手の MyStatus のバイト `0xA4` です（`obj+0x104`、`0x01424bf0`。既定値 `0x2D` は
`0x014245f0`、PKHeX の `MyStatus8.Game`）。応答パーサー `0x011a2870` は、バイト 0 をステータスとして扱います（2 は古い鍵で、
取得し直しが必要。`R+0x70` はステータスが 1 のとき真）。バイト 5-6 は BE の件数 n で、6 に制限します。n 個の BE u32 を
`R+0x74` に格納し、ステータスが 0 のときは 0x100 バイトを `R+0x8c` に格納します（`0x011a29e4`）。呼び出し元 5 か所のうち、
これを保持するのは `0x014f8c00` だけです（`obj+0x183` へ）。`0x014f808c` は、記述子の形をした 0x136 バイトの領域の `+0x36` に書き込みます。

マッチの種類は、vtable `0x2538238`、`0x2538358`、`0x2538478` のスロット 8（`+0x40`）です。1 がカジュアル（`0x00adda60`）、
2 がランク（`0x00adde60`）、3 がオンライン対戦（`0x00ade340`）です。`0x00adcfb0(obj,
mode)` がこれらを組み立てます。呼び出されるのは `0x00b1cdd0` からだけで、それは `StateBtlSpotTop`（`0x00b23080`）の 6 か所から呼ばれます。
`StateBtlSpotTop` は `StateBtlSpotCasualMatchEntrance`、`StateBtlSpotRankMatchEntrance`、`StateBtlSpotCompTop` のいずれかへ遷移します。
`+0x108` は、`StateBtlSpotCompTop` の `0x00b41b7c` から `0x00adc8b0` が書き込みます（`str x1,[x0,#0x1ab8]`、フラグ `+0x1ab0 = 1`）。
値は、セーブブロック `0x88F6D6AE` の 0x33C0 にある u64 です（0x33D0 バイト、キーは `0x02072fac`、読み取りは `0x01444ac0`。
直前のキー `0xEEE5A3F8` は PKHeX の `KOfficialCompetition` です）。

### プレイヤープロフィール

0xAEC から始まる 266 バイトで、0x1F にあるビーコンのレコードでもあります（[セッション](swsh_session.md#taking-a-seat)）。
`0x01111970` がシングルトン `[0x2610958]`（+0x310 から +0x564、ミューテックスは +0x580）からコピーします。
`0x01125080` はすべてのグループをまとめて詰めるか、何も詰めないので、レイアウトは固定です。ビットパックされたグループは、
下位ビットから順に詰められます。`trade_payload.read_tail` がこれを復号します。

    0x00  16  nn::oe::GetPseudoDeviceId
    0x10  16  nn::account::GetUserId, the account Uid
    0x20   8  nn::account::GetNetworkServiceAccountId, or zero
    0x28  24  trainer name, UTF-16, from MyStatus+0xB0; stale bytes after the terminator
    0x40  25  appearance, bit-packed:
                bit 0      MyStatus+0xA5 (gender) != 0
                bit 1      set by 0x0111cfec; 0 in every capture
                bits 2-5   MyStatus+0xA7, the language (3, French)
                bits 6-7   zero
                8 bits     MyStatus+0xCC
                17 x 10    the model 0x0111dd60 unpacks from the MyStatus bitfield at +0x00
                2, 2, 10   the last three of that unpacking
    0x59  55  position samples, bit-packed:
                2, 2 bits  `a` (sample object +0x47) and `b` (+0x1c9), below
                8 bits     generation: random at ring reset (0x00eb98a8), +1 per re-seed
                8 bits     player object byte +0x136 (7 in every capture)
                3 x 17     at 0x5C, 0x6D, 0x7E, newest first: 5-bit counter (0x01123be0, +1 per
                           push); 3-bit state (+0x1c8, below); x, y, z floats; yaw in radians
                           (Euler component 1, 0x006101c0). 0x00ebf590 pushes at most once a second
                           from the field object's +0x60 (position) and +0x50 (rotation);
                           0x00b4f140 and 0x00b54450 re-push the current sample three times.
                8 bits     at 0x8F: player object byte +0x140 (1 in every capture)
    0x90  37  activity, bit-packed: an 8-bit kind (below; 13 in trade), an optional 28-byte part,
              a 24-byte field, a u16 at 0xB2, a bool; only kind and u16 non-zero. The u16 is the
              location, the line of `script/place_name.dat` (0x00f27d70), PKHeX's met-location
              id (170 Challenge Beach); setter 0x0111bc60, readers 0x0111b8c8, 0x0111b8f4.
    0xB5  37  two more groups (56 and 16 source bytes), zero in every capture
    0xDA  32  sixteen u16 records, `0x010f5060`: Record8 indexes 6, 32, 0, 33, 17, 27, 34, 24,
              12, 3, 10, 35, 38, 7, 36, 37, each clamped to 0xFFFF (PKHeX `RecordList_8`:
              total_capture, evolution, egg_hatching, net_battle, trade, license_trade, cooking,
              campin, pretty, capture_raid, rotomu_circuit, poke_job_return, bike_dash, dress_up,
              get_rare_item, whistle)
    0xFA   8  an optional u64, zero in every capture
    0x102  8  zero

受信側（0x00dd5e24）は、位置を +0xB0 に、yaw をクォータニオンとして +0xA0（0x00992cd0）に格納します。
0x011a68a4 がサンプリングするのは、状態 1、2、3、6 のときだけです。

サンプルの状態は `[[0x261bd18]]` の `+0x1c8` です。設定されるのは `0x00ebf570` だけで、リセット `0x00eb97b0` で 0 になります
（呼び出し元は `0x00dd2e00` と `0x00eb986c` のみ）。0 のあいだ、push は即座に戻ります（`0x00ebf5a8`）:

| 状態 | 設定元 |
|---|---|
| 0 | `0x00dfd1d4`、vtable `0x2561ec8` のスロット 9 |
| 1 | フィールドプレイヤーのスロット 13（`0x00d98f28`）、30（`0x00d9d2a8`）、76（`0x00da0188`）。`0x00d97730` のモーション 0 |
| 2 | `0x00d97730` のモーション 1 または 2。スロット 7（`0x00d97644`、モーション 2 で `[+0x5b8]` が 1 か 2 になるとき、モーション 1）。スロット 75（`0x00d9fcc4`） |
| 3 | vtable `0x253e358` のスロット 15（`0x00b4f028`、`StateCreateSession` のクラス）。`[[obj+0x98]+0x58]` が設定されているとき、vtable `0x25614c0` のスロット 8（`0x00dedf80`） |
| 4 | vtable `0x253e8d0` のスロット 16（`0x00b54338`、`StateConnect` のクラス）。クリアされているときは `0x25614c0` のスロット |
| 5 | なし |
| 6 | `0x00da09a0`（`0x00da0a04`）。ネイティブの `CallRaidBattleMatchingEvent_`（`0x01466d30`、テーブル `0x25aac68`）内の `0x00cebe00`、`0x00cebea4`、`0x01466d90` から |

`0x00d97730(player, motion)` は、キー `[0x261e8a0]` の下で、`player->vtable[0x190]` を介してモーションを格納します。
これはネイティブの `IsPlayerRideBicycleType`（`0x0148b960` -> `0x00da0210`）も読みます。`0x00e57940` の Lua 列挙は、
モーションを `NORMAL`、`BICYCLE_GROUND`、`BICYCLE_WATER` と名付けています（`0x00e5793c` の `1 | 2<<32`）。
`0x25614c0` のスロットは、種類 11 のアクティビティレコードも開始し（`0x0111b660`）、それを `[0x26108d8]`（`0x00fa13c0`）に渡して、
サンプルを push します。`0x25614c0` のオブジェクトは、ポケモンキャンプへの訪問です。ネイティブの `PokeCampToVisit`
（リクエスト `0x0100`、状態 3）と `NpcPokeCampToVisit`（`0x0000`、状態 4）がこれを組み立てます。ネットワーク側のリクエスト
`0x0101`（状態 3）と `0x0001`（状態 4）も同様です。イメージには、マルチプレイのキャンプの `contents.pokecamp.pb.KwSyncData` が含まれます。
状態 6 はマックスレイドバトルのマッチングです。

`a` と `b` は、ビット 0-1 と 2-3 に詰められます（`0x01123710`。`0x01123760` は `b` の 3 を 0 として読みます）。
モード 0 のリセット `0x00eb97b0` は、エリアキー `[[[0x2617c48]]+0x180]`（`0x00eb97d0..0x00eb9848`）から `a` を設定します:

| キー | エリア | `a` |
|---|---|---|
| `0x5742865396e549d0` | `wr0101` | 1 |
| `0x5ea5c3539ab81c81` | `wr0201`、鎧の孤島（場所 170 で捕獲） | 2 |
| `0x674edc539f9f74a6` | `wr0301` | 3 |
| その他 | | 0 |

キーは、`a_wr0101`、`a_wr0201`、`a_wr0301` の FNV-1a 64 です（基底 `0xcbf29ce484222645`、`0x01369700..0x01369724`）。
末尾の 2 文字を反転させた場合も含め、いずれも `0xb8123d750dc24af8` に到達します。
これらは配列 `0x02061f98` にもなっており（`0x00ecd418` でループ）、`0x01c25bbb`、`0x01be5bad`、`0x01c2d3a5` にある nest-hole エミッター名に対応づけられます（`0x00ec51b4..0x00ec5218`）。
`0x00de19c0` は、最初のキーとして `a_wr0101`（`0x01c0ae67`）をコピーします。romfs には `a_wr0201.bnk` と `a_wr0301.bnk` のサウンドバンクがあります。
`b` は `0x00ebf580` が、vtable `0x259add8` のスロット 8（`0x0110e850(...) < 4`、`0x012dfaec`）と、スロット 11（2、`0x012dfca8`）から設定します。どちらの後にも push が続きます。

0x90 にあるアクティビティレコードは `0x49` バイトで、`0x0111b660(rec, kind)` が設定します（kind は `+0` に入り、`+4`、`+0x24..+0x3e`、`+0x40`、`+0x48` はゼロにします）。呼び出し箇所ごとの kind は次のとおりです: 9 は `0x0102437c`、11 は `0x00dedf3c`、`0x0126e37c`、12 は `0x01272120`、18 は `0x01027d38`、19、21、23、25 は `0x01027d2c`（`0x13 + 2n`）、27 は `0x00de7a0c`、28 は `0x00de727c`、29 は `0x01026290`、30 は `0x0109659c`、31 は `0x015d595c`、255 は `0x01027fac`、`0x010962b8`、`0x01097990`、`0x01097bc4`。そして `0x010961b0` は、通信モード `[obj+0x70]` により、`0x2066c40` のテーブルを通じて設定します:

    mode   0    1  2   3   4   5   6
    kind   255  1  13  14  15  16  30

`SetMode`（`0x01096d10`、`str w1,[x0,#0x70]`）は、通信交換（[セッション](swsh_session.md#how-a-searching-sword-finds-a-partner)）から 2 で呼ばれ、0、1、6 でも呼ばれます。

トレーナー名は 4 か所に現れます（MyStatus、トレーナーカード、パーティのレコード、プロフィール）。
`pokeldn/swsh/trade_payload.rewrite` は 4 つすべてを書き換え、交換画面はそのトレーナーの名前を表示します。空のパーティスロットはゼロで、0x810 の件数と一致します。

## PK8

`pokeldn/gen8.py` で BDSP と共有します（PKHeX の `PK8` と `PB8` はどちらも `G8PKM`）。Sword は 0x158 のパーティ形式（`pokeldn/swsh/pokemon.py`）を送り、BDSP は 0x148 の保存形式を送ります。

    0x00  u32  encryption constant, in the clear. Seeds the cipher and the block order
    0x06  u16  checksum, in the clear, over the decrypted body only
    0x08       four 80-byte blocks, LCG-encrypted and permuted by (EC >> 13) & 31
    0x148      the party stats, LCG-encrypted with the stream restarted, never permuted

黙って誤読される箇所が 2 つあります:

- パーティのステータスは LCG を再開します（`PokeCrypto.Decrypt8` は EC から `CryptArray` を 2 回シードします）。
  ストリームを続けて読むと、レベル 110 のような、もっともらしい誤値になります。
- `BLOCK_ORDER[sv]` は、ブロック *i* になるブロックを指定します。適用するだけで、逆変換はしないでください。
  32 個の `sv` のうち 16 個は逆変換が自分自身になり、チェックサムは順序を見ないため、逆向きでも正しく読めてしまうことがよくあります。
  PKHeX の `BlockPosition` の 24-31 は 0-7 と重複しています。

復号したパーティの確認項目: ニックネームが種族と一致すること、レベル（並べ替えない末尾）が経験値と一致すること、
ハイパートレーニング 0x126 が IV の語 0x8C と一致すること、MyStatus のトレーナー ID がすべての PK8 と一致すること。

## 交換メッセージ

`net_contents.trade.common.pokemon_trade.protocol_buffers`:

    Pokemon                  { 1 bytes   serializePokemonParam }
    PokemonTradeDataHolder   { 1 Pokemon pokemon }

オファーは、10050 で 344 バイトの PK8 を運びます（`trade.py` の `pokemon_offer`）。解析 `0x010d81d0` は、0x28 バイトの
メッセージ（`0x010d9c90`）を作り、`ParseFromArray`（`0x0070c180`）を呼んで `[msg+0x18]` を渡します。
`MergePartialFromCodedStream`（`0x010d9ee0`）は、タグ 0x0a だけを受け付けます。リスナーは `[Pokemon+0x18]` を
libc++ の `std::string` として読みます（バイト 0 のビット 0 が +0x10 のヒープポインタを選びます）。0x158 バイトをコピーします。

### 受信ハンドラーで黙って捨てられる 2 つの箇所

`0x010d5e40`:

    w0 = 0x006b5850(senderPointer)                            0xfd on failure
    if (w0 == 0xfd) { [content+0x1a4] = 1; return; }          silent
    memcpy(stack, body, 0x158)
    subscriber = [content + 0x30 + index*8]
    if (subscriber == null || its refcount is 0) return       silent
    ... invoke it, then content 50's send 0x010d6000

コンテンツ 50 の init は `+0x30` と `+0x38` しか埋めないため、局インデックスが 1 を超えると返ります。コンテンツ 40 の
`0x010dbc90` も、`+0x38`/`+0x40` で同じ制限をかけます。コンソールは 10050 に対して、すぐに自分のものを返します。
10050 の返信がないことは、オファーが `0x010d5e40` に届かなかったことを意味します。コンテンツ 30 のスロット（`0x010ce080`）は、
自分の id（`[[0x2616a30]]+0xf0`）だけを拒否します。owner のないオファーは box フェーズを通過し、コンテンツ 50 で失敗します。

### Content 40's message

`net_contents.trade.common.sync_save.protocol_buffers`:

    SyncSaveDataHolder { 1 SyncCommand syncCommand }
    SyncCommand        { 1 int32       data       }

解析 `0x010ddb20`（`0x010d81d0` と対になるもの）は、`0x010df6d0` がタグ 0x0a だけを受け付けます。
サブメッセージのパーサー `0x010debc0` はタグ 0x08 だけを受け付けます。`trade.sync_command` がこれを組み立てます。
