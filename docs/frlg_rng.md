---
title: 乱数生成器
parent: FireRed and LeafGreen
nav_order: 5
---

# gRngValue: 読み取り・予測・狙い撃ち

ここに載せた値はすべて、フランス語版ファイアレッド (BPRF のソフトウェアバージョン 0x0A) で計測しました。

## 生成器

```c
u16 Random(void) { gRngValue = 1103515245 * gRngValue + 24691; return gRngValue >> 16; }
void SeedRng(u16 seed) { gRngValue = seed; }
```

[src/random.c, include/random.h:18]

| シンボル | アドレス | 取得方法 |
|---|---|---|
| `gRngValue` | `0x03004220` | `Random` のリテラルプール |
| `Random` | `0x080486B0` | 4 MB を `RAND_MULT` = 0x41C64E6D で走査 |
| `SeedRng` | `0x080486D0` | そのプールが `gRngValue` の名前を再び参照している |
| `gSpecialVars` | `0x081639A8` | 2.75 MB 中で 2 ずつ増える 12 ワードの連続領域はこれだけだった |
| `gSpecialVar_0x8000` | `0x020370B4` | その連続領域の先頭要素 |

ARM 命令にも THUMB 命令にも `RAND_MULT` を直接埋め込むものはないため、この定数は `Random` のプールで `&gRngValue` の隣に置かれている。
走査では 11 件がヒットした。`ld_script.ld` は `src/random.o` を #86 に、定数のもう一つの使用元である
`src/title_screen.o` を #123 に配置しているので、最も若い番号のヒットが random.o のものになる。ダンプは `Random` の命令を
一つずつ [random.c:9-13] と照合すると一致し、`SeedRng` も同じプールワードに続く。

`Random` は状態の上位半分を返す。性格値 (2 回の抽選) からは 2<sup>16</sup>
個の候補状態が残る。`pokeldn/frlg/rom/lcg.py` が演算処理を担っており、`distance(a, b)` はどの範囲でも厳密に求まり、
baby-step/giant-step 法で (2<sup>17</sup> 回の演算)。この写像は全 2<sup>32</sup> 状態を並べ替えるので、
距離は常に存在するが、小さいときにだけ根拠になる (偶然の確率は N / 2<sup>32</sup>)。

`gRngValue` と `gSpecialVar_0x8000` はリンク時に決まるグローバル変数で、移動しない。セーブブロックのアドレスは
動く。`SetSaveBlocksPointers` が戦闘とロードのたびに 4 の倍数に揃えたオフセットを振り直すため [load_save.c:75]。

## 速度: 1 フレームあたりちょうど 2 ターン

`ScrCmd_delay` は正確に N フレーム後に再開する [scrcmd.c:651]。`--gift rng-rate-probe` は
`gRngValue` を読み、N フレーム待ってから再び読む。`rng_script.measure_rate` は `lcg.distance` を N で割る。

| フレーム数 (厳密) | ターン数 (厳密) | 2N + 2 |
|---|---|---|
| 600 | 1,202 | 1202 |
| 3000 | 6,002 | 6002 |

+2 は `delay` の前後にある余分な 1 フレーム分である。1 フレームあたり 2.003333 とするモデルは N=600 には合うが、
N=3000 では 6010 と予測してしまう。ふしぎなおくりもののリンクメニューで 1 フレームに 1 回 `gRngValue` を `rng-trace` で採取したところ、
間隔はすべて正確に 2 だった (95 件中 95 件)。ある読み取りから n フレーム後の状態は `advance(S, 2n)` である。

プローブは `delay` の間、操作不能のプレイヤーを計測する (`lock=False` なら操作可能な状態を計測)。通常のフィールド移動中のターン数は
偶数になる。カウントは 2 回の読み取り (`distance`) か、結果として出現するポケモン (`recover_wild_state`) から得る。
1 ターンは約 8 ms であるため、手で計った秒数では決まらない。

## シードの出所と、持ち越せない理由

```c
void SeedRngAndSetTrainerId(void) { u16 val = REG_TM1CNT_L; SeedRng(val); gTrainerId = val; }
```

