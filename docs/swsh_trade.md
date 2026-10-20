---
title: リンク交換
parent: Sword and Shield
nav_order: 3
---

# 市販のソード・シールドでの交換

ソード・シールドのリンク交換の、スナップショット交換からセーブまでです。フレーミング、コンテンツ登録
および PK8 は[同期フレームワーク](swsh_protocol.md)にあります。

## 交換の流れ

    1  the console broadcasts its 3456-byte snapshot on protocol 0x84, port 0
    2  the client acknowledges the fragments (0x21) and the transfer (0x19 / 0x28)
    3  the client sends its own snapshot on 0x84 port 1 and reports it complete
    4  the trade screen opens
    5  the console sends the 40030 RPC pair on 0x7C port 1, several times a second
    6  the client answers the pair and sends `imReady` on 20030
    7  the console offers a Pokemon on 20030; the client offers one on 10050
    8  the player accepts (box command 4)
    9  the confirmation ladder runs on content 40
    10 the console writes its save

ホストになっている本体は、参加側が port 0 で送ったスナップショットには応答しません。参加側のスナップショットは port 1 で送ります。
0x84 への応答があるまで、本体は他のデータを一切送らず、それを無期限に再送し続けます。
`pokeldn/ldn/broadcast4.py` が4種類すべてを実装しています。

