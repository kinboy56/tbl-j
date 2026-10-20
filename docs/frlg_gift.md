---
title: ふしぎなおくりもの
parent: FireRed and LeafGreen
nav_order: 2
---

# ふしぎなおくりもの

ふしぎなおくりもののメニューは、ポケモンセンターを必要としない。ふしぎなカードの画面にあるコンソールは、ふしぎなカード、配信スクリプト、ふしぎなニュース、
訪問してくるバトルタワーのトレーナー、そしてポケモン 1 匹を手持ちへ直接受け取れる。
Mystery Event VM とネイティブの ARM コードは [コンソールでのコード実行](frlg_rom.md) にある。

## セッション

コンソールのクライアントは `{CLI_RECV, MG_LINKID_CLIENT_SCRIPT}, {CLI_COPY_RECV}`
[mystery_gift_scripts.c:15] の設定で起動し、受け取った `MysteryGiftClientCmd` 配列を
何でも実行する [mystery_gift_client.c:87]。命令セット全体を理解するので、ホストは
FireRed の 2 つの ROM サーバースクリプトを超える流れも動かせる。

    console joins  ->  SEND_PLAYER_IDS  ->  LinkPlayer block exchange  ->  one standby barrier
                                                                              |
      server -> CLIENT_SCRIPT (sClientScript_SendGameData, 32 B)
      client -> GAME_DATA     (MysteryGiftLinkGameData, 96 B)   -- validated, card flag compared
      server -> CLIENT_SCRIPT (sClientScript_SaveCard, 48 B)
      server -> CARD          (struct WonderCard, 332 B)
      server -> RAM_SCRIPT    (1024 B: the delivery bytecode, zero-padded)
      client -> READY_END     (1024 B)
                                                                              |
                                              close-link handshake  ->  disconnect

ホストは LinkPlayer ブロックの要求を 1 回出し、コンソールの有効なブロックを待ってから自分のブロックを送り、
その後スタンバイのバリアを待つ。

### フレーミングの規則

サイズ 0 は 1024 を意味する。`MysteryGiftLink_InitSend` [mystery_gift_link.c:55] はこれを
`MG_LINK_BUFFER_SIZE` に展開し、`SVR_COPY_SAVED_RAM_SCRIPT` は `ramScriptSize` を一度も設定しない
[mystery_gift_server.c:275] ため、RAM スクリプトと `CLI_SEND_READY_END` はどちらも 1024 バイトの完全なメッセージになる。
CRC は詰め物を含むバッファ全体を対象にする。

