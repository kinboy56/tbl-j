---
title: ROM マップ
parent: FireRed and LeafGreen
nav_order: 4
---

# ROM マップ

ここにあるアドレスはすべて、ふしぎなおくりもののリンク越しに、コンソール自身のカートリッジから読み取ったものです
（[コンソールでのコード実行](frlg_rom.md) の `memory-dump`、`memory-scan`、`table-scan`、`call-chain`）。
`pokeldn/frlg/rom/rom_map.py` は各アドレスの取得方法を記録しています。`tests/test_rom_map.py` はそれをダンプと照合します。
カートリッジイメージは、そのすべてと一致します。

アドレスは、フランス語版ファイアレッド（カートリッジ BPRF、ソフトウェアバージョン 0x0A）のものです。リーフグリーンのアドレスは
[リーフグリーン](frlg_leafgreen.md) にあります。

## カートリッジイメージ

Switch 版では、GBA の ROM が RomFS の唯一のファイルとして入っています。

| タイトル ID | RomFS ファイル | サイズ | sha1 |
|---|---|---|---|
| 01004B3023412000 | `/FireRed_f.gba` | 16777216 | `07566b82dbd2a91321698f730f6400ae4c56ddf1` |
| 010087C02342E000 | `/LeafGreen_f.gba` | 16777216 | `9f774956dfbad7f69ddb91fb91d2c26e54408f75` |
| 0100554023408000 | `/FireRed_e.gba` | 16777216 | `baa452d0b24629dd7782cfc07a8984085dde1311` |
| 010034D02340E000 | `/LeafGreen_e.gba` | 16777216 | `62b9fc77549dbc67032eb6cbd0ea6ad3b825690f` |

0xA0 のヘッダーには `POKEMON FIRE` `BPRF` と `POKEMON LEAF` `BPGF` があり、バージョンは 0x0A です。

フランス語版ファイアレッド 1.0.1 のアップデートは、カートリッジを変えません。市販の本体で `rom-checksum` と、差分のある 1 区間のバイト読み取りを通して読んだゲスト側の ROM は、
`FireRed_f.gba` と一致します。範囲は `0x08000000..0x09000000` で、
ラッパーのロード時パッチ 3 つ（[frlg_rom.md](frlg_rom.md)、ブレークポイントフック）は除きます。
`0x09000000..0x0A000000` は、両方ともオープンバスを読みます（各ハーフワードが、自身のアドレスを
半分にした値を返す）。

英語版の組（`BPRE`、`BPGE`、バージョン 0x0A、ベースの v0 パッケージ）は、
`pret/pokefirered` が固定している `pokefirered_switch.gba` と `pokeleafgreen_switch.gba` とバイト単位で同一です。デコンプは
英語版リリースの正確な地図です。

Program NCA には CTR RomFS のセクションが 1 つあり、アップデートはありません。そのため `bktr_read.py` はこれを扱えません。
`scratchpad/base_romfs.py` で読みます。

    ./.venv/bin/python scratchpad/base_romfs.py "$NSP" --list
    ./.venv/bin/python scratchpad/base_romfs.py "$NSP" --extract /FireRed_f.gba --out scratchpad/FireRed_f.gba

`rom_map.CREATE_MON` は 0x08041150 で、イメージはそこで `f0b5 4746 80b4 87b0` を読みます。これは、
`asm/create-mon.s` が前提とするプロローグです。どの関数の本体も、イメージからオフラインで読めます。

## EWRAM は両方のビルドで同じアドレスにある

英語版ビルドの EWRAM シンボルは、フランス語版カートリッジの EWRAM アドレスと同じです。独立に測った 3 つの値は
`pokefirered_switch.elf` と一致します。

    gDecompressionBuffer  0x0201C000
    gPlayerParty          0x02024280
    gPlayerPartyCount     0x02024025

IWRAM は移りません。`gSaveBlock1Ptr` は英語版ビルドでは 0x030042D8、
カートリッジでは 0x03004228 です。

ELF 内のサイズ付き EWRAM シンボルをすべて領域から引くと、どのシンボルも使っていない区間が 1 つ残ります
（`nm -S pokefirered_switch.elf`、`scratchpad/ram_survey.py`）。

    highest symbol end   0x0203FBAC
    EWRAM end            0x02040000

## 英語版カートリッジ

ふしぎなおくりもののホストは、それぞれのアドレスでビルドされた英語版ファイアレッド（`BPRE`）と英語版リーフグリーン（`BPGE`）のコードを送ります。
`pokeldn/frlg/rom/builds.py` には、カートリッジごとに 1 つのテーブルがあります（`BPRF`、`BPGF`、`BPRE`、
`BPGE`、`BPRS`、`BPGS`、`BPRD`、`BPGD`、`BPRI`、`BPGI`、`BPRJ`、`BPGJ`）。内容は、IWRAM のグローバル変数、ペイロード・フック・フィールドスタブが呼ぶ ROM 関数、
`call-chain` が挙げる関数、そしてギフトが運ぶ ROM のデータポインタです。

ホストは、コンソールの `MysteryGiftLinkGameData` にあるゲームコードからテーブルを選びます
[mystery_gift.c:369]。これは、アドレスに依存するものが送られる前に届きます。バイト列がビルドごとに異なるペイロードは、
ビルドごとに作られます。テーブルがないコンソール、または実行時に除外されたコンソールは、
（`--console-build CODE`、または別バージョンに対する `--version`）、何も送らずに拒否されます。
ビルドに依存しないペイロードは、どのコンソールにも送られます。

英語版のアドレスは `pokefirered_switch.elf` と `pokeleafgreen_switch.elf` から得ており、それぞれ市販のイメージと照合しています。
関数はバイト列で、IWRAM のグローバル変数はそれを使うリテラルプールで確かめ、
フランス語版カートリッジの同じプールと対応づけています。

| 領域 | フランス語版から英語版へ |
|---|---|
| EWRAM | 変化なし |
| IWRAM 0x03000000..0x03001B6F | 変化なし |
| IWRAM 0x03002370..0x0300602F 英語版、`gMain` から（フランス語版 0x030022D0、英語版 0x03002380） | +0xB0 |
| IWRAM 0x03006090..0x03007583 英語版、`gSoundInfo`（フランス語版 0x03005F80）から `gFlash` まで | +0x110 |
| IWRAM 0x03007590 以降: スタック、割り込みベクター | 変化なし |
| ROM | 単一のオフセットはなく、関数ごとに探す |

英語版の値は、`gRngValue` 0x030042D0、`gSaveBlock1Ptr` 0x030042D8、`gSaveBlock2Ptr` 0x030042DC、
`gIntrTable[4]` 0x030027E0、`gLastWrittenSector` 0x03004650、`gSaveCounter` 0x03004660 です。英語版リーフグリーンの RAM は
英語版ファイアレッドのものと同じです。常駐フックのアドレスのうち動くのは `m4aSoundMain` だけで、
（0x081E07C4 対 0x081E07E8）。

`tests/test_frlg_english_cartridges.py` は、両方の市販イメージに対して英語版ペイロードを、各カートリッジ自身の
`Client_RunBufferScript` を通して実行します。`scratchpad/frlg_en/` にイメージがないときは、スキップします。

英語版ファイアレッド（USA v0、`0100554023408000`）は、ゲームデータで `BPRE` を報告し、ふしぎなカードを受け取って表示し、
英語版の `call-chain` を実行します。`0x08046A41` の `SpeciesToNationalPokedexNum` は種族 277 に対して 252 を返し、
`0x08071CD5` の `VarGet` は戻ります。エミュレートしたコンソールで確認しました。どちらのセッションも、
成功メッセージで終わり、これによりセーブが行われます [mystery_gift_menu.c:1379]。

