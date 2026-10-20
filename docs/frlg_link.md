---
title: リンクプロトコル
parent: FireRed and LeafGreen
nav_order: 1
---

# GBA リンクと、その上で動くもの

市販の実機で計測したもの、または [pret/pokefirered](https://github.com/pret/pokefirered) から読み取ったものです。

# RFU リンク層

## 親機のポーリングごとに子機スロット 1 つ

`RfuMain2_Parent` は、ポーリングごとに 1 つの子機スロットを `gRfu.childRecvBuffer[i]` に保持し、その
`childSendCmdId` タグが直前に保持した値の `+1 mod 8` になっているかを確認します [link_rfu_2.c:876-892]。不正なタグを受けると
`numChildRecvErrors[i]` が増え、`> 4` になると `RfuSetErrorParams` を呼んでリンクを切断します。正しいタグを受けるとカウンタはリセットされるため、
切断には 5 回連続で不正なポーリングが必要です。1 回のポーリングに 2 つ目のスロットが入ると、タグが 1 つ抜け落ちた扱いになります。次の表は、
交換ルームへ歩いて入る市販版ファイアレッドで、子機の送信レートごとに計測した結果です。

| 子機の送信 | 本体がポーリングを停止した時点 |
|---|---|
| 自走 (本体の ~55/s に対して ~57/s) | 歩き始めてから 0.10 s 後 |
| 1 回のポーリングに 2 スロット | 歩き始めてから 0.28 s 後 |
| 1 回のポーリングに 1 スロット | 歩き始めてから 5.8 s 後 |

1 回のポーリングに 1 スロットの場合は、タグがすべて正しい順序で届き、`numChildRecvErrors` は一度も増えません。
その速度なら歩いた先で席に着けるため、5.8 s の停止はタグ検査が原因ではありません。

例外はなく、席までの歩きも含まれます。

## 親機テーブルの 1 行目は、本体自身のコマンドがミラーされて戻ったもの

この規則は、本体が送信中に起きるすべての停止を支配します。どの活動中でも同じです。

`MGL_Send` は 252 バイトごとに分割し、各チャンクの前に `MGL_HasReceived(link->sendPlayerId)` を待ちます。
末尾でも同じ待ちを行います [mystery_gift_link.c:176,205]。`sendPlayerId` は本体自身の ID 1 です
[mystery_gift_client.c:33] そのため、自分のブロックが親機テーブルの 1 行目を通って完全に戻るまで待ちます。
戻ってくるのは、ホストがミラーする親機の `gRecvCmds` です。`RfuHandleReceiveCommand` は
子機自身を含むすべてのプレイヤーについてブロックを組み立てます [link_rfu_2.c:1125]。`RfuMain1_Child` は
自分の行を含めて親機テーブルから `gRecvCmds` を埋めます [:970]。

RFU のブロック送信側も同じミラーを待ちます。`HandleBlockSend` は、ミラーで見えるまで INIT を保持し、
`SendLastBlock` はミラーされるまで最後のフラグメントを繰り返し、その後、ミラーされたビットマスクに
欠けているフラグメントを再キューします [HandleSendFailure, link_rfu_2.c:1366-1416]。本体は欠けたフラグメントを
指定できないため、すべてを再送します。

`rfu_leader.ChildEcho` には 2 つの規則があります。

- 異なるコマンドは落とさないこと。落としたフラグメントは盲目的な修復ラウンドになります。上限付きのエコーキューは、
  バーストで届くコマンドを落とします(21 フラグメントのチャンクは 2 件の上限を超えます)。すると本体はブロック全体を繰り返し、
  2 回目の欠落でリンク切断になります。
- 待機中の繰り返しは統合すること (`SendLastBlock` は毎フレーム再送します)。エントリは 1 件で足ります。繰り返しが
  ミラーの送出後に届いたものなら、新しい質問なので応答します。

子機は受け取った親機フレームごとに 1 コマンドを送ります (`childSendCount` は
`recv.newDataFlag` [link_rfu_2.c:600] のときだけ増えます)。そのためミラーのキューは単独で膨らみません。

## タイルの 1 歩はちょうど 16 回のリンク更新

`FacingHandler_DpadMovement` は `objEvent->directionSequenceIndex = 16` を設定します。
`MovementStatusHandler_TryAdvanceScript` は、キーが無視されている間に、リンク更新 1 回ごとにこの値を 1 ずつ減らします。
(`MOVEMENT_MODE_FROZEN`) [overworld.c:3432-3470]。方向キー N 回の入力では、その最後の
1 歩のうち `N mod 16` 回分が消費された状態で残るため、スクリプト化された歩行の隙間に必要なのは `16 - (N mod 16)` 回の更新だけです。行き過ぎた分は
更新 1 回ごとにスロット 1 つを消費するだけで、ほかの影響はありません。

## プレイヤーの入力が必ず送信されるとは限らない

`UpdateHeldKeyCode` は、EMPTY・十字キーの 4 コード・START・A を、
`GetLinkSendQueueLength() > 1` のとき `LINK_KEY_CODE_NULL` に書き換えます [overworld.c:2786-2810]。すると `SendKeysToRfu` は何も送らず、
キュー圧迫中に歩くプレイヤーは止まって見えます。`LINK_KEY_CODE_READY` (0x16) だけは例外なので、判定は 0x16 を基準にし、
十字キーのコードが無いことを基準にしないでください。

## 座席は相互の待ち合わせ

`Task_EnterCableClubSeat` は「Please wait」を表示し、`SetInCableClubSeat()` を呼び出し (次に送られる保持キーが
`LINK_KEY_CODE_READY` になります)、その後 `GetCableClubPartnersReady()` を回し続けます
[cable_club.c:827-869]。これは `AreAllPlayersInLinkState(PLAYER_LINK_STATE_READY)` が真になったときだけ成功します
[overworld.c:2988-2999]。その後 `Task_StartWirelessTrade` が `SetLinkStandbyCallback()` を実行します
[cable_club.c:910-943]。これが座席後のスタンバイラウンドの発生源です。両方のプレイヤーが
READY になってから駆動してください。`CABLE_SEAT_WAITING` のままの本体に対して駆動すると、その座席のステートマシンが故障します。

## 実機の子機が端から端まで送るもの

ホストと対戦する、市販のフランス語版ファイアレッドを子機にした場合です。順序はプロトコルで決まっており、IDLE の
回数は 1 回の交換から取ったものです。

```
IDLE x8
SEND_BLOCK_INIT w1=0x0011 x4   + 17 fragments      (LinkPlayer, 0x11 = 17)
IDLE x31
SEND_BLOCK_INIT w1=0x0009 x4   + 9 fragments       (trainer card)
IDLE x262   READY_EXIT_STANDBY w1=0x0000 x1
IDLE x72    READY_EXIT_STANDBY w1=0x0001 x1
IDLE x45    <room: SEND_HELD_KEYS 0x1b x24, EMPTY, 0x1a once, the walk, READY, then EMPTY x13>
IDLE x22    READY_EXIT_STANDBY w1=0x0002 x1
IDLE x28    READY_EXIT_STANDBY w1=0x0003 x1
IDLE x75    -> the host pulls the party
```

- 各スタンバイラウンドは 1 フレームで、届くまで Pia Reliable が再送します。カウントを繰り返すと、
  ホストには完了済みのラウンドを見せることになります。
- ラウンドの間、子機は完全に IDLE です (`gSendCmd` がすべてゼロ)。パーティ交換の前にあるリーダーの静かなフレームカウンタは、
  完全に空のスロットでだけ進むため、EMPTY のキープアライブはそれを行き詰まらせます。
- ルーム読み込み時の前置き (`0x1b` = HANDLE_RECV_QUEUE を 24 回、続けて `0x1a` = IDLE) は、キューの後片付けです。
  `LINK_KEY_CODE_IDLE` は相手側の `sPlayerLinkStates[player]` を IDLE に設定し [overworld.c:2755]、
  `HandleLinkPlayerKeyInput` は IDLE の状態にあるプレイヤーに対してだけタイルのスクリプトを実行します。

## 両側は、相手から何か受け取ったときだけ進む

`CB1_UpdateLinkState` は、`!IsRfuRecvQueueEmpty()` のときだけ `UpdateAllLinkPlayers` を実行します。この関数は
`gRecvCmds` のエントリがどれか 1 つでもゼロ以外なら FALSE を返します [link_rfu_2.c:787-800]。`MoveSendCmdToRecv` は親機自身の
`gSendCmd` を `gRecvCmds[0]` にコピーするため、親機は自分で動きを保てます。実際には、ループは
1 往復につき 1 回の交換に落ち着きます。

ホストの `SEND_HELD_KEYS` の上位バイトは `heldKeyCount` で、準備されたコマンドごとに 1 ずつ増えます。そのため最後の値は、
交換ルームが耐えたリンク更新の回数を数えます。

## 座席後のスタンバイゲートと退室

両者が座ると、本体は mpId 0 で自分の `READY_EXIT_STANDBY` を count=2 で送信します。その時点は、ホストのものを
反映した 130 ms 後です (記録された交換の場合)。本体は子機の count を、自分の count と一致するときだけ受け付けます
(`Rfu_LinkStandby` の受信ゲート、link_rfu_2.c:1577-1591)。そのため、ホストの count=2 の反映に対して送られた count=3 は無視されます。反映が示すのは、親機がスロットを見たことだけです。count=3 は
ホストの mp0 の count=2 を基準にしてゲートし、`BufferTradeParties` の前にある
ホストの静かな期間より長い間隔を空けて、何度でも再武装し続けてください。
(`HostTradeTiming.entry_final_standby_quiet_frames`、75 スロット。子機が
`READY_EXIT_STANDBY` を再送する 60 フレームより長い [link_rfu_2.c:1529])。

退室の流れは次のとおりです。ホストは `LINK_KEY_CODE_EXIT_ROOM` (0x17) を出し、
`AreAllPlayersInLinkState(EXITING_ROOM)` が真になるまで `KeyInterCB_WaitForPlayersToExit` で待機します
[overworld.c:2962-2981]。子機は保持キーの流れで自分の 0x17 を返さなければなりません。全ゼロのスロット
はキーとして扱われません。

## 部屋からの退出

トレードセンターから、座る前や対戦の後に歩いて出た本体は、保持キー `0x17` (EXITING_ROOM) を送り、
全プレイヤーが送るまで「Please wait」で待機します
[KeyInterCB_WaitForPlayersToExit, overworld.c:2977]。ホストは自分の `0x17` を
`H_ENTRY_SEAT`、`H_RETURN_FIELD` と対戦の各状態で返します (`host_trade._child_send_held_keys`)。市販版ファイアレッドは、ホストがまだ椅子へ歩いている間に
退出した場合、「Please wait」のまま、
`H_ENTRY_SEAT` が応答するまで待たされました。

### 静かな間のゲート

本体が `QUIET_GATE_SECONDS` (0.25 s) の間何も送っていないとき、ホストは自分の再送を捨てます。
ただし `QUIET_RETRANSMIT_SECONDS` (1 s) ごとに 1 回だけ再送します。そのため、ホストの穴のガード
(`HOST_OUTSTANDING_MAX`) の後ろで止められているフレームを待つ本体も、それを受け取れます
(`tests/test_host_quiet_gate.py`)。

## 交換後のキャンセル

`BufferTradeParties` は、ギフトリボンの交換の後に
受信済みブロックのフラグを消去します [trade.c:1549]。`Leader_ReadLinkBuffer` がメニューのコマンドを読むのは [1593-1633] その後です。その交換の間に完了したキャンセル
要求は、相手の選択を設定する前に消去されることがあります。
リーダー自身の `REQUEST_CANCEL` (`0xEEAA`) は、リーダーがキャンセル入力を処理していることを示します [2049]。
`bin/frlg_trade_join.py` は、設定された交換を終えてキャンセルを選んでいると、
リーダーの `REQUEST_CANCEL` が届いたときに自分の `REQUEST_CANCEL` を再び送ります (`pokeldn/frlg/link/trade.py`、
`_on_linkcmd`)。
`BOTH_CANCEL_TRADE` は、退出のスタンバイラウンドの前に保留中の要求をすべて消去します。

フォロワー側は、YES の後に生きているメニューから `REQUEST_CANCEL` を送り、「waiting for friend」と表示して、
`BOTH_CANCEL_TRADE` が届くまでアイドル状態になります [trade.c:2049, 1643]。キャンセルを選んだプレイヤーのリーダーは、
それを読んだときに応答します [1715-1722]。ホストは本体の `REQUEST_CANCEL` を、最後のメニューで
すぐに受け付けます。

## 片側だけのキャンセルは両側をメニューに戻す

`PLAYER_CANCEL_TRADE` / `PARTNER_CANCEL_TRADE` は `CB_HandleTradeCanceled` → `CB_MAIN_MENU` を通ります
[trade.c:2094-2113]。セッションを終えるのは `BOTH_CANCEL_TRADE` だけです [1715-1722]。参加側は、
S4_PARTY に 60 フレーム後に戻ってもう一度選びます。`REQUEST_CANCEL` に `PARTNER_CANCEL_TRADE` で応答すると、
本体は "votre ami veut échanger des Pokémon" の表示でループします。`bin/frlg_trade_host.py` は、
最初の 1 回には `BOTH_CANCEL_TRADE` で応答します。これが終了経路です。リーダー自身の選択は何も送りません (`SetReadyToTrade`
[trade.c:1811-1828])。そのキャンセルは `REQUEST_CANCEL` としてすべてのプレイヤーに送られます [trade.c:2049]。参加側が先に
`READY_TO_TRADE` を送っていた場合、リーダーの最初のキャンセルで `PLAYER_CANCEL_TRADE` を受けます。
`bin/frlg_trade_join.py` はそのあとメニューでキャンセルするため、リーダーの 2 回目のキャンセルでセッションが終わります。

## リンク上のバージョンと言語

`IsTryingToTradeAcrossVersionTooSoon` [union_room.c:1499] は、FireRed でも LeafGreen でもない相手にだけ発動し、
リンクを切断せずにメッセージを表示します。FR↔LG の交換は実機で動きます。`ConvertInternationalString` は日本語の名前を
特別扱いします。フランス語版のファイアレッドは、英語版の Wonder Card を受け付けます。ユニオンルームの
`Task_SearchForChildOrParent` は日本語の候補を飛ばします [union_room.c:3726]。ふしぎなおくりものは
`Task_ListenForCompatiblePartners` を使い、その互換性判定は言語フィルターを使わず、シリアル番号と
公開された名前のフラグで行います。日本語版のカートリッジは、ほかの Wonder Card や Wonder News の
活動番号を受け付けます
([日本語版のレイアウト](frlg_rom_map.md#japanese-layout))。

ふしぎなおくりもののともだちリストで pokeldn を選ぶと、本体は GameData の中で ROM のゲームコードを送ります。
ホストは配送の前に、そのカートリッジのアドレスとカードのレイアウトを選びます。
GUI は、この自動の言語判定を Basic 画面のバージョンの横に表示します。

## エミュレーターが自分でリンクを閉じることがある

`HandleLinkConnection` は Switch 版のビルドでだけ `svc_51` を実行します [link.c:1654]。

    #if REVISION >= 0xA
        if (svc_51())
        {
            ...
            CloseLink();
        }
    #endif

`svc_51` (`swi 0x51`) はエミュレーターが応答します [sloopsvc.c:120]。非ゼロなら `CloseLink()` を実行し、
`Task_MysteryGift` の下で `RfuSoftReset()` を、そうでなければ `RfuReloadSave()` を実行します [link.c:1674]。Switch のエラーは
2318-0006 です。RFU 層でホストがきれいなまま切断されたなら、原因は LDN/Pia にあると考えられます。

| SVC | 呼び出し元 | 動作 |
|---|---|---|
| `swi 0x45` | librfu_rfu.c:667,749 | エミュレーターに `gRfuLinkStatus` を渡す |
| `swi 0x49` | AgbRfu_LinkManager.c:657 | 非ゼロの間、SEARCH_CHILD 中は `connect_period` を開いたままにする |
| `swi 0x4a` | AgbRfu_LinkManager.c:720 | 同じ。SEARCH_PARENT 中 |
| `swi 0x4b` | link_rfu_2.c:2114, union_room_player_avatar.c:518 | `SVC4B_EXIT_EARLY` は SpawnGroupLeader から抜ける。`SVC4B_RESEED_RNG` はホストのトレーナー ID から乱数を再シードする |
| `swi 0x51` | link.c:1654 | リンクを今すぐ閉じる (ふしぎなおくりもの中はソフトリセット) |
| `swi 0x53` | wireless_communication_status_screen.c:328 | ステータス画面から、エミュレーター主導で終了する |

# このゲームが使う無線層

## 広告されるレートの集合

6、9、12 Mbit/s を含まない association response を受けると、本体は Pia セッションの確定後、どのリンク段階でも、LDN ネットワークから
関連付けの 2.9 から 3.9 s 後に離脱します。このような最初の関連付けは 76 件中 42 件が離脱し、それらのレートを含む場合は 284 件中 0 件でした。
Pia の type 2 Join Response が欠けると、
同じ時点で離脱します ([The console as a Pia child](#the-console-as-a-pia-child))。
ホスト型の FireRed と LeafGreen セッションの 360 件の空中キャプチャから、各実行の最初の関連付けが
6 s 以内に本体が離脱したかどうかと対応づけたものです。

| association response | beacon | 本体の association request | 3 s 以内に離脱 | 残留 |
|---|---|---|---|---|
| 6, 9, 12 あり | あり | あり | 0 | 139 |
| 6, 9, 12 あり | あり | なし | 0 | 84 |
| 6, 9, 12 あり | なし | あり | 0 | 26 |
| 6, 9, 12 あり | なし | なし | 0 | 35 |
| 6, 9, 12 なし | なし | なし | 42 | 34 |

応答のレートの集合だけを交互に変えると、そのレートがある場合は 20 件中 0 件が離脱し、ない場合は 8 件中 4 件が離脱しました。失敗した
集合は `1B 2B 5.5B 11B 18 24 36 54` でした (6、9、12、48 がない)。そのうちどれを本体が必要とするかは
不明です。ESP32 softAP の association response は 12 個のレートをすべて含みます
([hardware_esp32.md](hardware_esp32.md)、The access point's frames)。

ホストは Switch のレート集合を広告します (1B 2B 5.5B 11B 6 9 12 18、拡張として 24 36 48 54)。Switch の
そのほかの要素 (DTIM 2、ERP、capability 0x411、Nintendo のベンダー要素、HT/HE、WMM) は
必要ありません。本体は関連付け要求のレートをビーコンから組み立てるため、ビーコンの先頭には要素 0 (SSID)、
1 (Supported Rates)、3 (DS Params) を入れます。これは 41 バイトの部分集合です。WMM を含むビーコンを受けると、
本体は QoS データフレーム (サブタイプ 8) を送ります。ホストはこれを復号しなければなりません。同梱の LDN
デコーダーは TID を CCMP のノンスと AAD に取り込みます。`host_pia` は type 5 Update Session をユニキャストします。
privacy capability のもとで RSN のない probe response を返すと、本体の関連付け要求には
応答されないままになります。

## Pia の子機としての本体

- 本体の子機には type 2 Join Response が必要です。type 5 だけでは、join 要求を
  0.5 s ごとに再送し、ユニキャストの type 5 のコピーを無視し、参加後およそ 3 s でネットワークから離脱します
  (3.05 から 3.20 s で計測)。これは短いレート集合と同じ症状です。
- ふしぎなおくりものをホストしている本体は、子機の Net 0x11 から 50 から 66 ms 以内に、子機へ type 5 で応答します。
  `bin/frlg_mg_client.py` には type 2 を送らず、2.0 s 後に type 5 を送ります。join 要求のどのフィールドがこの差を生むかは
  不明です。交換をホストしている本体は、34 ms 以内に type 5 と type 2 を送ります。RTT
  は確定後にだけ行われます。

ふしぎなおくりものをホストしている本体から受信クライアントまでの流れを、その Net 0x11 からの時刻で示します。

    0.000  Net 0x11 (once)
    0.006  client: Net 0x12 + Session join
    0.252  RTT request, then every 316ms (6 probes, nothing else)
    2.038  Session type 5, no type 2 ever
    2.040  client: type 6 (finalize); 2.159 reliable open
    2.318  host 'A' frame
    4.63   host's own NI (join status = the player's YES on the console)
    6.82   SEND_PLAYER_IDS, 6.87 BLOCK_REQ (2.2s after the YES)
    8.15   Net 0x50 property update every ~0.5s, acked 0x51

## Pia セッションからの離脱

子機は、RFU の `D` フレームの後、3 段階でセッションを離れます。一時停止、セッションの type-3 離脱
要求、その後の LDN 認証解除 (理由 3) です。type-4 の応答がない、ホスト型の交換とふしぎなおくりものの
33 件のキャプチャで計測しました。

| 区間 | 計測値 |
|---|---|
| `D` から最初の type 3 まで | 1.97 から 2.02 s |
| type 3 から type 3 まで (4 回送信) | 0.48 から 0.54 s |
| 最初の type 3 から LDN の離脱まで | 2.03 から 2.08 s |
| `D` から LDN の離脱まで | 4.02 から 4.08 s |

    type 3, 22 bytes:  03 | random u32 | constant id (8) | variable id (2) | kind 0 | IPv4 | port
    type 4, 15 bytes:  04 | random u32 | the request's constant id and variable id

オフセットは GBA アプリの `main` のイメージ先頭からのものです。`LeaveMeshJob` は、500 ms の期限で要求を
`0xcacf4` に送り、`WaitLeaveResponse` (`0xcaf38`) は、ジョブの応答フラグ `+0x99` で終わるか、
期限ごとにカウントが 2 以下の間は再送します (`0xcb06c`..
`0xcb08c`)。4 回送ると、ジョブが完了して局がネットワークから離れます。セッションの
ディスパッチャーの type-4 の case (`0xba028`、テーブル `0x180fc9`) は、5 にある constant id と 13 にある variable id が
局自身のものである 15 バイトのメッセージに対して、そのフラグを立てます。ホストの type-3 ハンドラー (`0xbf2d4`) は、
22 または 34 バイトを受け取り (kind 1 は 18 バイトのアドレスを持つ、`0xbf3ac`)、15 バイトで応答します。その中身は type 4、新しいランダムな語、要求のバイト 5 から 14 です (`0xbf454`..`0xbf4e4`)。
ホストは、すべての type 3 に対して
その形式で、ユニキャストで応答します (`pokeldn.ldn.host_pia.HostPeerProtocol`)。ヘッダーは
`(console variable, 0x00C6)` で、その本体へのほかのユニキャストパケットのカウンタに番号を振ります。
市販版 FireRed は、同じ局から送られた最大 6203 の番号のユニキャストパケットの後に、番号 560 から 573 の 4 つの type 4 を無視しました
(4 つの type 3 はすべて送られ、最初の 2.05 s 後に離脱)。ユニキャストのカウンタに番号を振ったところ、最初の type 4 は受け付けられました
(type 3 は 1 つ、その 0.07 s 後に LDN の離脱)。本体自身のパケットは、宛先ごとに 1 つのカウンタを持ちます
(ホストの variable と variable 1 へ)。

最初の type 3 の前の一時停止は、GBA アプリのネットワークマネージャーにある、120 ティック固定のカウントダウンです。
ホストには依存しません。マネージャーは 1 秒に 60 ティックを数えます (そのタイムアウトは、ティックカウンタ `+0x2774` を
秒数 × 60 `0x4ff64` と比べます)。子機の `TryDisconnectRfu` は `swi 0x44` を、`rfu_REQ_disconnect` が
`D` を送る前に発行します [link_rfu_2.c:1446-1455, 983-985]。`LinkRfu_Shutdown` も同じものを発行します
[link_rfu_2.c:625]。その `0x58b0c` のハンドラーは、切断要求 1 を、マネージャーのスロット `0x68`
(`0x53d98` -> `0x4f0ec`、フィールド `+0x730`) を通して投稿します。毎ティックの更新 `0x4ee34` は、切断要求 1 を
カウントダウン `+0x734 = 0x78` (`0x4ef00`) に変え、ティックごとに 1 ずつ減らします。0 になると、マネージャーのスロット `0x78`
(`0x4ef60`) を呼びます (サブクラス `0x53328` は、`0x4f0b4` を通じて状態を 1 に設定します)。次の更新 (`0x5392c` -> `0x536f0`) は
`Session::LeaveAsync` (`0xb1060`、`0x537e0` にある) を呼び、
離脱ジョブの最初のステップが type 3 を送ります ([pia.md](pia.md)、Leaving a session)。

ネットワークから受けたものは、カウントダウンを短くしません。切断要求 2 または 3 (`swi 0x42`、または PID がゼロの `swi 0x43`)
が来たとき、または Pia のエラーかタイムアウトが切断理由 `+0x277c` を設定したときだけ、早く終わります (`0x4ee64` で最初に確認され、
`0x5110c`、`0x52470`、`0x53440` から `0x4f020` を通じて設定されます)。
カウントダウンのコードには役割の確認がありません。

## 802.11 の挙動

- 局としての本体: 省電力なし (PM ビットは常に 0)、データフレームの前に毎回 RTS を送ります。離脱するときは、
  その前にプローブもヌルフレームもなしで、認証解除を 1 回送ります。
- Switch のホストは 11M でビーコンを送り、48 から 54M でデータを送ります。

## Pia ヘッダーのノンスは、本体が強制するカウンタ

本体はチャンネルごとに、最後に受け入れた 8 バイトのヘッダーノンス (ビッグエンディアン) を保持し、
それより厳密に大きくないデータグラムを捨てます。そのためランダムなノンスは、約半分の確率でしか通りません。

    random nonce:   ~1.1 duplicate outgoing reliable deliveries per unique frame, ~0.3 inbound
    counter nonce:  0.34 outgoing, 0.00 inbound

`Sim._next_nonce` とホストの両方が数えます。

## キャリーフォワード

信頼性のあるフレームは、次の 4 つのデータグラムで繰り返され、元のものから約 17 ms 後に確認応答されます。
空中の損失は 1 から 2% で、バースト状に起きます (本体自身の再送率はフレームあたり 0.01 から 0.02)。Pia は順序どおりに配送し、
穴があると、再送されるまで後続のすべてのスロットを保持します。再送されると、それらをまとめて配送し、本体の
8 段の RFU 受信キューを溢れさせます。キャリーが 0 のとき、失われた親機のスロット 1 つと
その 68 ms の再送で、10 スロットが一度に本体へ入り、300 ms 後に切断されました。
ホストはシーケンス番号で重複を除きます。

## 本体の確認応答の遅れは 512 ms のメトロノーム

本体の累積 ack は CTRL フレームのペイロード (`parse_bulk_ack`) にあります。信頼性のあるヘッダーの
`ack` フィールドは、送信者自身の最も古い保留中のシーケンスです。

本体は一度に 50 から 70 ms 止まります。ベースラインの 15 ms に対して受信が 35 から 70 ms 止まり、
自分の最後のフレームを再送してから、累積 ack が数フレーム分まとめて進みます:

    26.553 in   D1849 D1850 ACK next=919
    26.587 OUT  D919 D920 ACK next=1851
    26.617 in   D1849* D1850*        (retransmits, no ack)
    26.653 in   ACK next=924         (catches up five frames at once)

357 行目のデータは、両方のカートリッジと 3 つの活動にわたる 42 の間隔で、どの周期も 16 ms (1 フレーム) 以内で、512 ms スロットの整数個に
一致しました。観測された数は 16 と 17 スロットで、1 回だけ 18 と、ティックが飛んだ場合の 33 がありました。
グリッドは LDN の参加に位相がロックされています。103 s の間の 12 回の停止は、18
ms の位相の範囲に収まります。アイドル中のチャットリンクでは、保留中の数が 5 に達しても害はありません。ふしぎなおくりものや
交換の負荷中には、停止が送信ウィンドウ全体に当たります。ROM には 512 ms のティックがないため、タイマーは
エミュレーターか Pia/LDN 層に属します。その正体は不明です。

## ident-25 の停止: 穴と上限のないバックログ

本体は最後の配送スクリプトブロック (ident 25、`MG_LINKID_RAM_SCRIPT`) の後にアイドルになることがあり、
ident 20 (READY_END) を送らずに離脱します。親機のブロックは決して反映されないため、失われたフラグメントは
キャプチャでは見えません。

上限のないバックログの後ろで穴が閉じると、本体は保持していたフレームをすべて一度にゲームへ渡します。
その 8 段の RFU キューが ident-25 のフラグメントを落とし、リンクは生きたまま "Transmission..." に留まります。計測した 1 回の停止では、連続する 2 フレームが失われ、
累積 ack が 1.75 s 止まりました。その間ホストは、すべてのデータグラムで再送し (約 100 コピー)、データグラムごとに新しい 5 フレームを加えました。
1 つのデータグラムで 97 フレームがゲームに届き、本体は "Transmission..." に
150 s 留まりました。
ack の通常の遅れは、ack の 99% で 0 から 1 フレームです。

`HostSession` は、`HOST_OUTSTANDING_MAX` (6) フレームが未確認の間、新しいフレームを保持し、
欠けた部分を再送し続けます。ack が追いつくと再開するため、閉じた穴から一度に解放されるのは最大 6 フレームです。
`--ram-script-block-repeat 3` は ident-25 のブロックを 3 回送ります。5 件のセッションのうち 5 件が配送されました。`--block-repeat 1` のときは、
計測した LeafGreen の 1 セッションが ident 25 で止まりました。フラグメントを落とすのが空中の損失か、本体の RFU からゲームへの受け渡しかは
不明です。

## カードの後の送信フェーズでの切断

250 ms から 3 s の送信停止の後に ack がなくなり、未確認のフレームをティックごとにすべて再送した結果、
カードの後のセッションが切断されました。ホストの UDP 出力は 0.25 s あたり最大 52 から 519 データグラムで、完了したセッションでは
23 から 36 データグラムでした。ホストのガードは、すべてデフォルトで有効です。

- `HOST_RTX_LIMIT` は、1 回の VBlank あたりの Reliable の再送数を制限します。
- `FRLG_ECHO_MAX` は、子機のブロックスロットの FIFO エコーを制限します。
- 送信ソケットはノンブロッキングで、`FRLG_QUIET_GATE_MS` は、再送とキャリーフォワードを保持します。
  本体が黙っている間 (カードを受け入れてフラッシュへ保存した後、約 0.5 s。そこでブロッキング送信を行うと
  ホストが 6 から 11 s 固まりました)。

## ホストとエアの間のソケットで保持されるデータグラム

空中に出る前に 0.1 から 1.1 s 保持されたホストのデータグラム (計測した 1 回の対戦では、200 ms で 71 フレームのバーストとして解放されました) は、
ゲームがリンク切断の経路に入り、フランス語の "erreur de connexion, rapprochez-vous" の表示を出すのに十分な長さです
[link_rfu_2.c:2312 → CB2_PrintErrorMessage, link.c:1521]。ESP32 の基板では、4 回の FireRed の交換にわたる、
ソケットから確認応答までの最長の待ち時間は 265 ms で、その時間は基板の
送信キュー内で費やされました ([ESP32 radio](hardware_esp32.md)、TX_DONE)。

# ユニオンルーム

## 掲載される: 広告される活動バイト

検索する本体は、広告している活動が検索するリンクグループの受け入れリストに含まれる場合にだけ候補を残します
[`IsPartnerActivityAcceptable`, union_room.c:1590; `sAcceptedActivityIds`、
src/data/union_room.h:398-456]。ほとんどのリストは 1 つの ID を持ちます。ユニオンルームにいる本体は
`ACTIVITY_SEARCH` (12) を広告し、`LINK_GROUP_UNION_ROOM_INIT` を検索します。リストは `{ACTIVITY_SEARCH, 0xFF}` です
[src/data/union_room.h:419]。そのため、交換ホスト (`ACTIVITY_TRADE`、4) やふしぎなおくりものホスト
(`ACTIVITY_WONDER_CARD`、21) は、そのバイトだけで除外されます。部屋の中では、検索は
`LINK_GROUP_UNION_ROOM_RESUME` [union_room.c:2664] で行われます: `IN_UNION_ROOM | activity`、`IN_UNION_ROOM = 1 << 6`
[include/constants/union_room.h:49]。

レコードのバイト 16 は活動です。21 は Wonder Cards 画面でホストを一覧に出し、22 は Wonder News
画面で、本体がそのホストに参加してセッションを完了する場所を示します。

## 接続: IN_UNION_ROOM だけ、完全一致で

`IsPartnerActivityIncompatible` [src/link_rfu_2.c:2925] は次を検査します

    else if (partner->activity != IN_UNION_ROOM)   // [link_rfu_2.c:2933]
        return TRUE;

完全一致として比較します。`IN_UNION_ROOM | ACTIVITY_TRADE` (0x44) はアバターを出現させますが、話しかけると
「Communication avec POKELDN」と「le DRESSEUR est occupé」が表示され、空中にはパケットが出ないまま、接続が
`Task_TryConnectToUnionRoomParent` [link_rfu_2.c:2963] の中で拒否されます。交換の意図は
`sPlayerCurrActivity` にあり、リンクの確立後に交渉されます。素の `IN_UNION_ROOM` (0x40) は接続します。アバターは
ビーコンをリアルタイムに追い、ホストが再起動すると、外に出てから中へ戻ります。

## 親機の NI はなし、5 フレームの規則

ユニオンルームの子機は、親機の参加ステータス NI を期待しません。その名前の NI が成功すると
[AgbRfu_LinkManager.c:1203]、`LinkManagerCB_UnionRoom` [link_rfu_2.c:2526] は TYPE_UNI の受信バッファを設定し、TYPE_NI は設定しません。`Task_UnionRoomListen`
[link_rfu_2.c:533] は `Task_PlayerExchange` を MODE_CHILD として開始します。TYPE_NI のバッファを追加するのは
トレードセンターのコールバック [link_rfu_2.c:2364] だけです。

NI の本体は致命的です。`rfu_STC_NI_receive` はゲームのバッファなしで LCOM_NI_START を受け取りますが
[librfu_rfu.c:2202]、本体はバッファを必要とします。`rfu_STC_NI_initSlot_asRecvDataEntity` は ERR_RECV_BUFF_OVER で失敗し
[librfu_rfu.c:2300]、`recvErrorFlag` → REQ エラー → `LMAN_MSG_REQ_API_ERROR` →
`RfuSetErrorParams` → 「erreur de connexion, rapprochez-vous」になります [link_rfu_2.c:2585]。
`--union-room` で有効になる `RFULeader(skip_parent_ni=True)` は、これを飛ばします。

本体は、応答しないまま残した 5 番目の親機フレームで切断します (3 件のキャプチャが一致します)。

    child NULL 28.361  host NI ts=8..12, K only   D 28.496   (NI_STARTs ts=6,7 were mirrored)
    child NULL 34.069  host NI ts=8..12, K only   D 34.199
    child NULL 36.527  host UNI ts=6..10, K only  D 36.628

子機の NULL から: 135/130/101 ms。3 番目は、欠けていた 2 つのミラーのぶんだけ 2 フレーム早く、
`maxMFrame = 4` と一致します [link_rfu_2.c:128]。完了したトレードセンターのキャプチャでは、最大 2 でした。

`--union-room-keepalive N` (120 が有効) は、UNI の前に最初の親機 NI_START を N VBlank のあいだ再提示し、
そのあいだ本体は常に送る確認応答を持ちます。その後は約 8 秒待ちます。
`Task_UnionRoomListen` は `rfu_UNI_setSendData` を毎フレーム再試行し、
NI_START が保留中の間は ERR_SUBFRAME_SIZE で失敗します。受信制御が
子機の 16 バイトの LL フレームのうち 2 バイトを使い [librfu_rfu.c:2262]、UNI のサブフレームには 16 バイトが必要です [librfu_rfu.c:1449]。保留中の受信は
`NI_failCounter_limit` によってのみ解放され、最後の NI_START から 480 フレーム後です
[link_rfu_2.c:139, AgbRfu_LinkManager.c:1328] (計測値: 最初の UNI フレームは、ホストの最後のフレームから 482 フレーム後)。
それより早く解放されることはありません。

## 接続後の本体の動作

[src/union_room.c:2858-2879]

    if (gReceivedRemoteLinkPlayers) {
        CreateTrainerCardInBuffer(gBlockSendBuffer, TRUE);
        CreateTask(Task_ExchangeCards, 5);
        uroom->state = UR_STATE_COMMUNICATING_WAIT_FOR_DATA;
    }
    ... then, if sPlayerCurrActivity == (ACTIVITY_TRADE | IN_UNION_ROOM),
        UR_STATE_SEND_TRADE_REQUST

表示は "POKELDN: oh bonjour \<name>, vous désirez quelque chose ?" で、Salut / Combat /
Tchat / Retour が並びます。選択ごとに 1 つの `SEND_PACKET` を送り、待機します
[UR_STATE_HANDLE_ACTIVITY_REQUEST, union_room.c:3151]。

| 本体の送信 | 活動 | ホストの応答 |
|---|---|---|
| 0x48 | CARD (Salut) | 0x51 ACCEPT |
| 0x44 | TRADE (交換ボード) | 0x51 ACCEPT |
| 0x41 | BATTLE (Combat) | 0x51 ACCEPT |
| 0x45 | CHAT (Tchat) | 0x51 ACCEPT |
| 0x40 | EXIT (Retour) | 本体の終了として扱う |

各活動の後、両側は `SetLinkStandbyCallback` [union_room.c:2995, :3012] を実行し、本体は
プロンプトに戻ります。Switch 版のビルドでは、受け入れる側も
`svc_CommsAllowedByParentalControls()` [union_room.c:3159, :3037, REVISION >= 0xA] で制御されます。そのため本体自身のペアレンタルコントロールが、
その要求を DECLINE に変えることがあります。Salut ではホストのトレーナーカードを表示し、同じ表示を
繰り返します。Retour は 0x40 を送り、READY_CLOSE_LINK を送ります。応答があると本体は通常どおり切断し、
エラーなしで部屋に残ります。

## 交換ボード

ボードは、広告に `tradeSpecies`、`tradeType`、`tradeLevel` を含むパートナーを一覧にします
[union_room.c:3400]。レコードのバイト 18 (`type << 2`)、19 (`gameClear | level << 1`、ラッパーがそう詰めるとおり、
[広告レコード](frlg_rom.md#the-advertisement-record))、22:24 (リトルエンディアンの
`RfuGameData` の `tradeSpecies:10` [include/link_rfu.h:107]) を使います。種族 277 (Treecko。下位バイトだけでは 21 で
Spearow) は "POKELDN / NORMAL / ARCKO / 26" として一覧に出ます。これはバイト 23 を計測したものです。

交換ボードの交換は、リンクごとに 1 回の交換です。`Task_StartUnionRoomTrade` は、`CB2_LinkTrade` の前に `gMain.savedCallback =
CB2_ReturnToField` を設定し [union_room.c:1744]、`CB2_SaveAndEndTrade` は、保存されたコールバックがトレードセンターの
`CB2_StartCreateTradeMenu` のときだけリンクを維持します [trade_scene.c:2722-2725]。それ以外は `SetCloseLinkCallback` で終わります。手順は次のとおりです。本体の
Pokemon ブロック (カウント 9)、ホストのもの、そのメールブロック (カウント 19)、ホストのもの、アニメーション、本体の
READY_FINISH、ホストの CONFIRM_FINISH、保存の待ち合わせ、両方向の READY_CLOSE_LINK、
本体の通常の切断で部屋に戻ります。ボードの次の交換は新しい接続です。トレードセンターでは、代わりに
交換メニューに戻ります [trade.c:2094-2113]。
パーティ交換、メニュー、部屋の経路はありません。

## チャット

チャットは `SendBlock` を使い、`Rfu_SendPacket` を通りません。すべてのメンバーが、要求なしで `SendBlock(0, sendMessageBuffer,
0x28)` を呼び出し、`BLOCK_REQ` はありません [`ChatEntryRoutine_Join`, union_room_chat.c:429;
`ChatEntryRoutine_SendMessage`, :823]。0x28 バイトのブロックのカウントは 4 です。

    [0]      command: 0 NULL, 1 CHAT, 2 JOIN, 3 LEAVE, 4 DROP, 5 DISBAND
    [1..8]   player name, PLAYER_NAME_LENGTH + 1 bytes, EOS-terminated
    [9]      multiplayer id            (JOIN / LEAVE / DROP / DISBAND)
    [9..39]  message text, EOS-terminated (CHAT)

[`PrepareSendBuffer_*`, union_room_chat.c:1256-1281; `ProcessReceivedChatMessage`, :1283]。0x45 の ACCEPT では、
本体は `Task_StartActivity` のチャット分岐に入ります [union_room.c:1938]。この分岐は新しい接続だけを止め (`rfu_LMAN_stopManager(FALSE)`)、
リンクは維持します。両方のメンバーは、入るときに JOIN を待たずに送ります。
ホストは本体の JOIN に反応します。

1 行は 15 エントリです (`MESSAGE_BUFFER_NCHAR` [union_room_chat.c:21]。31 バイトのバッファは
0xF9 のペアを許します [string_util.c:560])。受信側は届いたものを [:1308]、切り詰めずに 1 行へコピーするため、
16 番目以降のエントリは画面の外にはみ出します。ホストは、長すぎる `--chat-message` や `--chat-file` の行を
起動時に拒否します (`uroom_chat.entry_count`)。

リーダーは LEAVE で閉じなければなりません。離脱する側は、親機がリンクを落とすのを待ちます
[`ChatEntryRoutine_AskQuitChatting` の case 2/4/5、union_room_chat.c:596-660]。さもないと "quit the chat?" の表示で止まります。
ホストは LEAVE の 0.1 s 後に DROP を送り、リンクを閉じる握手を行います。`--chat-file` に
追記された行は、書き込みの約 1.7 s 後に本体へ届きます。

## リンクバトル (UR_BATTLE 0x41)

`HasAtLeastTwoMonsOfLevel30OrLower` [union_room.c:4565] は、レベル 30 以下の
タマゴでないポケモンを手持ちに 2 体要求します。本体が提示するとき [union_room.c:2923] と、受け入れるとき [union_room.c:3176、
DECLINE を送る] に確認されます。それぞれの側は自分の `gPlayerParty` だけを判定し、拒否は画面上のメッセージになります。

両方がそれぞれ 2 体を選ぶと、各側は 1 つの 0x20 バイトのブロックを送ります。その先頭バイトは `ACTIVITY_ACCEPT | 0x40`
= 0x51 (キャンセルされた場合は 0x52)、残りはゼロです [union_room_battle.c、`CB2_UnionRoomBattle`]。それ以外は、
"refused" でリンクを閉じます。Switch の経路では、スタンバイを挟んで 2 つのリンクタスクの待ちがあります
(`#if REVISION >= 0xA` の case 50/51/52)。

`SetUpPartiesAndStartBattle` は選ばれた 2 体を残し、残りの 4 体をゼロにして、
`StartUnionRoomBattle(BATTLE_TYPE_LINK | BATTLE_TYPE_TRAINER)` [union_room.c:1811] を呼び出し、
`gLinkPlayers[0].linkType = LINKTYPE_BATTLE` (0x2211) [link.h:92] を設定します。`TryReceiveLinkBattleData` はこれを
完全一致で判定します [battle_controllers.c:520]。その後 [`CB2_HandleStartBattle`, battle_main.c:934] の処理に入ります:

    state 1  SendBlock struct LinkBattlerHeader {versionSignatureLo, versionSignatureHi,
             vsScreenHealthFlagsLo, vsScreenHealthFlagsHi, struct BattleEnigmaBerry}
    state 3  SendBlock gPlayerParty[0..1]   200 bytes      state 4  recv -> gEnemyParty
    state 7  SendBlock gPlayerParty[2..3]   200 bytes      state 8  recv
    state 11 SendBlock gPlayerParty[4..5]   200 bytes      state 12 recv
    state 15 InitBattleControllers

パーティ交換は、交換の 3 × 200 バイトの転送と同じです (`mon.party_blocks`。`Rfu_InitBlockSend` は
252 バイトまで許します)。

### マスターの選出

リンク対戦の 1 対 1 では、`BATTLE_TYPE_IS_MASTER` の側だけが `gBattleMainFunc =
BeginBattleIntro` を設定し、もう一方は `BeginBattleIntroDummy` のままです [`InitLinkBtlControllers`、
battle_controllers.c:141; `SetUpBattleVars`、:45]。マスターでない側は、ターンの解決、ダメージ、乱数を実行しません。
BUFFER_A のコントローラーコマンドを受け取って表示し、応答します。そのため、マスターでないホストは
バトルコントローラーとして働きます。

`LinkBattleComputeBattleTypeFlags` [battle_main.c:886] は、マルチプレイヤー ID 1 の本体について、
`gBlockRecvBuffer[0][0] == 0x100` か、両方の署名が等しければ、プレイヤー 0 がマスターになります。そうでなければ、
最も高いバージョンを持つ最も小さいインデックスになります。0x100 以外で 0x201 未満の署名は、本体をマスターにします。
ホストは 0x200 を送ります。

### リンクバッファのプロトコル

すべてのコントローラーコマンドは、8 バイトのヘッダーを持つ 1 つの SendBlock です [battle_controllers.c:401-435]。ヘッダーは次の順です。
`LINK_BUFF_BUFFER_ID, ACTIVE_BATTLER, ATTACKER, TARGET, SIZE_LO, SIZE_HI, ABSENT_BATTLER_FLAGS,
EFFECT_BATTLER`、その後にペイロードが続きます。`alignedSize = size - size % 4 + 4` として格納されます (4 バイトのペイロードは
8 バイトになります)。`bufferId` 0 = BUFFER_A (コマンド)、1 = BUFFER_B (応答)、2 = 実行フラグのクリアで、その 1 バイトは
送信者のマルチプレイヤー ID です [Task_HandleCopyReceivedLinkBuffersData:566-594]。バトラー 0 はマスターのポケモン、
バトラー 1 はホストのものです。

同期規則 [battle_util.c:185-201] では、`MarkBattlerForControllerExec` がビット `28+battler` を立てます。コマンドのブロックが戻ると、
`MarkBattlerReceivedLinkData` は、各プレイヤー i ごとに `gBitTable[battler] << (i*4)` を設定し、ビット 28+battler を消します。
各プレイヤーは bufferId 2 で自分のニブルを消します。マスターは
`gBattleControllerExecFlags == 0` のときだけ進むため、すべてのコマンドは両方のバトラーについて確認応答されなければなりません。

プレイヤーバッファの 56 個のコマンド [battle_controller_player.c:110] のうち、確認応答以上を必要とするのは次のものです。

    CONTROLLER_GETMONDATA    -> EmitDataTransfer(BUFFER_B, size, data)   [player.c:1515]
    CONTROLLER_CHOOSEACTION  -> EmitTwoReturnValues(1, B_ACTION_*, 0)    [player.c:232-241]
    CONTROLLER_CHOOSEMOVE    -> EmitTwoReturnValues(1, 10, move | target << 8) [player.c:342]
    CONTROLLER_CHOOSEPOKEMON -> EmitChosenMonReturnValue(1, partyId, order)    [player.c:1316]
    CONTROLLER_OPENBAG       -> EmitOneReturnValue(1, itemId)            [player.c:1340]
    CONTROLLER_EXPUPDATE     -> EmitTwoReturnValues(1, RET_VALUE_LEVELED_UP, exp) [player.c:1051]
    CONTROLLER_ENDLINKBATTLE -> gBattleOutcome = payload[1], then ack     [player.c:2876]

`B_ACTION_USE_MOVE` は 0、`USE_ITEM` は 1、`SWITCH` は 2、`RUN` は 3 です [battle.h:34]。

最初のコマンドは `GETMONDATA` の `REQUEST_ALL_BATTLE` です [battle_main.c:2519]。応答は、`CopyPlayerMonData` が組み立てる
0x58 バイトの `struct BattlePokemon` [pokemon.h:170] です [player.c:1519]。
`statStages`、`ability`、`type1`、`type2`、`status2`、`unknown` は再計算されるため、ゼロで足ります。バトラー 0 の BUFFER_B の応答は省略してかまいません。本体は自分の GETMONDATA に対して、
`gEnemyParty` から応答します [link_opponent.c:444]。

リンクのバトラーは、[battle_main.c:3239] で先頭のターン順 [:3548-3560] で動くことがあるため、最初の
`CHOOSEACTION` に `B_ACTION_RUN` で応答すると、経路全体を通り、バトルが終わります。

### decomp に見えない 2 つの規則

ブロックのサイズは経路を決めません。すべての確認応答と短いコマンド (最初の `GETMONDATA` を含む) は
16 バイトの 2 フラグメントのレコードです。そのため、バトル中は状態がブロックの経路を決め、
サイズでは決めません。

確認応答は、それが確認するブロックのエコーを決して追い越してはなりません。本体は、自分のブロックが戻ったときにだけ実行フラグのビットを立て
[`MarkBattlerReceivedLinkData`, battle_util.c:193]、ホストの確認応答が
それを消します [battle_controllers.c:585]。2 フラグメントの確認応答は、7 フラグメントのエコーを追い越せます。

    104.268 console bufferA battler 0 PRINTSTRING (72 B)
    104.366 host    ack battler 0                     <-- host's ack first
    104.383 echo    bufferA battler 0 PRINTSTRING

その後エコーがビットを立てると、本体は `gBattleControllerExecFlags == 0` で永久に待ちました。この条件は
`Cmd_waitmessage` [battle_script_commands.c:2041] を制御するもので、リンクは生きたまま、バトル終了のメッセージで固まりました。
`HostTradeEngine._echo_owed` は、子機のコマンドがミラーを待っている間、新しいブロックを保持します。

確認応答は、エコーのすべてのフラグメントも待ちます。`rfu_leader.echo_blocks` は、エコーされた各
`SEND_BLOCK_INIT` を、送出したフラグメント番号の集合とともに保持し、確認応答は 0 から count-1 を待ちます。空の
エコーキューは 24 倍遅すぎます。エコーを数えること、または最後のフラグメントだけを見ることは、
再送されたフラグメントのエコーが確認応答と同じフレームに乗ることを許します。`gBattleControllerExecFlags` で
待っている本体は、後続のメッセージでは解放されません。ホストを終了すると、エラー画面を通して部屋に戻ります。

### テンポは RFU の VBlank の予算によって決まる

リンクバトルの 1 ステップは、両方向とも、ホストに対しておよそ 1 秒かかります。データグラムの往復は
中央値 5.8 ms です (8204 件の応答に対する p99 は 16.6 ms。本体の中央値は 8.1 ms)。連続する `bufferA`
コマンドは中央値 800 ms 間隔です。本体のブロックは 19 ms でエコーされ、ホストの応答ブロックは
355 ms 後に続きます。`HostSession.tick` は、呼び出しごとに 1 つの RFU スロットを、`HOST_VBLANK_SECONDS =
1/59.727` = 16.74 ms で出します (17 ms × 5035、16 ms × 413 を計測)。コマンドブロックはおよそ 20 フレーム、
~340 ms にわたり、1 ステップはそれに戻りの区間を加えたものです。1 回の VBlank に 1 スロットより多く出すのは、
実機のリンクがすることではありません。

## 実機の 2 台が空中に出すもの

実機の 2 台の間の交換を、受動的にキャプチャしたものです (ファイアレッドが第 3 の NPC を通してホストし、
LeafGreen が参加)。モニターのフレーム数は下限で、802.11 のシーケンスカウンタが、モニターの取りこぼしたフレームを
補います。シーケンスで補正したフレームレート (同じファイアレッドの本体)。

| 局 | ホストとの通信 | 実機との通信 |
|---|---|---|
| ファイアレッドの本体 | 161.8/s | 25.5/s |
| 相手側 | 58.9/s、ホストは 1 回の VBlank に 1 スロット | 6.1/s、LeafGreen |

25.5/s という数値は、最も弱いキャプチャに基づいています。

Direct Corner 経由でホストしているファイアレッドの本体は、Switch のプロファイル名を、`application_data` のオフセット
0x11 に平文の ASCII で載せます。ゲーム内のトレーナー名は、広告には含まれません。

## ホストのティックレートと本体の出力レート

`--tick-hz` はホストの RFU スロットを毎秒いくつ出すかを設定します (既定は VBlank ごとに 1 つ)。各レートで、
同じ本体との 1 回の交換を行いました。

| | 59.727 Hz (既定) | 20 Hz |
|---|---|---|
| ホストが送るデータグラム | 79.3/s | 39.3/s |
| 本体が受け取るデータグラム | 61.8/s | 51.8/s |
| 交換の時間 | 92 s | 197 s |
| 本体のデータグラム (セッション全体) | 5663 | 10218 |

20 Hz では、本体の受信レートが 6 分の 1 減り、交換は 92 s ではなく 197 s かかり、本体は
80% 多くのデータグラムを送りました。既定は VBlank ごとに 1 スロットです。

# ケーブルクラブのコロシアム

`frlg_trade_host.py --colosseum` は、ポケモンセンター 2F → 第 3 の NPC (club sans fil) → コロシアム →
シングルバトル → JOIN をホストします。`CB2_ReturnFromCableClubBattle` だけが Wonder Card の `battlesWon` を増やします
[src/cable_club.c:792]。ユニオンルームのバトルは `CB2_ReturnToField` を通って戻ります。

`Task_StartActivity` は `ACTIVITY_BATTLE_SINGLE` を、次の点を除いて `ACTIVITY_TRADE` と同じように扱います [union_room.c:1903]。
マップ (`MAP_TRADE_CENTER` (5, 8) ではなく `MAP_BATTLE_COLOSSEUM_2P` (6, 8))、および
`HealPlayerParty()` の呼び出しです。そのため、トレーナーカードの交換と `--card-flag-id` はそのまま動きます。変わるのは
次の 4 点です。

1. 活動バイト: `LINK_GROUP_SINGLE_BATTLE` は `{ACTIVITY_BATTLE_SINGLE, 0xFF}` を受け入れます
   [src/data/union_room.h:398]。`build_colosseum_app_data` は、そのバイトだけを変えます。
2. `BattleColosseum_2P_EventScript_PlayerSpot0/1` には、パーティの確認がありません [data/scripts/cable_club.inc:576]
   (4P の `ChooseHalfPartyForBattle` には選択のステップがあります)。座席の握手は変わりません。
3. `Task_StartWirelessCableClubBattle` の case 2 は `SendBlock(0, &gLocalLinkPlayer,
   sizeof(gLocalLinkPlayer))` を送ります [cable_club.c:701]。これは、素の 28 バイトの `struct LinkPlayer` です。入場ブロックの GameFreak のマジックは含まれず、両側から要求なしに送られます。本体は
   すべてのレコードが届くまで case 3 で待機します。
   その後 20 フレーム、`IsLinkTaskFinished`、`SetLinkStandbyCallback`、もう 1 回の待ち
   (case 4-6、`REVISION >= 0xA`)、`CB2_InitBattle` と続きます。
4. パーティ全体で戦います。`party_blocks` は、現在のパーティをそのまま取ります [union_room_battle.c:47]。

`CB2_InitBattle` 以降は、署名 0x200 を含むユニオンルームのバトルになります。0x20 バイトの
0x51 選択ブロックはありません (`CB2_UnionRoomBattle` だけです)。

座席には READY のキーだけが必要です。移動を送らなければ、本体は
`Task_StartWirelessCableClubBattle` に進みます (`GetCableClubPartnersReady` はリンク状態だけを読みます
[overworld.c:2989])。座席後のスタンバイラウンドはありません。それを待つホストは行き詰まります。一方、case 3 で待機している本体は、
`SEND_BLOCK_INIT` の count 3 を送ります。その内容は
`04 40 00 80 65 df | bb c8 ff 00 11 00 | 01 00 03 00 00 00` です (バージョン 0x4004 のファイアレッド、`lp_field_2`
0x8000、そのトレーナー ID)。そのブロックが届くと、入場が終わります。

ホストも、バトルの後に退出のキーを送る義務があります。ドアは、すべてのプレイヤーが
`PLAYER_LINK_STATE_EXITING_ROOM` になるまで待ちます [overworld.c:2977]。さもないと本体は *"conduire a la sortie
de la piece, veuillez patienter"* の表示で、リンクがエラーになるまで待たされます。

## コロシアムの run が数えられるかを決めるもの

不戦敗は本体にとって勝ちになります。`HandleAction_Run` は、逃げなかった側に
`B_OUTCOME_WON` を設定し、`B_OUTCOME_LINK_BATTLE_RAN` (1 << 7) と OR します [battle_main.c:4300]。その値は
`CB2_ReturnFromCableClubBattle` が切り替える前に、`HandleEndTurn_BattleWon` が消します [battle_main.c:3734]。記録される ID は
`gLinkPlayers[GetMultiplayerId() ^ 1].trainerId` [cable_club.c:794] で、28 バイトのレコードから取られます。
統計ごとに 5 件を覚えるため、1 件ずつ数えられます [mystery_gift.c:630]。3 つの異なる `--id` の値に対する 3 回の不戦敗で、
`battlesWon` が 3 になります。Battle Count Card は
これに対して POTION を与えます。1 回の後は次のとおりです。

    save 0x3434:  0100 0000 0000 2300    battlesWon 1, lost 0, trades 0, icon 35 CARD_TYPE_LINK_STAT
    game data:    "1 battles won"

# 2 つのホスト側の制約

閉じる経路は穴のガードに依存します。`done` は切断の経路から来ます。その経路は
`disconnect_requested` で制御され、これは `activity.tick()` の中の `_tick_close_link` が設定します。
`HostSession.tick` は、穴のガードが保持している間はこれを飛ばします。確認応答を止めた本体はガードを固定するため、閉じるための
タイマーが止まります。確認された退出の後、本体が LDN から離れると、実行が止まります。

全ゼロの `easyChatProfile` は、トレーナーカードに "??? ???" と表示されます。ワード 0 は `EC_GROUP_POKEMON_2` の
インデックス 0 (`SPECIES_NONE`) で、`IsECWordInvalid` が拒否し、`CopyEasyChatWord` が
`gText_ThreeQuestionMarks` に置き換えます [easy_chat.c:166-171]。ワードは `(group & 0x7F) << 9 | (index & 0x1FF)` です
[easy_chat.h:1089]。カードごとに 4 つあります [trainer_card.h:28]。短いフレーズは `EC_WORD_UNDEFINED`
(0xFFFF) で埋めます。これは何も表示しません。
