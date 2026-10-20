---
title: The game protocol
parent: Brilliant Diamond and Shining Pearl
nav_order: 2
---

# BDSP 独自のプロトコルとキャラクターの操作

Pia のペイロードの中で BDSP は型付きのプロトコルを動かしている。`TeamLumi/opendpr` にある `Dpr.NetworkUtils.NetDataParser` は、ゲームを逆コンパイルして C# で再現したもので、全メッセージを列挙している。各メッセージは、1 バイトの `DataID` を持つ `ANetData<T>` が素の構造体 `T` を包んだ形をとる。

## フレーミング

    0x0  1  data id
    0x1  2  payload length, big-endian
    0x3  .  the struct, little-endian, packed

`JoinData`（`byte, byte, byte, short, Vector3`）は通信上 17 バイトだが、C# の既定のアライメントでは 20 バイトになる。

`NetDataParser` は 65 個のメッセージを登録している。`pokeldn/bdsp/netdata.py` はそのすべてを保持し、`scripts/gen_bdsp_netdata.py` が `opendpr` のチェックアウトから生成している。ソースで決まるレイアウトは 53 個。残りの 12 個は C# の文字列・配列・リストを含み（`netdata.OPAQUE`）、そのレイアウトはバイナリで決める。

ペイロードは、マーシャリングされた構造体である（`ANetData<T>.ConvertStructToBytes` [main.bin 0x27bb0e0]：`Marshal.SizeOf`、`AllocHGlobal`、`StructureToPtr`）。文字列と配列は固定長のフィールドとして扱われる。構造体のサイズは `Il2CppTypeDefinitionSizes` の `native_size` で、文字列の長さは `global-metadata.dat` の `fieldMarshaledSizes` にある。配列の要素数は、他のフィールドを引いた残りで決まる。正確なバイト列は `CodeRegistration.interopData` の `marshalToNative` が書き込む [1.3.0 main 0x4acd0c8、56 バイトの 0x255 エントリ]。キャプチャした不透明な 4 つのペイロードは、バイト単位で一致する（`room.NATIVE_SIZES`、`room.MEASURED`）。1.3.0 main で示すアドレスはシャイニングパールのもので、ブリリアントダイヤモンドの 1.3.0 では同じメソッドが別の位置にある。

| id | ペイロード | バイト数 | レイアウト |
|---|---|---|---|
| 0x02 | `PosListData` | 72 | `PosData`（ushort posX、ushort posZ、short rotY）を 12 個 |
| 0x13 | `TradePokeData` | 328 | 格納サイズの暗号化された PB8 1 個 |
| 0x14 | `NetRecodeData` | 694 | `RECORD` 120、`RANDOM_SEED` 132、`TvRecodeData` 204、`TV_STR_DATA` 36 を 4 個、`RECORD_HEAD` 48、int 10 個、byte 6 個 |
| 0x15 | `BallDecoData` | 143 | affixSealCount、Is3DEditMode、IsAppliedTemplate、続いて `AttachSealData` 140（`SealParam{short x, y, z; byte id}` を 20 個） |
| 0x18, 0x54 | `UgSecretBase` | 616 | short zoneID、posX、posY、byte direction、expansionStatus、int goodCount、`UgStoneStatue` 20 を 30 個、bool isEnable (4) |
| 0x22 | `StanbyListData` | 20 | `StandbyData`（isAddPlayer、hostIndex、myIndex、langId）を 5 個 |
| 0x24 | `TradeTranerData` | 32 | UTF-16 の 13 文字、uint tranerId、byte cassetVersion、byte langId |
| 0x29, 0x61 | `UgStationID_to_DigFossilIDList` | 8 | `byte DigFossilIDs[8]`。0 から 7 の順列 |
| 0x38 | `BattleMatchingPokeData` | 481 | 328 バイトの PB8、`SealParam` を 20 個（各 7）、uint attachPokemonId、uint attachPersonalRnd、byte index、num、is3DEditMode、isAppliedTemplate、affixSealCount |
| 0x42 | `NetPlayerName` | 28 | UTF-16 の 13 文字、byte genderid、byte languageId |

`BattleMatchingPokeData` の 2 つの配列は 468 バイトを 328 + 20 個の seal（計 140 バイト）に分ける。`TradePokeData` と `AttachSealData` に合う分け方はこれだけである。

`SendStandbyPlayerData`（[1.3.0 main 0x01e52fa0]）は、`UnionStateController.unionMatchWaitDataList` から 0x22 の 5 つのスロットを埋める（`isAddPlayer = 1`、`hostIndex` は 0）。リストが空なら 20 バイトすべてゼロになる。ステーションは `NetDataStandbyWaitData`（0x59、同じ 4 バイト）で自分を追加する。`59 0004 01 00 01 03`（ステーション 1、フランス語）の後に `ReciveMatchWaitData`（[0x01e539b0]）が受理すると、次の 0x22 は `22 0014 01 00 01 03` で、続く 16 バイトはゼロになる。`isAddPlayer = 0` で削除される。画面は変わらない。

マーシャラーはバッファをクリアしない。固定長フィールドは、値の後ろに前回のヒープの残りかすを持つ（[交換ページ](bdsp_trade.md)）。ID は上位ニブルごとに並んでおり、下位ニブルが 0xA に届くものはない。

## これまでに電波に乗ったもの

部屋に置きっぱなしの本体は、65 個のうち 4 個を送信する（受信側は自分に戻ってきたブロードキャストを捨てる）：

| id | クラス | reliable | unreliable | サイズ |
|---|---|---|---|---|
| 0x01 | `NetJoinData` | 2070 | 0 | 17 |
| 0x02 | `NetPosData` | 0 | 60 | 72 |
| 0x12 | `NetRequestData` | 4333 | 55 | 1 |
| 0x23 | `NetDataIsMatchWaitData` | 1734 | 0 | 1 |

要求があると、本体は 0x09（1 バイト。バトルが設定されていなければ 0）と 0x22 も送る。活動中は 0x14 と 0x15 を送る。交換のメッセージは [交換ページ](bdsp_trade.md) にある。