## BIOS ラッパー

`libagbsyscall.s` は、デコンプの順序で並ぶ THUMB の `svc N ; bx lr` の組を 1 つのブロックとしてリンクします
[src/libagbsyscall.s]。両方のカートリッジで同一で、リーフグリーンでは 0x24 低い位置にあります。

| ラッパー | svc | ファイアレッド | リーフグリーン |
|---|---|---|---|
| `ArcTan2` | 0x0A | 0x081E21D4 | 0x081E21B0 |
| `BgAffineSet` | 0x0E | 0x081E21D8 | 0x081E21B4 |
| `CpuFastSet` | 0x0C | 0x081E21DC | 0x081E21B8 |
| `CpuSet` | 0x0B | 0x081E21E0 | 0x081E21BC |
| `Div` | 0x06 | 0x081E21E4 | 0x081E21C0 |
| `LZ77UnCompVram` | 0x12 | 0x081E21E8 | 0x081E21C4 |
| `LZ77UnCompWram` | 0x11 | 0x081E21EC | 0x081E21C8 |

ゲームの LZ77 ストリームは VRAM に安全です（コピー距離 1 は使われていません）。
ファイアレッドのフシギダネの正面スプライト（676 バイト、0x08D2FBD4 にあります）と、パレット（40 バイト、0x08D2FE78 にあります）は、
`gbagfx` がデコンプのソースから書き出すものとバイト単位で一致します。LZ77 はペイロードのコードを縮めません（常駐フックは
同じか、むしろ大きくなります）。1024 バイトを超えるデータに対してだけ効果があります。

## 最初のアンカー

`anchors` は、`Client_RunBufferScript` の呼び出しの後のアドレス `0x08148C75` を返します。ほかのすべてのアドレスは、そこから到達します。
呼び出し元のリテラルプールと `bl` の飛び先は次の関数を示し、それが示すポインタテーブルはさらに多くを与えます。
すべてのエントリは、既知のプロローグに着かなければなりません（`scratchpad/rom_read.py`）。

0x08148A00 のダンプを逆アセンブルすると `Client_RunBufferScript` [mystery_gift_client.c:274] になり、`cmp r0,#1` は
0x08148C74 にあります。そのリテラルプールは次のとおりです。

    0x08148C88 -> 0x0201C000   gDecompressionBuffer
    0x08148C8C -> 0x0300422C   &gSaveBlock2Ptr
    0x08148C90 -> 0x03004228   &gSaveBlock1Ptr

`MysteryGiftClient_CallFunc` は 0x08148C94 から続き、0x0845DBD0（sClientFuncs）から 8 ワードをコピーします。
その添字は `[r0,#8]` の `client->funcId` で、8 つすべてが `push {r4, lr}` に着きます。エントリ 7 は 0x08148C61 で、
これは `anchors` が測定した関数です。

0x08000000 のダンプにあるカートリッジヘッダーにより、`REVISION >= 0xA` の分岐が実行されることを確認できます。

    entry      b 0x08000204
    title      POKEMON FIRE          [0xA0]
    game code  BPRF                  [0xAC]  BPR = FireRed, F = French
    version    0x0a                  [0xBC]
    header checksum 0x5d, recomputed 0x5d -> VALID

## 4 つの関数テーブル

    gScriptCmdTable              0x08163650   214 entries   the field script commands
    gSpecialVars                 0x081639A8    21 entries
    gSpecials                    0x081639FC   444 entries   gSpecialsEnd 0x081640EC
    gStdScripts                  0x081640EC    10 entries
    gMysteryEventScriptCmdTable  0x081DE144    17 entries

`script_data` は 0x08163650 から 0x081DE188 まで続き、そこで `lib_text` が始まります。

### 形で見つけた `gSpecialVars`

最初の 12 ワードはそれぞれ、直前のものより正確に 2 上の位置にあります。`gSpecialVar_0x8000` から `0x800B` までは、
連続する 12 個の `u16` です [event_data.c:16]。`table-scan` は、その連なりの開始位置と最初の値を、1 回の実行で返します。
`gSpecialVars` = 0x081639A8、`gSpecialVar_0x8000` = 0x020370B4 で、2.75 MB の中でこのような連なりは
これだけです。この範囲はリンク順から得ました。`script_data` はすべての `.text` オブジェクトの後に続き
[ld_script_rev10.ld:318]、`.rodata` は `gSpeciesInfo` より下から始まります。`gSpecialVar_0x8000` は `EWRAM_DATA` にある、
リンク時のグローバル変数です。定数として名前を付けても差し支えありません。

### `gScriptCmdTable`

`script_data` は `gScriptCmdTable`（4 バイトのエントリ 214 個）で始まり、その直後に `gSpecialVars` を置きます
[ld_script_rev10.ld:318]。そのため 0x08163650 から始まり、856 バイトのダンプ 1 回で読めます。214 ワードすべてが
10488 バイトの範囲を指す THUMB ポインタです。アドレスを共有するのは 0 と 213 の 2 つだけで、どちらも
`ScrCmd_nop` です。`ScrCmd_nop1` は添字 1 にあり、別のアドレスを持ちます。エントリを 1 つずらして読むと、この結果にはなりません。

    0x23 callnative   0x0806D854      0x44 additem      0x0806DED0
    0x25 special      0x0806D7EC      0x79 givemon      0x0806F834
    0x29 setflag      0x0806E0EC      0x90 addmoney     0x0806F998

`pokeldn/frlg/rom/scrcmd_names.py` は、すべてのエントリにオペコードで名前を付けます。
`scrcmd_names.handler("additem")` はオフラインで答えを返します。

### `gSpecials` と `gStdScripts`

`ScrCmd_special` は u16 で `gSpecials` を添字引きし、`gSpecialsEnd` に対して範囲を検査してから、
ベニア（veneer）経由で呼び出します [scrcmd.c:101]。そのリテラルプールは、`gSpecials` = 0x081639FC と `gSpecialsEnd` =
0x081640EC を与えます。0x6F0 = 444 × 4 で、`data/specials.inc` の 444 個のエントリであり、`gSpecialVars + 21 * 4` から始まります。
呼び出しは 0x081E2224 を通り、これは `_call_via_r1` の 4 バイト手前です。ダンプは 444 個の THUMB ポインタを与え、
171 個の `NullFieldSpecial` の添字は、2 つのダンプをまたいで 1 つのアドレスを指します。`rom_map.SPECIAL_ADDRESSES` がそれらを保持し、
`rom_map.special_function("HealPlayerParty")` は名前で 1 つを解決します。

`gStdScripts` は `data/event_scripts.s` で `gSpecials` の後に続きます。`.align 2` は既に満たされており、
0x081640EC にあります。10 ワードはいずれも 0x081A76xx..0x081AB5xx の範囲を指し、そのうち 5 つのメッセージボックス用スクリプトは
互いに 40 バイト以内にあります。

### 実行中の構造体から見た `gMysteryEventScriptCmdTable`

テーブルの 17 個のエントリは互いに無関係なアドレスですが、そのアドレスは構造体に保持されています。

```c
static void InitMysteryEventScript(struct ScriptContext *ctx, u8 *script)
{
    InitScriptContext(ctx, gMysteryEventScriptCmdTable, gMysteryEventScriptCmdTableEnd);
```

[mystery_event_script.c:52]。`struct ScriptContext` は、その組を +0x5C と +0x60 に保持しており
[include/script.h]、2 つは 68 バイト離れています。コンテキストは `EWRAM_DATA static struct ScriptContext
sMysteryEventScriptContext` です [mystery_event_script.c:27]。EWRAM を対象に `table-scan --table-delta 0x44 --table-runlen 2` を実行すると、
これが見つかります。コンテキストはスクリプトが走るまでゼロのため、スキャンは同じ起動の中で
ふしぎなおくりもののイベントを流した後に行う必要があります。