[main.c:264]。フェードの後、`Task_TitleScreenMain` から
`SetMainCallback2(CB2_InitMainMenu)` の直前に呼ばれる [title_screen.c:735]。`StartTimer1` は `CB2_InitTitleScreen`
[:351] で動く。シードは START 押下時にサンプリングされる自走タイマーの値で、取りうる値は 65536 通り。

通信中に設定したシードは残らない。ふしぎなおくりもののメニューから戻ると
`MainCB_FreeAllBuffersAndReturnToInitTitleScreen` → `CB2_InitTitleScreen` [mystery_gift_menu.c:463] が実行され、
START でシードが設定し直される。ふしぎなおくりもののメニューからフィールドへ至る経路に、これを避けるものはない。そこで設定した `0xC0DE` は、
次の遭遇時の状態の祖先ではない (1,898,278,119 ターン離れていた)。

| `SeedRng` の呼び出し箇所 | いつ |
|---|---|
| `SeedRngAndSetTrainerId` [title_screen.c:735] | タイトル画面 |
| `LinkTestScreen` [link.c:318] | 未使用のデバッグ画面 |
| `Debug_RfuIdle` [link_rfu_2.c:2670] | 未使用のデバッグ画面 |
| `RfuMain1` [link_rfu_2.c:2116] | Switch 専用で、Sloop のシステムコールによって制御される |

```c
if ((svc_4b() & SVC4B_RESEED_RNG) != 0)
    SeedRng(ReadU16(&GetHostRfuGameData()->compatibility.playerTrainerId));
```

[link_rfu_2.c:2114, `#if REVISION >= 0xA`]。RFU が有効な間、`RfuMain1` は毎フレーム実行されるので、ビットが立つと
状態は公開されている `playerTrainerId` の付近に固定されるはずである。しかし計測したどの状態もそこから派生していない。
ふしぎなおくりもののメニューで採取した状態は、本体の
`playerTrainerId` から 13億7000万ターン、ユニオンサークルでの通信後のエンカウントで採取した状態は 21億ターン離れていた。
ビットがいつ立つのかは分かっていない。

START のタイミングではシードを選べない。タイマー 1 は F/1 で動き、1 フレームは 280,896 サイクルなので、フレーム境界に揃えた
読み取りはすべて `gcd(280896 mod 65536, 65536) = 64` の倍数になる。復元したシード `0xB8C0`、
`0x3742`、`0x8E94`、`0x1376` は 64 で割った余りが 0、2、20、54 であり、読み取りにはフレーム未満のゆらぎがある。

## ポケモンから、それを生んだ状態を逆算する

`GenerateWildMon` は `CreateMonWithNature(..., USE_RANDOM_IVS, Random() % NUM_NATURES)` を呼ぶ
[wild_encounter.c:233]。ナチュラルが一致するまで性格値を振り直し、その後で個体値を振る。野生ポケモンは
4 回の抽選からなり、性格値の下位、性格値の上位、HP/ATK/DEF、SPEED/SPATK/SPDEF の順に引かれる。個体値の 2 回の抽選が
30 ビットの検証情報を加え、状態は 1 つだけ残る (`lcg.recover_wild_state`)。`Random32()`、
`(Random() | (Random() << 16))` は、両方の呼び出し箇所で下位半分を先に引く。

両方の間隔を検索する必要がある。配置はポケモンごとに異なる。製品版で観測したもの:

| 方式 | 個体値の前の間隔 | 個体値の間の間隔 | 観測対象 |
|---|---|---|---|
| 1 | 0 | 0 | スクリプトのメタモン、スクリプトのコイキング (2) |
| 2 | 1 | 0 | 野生のビードル、スクリプトのコイキング |
| 4 | 0 | 1 | 野生のキャタピー、ビードル、マンキー、スクリプトのコイキング |

片方の間隔だけで検索すると、もう一方の配置については「この個体を作る状態はない」と報告される。余分な抽選は
`CreateBoxMon` のどの行にも現れず、出所は分かっていない。スクリプトによる遭遇では断続的に起きる。色違いでようき、SPEED >= 20 を要求するスタブは
SPEED 10 の色違いようきのコイキングを生成した。2<sup>32</sup> 個の状態のうち、次の 2 回の抽選でその PID になるものは
ちょうど 1 つだけあり、次のとおり:

    state 0x429D2189
      draws 3,4 -> 15/0/12/25/7/14      what the stub tested: SPEED 25, passes
      draws 4,5 -> 25/7/14/10/10/30     the mon that appeared