0x23 は、存在を示すバイト 0x00 の後ろに置かれることがある（数えた複製のうち 1505 件）。これは、0x00 をメッセージの始まりとして扱う走査でのみ読める（[メッセージのフレーミング](pia.md#message-framing)）。すべてのペイロード（8000 を超える）は、長さがバイト数と正確に一致する整形済みのメッセージである。

## 本体が繰り返すメッセージ

`NetJoinData` はプレイヤーの到着を表す。`JoinData` の全体である：

| オフセット | サイズ | フィールド |
|---|---|---|
| 0x00 | 1 | `avatarId`（8 は女の子、0 は男の子。[モデル](#the-model)参照） |
| 0x01 | 1 | `colorId`、0 |
| 0x02 | 1 | `cassetVersion`、0x31 |
| 0x03 | 2 | `InitRotY`、向きを度で表す（測定では 45 の倍数）。リトルエンディアン、非アライン |
| 0x05 | 12 | `InitPos`。リトルエンディアンの float 3 つ（x、y、z） |

`NetRequestData` は 1 バイトで、欲しいメッセージの ID である。`NetDataIsMatchWaitData` `{isMatchWait = 0}` は、本体が自分のリクエストに答えるものである。本体は 2 種類のものを要求する：

| 要求 | 回数 |
|---|---|
| `NetDataIsMatchWaitData` (0x23) | 4333 |
| `NetCharacterStateData` (0x04) | 55 |

ゲームはキャラクターを作成するとき、そのキャラクターのステーションに状態を尋ねる。そのため 0x04 のリクエストは、キャラクターが存在することを意味する（55 件のリクエストがあったが、キャラクターが描画されなかった 265 回の参加ではゼロ件）。`bin/bdsp_connect.py` はこれを判定として表示する。本体は、クライアントがリライアブルウィンドウを確認応答するまで 0x23 のリクエストを繰り返す。確認応答がない場合、75 秒間に 487 件のリクエストが届いた。

`UnionRoomManager$$SetNetData`（[1.3.0 main 0x01e50700]）は、6 種類の ID に対する `NetRequestData` に応答し、それ以外は無視する：

| 要求 | 本体が送るもの | 送信元 |
|---|---|---|
| 0x01 `NetJoinData` | 参加レコード | `UnionRoomManager$$SendJoinData` |
| 0x04 `NetCharacterStateData` | キャラクターの状態 | `UnionRoomManager$$SendOpcStateData` |
| 0x09 `NetDataBattleTypeData` | バトルルール | `UnionRoomManager$$SendBattleRuleData` |
| 0x13 `NetTradePokeData` | 提供しているポケモン | `UnionRoomManager$$SendPokeData` |
| 0x22 `NetDataStandbyWaitListData` | 待機リスト | `UnionRoomManager$$SendStandbyPlayerData` |
| 0x23 `NetDataIsMatchWaitData` | マッチング待機中かどうか | `UnionRoomManager$$SendIsMatchWait` |

部屋にキャラクターがいるステーションからは、5 種類が信頼性ストリームで応答される。応答は要求から 30 ～ 130 ms 後に計測された。ポケモンが選択されていない場合は 0x13 に応答しない（`SendPokeData` が戻る）。ベースゲームのハンドラ（[base main 0x01fd4600]）も同じ 6 種類に応答し、0x22 は `NetDataTradeStandbyData` という名前になっている。リクエストは、その応答に使われるストリームで届く。0x23 の 4333 件はすべて信頼性ストリーム、0x04 の 55 件はすべて非信頼性ストリームで届いた。

### 圧縮

リライアブルヘッダのフラグ 0x10 は zlib ストリームであることを示す。送信側はサイズで判断する。`ReliableSlidingWindow` の push（[1.3.0 main `0x15a10d0`]）はウィンドウの圧縮スイッチ（+0x7ac、`0x15a1364`）を調べ、スイッチが ON なら `0x159a500` がゲームメッセージ全体を、3 バイトのヘッダを含めて deflate する：

    0x1685384   deflateInit2: level 5, window bits 12, memLevel 5, strategy 0, 4000-byte output
    0x16854e0   deflate(Z_SYNC_FLUSH)
    0x1685608   deflate(Z_FINISH)
    0x159a5b0   compressed size >= raw size  ->  error 0x2c7d, the message goes raw
    0x15a1398   otherwise: header |= 0x10, send the compressed bytes

フラグが立っていた本体のメッセージ 55 件は、すべて Python の `zlib.compressobj(5, zlib.DEFLATED, 12, 5)` で sync flush の後に finish したものとバイト単位で一致した。フラグのない 10955 件は、この圧縮を掛けても縮まなかった。参加メッセージと 1 件だけのレコードを持つ 0x22 リストは生のまま送られる。全ゼロの 0x22 リスト（生のまま読むと `NetBonusStart` のように見える）、レコード（0x14）、ボールカプセル（0x15）は縮む。受信側が圧縮する必要はない。`bin/bdsp_connect.py` はフラグを見て展開する。

### その他の不透明なメッセージの送信元

1.3.0 にある各 `ANetData<T>.SendReliableData` の呼び出し元から判明したもの：

| id | クラス | 送信元 |
|---|---|---|
| 0x14 | `NetDataRecodeData` | `RecodeMatching$$SendRecodeData`、`UnionStateController$$SendRecodeData` |
| 0x15 | `NetDataAttachSealNetData` | `BallDecoMatching$$SendBallDecoData` |
| 0x18 | `NetSecretBaseData` | `UgNetworkManager$$SendMySecretBaseData`、リクエスト時（`OnReceiveRequestData`） |
| 0x61 | `NetDigTableData` | リクエスト時（`OnReceiveRequestData`） |
| 0x38 | `NetDataBattleMatchingSelectPokemon` | `BattleMatchingManager$$SendSelectPokemonData` |
| 0x42 | `NetPlayerNameData` | `UgNetworkManager$$SendOnJoinNewPlayer`、`SendPlayerNameData` |
| 0x54 | `NetSecretBaseUpdate` | 信頼性ストリームの送信元はない。`netdata.py` が名前を付けている |

### 非信頼性ストリーム

ここでは 3 種類のメッセージが流れる：

| バイト列 | 回数 | メッセージ |
|---|---|---|
| `04 0002 00 00` | 853 | `NetCharacterStateData{state: NONE, isRecruiment: 0}`、2 秒ごとに計測した |
| `12 0001 04` | 55 | `NetRequestData`：「あなたの `NetCharacterStateData` を送って」 |
| `02 0048 <72 B>` | 60 | `NetPosData`、12 点。本体のアバターが歩いている間 |

`room.build_state()` は 1 つ目を、`room.build_match_wait(False)` は本体自身の 0x23 への応答を組み立てる。

本体は、確認応答がそのメッセージを覆うまでリライアブルメッセージを再送する。再送は毎秒 5 回の頻度で計測された。最後のシーケンスから 2 つ先の確認応答は無視される。`bin/bdsp_connect.py` は、本体がクライアント自身のデータを確認応答した後、最後のシーケンスに 1 を足した値で確認応答を返す。

## ゲームが動作するメッセージの送信

リライアブルのシーケンス ID は、本体自身の送信と共有されている。確認応答が示す ID より小さいメッセージは、何も通知されず捨てられる。送信のたびに、その直前の ID を読み取ること。

確認応答されたシーケンスで送られた参加は、動作の対象になる。0.4 秒間隔で送った 15 回の参加のうち、本体は 12 ～ 15 回について 0x04 を要求した。最初の要求は、最初の参加から 0.10 ～ 0.57 秒後だった。
`--room-pattern fixed` は最初の要求で停止し、参加ごとに `--join-wait` 秒（既定値 1.0）を与える。

`UnionOpcManager` は参加ごとに `CreateCharacter(joinData)` を呼ぶ。40 回の参加は 40 回の到着である。製品版の本体では、セッションが裏にないまま作られたキャラクターは、ゲームを再起動するまでマップをまたいでプレイヤーについてきた。移動されたキャラクターは、そのステーションがメッシュから離れると削除された。退出時の削除経路は追跡していない（`OpcManager$$RemoveCharacter` [0x02279ee4] はステーションのインデックスを受け取る）。

## キャラクターレコード

`OpcManager.CharaData` は `{int stationIndex, string assetName, int colorId, int avatarId, int sexId, int cassetVersion}` で、ステーションごとに管理される（`RemoveCharacter(int stationIndex)`）。

### モデル

`OpcManager.CreateCharaData(ANetData<JoinData>)` は参加メッセージからレコードを作り、`avatarId` で表示される人物が決まる：

    NetJoinData.avatarId  ->  CreateCharaData  ->  CharaData.avatarId
                          ->  UnionCharacterTable.SheetSheet1{ID, AssetName}
                          ->  OpLoadCharacter("persons/field/" + assetName)

`CreateCharaData`（[1.3.0 main 0x022796b0]）は `sexId` を avatar だけから決める。条件は `avatarId > 7 && avatarId != 16`（`0x02279770`）で、8 ～ 15 は女の子、0 ～ 7 と 16 は男の子になる。`GetOpcSexId`（[0x02279840]）はその値を返し、`GetNpcColorId` は同じ avatar を読む。8 は女の子、0 は青い帽子をかぶった男の子になる（`--join-avatar N`）。

本体自身の参加は、ワーク 360 を持つ（`SendJoinData`（[0x01e4f630]）から `GetAvaterId`（[0x01e54030]）まで）。新しいセーブは 0 から始まり、セーブを作るときにユニオンルームの外見は選ばれない。ネイティブコードはワークを読むだけで（この 2 つのメソッド）、書き込むのは 2 つのスクリプトである。任意の外見カウンタ（`ev_union_character_select_*`）は、セーブの性別（`_GET_MY_SEX`）に応じた 8 種類のトレーナークラスを提示し、男の子なら 16 または 1 ～ 7、女の子なら 8 ～ 15 を格納する。それ以外では、ワーク 360 が 0 のとき、`ev_connect_union_check_save` と `ev_connect_global_check_net` が `ev_traner_other_see_random` を呼ぶ。これは 1 に 16 未満の乱数（`_GET_RND`、`_ADD_WK`）を足した値を、性別を問わず設定する。参加メッセージが示す性別はカウンタを経た後のセーブの性別であり、プレイヤーがカウンタを使ったかどうかは、その中からは分からない。

`NetDataTranerCardData`（0x05、75 バイト。`fashionId`、`bodyType`、`genderid` など）は `UnionOpcManager.CreateTranerCard()` に渡される。受信しても、ユニオンルームの画面には何も描画されない。

### 状態バイト

`StateData` は `{byte state, byte isRecruiment}` で、`state` は `OpcState.OnlineState` である：

| 値 | 名前 | 値 | 名前 |
|---|---|---|---|
| 0 | `NONE` | 5 | `RECRUITMENT_RECORD` |
| 1 | `DIG_FOSILL` | 6 | `RECRUITMENT_GREETINGS` |
| 2 | `SECRETBASE_ACTION` | 7 | `RECRUITMENT_BALL_DECORATION` |
| 3 | `RECRUITMENT_BATTLE` | 8 | `COMMUNICATE` |
| 4 | `RECRUITMENT_TRADE` | | |

17 ～ 21 は `NOW_*` の状態で、活動ごとに 1 つある（[transitionType の表](#talkstate-and-the-value-that-crashes-the-game)）。
`OpcController.ShowEmoticon(OnlineState)` と `GetEmoticonType(state)` がこれを読む。`RECRUITMENT_*` の値は吹き出しを表示させる。本体のリクエストに `StateData{RECRUITMENT_TRADE, 1}` で応答すると、製品版の画面に交換の吹き出しが出る。`StateData{NONE, 0}` では何も変わらない。

受信した 0x04 は、`UnionRoomManager$$OnReceiveData`（[1.3.0 main 0x01e506c0]）→ `OpcManager$$SetNetData`（[0x02279450]、ステーションにキャラクターがなければ捨てられる）→ `UnionOpcController$$SetNetData`（[0x01e48c50]、唯一の利用先）の順に渡る：

    if state != GetOpcOnlineState():      SetOpcOnlineState(state)                 0x02277be0
                                          isRecruiment == 1 ? SetEmoticonHost()    0x022779b0
                                                            : SetEmoticonNormal()  0x02277a50
    if state != 0:                        AnimationPlayer.Play(animation 0)

地下版の対応物は `UgOpcController$$SetNetData`（[0x01f7f820]）である。

`OpcState` は `{OnlineState _curretOnlineState +0x18; Action<OnlineState> _OnChangeState +0x20}` で、`OnlinePlayerCharacter$$Start`（[0x02276ff0]）が追加する。その Action は `ShowEmoticon`（[0x02277094]）に結び付けられ、状態は 0 から始まる。`SetOpcOnlineState`（[0x02277c80]）が唯一の有効な格納先である（`OpcState$$OnChangeOnlineState`（[0x02277d10]）を呼ぶものはない）。このセッターの呼び出し元（仮想、クラスオフセット 0x190）のうち、リモートキャラクターに作用するのは 3 つである。受信した状態を渡す `UnionOpcController$$SetNetData`（[0x01e48d7c]）と、0 を渡す `OpcManager$$RemoveCharacter`（[0x02279ee4]）および `UgOpcManager$$RemoveAllCharacter`（[0x01f80cac]）。つまり、リモートキャラクターの状態が変わるのは、そのステーションからの 0x04 だけである。残りは本体自身の状態を設定する。`UnionSystemController$$ChangeOpcState`（[0x01c2d3f8]、その引数の後に 0x04 のブロードキャスト [0x01c2d4d0]）、`RusultGreetJoinYesNoWindow`（[0x01c2e8ac]）が 6、`<CreateContextBattleTypeMenu>b__0`（[0x01f8b790]）が 3、`UnionTradeManager$$InitPlayerState`（[0x01c33a58]）、`UnionStateController$$InitPlayerState`（[0x01e4e83c]）と `SwitchCancelEnd`（[0x01e4fd70]）が 0、`UnionStateTransitionController$$StartFadeOut`（[0x01e5a00c]）はそのモデルの +0x2c、`UgNetworkManager$$OnReceiveJoinDigPermission`（[0x01f7ca94]）と `SendOnPlayDigFossil`（[0x01f79f78]）が 14、`UgNetworkManager$$SetMyEmoticon`（[0x01f79148]）である。

`Start` が実行されるまで、ゲッターは 0 を返し（[0x02277dcc]）、セッターは値を捨てる（[0x02277c50]）。キャラクターの 0x04 リクエストは、`CreateCharacter` のロードコールバック（[0x01e49854]）で `SetActive(true)` の直後に送られる。そのため `Start` より前に届いた応答は失われる。本体と同じく、状態は 2 秒ごとに送り直すこと。

`OnlinePlayerCharacter$$IsCanTalkState`（[0x02277af0]）：プレイヤーは状態 1、3 ～ 8、17 ～ 21 に話しかけられるが、0、2、9 ～ 16 には決して話しかけられない。

### 歩行

本体自身の `NetPosData` は 0.410 秒ごとに届き、0.935 単位の範囲を覆う（`room.POS_PERIOD`、`room.POS_STRIDE`。`--room-walk-period`、`--room-walk-stride`）。0.35 秒ごとに 0.1 単位ずつ歩くと、カクつくように見える。

`PosData` は `{ushort posX, ushort posZ, short rotY}` で、`pos = (-posX * 0.05, posZ * 0.05)` である。1 単位の 20 分の 1 で、x は符号を反転する。リモートキャラクターは壁と衝突し（プレイヤーはリモートキャラクターと衝突しない）、`rot_y` をそのまま保つ。歩行は部屋の中に収めること。本体のストライドで 60 件のメッセージを送ると部屋を横切り、奥の壁から出てしまう。`--room-walk-steps 8` なら中に収まる。

交換には歩行は必要ない。製品版の本体は、`NetPosData` を送らないクライアントのキャラクターとの交換を 2 回完了した（`--room-walk-steps 0`。交換経路の既定値）。

## 話しかけられる

エモートを出しているプレイヤーは、誰かが操作するまでその場にとどまる。プレイヤーの状態が 0 でない間は、プレイヤー自身の A ボタンは何もしない（[本体が近づく](#the-console-approaching)）。本体はエモートをブロードキャストする：

    NetCharacterStateData{state: 4, isRecruiment: 1}     the trade emote, up
    NetCharacterStateData{state: 0, isRecruiment: 0}     and down again

`isRecruiment` で制御すること。状態 18 は、すでに交換中の本体であり、近づくことは拒否される。製品版の本体は、交換エモートの 0.0 秒後に送られた接近に 40 ms 後に応答した。これは 3.0 秒後に送られた接近に対する応答と同じである。応答は `IsCanTalk 0, IsRecruitment 1, emoticonStateType 4` である（`bin/bdsp_host.py --approach-delay`、既定値 0）。

接近は `NetDataTalkReserveData`（0x63、`63 00 01 00`）で、本体が送るのと同じ形で行う（`bin/bdsp_connect.py --initiate-talk`）。本体のプレイヤーがクライアントのキャラクターに近づくときの順序は次のとおり：

    cli ->  64 0003 00 01 04      NetDataTalkReserveResultData{IsCanTalk, IsRecruitment, emoticonStateType}
    con ->  06 0005 00 01000000   NetDataTalkData{talkOpcSexId: 0, talkState: GREETING}
    con ->  10 0002 01 04         NetDataTalkCancelEndData{IsRecruitment: 1, emoticonStateType: 4}

会話は 0x64 を待つ。応答がなければ、プレイヤーのキャラクターは固まったままになる。

| `IsCanTalk` | その後に送るもの | 画面 |
|---|---|---|
| 0 | 何も送らない | あいさつが進み、「one second!」で止まる |
| 1 | 何も送らない、または `NetDataSelectData{0}` か `{1}` | 「sorry, I have other plans」と表示され、会話が閉じる |

ステーションが切断すると、止まっていたあいさつは解放される。B では確実には残らなかった。

### 本体が近づく

`UnionRoomManager$$MyUpdate`（[1.3.0 main 0x01e49fb0]）が A ボタンの入力を処理するのは（[0x01e4a644]）、次のすべてを満たすときだけである。プレイヤー自身の状態が 0 であること、`UnionWork.isTalking`（static +0x85）が 0 であること、メニューやメッセージウィンドウが開いていないこと、プレイヤーがどの `EnterCollision` 円の外にもいること。10.0 単位以内で、プレイヤーの `talkDistance`（+0x38）の中にあり、状態が `IsCanTalkState` を通るキャラクターを歩いて対象にする：

| キャラクターの状態 | A ボタンの動作 |
|---|---|
| 0、2、9 ～ 16 | 何もしない |
| 8、17 ～ 21 | 歩いた後、最も近い相手に `UnionRoomManager$$StartTalk(opc, 0, 0, 0)`（[0x01e4c2d0]）を実行する。メッセージは出ない |
| 4 | `UnionSystemController$$CheckErrorMessageTrade`（[0x01c2e8d0]）。エラーがなければ最後の行と同じ |
| 7 | `UnionSystemController$$CheckBallDeco`（[0x01c2ec20]）。通ったら最後の行と同じ |
| 1、3、5、6 | キャラクターのステーションへ `NetDataTalkReserveData`（0x63）を送る（[0x01e4ac94]）。状態は `nowTalkReserveState`（+0x188）に格納され、`isTalking` を設定し、`UnionStateController$$CreateSelectStateModel(state, 1)`（[0x01e4b620]）を呼ぶ |

状態 1 と 3 ～ 7 の場合、歩行は、それまでに見たものより近く条件を満たす最初のキャラクターに対して、すぐに作用する。

本体が 0x63 に応答するのは、`UnionRoomManager$$SetNetData`（[0x01e50c3c]）の中である：

    r.IsCanTalk = UnionWork.isTalking
    if stateController._currentModel (+0x88) is null:
        r.IsCanTalk = 1; SendOpcStateData(requester)
    r.IsRecruitment = 1
    r.emoticonStateType = the console player's own state
    send r (0x64) to the requester                           0x01e5110c
    if isTalking was 0 and r.IsCanTalk is 0: isTalking = 1

本体が自身の 0x63 の後に 0x64 を読むとき（[0x01e50dec]）：

    canTalk = IsCanTalk == 0 and not isCancelStock (+0x183, cleared here)
    if emoticonStateType != nowTalkReserveState:
        send NetDataTalkCancelEndData{0, 0}                  0x01e50f24
    else:
        nowTalkReserveState = 0                              0x01e50ea0
        canTalk ? StartTalk(opc, IsRecruitment == 0, 1, 0)
                : the refusal path 0x01e512f0 (the character's state against 3 to 7)

`{4, 1}` を広告しているキャラクターは、A ボタンで `63 0001 00` を送り、`64 0003 00 01 04` で会話を開く。それ以外の `emoticonStateType` では `NetDataTalkCancelEndData{0, 0}` が返る。同じ 0x63 に対する 2 回目の 0x64 も同様である。最初の応答が `nowTalkReserveState` を消去しているためだ（[0x01e50e94]）。0x63 には、再送された複製ではなく、1 回だけ応答すること。

本体が話しかける側のとき、クライアントのキャラクターが募集する側になる。交換の申し出に対する募集側の「はい」（`UnionTradeContextMenu.<>c__DisplayClass10_0.<ShowTradeYesNoWindow>b__0`（[0x01c32f70]）。「いいえ」は `TradeRecruitmentStateModel$$Cancel`（0x01c24150））は、メッセージ 8 を開き、`UnionContextMenu$$SendTransitionData(station, 0)`（[0x01f86480]）に末尾呼び出しで渡る。これが `NetDataTransitionData{menu+0x50, 0}` を信頼性ストリームで送る。`SetTransitionType`（[0x01c32ef0]）は +0x50 を 18 に設定する。「はい」は `07 0002 12 00` で、クライアントが近づいたときに募集中の本体が送るメッセージである（[交換ページ](bdsp_trade.md#the-message-sequence)）。0x64 の後に届いたタイプ 18 の 0x07 は、話しかけている本体でプレイヤーの「はい」なしに交換を開始する（`SwitchTransitionMessage`（[0x01e53bf0]）→ 下記の `TransitionTradePoke`（[0x01e5c1c0]））。本体は自分の `NetDataTradeTranerData`（0x24）を送り、交換ボックスを開く。製品版の本体は、クライアントの状態 4 のキャラクターに近づいたとき、0x64 の応答の後に送られた `07 0002 12 00` を受け取り、交換を完了した。

話しかけている本体では、状態 4 のキャラクターに A ボタンを押すと、`UnionStateController+0x50` に `TradeJoinStateModel` が作られる（`CreateSelectStateModel(4, 1)`、格納先 0x01e4b814。唯一の格納先で、消去されることはない）。0x64 は `SwitchTalkStateMine`（[0x01e4c41c]）を実行する。これは `UnionSystemController.onlinePlayerSelectState`（+0x28）に 18 を入れ、`SetTargetStationIndex(station, cassetVersion, 1)`（[0x01e54ae8]）を呼ぶ。0x07 は状態の検査なしで（[0x01e52610]）`SwitchTransitionMessage`（[0x01e53bf0]）に渡される。これはタイプのモデル（バイト表 0x03db869b）を null の検査なしで読む。交換の場合は、`TradeJoinStateModel$$OpenSwitchFadeMsg`（[0x01c22f50]）が、送信者の性別に応じてメッセージ 9 または 10 を開く。話し手は `OPPONENT`（`SpeakerID` 1）で、`StartFadeOut` に入って閉じる。フェードの後、`SwitchTransition`（[0x01e5ba70]）が 18 を `TransitionTradePoke`（[0x01e5c1c0]）に送る。これは対象のステーションを取り出して消去し、`IsGamerActive` を確かめ、本体の 0x24 を送る（[`SendTranerData` 0x01e5c2b0]）。参加者の方向で交換が始まる。`UnionStateController` を作ってから一度も交換募集者に A ボタンを押していない本体に 0x07 が届くと、null を読んでしまう（0x01e53c4c）。他の遷移タイプも、それぞれのモデルで同じ形をとる。

### あいさつの中の名前

あいさつ、話し手のラベル、バトルラダーの「選んでいます」の行は、キャラクターをそのステーションの Pia プレイヤー名で指す。`UnionBaseMsgWindow$$SetTargetDataMessage`（[1.3.0 main 0x01f86810]）と `GetSpeakerName`（[0x01f87050]）は `NetworkManager$$GetGamerData(station)`（[0x02250e50]）を読む（`IlcaNetGamer.gamerName` +0x30、`nameStringLanguage` +0x38）。`NetGamerNameGet`（[0x0273a9a0]）は Pia の参加イベントで、`INLpiaSessionGetPlayerInfo`（[0x01e15cd0]）から受け取る 80 バイトのバッファのうち `length - 1` バイトを UTF-8 としてデコードし（長さが 2 のときは空）、言語のバイトはそのままコピーする。

`Utils$$CheckNGTrainerName`（[0x01cbdc30]）は、名前が空のとき、`CheckNgWords`（[0x01c99220]）が指摘したとき、または UTF-16 の単位数が `SoftwareKeyboard$$LanguageMaxLength(6, lang)`（[0x01c995b0]）より多いときに、名前を `Utils$$GetReplacedNGName(UnionWork.nowTargetCassetVersion)`（[0x01cbde20]）の結果で置き換える：

| `lang` | 上限 |
|---|---|
| 1, 8, 9, 10 | 6 |
| それ以外の正の値 | 12 |
| 0 以下 | UI の現在の言語で決まる |

言語は PlayerInfo のバイト 0x7A である。Mesh Station Protocol のパーサ（[0x0154f044..0x0154f5e0]）は、1 回の反復で 195 バイトの PlayerInfo を 1 つ読む（`add w23, w23, #0xc3` 0x0154f588）。オフセットは、エンコーディングにかかわらず固定である：

| offset | フィールド | 読み取り |
|---|---|---|
| 0x00 | 名前のエンコーディング | 2 のとき UTF-16 の 20 単位（0x0154f078..0x0154f15c）。それ以外は 80 バイト |
| 0x01..0x50 | 名前 | ステーションのレコードの +0x120 + index*0x80 へ |
| 0x51 | 2 番目の文字列のエンコーディング | 0x0154f220 |
| 0x52..0x79 | 2 番目の文字列、40 バイト | エンコーディング 2 のとき UTF-16 の 10 単位、+0x320 + index*0x58 へ |
| 0x7A | 言語 | `strb w8, [x20+x25, #0x480]`（0x0154f5a0） |
| 0x7B..0xBA | 64 バイト | 0x0154f5b8 |
| 0xBB..0xC2 | u64 | +0x490 + index*8 へ |

`INLpiaSessionGetPlayerInfo` は `0x01391a90` に到達する。ここでバイトを 0xFF に既定し（[0x01391b44]）、ステーションのレコードの +0x480 をそのまま `NetGamerNameGet` の `nameStringLanguage` にコピーする（[0x01391bb0]）。このバイトは `MsgLangId` である：

    JPN 1, USA 2, FRA 3, ITA 4, DEU 5, ESP 7, KOR 8, SCH 9, TCH 10

本体は、自身の `MessageManager$$get_UserLanguageID` と、`CheckNGTrainerName` で検査された名前を渡す（`SessionConnector$$ResetParam`（[0x0202f050]））。これは `IlcaNetBase$$PlatformInitialize2`（[0x01e13ce0]）と `PiaPlugin$$RegisterStartupSessionSetting`（[0x0227d5a0]）を経由し、ステーションプロトコルの PlayerInfo ライタが +0x480 をバイト 0x7A に書く（[0x01550e98]）。フランス語の本体の接続応答（ステーションプロトコルの種別 2）は、エンコーディング 1、名前、バイト 0x79 の 0、バイト 0x7A の 3 を持つ。

言語は送り手のゲーム内テキスト言語である。保存された `CONFIG.msg_lang_id`（PlayerWork +0xac、`get_msgLangID`（[0x0237e100]））から取る。`GameManager.<OnetimeInitializeOperation>`（[0x01e0eb44]）は、保存値が 1 ～ 10 の範囲外のときだけシステム言語（`GetCurrentIetfCode`）から埋める。自身のステーションレコードの +0x480 は、PlayerInfo の言語バイトごとに 4 つのコピーで埋められる：

| 段階 | コード | 元 | 先 |
|---|---|---|---|
| `RegisterStartupSessionSetting` | `0x0156f918`（設定のビルダ、`0x0156fa08`）、`0x0168d4c8` | プラグインのプレイヤー配列（ストライド 0x28） | フレームワークが保持する設定 +0x3220：PlayerInfo +0x18、言語 +0x98、ストライド 0x98 |
| `ChangeStateJob::StartupSession` | `0x01690d6c` -> `0x0168c5b8` -> `0x0157d36c` -> コントローラの vfunc 2（`LocalMatchMeshLayerController` `0x016c3e70`） | 設定の最初の PlayerInfo（`0x016c3f48`） | コントローラのプレイヤーリスト +8、言語 +0x88、+0x268 のカウント 1 |
| `CreateSessionJob::MeshStartup` `0x015873b0`、`JoinSessionJob::MeshStartup` `0x015884bc` | コントローラの vfunc 4（`MeshLayerController` `0x0171a068`）-> `0x01546fa4` | コントローラのリスト | メッシュオブジェクト（`[0x04c4db60]`）の +0x128 + n*0x98、言語 +0x80（`0x015472e4`）、カウント +0x388 |
| `CreateMeshJob::SetupLocalPlayerInfo` `0x0155a764`、`JoinMeshJob::SetupLocalPlayerInfo` `0x0155b988` | `0x015494f0` | メッシュオブジェクトのリスト | 自身のステーションレコードの +0x480 + n（`strb w8, [x23, x22]` `0x0154956c`） |

`ChangeStateJob::StartupSessionBegin`（`0x01690c70`）は、直接 StartupSession に至るか、フレームワークの +0x7c が 0 のときは `StartupSessionJob`（`0x0168ec7c`。ジョブの +0x70 に設定）と `WaitStartupSessionLdnInitialize`（`0x01690e30`）を経て至る。`StartupSessionJob` の作業ステップは 1 つだけで、InitializeLdn（`0x0168ed74`）であり、PlayerInfo はコピーしない。フランス語の本体は、30 件すべての PlayerInfo（接続応答 17 件、接続要求 13 件）で、バイト 0x7A に 3、バイト 0x51 に 0 を送った。

`bin/bdsp_connect.py` と `bin/bdsp_host.py` は、どちらも `pokeldn/bdsp/host.py` の `player_info` で PlayerInfo を作る（エンコーディング 1、UTF-8 の名前、バイト 0x51 は 0）。`--language` を渡し、既定値は 3 である。デスクトップアプリはトレーナーの言語を渡す。言語 3 のとき、11 文字の名前は省略されずに表示される。

代わりの名前は、話しかけられたキャラクターの `CharaData.cassetVersion` に依る。これは `NetJoinData` のバイト 2 で、あいさつの 2 つの呼び出し元（`SwitchSpokenStateMine` 0x01e53b58、`MessageEndSpokenData` 0x01e57ab8）が `UnionWork.nowTargetCassetVersion`（static +0x8C）に格納する。
`GetReplacedNGName` は、0x31 のとき `dp_characters` のラベル 247、それ以外ではラベル 206 を取り（0x01cbdecc）、末尾にピリオドを付ける。文言は 1.3.0 の RomFS にある `Message/<language>` のバンドルから取った：

| バンドル | 0x31（シャイニングパール）、`DP_CHARACTERS_247` | それ以外、`DP_CHARACTERS_206` |
|---|---|---|
| english | `Pearl.` | `Diamond.` |
| french | `Perlo.` | `Diamant.` |
| german | `Perl.` | `Diamant.` |
| italian | `Perl.` | `Diaman.` |
| spanish | `Perla.` | `Diamant.` |
| jpn, jpn_kanji | `パール.` | `ダイヤ.` |
| korean | `펄.` | `다이아몬드.` |
| simp_chinese | `帕尔.` | `戴亚.` |
| trad_chinese | `帕爾.` | `戴亞.` |

### talkState と、ゲームをクラッシュさせる値

`TalkState` は `{CHECK = 0, GREETING = 1, NONE = 2}` で、止まっている本体は `GREETING` で待つ。`UnionStateController$$SwitchSpokenStateMine` は、CHECK 以外の値を `StartOpenGreetingMsgWindow` に渡す（0x1fd5e80 の `b 0x1fd85e0`）。CHECK は `systemController->msgWindow` を読み込み（0x1fd5e84）、null のガードなしで参照する（0x1fd5ec0）。エモートを出して立っているプレイヤーはメッセージウィンドウを持たないので、CHECK はゲームをクラッシュさせる。`pokeldn/bdsp/room.py` はこの値を送らないように拒否する。送り手から読み取った状態の値は、受け手のハンドラを読むまで送ってはならない。

`NetDataTransitionData{transitionType, isRecruitment}`（0x07）は、止まっているあいさつを活動に進める。ゲームはこれを末尾呼び出しで送るため、BL 命令だけを対象にした呼び出し元の走査では、送信されないと報告されてしまう（[呼び出し元の調べ方](switch_re.md#finding-callers)）。`transitionType` は `OnlineState` である。`UnionStateTransitionController$$SwitchTransition`（[1.3.0 main 0x01e5ba70]）は、19 エントリの表でこれを振り分ける。`SwitchTransitionMessage` は、下のオフセットで `UnionStateController` のモデルを読む：

| transitionType | 活動 | 入るまでの経路 | モデル |
|---|---|---|---|
| 3、17 | バトル | `TransitionBattle` | `isRecruitment` により +0x38 または +0x40（`tbz w4`） |
| 4、18 | 交換 | `TransitionTradePoke` | +0x50、`tradeJoinStateModel` |
| 5、19 | レコードミックス | `RecodeMatching$$Open` | +0x60 |
| 6、20 | トレーナーカード | `TransitionShowTrainerCard` | +0x70 |
| 7、21 | ボールカプセル | `BallDecoMatching$$Open` | +0x80 |
| 8 ～ 16 | なし | | なし。そのまま戻る |

`RecodeMatching$$Open` と `BallDecoMatching$$Open` は、本体自身のレコード（0x14、0x15）をすぐに送って待つ。セーブに書き込むのは、相手側の `StartRecodeTradeFlow` と `StartBallDecoTradeFlow` だけである。

### レコードミックス

本体は、Y メニューの「Échanger des données」で募集する（状態バイト 5）。クライアントが近づき、プレイヤーが「はい」と答えると、`NetDataTransitionData{5, 0}` を送り、続けて 694 バイトの `NetDataRecodeData`（205 バイトの zlib ストリーム。計測では 1.5 秒後）を送る。そして状態 19（`NOW_RECORD`）で待つ。応答がないと「un des participants n'est plus disponible」と表示されて部屋に戻る。計測された待機時間は 1 回だけで 44 秒だった。タイマーの場所は特定していない。`room.parse` はレコードのヘッダを `recode` として返す：

    RECORD.record[30]        thirty uint counters indexed by RECORD_ID (CLEAR_TIME, DENDOU_CNT,
                             CAPTURE_POKE, ..., CONTEST_RATE_SINGLE)
    RANDOM_SEED              group_name (16 chars), name (32 chars), int sex, int region_code,
                             ulong seed, ulong random, long time_stmp (a Windows FILETIME),
                             int user_id (the trainer id)
    TvRecodeData             five TV records (personality, ball decoration, fossil digging,
                             statue, fashion), each `bool isEmpty` (4 bytes), ints, a TV_STR_DATA
    4 x TV_STR_DATA          {16-char value, byte language, genderId, two reserved}
    RECORD_HEAD              13-char username, int language, byte sex, int body_type,
                             uint uniqueID (the trainer id again)
    ten ints, six bytes      the per-TV branch values, myVersion (0x31), five *IsNotEmpty flags

書き込まれていないフィールドには、64 ビットのヒープポインタが入っている。1 つのレコードの中で `RECORD_HEAD.sex` と `RANDOM_SEED.sex` が食い違うことがある（キャプチャしたものでは 0 と 1）。`RECORD`、`RANDOM_SEED`、`RECORD_HEAD`（名前空間 `DPData`）はパック 4 でマーシャルされ、`TvRecode*` 構造体はパック 8 でマーシャルされる。このフィールドサイズではパディングは生じない。

### バトルラダー

バトル（「Combattre」、募集中の状態バイト 3）は、ラダーを順に進む。募集中の本体は、各段で参加者（ここではクライアント）を待つ：

| 参加者が送るもの | 本体の動作 |
|---|---|
| `NetDataTalkData{GREETING}` | 「Un combat ? OK ! Donne-moi juste une minute !」と表示して待つ |
| `NetDataSelectData{0}`（0x08） | 「POKELDN est en train de choisir quoi faire...」と表示して待つ |
| `NetDataBattleTypeData{0}`（0x09、`BattleModeID.Single`） | プレイヤーに「voulez-vous faire un combat selon ces règles ?」と尋ねる。「はい」なら `NetDataTransitionData{17, 0}` を送り（状態バイト 17）、「connexion en cours」でソロのロビーを開く |
| `NetDataBattleMatchingJoin`（0x30）`{uint id, byte stationIndex, index, language, colorId, avatarId, sexId, cassetVersion}` | 自身の参加で応答する（`id` はそのトレーナー ID、ステーション 0、index 0）。参加者の参加も中継する。参加者のキャラクターは、ロビーの 2 番目のスロットに現れる |
| `NetDataBattleMatchingReady`（0x32、空） | `BattleMatchingManager$$ReceiveReadyData`。全メンバーが準備完了すると、`NetDataBattleMatchingState{0, 6}`（0x33）、`MatchingState.SelectBattleTeam`（ソロバトルでは 4 と 5 を飛ばす）が送られ、プレイヤーにチーム選択ボタンが付く |
| 何もしない | プレイヤーがチームを選ぶと、`NetDataBattleMatchingSelectPokemon`（0x38、481 バイト）を 6 件送り、「en attente d'autres personnes」と表示する |

各 0x38 は、チェックサムが一致する暗号化された PB8 を持つ（選んだチームを順番どおりに）。同一のヒープ残りかすを持つ 20 個の `SealParam` スロットがあり、`affixSealCount` は 0、`attachPokemonId` と `attachPersonalRnd` は 0、`index` は 0 ～ 5、`num` は 6 である。本体は `id` が 0x0badc0de の 0x30 を受理した。そのフィールドの検査は見つかっていない。`BattleMatchingManager.MatchingState` は、None 0、Initialize 1、Load 2、RecruitmentMember 3、SelectTeamMember 4、SelectRule 5、SelectBattleTeam 6、SelectPokemon 7、GoBattle 8、Result 9、Resume 10、Closing 11、LeavedOtherMembers 12 である。

`NetDataSelectData`（0x08）は `{byte index}` である。その受信側は 1 つだけ（[`UnionRoomManager$$SetNetData`、0x01e52a30]）で、送り手のステーションを読み、インデックスは決して読まない：

    m = stateController.battleRecruitmentModel                  UnionStateController +0x38
    m.ChangeBattleRecruitmentState(BATTLE_RULE_SELECT_WAIT 4)   0x01d2aab0
    m.currentCancelModel = {SelectCancel 0, station}
    m.CloseWindow()
    if m.unionMsgBattleWindow != null:
        SetTargetDataMessage(window, station, 1, 1); OpenMsgWindow(window, 3, 2)   0x01f86fa0

開いている会話が何であっても、0x08 はバトルの募集モデルを動かす。両方の送り手は 0 を書き込む（`UnionBattleContextMenu$$SendRuleSelectState`（[0x01f87c60]。末尾呼び出し）と `<ShowBattleJoinYesNoWindow>b__0`（[0x01f8800c]））。

受信側はモデルの null 検査をしない。+0x38 を読み込み（[0x01e52a78]）、case 4 がそれを `NetStateModel$$SetState`（[0x023e2604]）で書き込む。+0x38 への唯一の格納は `CreateSelectStateModel`（[0x01e4bb08]）である。これは、プレイヤーがバトルを募集したとき、状態 3 か 17 の `BattleRecruitmentStateModel` を作る（`stateModelType` は 0。A ボタンは 1 を渡して `BattleJoinStateModel` を作る）。`UnionStateController` は `UnionRoomManager` ごとに 1 回作られ（`UnionRoomManager$$SetUp`、`.ctor` 0x01e4d01c）、通信バトルでは両方が保たれる（`EvDataManager$$UpdateStart` -> `UnionRoomManager$$ReturnBattle`（[0x01b02a78]）。コンストラクタは呼ばれない）。

各エントリは新しい `UnionRoomManager` を作る。`EvDataManager$$EvCmdUnionProc`（[0x01b35f70]）は、部屋へワープする前にそれを `new GameObject("UnionRoomManager")`（[0x01b36090]）に追加する。`DontDestroyOnLoad` は付かない。`UnionRoomManager$$Init`（[0x01e49e40]）は、ゾーン {484, 491, 492, 493}（`UNION`、`UNION01` ～ `UNION03`）を `NetUseManager.SetEnableZone`（[0x026cfca0]）に渡す。これは `FieldManager` のゾーン変更イベントを購読する。`NetUseManager.OnZoneChange`（[0x026cfef0]）は、リストの外にある最初のゾーンに入ったとき `Object.Destroy(gameObject)`（[0x026d00f0]）を呼ぶ。退出（`LeaveUnion`（[0x01e4e300]）。そのコルーチンは遷移ゾーンを [0x01e560e0] で設定する）はこの変更に当たるため、`UnionRoomManager$$OnDestroy`（[0x01e4c540]）が走って `Clear` を呼ぶ。そのため募集モデルは、訪問のたびに null から始まる。その訪問でプレイヤーがバトルを募集していない本体に 0x08 が届くと、null に書き込んでしまう。受信側は null のモデルを、null の検査なしに `BattleRecruitmentStateModel$$ChangeBattleRecruitmentState`（[0x01d2aab0]）へ渡す。その case 4 はそれを使って `NetStateModel$$SetState`（[0x023e25f0]）を末尾呼び出しし、`str x2, [x20, #0x18]!`（[0x023e2604]）がアドレス 0x18 に格納する。このパスには IL2CPP の null チェックは生成されず、イメージは `nn::os::SetUserExceptionHandler` を import していない。

ラダーの 0x08 の行は、バトルを募集していた本体で計測された。クライアントがすでに使ったシーケンス ID の下の 0x08 は、リライアブルウィンドウが捨てる（[Pia のページ](pia.md#what-the-receiver-discards-in-silence)）。募集していない、話しかけている本体に送った 22 件は、クライアント自身の 0x64 応答がすでに持っている ID の下で送られたため、どれも null のパスには届かなかった。

本体自身の 0x04 が状態 3 かつ `isRecruiment` 1 を示すのでなければ、0x08 を送ってはならない。

### 地下大穴

地下大穴は、シーン ID 12608 の下で同じ `local_communication_id` を広告する。同じセッションの行に関連付けられ、参加レコードはない（`--room-walk 0`）。ステーションが到着すると `UgNetworkManager` は次を送る：

| id | クラス | バイト数 | 内容 |
|---|---|---|---|
| 0x17 | `NetZoneData` | 16 | `Vector3 pos`、`int zoneID`（キャプチャでは 519） |
| 0x42 | `NetPlayerNameData` | 28 | UTF-16 の 13 文字、byte genderid、byte languageId（3、フランス語） |
| 0x50 | `NetKousekiCount` | 4 | `int Value` |
| 0x41 | `NetSecretBaseInfo` | 16 | 秘密基地（ゾーン 633）の中では 0x17 の代わりに使う：`Vector3 pos`、`int zoneID`、真上の階への入口（519） |

プレイヤーのゾーンを示す `NetUgJoinData`（0x16、18 バイト：`byte avatarId, colorId; short zoneID, InitRotY; Vector3 InitPos`）は、プレイヤーの隣にキャラクターを置く。`OnReceiveJoinData`（[1.3.0 main 0x01f7bd80]）はそのキャラクターの `NetCharacterStateData` と `NetNaminoriData`（0x55、4 バイトの bool）を要求し、本体は 0.41 秒ごとに `NetPosData` を送る。位置は部屋と同じエンコーディングを使う（`--ug-join --room-walk-steps N`）。

`UgNetworkManager$$OnReceiveRequestData`（[1.3.0 main 0x01f7c050]）は、0x01、0x04、0x18、0x19（`NetDigData`）、0x54、0x55、0x61 の要求に応答する：

| 要求 | バイト数 | 内容 |
|---|---|---|
| 0x18 `NetSecretBaseData` | 616 | プレイヤー自身の `UgSecretBase`。`zoneID` が 0 のときは何も返さない |
| 0x54 `NetSecretBaseUpdate` | 616 | 同じ 616 バイト |
| 0x61 `NetDigTableData` | 8 | 掘り化石の ID 8 個、`01 06 04 02 05 03 00 07` |

本体のウィンドウが何も運んでいない時点で、シーケンス 1 で送られた要求は応答されなかった。それ以外の ID への要求には何も返らない（0x29 と 0x42 を試した）。

0x61 は `UgFieldManager.ugDigGroupList` で、0 ～ 7 を `Guid.NewGuid()` で並べ替えたものである（`UgStationID_to_DigFossilIDList$$Init`（[0x02031830]））。マーシャラ（[0x002498c0]）は要素 0 ～ 7 をコピーし、長さの接頭辞はなく、足りない場合は例外を投げる。本体は、自身の表の準備ができてから初めて 0x61 の要求に応答する（`UgNetworkManager.IsDigTableReady`、+0xA8、0x01f7c440 で検査）。

`UgNetworkManager$$OnSessionEvent`（[0x01f77510]）は `SessionEventType` で分岐する（バイト表 0x03db8970）：

| イベント | 対象 | 効果 |
|---|---|---|
| 1 `JoinIn_Mine` | 0x01f77588 | ホストの場合：フラグを立て（[0x01f7763c]）、`DeleteAllDigPoints`、`CreateDigPoints(MyStationIndex)`。ホストでない場合：全員に `NetRequestData{0x61}` を送り（[0x01f77738]）、フラグは 0 のままにする |
| 2 `JoinIn_OtherPlayer` | 0x01f776bc | `SendOnJoinNewPlayer` |
| 3 `ChangeHost_Mine` | 0x01f776d4 | 0 のときフラグを立てる（[0x01f77740]）。新しい表は作らない |
| 4、5 | 0x01f776dc | 何もしない |
| 6 `Leave_OtherPlayer` | 0x01f776ec | `OnLeaveOtherPlayer` |
| 7、8、9 | 0x01f77574 | `OnCrash` |

`UgNetworkManager$$OnReceiveDigTableData`（[0x01f7dad0]）は、フラグが 0 の間に届いた最初の 0x61 を、どのステーションからのものでも、自身の `JoinIn_Mine` の前でも後でも採用する。検査するのはフラグ（[0x01f7db0c]）とクラスだけで、配列を検査せずに `ugDigGroupList`（[0x01f7db94]）に格納し、掘りポイントを作り直して（[0x01cfd6d0、0x01cfdab0]）フラグを立てる。+0xA8 への 3 つの格納はすべて 1 を書くので、以後の 0x61 はすべて無視される。共通の振り分け（`SessionManager$$OnReceivePacket` `0x1df8ac0`）は、`INetData.FromStationIndex` を参照しない。

`CreateDigPoints` は要素 `[MyStationIndex]`（[0x01cfdc7c]）を読む（8 以上なら例外を投げる）。その `ID` を持つ `UgDigFossilePosGroup` を探し（`List.Find` 0x01cfdcd8）、`CreateDigPointModel`（[0x01cfe470]）でその `Grids` を null の検査なしに読む。1.3.0 の `ugdata` バンドルでは、35 個のゾーン（508 ～ 542）それぞれに 8 つのグループがあり（ID は 0 ～ 7）、セル数は 6 ～ 26 である。送るのは 0 ～ 7 の順列だけにすること。それより大きいバイトがあると、本体が異常終了する。

各 `UgFieldManager` は、`StartSession`（[0x01cfebb0]）で新しい `UgNetworkManager` を作り（唯一の `AddComponent<UgNetworkManager>`、0x01cfed54）、`OnDestroy`（[0x01cff530]）でそれを破棄する。そのため、各 `UgNetworkManager` は表を毎回新たに採用する。どちらのマネージャもシーンには配置されていない。1.3.0 RomFS の 14052 個のアセットバンドルとルートファイルにある 53861 個の MonoBehaviour のどれも、スクリプトとして `UnionRoomManager` や `UgNetworkManager` を持たない。両方とも `globalgamemanagers.assets` のスクリプトテーブルにのみ存在し、どのバンドルもそれを参照しない。

0x29 の `NetDigGroupIdData` は、0x61 と構造体とメソッドを共有する。これを参照するのは `NetDataParser` のコンストラクタだけ（[0x0224a420]）で、送るものはなく、`UgNetworkManager$$OnReceiveData`（[0x01f7a880]、22 個の ID）にも分岐がない。

`bin/bdsp_connect.py --inject-file PATH` は、ファイル内の新しい `ID:HEX` 行をそれぞれリライアブルに送る。

### ボールカプセル

本体は「Déco Capsule」で募集する（状態バイト 7）。`NetDataTransitionData{7, 0}` を送り、状態 21（`NOW_BALL_DECORATION`）に入り、143 バイトの `NetDataAttachSealNetData` を 116 バイトの zlib ストリームとして送る（計測では 1.8 秒後）。応答がないと「quelqu'un a mis fin à la communication」と表示されて部屋に戻る。計測された待機時間は 1 回だけで 45 秒だった。タイマーの場所は特定していない。クライアント自身の 143 バイトで応答すると（`bin/bdsp_connect.py --answer-with 0x15:FILE`）、それを適用し（`BallDecoMatching$$ReceiveBallDecoData`、下記）、状態バイト 0 で部屋に戻る。その戻りは、計測で 5 秒未満だった。応答は独立したタスクから送ること。受信側の内部から送ると、それが待っている確認応答は読まれない。

1.3.0 のイメージでは、本体が自身のデザイン（シールが付いている最初のスロット）を送った後に、`BallDecoMatching$$ReceiveBallDecoData`（`0x021ca5e0`）が実行される。99 個のカプセルスロットのうち、シールのない最初のものに書き込む（`0x021ca674`）。すべてにシールがあれば何も書き込まない。`BallDecoWork$$CopyTradeCapsuleData`（`0x01f23eb0`。ベースゲームには無い）はスロットを消去し（付いているポケモンを含む）、`Is3DEditMode` と `IsAppliedTemplate` を `byte == 1` として格納し、`affixSealCount` 個のシールを順にたどる：

    if SaveSealData[id].Count >= 1:  place it at (x, y, z) / 100, then SubSealCount(id, 1)
    else:                            drop it

プレイヤーのカプセルにあるシールは、外されるまで在庫がない扱いになる（`CapsuleInfo$$RemoveAffixSeal`（`0x01e940d0`）が `BallDecoWork$$ReturnSealCount` を呼ぶ）。在庫がなければ何も書き込まれない。スロットには、配置できたシールが詰めて格納される。少なくとも 1 つが配置でき、1 つも捨てられなかった場合に結果は true になり、終了メッセージ（`0x021c9f80`）を選ぶ。true なら `DLP_net_union_room_090`、それ以外は `_114`（「Seuls les sceaux que vous possédez ont été collés」）になる。`affixSealCount` が 20 を超える場合、またはシール ID が 200 以上の場合は、配列の範囲外を指して（`0x01f24254`、`0x0238b7b0`）例外を投げる。

位置は、カプセル半径の 100 分の 1 で表す。両モードとも、表面は大きさ 100 にある。「シールあり」は `AffixSealCount != 0`（`0x01e93630`）である。3D モードでは、各シールをその位置に描画する。2D モード（`Capsule2DViewController$$UpdateGridCells`、`0x01e90de0`）は、グリッドセルの `BallDecoWork$$Convert2DPosition`（`0x01f24da0`）の位置にだけシールを描画する。半径は 1 で、行と列の間隔は、前面（+z）では 28 度、背面では 25 度。各成分は 0.01 単位に丸める。

`Capsule2DViewController$$Initialize`（`0x01e909b0`）は、グリッドのルートの `Capsule2DGridCell` の子を取る。中央の 1 つを中心（+0x78）とし、ルートの `GridLayoutGroup` のセルサイズと間隔の和を刻みとして使う。各セルには、中心からのオフセットを刻みで割って丸めた `GridPosition` を与える（`0x01e90c48`）。`Capsule2DGridCell$$Setup`（`0x01e905e0`）は `Convert2DPosition(GridPosition, isFront)` を +0x30 に格納する。1.3.0 のバンドル `/Data/StreamingAssets/AssetAssistant/UIs/ui/uiresidentwindow` には `Capsule2DViewController` が 3 つあり、それぞれのルート `Grid` は `GridLayoutGroup`（セル 72x72、間隔 3x3、固定 7 列）で、37 個のセルと 12 個の角のスペーサーを持つ。前面と背面のグリッドは、7 x 7 の正方形から `{(3, 2), (2, 3), (3, 3)}` の `(|x|, |y|)` を除いたものである：

    row  3   columns -1..1
    row  2   columns -2..2
    row  1   columns -3..3
    row  0   columns -3..3
    row -1   columns -3..3
    row -2   columns -2..2
    row -3   columns -1..1

本体自身の 2D カプセルは、前面のセルにシールを持つ。グリッドの外にある 2D デザインは、デコ済みと印の付いたスロットを埋めるが、何も描画しない。前面のセルにあるシールは、在庫があれば描画される。

グリッドの位置は製品版の縮尺でも成り立つ。`Initialize` はワーク空間のオフセット（`Transform$$get_position`、`0x01e90bd0`）を、両軸とも 75 の刻みで割る（`fdiv` `0x01e90c10`）。ルートの `Seal` または `SealTemplate` までの祖先は、すべてローカルスケールが 1 である。ルートの `Canvas` はスクリーン空間のオーバーレイで、その `CanvasScaler` は幅について 1280 x 720 から拡縮する。`Window` のアニメータは平行移動だけを結び付ける。係数 `Screen.width / 1280` は 1 である。1.3.0 の `/Data/rawsettings` の +0x1c にある u32 は 0 で、これにより既定解像度の切り替え `0x006062e8` は、据え置き時も携帯時も 1280 x 720 に保たれる（1 は操作モードに、2 は性能モードに、3 は両方に従う）。マネージドコードは `SetResolution` も `Screen` のセッターも呼ばない。

`Screen.width` は、単一のネイティブ画面オブジェクト（`0x04efe760`）の +0x68 にある int で、vtable のスロット 0xa8（`0x002c2c24`）を通して読まれる。これに書き込むのは 3 か所である。コンストラクタ `0x002c257c`（1280 x 720）、起動時に 1 回だけ呼ばれる `SetMode(0)`（`0x002c2858`。値は `0x006062e8` のもの）、`SetResolution`（`0x002c2888`）。`SetResolution` を呼ぶのは操作モードと性能モードのハンドラ（`0x002c2a1c` と `0x002c2af0`）だけで、rawsettings の +0x1c が 0 でない場合に限られる。`globalgamemanagers` のプレイヤー設定は、この経路では読まれない。マネージドの `Screen.SetResolution` は引数を `[obj+8]` に格納するだけで、何も変えない。