86 回の呼び出しで EWRAM 256 KB 全体を走査したところ、0x081DE144 を保持する 0x0203AA94 で 1 件だけ一致しました。
`sMysteryEventScriptContext` = 0x0203AA38、テーブル = 0x081DE144 です。17 個のエントリは奇数（THUMB）で互いに異なり、
`.text` の内側にあって、1084 バイトの範囲に収まり、`mystery_event.OPCODE_NAMES` の順に並んでいます。

| # | コマンド | ハンドラー | | # | コマンド | ハンドラー |
|---|---|---|---|---|---|---|
| 0 | `nop` | 0x080DE451 | | 9 | `givenationaldex` | 0x080DE61D |
| 1 | `checkcompat` | 0x080DE401 | | 10 | `addrareword` | 0x080DE641 |
| 2 | `end` | 0x080DE3F5 | | 11 | `setrecordmixinggift` | 0x080DE66D |
| 3 | `setmsg` | 0x080DE465 | | 12 | `givepokemon` | 0x080DE681 |
| 4 | `setstatus` | 0x080DE455 | | 13 | `addtrainer` | 0x080DE78D |
| 5 | `runscript` | 0x080DE49D | | 14 | `enableresetrtc` | 0x080DE7D5 |
| 6 | `initramscript` | 0x080DE5B5 | | 15 | `checksum` | 0x080DE7E9 |
| 7 | `setenigmaberry` | 0x080DE4B9 | | 16 | `crc` | 0x080DE831 |
| 8 | `giveribbon` | 0x080DE581 | | | | |

`data/mystery_event_script_cmd_table.o(script_data)` は `script_data` の最後のメンバーです
[ld_script_rev10.ld:318-330]。0x081DE188 は `0x4C41B510`（`push {r4, lr}`）を読み、これは
`lib_text` の先頭にある `libgcnmultiboot` のプロローグです。

## テーブルのエントリから、その背後の関数へ

ハンドラーは `struct ScriptContext *` を受け取り、引数をスクリプトから読みます。アドレス順に並ぶその `bl` の飛び先は、
デコンプのソース順の呼び出しに対応します（引数ごとに `VarGet(ScriptReadHalfword(ctx))`、
その後に 1 回の呼び出し [scrcmd.c:463-590]）。そのため、1 つのダンプから位置によってワーカーに名前を付けられます。

    ScriptReadHalfword   0x0806D1E8      AddBagItem           0x0809DA70
    VarGet               0x08071DDC      RemoveBagItem        0x0809DBC4
    GetVarPointer        0x08071CC8      CheckBagHasSpace     0x0809D9EC
    FlagSet              0x08071EF4      CheckBagHasItem      0x0809D92C
    FlagClear            0x08071F1C      AddPCItem            0x0809DDB4
    FlagGet              0x08071F44      IncrementGameStat    0x080587A4

`ScrCmd_additem` は、デコンプの命令と一対一で一致します（`(u8)quantity` のキャストは
`lsls r1, #24; lsrs r1, #24` です）。そして 0x020370CC の `gSpecialVar_Result` に格納します。`ScrCmd_random` の
3 回目の呼び出しは 0x080486B0 の `Random` で、これは独自のリテラルプールから独立に見つかりました。

`scratchpad/handler_workers.py` は、各ハンドラーの `bl` の飛び先を順に出力し、既知のものには名前を付けます。
`tools/frlg/rom_functions.py --table specials|field|mystery-event|callable` は、テーブル全体に対してこれを行い、
各本体を、次のエントリと自身のエピローグで区切り、次の実行の計画を出力します。まだ保持していない
エントリを、順位付けされた `--dump-address` の窓にまとめます。

この ROM は agbcc でビルドされており、THUMB の関数は `pop {r4,r5,r6}; pop {r1}; bx r1`（BC70 BC02 4708）で終わります。
`pop {..., pc}` で終わることはありません。0xBDxx だけを探す読み取りは、次の関数に踏み込んでしまいます。
`pokeldn/frlg/rom/thumb.py` も `bx Rn` に一致させ、
戻りは、プロローグが後に続く場合にだけ境界として扱います。

### 300 個のワーカーの名前をオフラインで付ける

`scripts/gen_worker_names.py` は、ダンプされたすべての本体をデコンプと突き合わせ、4 つのテーブルすべてを一度に処理して、
`pokeldn/frlg/rom/worker_names.py` を書き出します。確認は次の 4 つです。

| 確認 | 除外するもの |
|---|---|
| 長さ | インライン化、`__umodsi3`、呼び出しとして読まれたマクロ |
| アンカー | 位置がずれた本体。測定したすべてのアドレスは、自身の名前に戻らなければならない |
| 一致 | 2 つの呼び出し元から別々の名前を付けられた対象 |
| リンク順 | ROM 内の誤った位置にある名前 |

揃った 164 個の本体にわたって、アンカーの確認は、測定した 68 個の異なる名前に 557 回当たります。それぞれが
自身のアドレスに当たります。agbcc は翻訳単位を定義の順に出力し、`ld_script_rev10.ld:53` はオブジェクトをレイアウトの順に並べるため、
名前とアンカーは 1 本の昇順の並びを作ります。
確認は、最初の途切れではなく、最も長い昇順の連なりを残します。

2 つのアンカーの間の隙間に、名前のない対象 1 つと呼び出し 1 つだけがある場合、その対応は強制され、
長さの規則を緩めます。最初のアンカーより前と最後のアンカーより後の開いた区間には、何も名前を付けません。

ソースを読むと、次のことが分かります。

- `firered_switch` は `GAME_VERSION=FIRERED GAME_REVISION=10 MODERN=0` です [Makefile:227]。203 個の
  `#if REVISION >= 0xA` ブロックが有効で、その `#else` は有効ではありません。
- 評価は後順です。`VarGet(ScriptReadHalfword(ctx))` は、`bl ScriptReadHalfword` の後に `bl VarGet` になります。
- マクロは呼び出しではありません。`#define ScriptReadByte(ctx) (*(ctx->scriptPtr++))` [include/script.h:24]
  は `bl` を出力せず、151 個の本体に現れます。
- `NDEBUG` が有効です。`ScrCmd_special` はカートリッジ上で 2 回の呼び出しを行い、assert があれば 3 回目が加わることになります。

デコンプにも存在する `rom_map` の名前は、どの本体もそのアドレスに到達することで確かめられます
（結合は `rom_map.DECOMP_NAMES` が行います）。

| `rom_map` の名前 | デコンプ | 一致した本体 |
|---|---|---|
| `GET_MON_DATA` | `GetMonData3` | 15 |
| `SCRIPT_CONTEXT_SET_NATIVE` | `SetupNativeScript` | 11 |
| `SET_RESPAWN` | `SetLastHealLocationWarp` | 1 |
| `SCRIPT_MOVEMENT_START` | `ScriptMovement_StartObjectMovementScript` | 2 |
| `CHANGE_AMOUNT_MONEY_BOX` | `ChangeAmountInMoneyBox` | 2 |
| `ME_CHECK_COMPATIBILITY` / `ME_SET_INCOMPATIBLE` | `CheckCompatibility` / `SetIncompatible` | 1 / 3 |

`GetMonData` は、引数の数で振り分けるマクロです [include/pokemon.h:343]。`GetMonData2` は
`__attribute__((alias("GetMonData3")))` です [pokemon.c:2970]。1 つのアドレスに 3 つの名前があります。