RNG に関する主張をする前に、ポケモンの PID と個体値はサマリー画面の 6 つの能力値と照らし合わせて確認している。

## スクリプトによる野生戦

```
setptr b0..b3 -> 0x03004220      gRngValue = seed          (opcode 0x11)
setwildbattle <species> <level>  CreateMon rolls PID, PID, IV, IV   (0xB6)
dowildbattle                     the battle starts          (0xB7)
```

`ScrCmd_setptr` は即値のバイトを絶対アドレスに書き込む [scrcmd.c:300]。`setwildbattle` は
`CreateScriptedWildMon` → `CreateMon(&gEnemyParty[0], species, level, 32, 0, 0, OT_ID_PLAYER_ID, 0)` を呼ぶ
[script_pokemon_util.c:128]。個体値はランダム、性格値は固定なし、ナチュラルの振り直しなし、素の 4 回抽選 (方式 1) である。
両コマンドとも FALSE を返し、フィールドエンジンは TRUE が返るまでコマンドを実行し続けるので、4 つの
`setptr` と生成処理が 1 フレームで実行される。間に制御を譲る命令を置いてはならない (`playse` を挟むと
壊れる)。テストでそれが無いことを確認している。

`initramscript` でマップオブジェクトに結び付けた RAM スクリプトとして配布する。終端は `endram` (0x0d) ではなく
`end` (0x02) にして結び付けが残り、再度発火するようにする。`setwildbattle` には草むらもエンカウント判定も要らない。
本体がシードを見る前にオフラインで予測した、マサラタウンの色違い Lv50 メタモン:

```
PREDICTED   PID 0x026F38B2   nature 17   IVs 31/23/27/18/30/30   shiny
ACTUAL      PID 0x026F38B2   nature 17   IVs 31/23/27/18/30/30   shiny
```

### 誰もシードを設定していない状態から予測したポケモン

何も書き込まないスクリプトは、現在の状態を読み、表示し、ポケモンを生成して、もう一度表示する:

```
BEFORE  0x9A4F5DAA        (read off the console)
AFTER   0x8EEB8648
```

`BEFORE` の値だけから予測したポケモンは、`gPlayerParty` から出力したものと 7 項目すべてが一致した。PID
0x0BF87DD1、ナチュラル 13 (ようき)、色違いではない、個体値 25/10/28/9/19/3。4 回の抽選は読み取った状態から始まり、
オフセットはゼロである。

`distance(BEFORE, AFTER)` は 6 だが、`CreateBoxMon` が消費するのは 4 である。内訳は `Random32()` に 2、OT に 0 (プレイヤーが OT のため)
[pokemon.c:1796]、個体値に 2 [:1836,1845]。残りの 2 (フィールドでの 1 フレーム分の消費) は
生成の後に入る。

## シードを読み上げる NPC

`--gift rng-seed-reader`、フラグ ID 1015。コマンド 6 つを `initramscript` で RAM スクリプトとして一度だけ登録し、
結び付けが残るよう `end` で終える:

```
copybyte gSpecialVar_0x8000+0, 0x03004220      (opcode 0x15, byte at any address to any address)
copybyte gSpecialVar_0x8000+1, 0x03004221
copybyte gSpecialVar_0x8001+0, 0x03004222
copybyte gSpecialVar_0x8001+1, 0x03004223
buffernumberstring 0, VAR_0x8000               (0x83)
msgbox                                          the NPC prints the value
```

状態は一切変更しない。`rng_script.seed_from_printed(low, high)` が表示された値から 1 ワードを組み立て直す。

- 読み取りは不可分である。乱数は止まらないので、バイトのコピーを複数フレームに分けると、本体が一度も保持したことのない値になって裂けてしまう。
  `copybyte` と `buffernumberstring` は FALSE を返すため、6 つすべてが 1 フレームで実行される。
  テストで、その間に制御を譲るものが無いことを確認している。
