---
title: ユニオンルームでの交換
parent: Brilliant Diamond and Shining Pearl
nav_order: 3
---

# BDSP の交換

市販のシャイニングパールが、pokeldn の作ったキャラクターと交換し、pokeldn の組み立てたポケモンを受け取ってセーブデータを書き込みます。
接続の手順と挨拶は [ゲームプロトコル](bdsp_protocol.md) にあります。

## メッセージの流れ

    NetDataTransitionData{transitionType: 18}     entering the trade
    NetDataTradeTranerData        32 B            who they are
    NetTradePokeData             328 B            the Pokemon
    NetDataTradePokeCheckOkData    1 B, value 1   "yours is fine"
    NetDataTradeReadyOkData        2 B            the last message before the exchange

それぞれ、クライアント自身のメッセージのいずれかで応答します。

本体側で唯一稼働している check-ok の送信元は、`TradeSelectPokeModel.<OpenTradeBoxWindow>b__54_2`
[1.3.0 main 0x1c28520] で、`BoxWindow$$Open(otherName, msgLangId, onSelected,
onDecide, onConfirm, onComplete, onCancelSelect, ...)` の `onDecide` です (ラムダ `b__54_1` から `b__54_5` のひとつ)。自身のチェック状態 (+0x80) を 6 にし、`{isCheckOk: 1}` を
`tradeTargetIndex` (+0x48) に送って、`isWaitingOK` (+0x7b) を 1、`isWaitingSelect` (+0x7c) を 0 にします
[0x1c285a4]。`isCheckOk` は常に 1 です。本体はこのメッセージでポケモンを拒否することはありません
(`SendTradePokeCheckOk` [0x1c27cb0] を呼ぶ箇所はありません)。`BoxWindow$$OnTradeContextMenu` は
`onDecide` を呼び出し [0x212e99c]、その後ボックスの交換フェーズを 1 つ進めます [0x212e9dc]。