デコンプが呼ぶとしている回数だけコマンドから到達するワーカーは、確かめられています。`Compare` は、8 個の `compare_*` コマンドから
到達します。`StringCopy` は、7 個の `buffer*` と、`gText_BigGuy` から名前を組み立てる 2 つの特殊処理から到達します。
`0x0806D0EC` は `StopScript(ctx)` [script.c:76] で、これは `ScrCmd_end` の呼び出しです。
`ScriptContext_Stop(void)` [:360] は 0x0806D418 で、12 個のハンドラーと、
`ScrCmd_waitstate` 全体（ただし `return TRUE` は除く）から呼ばれます。

### 2 つの混在イメージの危険

- 2 枚のカートリッジのダンプを 1 つのイメージとして読んではいけません。リーフグリーンは、このコードを −0x2C 離れた位置に置くため、
  リーフグリーンのダンプをファイアレッドの `--dump-address` に置くと、誤った答えを返します。混在したイメージでは、`gSpecials[54]` の呼び出し
  （`Script_HasTrainerBeenFought`、`FlagGet(GetTrainerAFlag())`）が `FlagSet` として読まれます。これは
  +0x2C の `SetBattledTrainerFlag2` です。`script_read.every_dump` は 1 枚のカートリッジを使います（既定はファイアレッド）。
  実行の `--expect-console` から決め、ない場合はタグにフォールバックします。
- すべてのダンプを一度に読みます。`scrcmd.Memory` は重なったダンプと隣接したダンプを結合するので、2 つの実行をまたぐブロックも
  走査できます（`tools/frlg/script_read.py --with-every-dump`）。1 つずつ読むと、すでに保持しているバイト列に対して
  実行を提案してしまいます。

### 測定したもの

すべてのテーブルの背後にある本体は、カートリッジから得ています。フィールドコマンド 213 個、ふしぎなおくりもののイベントのオペコード 17 個、呼び出し可能な関数 27 個、
特殊処理 272 個です。

- `NullFieldSpecial` は `bx lr`（2 バイト）で、0x080CE8DC にあります。
- 特殊変数は ID の順に、それぞれ 2 バイトずつ並んでいます。`ShakeScreen` [310] は 0x020370BC、BE、C0、C2 を読み込み、
  デコンプは `gSpecialVar_0x8004..0x8007` を読みます。`GetPlayerXY` [143] は最初の 2 つを書き込み、
  `GetPartyMonSpecies` [327] は最初のものを読みます。
- 0x080CE818 の `GetLeadMonIndex` はテーブルにありません。先頭のポケモンを扱う 4 つの特殊処理がこれを呼び出します。
- 1 回の読み込みだけの特殊処理は、グローバル変数に名前を付けます。`GetBattleOutcome` [180] は `ldr; ldrb; bx lr` で、そのプールの語は
  `gBattleOutcome` です。同じ方法で `gStringVar1` と `gStringVar4`（`ShowFieldMessageStringVar4` [141]）が得られました。
- テーブル同士の照合も一致します。特殊処理は、ワープのワーカーと同じく 0x08081CC8 を `DoDiveWarp`、0x08081DA0 を `DoFallWarp` と名付けます。
  `CalculatePlayerPartyCount` は特殊処理 131 で、`ScrCmd_getpartysize` の唯一の呼び出しです。
  `GetPlayerFacingDirection` は特殊処理 287 で、`ScrCmd_faceplayer` の最初の呼び出しです。

ふしぎなおくりもののイベント VM のワーカーは次のとおりです。

| ワーカー | アドレス | 呼び出し元 |
|---|---|---|
| `StringExpandPlaceholders` | 0x0800CADC | メッセージを出すすべてのハンドラーの最後の呼び出し |
| `RunScriptImmediately` | 0x0806D438 | `runscript`。`ScriptReadWord` の後で、ほかには呼ばれない |
| `InitRamScript` | 0x0806D5F0 | `initramscript` |
| `GiveGiftRibbonToParty` | 0x080A43B0 | `giveribbon`。2 回のうち最初 |
| `EnableRareWord` | 0x080C1658 | `addrareword`。2 回のうち最初 |
| `CheckCompatibility` | 0x080DE300 | `checkcompat`。分岐の前 |
| `SetIncompatible` | 0x080DE330 | `checkcompat` の else。どちらの未使用オペコードもこれだけを呼ぶ |
| `memcpy` | 0x081E44F4 | `addtrainer`。2 回目の呼び出し |
| `CalcCRC16` | 0x080489A0 | `crc`。唯一のワーカー |
| `StringCopyN` | 0x0800C8CC | `setenigmaberry` と `givepokemon`。それぞれ 2 回ずつ |
| `StringCompare`, `SetEnigmaBerry` | 0x0800C938, 0x080A01B0 | `setenigmaberry` |
| `VarSet` | 0x08071DF8 | `setenigmaberry`。最後の呼び出し |
| `SpeciesToNationalPokedexNum` | 0x08046994 | `givepokemon` |
| `GetSetPokedexFlag` | 0x0808C860 | `givepokemon`。2 回: まず SEEN、次に CAUGHT |
| `ItemIsMail`, `GiveMailToMon2`, `CompactPartySlots` | 0x0809BB18, 0x0809B964, 0x080971FC | `givepokemon` |

`memcpy`（libgcc、`lib_text` 内）は 0x081E44F4 にあり、0x081DE188 より上に位置して、境界と一致します。

`VarSet` にはここからしか到達できません。どの ScrCmd の本体も呼び出しません（`setvar` は
`GetVarPointer` の結果を通して格納するため、`call-chain` の `prev` にあたります）。`event_data.c` は `GetVarPointer`、
`VarGet`、`VarSet` をこの順で宣言し、0x08071CC8 < 0x08071DDC < 0x08071DF8 に配置されます。後ろの 2 つの間にある 0x1C は、
`VarGet` の本体です。

呼び出しベニアは `bx rN` にアラインメントを加えたもので、それぞれ 4 バイトです。測定したのは r0 が 0x081E2224、r1 が 0x081E2228、r3 が
0x081E2230 なので、0x081E2234 は `_call_via_r4`、0x081E223C は `_call_via_r6` です。

## セーブのバックアップと復元

すべてのカートリッジで、`LoadGameSave` [decomp:src/save.c:803] と `gRfu.sendQueue.count` [link_rfu_2.c:3131] を、
英語版リビジョン 0x0A の ELF と照合しました。`bl` の飛び先とリテラルプールの語はマスクしています。
`LoadGameSave` は `push {r4-r6, lr}; lsls r0, r0, #24`（`70 b5 00 06`）で始まり、12 個すべてで
`gDecompressionBuffer`（`0x0201C000`）をプールに持ちます。キューの数は 12 バイトの葉関数で読みます。
`ldr r0, =gRfu; ldr r1, =0x8D2; adds r0, r0, r1; ldrb r0, [r0]; bx lr`。カートリッジごとに 1 つのコピーがあります。

| カートリッジ | `LoadGameSave` | `gRfu` | `gRfu.sendQueue.count` |
|---|---|---|---|
| `BPRE` | `0x080DDBF4` | `0x03005590` | `0x03005E62` |
| `BPGE` | `0x080DDBC8` | `0x03005590` | `0x03005E62` |
| `BPRF` | `0x080DDFD4` | `0x030054E0` | `0x03005DB2` |
| `BPGF` | `0x080DDFA8` | `0x030054E0` | `0x03005DB2` |
| `BPRD`, `BPRI` | `0x080DDF14` | `0x030054E0` | `0x03005DB2` |
| `BPGD`, `BPGI` | `0x080DDEE8` | `0x030054E0` | `0x03005DB2` |
| `BPRS` | `0x080DDFFC` | `0x030054E0` | `0x03005DB2` |
| `BPGS` | `0x080DDFD0` | `0x030054E0` | `0x03005DB2` |
| `BPRJ` | `0x080DED68` | `0x03005520` | `0x03005DF2` |
| `BPGJ` | `0x080DED3C` | `0x03005520` | `0x03005DF2` |