- テキストのポインタは相対指定である。RAM スクリプトは `gSaveBlock1Ptr->ramScript` に置かれ、その基準アドレスは
  戦闘とロードのたびに振り直される。`setvaddress` (0xB8) は
  `sAddressOffset = addr2 - (ctx->scriptPtr - 1)` を設定し [scrcmd.c:171]、`vmessage` (0xBD) がそれを引く。
  そのため、オペランドはスクリプト本体内のオフセットになる。
- `buffernumberstring` は `u16` を表示する [scrcmd.c:1678]。そのためシードは 2 つの変数と 2 行を使う。

約 20 秒離れた 2 回の読み取り結果:

```
reading 1   RNG HI 4685   RNG LO 26687   -> 0x124D683F
reading 2   RNG HI 54871  RNG LO 55616   -> 0xD657D940
distance    2,595 turns
```

無関係な 2 つのワードは約 2<sup>31</sup> 離れている。2,595 が偶然に起きる確率は 1,655,093 分の 1 である
(`rng_script.check_two_readings`)。

プレイヤーの母親 (group 4、map 0、object 1) に結び付ける: `MOVEMENT_TYPE_FACE_LEFT`、フラグ 0 のため決して非表示にならず、
プレイヤーから 1 歩の位置にある。マサラタウンの両方のオブジェクトイベントは `MOVEMENT_TYPE_WANDER_AROUND` で、
[data/maps/PalletTown/map.json] のとおり、カウントダウンの途中で歩いて離れてしまう。

## 人間がAボタンをどれだけ正確に押せるか

1 人のプレイヤーによる 4 回の押下。目標は 30.00 秒先で、シードを表示する NPC から読み取った:

| 試行 | 経過フレーム数 | 1791.8 との誤差 |
|---|---|---|
| 1 | 1801 | +9.2 |
| 2 | 1807 | +15.2 |
| 3 | 1800 | +8.2 |
| 4 | 1796 | +4.2 |

平均 +9.2 フレーム (打ち消し合う固定のずれ)、標準偏差 4.5、範囲 11。4 回すべてのターン数は
偶数である。

色違いのフレームは約 8192 フレームごと (約 137 秒) に来る。標準偏差 4.5 フレームの押下で選んだ 1 フレームに当たる確率は
約 9% なので、手で狙った色違いは約 25 分で得られ、ランダムな遭遇の約 23 時間と比べて短い。
外れは厳密に計測される。`pokeldn/frlg/rom/rng_countdown.py` がカウントダウンを担い、`--aimed-at STATE` は外れを符号付きのフレーム数に変換する。
RAM スクリプトのスロットがふしぎなカードで埋まっているときは、この方法が引き続き経路になる。

## 検索を行うスタブ

`--gift rng-shiny-hunt`、`pokeldn/frlg/rom/native_script.py`、`asm/field/shiny-seek.s`。このページのスタブはそれぞれ
製品版のハードウェアで検証済みである:

| スタブ | ギフト | 製品版での検証 |
|---|---|---|
| `shiny-seek.s` | `rng-shiny-hunt` | 母親から受け取った色違いのメタモン (2 つの異なる状態から) |
| `mon-seek.s` | `rng-mon-hunt` | Lv5 のコイキングで、色違い、ようき、SPEED 個体値 >= 20 |
| `mon-seek-far.s` | `rng-mon-hunt-far` | 色違いのようきのコイキング。559 バイトの埋め草が届いたことを示す |
| `mon-seek-both.s` | `rng-mon-hunt-both` | 色違い、ようき、SPEED 22。リーフグリーンで色違いのミュウツー |

```c
bool8 ScrCmd_setptr(struct ScriptContext * ctx)          // 0x11
{ u8 value = ScriptReadByte(ctx); *(u8 *)ScriptReadWord(ctx) = value; }
bool8 ScrCmd_callnative(struct ScriptContext * ctx)      // 0x23
{ void (*func)(void) = ((void (*)(void))ScriptReadWord(ctx)); func(); return FALSE; }
```

[scrcmd.c:300, :120]