`TradeSelectPokeModel$$PokeSelectWait` [0x1c26070] は、ボックスの交換フェーズが 2 を超え、受信 (+0x78) と自分の選択 (+0x79) の両方のフラグが立つと、
相手のポケモンを表示します。これを確認する処理はありません。`TradePokeCheckOkWait` [0x1c25f50] は、両方のチェック状態 (自分 +0x80、相手 +0x84) が 6 になると
次に進みます。信頼性ウィンドウが届けずに確認応答だけを返したポケモンは、エラーなしで
プレイヤーを "en attente d'une réponse" の画面に残します
([Pia 層](pia.md#what-the-receiver-discards-in-silence))。

ローカル無線では、プレイヤーが選んだポケモンは
セーブデータの不正フラグ (`CoreParam$$GetDprIllegalFlag`) だけを確認されます。`NetworkManager$$RequestValidateTrade` は
`UnionFrontDeskStateController.isGlobal` が立っているときにだけ実行されます。

`RequestValidateTrade` [0x02251cb0] では、クライアント側で何も確認しません。各ポケモンの `CoreParam` を
0x148 バイトのリクエストにコピーして、任天堂の検証サーバー
(`IlcaNetServerValidate.CheckRequestAutoAsync` [0x027318c0]) に送ります。応答は `ValidateResultID`
(None, InvalidData, SignatureError, ProcessError) で、署名エラーか処理エラーのときはエラーダイアログを表示します。
プレイヤー自身が選んだポケモンにフラグが立っていたときのメッセージ `SS_box_182` は、
「ポケモンに問題があるため、交換できません。」という文です (フランス語: "Un problème avec votre Pokémon rend tout échange impossible.")。
`SS_box_181` は、相手のポケモンについて同じ内容を表示します。

## 交換の状態遷移

`UnionTradeManager.currentState` (+0x88。1.3.0 では +0x80) は
`TradeFlowState {NONE 0, SELECT_WINDOW 1, SECURIY_TRADE 2, PLAY_DEMO 3, END 4}` です:

    UnionTradeManager$$RecivePokeData          0x1dd2800   currentState == SELECT_WINDOW only
    UnionTradeManager$$SetTargetTranerParam    0x1dd2130   TargetTranerParam{uint id, string name}
    UnionTradeManager$$ReciveTradeReadyOkData  0x1dd2a70   routed on the same field

ポケモンを受け取るのは SELECT_WINDOW の中だけで、`isRecivePokeParam` を立てたうえで
`tradeSelectModel.targetPokemonParam` に入ります。`NetDataTradeReadyOkData` (0x21) は、SELECT_WINDOW では
`TradeSelectPokeModel$$ReciveReadyOk` に渡され ([ボックスの各フェーズ](#box-phases-and-the-messages-that-reset-a-round))、
SECURIY_TRADE では `TradeSecurityController$$ReciveState` に渡されます (1.3.0 では [0x1c34290] で、
コントローラーの `GetCurrentState` が 0 以外のときだけ。0 のときは `SettingSecurityControllerParam` [0x1c343f4] を呼んでメッセージを捨てます)。
それ以外では捨てられます。

`ReciveReadyOk` [1.3.0 0x1c27d40] は、メッセージの 2 バイト目 `tradeState` を
`targetTradeState` (+0x78。1.3.0 では +0x84 で、受け取った check-ok もここに 6 を書き込みます) にコピーします。
`isTradeOk` は読まれません。`UnionTradeManager.<WaitBoxWindowComplete>d__24` は、`myTradeState` (+0x74。プレイヤーの `MyReadyOk` で設定) と
`targetTradeState` が両方とも 2 (`WAIT`) になるまで待ち、`currentState` を SECURIY_TRADE に設定して、
選択モデルをクリアし、`tradeState` 0 の 0x21 を自分から送ります。
本体を SELECT_WINDOW から抜けさせるのは、相手の 0x21 だけです。

### ボックスの各フェーズと、交換を巻き戻すメッセージ

`BoxWindow.NetTradePhase`:

    None 0, WaitSave 1, PlayerSelecting 2, WaitSend 3, OtherPokeConfirm 4, WaitOtherDecide 5,
    LastConfirm 6, WaitTrading 7, Complete 8, WaitClose 9, CancelOther 10, Error 11

`WaitSave` は何も書き込みません。`BoxWindow.<WaitTradeSave>d__203` が
`FieldCommonParam[0xEB]` * 0.001 秒を `Time.deltaTime` に対して数えます。

巻き戻しは `TradeSelectPokeModel$$ReciveReturnSelectPoke` [1.3.0 main 0x1c27f00] が行います。`isReturnSelect` が 1 のとき
は `NetDataReturnSelectData{0}` [0x1c27f48] で応答するので、本体は `{1}` のすべてに `{0}` で答えます。どの場合も
`isWaitingOK`/`isWaitingSelect` [0x1c27fb0]、`isRecivePokeParam`/`isSendPokeParam` [0x1c2803c]、両方の交換状態 [0x1c28040] をクリアします。
そのうえで `UnionWork` の static +0x50 (`boxState`) を 6 (`CANCEL_SELECT` [0x1c28030]) にします。
ボックスの交換フェーズが 3 以降のとき、
またはボックスがないときは、続けて `CloseOverUIWindows` [0x1c28168] を呼びます。

`BoxWindow$$UpdateNetworkTrade` [0x2121628] は `CANCEL_SELECT` を受け取って消去します [0x2121c64]。フェーズ 2 のときは、確定していない強調表示を外します。
それ以外のフェーズでは [0x21225b0]、ウィンドウを閉じ、フェーズを 2 にして、
`SS_box_588` を表示します ("L'autre joueur a choisi d'annuler l'échange.")。

| 相手からのメッセージ | ボックスのフェーズ | 本体の動作 |
|---|---|---|
| check-ok (0x46) | ボックスがないか、6 未満 | 相手のチェック状態 (+0x84) を 6 にする (`ReciveTradePokeCheckOk` 0x1c34630) |
| check-ok (0x46) | 6 `LastConfirm` 以降 | `{1}` と同じ巻き戻し [0x1c34718]、その後 `securityController?.ResetTradeState()` |
| ready-ok (0x21), SELECT_WINDOW | ラッチ +0x71 が立っている | 捨てる [0x1c342e4] |
| ready-ok (0x21), SELECT_WINDOW | ボックスがあり、フェーズ 5 以下 | `{1}` と同じ巻き戻し [0x1c3440c]、その後 `ResetTradeState()` |
| ready-ok (0x21), SELECT_WINDOW | フェーズ 6 以降、またはボックスがない | `ReciveReadyOk` を呼び、ラッチ +0x71 を 1 にする [0x1c3435c] |
| return-select (0x45), どちらの値でも | 2 `PlayerSelecting` | 巻き戻す。ボックスは確定していない強調表示を外す |
| return-select (0x45), どちらの値でも | 3 以降 | 巻き戻す。ボックスはフェーズ 2 に戻り、`SS_box_588` を表示する |
| return-select (0x45) | PLAY_DEMO 中ならどのフェーズでも | 巻き戻さない。[0x1c345c0] が `targetDemoPokemonParam` をクリアする |

ラッチ (+0x71、`<isLoadingBox>k__BackingField`) は、`Init`、`WaitBoxWindowComplete`、`Cancel`、`RecivePokeData` と
`ReciveCancelData` でクリアされます。セッター [0x1c33040] を呼ぶ箇所はありません。本体は、最後の確定のあとでボックスを閉じると
自分の ready-ok を送ります
(`b__54_4` -> `BoxCloseComplete` -> `MyReadyOk` 0x1c26d90 -> `SendReadyOk` 0x1c26f14)。

本体が自分で取り消す `onCancelSelect` (`b__54_5` 0x1c28620) は、ボックスをフェーズ 2 にし
(`ToNextPhase(box, 2)`。0 以外の引数は `[[box+0x390]+0x50]` に格納され、0 のときは 1 つ進める)、
`NetDataReturnSelectData{1}` [0x1c286a4] を送ります。そのうえで待機フラグ、ポケモンのフラグ、両方の交換状態をクリアします。
`CANCEL_SELECT` は立てません。その後、新しい `NetTradePokeData` を待ちます
(`isRecivePokeParam` を設定するのは `UnionTradeManager$$RecivePokeData` だけで、SELECT_WINDOW 内ならどのフェーズでも [0x1c33e9c])。
相手の `{0}` では待ちは解除されません。

- 本体の check-ok には、1 回だけ応答します。最初の応答のあとで届いた複製に答えると、ボックスが `LastConfirm` に進んでいて、交換が巻き戻ります。
  信頼性ウィンドウは ID ごとに最初のメッセージを保持します ([Pia のページ](pia.md#what-the-receiver-discards-in-silence))。
  取り込んだ本体の信頼性メッセージ 12858 件のうち、ID が重複していた 322 件は、
  バイト単位で同一の複製でした。受信側は、すでに配信した ID を
  捨てます。`pokeldn.ldn.reliable5.Reassembler` がこれを行い、`bin/bdsp_connect.py` がこれを使います。
- 本体の `45 0001 01` には `45 0001 00` で応答します。`{1}` が返ってきたら、それはクライアント自身の取り消しです。
  交換用のポケモンを送ったあとや、本体が選び直したあとに 0x45 を送ってはいけません。
  交換が巻き戻ってしまいます。
- 選択ウィンドウでは、本体の最後の確定より前に届いた 0x21 は取り消しとして扱われ、
  確定のあとに届いた場合は最初の 1 つだけが有効です。
  本体が選択ウィンドウに戻れる前に、セキュリティ状態の中継を止めてください。

## セキュリティの段階

次に `TradeSecurityController` -> `CreateTradeStateModel` -> `TradeStateModel` へ進みます。セーブデータを扱うのは `TradeStateModel` です:

    TradeStateModel.TradeState
    NONE 0, INIT 1, WAIT 2, SEND_POKE 3, WAIT_POKE 4,
    SEND_READYOK 5, WAIT_READYOK 6, START_WRITE_SAVE 7, WRITEING_SAVE 8

`TradeStateModel$$InitState` は最初に `PlayerSave` を呼びます。`WriteSaveData` は末尾で `ReplacePoke` を呼びます。
どちらも登録されたデリゲートとしてしか呼ばれません。`FirstSave` は書き込みの前に切断ペナルティを有効にします。

セキュリティの段階では、ポケモンのメッセージは次の段階に進むきっかけになるだけです。SECURIY_TRADE 中の
`UnionRoomManager$$RecivePokeData` は、デコードしたポケモンを捨てて `SetSecurityTradeParam()` を呼び、
`manager.targetPokemonParam` (+0x48。プレイヤーが全画面表示で確定したときに設定される) を、セキュリティコントローラーに渡します。
プレイヤーが確定するまではこれが null のままで、WAIT_POKE は終わりません。

### 先導する側: よりレアなポケモン

`CreateTradeStateModel` [1.3.0 main.bin 0x1c24620] は、どちらの役割にも `TradeParentStateModel` を作ります
(`TradeChildStateModel` のオーバーライドは中身のない `ret` だけです)。役割は `tradeParent` (+0x94) に入り、
`TradeParent {NONE 0, PARENT 1, CHILD 2}` の値を取り、`TradeSecurityController$$CheckPokeRarity` が書き込みます。
相手のポケモンが届くと ([0x1c25060]、`UnionTradeManager$$SetSecurityTradeParam` [0x1c34750])、
両方の種族について `Dpr.SubContents.Utils$$GetPokeRarityNum` [0x1cbe560] の値から:

    POKE_RARITY_VERY_RARE 3, POKE_RARITY_LEGEND_RARE 2, POKE_RARITY_SUB_LEGEND_RARE 1, anything else 0
    mine > theirs   PARENT
    mine < theirs   CHILD
    equal           PARENT if isRecruiment (+0x38), else CHILD
    species -1      no role set (to 0x1c25120)

`GetPokeRarityNum` は、3 つの静的な `MonsNo[]` のリストを順に走査します (0x1cbe614、0x1cbe698、0x1cbe71c。
空のリストは 0x1cbe730)。これらは 1.3.0 の `global-metadata.dat` から `Utils$$.cctor` [0x1cbe9a0] が埋めます:

| 静的フィールド | レア度 | メタデータ | 種族 |
|---|---|---|---|
| `very_rare_monsno` +0x78 | 3 | `0x666aa6` | 151, 251, 385, 386, 489, 490, 491, 492, 493 (幻のポケモン) |
| `legend_rare_monsno` +0x80 | 2 | `0x666b06` | 150, 249, 250, 382, 383, 384, 483, 484, 487 (ボックス伝説) |
| `sub_legend_rare_monsno` +0x88 | 1 | `0x667c39` | 144, 145, 146, 243, 244, 245, 377-381, 480, 481, 482, 485, 486, 488 |

本体のミュウに対して提示されたディアルガは、本体を PARENT のままにします。

`TradeParentStateModel$$StateProc` [0x1c23350] のテーブルは 0x3d80f2f にあります:

    2 WAIT            targetState == WAIT                    -> 3
    3 SEND_POKE       waitRndTime runs out; SendPokeData     -> 4
    4 WAIT_POKE       targetPokeData non-null                -> 5
    5 SEND_READYOK    PARENT only: send own state            -> 6
    6 WAIT_READYOK    targetIsTradeReadyOk; PARENT sends     -> 7
    7 START_WRITE_SAVE  WriteSaveData                        -> 8
    8 WRITEING_SAVE     CheckReplacePokeData                 -> 10

`TradeSecurityController$$ReciveState` [0x1c24f10] は、本体自身の状態で分岐します。テーブルは
0x3d80f39 にあります:

    1 INIT            -> WAIT, send own state
    3 SEND_POKE       send own state
    5 SEND_READYOK    CHILD only: peer 5 -> send, go to 6; peer 6 -> send, go to 6, set targetIsTradeReadyOk
    6 WAIT_READYOK    set targetIsTradeReadyOk
    every case        targetState = the peer's byte

`TradeStateModel$$SetTragetPokeData` [0x1c24d70] は、本体の状態 WAIT_POKE を送って終わります。
CHILD の本体はその後 SEND_READYOK に移り、相手の状態が 5 か 6 になるまで無言で待つので、
WAIT_POKE を返すクライアントは本体をデッドロックさせます。
`room.mirror_trade_state` は WAIT_POKE に SEND_READYOK で応答します。これは、どちらの役割でも `ReciveState` が受け取ります。
CHILD は SEND_READYOK (case 5) で、PARENT は WAIT_READYOK (case 6) で受け取ります。
`tests/test_bdsp_trade_states.py` は、両方の関数をモデルとして、ランダムな遅延のもとでクライアントの方針と照らして実行します。

信頼性シーケンスの ID 1 つにつき、メッセージは 1 つだけ送ります。`their_ack_id` は本体が確認応答したときにだけ進むので、
同じ ID の後続のメッセージは再送と見なされて捨てられます。
`bdsp_connect` は独自のカウンターを持ちます。

## 交換の完了

本体が勧誘側かつ PARENT のときの、セキュリティ状態の順です
(本体の状態、その後にクライアントの応答):

    their READY-OK {isTradeOk 0, tradeState 2}  ->  the client's READY-OK
    INIT          ->  WAIT
    WAIT          ->  WAIT
    SEND_POKE     ->  SEND_POKE, and its Pokemon
    WAIT_POKE     ->  WAIT_POKE
    SEND_READYOK  ->  SEND_READYOK
    WAIT_READYOK  ->  SEND_READYOK

WAIT_READYOK のあと、本体はセーブデータを書き込み、アニメーションを再生して
`ReplacePoke` を実行します。このとき送るのは `NetCharacterStateData` だけです。WAIT_READYOK を抜けるには、
相手からもう 1 つメッセージが必要です。本体は、どちらの役割でも WAIT_READYOK に入るときに
SEND_READYOK を報告します (親は `StateProc` の case 5、子は `ReciveState` の case 5)。そのため、この報告へのクライアントの応答は、
報告の中で届きます。市販の交換 25 件 (両方の役割、25 件すべて) では、本体の次の信頼性メッセージが、
繰り返しより前に SEND_READYOK の 0.16 から 0.24 秒後に届きました。親の WAIT_READYOK の報告は、その 0.02 から 0.06 秒後に続きました (19 件中 19 件)。
クライアントは、本体の SEND_READYOK より前だけ、1 秒ごとに自分の状態を繰り返します (`room.repeats_trade_state`)。
それより前では、子の SEND_READYOK は、その中で届くメッセージでしか終わりません。この間は、クライアントが接続を離れてはいけません。
途切れると、本体は `FirstSave` と `SecondSave` の間で止まります。
部屋に参加する側を本体、ホストを pokeldn にしても、同じ手順になります。

市販の BDSP が `bin/bdsp_host.py` に参加したときの、本体の `tradeState` 6 からの時系列です。アニメーションは約 2.7 秒で始まり、
受け取ったポケモンは約 18.6 秒で表示され、プレイヤーの操作は約 29 秒で可能になります
(手で押した計測で、最大 2 秒の遅れがあります)。

完了した交換は、メッセージで終わることはありません。クライアントは、本体の SEND_READYOK に応答したときに完了として数えます。
次の周回は、本体の次の `NetTradePokeData` から始まります。

`NetDataReturnSelectData` (0x45)、`45 00 01 00` (`{isReturnSelect: 0}`) は、巻き戻しに対する本体の応答です
([ボックスの各フェーズ](#box-phases-and-the-messages-that-reset-a-round))。`{1}` のとき、またはボックスのフェーズが 5 以下のときに
選択ウィンドウに 0x21 が届いたときに、同じ経路で巻き戻し [0x1c3440c] を行い、`SS_box_588` を表示します。
これには応答を求めません (`{1}` に答えると `{0}` が返り、すでに空の周回を巻き戻すことになります)。
アニメーション中にクライアントが 1 秒ごとに自分の状態を繰り返した、取り込み済みの交換 26 件すべてで、
クライアントの 0x21 のひとつに 25 から 300 ミリ秒遅れて `{0}` が届き、プレイヤーは `SS_box_588` を見ました。
本体の SEND_READYOK のあとに 0x21 がない場合は、両方の役割の市販の交換 5 件 (それぞれ 1 つの接続で 2 件ずつ並べた) で、
本体は 0x45 を送らず、キャンセルも表示しませんでした。`TradeSelectPokeModel$$SendReturnSelectPoke` [0x01c27c20] がこれを組み立てます
(`isReturnSelect` は引数の否定で、`tradeTargetIndex` +0x48 に送ります)。直接の `bl` 呼び出し元はありません。

交換は 1 つの接続で連続して行え、それぞれ選択ウィンドウから、再度の接続やトレーナーの記録なしに繰り返されます。
`bin/bdsp_connect.py` で 3 件を続けて完了し (毎回ボックス画面に戻る)、`bin/bdsp_host.py` をホストにして 2 件を完了しました。
`TradeStateModel$$ReturnTradePokeSelectWindow` [0x01c29590] は `PlayerSave` を実行し、そのあとモデルの +0x80 のコールバックを呼びます。
呼び出し元は追っていません。2 回目の交換は、本体が保存した内容を読み戻します。
プレイヤーが選んでいる間 (ボックスのフェーズが 5 以下のとき) に SEND_READYOK (5) の 0x21 が届くと周回が巻き戻るため、
本体の SEND_READYOK のあとに繰り返しは続きません。

## ポケモン

`NetTradePokeData` は 328 バイトで、第 8 世代の `SIZE_STORED` です。暗号化された PB8 で、形式は
[ソード・シールドのページ](swsh_protocol.md#the-pk8) にあります (`pokeldn/gen8.py`。PB8 の表示は `pokeldn/bdsp/pokemon.py`)。
0x06 のチェックサムは復号した本体の合計で、組み立てたポケモンを復号し直すと検証できます。
ただし、ブロックの順番が誤っている場合は、16 ビット語の合計では検出できません。

`NetDataTradeTranerData` は、`string tranerName; uint tranerId; byte
cassetVersion; byte langId` を並べたマーシャリングされた構造体です ([フレーミング](bdsp_protocol.md#framing)):

    0x00  26  tranerName, UTF-16LE, NUL-terminated
    0x1a   4  tranerId, the full 32-bit id, secret id in its high half
    0x1e   1  cassetVersion, the save's version (49)
    0x1f   1  langId, the save's language (3)

本体は交換が始まると、ポケモンより前にこれを送るので、セーブデータのトレーナーの名前になります。
受け取る側のプレイヤー向けに作られたギフト (`room.traner_trainer`、
[受け取る側のプレイヤー向けに作る](gift_simulated.md#made-for-the-receiving-player)) がこれを読みます。
性別は含まれず、セッション中の他のメッセージもこの名前を示しません。参加時のアバターは、
プレイヤーがユニオンルームの任意の外見設定を使ったあとにのみ、セーブデータの性別に従います。
それまでは、最初に接続したときにどちらかの性別のランダムなアバターが割り当てられます ([モデル](bdsp_protocol.md#the-model))。
本体が提示するポケモンは、この名前を使います。セーブデータが元のトレーナーのものなら OT の名前、
それ以外なら取り扱ったトレーナーの名前です ([本体が受け取ったポケモンに行うこと](#what-the-console-does-to-a-received-pokemon))。

交換画面は、この記録から相手の名前を表示します。挨拶では Pia のプレイヤー名を使います
([プロトコルのページ](bdsp_protocol.md#the-name-in-the-greeting))。`UnionRoomManager$$SetNetData` [1.3.0 main.bin 0x1e51940] の 0x24 の分岐は、
メッセージから 4 つのフィールドをすべて取り、フォントの言語だけを `GetGamerData(...).nameStringLanguage` から取ります。
名前は `MessageHelper$$SurroundFontTag` で包み、
`UnionTradeManager$$SetTargetTranerParam` に渡します
[0x1c33330]。

名前の終端と ID の間にある 10 バイトは、ヒープの残りです (`AllocHGlobal` はゼロクリアしません。
0x14 の語は毎回変わります)。`room.build_trade_traner` は、本体の残りのデータを
バイト単位でそのまま運びます。

提示するポケモンは、名前付きのフィールドを変えた実在のポケモンです (`pokemon.build_from`)。出会いの記録、リボン、
取り扱いの記録、言語は、本体自身のポケモンでは 0 以外になります。

## 本体が受け取ったポケモンに行うこと

元のトレーナーが受け取り直した場合は、2 バイトが変わります。`IsNicknamed` (0x08F の bit 7、IV32 の bit 31) は、
名前がゲームの言語での種族名と異なるときに立ち、チェックサムも追従します。名前の文字列は変わりません。
種族名は種族名のままです。

別のトレーナーが受け取った場合は、11 バイトが変わり、`ot_name` は変わりません:

    0x0A8..0x0B2   HandlingTrainerName, UTF-16LE
    0x0C3          HandlingTrainerLanguage (3, French)
    0x0C4          CurrentHandler, 0 -> 1
    0x0C8          HandlingTrainerFriendship, 50
    0x006..0x007   the checksum

0xC6 (PKHeX の `HandlingTrainerID`、"unused?") は 0 のままです。
PKHeX の `IsUntraded` (`Data[0xA8] == 0`) は false になります。

`TradeStateModel$$ReplacePoke` [1.3.0 main 0x1c29120] がこの変更を行います。セーブデータの `DPData.MYSTATUS` を元に
(`PlayerWork$$get_playerStatus`、コンストラクター [0x24a1b30]) `OwnerInfo` `{trainerId, sex, langID, name}` を組み立てます。
内容は、ID、`sex ^ 1`、`region_code`、名前です。ステータスの `sex` は、`_GET_MY_SEX` が読むとおり、男の子のとき true です。
`OwnerInfo` を `CoreParam$$UpdateOwnerInfo` [0x2495920] に渡し、そこでポケモンの OT の性別、32 ビットの ID、
OT の名前と比べます:

- 3 つとも一致: 取り扱いのフラグをクリアし、他には何も書き込まない
- それ以外: フラグを立て、取り扱ったトレーナーの名前、性別 (0xC2)、言語をセーブデータから設定する。
  取り扱ったトレーナーの記憶はゼロにし、なつき度は種族の基本なつき度に設定する

0xC2 は、別のトレーナーが受け取るたびに書き込まれます。上の計測では、すでに受け取り側の性別が入っていました。
取り扱った記録の ID やバージョンは残りません。GTS (`GMSSceneDataModel$$OnReceivePokeData`)、
ゲーム内交換 (`LocalKoukan$$CreateTradePokeParam`)、ふしぎなおくりもの (`MysteryGiftWork$$CreatePokemonParam`)、
`RenkeiData$$Get_1`/`Get_2` は、セーブデータから同じ `OwnerInfo` を組み立てます。
`UpdateOwnerInfo` は、ゲームが取り扱ったトレーナーの性別を書き込む唯一の箇所です。
ゲームにはポケモン HOME のコードはありません。HOME から届くポケモンは、HOME が書き込みます。

### 重複の検出

`PokeDupeChecker` (1.3.0 で追加) は、重複したポケモンに不正フラグを立てます。フラグは、
復号した PB8 の 0x52 バイトの bit 0 です (ブロック A + 0x4A、`CoreDataBlockA.set_dpr_illegal_flag` `0x027bb040`)。
PKHeX はこれを `PB8.IsDprIllegal` として読みます。フラグの立ったポケモンは交換できません
("Un probleme avec votre Pokemon rend tout echange impossible.")。

`UpdateIllegalFlagAll` [`0x01de5860`] は、手持ち、ボックス 1 から 40、育て屋の順に `CheckDuplicate` [`0x01de59d0`] を実行します。
ポケモンが対象になるのは、出身ゲームがブリリアントダイヤモンドかシャイニングパール (`version & ~1 == 0x30`) で、タマゴでなく、
フラグが立っておらず、ゲーム内交換のポケモンでもない場合です (`IsLocalKoukanPokemonParam` [`0x01de66d0`]:
出会った場所 30001 で、トレーナー ID、暗号化定数、性格が `LocalKoukanData` のエントリと一致するもの)。
各ポケモンは、それより前のすべてのポケモンと比べられます。最初のコピーは汚れていないままで、
以降のコピーはすべてフラグが立ちます。

`IsDuplicatedPokemonParam` [`0x01de62f0`] は、次のすべてが一致するかで判定します:

| フィールド | アクセサ | PB8 のオフセット |
|---|---|---|
| 暗号化定数 | `GetPersonalRnd` | 0x00 |
| PID | `GetColorRnd` | 0x1C |
| トレーナー ID (TID16、SID16) | `GetID` | 0x0C |
| 性格 | `GetSeikaku` | 0x20 |
| 6 つの個体値 | `GetTalentHp` .. | 0x8C |

テッカニン (291) とヌケニン (292) の組は、重複になることはありません。種族、フォルム、ニックネーム、OT の名前は
比較しません。

`UpdateIllegalSpecialTraining` [`0x01de6830`] は、ブリリアントダイヤモンドかシャイニングパールのポケモンで、
レベル 99 以下かつハイパートレーニングのビットがひとつでも立っているものにフラグを立てます。

検査は、セーブデータの読み込み時 (`PlayerWork.OnPostLoad_NeedMD`)、交換ボックスでポケモンを選ぶたび
(`TradeSelectPokeModel` [`0x01c28310`])、ワンダー交換のセーブの前 (`Dpr.GMS`) に実行されます。
受け取りのときには何も実行されません。ローカル交換で、フラグの立ったポケモンを選ぶと、`UnionWork.boxState` が `INVALID_DATA` になり、
ボックスはそれを拒否します。オンライン交換では、代わりに選んだポケモンを
`NetworkManager.RequestValidateTrade` に送ります。`ClearIllegalFlagAll` を呼ぶ箇所はないので、
フラグが解除されることはありません。

暗号化定数、PID、トレーナー ID、性格、個体値がすべて、受け取り側のセーブデータにあるポケモンと一致する
ブリリアントダイヤモンドかシャイニングパールのレコードは、決して提示してはいけません。`bin/bdsp_host.py --fresh-pid` は、
色違いの状態を保ったまま、新しい暗号化定数と PID を選びます。元のポケモンを持つセーブデータへ、
そのように交換したレコードを入れても、フラグは付きませんでした。

## 切断のペナルティ

`FirstSave` と `SecondSave` の間でクライアントが切断されると、ペナルティは有効なまま残ります。
本体は新しいローカル交換を "vous ne pouvez pas faire d'echange en reseau pour le moment" というメッセージで拒否します。
解除されるまで続きます:

    TradeStateModel$$FirstSave    0x1cd4fc0   SetPenartyCounter(30); SetPenartyTime(now)
    TradeStateModel$$SecondSave   0x1cd5030   SetPenartyCounter(0)
    UnionFrontDeskStateController$$CheckPenarty  0x1fcc400   counter >= 1 AND not CheckDateTime()

交換が完了すると、ペナルティは解除されます。ペナルティがあるということは、`FirstSave` が実行され、本体が書き込んだことを意味します。
`UnionWork$$CheckDateTime` [0x1dd3e30] と `SetPenartyTime` [0x1dd32d0] は、重みのない和を使います:

    w8 = Year + Month + Day + Hour + Minute + Second
    cset w0, mi            ... on  (stored + 30.0) < w8

この和は 1 秒ごとに 1 増え、分または時が繰り上がるときに 58 減ります (18:45:20 は 2125、19:20:00 は 2081)。引き継がれるのは日だけです。
`Hour+Minute+Second` は 0 から 141 までの値を取るので、時間の途中で有効になると、待ち時間は翌日以降に及ぶことがあります。
コードには期間の定義がありません。

ペナルティを解除するために、本体の時計やシステム設定を変えてはいけません。BDSP が変更を検知し、
時間に依存する機能を 1 日ロックします。時計を戻すとペナルティが復活します。カウンターをゼロにするのは
`SecondSave` だけです。