`sSaveSlotLayout` は、セクター ID 4 を除く 12 個すべてで一致します。ラテン文字版のカートリッジでは 3816 バイト、
日本語版では 3776 バイトです。

# コンソールのスクリプトを読む

`gScriptCmdTable` を測定し、オペランドの幅をデコンプのマクロから生成した後
（`scripts/gen_scrcmd_args.py` → `pokeldn/frlg/rom/scrcmd_args.py`。各 `.macro` はオペコードを `.byte` として出力し、
続けて引数ごとに `.byte`、`.2byte`、`.4byte` のいずれかを出力します）、`memory-dump` と `scrcmd.disassemble` を組み合わせて、
カートリッジ内の任意のスクリプトを読めます。

    0x081A7624  6A  lock
    0x081A7625  5A  faceplayer
    0x081A7626  67  message 0x00000000
    0x081A762B  66  waitmessage
    0x081A762C  6D  waitbuttonpress
    0x081A762D  6C  release
    0x081A762E  03  return

これは `Std_MsgboxNPC` [data/scripts/std_msgbox.inc] です。正しさの証拠は、各スクリプトがどこで終わるかにあります。
各 `gStdScripts` のエントリから辿ると、次のエントリの 1 バイト前の終端で終わらなければなりません。どこか 1 箇所でも幅が 1 バイト違えば、
対応がずれてしまいます。

## 条件分岐を含むマクロ

11 個のマクロは、条件分岐を持つ本体を持ちます。生成器は各分岐を辿り、
未知のサブマクロに出会うと例外を送出します。

| コマンド | 幅 | 構造 |
|---|---|---|
| `applymovement` | 7 バイト | `.ifb \map` の分岐 |
| `waitmovement`, `removeobject`, `addobject` | 3 バイト | `.ifb \map` の分岐 |
| `applymovementat`, `waitmovementat`, `removeobjectat`, `addobjectat` | 9, 5, 5, 5 バイト | 別の分岐 |
| `warp` と、ほかの 8 個のワープ | 8 バイト | `formatwarp` サブマクロ |
| 10 個の `buffer*` コマンド | 4 バイト | `stringvar` サブマクロ |
| `showobjectat`, `hideobjectat`, `resetobjectsubpriority` | 5 バイト | `map` サブマクロ |
| `trainerbattle` | 6 + 4..16 バイト | 型による分岐 |

`warp` は `.byte 0x39` の後に `formatwarp` が続くもので、命令は 1 つです（呼び出し先はインライン化されています）。`giveitem` は
`loadword` の後に `callstd` が続くもので、命令は 2 つです（インライン化されません）。マクロがコマンドマクロになるのは、どの経路も
独自のリテラルのオペコードバイトで始まり、ほかのコマンドマクロに到達しない場合だけです。

`trainerbattle` は、型、トレーナー、localId からなる先頭部分の後に、型によって選ばれる 1 個から 4 個のポインタが続きます。これは
`scrcmd_args.VARIABLE` です。デコンプの 10 種類以外の型は、`scrcmd.shape` が `None` を返すようにします。

0x081A7699..0x081A77A3 は `data/scripts/trainer_battle.inc` です。マクロによって `applymovement` は 7 バイト、`waitmovement` は 3 バイトになります。これらを固定幅のコマンドとして読むと、
この領域の対応がずれます。正しく読むと、次のようになります。

    0x081A76A8  4F  applymovement 0x800F, 0x081A77B0     @ VAR_LAST_TALKED, Movement_RevealTrainer
    0x081A76AF  51  waitmovement 0x0000
    0x081A76B2  26  specialvar 0x800D, 0x0036            @ VAR_RESULT, Script_HasTrainerBeenFought

これは `EventScript_TryDoNormalTrainerBattle` [data/scripts/trainer_battle.inc:8] です。この領域は端から端までラベルで区切られ、
7 個の境界テストになります。`goto` は `end` と `return` と並ぶ終端です。
`EventScript_NoTrainerBattle` が、その 1 つの後ろのバイトに置かれているためです [:17]。標準スクリプト 5 個が使うのは固定幅のコマンド 7 種類だけで、
テーブルを証明するには足りません。トレーナーバトル領域の 266 バイトすべてが、
`tests/test_script_cmd_table.py` のフィクスチャになっています。

## オペランドに名前を付ける

`tools/frlg/script_read.py DUMP.bin --base ADDR` で実行します。

    0x081A76A8  4F  applymovement 0x800F (VAR_LAST_TALKED), 0x081A77B0
    0x081A76B2  26  specialvar 0x800D (VAR_RESULT), 0x0036 (Script_HasTrainerBeenFought)
    0x081A76BC  06  goto_if 0x05 (!=), 0x081A76CD
    0x081A76C2  25  special 0x0038 (PlayTrainerEncounterMusic)

- どのコマンドでも、0x4000 以上のオペランドは変数の参照です。`VarGet` は `VARS_START` より小さい値をそのまま返し、
  それ以上の値では変数を読みます [event_data.c:235,
  `GetVarPointer`:214]。`additem 0x8004` は `VAR_0x8004` に入っているアイテム ID を与えます。
  `pokeldn/frlg/rom/symbol_names.py` には、`include/constants/vars.h` と
  `flags.h` から評価した 274 個の変数名と 1470 個のフラグ名があります。
- 添字が指すテーブルは、マクロのパラメーター名から決まります。`scrcmd_args.PARAMS` が各オペランドに名前を付けます
  （`special` は `function`、`setflag` は `flag`）。`function` は共有されます。`ScrCmd_special` は u16 を `gSpecials` に読み、
  `ScrCmd_callstd` は u8 を `gStdScripts` に読みます。
- `goto_if 0x05` は `!=` です。`sScriptConditionTable` の行は、<、=、>、<=、>=、!= の順に並んでいます [scrcmd.c:65]。

Altering Cave のカウンターは SaveBlock1 + 0x1048 で動きます。`GetVarPointer` は `vars[idx - VARS_START]` なので、
これは変数 0x4024、つまり `VAR_ALTERING_CAVE_WILD_SET` です。`tests/test_script_symbols.py` がその名前を保持しています。

## スクリプトを辿って到達先を調べる

`scrcmd.follow` は、入口の集合から出発して、すべての `goto`、`call`、`goto_if`、`call_if` を辿ります。そして、
ダンプが保持していない到達先のアドレスをすべて集めます。データポインタ（`text`、`movements`、多肢選択の一覧）は、
報告するだけで、辿ることはありません。`scrcmd.dump_plan` は取りこぼしを `--dump-address` の行に変換し、
取りこぼしの大きいものから順に並べます。

`charmap.decode` は名前用です（固定幅で、制御コードはなく、未知のバイトは `.` になります）。会話には
`charmap.decode_message` が必要です。`scrcmd.data_pointers` と `scrcmd.read_string` は、ダンプが保持しているものを復号します。標準スクリプト 10 個すべてについて、
コードもテキストも、コンソールから読み取っています。

既知のコマンドのオペランドがダンプの末尾を越える場合は、ダンプが短いことを意味し、読み取り側がそれを伝えます。

# 種族テーブル

`gSpeciesInfo` = 0x0824CDFC、ストライドは 28 です。

デコンプの 26 バイトのストライドで `friendship`、`growthRate`、`eggGroups` を探すと、
16 MB の中で何にも一致しません。カートリッジのストライドは 28 です。データは英語版デコンプと一致します。`CalculateMonStats`
[pokemon.c:2095] が、5 体の手持ちポケモンの種族値、レベル、個体値、努力値、性格から
再計算した能力値は、保存された 30 個の能力値をすべて再現しました。