`setptr` は任意の場所に 1 バイト書き込み、`callnative` がそれを実行する。そのため RAM スクリプトはコードを EWRAM に用意し、
フィールド上で実行する。そこには `CLI_RUN_BUFFER_SCRIPT` が届かない。`notblisy/RUBYSAPPHIREDLC` は
ルビー・サファイアで同じことを行う (`writebytetoaddr` + `callasm`、PID が色違いになるまで LCG をループする)。

`CreateScriptedWildMon` では `hasFixedPersonality` が 0 なので、性格値は `Random32()` で、
`OT_ID_PLAYER_ID` は抽選を消費しない。色違いかどうかは、状態の後の最初の 2 回の抽選で決まる。`setptr`、
`callnative`、`setwildbattle` は FALSE を返すので、スタブが `gRngValue` に残した状態が
`CreateScriptedWildMon` が消費する状態である。

スタブは `playerTrainerId` を `gSaveBlock2Ptr + 0x0A` から読む (固定の IWRAM アドレスにあるポインタ)。そのため
同じバイト列がファイアレッドとリーフグリーンの両方で使える。書き込むのは 1 ワードの `gRngValue` だけである。検索には上限があり、
尽きた場合は `gRngValue` に手を付けず、遭遇は通常どおりになる。すべてのスタブは、
配置する前に unicorn 上で実行され (`tests/test_native_script.py`)、その答えは `rng_countdown` と照合される。

### RAM スクリプトは戦闘から戻らないことがある

`CB2_InitBattle` と `InitOverworldBgs` は `MoveSaveBlocks_ResetHeap` を呼び [battle_main.c:614,
overworld.c:1337]、`gSaveBlock1` を 0 から 124 の範囲で 4 の倍数だけ振り直す [`SAVEBLOCK_MOVE_RANGE` 128,
load_save.c:75]。エンジンは `gSaveBlock1Ptr->ramScript.data.script` へのポインタを保持しており、
[`GetRamScript`, script.c:514] を戦闘をまたいで保持する。そのためエンジンは、スクリプトがもう無い場所から再開してしまい、
`dowildbattle` より後の処理には到達できなくなる。

症状: 余分な 2 回目の戦闘 (0xB6/0xB7 に着地)、きれいな終了 (ゼロ埋めに着地、`nop`)、
A、B、START のどれも効かずフィールドが固まる (995 バイト中の 972 バイト目の再開地点が 6 バイトの
`setptr` レコードの内部に着地し、永久に待つコマンドとしてデコードされる)。`releaseall` + `end` では直らない。

修正では、セーブブロックの外から戦闘を開始し、10 バイトで済ませる [`rng_script.battle_and_exit`]:

    setvar 0x8000, 0x02B7      ->  0x020370B4: B7 02  =  dowildbattle ; end
    goto   0x020370B4

`gSpecialVar_0x8000` は移動せず、戦闘やフィールドのコードのどこも書き込まない。
スクリプトが停止すると `ScriptContext_RunScript` が `UnlockPlayerFieldControls()` を呼ぶ [script.c:335]。そのため
`end` で制御が戻る。戦闘やマップのロードをまたいでセーブブロックのアドレスに依存する RAM スクリプトは作ってはならない。

## ナチュラルと個体値の選択

`--gift rng-mon-hunt`、`asm/field/mon-seek.s`、フラグ ID 1019。4 回の抽選はすべて検査される。
性格値が色違いとナチュラルを決め (`personality % 25` [pokemon.c:5020])、抽選 3 と 4 は
個体値になる [pokemon.c:1836、HP/ATK/DEF、続いて SPE/SPATK/SPDEF]。

    --hunt-nature adamant,jolly   --hunt-iv speed=31 --hunt-iv attack=20   --hunt-cap N

捕まえたコイキングは、PID 0x01503B8A、色違い値 4、ようき、個体値 6/2/25/28/12/7 と読み戻される。
`lcg.recover_wild_state` は状態 0x7041F74F を与え、`rng_countdown` は全項目を再現する。

色違いは 15 命令のホットループで検査される。25 による除算と個体値の比較は
8192 状態に 1 回しか走らないので、条件を足しても 1 回あたりの速度は落ちず、必要な反復回数だけが増える:

| 指定条件 | 状態 N 個に 1 つ | 典型的な固まり時間 | 上限での最悪時間 |
|---|---|---|---|
| 色違い | 8,192 | 0.02 s | 0.10 s |
| 色違い + ナチュラル 1 種 | 204,800 | 0.55 s | 2.5 s |
| 色違い + ナチュラル + 個体値 1 つ >= 20 | 546,133 | 1.5 s | 6.7 s |
| 色違い + 個体値 2 つ = 31 | 8,388,608 | 22 s | 拒否 |

`native_script.search_cost` がこれを計算する。ホストは最悪時間が
`--hunt-freeze-frames` (既定値 900、約 15 秒) を超えるものは拒否する。検索している間、プレイヤーには
音楽が流れたままの静止画面が見える。

`mon-seek.s` は 163 バイトの配置枠のうち 160 バイトを使う。色違い値が自分自身の逆元になるので `pidLo` は
2 命令で戻り、除数はそれ自体がループカウンタを兼ね、個体値の下限には 30 で終端するビットを持たせ、
両方のフィールドをビット 27..31 にシフトして、5 ビットの比較が符号なしになるようにしている。

## スクリプト本体内のペイロード

`setptr` は 1 コードバイトあたり 6 スクリプトバイト (オペコード、即値、4 バイトのアドレス) を消費し、995 バイトの RAM
スクリプト本体には約 162 バイトのコードしか入らない。エンジンは本体をその場で実行し、最後の
コマンドより先は決して読まない:

```c
const u8 *GetRamScript(u8 objectId, const u8 *script)
{ ... return scriptData->script; }
```

[script.c:514]。`gSaveBlock1Ptr->ramScript.data.script` から切り出されるので、最後のコマンドの後ろに追加したバイトは
1 スクリプトバイトずつ格納領域として届く。セーブブロックのオフセットは、スクリプトが動くフレームの間は固定で、
`&gSaveBlock1Ptr` (0x03004228) がそれを保持するので、実行時の読み取りは正確に狙える。
配置されるのは `asm/field/ram-jump.s` (36 バイト) だけである:

| | 配置 | 本体 |
|---|---|---|
| ペイロード 1 バイトあたりの消費 | 6 スクリプトバイト | 1 スクリプトバイト |
| 995 バイト本体の空き | 162 バイトのコード | 755 バイト |

    setptr x36    the trampoline, into gDecompressionBuffer        216 bytes
    callnative    -> trampoline -> payload -> back                   5
    setwildbattle / setvar / goto                                   16
    pad to a multiple of four                                        3
    payload                                                        755

トランポリンは末尾分岐する (`bx r0`、`lr` は変更しない)。そのためペイロードの `pop {r4-r7, pc}` は
`ScrCmd_callnative` の呼び出し元に戻る。最初に `ramScript.data.magic` が `RAM_SCRIPT_MAGIC` = 51 であるかを確認する
[script.c:12]。オフセットが誤っていた場合は戻り、遭遇は通常どおりになる。

### 4 バイト境界への揃え

THUMB の `ldr rN, [pc, #imm]` と `adr` は `Align(PC, 4)` を使う。2 バイトずれると、分岐は着地してコードは
動くが、プールのワードはすべて 2 バイト遅れて読まれる。4 バイト境界は保たれる: `offset = Random() & ((SAVEBLOCK_MOVE_RANGE - 1) & ~3)`
[load_save.c:75] は `& 0x7C` であり、`gSaveBlock1` はワード境界に揃っていて、`RAMSCRIPT_BODY_OFFSET` (0x3624) が
そのように保つ。`native_script.emulate_body_script` は、実際のスクリプトバイトを辿って
ずれを検出する (36 個の `setptr`、`callnative`、トランポリンの `gSaveBlock1Ptr` の読み取り、本体への分岐)。

### 分岐ではなくサイズを証明する

色違いの結果が確認するのは分岐であって、届いたサイズではない。`asm/field/mon-seek-far.s` の後には、
995 バイト目まで非ゼロの埋め草が続く。スタブはそれを合計し、一致したときだけ検索する。`InitRamScript` は残りをゼロで埋める
[`ClearRamScript`, script.c:495]。そのため配送が短いと `gRngValue` は手付かずのまま残る。`--gift
rng-mon-hunt-far` はスタブ 196 バイトと埋め草 559 バイトを運ぶ。