ブロックの送信間隔には確認応答がない。`SEND_BLOCK_INIT` は、受信側のスロットが
`RECV_STATE_READY` [link_rfu_2.c:1146] でない限り無視される。この状態はコンソールの `MGL_ResetReceived` によってのみ戻される。親側の
`MGL_HasReceived` フラグは即座に立つ [link_rfu_2.c:1044]。4 VBlank のカウントダウン
[link_rfu_2.c:1220] は子のブロックにだけ適用される。`MysteryGiftTiming.inter_block_gap_frames` は 36 である。
コンソールのモデルでは、何も落とさなければ 1 ブロックに最大 13 フレームかかり、16 フレームで転送が切れる。
ブロックが 1 つでも落ちると、コンソールは永遠に待ち続ける。コンソールが送信している間の停止は
[ミラー規則](frlg_link.md#row-one-of-the-parents-table-is-the-consoles-own-command-mirrored-back) に従う。

### モジュール

特に断りがない限り、`pokeldn/frlg/gift/` 配下にある:

| ファイル | 役割 |
|---|---|
| `mg_link.py` | フレーミング: 6 バイトの `{ident, crc, size}` ヘッダーブロックと、≤252 バイトのチャンク |
| `mg_script.py` | クライアントスクリプトのアセンブラ、デコンプ由来の定型スクリプト、`MysteryGiftLinkGameData` のリーダー |
| `mg_server.py` | サーバースクリプトのインタプリタ (`SVR_*`) |
| `host_mystery_gift.py` | リーダーエンジン: `tick()` → 親の gSendCmd、`feed_child_slot()` ← 子の行 |
| `host_mg_app.py` | ホストのランタイム上のアプリケーションフック |
| `wonder_card.py` | バイト単位で一致するセレビィと伝説の獣のカード・RAM スクリプトのビルダー |
| `stamp_rally.py` | スタンプラリーのカード、スタンプ、有効化ラッパー、配信スクリプト |
| `gift_composer.py` | アクションの定義、カーソル状態の検証、RAM スクリプトのコンパイラ |
| `gift_registry.py` | カタログ |
| `gift_to_bin.py` | 外部の第 3 世代ふしぎなおくりもののツール向けの `.bin` 書き出し |
| `pokeldn/frlg/save/save_inject.py` | カード・RAM スクリプト・セクタのチェックサムを付けたセーブデータへの書き込み |
| `bin/frlg_mg_host.py` | CLI 本体 |

## 実行方法

    (them) Mystery Gift -> Wonder Cards (Recevoir) -> Friend (Ami), wait on the search screen
    (you)  POKELDN_RADIO=esp32:auto ./.venv/bin/python -u bin/frlg_mg_host.py --live --phy auto \
               --keys PROD_KEYS --gift beast-cutscene --flag-id 1005
    (them) join the host when it appears; YES on the replace-card prompt if one shows

無線の設定: [ESP32 無線](hardware_esp32.md)。`bin/frlg_mg_host.py` は 1 回の実行で 1 台のコンソールに応答し、
そのコンソールが LDN から離脱すると停止する ([ホストの実装](frlg_host.md)、終了とクリーンアップの節)。2 台目のコンソールには新しい実行が必要である。
`tests/test_mystery_gift_flow.py` はブロック受信のゲート、`MGL_Receive`、
フレームごとのクライアントコマンド 1 つをモデル化する。`tests/test_mystery_gift_end_to_end.py` は、障害を入れた
Reliable/RFU の経路を追加する。

## リンクで運べるもの

クライアント命令 22 個のうち 3 つだけが、何かを実行する [include/mystery_gift_client.h:18]:

| 命令 | コンソールがすること |
|---|---|
| `CLI_SAVE_RAM_SCRIPT` (17) | フィールドスクリプトを保存する。次に NPC と話したときに実行される |
| `CLI_RUN_MEVENT_SCRIPT` (15) | Mystery Event のバイトコードスクリプトを実行する。独自の 17 オペコードのテーブルを持つ 2 つ目の VM |
| `CLI_RUN_BUFFER_SCRIPT` (21) | `func = (void *)gDecompressionBuffer; func(&param, gSaveBlock2Ptr, gSaveBlock1Ptr)`。両方のセーブブロックのポインタを渡して、最大 1024 バイトの ARM を実行する |

## RAM スクリプトの枠は 1 つだけ

コンソールが保持できるのは、ふしぎなカードか紐づいた RAM スクリプトのどちらか一方だけで、両方は持てない。`ValidateSavedWonderCard` はカードの CRC を、
`ValidateWonderCard` はカードそのもの、次に `ValidateRamScript` [mystery_gift.c:186] を検査する。後者は、
`magic == RAM_SCRIPT_MAGIC`、マップグループと番号が `MAP_UNDEFINED`、`objectId == 0xFF` であることを要求する
[script.c:538]。フィールドでは `GetRamScript(gSpecialVar_LastTalked, script)` を通して RAM スクリプトを実行し
[field_control_avatar.c:458]、話しかけたオブジェクトの座標が必要になる。
`CLI_SAVE_RAM_SCRIPT` は `InitRamScript_NoObjectEvent` (MAP_UNDEFINED、0xFF [script.c:578]) を呼ぶ。
Mystery Event VM の `initramscript` は実際の座標を書き込む。紐づいたスクリプトがあると、カードはセーブデータに正しい CRC のまま残るが、
メニューからは隠され、`MysteryGift_LoadLinkGameData` は `flagId` 0 を返す
[mystery_gift.c:349] (`HAS_NO_CARD`)。

次のふしぎなカードが枠を再びバインドし (`magic` は 51 のまま、座標は 0xFF)、カードが戻ってくる。バッファスクリプトはカードを送らず、枠にも触れない。紐づいたスクリプトの上に通常のカードを配信すると、
+0x32E0 のカードと +0x361C の RAM スクリプトが、どちらも 1 つのセーブセクタ内で書き換わる。
SaveBlock2 には何も書き込まれない。実測では、SaveBlock1 の 15872 バイトのうち 564 バイトが変わった。

枠のチェックサム (`ramScript.checksum`、SaveBlock1 + 0x361C) は、`sizeof(RamScriptData)` の範囲に対する `CalcCRC16WithTable` である。
その大きさは 1000 バイトで、宣言された 999 バイトと、`InitRamScript` が先にゼロクリアする詰め物 1 バイトから成る [script.c:500]。
`CalculateRamScriptChecksum` は `250 << 2` を渡す (BPRE は `0x0806D43C`、BPRF は `0x0806D5A0`)。ゲームが書いた枠は、読み戻すと必ず 1000 バイトの CRC を持つ。
CRC が 999 バイトしか覆わない枠は、
オブジェクトに初めて話しかけたときに `GetRamScript` によって消される
[script.c:526]。

## ギフトのカタログ

`--gift NAME`。`--help` でフラグ ID (1000..1019) を一覧できる。重要なのは手元にあるカードだけで、同じ ID は
「このカードを既に持っている」を意味する (`MysteryGift_CompareCardFlags`)。

| ギフト | 内容 |
|---|---|
| `beast-cutscene-share` | 繰り返し見られる伝説の獣のカットシーン。既定のデモカード |
| `celebi` | 組み立てたレベル 50 のセレビィのカード |
| `porygon-tm-gift` | ポリゴンのカード、ピッピのシーン、TM29 サイコキネシス、続いて TM46 どろぼう |
| `solrock-stamp` / `lunatone-stamp` | 1 枚のスタンプラリーカードを 2 つに分けた各半分 |
| `altering-cave` | 公式の「かわりのどうくつ」イベントを移植したもの |
| `wish-egg`, `pokepark-egg`, `pc-japan-egg` | 公式の配布タマゴ。「配布タマゴ」の節を参照 |
| `event-pokemon` | 第 3 世代の配布ポケモンを手持ちに直接加える。「配布ポケモン」の節を参照 |
| `starter-egg` | 最初のパートナー 9 匹のいずれかのタマゴ。`random` で選ばれる |
| `rare-berries` | ナゾのみ、ランサのみ、スターのみを 1 段階ずつ |
| `national-dex` | `IsNationalPokedexEnabled` (403) が 1 を返さない限り `EnableNationalPokedex` (特殊 367) |
| `nature-mint`, `pc-anywhere` ほか 42 種 | GB-Link Team のカード。「GB-Link Team のカード」の節を参照 |
| `battle-count-card` | 公式のバトルカウントカード |
| `visiting-trainer` | バトルタワーのトレーナーを ident 26 として (FireRed のみ) |
| `mystery-event-probe` | `givenationaldex; setstatus 42; checksum`。VM 自身の自己テスト |
| `mystery-event-celebi` | `givepokemon`。メールを持った Lv30 のセレビィを手持ちに直接加える |
| `mystery-event-npc` | `initramscript`。フィールドスクリプトをマサラタウンのふとったおじさんに紐づける |
| `rng-seed-reader`, `rng-rate-probe`, `rng-shiny-hunt`, `rng-mon-hunt`, `rng-mon-hunt-both`, `rng-mon-hunt-log` | [乱数](frlg_rng.md) を参照 |

すでにそのカードを持っているコンソールは、確認なしで Mystery Event のギフトを単独で受け取る。

### 伝説の獣

配達員のカットシーンでランサのみとチイラのみを渡し、続いてマスターボールを渡す。
その後、セーブデータの最初のポケモンによって選ばれる Lv65 の伝説の獣との野生バトルを始める:

| 最初のポケモン | 伝説の獣 |
|---|---|
| フシギダネ | スイクン |
| ゼニガメ | エンテイ |
| ヒトカゲ | ライコウ |

保存されたスクリプトは残るので、イベントは繰り返される。

### ポリゴンの TM ギフト

ポリゴンのアイコンのカード。プレイヤーの右 3 マス先で西を向いたピッピのスプライトと、TM29 サイコキネシス、
続いて TM46 どろぼう。どろぼうを再試行してもサイコキネシスが重複しないよう、配信のチェックポイントは分けてある。
既定のフラグ ID は 1007。ビューアーでは右上に `7` と表示される (`flag_id % 100`)。

### スタンプラリー

2 つのイベントは 1 枚の `SUN AND MOON RALLY` カードを共有する (ドータクンのアイコン、スタンプ枠 2 つ、表示番号 `6`、
フラグ ID 1006)。どちらの順番で受け取ってもよい。

| 状態 | 意味 |
|---|---|
| `VAR_MYSTERY_GIFT_1` | セレビィの完了カーソル |
| `VAR_MYSTERY_GIFT_2 = 0/1/2` | ソルロックの不在 / 有効 / 受領済み |
| `VAR_MYSTERY_GIFT_3 = 0/1/2` | ルナトーンの不在 / 有効 / 受領済み |
| `FLAG_MYSTERY_GIFT_DONE` | ラリーの完了 |
| カード受領フラグ | セレビィの配信が成功したときに同期される |

スタンプ 1 つでレベル 30 のソルロックかルナトーンがもらえ、両方そろうとレベル 50 のセレビィがもらえる。いずれも単純な `givemon` で渡される。
手持ちとパソコンの空きを数える。両方が満杯なら何も進まない。保留中のスタンプが 2 つあると、3 匹すべてを一度に配信する。

```text
matching =
    saved flag ID == distribution flag ID
    and max stamps == 2
    and metadata icon species == CLAYDOL

if no card:                install shared card + delivery script + selected stamp, run activation
else if not matching:      offer the toss prompt; on accept, install as above
else if stamp species or ID already exists:   HAS_STAMP, no activation
else if neither slot empty:                   NO_ROOM_STAMPS, no activation
else:                      save the stamp, run activation, STAMP_RECEIVED
```

有効化は Mystery Event のラッパー (`runscript` と埋め込みのフィールドスクリプト) で行う。スタンプは
ライブのホストでのみ動く。`IsStampInMetadata` [mystery_gift.c:272] は、ID か種族が
衝突するスタンプを拒否する (最大 7 個)。`CLI_SAVE_STAMP` は `cardMetadata.stampData` [mystery_gift.c:307] だけを書き込み、
`CLI_SAVE_CARD` によるカードの消去を避ける。

### かわりのどうくつ

公式のスクリプト [data/mystery_event_msg.s:325] は `addvar VAR_ALTERING_CAVE_WILD_SET, 1` を実行し、
10 で巻き戻し [:328]、メッセージを出す。`end` で終わるので、話しかけるたびに 1 セットずつ進む。読み取り側は 9 以上を
テーブル 0 に丸める [wild_encounter.c:192]。変数 (0x4024) は SaveBlock1 + 0x1048 にある:

    --buffer-script save-dump --dump-block sav1 --dump-offset 0x1048 --dump-size 2

3 回話しかけると変数が 3 になり、GROTTE METAMO (Six Island) はテーブル 3
`sSixIslandAlteringCave_4_FireRed` [src/data/wild_encounters.json] から抽選する (市販の
FireRed では Lv16 のデルビル)。

| 変数 | ポケモン | | 変数 | ポケモン |
|---|---|---|---|---|
| 0 | ズバット | | 5 | エイパム |
| 1 | メリープ | | 6 | ツボツボ |
| 2 | ポットデス | | 7 | オドシシ |
| 3 | デルビル | | 8 | ドーブル |
| 4 | ヒメグマ | | | |

### 配布タマゴ

日本で配布された 3 種。GB-Link-Switch-LDN が持つバイト列から移植した (`web/js/gift/official.js`)。
各カードは、その配布のタマゴすべてを保持し、コンソールは `random` [scrcmd.c:455] で 1 つを選ぶ。
タマゴには、配布の 4 つの技、運命の出会いのビット、出会った場所 0xFF が、元のスクリプトどおりに設定される。
手持ちが満杯だと抽選の前に拒否され、カードは開いたままになる。

| カード | タマゴ |
|---|---|
| `wish-egg` (Pokemon Center New York) | ラッキー、スリープ、タマタマ、カモネギ、ガルーラ、ベロリンガ。いずれも「ねがいごと」を覚えている |
| `pokepark-egg` (PokePark Market Fantasia) | サボネア、ヘイガニ、サニーゴ、ププリン、マイナン、ピチュー、プラスル、コダック、エネコ、パッチール、ブーピッグ、アメタマ、スバメ、ゴニョニョ、ソーナノ |
| `pc-japan-egg` (Pokemon Center Japan) | マダツボミ (ふらふらダンス)、ニャース (はなびらのまい)、クサイハナ (やどりぎのたね)、ニョロモ (あまいキッス) |

概要画面は、出会った場所が 0xFF の場合か、運命の出会いのビットがある場合に
"Drôle d'ŒUF de POKéMON obtenu dans un bel endroit." と表示する [pokemon_summary_screen.c:2799]。

### 配布ポケモン

`--gift event-pokemon --event-pokemon NAME` は第 3 世代の配布ポケモンを新しく 1 匹送る。PKHeX.Core が
イベントテーブル (`EncounterGift3`、タマゴと日本版以外の項目) から、そのイベントの PID / IV の方式で作る。
トレーナー名、トレーナー ID、レベル、技、持ち物、リボン、
運命の出会いのビットを含み、適法性の検査に通らなければならない。レコードは、`mystery-event-celebi` と同じく、カードを保存した瞬間に
Mystery Event の `givepokemon` で手持ちに入る。手持ちが満杯だと
ステータス 3 を返して何も受け取れず、カードは再び受け取れる。`--event-pokemon` を付けないと、
カードは保存されている WISHMKR Jirachi を送る。

`NAME` はトレーナー名、半角スペース、種族名である: `WISHMKR Jirachi`、`CHANNEL Jirachi`、
`Aura Mew`、`MYSTRY Mew`、`DOEL Deoxys`、`SPACE C Deoxys`、`ROCKS Metang`、`10 ANIV Pikachu`、および
`10 ANIV` のその他の種族、欧州版の `10ANNIV`、`10JAHRE`、`10ANNI`、`10ANIV` の配信。複数の言語で配信されたイベントでは、
`--language` に一致する言語のものが送られる。

このカードはビルドに依存しない。Mystery Event VM の 17 項目のテーブル、その `givepokemon`、および
ギフトクライアントの 23 ケースの `Client_Run` スイッチは、12 本すべてのカートリッジで同じ配置を持つ。
`tests/test_frlg_english_cartridges.py` は、カードのスクリプトを各カートリッジ自身の
`RunMysteryEventScript` に通す (ステータス 2 で手持ちにレコードが入る。満杯ならステータス 3 で何も入らない)。

### GB-Link Team のカード

GB-Link Team のオリジナルのふしぎなカード (GB-Link-Switch-LDN の `cards/`、GPL-3.0) は、ふしぎなカードに
THUMB コードを載せた配達員の RAM スクリプトを加えたもので、`callnative` 経由で呼ばれる。ARM のソースは
`vendor/gblink-cards/` にある。`scripts/gen_team_cards.py` が 5 本のカートリッジ向けにこれをアセンブルし、
`pokeldn/frlg/data/team_cards.json` に書き出す。`pokeldn/frlg/gift/team_cards.py` は各カードを、その
`custom-` を外した ID で登録する (`--gift nature-mint`)。元のソースと RAM アドレスのまま、
ジェネレーターは元の `BPRE 1.10` ペイロードを、44 個すべてバイト単位で再現する。

`starter-egg`、`rare-berries`、`national-dex` は、ネイティブコードを必要としない 3 枚で、コンポーザーで組み直している。
木の実はアイテム 173、174、175 で、ぜんこくずかんのカードは
`FLAG_SYS_NATIONAL_DEX` (0x840) を立てる。配布ポケモンは PKHeX から取り
(「配布ポケモン」の節を参照)、PKHeX の表にない 4 匹を除く。フォロワー、マスターボール、速度アップ、
出会いのフックは、このプロジェクト独自のものでカバーしている。

| 分類 | カード |
|---|---|
| ポケモンを変える | `nature-mint`, `ability-capsule`, `poke-ball-changer`, `pokemon-gender`, `nickname`, `stat-judge`, `hidden-power`, `hidden-power-type`, `ev-training`, `friendship`, `pp-max`, `max-conditions`, `pokerus`, `unown-letters`, `trade-evolution`, `espeon-umbreon`, `move-tutor` |
| フレームごとのフック | `speed-2`, `speed-3`, `speed-4`, `speed-0-75`, `speed-0-5`, `fast-text`, `travel-anywhere`, `pc-anywhere`, `hm-moves`, `reusable-tms`, `physical-special-split`, `exp-share`, `shiny-hunting`, `roamer` |
| その他 | `no-encounters`, `legendary-respawn`, `instant-eggs`, `gift-box`, `pocket-casino`, `gift-ribbons`, `trainer-ids`, `gender-swap`, `rival-name` |
| 配布ポケモン | `box-eggs`, `colosseum-pikachu`, `ageto-celebi`, `mattle-ho-oh` |

元のビルドとの違い:

- リビジョン `0x0A` のカートリッジ 12 本すべて。ソースが使う 167 個のアドレスは、英語、フランス語、ドイツ語、
  イタリア語、スペイン語、日本語の各 FireRed / LeafGreen ROM で測定した。関数は一意な命令の並び、RAM とポインタを持つデータは
  リテラルプール、フィールドスクリプトのラベルは、ポインタをマスクしたコマンド列で求めている。`vendor/gblink-cards/symbols.json` に
  12 本分の表がすべて入っており、`tests/test_team_cards.py` は全カートリッジで 25 項目を `builds.py` と照合する。
  各スクリプトは、ヘッダーのゲーム文字、言語文字、リビジョンを検査する。
  [カートリッジのマップ](frlg_rom_map.md#the-international-revision-0x0a-cartridges) を参照。
- 移動したスクリプト (996 バイト) とメニューの一覧 (80 バイト) は、常駐フックが動く `0x0203FC00` ではなく、
  `0x0203F768` と `0x0203FB50` に置く。これらは newlib の malloc の状態の領域である。
  詳細は [ペイロードの置き場所](frlg_rom.md#where-a-payload-can-live) を参照。
- フックカードのインストーラーは、コピーの前に `gIntrTable[4]` を `VBlankIntr` に向け、
  見つけたハンドラーではなくそれに連鎖させ、`0x0203FBFC` に格納する。このプロジェクトの常駐インストーラーはそこを見に来る。
  フックカードは動いているゲームのブーストを置き換え、その逆も行う。どちらも古いコピーには連鎖しない。
  状態は `0x0203FF60` にあり、コピーはその下で終わる。
- 1019 より大きい ID には `sReceivedGiftFlags` のビットがない。そのため、
  レジストリは 1000 + (カードの ID 番号 mod 20) を送る。
- 2 つの文章を短くした (`hm-moves` は 4 文字、`physical-special-split` は 6 文字)。インストーラーの変更後も
  995 バイトに収めるためである。

すべてのカードを、12 本すべてのカートリッジで mGBA 上でママに紐づけて動作確認した。
この確認は、入場、メッセージ、メニューを対象にしており、各カードの選択肢すべてを網羅するものではない。`nature-mint`、`pc-anywhere`、
`rival-name` は市販のフランス語版 FireRed でも動いた。別のゲームのカートリッジに送ったペイロードは、
"This gift doesn't work with this version of the game." (日本語版では "Wrong game.") と答える。常駐フックが動いているとき、
`nature-mint` は `0x0203FC00..0x02040000` に触れず、`pc-anywhere` は `gIntrTable[4]` を引き継ぎ、
`0x0800071D` は `0x0203FBFC` に保たれる。

`colosseum-pikachu` と `ageto-celebi` は日本語のトレーナー名を持つため、欧州版のカートリッジでは
それが点の並びとして描画される。PKHeX は配布ポケモン 4 匹すべてを適法と判定する。

### バトルカウントカード

`MysteryEventScript_BattleCard` [data/mystery_event_msg.s:162] は `GET_CARD_BATTLES_WON` を
`GetMysteryGiftCardStat` (特殊 390) 経由で読み、ちょうど 3 回のときにキズぐすりを渡す。移植版は、公式の `FLAG_MYSTERY_GIFT_DONE` のゲートを
独自の景品変数に置き換えており、繰り返し受け取れる。

相手が集計を有効にする。`Task_ExchangeCards` は `MysteryGift_TryEnableStatsByFlagId` を、
`BLOCK_REQ_SIZE_100` バッファで 96 バイトのトレーナーカードの直後にある u16 が手持ちのカードのフラグ ID と一致する場合に限り有効にする
[union_room.c:1777]。これは交換センターかコロシアムに入ったとき (`frlg_trade_host.py
--card-flag-id N`) に行われる。

| 増える値 | 場所 | 規則 |
|---|---|---|
| `numTrades` | 完了した交換 [trade_scene.c:2609] | カードがまだ数えていないトレーナー ID |
| `battlesWon` / `battlesLost` | ケーブルクラブのバトル終了 [cable_club.c:792] | 同じ。統計ごとに 5 個の ID を記憶する |

`IncrementCardStatForNewTrainer` [mystery_gift.c:630] は、各トレーナー ID を 1 回だけ数える。ユニオンルームの
バトルは `CB2_ReturnToField` を経由して戻り、何も数えない。数えるのは
[コロシアム](frlg_link.md#the-cable-club-colosseum) だけである。集計値は SaveBlock1 + 0x3434
(`buffer_script.SAV1_CARD_METADATA`) にあり、CRC 検査なしで読まれる [mystery_gift.c:490]:

    0x3434: 0000 0000 0000 2300     battlesWon 0, lost 0, trades 0, icon 35 (CARD_TYPE_GIFT)
    0x3434: 0000 0000 0100 2300     trades 1                       (CARD_TYPE_LINK_STAT)

`CARD_TYPE_GIFT` のカードは、有効化されていてもゼロのままである。ワイヤレスクラブの交換センターは
`union_room.c` の `Task_StartActivity` を通る。オフセット 96 にフラグ ID を書き込む唯一のビルダーである。

### 訪問してくるトレーナー

`CLI_RECV_EREADER_TRAINER` (18、ident `MG_LINKID_EREADER_TRAINER` = 26) はバッファを
`gSaveBlock2Ptr->battleTower.ereaderTrainer` にコピーし、`ValidateEReaderTrainer` を呼ぶ
[mystery_gift_client.c:233]。この構造体は 188 バイトである [global.h:286]:

    0x00 u8  unk0                  0x10 u16 greeting[6]            0x34 BattleTowerPokemon party[3]
    0x01 u8  trainerClass          0x1C u16 farewellPlayerLost[6]  0xB8 u32 checksum
    0x02 u16 winStreak             0x28 u16 farewellPlayerWon[6]
    0x04 u8  name[8]
    0x0C u8  trainerId[4]

検証: 先頭の 46 ワードがすべてゼロでないこと、末尾の u32 がそれらの合計であること [battle_tower.c:1354, :1384]。
失敗した場合は黙って消去される。`SevenIsland_House_Room1` はこれだけで判定し、おばあさんが
2 号室で 3 対 3 のバトルを持ちかける。このバトルは構造体から `CreateBattleTowerMon` で作られ [battle_tower.c:928]、
その後回復し、繰り返し挑戦できる。レベルのルールと禁止リスト (`ShouldBattleEReaderTrainer` [:232]) は、ここでは一切呼ばれない。

`CreateBattleTowerMon` は、種族、持ち物、4 つの技 (PP は技テーブルから)、レベル、ppBonuses、
EV、IV、abilityNum、otId、性格、ニックネーム、なつき度を設定する。台詞は Easy Chat の語 6 語で、
`farewellPlayerWon` はプレイヤーが勝ったときに言われる。FRLG は名前の 8 バイトのうち 5 バイトを表示する
[`CopyEReaderTrainerName5`, battle_tower.c:1343]。`CLI_MSG_TRAINER_RECEIVED` (12) [strings.c:1296] は
成功として扱われるので、コンソールはセーブする。

`--gift visiting-trainer` は、1 回のセッションでカード、RAM スクリプト、トレーナーを送る。カードがない場合は 3 つすべて、
同じカードならトレーナーだけで、捨てる確認は出ない。別のカードなら捨てる確認を出し、その後 3 つすべてを送る。

## ふしぎなニュース

ふしぎなおくりもののメニューは、{ふしぎなカード, ふしぎなニュース} × {ワイヤレス通信, ともだち} である。
`struct WonderNews` [global.h:646] は 444 バイトで、送り主を特定する情報を持たない:

| オフセット | サイズ | フィールド | 備考 |
|---|---|---|---|
| 0x000 | 2 | `id` | `ValidateWonderNews` が検査する唯一の項目で、0 であってはならない [mystery_gift.c:113] |
| 0x002 | 1 | `sendType` | `SEND_TYPE_DISALLOWED` は、コンソール自身の「送る」の選択肢を隠す [mystery_gift.c:120] |
| 0x003 | 1 | `bgType` | 検査されない。`WonderNews_Init` は `>= NUM_WONDER_BGS` を 0 に丸める [mystery_gift_show_news.c:110] |
| 0x004 | 40 | `titleText` | centred in a 224 px window |
| 0x02C | 400 | `bodyText[10][40]` | 画面には 8 行が表示される。インデックス 7 を超える空でない行があると、スクロールインジケーターが有効になる [:346] |

ニュースには `flagId`、メタデータ、RAM スクリプト、受領フラグがなく、
`sReceivedGiftFlags` を参照することもない。`IsWonderNewsSameAsSaved` [mystery_gift.c:140] は 444 バイトすべてを比較するので、
1 バイトでも変わると古いニュースが新しいものになる (`--news-id N`)。ふしぎなカードのホストと比べると次のとおり:

- ニュースの受け入れリストは 1 つのアクティビティを持つ [`sAcceptedActivityIds_WonderNews`,
  src/data/union_room.h:406]。`build_wonder_news_app_data` は 21 ではなく 22 を広告する。`hasNews` ビットは
  ワイヤレスの経路でのみ意味を持つ [union_room.c:3777]。
- コンソールは `MG_LINKID_RESPONSE` (ident 19) [mystery_gift_client.c:210] で応答する: `FALSE` は
  保存済み、`TRUE` は既に持っている、の意味。`sServerScript_SendNews` [mystery_gift_scripts.c:126] は `TRUE` のとき
  `SVR_MSG_HAS_NEWS` で終わり、それ以外では `sClientScript_NewsReceived` を実行する。これは保存し、
  報酬を設定する。`SCRIPT_SEND_WONDER_NEWS` は、先頭の `SVR_COPY_SAVED_NEWS` を省く。
- カードの検査、捨てる確認、RAM スクリプトはない。ニュースとカードが互いを押し出すことはない。

ともだちから届くニュースは、`ITEM_RAZZ_BERRY` と `ITEM_NOMEL_BERRY` の間で木の実を 1 つ抽選する
[mystery_gift_menu.c:1367, wonder_news.c:21]。`CeruleanCity_House4` の男性が渡す。最大 5 回まで、
その後は 500 歩ごと [`MAX_REWARD`]。木の実 4 つの報酬には、閉ざされた経路である `WONDER_NEWS_RECV_WIRELESS` が必要である。

`--news` (`--news berry`、`--news-id N`)。プレイヤーはふしぎなニュースを選び、"input one?"、ともだち (
ニュースを持つコンソールはそれを表示する。A を押してから受け取る)。1 回のセッションのメッセージ順 (約 18 秒):

    ident 16  sClientScript_SendGameData
    ident 17  MysteryGiftLinkGameData
    ident 16  sClientScript_SaveNews
    ident 23  MG_LINKID_NEWS - 444 bytes in three blocks
    ident 19  MG_LINKID_RESPONSE - FALSE: the console saved it
    ident 16  sClientScript_NewsReceived
    ident 20  READY_END                              -> SVR_MSG_NEWS_SENT

## パスワードの門としてのアンケート

`SVR_CHECK_QUESTIONNAIRE` は、4 つのポケモンマートのアンケートの語を順に
[`MysteryGift_DoesQuestionnaireMatch`, mystery_gift.c:422] と比べ、`SVR_GOTO_IF_EQ` 用の `param` に結果を入れる。ROM の
サーバースクリプトはどれもこれを使わない。`mg_server.gate_on_questionnaire(script)` は、それをゲームデータの
接頭部の後ろに挿入する:

    MysteryGiftServer(card, ram_script, questionnaire=phrase, denied_message="Say the words.")
    bin/frlg_mg_host.py --gift ... --questionnaire species:55,FEELINGS/60,move:177,why

語は英語の名前、`species:N`、`move:N`、`GROUP/INDEX`、または生の ID でよい。誤った語句が来ると、
`CLIENT_SCRIPT_DYNAMIC_ERROR` と `SVR_MSG_NOTHING_SENT` を通して、ホストの 64 バイトのメッセージが返る。何も
送信されず、捨てられもしない。まず拒否を確かめること。通るゲートは、配線されていないゲートと同じに見える。

フランス語の語の ID は、コンソールから読み取る。どのセッションも 4 つの語を
`MysteryGiftLinkGameData` [mystery_gift.c:361] に含めて送るので、ホストはそれを記録する。

    questionnaire: POKEMON/55  done [FEELINGS/60]  MOVE_1/177  why [MISC/37]

AKWAKWAK FURAX AEROBLAST POURQUOI の場合: `EC_GROUP_POKEMON` は種族で索引し (ゴルダック、55)、
`EC_GROUP_MOVE_1` は技 ID で索引する (エアロブラスト、177)。英語の表は MISC/37 については正しく、
FEELINGS/60 については誤っている。[フランス語のかんたん会話の語彙](frlg_rom_map.md#the-french-easy-chat-vocabulary) を参照。

## コンソールが自ら明かす情報

どのセッションの `MysteryGiftLinkGameData` も、かんたん会話のプロフィールとカードの統計
(`CARD_STAT_BATTLES_WON` / `_LOST` / `_NUM_TRADES` / `_NUM_STAMPS`) を含む [mystery_gift.c:361]。
`--game-data-log PATH` (`pokeldn/frlg/gift/game_data_log.py`) は、各セッションを JSONL の台帳に追記し、
そのコンソールの前回からの変化を表示する。`tools/frlg/game_data_read.py PATH` で読める。統計値は、
同じカードのフラグ ID 上の差分としてのみ証拠になる。台帳には、
フランス語のかんたん会話の表にない語 ID がすべて記録される。

## セーブデータのバックアップと復元

ふしぎなカード、ともだちのセッションは、コンソールのセーブチップ 128 KiB 全体をホストにコピーするか、
`.sav` をそこに書き込んでゲームに読み込ませ、保存させる。カードは送られず、置き換えられることもない。2 つの
ペイロード `asm/save-backup.s` と `asm/save-restore.s` は、GB-Link Team の
`cards/savebackup.s` と `cards/saverestore.s` (`GB-Link/GB-Link-Switch-LDN`、GPL-3.0) から移植したもの。ホストは
`pokeldn.frlg.gift.save_transfer` と `bin/frlg_mg_host.py --save-backup FILE` / `--save-restore FILE`。
アプリは両方を、ふしぎなおくりもののツールの Your save タブから実行する ([Your saves](gui.md#your-saves))。

どちらも 12 本すべてのカートリッジで動く。ビルドのアドレスを 2 つペイロードにパッチする。セーブの配置の残りは
共通である ([セーブデータのバックアップと復元](frlg_rom_map.md#save-backup-and-restore))。

### トークン符号化

両方向とも、チップのバイトをトークンとして運ぶ。`n < 0x80` のバイトの後ろには `n + 1` 個のリテラルバイトが続き、
`n >= 0x80` のバイトの後ろには 1 バイトが `n - 0x80 + 3` 回繰り返される。3 つ以上の連続は丸ごと取られ、
最大 130 まで。リテラルの連続は最大 128 である。セーブデータは大部分が `0x00` と `0xFF` の連続である。

### バックアップ

クライアントスクリプトは、`CLI_LOAD_TOSS_RESPONSE, CLI_RUN_BUFFER_SCRIPT, CLI_SEND_LOADED` を 1 つのスクリプトにつき最大 32 回繰り返し、
次のスクリプトを要求する。ホストは、これまでのペースで残りに必要な回数に 1 を加えたパスを送る。
各パスは、チップの最大 8 KiB 分について最大 1 KiB のトークンを送り、
64 KiB のバンク境界をまたぐことはない。チップのオフセットは `client->param` に
`0x5A << 24 | offset` の形で入る [mystery_gift_client.c:276]。`0x5A` を含まない `param` はセッションの最初のパスであり、
ヘッダーのワード `first` から始まる。

| ペイロードのワード | オフセット | 値 |
|---|---|---|
| `first` | `0x004` | 最初のパスが始まるチップのオフセット |
| `send_queue` | `0x008` | `&gRfu.sendQueue.count` |

パスは、その区間を `gDecompressionBuffer + 0x800` に、1 フレームあたり `0x800` バイトずつステージし (0 を返す)、
その後 `gDecompressionBuffer + 0x400` に圧縮して、`link.sendBuffer` (`param + 0x3C`) と
`link.sendSize` (`param + 0x34`) をメッセージに向ける。`gRfu.sendQueue.count` が
0 でない間、パスは 0 を返す。失われた断片は、各フレームの送信の上に再度キューへ入れられ、40 命令のキューは
新しい送信がない間だけ空になる。

セッションは、64 バイトのメッセージの後に `CLI_MSG_BUFFER_FAILURE` で終わるので、コンソールはそれを表示し、
保存しない [mystery_gift_menu.c:1379]。途中で切れたバックアップは、ゲームコードとトレーナー ID ごとにホストが保持し、
そのコンソールの次のバックアップは、`first` を中断した位置に設定する。

### 復元

1. 最初のメッセージは 620 バイトのイメージ全体である。`install` はそれを `gDecompressionBuffer + 0x400` にコピーし、
   各メッセージが上書きする 1 KiB の先へ置く。そして 28 個のスロットセクタの 12 バイトのフッター (ID、チェックサム、
   シグネチャ、`+0xFF4` のカウンター) で応答する。
2. ホストは、`GetSaveValidStatus` と同じように、フッターからチップの最新の完全なスロットを探し、
   書き込みを計画する。ファイルのロード済みコピーを、カウンター `newest + 1` で他方のスロットへ入れ、
   その後、セクタ 28 から 31 (殿堂、トレーナータワー) をファイルが持つとおりに書く。
3. 置き換え中のスロットは、カウンターに関係なく、その 14 個の ID が通る間はまだ読み込まれる。そのため、書きかけのスロットが
   読み込まれるおそれがある。計画では、そのスロットの ID 0 のセクタを先に消去し、新しい ID 0 を最後に書く。
   コピーが完全になるまでスロットには ID がなく、チップ自身のコピーが読み込まれる。
4. 後のメッセージはそれぞれ `b RESIDENT + 4` (`0xEA0000FF`) で始まり、続いて命令が来る。`OP_DATA` (1) は
   セクタ、書き込みフラグ、u16 のオフセット、u16 のトークン長、4 KiB のステージングバッファを埋めるトークンを運ぶ。
   セクタの最後のメッセージは、それを `swi 0x48` で書き込み、窓を通して読み戻す。内容が違うセクタや、
   バッファを溢れさせるトークンは、失敗マスクのビットを立てる。
   ホストは、次のセクタの前にその報告を待つ。
5. `OP_FINISH` (2) は、どのセクタも失敗しなかった場合に `LoadGameSave(SAVE_NORMAL)` を呼び [save.c:803]、
   失敗マスクとロード結果を報告する。`SAVE_STATUS_OK` なら
   `CLI_MSG_BUFFER_SUCCESS` で終わり、コンソールは、ロードしたゲームを古いコピーが入っていたスロットに保存する。
   それ以外は `CLI_MSG_BUFFER_FAILURE` で終わり、コンソールは持っていたセーブデータを保つ。

| ペイロードのワード | オフセット | 値 |
|---|---|---|
| `b install` | `0x000` | 最初のメッセージの入口 |
| `b entry` | `0x004` | 後のすべてのメッセージの入口。`gDecompressionBuffer + 0x404` にある |
| `load_game_save` | `0x008` | `LoadGameSave \| 1` |

ホストは、ロード済みコピーが完全でないセーブと、別のレイアウトのセーブを拒否する。日本語版の
SaveBlock1 は 40 バイト短く、ゼロで埋められたセクタではそれがチェックサムに表れないため、レイアウトは
プレイヤー自身のポケモン (OT ID がトレーナー本人のもの) の言語バイト (`+0x12`) から読む。
拒否した場合は何も書き込まない。

### 測定結果

オフラインで、スクリプトで動くコンソールに対して (`tests/test_save_transfer.py`) 確認した。バックアップは、フランス語版 FireRed、英語版 LeafGreen、日本語版 FireRed の
チップをバイト単位で返す。40 KB の後で切れたバックアップは再開して完了する。復元では、ファイルのロード済みコピーがチップの最新になり、
余分なセクタはファイルのものと一致し、コンソールの古いコピーは完全なまま残る。8 セクタの後で切れた復元は、
コンソール自身のコピーの読み込みを残す。

ESP32 ボード経由で市販のフランス語版 FireRed に対して行ったバックアップは、最初のパスから最後のブロックまで 64 パス、219 秒かかった。ファイルの両方のスロットは完全で、そのトレーナーはコンソールのものだった。コンソールはメッセージを表示し、セーブデータを保持した。

ESP32 ボード経由で市販の英語版 FireRed (BPRE) に対して、`--console-build auto` がコンソールのゲームコードから BPRE を選んだ状態で、
バックアップは両方のスロットを完全な状態で読んだ (カウンター 2 と 1)。そのファイルの復元は
19 個のセクタすべてを書き込み、成功メッセージで終わった。その直後に取ったバックアップは、ファイルのロード済みコピーを
カウンター 3 のスロット 1 にバイト単位で保持し、ロードしたゲームのコンソール自身のセーブを、
カウンター 4 のスロット 0 に保持した。そのコピーは、セクタ 1 から 7 でファイルと異なる。これらは、
`LoadGameSave` が振り直す暗号鍵を運ぶブロックである。

2 回目の復元では、同じ英語版カートリッジにフランス語版 FireRed のセーブを書き込んだ。それは成功メッセージで終わり、
「つづきから」はフランス語版のゲームをフィールドに読み込んだ。

## ギフトの作成

`pokeldn.frlg.gift.gift_composer` は、変更不可の `WonderGift`
定義からカードと配達員のスクリプトを作る:

```python
MEWTWO_GIFT = WonderGift(
    slug="mewtwo-encounter",
    card=WonderCardSpec(icon_species=150, title="MYSTERIOUS ENCOUNTER",
                        body=("Visit the deliveryman.",), default_flag_id=1008),
    intro_message="A powerful presence is waiting!",
    event=GiftSpec(shareable="once"),          # or StampRallySpec(...)
    delivery=DeliveryPlan(delivery=(
        DeliveryStage(Message("Take this."), GiveItem(1)),  # Master Ball
        DeliveryStage(ShowSprite(0, RelativeToPlayer(dx=1)), BattleLegendary(150, level=70)),
    )),
    completed_message="That mysterious encounter is over.",
)
```

`GiftSpec` は repeatable と shareable を、`StampRallySpec` はラリーのスロットと完了フックを保持する。`DeliveryPlan` は
3 つの系列を持つ。`WonderGift.delivery` は `delivery` を使い、`StampSlot.delivery` と
`StampRallySpec.completion` は `pre_stages` と `post_stages` を使う。それ以外は拒否される。

コンパイラは `intro_message` を表示し、`VAR_MYSTERY_GIFT_1` からステージを再開し、成功すると
`FLAG_MYSTERY_GIFT_DONE` とカード受領フラグを立てる。後の訪問では `completed_message` を表示する。
`GiftSpec(repeatable=True)` の場合は、代わりにカーソルをリセットする。

### ステージと条件

各 `DeliveryStage` は 1 つのチェックポイントである。報酬が失敗すると、そのステージを再度提示し、成功した
前のステージは飛ばす。1 つのステージに、失敗しうる報酬 (`GiveItem`、`GivePokemon`、`GiveEgg`) を 2 つ入れてはならない。
`GiveEgg` は `GivePokemon` と同じ `moves=(...)` を取る。技を持つタマゴは手持ちの空きが必要なので、
手持ちが満杯なら、パソコンへ送らずに後で再試行する。最初の後の技の値が 0 だと、その
スロットは空になる。`GiveRandomEgg(eggs)` は `(species, moves)` の組を取り、`random` で選ばれた 1 つを渡す。ジャンプテーブルなので、
技 4 つずつの 15 個のタマゴが、1 つの RAM スクリプト (944 バイト) に収まる。

`condition=` (`VarEquals`、`FlagSet`、`Not`、`AllOf`、`AnyOf`) は、偽のときステージの動作を飛ばすが、
カーソルは進める (排他的な分岐のため)。`RequireSpecialResult(...)` はフィールドの特殊処理を
`VAR_RESULT` に呼び出して比較し、失敗すると進めずにメッセージを表示する。

```python
DeliveryStage(ShowSprite(142, RelativeToPlayer(dx=1)),
              condition=VarEquals(0x4031, 0))  # VAR_STARTER_MON == Bulbasaur
DeliveryStage(BattleLegendary(243, level=65),
              condition=Not(AnyOf((VarEquals(0x4031, 0), VarEquals(0x4031, 1)))))
DeliveryStage(RequireSpecialResult(SPECIAL_HAS_ALL_KANTO_MONS, 1, "Finish the KANTO POKEDEX first."),
              GivePokemon(251, level=50))
```

### プレイヤーのセーブデータの書き込み

`SetVar(variable, value)` は `setvar` (0x16) を、`AddVar(variable, value)` は `addvar` (0x17) を出力する。
どちらも、セーブされる変数 (0x4000..0x40FF) か特殊変数 (0x8000..0x8011) に限られる。

### バトル

`BattleLegendary` は `setwildbattle`、`special StartLegendaryBattle`、そして `waitstate` を付けない `end` を出力する。
そのため、バトルが SaveBlock のメモリを動かした後に RAM スクリプトのポインタを再開するものはない
([RAM スクリプトはバトルから戻らないことがある](frlg_rng.md#a-ram-script-may-not-come-back-from-a-battle))。
`BattlePokemon` は通常の `dowildbattle` を出力する。どちらも、そのステージの最後の動作でなければならない。
スタンプのスロットの経路では、ラリーの共有の中間部分を含め、バトルは禁止される。条件付きのバトル
ステージは、終端の代替として許可される。

### Sharing

`GiftSpec.shareable` は、ふしぎなカードの `sendType` のビットに対応する:

| 値 | 動作 |
|---|---|
| `"never"` | 次へ送ることはできない |
| `"once"` | 1 回だけ送ることができる。受け取ったゲームはカードを送れない状態に切り替える |
| `"always"` | 受け取った後も送り続けられる |

### 運命の出会いの印

`GivePokemon(..., fateful_encounter=True)` (と `GiveEgg`) は、公式の Surf Pichu の組を出力する:
`setmonmodernfatefulencounter` (`0xCD`) と `setmonmetlocation` (`0xD2`、`METLOC_FATEFUL_ENCOUNTER` =
0xFF) [data/mystery_event_msg.s:71]。これは任意指定なので、古いカードはバイト単位で同一のままである。

`ScrCmd_setmonmodernfatefulencounter` は、そのインデックスの範囲を検査しない [scrcmd.c:2239] (`setmonmove` は
[script_pokemon_util.c:144] で丸める)。そのため、コンポーザーの `LAST_PARTY_MON_INDEX` 7 がここに届いてはならない。
インデックスは、渡す前の手持ちの数である (`specialvar ... CalculatePlayerPartyCount`)。手持ちが
満杯だと失敗ラベルへ飛ぶので、パソコンへ送られたポケモンには印が付くことがない。

概要画面の運命の出会いの行は、出会った場所だけから決まる
[pokemon_summary_screen.c:2665、:2799 で OR される]。`modernFatefulEncounter` は、Misc+0x08 のリボンワードのビット 31 である
[include/pokemon.h:40-82]。`mon.decode_mon` は、手持ちのダンプからこれを読む。

### コンポーザーでの `initramscript`

`gift_composer.build_bound_script(actions)` は、コンポーザーのアクションを、フィールドスクリプト
`initramscript` が紐づけるものにコンパイルする。`build_mevent_npc_script(actions=...)` はそれらを直接受け取る。バイトコード、
インタプリタ、`ramScript` のスロットは同じなので、アイテム、ポケモン、スプライト、バトルがすべて動く。ステージのカーソルや
受領フラグはない。スクリプトは `end` で終わり、全体が毎回再実行されるので、一度だけの効果には独自の
`SetVar` か条件が必要である。

### 登録と検証

```python
from pokeldn.frlg.gift.gift_registry import GIFT_REGISTRY
GIFT_REGISTRY.register_definition(MEWTWO_GIFT)
```

登録時に既定のフラグ ID を検証してコンパイルする。実行時の `--flag-id` は、改めてコンパイルする。
検証の対象は、カードの文章とフラグ、計画の構造、アクションの範囲、カーソルの境界、スタンプの一意性、
バトルの配置、仮想ポインタ、995 バイトの RAM スクリプトの上限であり、エラーではセクションを明記する:

```text
example-rally.event.slots[1].delivery.post_stages[0].actions[1]: battles are not allowed in stamp-slot delivery plans
```

## 静的ツール

```bash
./.venv/bin/python -m pokeldn.frlg.gift.gift_to_bin --gift beast-cutscene --flag-id 1005 --out-dir exported-gift
./.venv/bin/python -m pokeldn.frlg.save.save_inject game.sav --gift beast-cutscene --flag-id 1005
```

`gift_to_bin` は、336 バイトのふしぎなカードと 1004 バイトの RAM スクリプトを、
`pokemon-gen3-mysterygift-tool` が期待する形式で書き出す。`save_inject` は両方を有効なセーブのスロットに書き込み、
カードの CRC、RAM スクリプトの CRC、セクタのチェックサムを作り直し、`<save>.gift.sav` を保存する (`--in-place` は
上書きする)。`--make-artifact` は、`artifacts/` の下に決定的な `.ram.lst` を書き出す (バイト列、デコードした
命令、チェックサム、ターゲット、ステージの要約)。

## 閉ざされた経路

### ワイヤレス通信 (JoySpot)

RFU のシリアル番号の関門で止められている。どちらの経路も同じギフトの会話に至る:

| | ともだち | ワイヤレス通信 |
|---|---|---|
| 受信側 | `Task_ListenForCompatiblePartners` [union_room.c:3757] | `Task_ListenForWonderDistributor` [union_room.c:3799] |
| 受け入れる RFU のシリアル | `IsRfuSerialNumberValid` → `{0x0002, 0x7F7D}` | `== 0x7F7D` のみ [link_rfu_3.c:920] |
| 選択 | プレイヤーが一覧から選ぶ | 自動で接続し、ボタンを押す必要はない |
| Switch から到達可能か | はい | いいえ |

`Task_CardOrNewsOverWireless` [union_room.c:2415] は探索し、120 フレーム待ってから候補 0 を関門にかける:

1. `Rfu_GetWonderDistributorPlayerData` [link_rfu_3.c:917] は、
   `partner[idx].serialNo == RFU_SERIAL_WONDER_DISTRIBUTOR (0x7F7D)` の場合に限り候補を残し、それ以外は 0 にする。
2. `groupScheduledAnim == UNION_ROOM_SPAWN_IN && !startedActivity`。
3. `HasWonderCardOrNewsByLinkGroup`: 広告された `hasCard` ビット。失敗すると SE_BOO が鳴る。
4. `CreateTask_RfuReconnectWithParent(...)`。

誤ったシリアルは関門 1 で黙って失敗する (SE_BOO は鳴らない)。Switch のブリッジは `0x0002`
(`RFU_SERIAL_GAME`) を報告する。ともだち (`sAcceptedSerialNos` [link_rfu_2.c:240]) は候補をすべて一覧に出し、
ワイヤレス通信はそのすべてを無視する。広告にはシリアルのフィールドがなく、ネイティブの広告は
次の 4 つのフィールド以外がゼロである:

```
50 10 | c1 cc bf bf c8 ff 00 00 | 65 ac | 00 00 00 00 | 84 15 | 00 00 00 00 00 00
TID   | uname                   | parent| unexplained | search| unexplained
```

`svc_47` [sloopsvc.c:34] は `{u8 HostRfuGameData[0x10]; u8 HostRfuUsername[8]}` を取る。これは 24 バイトでシリアルを含まず、一方ブリッジは
候補の一覧を `svc_45_rfu_link_status()` 経由で書き込む。

ワイヤレス経路から 802.11 の認証を引き出せなかった広告 (21 通り試行): シーン ID
(0、21、0x7F7D)、LDN と Pia のアプリのバージョン、オフセット 12、13、14、18、
19、20、22 の `0x7F7D` (両方のバイト順)、アクティビティ (0、4、21)、`hasCard`、検索ワードのビット 7。固定した値: `local_communication_id =
0x01006fa0233f8000`、LDN バージョン 4、チャンネル 1、`max_participants = 2`、Pia `sysCommVer = 22`、シーン
22287。

ネイティブの取り込みでは `0x1584 & 0x7F = 4 = ACTIVITY_TRADE` で、そのオフセットのアクティビティ 21 ではともだちの一覧と参加が得られた。そのため、
検索ワード `record[16:18]` は
`activity:7 | bit7 | version:3 | language:3 | hasCard?:1 | startedActivity:1` である (version 5 = LeafGreen、
language 2 = English、その取り込みでの値)。

未検証: `local_communication_id` (変えるとホストが隠れ、関門と見分けがつかない)、
シーンの総当たり、複数の変数の組み合わせ。ブリッジが、広告できる何かから `partner[].serialNo` を割り当てる証拠か、
Switch がふしぎなカードの配布元として扱う広告の取り込みが出れば、再び開く。この遮断により、
ボタンを押さない経路と、木の実 4 つのふしぎなニュースの報酬が失われる。

### e-Reader 自体

トレーナータワーの設定と `CEReaderTool_SaveTrainerTower` では、`ereader_screen.c` が
無線アダプターではなく、GBA のシリアル通信経由で `gLinkType = LINKTYPE_EREADER_FRLG` を開く。

### オーロラチケットとミスティックチケット

チケットのカードは Switch 版では何もしない。配布のスクリプトは `data/mystery_event_msg.s:200` にあるが、Switch 版は両方のチケットを与える
と両方の `FLAG_RECEIVED_*` フラグを、最初の殿堂入りの時点で立てる
[post_battle_event_funcs.c:52, `#if REVISION >= 0xA`] ので、完了したセーブではスクリプトは何もしない。
ギャラリーの `FL - Item AuroraTicket` スクリプトは、まず `FLAG_RECEIVED_AURORA_TICKET` を検査する。殿堂入りの後では配達員は
"Merci d'utiliser le système CADEAU MYST." とだけ言い、何も渡さない。このカードの `iconSpecies` は `0xFFFF` で、
`SPECIES_NONE` 以外のどの値でもアイコンが描かれ、
`SPECIES_UNOWN_B - 1` を超える種族は `SPECIES_NONE` の「?」を描画する
[mystery_gift_show_card.c:466, pokemon_icon.c:1102]。
古い海図 (Old Sea Map) は Emerald 専用である [mystery_gift.c:30]。

## 落とし穴

- `charmap.encode` は、改行を含む未知の文字を落とす。改行は 0xFE で、`mg_server` の
  エンコーダーは `\n` で分割し、0xFE で連結する。3 行目や、そのウィンドウで ROM の最長の文字列より広い行は拒否する
  ("A WONDER CARD has been received"、31 文字 [strings.c:1291])。
  ウィンドウ 1 は 28 タイル × 4 タイルである [mystery_gift_menu.c:97,524]。
- 新しいペイロードは、`buffer_script.py` の `DUMP_SCRIPTS` と `DECODED_SCRIPTS`、およびランチャーの
  `--dump-file` の行にも追加する。追加しないと、ハードウェアの経路が未検証のまま、オフラインのハーネスだけが通る。