ミュウ、セレビィ、ジラーチは、6 つの種族値がすべて 100 です。そのため `0x64646464` は、エントリのオフセット 0 と 2 にあり、
ストライドにかかわらず、そのうち 1 つはワード境界に揃っています。

    scan: 3 match(es) for 0x64646464 in 0x08000000..0x08400000
       0x0824DE80   0x0824E970   0x0824FAB8

2800 と 4424 の隙間は、28 バイトのエントリ 100 個と 158 個にあたります。基点の 60 バイト前から読んだダンプは、
デコンプと同一の 34 個のエントリ（884 バイト）を読みました。余分な 2 バイトは、
すべてのエントリにある `00 00` のパディングです。

デコンプをモデル化する際の落とし穴は次のとおりです。`[SPECIES_NONE] = {0},` は 1 行のブロックで、複数行の正規表現が飲み込んでしまいます。
`genderRatio` はマクロ `PERCENT_FEMALE(x)` です。`noFlip` は常にセットされるとは限らないビットフィールドで、16 進数の
`#define` は、10 進数のみの正規表現をすり抜けます。

## ここから `CreateMon` を見つける

`0x0824CDFC` を探すと、`&gSpeciesInfo` を保持するリテラルプールがすべて見つかります。`0x08028000..0x08048800` に 31 件あり、
0x080486B0 の `Random` より下にあります。そのうち 5 件は正確に `0x14` 離れており、1 つの関数に複数のプールがあるため、
件数は証拠になりません。オブジェクトの境界が証拠になります。最大の隙間（37 KB）は、`pokemon.o` ではなく
`battle_ai_switch_items.c:88` の下にあります。次の隙間 15.6 KB は、
`src/battle_controller_link_opponent.o` にあり、これは `gSpeciesInfo` を一度も参照しません。

`pokemon.o` のブロックで最初に一致するのは `CreateMon` [pokemon.c:1755] で、命令が一対一で一致します。

| シンボル | アドレス | 確かめ方 |
|---|---|---|
| `CreateMon` | `0x08041150` | 逆アセンブル |
| `ZeroMonData` | `0x08041090` | 最初の呼び出し |
| `CreateBoxMon` | `0x080411C0` | ZeroMonData と SetMonData の間の呼び出し |
| `SetMonData` | `0x08043A78` | 56（`MON_DATA_LEVEL`）で呼ばれ、続けて 255（`MAIL_NONE`）を伴う 64（`MON_DATA_MAIL`）で呼ばれる |
| `CalculateMonStats` | `0x08041B78` | 最後の呼び出し |

## スペイン語版ファイアレッドのカートリッジ

スペイン語版ファイアレッドの基本パッケージ v0（`0100EB702342C000`、表示バージョン 1.0.0）には、`FireRed_s.gba` が入っており、
ゲームコードは `BPRS`、リビジョンは 0x0A です。16 MiB の ROM の SHA-256 は
`d4dee5aeb5313e073d6067bee37278b0204886958467633978bb746bbe3d5b76` です。
パッケージの 4 つの NCA 署名、セクションヘッダーのハッシュ、PFS0 と IVFC のブロックハッシュ、
CNMT のコンテンツハッシュは、すべて検証に通ります。コントロールのタイトルは “Pokémon FireRed Version (Spanish Ver.)” と
“Pokémon Edición Rojo Fuego” です。