## 余分な抽選が答えを動かさないようにする検索

`asm/field/mon-seek-both.s` (232 バイト、`--gift rng-mon-hunt-both`、フラグ ID 1001) は、3 つの方式すべてを
カバーする 2 通りの配置で下限を検査する。性格値の後の抽選を d3、d4、d5 とすると:

| 方式 | 1 組目 (HP/ATK/DEF) | 2 組目 (SPE/SPATK/SPDEF) |
|---|---|---|
| 1 (きれい) | d3 | d4 |
| 2 | d4 | d5 |
| 4 | d3 | d5 |

ワード A は `d3 | d4<<15`、ワード B は `d4 | d5<<15`。両方を要求すると、1 組目の下限は d3 と
d4 に、2 組目の下限は d4 と d5 に載るので、方式 4 は独自のワードなしで通る。

個体値の項だけが二乗になる。色違い + ようき + SPEED >= 20 は、出現確率が 546,000 分の 1 から 1,456,000 分の 1 になり、
典型的には約 4 秒である。上限は 95% とする。外れのコストは A ボタンを 1 回押すことだが、99% にすると運の悪い
場合に 18 秒かかるためである。

状態 0xFCB5674F は、製品版で方式 1 により色違い、ようき、SPEED 22 を与え、どの方式も通る:

| 方式 | 個体値 | 下限 |
|---|---|---|
| 1 (きれい) | 4/1/10/22/14/21 | OK、本体が作ったもの |
| 2 | 22/14/21/25/1/18 | OK |
| 4 | 4/1/10/25/1/18 | OK |

状態 0x4FB97B07 は PID を正確に予測し、個体値は方式 4 によるもの (SPEED 21、下限 20):

      Method 1 (clean)  25/10/30/20/ 9/25
      Method 2 (stray)  20/ 9/25/21/ 3/ 1
      Method 4          25/10/30/21/ 3/ 1   <- the mon that appeared

## 検索が結果を書き込む場所

`asm/field/mon-seek-log.s` (288 バイト、`--gift rng-mon-hunt-log`、フラグ ID 1002) は
`{marker, start, found, iterations, cap}` を `gSaveBlock1Ptr + 0x348C` に書き込む。これは `u8 unused_348C[400]`
[include/global.h] である。`unused_348C` に書き込むコードはなく、製品版のセーブでは 0 を読む。このブロックは `ramScript` の外にあるので、
`CalculateRamScriptChecksum` は変化せず、結び付けも残る。話しかけるたびにログは上書きされる。
ログは戦闘をまたいで残り (`MoveSaveBlocks_ResetHeap` がブロックをコピーする)、セーブ時にフラッシュに届く。

    --buffer-script save-dump --dump-block sav1 --dump-offset 0x348C --dump-size 32

`native_script.decode_hunt_log` がそれを読む。尽きた検索は、マーカーとともに `found` に 0 を書き込む。

| | |
|---|---|
| 反復回数 | 603,745 |
| `lcg.distance(start, found)` | 603,745、差 0 |
| 命令数 (1 反復あたり 15) | 9,056,175 |
| 1 命令 3 サイクルとしたモデル | 1.62 s |
| プレイヤーが観測した時間 | 2-3 s |
| 推定値 | 1 命令あたり 3.7-5.6 サイクル |

`CYCLES_PER_INSTRUCTION_FROM_EWRAM` は 3 である。観測時間はストップウォッチによる計測値で、検索時間は
指数分布に従う。過小評価は、固まり時間の上限が早めに拒否する方向にしか働かない。

## リーフグリーン

スタブは移植されていないまま動く。どのリテラルも、リンク時に決まる IWRAM のワードか、リーフグリーンでも同一の定数である。
`TID ^ SID` は実行時に読まれる。ミュウツーに結び付けると、ミュウツーとの遭遇を狙うことになる。詳しくは
[リーフグリーン](frlg_leafgreen.md) を参照。