セーブの後の内容は[交換をホストする](#hosting-a-trade)と
[ホスト移行](#host-migration)にあります。

## 交換の RPC

ping、ブロックメッセージ、オファーは port 0、40030 のペアは port 1 を使います。違うポートへの応答は
読み取られません。

    id 40030 = 40000 + 30
      1  offset      30
      2  base        10000, and 20000 in the other member
      3  station id  the sender's, equal to the host_constant of the seat record
      4  clock       a counter advancing between messages
      5  bytes(4)    00000000 for the 10000 member, 000018fc for the 20000 one

`000018fc` は番兵の値 `0xfc18` です ([ステップ本体](#the-step-body))。本体のメッセージは
解析したフィールドからバイト単位で再構築できます (`pokeldn/swsh/trade.py`)。`3e4e000012020801` は 20030 で、
`imReady{isReady:true}` を運びます。参加側のクライアントがこれを送り、ホストとして動く本体はそのあと
確認のために提示されたポケモンを表示します。ホストになっている本体がこれを送るのは確認されていません。

- 応答は各 1 回だけ送ります (`--answer-once`)。タイマーで応答を作り直すと、オファーが再送されてしまいます。
- リーダーは例外を送出してはなりません。信頼性ウィンドウ上のメッシュメッセージ (`44 00 01`、
  [ホスト移行](#host-migration)) は交換のメッセージではなく、例外を送出するリーダーは受信タスクと交換を終了させます
  (`la communication avec l'autre joueur a été interrompue`)。`trade.py` のリーダーは
  `None` を返します。

## ホスト移行

セッションを離れるホスト役のソードは、プロトコル 0x18 の port 1 でホスト移行を要求します
(`nn::pia::mesh::LeaveWithHostMigrationJob`):

    0f 00 00 03 00 01 00 01   44 00 01
    ^ version 4's reliable header, sequence 1     ^ the mesh message

`44` は MIGRATION_START で、`[0x44, host index 0, new host index 1]` です。本体は次のホストとなるクライアントを指定します
(`nn::pia::mesh::LeaveWithHostMigrationJob`)。指定されたクライアントは、メッシュ port 1 の信頼性ウィンドウで
`MIGRATION_FINISH` を送信する必要があります
([Pia 層](pia.md#host-migration))。

`--answer-migration --update-mesh` を指定すると、クライアントは finish を送信し、メッシュの更新を
新しいホストインデックスの下で公開します。移行の後もセッションを維持できるかは未検証です。

市販のソードでホストしている場合、MIGRATION_START は次の 2 つの場合に 1 回だけ送られます。

| タイミング | 条件 |
|---|---|
| ボックスコマンド 3 の 0.8 秒後 | 交換が完了した後 |
| プレイヤーが承諾した数秒後。本体が `2-ALZAA-0016` で接続を閉じる前 | 確認の手順に入る前に交換が失敗し、参加側が確認コマンドを送らなかった場合 |

`bin/swsh_host.py --migrate` は、ホストした交換をボックスコマンド 3 と MIGRATION_START で終了させます。エミュレートした
シールドの参加側は、セーブのあとに通信が中断されたことを示すメッセージを表示します。既定では
ホストがセッションを維持します。

## 交換をホストする

`bin/swsh_host.py` はホストを行い、`pokeldn/swsh/host_trade.py` はホスト役のソードと同じように振る舞います。市販の
フランス語版ソード 1.3.2 とエミュレートしたシールド 1.3.2 がそれぞれ参加して交換します。以下の詳細は
エミュレートしたシールド 1.3.2 同士の交換から読み取ったものです。

ホストは、新しいネットワーク ID、デバイス ID、アカウント uid を使ってステーション広告を作成します。0x84 では
参加側の 3 つのスナップショット断片に応答し、そのライブのスナップショットをトレーナー情報と
`--offer-file` の PK8 で書き換えたうえで、結果を port 0 で送ります。ホストは本体がオファーした
PK8 を `--received` に書き出します。`--advert FILE` と `--snapshot FILE` は、比較のために
保存したレコードの経路を残します。

ホスト側から見たステーションのハンドシェイクは次のとおりです。

    joiner -> host   connection request, [1] a random byte per request
    host   -> joiner ack (05 000000 and the request's trailing id)
    host   -> joiner its own request, the joiner's [1] at [0x10]
    joiner -> host   ack, then its connection response
    host   -> joiner ack, then its response

応答がない場合、参加側は要求を繰り返し、ホストの要求には応答しません。[0x10] が一致しない場合は
無視されます。メッシュへの参加後、参加側は 2 秒ごとに Sync Clock (0x1C) を送り、1 回 Clone
Clock (0x77) を送ります。ホストは、要求の tick とメッシュのクロック (ミリ秒) を返し、さらに 01、
要求のバイト 1 から 9、クローンのクロック (ミリ秒) を返します。

両方向の[ping ハンドシェイク](swsh_session.md)の後、アプリケーション層は次のとおりです。

    0x84  each side's snapshot on the port of its station index, the joiner's on port 1 and the
          host's on port 0, each acked on the same port; `host_trade.py`
          builds its own from the joiner's and sends it second, a retail host sends first
    110   the joiner pings first once its trade screen is up; the host answers with its own ping
          and the reply (a host ping before that screen is acked and dropped)
    30    the host opens content 30; each side offers on 20030 and sends box command 1; each
          acceptance is box command 4
    130   the joiner pings first
    50    the host publishes its Pokemon as element 0 and relays the joiner's 10050 as element 1
    120   the host pings first
    40    the ladder: host commands as element 0, the joiner's 10040 as element 1, phases 0 to 4

## 交換のアニメーション

本体は、最後の syncCommand 40 (`3`) の後に交換のアニメーションを再生します。その間、交換のメッセージは
送られません。`bin/swsh_host.py` に参加した市販のソードは、その syncCommand の 1.2 秒後にアニメーションを
開始し、約 24 秒でプレイヤーに操作を渡しました。プレイヤーがボックス画面から戻る (ボックスコマンド 3) まで、
アプリケーションのメッセージは送りませんでした。

## 1 つのセッションでの連続した交換

1 つのセッションには、交換が次々と載ります。交換の状態 9 はイベント 7 (成功) か 8 (失敗) を送り、
`0x010ca360` に状態 0 を書き込みます。プレイヤーはボックスに戻ります。コンテンツ 30 と ping 110 は
セッションの間ずっと残ります (セッション設定 `0x010c9280` で一度だけ作られます)。コンテンツ 50 と 40 およびそれらの ping は
交換ごとに作り直されます。

| 対象 | 寿命 | コード |
|---|---|---|
| コンテンツ 30、ping 110 | セッション | `0x010c9280` -> `0x010cca10` |
| コンテンツ 50、ping 130 | 1 回の交換 | 交換の状態 1 -> `0x010d5440` -> 初期化 `0x010d4d90` (新しいコンテンツを作り、古いものは解放)、フェーズ 0 で `0x010d53fc` により要素を生成。状態 3 で `0x010d54b0` により破棄 |
| コンテンツ 40、ping 120 | 1 回の交換 | 状態 6 -> `0x010dabc0` -> 初期化 `0x010da470`。状態 8 で `0x010dac90` により解放 |
| コンテンツの SyncPing | その要素 | 生成処理 `0x006d44e0` は、id オフセット + 0x50 (`0x006d46d0`) で新しい SyncPing を作り、前のもの (`0x006cd940`) を破棄する |

同期済みになった SyncPing はその場でリセットされないため、本体がコンテンツ 50 を持たない間に送られた
ping 130 は、保持者に届きません。

ボックス画面のステップマシン (`0x00aa5160`、テーブル `0x2059218`) は、リスナー `0x00c8d900` が設定した
相手のボックスコマンドのフラグを読みます。ステップ 3 では、相手のフラグ 1 (そのオファー) を受けて、フラグ 1 と
4 をまとめて消去します (`0x00aa5688..0x00aa569c`)。ステップ 7 はフラグ 4 を待ち (`0x00aa5628`)、ステップ 10 でのみ
アクション 6 を発行して交換の状態 1 を開始します。オファーと同じバーストで送られたボックスコマンド 4 は消去され、
本体はステップ 7 で "En attente d'une réponse" と表示されたまま待機します。相手のボックスコマンド 4 は、本体がそのオファーを
処理した後に届く必要があります。`pokeldn.swsh.host_trade` は、参加側の 4 を受け取った後にのみ自分の 4 を送り、
参加側のランチャーはホストの 4 に応答して自分の 4 を送ります。

ステップ 7 は、プレイヤーが B を押したときにも終わります。タイマーでは終わりません。ステップマシンの ui は、
`[this+0x80]` の View_Model (`0x00aa4e78`、vtable `0x25376d8`) です。その入力ハンドラ `0x00aab0b0` (スロット 16、
UI ディスパッチャ `0x00efad28` から呼ばれる) は、`ui+0x5d0` が有効化されていて、そのフレームで押されたマスクに
ビット 49 が含まれるとき `ui+0x5cc = 1` を設定します。このビットは、B だけから remap テーブル `0x02062928` が作ります。
ステップ 2 と 6 は、オファーと承諾 (`0x00aa5434`) の後にこれを有効化します。毎ティックでクリアされます
(`0x00aa5608`)。ステップ 7 では、押下によって相手のフラグを消去し、ボックスコマンド 5 (承諾の取り消し) を送り、
シーケンスを抜けます (`0x00aa57d0`)。ステップ 3 では、ボックスコマンド 2 を送ります。

そのボックスコマンド 5 より前に送られた相手のオファーとボックスコマンド 1 は、フラグとともに消去されます。そのため、本体の
次のオファーは、それらが再び届くまでステップ 3 で待機します。`pokeldn.swsh.host_trade` と
`bin/swsh_connect.py` は、ボックスコマンド 5 の後に自分のオファーとボックスコマンド 1 を再び送ります。どちらの役割でも市販のソードが
オファー、承諾、B の押下、再度のオファーを行った場合、その後交換は完了しました。
2 回目の送信がなければ、本体は "En attente d'une réponse" のまま止まりました。

以降の交換はそれぞれ、交換ごとの部分を繰り返します。両方のオファーとボックスコマンド 1、その順番での 2 つのボックスコマンド
4、ping 130 (参加側が先に ping を送る)、フェーズ 0 での新しいコンテンツ 50、ping 120、フェーズ 0 から 4 までの新しい
コンテンツ 40 です。交換の合間に 0x84 のスナップショット、ping 97 や 110、ボックスコマンド 3、コンテンツ 30 の公開は
挟まれません。繰り返しの `--offer-file` を指定した `bin/swsh_host.py` と `bin/swsh_connect.py` は、それぞれ市販のソードの参加側と
市販のソードのホストと 1 つのセッションで組み、交換ごとにキューのレコードを 1 件ずつ使います。
本体が離れるとホストは終了します。

繰り返しの `--offer-file` を指定した `bin/swsh_connect.py` は、確定した確認の手順 (40040 のフェーズ 4) の後に
本体のオファーを次の交換として受け取ります。そのオファーに、次のレコードで応答します。このとき本体のボックスの手順は
すでにステップ 3 にあり、自分のオファーを保持しています。そのうえで、交換ごとの状態を消去します。
消去されるのは、応答済みの 40050 と 40040 のペアとボディ、確認コマンドのキュー、選択オファーのラッチです。
40030 のペアと ping への応答は引き継がれます。`bin/swsh_host.py --accept-first --lead
SECONDS` は、本体ホストのプレイヤーを模します (先に承諾し、交換の後にキューの次のレコードをボックスからオファーします)。
2 つのランチャーは、1 つのセッション上で両方向に 2 件ずつのレコードを
シミュレートされたボード (`tests/test_esp32.py`) 上で交換します。

市販の参加側のプレイヤーがボックスで B を押すと、本体はボックスコマンド 2 と 3、およびメッシュの
`0401` を送り、エラーなしで認証を解除します。その次の検索では、同じホストされたネットワーク (同じ
ネットワーク ID) に参加できます。検索中の本体は、
[照合ルール](swsh_session.md#how-a-searching-sword-finds-a-partner) を満たす任意のネットワークに参加します。

## ボックスのステートマシン

コンテンツ 30 は、交換画面からオファーまでのすべてを担います。`onBoxSyncStateCommand`
(`0x010ce180`、`session+0x120` にある 0x50 バイトのラッパーのスロット 1、vtable グループ `0x2625808`) です。

1. 自分自身のエコーは無視します (`sender == [[0x2616a30]]+0xf0`)。
2. `if ((msg->data - 1) > 5) return` の場合、0 と 7 以上は黙って破棄されます。
3. ジャンプテーブル `0x2067bec` では、ワイヤ上のコマンド *N* がイベント *N* をすべてのリスナーに通知します。

フィールド 1 の `boxSendPokemon` は、イベント 0 としてスロット 0 `0x010ce080` に渡されます。ディスパッチャは 0..6 を受け取り、
送信側は 1..5 を送ります。

リスナー `0x00c8d900` は `[owner + 0x218 + code] = 1` を設定します。コード 2 は、まずコード 1 のフラグ
(`+0x219`) を消去し、コード 5 はコード 4 のフラグ (`+0x21c`) を消去します。1 はオファー、2 はオファーの取り消し、4 は確認、5 は
4 の取り消しです。確認画面にいるプレイヤーのホスト本体は、参加側がコマンド 1 の後に 2 を送って
から 4 秒後に離脱を始めました。

送信関数 `0x010cda70(content, command)` は、ラッパー `0x010cde90` .. `0x010cded0`
(コマンド 1 から 5) の背後にあり、シーンのジャンプテーブル `0x00a96d10` が、そのアクション `+0x78` で駆動します。

    action 1  ->  command 3
    action 2  ->  the Pokemon (0x010ca430), then command 1
    action 3  ->  command 2
    action 4  ->  command 5
    action 5  ->  command 4
    action 6  ->  0x010ca640, which stores its Pokemon and sets the trade state to 1

送信は、次の 2 つの無音のゲートで守られています。

    [content+0x48] a pending command; a different one arriving sets the error byte [content+0x4d]
                   and sends nothing
    [content+0x4c] channel ready, set by 0x010ce040 on result 0; while clear the command waits in
                   +0x48 and never goes out

コマンド 3 は開始の合図です。フレームごとの更新 `0x010c9bb0` は、状態を切り替える前に次を実行します。

    if ([session+0x418]) { if (![session+0x419]) send command 3; [session+0x418] = 0; }

セットアップ `0x010c9280` は `+0x418 = 1` と、`+0x419 = 0x00dceea0() & 1` (プレイヤーのリストから) を設定します。
どちらか一方だけがコマンド 3 を送ります。`--box-open` は、0x84 の応答でボックスコマンドを送ります。

## 交換のステートマシン

`[session+0x140]` の範囲は 1..10 で、テーブルは `0x2067b68` です。

    1  0x010c9c78  -> 0x010ca0a0: start content 50 and send its Pokemon; true -> 2, false -> 9
    2  wait        5  wait        7  wait
    3  0x010c9f60  the Pokemon exchange (0x010d54b0 on content 50), then 4, or 9 on error
    4  0x010c9c9c  0x01109320 decides; false -> 6
    6  0x010c9f84  -> 0x010ca1ac: install four delegates and start content 40
    8  0x010c9fa8  0x010dac90(content40, session+0x170, session+0x210): unhook from
                   [content+0x20]+0x60, [content+0x28]+0x60, release 0x010daac0; -> 9
    9  0x010c9ef4  terminal: [+0x144] set means failure; listeners get event 8
    10 0x010ca838

ボックスのコールバック `0x010ca800` は、状態が 0 でない限りイベントを破棄します。状態 2 はデリゲート
`0x010cc380` (`session+0x2e0`) で終わります。相手のポケモンを `session+0x70` へ、0x60 バイトのレコード
(`0x010f5cc0`) として書き込み、状態 3 に進みます。中断は状態 1 で始まります。コンテンツ 50 の初期化 `0x010d4d90` (`mov w2,
#0x32`) が失敗すると、その送信 `0x010d5440` は何もせず、呼び出し元が `[session+0x144] = 1` を設定して状態 9 に移り、
通信の中断を示すメッセージが表示されます。

## 確認の手順

コンテンツ 40 は障壁であり、コマンドごとに 1 段ずつ進みます。

### フェーズと状態の対応表

状態は `delegate+0x5c` に格納され、格納するのは初期化 (0) と、マシン `0x010dae70` 内の 4 か所、および
`0x010dbf40` だけです。マシンは `state - 1` を 14 件のテーブル `0x2067ed0` に振り分けます。状態 2、4、7、9 (送信によって
ここを離れる状態) は既定の `0x010db38c` (エピローグ) に入り、次の関数を経由してのみ抜けます。
`0x010dbf40` は 4 を超える u16 を無視し、それ以外はテーブル `0x2067f4c` を索引します。

    phase 0  -> state 1              -> send(0), announcing 1     0x010db308
    phase 1  -> state 3              -> send(1), announcing 2     0x010db0b0
    phase 2  -> state 5 -> 6 or 8    -> send(2), announcing 3     0x010db0dc / 0x010db104
    phase 3  -> state 10 or 11 -> 12 -> send(3), announcing 4     0x010db16c
    phase 4  -> state 13 -> 14       -> 0x010db970 tears down, content+0x84 = 0xfc18; no send

`[delegate+0x58]` (`0x110e620(...) & 1`) が役割を表します。状態 7 は即座に 2 を送り、状態 9 は
カウントダウン `[delegate+0x60]` の後に送ります。その待機時間は xorshift による 2..302 の抽選です。

vtable グループ `0x257fe90`: 主 `0x257fea0` (スロット 0 は SyncCommand の受信 `0x010dbc90`、スロット 3
`0x010dbf40`)、`0x257fec8` の offset-to-top −8、副 `0x257fed8` (そのスロット 0 `0x010dbfd0` は
`+0x50`/`+0x54` に対する同じコードです。テーブルは `0x02067f60`、オフセットは `0x70 0x38 0x40 0x48 0x6c`)。スロット 4
`0x010dc070` は `0x010f8e30` に末尾呼び出しし、残りは `ret` です。初期化は両方を次のように設定します。

    0x010da6bc   add x9, x19, #8       ->  [content+0x2c0] = delegate + 8     the second interface
    0x010da6d0   str x19, [x8, #0x168] ->  the 10040 holder's listener        the first

### ポンプ

マシンより前に、コンテンツ 40 のティックから実行される `0x010db3e0` は、
共通の末尾を持つ `[content+0x80]` 上の 17 状態のマシンです。

    w1 = [content+0x17c]                    the content's PHASE
    if (w1 != [content+0x84] && w1 != [content+0x86])                     0x010db758..0x010db778
        { 0x010de310(content, w1); B->slot7(w1); }                        slot 7 is a ret
    w1 = [content+0x17c]
    if (w1 != [content+0x84] && w1 == [content+0x86])                     0x010db794..0x010db7b4
        { 0x010de310(content, w1); B->slot0(w1); }   <- the state setter
    0x010ddf40(content)                     the queued jobs                0x010db7d4
    0x006d4b80(content+0xd0)                the element update, last       0x010db7e8

`B` は `[content+0x2c0]` (`delegate+8`) です。状態の切り替えが最初に実行され、状態 0 (`0x010db740`) は
末尾を飛ばし、状態 16 (`0x010db730`) は `0x006d4b30` が偽のとき早期に戻ります。コミットも通知もされていないフェーズは
黙ってコミットされます (フラグとジョブが消去され、ポンプの状態は 2)。

| フィールド | 意味 | 書き込み元 |
|---|---|---|
| `+0x84` | コミット済みのフェーズ | `0x010de310`。保留中のボディも破棄し、ジョブキュー `content+0x310` を空にします (`0x010de348..0x010de3a0`)。末尾 (`0x010db778`、`0x010db7b4`) と状態 14 (`0x010db6ac`) からのみ呼ばれ、フェーズが `+0x84` と異なるときに限られます |
| `+0x86` | 直近のコマンドで通知されたフェーズ | `0x010dbab0` のみ。5 か所の送信箇所すべてで `data + 1` |
| `+0x17c` | フェーズ (`element+0xac`) | コンストラクタ `0x006d3ff8`、生成 `0x006d4500`、採用 `0x006d4ccc` |

フェーズが `+0x86` に達すると、確認の手順は進みます。コンストラクタは `+0x84 = 0xfc18` を設定し
(`0x010dc228`、`0x010dc260`)、`+0x86 = 0` (`0x010dc25c`) を設定します。レジストラは `+0x84` を 0 にし
(`0x010da810`、`0x010daa78`)、0 で要素を生成し (`0x010daa7c`)、`[content+0x80] =
1` のままにします。そのあとポンプの状態 1 は、ゲートの外で状態セッターを呼びます。コマンド 0、通知 1 です。

ポンプの状態 (テーブル `0x2067f08`。`[+0x1c0]` は要素の `+0xf0` チャネル、`[+0x2a0]` は
[コマンド](#the-command) のフラグです):

    1   0x010db418  element ready (0x006d4e70) -> 2; delegate slot 1, then slot 0 with the phase
    2   0x010db47c  a pending body at +0x88 -> 3
    3   0x010db48c  0x006d4f10 (all ready, every pair's low half == phase): send the body
                    (0x010de080) to station [[0x2616a30]+0xf8] -> 4
    4   0x010db4dc  0x006d3690([+0x1c0], [+0x86]) publishes the high half, must return 1;
                    then 0x018407f0 ? 8 : 7                                      cinc 0x010db514
    5   0x010db51c  the same through [+0x2b8] (0x008b7300) with [+0xa8] -> 6; dead in content 40
    8   0x010db594  0x006a2840([+0x2a0]): every station has sent a command -> 9
    9   0x010db5b4  0x006d4fb0 == 0: no resend byte set -> 10
    10  0x010db5d4  0x006d4da0 && 0x006d3060([+0x1c0], [+0x86]) (every high half == announced)
                    && 0x006d4f50; then 0x006d33b0 writes shared value = announced -> 7
    11  0x010db62c  0x006a2760([+0x2a0]) clears every flag; [+0x374] ? 12 : 2
    6, 7, 12        the default, the shared tail at 0x010db758
    14  0x010db68c  the gated commit (13, 15, 16 unread)

状態 8 から 10 はマスター (`0x018407f0`) でのみ実行されます。ほかは 4 から 7 に進み、共有値に従います。
`0x010d9000..0x010df000` の中で `[content+0x80]` に 5 を、`[content+0xa8]` に 0 以外を格納するものはありません
(消去のみ。`0x010db9ec`、`0x010dc530`、`0x010de3a4`)。`[content+0x80]` へのほかの書き込み元は次のとおりです。

| 箇所 | 関数 | 状態 |
|---|---|---|
| `0x010dc234` | コンストラクタ | 0 |
| `0x010daa9c` | レジストラ | 1 |
| `0x010dbc70` | `0x010dbab0`、コマンド送信 (2 を要求し、`+0x86` を書き込む) | 3 |
| `0x010de3c4` | `0x010de310`、コミット。`0x010de344` で全フラグを消去 | 2 |
| `0x010dc7a0` | `0x010dc720` (スロット `0x257ff70`)、ステーションの離脱。`[+0x374]` がある場合はデリゲートのスロット 4 に通知し、ない場合は要素 (`0x006d50e0`) とフラグ (`0x006a2140`) から削除 | 11 |
| `0x010dc6f4` | `0x010dc6c0` (スロット `0x257ff68`)。`[+0x374]` と `0x006a2a80([+0x2a0])` を要求 | 13 |
| `0x010dccec` | スロット `0x257ff90` | 2 |
| `0x010dcee8` | `0x010dce70`、CancelAccepted (`x19 = content+0x68`)。状態 16 の場合を除く | 2 |
| `0x010db998` | `0x010db970`、破棄 | 16 |

コンテンツ 40 のレジストラは、`[+0x374]` (`0x010da800`) に `w2 = 1` (`0x010da6a8`) を格納します。ステーションが
離脱すると 11、12 と進み、確認の手順は終了します。

### フェーズは要素のフィールドです

`0x010c0000..0x010e0000` のどこも `content+0x17c` に格納しません。レジストラは、40040 の要素を
`content+0xd0` (`0x010daa4c`) に作り、サブ要素 (`0x006d4ff0`) を加えて、`0x006d44e0(element, w20)` で生成します。
この生成は `str wzr,[x0,#0xa8]; strh w1,[x0,#0xac]` を開きます。フェーズは
`element+0xac` (`0xd0 + 0xac = 0x17c`) で、初期化の `wzr` から始まります。

フェーズが進むのは `0x006d4ca0` の中だけです。

    w0  = 0x006d3260([element+0xf0])        the shared value; 0xfc18 when none
    if (w0 == [element+0xac]) done
    if (0x006d3980([element+0xf0], w0)) {   publish it as its own; on success
        [element+0xac] = w0                 the phase moves
        [element+0xa0]->vtable[0]()         and the content is told
    }

これは要素の更新 `0x006d4b80` の末尾です (すべての同期ポンプが最後に呼び、コンテンツ 40 では
`0x010db7e8`)。`0x006ce5a0([element+0xb0])` が 1 を返したときに実行されます。すべてのステーションの準備が整う前は次のとおりです。

    if (!0x006d2e20(shared) && !0x006da180(hash channel)) {
        if (0x018407f0([[0x2616a30]]))  0x006d33b0(shared, [element+0xac])
        0x006da3d0(hash channel, element+0xa8); 0x006d3980(shared, [element+0xac])
        return
    }

`0x018407f0` は、Pia のメッシュをホストする本体で真になります
([セッション](swsh_session.md#the-pia-session-object)。バックエンド呼び出し `+0xe0` は
`[x0 + 0x168 + [x0+0x162]*8]` 上にあります)。共有値を設定するのはこれだけで、`+0xf0` チャネルの 16 ビット
サブ要素 ([サブ要素の種類](swsh_protocol.md#sub-element-kinds)) で、すべてのステーションがそれを採用します。

    0x006d3260  read            ready [sub+0x61] ? [sub+0x88] : 0xfc18
    0x006d33b0  write(v)        0x006d3570 when not ready or body != v
    0x006d3570  publish         body [sub+0x88] = v; clock from 0x01766740 (-1: error 0x2c27);
                                [sub+0x80]->vtable[0] with (sub+0x88, 2, clock, [sub+0x62], [sub+0x68])
                                when [sub+0x80]->vtable[1]() or [sub+0x70] allows, resend byte
                                [sub+0x60] = !sent; else [sub+0x60] = 1
    0x006d2e20  all ready       the shared value and every pair
    0x006d2c20  all low == v    all ready, every pair's [sub+0x88] == v
    0x006d3060  all high == v   all ready, every pair's [sub+0x8a] == v

書き込みは ready バイトには触れません (`+0x60` を格納するのは受信 `0x006d6a08` だけで、書き込みは
`0x006d35b8 strh w8,[x20,#0x88]!`)。そのため、メッセージが届くまで読み取りは `0xfc18` を返します。マスターでは
そのメッセージは自分自身のものです。公開は要素のスロット 0 (`0x006d5730`) に届き、そこでステーションごとの
ボディがキューに入ります。1 回の完了した交換で 40040 上にあった 149 件のクライアントメッセージは、すべて 4 バイトのボディを持っていました。
コンテンツのほかの形は[下記](#shapes-the-confirmation-content-also-sends)のとおりです。

### ステップ本体

コンテンツ 40 の elementId 20000 にある 4 バイトのボディは、下位半分がフェーズ、上位半分が直近の
通知値です。

    0x006d3980(channel, v)   body = <u16 v><u16 [sub+0x8a]>    low; from the element update
    0x006d3690(channel, v)   body = <u16 [sub+0x88]><u16 v>    high; from pump state 4, [content+0x86]

どちらも、チャネルの `{ownerId, entry}` テーブルから自分の id (`[[0x2616a30]]+0xf0`) を探し、
`[entry+0x60] - 0x50` を取り、`0x006d3860` を呼びます (一段下で `0x010dbe20`)。`[content+0x1c0]` は
`[element+0xf0]` です。観測された確認の手順は次のとおりです。

    000018fc   phase 0, announced 0xfc18   the birth sentinel
    00000100   phase 0, announced 1        the cue; command 0 already sent
    01000100   phase 1, announced 1        caught up, committed
    01000200   phase 1, announced 2        state 3 sent command 1
    02000200   phase 2, announced 2        caught up
    02000300   phase 2, announced 3        committed, sent command 2
    04000400   phase 4                     the teardown rung

`trade.parse_sync_step`、`SYNC_LADDER`、`sync_announced_phase` がこれを解釈します。

### コマンド

10040 上の `SyncSaveDataHolder{syncCommand{data:N}}`、信頼性ポート 0。ハンドラ `0x010dbc90` は、
`0x006b5850` が解決できない送信元 (`0xfd`) を破棄し、`[arg1+0x14]` の int32 を取り出し、購読者を
ステーション・インデックスで `+0x38` から選び、int32 を中継 `0x010dbe20` と購読者の `+0x18` に渡し、
その後は常に次を実行します。

    0x010dbdf8  ldr  x8, [x20, #0x18]
    0x010dbdfc  ldr  x0, [x8, #0x2a0]
    0x010dbe00  mov  w2, #1          <- a constant
    0x010dbe04  mov  x1, x19         <- keyed by the sender
    0x010dbe08  bl   #0x6a24a0

値は中継にしか届きません (elementId 1 のボディ `00000000` は `syncCommand{data:0}` を返します)。
状態が動くのは、ポンプが自分のフェーズで呼ぶ `0x010dbf40` を経由する場合だけです。どのコマンドも
段を 1 つ進めます。`[content+0x2a0]` はステーションごとのフラグのオブジェクトです (`0x006a24a0` の呼び出し箇所は 44 か所)。

    0x006a24a0(obj, id, flag)   map[id] = flag & 1; map at obj+0x1c0, find-or-insert
                                (0x006a24d0 hashes by udiv/msub on the bucket count at +0x10)
    0x006a2840(obj)             1 when every station listed at obj+0x100 (count +0x108) is set
    0x006a2760(obj)             clear(): frees nodes, zeroes buckets and the size at +0x270
    0x006a2480(obj)             obj+0xc0, the station list the registrar walks

状態 8 は `0x006a2840` を待ち、コミットと状態 11 がマップを消去します。フラグは、ポンプの外側の
ネットワーク更新のドレインで設定されます。`0x006a9a20` は Pia のディスパッチ `0x006a8380` を呼び、
(`0x006a9a54`)、その後ドレイン `0x006db3b0` (`0x006a9a90`、
[ルーティング](swsh_protocol.md#the-routing-path)) を呼びます。その呼び出し元 `0x00ef4a9c`、`0x01109250`、
`0x01109308` のうち、フレーム `0x00f1df30` は 2 つを、ゲーム更新の前後で実行します。

    0x00f1df30  bl 0x01109240     network update: 0x01109250 bl 0x006a9a20, the drain
                bl 0x00fa09a0
                bl 0x00f1cc60     the game update
                bl 0x00793ec0
                bl 0x011092f0     0x01109304 bl 0x01111db0, 0x01109308 bl 0x006a9a20, the drain again

コンテンツ 40 のコマンドのティック `0x010dae70` は、交換シーンのタスクで実行されます。`0x00c8c6b0` は
`0x00c8c6d8` で `0x010c9bb0` を呼び、それが `0x010c9c34` のティックを呼びます。タスクのディスパッチャは
`0x00f19770` と `0x00f19080` を経由してそこに到達します。エミュレータのトレースでは、このティックは
`0x01109250`、`0x00f1cc60`、`0x01109308` の順より前に置かれます。そこでドレインされたコマンドは、
続くシーンのティックで消費されることがあります。

RequestCancel と RequestCancelAll も、マップを読みます ([下記](#the-cancel-and-proceed-messages))。

1 段は、前のコミットの後にマスターに届く 1 つのコマンドで進みます。段 0 では、どのコマンドも
数えられ、合図 `00000100` 上のものも含まれます。その後は、合図 (`01000200`、
`02000300`、`03000400`) 上のコマンドは、コミットの前に届くと消去され、追いついたボディ
(`01000100`、`02000200`、`03000300`) 上のものだけが数えられます。

| 新しいボディごとに 1 件のコマンド | 最後のボディ |
|---|---|
| `00000100` 上の 1 件 | `01000200`: 段 0 |
| 4 件、`000018fc` から `01000200` まで | `02000300`: 段 0 と 1 |
| 9 件、`000018fc` から `04000400` まで | `04000400`: 交換全体 |

`--confirm-commands 0,1,2,3,0,1,2,3,0,1,2,3` は、新しいボディごとに 1 件を取り出します (`000018fc` を含む)。キューを
空にしてはいけません。`answer_rpc` はボディをそのまま返し、どちらの半分も動かしません。`--confirm-phase N` は
下位半分を書き込みます。

### 取り消しと続行のメッセージ

30040 の `SequenceDataHolder` ([プロトコルのページ](swsh_protocol.md#the-30000-holder)) は、
`[content+0x2b8]` (`0x010da9b8`) に格納され、リスナー `content+0x68` (`0x010da9b4`、`0x010da9bc`、
vtable `0x02580030`、`0x010dc238`) を持ちます。

| ケース | メッセージ | ハンドラ |
|---|---|---|
| 1 | CancelAccepted | `0x010dce70` |
| 2 | RequestCancel | `0x010dc920`、サンク `0x010dcf20` 経由 |
| 3 | RequestCancelAll | `0x010dcaa0`、`0x010dcf30` 経由 |
| 4 | RequestForcedProceed | `0x010dc7b0`、`0x010dcf40` 経由 |

それぞれは、`currentSeqNo` がコミット済みのフェーズ `(s16)[content+0x84]` と等しいときにのみ動作します。

- RequestCancel は、すべてのステーションにフラグが立っていないとき (`0x006a2840`、`0x010dc954`) に動作します。ジョブ `0x010dd0c0` は
  `[content+0x2b8]` 経由で送信元に `CancelAccepted{currentSeqNo: [+0x84], isForced: 0}` を送り、
  (`0x008b7230`)、送信元のフラグを消去します (`0x010dd114`)。
- RequestCancelAll も同じガードです。ジョブ `0x010dd190` は、リストされたステーションごとに (`0x010dcaf4..0x010dcb1c`)
  `CancelAccepted{[+0x84], isForced: 1}` を送り、その後すべてのフラグを消去します (`0x010dd208`)。
- CancelAccepted: `isForced` があるとき、`[content+0x370]` を mod 255 で `0x006db470` に進め、ポンプの状態を 2 にします
  (状態 16 の場合を除く。`0x010dcee8`)。デリゲートのスロット 6 (`ret`) を呼びます。状態 2 から 4 は再実行され、フェーズは変わりません。
- RequestForcedProceed: ジョブ `0x010dd040` `{content, u16 targetSeqNo, sender index}` は即座に実行されます
  (`0x010dc8a8`)。偽のときはキューに入ります (`0x010dc8bc`)。`0x006d4da0(content+0xd0)` の後、
  `0x006d33b0([content+0x1c0], targetSeqNo)` を呼びます。これは、状態 8 と 9 を経ず、
  `0x006d3060`/`0x006d4f50` の検査やマスターの検査も行わない状態 10 の書き込みです。送信元インデックスは読み取られません。

`RequestForcedProceed` は、`syncCommand` なしで共有フェーズを進めます (エミュレートされたシールドによる)。これは
確認コマンドを置き換えるものではありません。後続のコマンドがこれらの要求である確認の手順は、セーブの前の
"Communicating" で止まります。

ジョブキュー `content+0x310` (エントリ `+0x350`、件数 `+0x358`) は、ポンプ (`0x010db7d4`) から
`0x010ddf40` で実行されます。true を返したジョブは取り除かれます。コミットがキューを空にします。

### 確認コンテンツがほかに送る形

ownerId がなく、4 バイト以外のボディを持つ 40040 のエンベロープです。

    elementId 20000, no owner, 2 bytes   0000   then   0100
    elementId 1,     no owner, 4 bytes   00000000        after a syncCommand{data:0}
    no elementId,    no owner, 4 bytes   00000000  then  01000000

ペアの受信 `0x006d6490` は 2 バイトのものを破棄します (`cmp x2,#4`)。それらは共有値です
([フェーズは要素のフィールドです](#the-phase-is-the-elements-field))。

## リーグカード

交換の後、各本体は相手のリーグカード (スナップショット内の TrainerCard、
0x924 の位置。[プロトコル](swsh_protocol.md#the-party-payload-on-protocol-0x84)) を保持するかどうかを尋ねます。その回答では
何も送信しません。

Oui (はい) を選ぶと、カードは変更されずにセーブブロック `0x28e707f5` に格納されます。`album+0x230` から 0x1d0 バイトのスロットが 300 個あり、
1 つの `0x21fc0` バイトのブロック (`0x013fad4c`) として読み込まれ、`+0x1c8` が 0 でない間は空きです。`0x013fbab0` は
最初の空きスロットを埋めます: 0x1c4 バイト、`+0x1c8`/`+0x1c9` をゼロにし、年 `+0x1ca` (`tm_year + 0x76c`)、月 + 1
`+0x1cc`、日 `+0x1cd`、`+0x1ce`/`+0x1cf` は引数から (`00 00 ea 07 09 1a 02 00` は、年 0x07ea、月 9、日 0x1a)。
300 個すべてが使われているとき、`0x013fbc00` は 1 を返します。

`0x013fbc40(album, card)` (呼び出し元 `0x00aa6414`、`0x0106612c`、`0x015253f0`、`0x01543868`) は、
使用中のスロットが、カードの 0x1C にある u32 (`0x013fbc4c`) と、0x1A8 にある 8 バイトに一致すると、確認を飛ばします
(`0x013fbc5c`、64 ビットの比較 1 回)。そのスロットを空にすると、確認が戻ります。0x1C は PKHeX の
`TrainerCard8.TrainerID` で、`(SID << 16 | TID) mod 10**6` です (56909/48474 の場合は 848973)。0x1A8 は
`TimestampPrinted` で、Unix 時刻です (市販のカードでは `9bc62a5e00000000`、2020-01-24)。ゲームは
8 バイトすべてを比較します。`pokeldn.swsh.league_card` は u32 を対応付けます。`trade_payload.rewrite` はトレーナー ID を設定します。
ホストのカードを持つエミュレートされたシールドに対する結果は次のとおりです。

| ホストのカード | 確認 |
|---|---|
| 変更なし | なし |
| トレーナー ID 848973 -> 111111 | あり。最初のスロットの隣の新しいスロットに格納 |
| 図鑑数 400 -> 401 | なし |
| 名前 (1 文字追加) | なし |
| timestamp_printed、0x1A8 のバイトだけ変更 | あり |

市販のソードは、受け取った日付、左上に `game` (0x24、0 はソード) のロゴ、
左下に 0x39 の 3 つの ASCII バイト、`dex_complete` (0x30) 用のロトム図鑑の王冠、そして星を描きます。ビュー
`0x01592e70` はカードを `view+0x3c0` にコピーします。`0x015a52f0(view, count)` は、7 つのパネル (`view+0x618..+0x648`) のうち
`count + 1` 個を点灯します。検証はカウントを 6 で打ち切ります。次の式で数えます。

    count = card[0x177] + (card[0x1b6] != 0) + (card[0x1b7] != 0)      0x015930bc..0x015930dc

また、`card[0x24]` を `0x015a5260` に、`card[0x1b3] != 0` を `0x015a50d0` に渡します。ビルダー
`0x0158eef0` (呼び出し元 `0x00c77830`、`0x014be578`、`0x0156167c`、`0x01561b0c`、`0x01568f4c`、
`0x015691bc`) は、セーブから埋めます。

| バイト | 値 | 箇所 |
|---|---|---|
| 0x177 | `DesignLevel`: `FSYS_GAME_CLEAR` (`0x7d0b1ced4dbe8a87`) があれば 4、それ以外は、バッジが 6 超、3 超、0 超のとき 3、2、1、そのほかは 0 | `0x0158f084`、`0x0158f484..0x0158f4dc`、`0x0158f0c4` |
| 0x1B1 | `FSYS_SEED_CHALLENGE` (`0xbd07c2b80232e9b0`) | `0x0158f564..0x0158f5a0` |
| 0x1B3 | `FSYS_INPUT_SHIRT_NUMBER` (`0xe44b16771524b07e`) | `0x0158f0fc` |
| 0x1B6 | `FSYS_R1_GAME_CLEAR` (`0x8f1a133dff5c0ecf`)、星 | `0x0158f11c` |
| 0x1B7 | `FSYS_R2_GAME_CLEAR` (`0xd450875d834cfc30`)、星 | `0x0158f12c` |

フラグのキーは、名前の FNV-1a 64 です ([プロトコル](swsh_protocol.md#the-player-profile))。読み取り元は
`0x01410f30` です。バッジ数は `0x01438fb0` で、`[status+0x60]` のポップカウントです。`DesignLevel` は
`capture_data.prmb` 列 (ハッシュ `0x3bc4598485d2a2ef`、文字列 `0x01c2a2e9`) で、
`0x0158ead4..0x0158eb30` により格納されます。`GlossIndex` は 0x178 です。

`0x0158f5e0` は、カードを拒否するために 1 を返し、そのカードは既定値に置き換えられます。unicorn の下では、
pokeldn が送るカード (0x177 = 4、0x1B3 = 1、言語 3) と、0x1B6 か 0x1B7 が立っているカードは受け入れます。拒否するのは
DesignLevel 5、言語 6、GlossIndex 9 です。`bin/swsh_host.py --card-set FIELD=VALUE` は送るカード
(`pokeldn.swsh.league_card` のフィールド) を編集します。新しい `trainer_id` を使うと、本体は再度確認します。

## 提示されたレコード

市販のソードに交換して戻した、組み立てたピカチュウは、要求されたオファーのすべての選択肢を保持します。返された
PK8 は有効なチェックサムを持ち、PKHeX の合法性検査に通ります。PID、元のトレーナー ID、個体値、努力値は、
送ったオファーと一致します。

| 要求したフィールド | 返されたレコード |
|---|---|
| レベル | 30 |
| 性格と能力の性格 | いじっぱり、3 |
| 特性 | ひらいしん、31 |
| 性別 | メス、1 |
| ボール | ハイパーボール、2 |
| 持ち物 | でんきだま、236 |
| 選んだ個体値 | HP 31、こうげき 0、すばやさ 31 |
| 努力値 | HP 252、すばやさ 4、ほかのすべての能力 0 |

`0x011e3458` で読み取られる暗号化定数は、パーティのインデックスとともにポケモンキャンプの
モデルキー `[model+0x368]` に入り、キャンプの同期で使われます。交換やボックスのコードには届きません。

20030 のオファーは、スナップショットのパーティのスロット `--offer-slot` を、その場で編集したものです。そのため、表示されるパーティと
オファーは一致します。アイデンティティの書き換えが先に実行され、`party_matches_trainer` が成り立ちます。
`pokeldn.swsh.pokemon.build_from` は、チェックサムを書き換え、新しい暗号化定数の下で並べ替えを行い、
名前のないバイト (リボン、思い出、出会いのデータ、ハンドラのレコード) はすべて保持します。

フランス語版ソード 1.3.2 は、セーブにすでにあるレコード (同じ PID と EC) を、4 回の交換で受け入れ、
そのまま交換に出しました。シールド 1.3.2 で読んだ経路には重複チェックがありません。PID の取得関数
(`0x0076bc20`、ブロック A + 0x14) は色違いの判定テストからだけ呼ばれ、暗号化定数の取得関数
(`0x0077ec90`) は 12 か所で読まれますが、どれもボックスを走査しません。ボックスのコード
(`obj+0x60 + box*0x2850 + slot*0x158`) は種族とタマゴのフラグだけを読みます。また、どのアクセサも
PK8 のバイト 0x52 には触れません。ブリリアントダイヤモンドは、そこに不正フラグを保持します
([BDSP の交換のページ](bdsp_trade.md#duplicate-detection))。`bin/swsh_host.py --fresh-pid` と
`bin/swsh_connect.py --fresh-pid` は、新しい暗号化定数と PID を生成します。ソードでは、これは
念のための措置です。

計算された長さを使う `memcmp` スタブ `0x018fe460` への 375 回の呼び出しは、ポケモン同士を
比較していません。374 回は文字列やライブラリのデータ (sol2、protobuf、NEX、Lua、Pia、SDK のグラフィック、名前の
辞書、バトルデータのホルダー、FlatBuffers の vtable) を比較し、`0x014514ac` は `obj+0x60` の 0x158 バイトのスロットを
ゼロと比較します。交換の受信経路 (`0x010d81d0` から `0x010f5cc0` まで) やボックスの経路 (`0x010ce180`、`0x00c8d900`) からは、
直接の呼び出しで到達するものはありません。交換のコードが到達するのは protobuf のディスクリプタ登録 (`0x006e26f0`) だけで、
そのオペランドは rodata の名前です。
vtable 経由の呼び出しは追っていません。

    --offer-slot 1 --offer-nickname POKELDN --offer-ivs 31,31,31,31,31,31

| フラグ | 効果 |
|---|---|
| `--offer-nickname`、`--offer-ot` | 26 バイトの UTF-16 フィールド、最大 12 文字。ニックネームを指定するとニックネームのフラグが立ち、そうでなければ種族名が表示されます |
| `--offer-species` | 種族の値だけを変え、ほかはテンプレートのままにします |
| `--offer-ability ID`、`--offer-moves A,B,C,D` | この 2 つのフィールドを変えます。PP と再習得の技はそのままです |
| `--offer-level N` | 0x148 のレベルのバイト |
| `--offer-experience N` | 0x10 の経験値のワード |
| `--offer-ivs` | 6 つの個体値 |
| `--offer-file FILE` | 4 種類のいずれかの形の `.pk8` (保存されたものかパーティのもの、暗号化されたものか PKHeX の復号済みエクスポート。ヘッダーのチェックサムで区別します)。OT の名前と ID はスナップショットのトレーナーに移されます |
| `--offer-file-as-is` | ファイルの OT をそのまま保持します |
| `--save-offer FILE` | 無線に触れる前に、組み立てたレコードを書き出します |

フランス語版ソード 1.3.2 が、編集されたレコードをどう扱うかは次のとおりです。

| 送信したレコード | 結果 |
|---|---|
| テンプレート、ニックネーム、個体値 31 | 受け入れ。名前のないフィールドはテンプレートのままです |
| ゲンガーの特性 130 と技を付けた種族 93 | ゴーストとして描画され、交換で進化した扱い。特性と技は検査されません |
| 種族 25 | 能力値は個体値、努力値、性格、ハイパートレーニング 0x126 から再構築され、0x14A の能力値のバイトは破棄されます |
| レベルのバイト 0x148 = 50、レベル 100 の経験値 | レベル 100。レベルは経験値 (0x10) から決まり、パーティの末尾は再構築されます |
| OT を 12345/54321 に移動 | 表示される ID は 993401、`(SID << 16 \| TID) mod 1000000` |
| 他人の MyStatus の下で、受け取り側自身の OT | プレイヤー自身の捕獲として受け入れ。OT の ID は MyStatus と比較されません |
| PID の上位半分 = `low ^ TID ^ SID` | 色違い |
| 種族 152 (存在しない) | 格納されます。ピカチュウのモデル、名前のフィールド、黒いモンスターボールのアイコン、成長グループ 0、基礎能力値 0。変わるのは現在の HP (0x8A) とハンドラのブロック (0xA8、0xC2、0xC3、0xC8、0xCB-0xCC) だけです |

未測定の項目は、ほかの存在しない種族がどう表示されるか、そして名前が名前フィールドのものか
検索の失敗によるものかです。

## 完了した交換のコマンドライン

`swsh_connect.py --preset trade` は、交換に必要なフラグを持ちます。後ろに指定したフラグは、
プリセットを上書きします。本体にローカルのリンク交換を検索させ (Y-Comm、Link Trade、local、
両方のメッセージで A)、次を実行します。

    POKELDN_RADIO=esp32:auto ./.venv/bin/python -u bin/swsh_connect.py --keys PROD_KEYS \
        --preset trade --save-offered offered.pk8 --capture trade.jsonl

返されるスナップショットは、同じセッションの本体のもの (`--send-snapshot live`、
プリセットの既定値) です。3 つの 0x84 の断片は届いた順に組み立てられ、トレーナー名、TID、
SID は `--snapshot-name/-tid/-sid` に書き換えられ、本体のものが
組み立てられた時点で送られます。そのため、事前のキャプチャセッションは不要です。`--send-snapshot FILE` は、
代わりに保存された 3456 バイトのペイロードを送ります (`--preset capture` の後、`tools/switch/swsh_snapshot.py` がそれを書き出します)。

`--offer-file FILE` は `.pk8` をオファーするパーティのスロットに入れ、その OT をスナップショットのトレーナーに移します。
繰り返し指定すると、セッション上で交換ごとに 1 件のレコードをキューに入れます。最後のものは、以降のすべての交換に使われ、
`--fresh-pid` を付けると新しい PID で、交換 N は `-N` 付きの `--save-offered` に書き出します。本体がオファーした
交換で、確認の手順がまだ終わっていないものは、
`--hold` を過ぎてから最大 `--grace` 秒 (300 秒) の間セッションを保持します。
パーティの能力値を持たない保存形式のレコードも交換できます。本体は
経験値からレベルを計算します。

## ペナルティと、実行の正常な終了

リンクが生きたままの交換がタイムアウトすると、その交換は失敗となり、本体は交換から締め出されます
(`msg_ui_live_comm_app_alert_00`、`/bin/message/French/common/live_comm.dat` の 331 行目)。その文面は
期間を示さず、そのラベルは `main.bin` にありません。ロックの長さは読み取られていません。
`04000400` の 15 秒後、セーブの最中にリンクを切ると、通信エラー `2-ALZAA-0016` が表示され、
交換のロックはかからず、すぐに新しいローカル検索が始まります。

`bin/swsh_connect.py --abort-on-stall SECONDS` は、確認の手順が始まった後に SECONDS 秒が過ぎ、
新しいボディが届かなければリンクを切ります。フェーズ 4 では手を引きます。無音で破棄する段です (`stall_abort()`、
`final_phase_seen`、`LADDER_FINAL_PHASE`)。そのため、セーブの最中にリンクが切られることはありません。

## 完了した交換の検証

受け取ったポケモンは 0xC4 に `CurrentHandler` 1 を持ち、受け取り側の名前が
0xA8 の `HandlingTrainerName` に入り、送り手の元のトレーナーを上書きします。`--offer-echo` は本体自身のレコードを返すため、
種族が異なる場合だけが交換の証拠になります。