`builds.BPRS` は、ネイティブのギフト、イベントのポケモン、常駐フックのためのスペイン語版アドレスを提供します。
`vendor/gblink-cards/symbols.json` は 167 個のカードのシンボルを提供します。生成器の言語 ID は 7 です。
ホストは、ペイロードを送る前にコンソールのゲームデータから `BPRS` を選びます。スペイン語版リーフグリーン（`BPGS`）の測定済みテーブルは、
[国際版リビジョン 0x0A のカートリッジ](#the-international-revision-0x0a-cartridges) にあります。

関数は、バイト単位で同一の英語版デコンプと、一意な本体のウィンドウによって照合します。その後、
ポインタの語と THUMB の BL のオペランドをマスクした命令列で照合します。RAM またはデータのポインタは、
対応する写像済みの関数のリテラルプールから読みます。同じ命令列を共有する小さな関数には、別の参照が必要です。
`IsEnoughMoney` は 0x080A376C で `GetMoney` を呼び出し、
`SpeciesToNationalPokedexNum` は 0x080469A8 にあり、0x0824B9DE のテーブルを読みます。そのテーブルの種族 277 のエントリは
252 を返します。これらのアドレスは個別に見つけたものです。ROM のオフセットは 1 つのビルドの中でも異なります。

| シンボル | スペイン語版ファイアレッド |
|---|---|
| `gRngValue`, `gSaveBlock1Ptr`, `gSaveBlock2Ptr` | 0x03004220, 0x03004228, 0x0300422C |
| `gMain`, `gIntrTable[4]`, `gSoundInfo` | 0x030022D0, 0x03002730, 0x03005F80 |
| `VBlankIntr`, `Client_RunBufferScript` | 0x0800071C, 0x08148CD0 |
| `CreateMon`, `GetMonData3`, `Random` | 0x08041164, 0x080432F8, 0x080486C4 |
| `CB1_Overworld`, `CB2_Overworld`, `RunTextPrinters` | 0x08059E5C, 0x08059EDC, 0x08002D50 |
| `m4aSoundMain`, `ReadFlash` | 0x081E089C, 0x081E224C |
| `MapGridGetElevationAt`, `MapGridGetCollisionAt` | 0x0805C658, 0x0805C6D8 |
| いあいぎり、いわくだき、かいりきのメッセージの戻りポインタ | 0x081C1909, 0x081C19FD, 0x081C1AE6 |

ひでんわざの戻りポインタは、スクリプトの 8 バイトの `loadword` とメッセージ呼び出しの並びに続きます。
バッジの確認は、0x081C1901、0x081C19F5、0x081C1ADE のメッセージスクリプトを指します。
翻訳後のテキストの長さは、各メッセージと再開用スクリプトとの距離を変えます。

44 枚のカードはすべて、シミュレートしたホストとクライアントの会話を通して受け取られ、mGBA 上で
この ROM に対して、プレイヤーの母親に紐づけて実行されます。出荷されている 4× 速度のカードのトランポリンは、`gIntrTable[4]` に 0x0203FC01 を入れ、
0x0800071D を 0x0203FBFC に保持します。R を押す、離す、もう一度押すと、
それぞれ、追加のオーバーワールドのコールバックのペアを 3、3、0 組と、テキストの実行を 4、4、0 回ディスパッチします。
毎フレーム 1 回、元の V-blank ハンドラーを呼び出します（`tests/test_team_cards.py`）。

`tests/test_frlg_english_cartridges.py`、`tests/test_noclip.py`、`tests/test_follower.py` は、
`scratchpad/frlg_es/FireRed_s.gba` のスペイン語版イメージを取り込みます。ROM を使う確認は、イメージがないとスキップされます。
カートリッジ自身のふしぎなおくりもののクライアントは、トレーナーの確認から戻り、言語 7 のチェックサム付きの
ピカチュウを作り、`GetVarPointer` を呼び出し、常駐フックを組み込んで、セーブに保持されたフックを読み込みます。
衝突とフォロワーの確認も、スペイン語版のアドレステーブルで実行されます。

# フランス語のかんたん会話の語彙

言語に依存するかんたん会話の語 1006 個はすべて、コンソールの ROM から読み出しています。
`pokeldn/frlg/text/easychat_french_words.py` がテーブルで、`easychat_french.french(id)` はそこから答えます。

## スロットの問題

かんたん会話の id は `(group << 9) | index` で、スロットを表します。`pokeldn/frlg/text/easychat_words.py` は英語版デコンプから来ており、
英語版 ROM がそこに保持しているものを示します。各ローカライズ版 ROM には、独自の
`gEasyChatGroup_*` テーブルがあります。メール、トレーナーカードの引用、訪問したトレーナーの台詞、
`--denied-message`、アンケートの条件は、すべてフランス語版のテーブルに依存します。コンソール上で描画すると、
`EC_WORD_ENJOY` は STRESSE、`EC_WORD_DONE` は FURAX、`SPEECH/12` は LES と描画されます。

## テーブルを見つける

`sEasyChatGroups[]` は 22 個のエントリからなり、
`struct EasyChatGroup { const void *wordData; u16 numWords; u16 numEnabledWords; }`
の構造体で [src/data/easy_chat/easy_chat_groups.h:26]、各 8 バイトです。グループ 8、9、10（Endings、Feelings、
Conditions）は、語 69 個、有効な語 69 個を持つため、`0x00450045` は 8 バイトずつ離れて 3 回現れます。
`--buffer-script memory-scan --scan-word 0x00450045` を 0x08000000..0x08480000 に対して実行すると、ちょうどその 3 件が返りました。
範囲は次のとおりです。`src/easy_chat.o(.rodata)` は `ld_script.ld` でオブジェクト #99 にあり、
`src/mystery_gift_client.o(.rodata)`（#182、`sClientFuncs` は既知）より下にあります。テーブルは 0x083E3700 にあります。

22 個のエントリはすべて、英語版ビルドと同じ個数を持つため、id は両方の言語で同じスロットを指します。
語の配列とその文字列は 0x083DE2C8..0x083E3700 の範囲にあり、21560 バイトです。各グループの文字列は、その配列と
次の配列の間にあります。`string-gather` は 1 回の実行で 1 グループを読みます。エントリ 42 と 60 は、描画されたとおり STRESSE と FURAX です。
`tests/test_easychat_french.py` は、描画の証拠と ROM の証拠が一致することを要求します。

`TRAINER/11` は DRESSEUR（単数形）です。DRESSEURS はテーブルにありません。

## テーブルが示すこと

英語版テーブルとの違いは、グループによって異なります。

- `EC_GROUP_STATUS`（特性の名前 109 個）は、ほぼ完全に一致します。`stench`→PUANTEUR、`thick_fat`→ISOGRAISSE、
  `rain_dish`→CUVETTE、`drizzle`→CRACHIN、`arena_trap`→PIEGE、`rock_head`→TETE DE ROC、`air_lock`→AIR
  LOCK。
- `EC_GROUP_FEELINGS` は、ほぼ全面的に組み直されています。英語版には動詞（meet、play、eat、drink、see、
  hear、got、goes、go home）が混ざっていますが、フランス語版は感情の状態だけを持ちます。スロット 13（`disappoints`）は RAVI、44
  （`eat`）は HUMILIE、51（`drink`）は HONTEUX です。
- `EC_GROUP_SPEECH`: `but`→MAIS、`however`→CEPENDANT、`how`→COMMENT、`the`→LE が並び、LES と L' は
  英語版が `case` と `miss` に使っているスロットを取っています。

固有名詞のグループはほとんど違いがなく、一般的な語彙は大きく違います。

807 個のスロットは読み取る必要がありません。`EC_GROUP_POKEMON`、`POKEMON_2`、`MOVE_1`、`MOVE_2` は、
`gSpeciesNames` / `gMoveNames` [easy_chat.c:155] から出力され、コンソールがローカライズします。`easychat.species_word(55)`
と `easychat.move_word(177)` がそれらを組み立て、`easychat.is_language_safe` がそれを認識します。

## 使い方

```python
from pokeldn.frlg.text import easychat, easychat_french
easychat_french.french(easychat.WORDS["enjoy"])     # 'STRESSE', not 'enjoy'
easychat_french.render(ids)                          # the line as the console will print it
easychat_french.check(ids, strict=True)              # raises on anything unread
```

`check` は、語ではない id を検出します。たとえば、グループの末尾を越えた添字などです。

    POKELDN_RADIO=esp32:auto ./.venv/bin/python -u bin/frlg_mg_host.py --live --keys PROD_KEYS \
        --buffer-script string-gather --gather-address 0x083DF5C0 --gather-count 42 \
        --gather-stride 12 --dump-file DUMP.bin --version firered    # one group per run
    ./.venv/bin/python scratchpad/ec_words.py --group 4 DUMP.bin
    ./.venv/bin/python scratchpad/ec_words.py --report

`scratchpad/ec_locate.py` は、スキャンの答えからテーブルを見つけ、ダンプをデコンプの個数と照合します。
`scratchpad/ec_words.py` は、22 個の語配列のアドレスを保持しています。

リーフグリーンのかんたん会話の領域全体は、ファイアレッドのものを −0x1C4 ずらしたもので、語彙は同一です。22 個のエントリ
すべての個数が等しく、1 グループの `string-gather` は同じスロットに同じ語を読み出します。
`easychat_french` は両方のコンソールに答えます。

## 国際版リビジョン 0x0A のカートリッジ

6 つの言語版は、それぞれヘッダーのリビジョンが `0x0A` の 16 MiB の GBA ROM を含みます。ホストは、英語、フランス語、ドイツ語、イタリア語、スペイン語、日本語の各言語版について、
両方のバージョンのアドレステーブルを持っています。

関数は、一意な命令のウィンドウによって英語版リビジョン 0x0A の ELF と対にします。必要に応じて、相対呼び出しとポインタのリテラルはマスクします。
RAM のグローバル変数とポインタテーブルは、対応するリテラルプールから読みます。
ひでんわざのスクリプトのファミリーは、ROM のポインタをマスクしたコマンドの並びとして対にします。
ローカライズされたテキストは、その並びに従います。
`SpeciesToNationalPokedexNum` は、隣接する 2 つの変換関数の両方と対にします。
3 つの葉関数が同じ命令列を共有するためです。

| カートリッジ | ROM ファイル | SHA-256 |
|---|---|---|
| `BPRD` | `FireRed_d.gba` | `04f43a43f7cf9109561cd1744d108f50202c931ce31b30abaa6f18950824d20d` |
| `BPRE` | `FireRed_e.gba` | `d32c8df8702293716ad8de68755fe5903161f4829a0a2e486bfa7e74a0b62e94` |
| `BPRF` | `FireRed_f.gba` | `9edf7a3137536b0ccf024a732025da9a0ea1330eb9fd13b2ffed19f5e008c147` |
| `BPRI` | `FireRed_i.gba` | `cf7fa25cf57fbf12f24709efdd1d1aa8056efb4e9b6520ac7d068d3de13ff135` |
| `BPRJ` | `FireRed_j.gba` | `e2cdfb0415ef09e887e9d27b925ff02a713a92f004843490cf6b88b68db6cd01` |
| `BPRS` | `FireRed_s.gba` | `d4dee5aeb5313e073d6067bee37278b0204886958467633978bb746bbe3d5b76` |
| `BPGD` | `LeafGreen_d.gba` | `eb0e340b5efb3ccb7bab104183d2050fa834261dd816c5ad8221e4e98c04342b` |
| `BPGE` | `LeafGreen_e.gba` | `993a8a5695a4f7e4dfe8ceec55beb43e020e82b492854d457d4aa3f864d20f08` |
| `BPGF` | `LeafGreen_f.gba` | `751346c16c0cf3a3ec601d6c2d3c482b54609c5d00ca74f2f668d75b31b8efde` |
| `BPGI` | `LeafGreen_i.gba` | `b99b9c58c83a61b5bd8a1a69b6f21ce0cfab8d1f6065b53b81a323801c9b2895` |
| `BPGJ` | `LeafGreen_j.gba` | `a2aa939a23a36610902be51169042efac78f2f6db6a433e06443a2ee09d4f19e` |
| `BPGS` | `LeafGreen_s.gba` | `a943ecfd6560115b184dc244daed71ba328a67334f343eb35b39932c3ce7c731` |

| シンボル | ドイツ語版ファイアレッド | ドイツ語版リーフグリーン | イタリア語版ファイアレッド | イタリア語版リーフグリーン | スペイン語版リーフグリーン |
|---|---|---|---|---|---|
| `gmain` | `0x030022D0` | `0x030022D0` | `0x030022D0` | `0x030022D0` | `0x030022D0` |
| `sb1ptr` | `0x03004228` | `0x03004228` | `0x03004228` | `0x03004228` | `0x03004228` |
| `sb2ptr` | `0x0300422C` | `0x0300422C` | `0x0300422C` | `0x0300422C` | `0x0300422C` |
| `rng` | `0x03004220` | `0x03004220` | `0x03004220` | `0x03004220` | `0x03004220` |
| `vblank_intr` | `0x08000730` | `0x08000730` | `0x08000730` | `0x08000730` | `0x0800071C` |
| `create_mon` | `0x08041178` | `0x08041178` | `0x08041164` | `0x08041164` | `0x08041164` |
| `client_run_buffer_script` | `0x08148BA4` | `0x08148B80` | `0x08148BE4` | `0x08148BC0` | `0x08148CAC` |
| `save_slot_layout` | `0x083FCEC0` | `0x083FCCFC` | `0x083F464C` | `0x083F4488` | `0x083F7674` |

ROM を使うテストは、対応する各カートリッジの `Client_RunBufferScript`、`CreateMon`、
`GetVarPointer`、`VBlankIntr`、保存されたフックのローダーを実行します。衝突のテストは、フックを組み込んだ後、カートリッジの
`MapGridGetCollisionAt` と `MapGridGetElevationAt` を呼び出します。

スペイン語版ファイアレッドの `EventScript_CurrentTooFast` は `0x081AA346` から始まり、`lockall`（`0x69`）があります。
以前のカードのテーブルは、続く `loadword`（`0x0F`）の `0x081AA347` を指していました。生成された
HM のカードは、今はスクリプトの入口を使います。

### 日本語版のレイアウト

日本語版のファイアレッドとリーフグリーンは、異なる EWRAM のレイアウトを使います。手持ちは
`0x020241E0` から始まり、プレイヤーのアバターは `0x02036FA8`、オブジェクトイベントは `0x02036D68`、パレットのフェードは
`0x020379E8`、野生ポケモンとの遭遇を無効にするバイトは `0x02038624` にあります。
`0x08002D38` の `RunTextPrinters` は、`0x08002D60` のリテラルから配列のポインタ `0x02020030` を読み、
`0x08002D94` で `0x20` ずつ進みます。国際版の配列は `0x24` ずつ進みます。

日本語版の両方のカートリッジは、`gMain` を `0x030022E0`、`gSaveBlock1Ptr` を `0x03004238`、
`gSaveBlock2Ptr` を `0x0300423C`、`gRngValue` を `0x03004230`、`gIntrTable[4]` を `0x03002740` に置いて使います。
`gSpecialVar_0x8000` は `0x02036FE8`、手持ちの数は `0x02023F85` にあります。

日本語版の SaveBlock1 は `0x3D40` バイト、国際版の SaveBlock1 は `0x3D68` バイトです。最後のチャンク、
セクター ID 4 は、日本語版では `0xEC0` バイト、国際版では `0xEE8` バイトを扱います。セーブの注入と
ネイティブのフラッシュへのパッチは、選ばれたカートリッジの長さを使います。`RamScript` はそのオフセットを保持します。
チェックサムは `0x361C`、マジック値は `0x3620`、スクリプト本体は `0x3624` です。チャンクの長さは、ROM を使ったチェックサムの確認で、
各 ROM の `sSaveSlotLayout` から読まれます。

日本語版のふしぎなカードは 164 バイト、ふしぎなニュースは 224 バイトを占めます。`BPRJ` の
`SaveWonderCard`（`0x081481F4`）は 164 バイトをコピーしてチェックサムを取ります。`BPGJ` は `0x081481CC` を使います。
対応する検証ルーチンは `0x08148250` と `0x08148228` です。どちらのルーチンも、ROM を使った確認で、
アプリが生成したカードを保存して検証します。カードの CRC は SaveBlock1 の
`+0x3204` にあり、データは `+0x3208` にあります。ニュースのデータは `+0x3124` から始まります。
`BufferCardText`（`0x08149934`）は、タイトルのバイト 10..27、サブタイトル 28..40、20 バイトの本文 4 行（41..120）、
20 バイトのフッター 2 行（121..160）を読みます。内蔵の配布では、日本語版のカートリッジ向けにこのコンパクトな
レイアウトを使います。ネイティブの `.wc3` レイアウトは [ふしぎなおくりもののファイル](gifts.md#native-formats) に記載されています。

日本語版のカートリッジは、別のアクティビティ番号の下で、ふしぎなおくりもののフレンドを一覧に出します。その
ふしぎなカードとふしぎなニュースの `sAcceptedActivityIds` のエントリは、6 と 7 を保持しています（`BPRJ` は
`0x08410EC4` と `0x08410EC8`、`BPGJ` は `0x08410E4C` と `0x08410E50`）。ほかのカートリッジは、それぞれ 21 と
22 を保持します [src/data/union_room.h:405-406]。`LINK_GROUP_UNK_11` の一覧も、同じ箇所に 6 と 7 を持ちます。
21 を告知するホストは、日本語版のフレンドの画面には決して現れません。受け入れられる RFU のシリアルは、
`{0x0002, 0x7F7F}`（`BPRJ` は `0x083FC378`）であり、国際版の `{0x0002, 0x7F7D}` とは異なります。
Switch の実行ファイルは、日本語版とフランス語版のファイアレッドで同じコードです。両者の違いは、
文字列と ROM のパスだけで、12 個のタイトルすべてが、最初のローカル通信 ID として
`0x01006fa0233f8000` を列挙しています。

日本語版の両方のカートリッジは、ネイティブのクライアント、`CreateMon`、変数の参照、VBlank ハンドラー、
保存されたフックのローダーを Unicorn の下で実行します。`CreateMon` は、種族番号 25 に対して日本語のニックネームのピカチュウを作ります。
日本語の名前はかな表を使い、トレーナー名の上限は 5 文字です。
ラテン文字のアクセントは、かなと同じバイト値を共有します。内蔵のギフトの文章はローマ字のままで、
日本語版のカートリッジではアクセントを除きます。会話の文章は 26 文字で折り返され、2 行ごとにページを改めます。
日本語版の会話フォントのほうが幅広いためです。日本語版のチームのカードは、そのソースの日本語の構造体の
サイズと、動的なメニューの幅を使います。2 つのプロンプトは、RAM スクリプトの上限に収まるよう短くしています。

追加されたドイツ語版とイタリア語版の組、スペイン語版リーフグリーン、日本語版の組は、mGBA でのカードの確認を持ちます。
これらは、抽出したカートリッジの ROM に対するオフラインの確認です。

### 市販機での検証

| カートリッジ | 市販機での検証 | 出典 |
|---|---|---|
| イタリア語版リーフグリーン（`BPGI`、リビジョン `0x0A`） | 41 個の `Build` フィールドすべてがコンソールのメモリと一致、フォロワーの導入、マスターボールのカードとアイテムの配信 | [投稿者による検証](https://github.com/Decryptu/pokeldn/issues/13) |
