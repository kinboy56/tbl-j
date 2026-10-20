---
title: セッションへの参加と Pia 層
parent: Brilliant Diamond and Shining Pearl
nav_order: 1
---

# BDSP セッションへの参加と、暗号化の内側

## アドバタイズメント

    local_communication_id  0100000011d90000
    scene_id                4352 (0x1100) Union Room; 5120 (0x1400) Union Room entered with a
                            password; 12608 (0x3140) Grand Underground
    version                 4
    channel                 a 2.4 GHz channel, band 2, chosen per session (6 in the capture)
    accept_policy           ALL
    participants            1/8
    application_data        17 bytes

comm id は、ブリリアントダイヤモンドのタイトル ID で、シャイニングパールも同じ値を共有しています (`010018e011d92000`)。
アドバタイズメントは `prod.keys` だけで復号できます (`tools/ldn/ldn_scan.py`)。アプリケーションデータの 17 バイトは、
Pia の LDN アドバタイズメントヘッダー ([無線層](ldn.md)) の 16 バイトと、続く 1 バイトのアプリケーションデータです。
パスワードなしで開いた部屋では、CRC32 フィールドは 0 です。

パスワードを付けて開いた部屋は、アドバタイズメントにパスワードの ASCII 数字の CRC32 をリトルエンディアンで持ち (00000000 -> `0xC0088D03`)、
シーン ID は 5120 です。`bin/bdsp_connect.py` は、
そのような部屋に変更なしで参加できます。`bin/bdsp_host.py --password 00000000` は両方を広告し、
そのパスワードを入力した本体は参加してトレードできます。本体は次の 3 つの段階で確認します (1.3.0 `main`):

| 段階 | コード | 確認内容 |
|---|---|---|
| スキャン | `NetworkHelper.CreateUnionGameMode` `0x224e630`; `nn::ldn::Scan` `0x16be184` | シーン ID がゲームモードを表す (0x1100 通常、0x1200 グループ、0x1400 パスワード) かを確認し、スキャンフィルター (フラグ 0x25: comm id、ネットワーク種別、シーン ID) を使う。別のシーンの部屋がゲームに届くことはない |
| 一覧 | `LdnMatchmakeSession` `0x16c1abc`; `GameState_BrowseSessionAfter_LocalRandom2` `0x273f804` | +0x04 の u32 が 0 でないとき、その部屋はパスワード付きと判定される。パスワードを入力済みの本体はパスワードなしの部屋を読み飛ばし、入力していない本体はパスワード付きの部屋を読み飛ばす |
| 接続 | `LocalMatchJoinSessionJob` `0x16c36c8`; 確認処理 `0x16b8890` | 参加側は入力したパスワードの `crc32` を計算する (`0x1719204`)。+0x08 は 8 でなければならず、広告された +0x04 の u32 と一致しない CRC は `nn::ldn::Connect` の前に 0x6c51 で失敗する |

ホストは `LdnProtocol` `0x16b6a14` でそのヘッダーを書き込みます。内容は network id、
パスワードの CRC32、値 8 のバイト、セッションパラメータです。
参加側の CRC をホストに伝えるメッセージはありません。間違ったパスワードを入力した本体は、ホストに何も送らず、自分の部屋を開きます。
8 文字を超えるパスワードは、先頭 8 文字だけを Pia に使い、
残りは `INL1` の後ろのアプリケーションデータに入ります (`IlcaNetSession.SettingSet` `0x2735f14`)。

## パスフレーズ

    WirelessStrongCryptoKey2021

そのまま使います。27 バイトで、パディングもハッシュもしません (LDN は長さを明示して 16 から 64 バイトを受け付けます)。
ゲームは `nn::ldn::SecurityConfig` に入れて `nn::ldn::CreateNetwork` に渡し、
Pia の暗号処理には届きません。

## 席に着く

`bin/bdsp_join.py` はスキャンして接続し、参加者の一覧を表示します:

    participant 0: ip=169.254.54.1  mac=48f1eb209b22                 <- the console
    participant 1: ip=169.254.54.2  mac=58d8122149a2  name=b'POKELDN' <- the client

IP は本体が割り当てます。ユニオンルームの 8 席は LDN の `max_participants` にあたります。LDN の席はゲームより下の層にあり、
画面には何も表示されず、Pia セッションの席でもありません。

`Connect failed with status code 1` は接続 (アソシエーション) の失敗です。
ESP32 ボードでは約 2 回に 1 回失敗するので、原因を調べる前に再試行してください。
部屋に入り直すと新しいネットワーク (新しいチャンネル、SSID、セッションパラメータ) が開きます。鍵の導出は、その変更を実行中に処理します。

部屋にいる本体は、画面に変化がないままアドバタイズを止めることがあります。ランダムマッチングが自分のセッションを閉じてから、もう一度マッチングし直すためです。
ユニオンルームは `NetworkManager$$StartSessionRandomJoin` [1.3.0 main 0x0224f600] (`UnionRoomManager$$SetUp` と `$$SessionStart` から呼ばれる `SessionManager$$StartSession` 0x01df89e0) でセッションを開始します。
このとき使う `NetworkParam` は、
`Reset` [0x0202ec30] が `matchingMode` を 1 (Random) に、
ネットワークをローカルに設定したものです。
ランダムマッチングでセッションを作成または参加したあと (`GameState_JoinProcessAll` -> `ToGameFrontBeforeLocalRandom` [0x02743940]。
ゲーム状態 20 に入る唯一の経路です)、
`INL1.IlcaNetSession$$GameState_GameFrontBefore_LocalRandom` [0x02740640] がセッションの更新ごとに実行されます:

| ステーション数 | 効果 |
|---|---|
| 2 以上 | `GameFrontRnoInit` が両方のカウンターをクリアし、ゲームのフロントに進む |
| 1 | カウンター 0 とカウンター 1 (`gameFront_cnt`) がそれぞれ 1 増える |
| 1、カウンター 0 が `localRandomMatchmakeHostWaitTime + (localRandomMatchmakeHostWaitTimeMask & r)` を超え、カウンター 1 が `localRandomMatchmakeTimeUp` 以下 | `CleanupRecoveryToLoggedIn` [0x0273a7f0]、ゲーム状態 9 (`GS_LoggedInReturnWaitWorker`): セッションが閉じられ、マッチングがやり直される |
| 1、カウンター 0 が待ち時間を超え、カウンター 1 が時間切れを超える | `GameFrontRnoInit`。本体は自分のセッションのホストのままになる |

`r` は、状態 20 に入るたびに引かれる新しい乱数です。このときカウンター 0 もクリアされ、カウンター 1 は動き続けます。
`IlcaNetSessionSetting` のコンストラクター [0x01f46fc0] は待ち時間を 25、
マスクを 0x7F、時間切れを 270 に設定し、`SessionConnector$$StartSession` [0x0202f2c0] はこれらを変更しません。
そのため、新しいセッションに一人でいる本体は、25 から 152 回の更新のあとにセッションを閉じ、
それを繰り返します。一人でいた更新の合計が 270 回になると、最後のネットワークを保持します。
LDN インターフェースで受信するプログラムは、自分の送信元 IP を除外しなければなりません。ブロードキャストは自分にも戻ってくるためです。

セッションは描画フレームごとに、1 秒間に 30 回更新されるので、待ち時間は 0.83 から 5.07 秒、時間切れは 9.0 秒です。フレームが落ちると両方とも長くなります。
1.3.0 main での呼び出しの連鎖は次のとおりです:

| 段階 | 箇所 |
|---|---|
| `NetworkManager.<IE_Start>d__28$$MoveNext` が `NetworkManager$$OnUpdate` を購読する | `Sequencer$$SubscribeUpdate` の呼び出し [0x0202da38] |
| `SubscribeUpdate` がコールバックを `Sequencer._orderableList` (static +0x8) に挿入する | [0x01a8fed0] |
| Unity の `Update` である `Sequencer$$Update` が、`isSuspendUpdate` (static +0xC8) の判定の前に、リスト内の各コールバックを `Time.deltaTime` を渡して 1 回ずつ呼び出す | [0x01a906d8]..[0x01a9071c] |
| `[[this+0x30]+0x60]` が設定されているとき、`NetworkManager$$OnUpdate` が `SessionConnector$$OnUpdate` を呼び出す | [0x02253190] |
| `SessionConnector$$OnUpdate` が `INL1.IlcaNetSession$$Update` を 1 回呼び出す | [0x02030a38] |

`Sequencer$$Awake` は `Application.targetFrameRate` を 30 に設定します [0x01a8ed68]。
1.3.0 の `globalgamemanagers` にある唯一の品質レベル (`Ultra`、Unity 2019.4.27f1) は `vSyncCount` が 2 です。
60 Hz の出力では、これも 1 秒 30 フレームになり、Unity は `targetFrameRate` を無視します。
管理コードは `vSyncCount` も品質レベルも設定していません。`TimeManager` は 1/30 秒の固定タイムステップを持ちます。
`NetworkManager$$OnUpdate` のほかの呼び出し元は `SoftwareKeyboard$$Open` [0x01c980c8] だけです。

認証されていない Pia のパケットは、エラーも出さずに黙って破棄され、席は失われません。

## 通信上のデータ

ステーションが確認応答を返すまで、ホストになっている本体は 100 ms ごとに、
更新セッションを `169.254.x.255:12345` にブロードキャストします。
宛先は `dst_var = 0` です。キャプチャ内のデータグラムはすべて 176 バイトです:

    32ab9864 89 00000000 11bac90d 0000 00 f5a83bd383ce712d 59baa5cbc320cb56 <144 bytes>
    magic    v  dst=0    src      pid  f  nonce (a counter) tag              ciphertext

バージョンバイト `0x89` は暗号化されており、バージョン 9 です。これは Pia 5.27 から 5.45 に対応し、
信頼性プロトコルのバージョン 3 では 5.31 から 5.43 に絞られます。ヘッダー、メッセージのフレーミング、トランスポートプロトコルは [Pia 層](pia.md) にあります。
`pokeldn/ldn/pia5.py` は、キャプチャした 674 個のパケットをバイト単位で一致したまま往復変換できます。

## 鍵の階層

    cryptoKeyDataSeed  9918bd0fdcfa65779918bd0fdcfa6577    from global-metadata.dat
    game key           9900bd0cdcfa65639918bd0fc7fa6577    = seed derived with version 199
    session param      0x36dee059                          advertisement +0x0c, little-endian
    session key        7b182cb087eeabd228a2efd91a8be147    = AES-ECB(game key) over 16 SEAD bytes
    network id         b4c85cf8                            advertisement +0x00, little-endian
    source MAC         48:f1:eb:20:9b:22
    crc32(netid||MAC)  0xda291352
    IV (first packet)  da29130df5a83bd383ce712d

参照キャプチャの 674 個のパケットはすべて認証に成功します。各平文を再暗号化すると、
本体の暗号文とタグがバイト単位で一致します。

### シードの所在

`INL1.IlcaNetSessionSetting` のコンストラクターが既定値を設定します:

    IlcaNetSessionSetting..ctor
      byte[16] cryptoKeyDataSeed  <- RuntimeHelpers.InitializeArray(array, fieldHandle)
      string   wirelessCryptoKey  <- the "WirelessStrongCryptoKey2021" literal
      ulong    localCommunicationId = 0x0100000011d90000

フィールドハンドルは `<PrivateImplementationDetails>.33F804682DF9E210AABDC4D939CBCD380EC7517F` に解決されます。
これは、その 16 バイトの SHA-1 です。
このブロブは `global-metadata.dat` のフィールド既定値のセクションにあり、実行ファイルにも RomFS にもありません。
その手法は [Switch タイトルのリバースエンジニアリング](switch_re.md) にあります。

### 公開されている鍵は、シードから導出される

    seed (metadata)   9918bd0f dcfa6577 9918bd0f dcfa6577
    published key     9900bd0c dcfa6563 9918bd0f c7fa6577
                        ^^   ^^         ^^         ^^        bytes 1, 3, 7, 12

ゲームは、ローカル通信バージョンからバイト 1、3、7、12 を上書きします。1.3.0 のバージョンは 199 です (アドバタイズメントの `app_version`)。
`ldn_game_key(seed, 199)` で、公開されている鍵が再現されます。

### セッション鍵

`nn::pia::local::LocalProtocol` から:

    seed  = a u32 session value held at LocalProtocol+0x5b0
    state = for i in 1..4:  prev = ((prev ^ (prev >> 30)) * 0x6C078965 + i)
    rnd   = four consecutive xorshift128 draws (shifts 11, 8, 19) -> 16 bytes, little-endian
    key   = AES-128-ECB(game key at LocalProtocol+0x5bc).encrypt(rnd)

生成器は SEAD の乱数生成器です。`pokeldn/ldn/sead.py` がこれを実装しています。

### GCM のノンス

IV はストリームオブジェクトが組み立てます。ネットワークの系統ごとに 1 つあります:

    nn::pia::local::LdnOutputStream::vfunc3     0x16b39c4      the LDN sender
    nn::pia::local::LocalOutputStream::vfunc3   0x16bca80
    nn::pia::lan::LanOutputStream::vfunc3       0x16a0f80
    nn::pia::nex::NexOutputStream::vfunc3       0x16eca0c

それぞれ `(this, buf, buflen, packet)` を受け取ります。送信側は `PacketWriter+0x948` で、
受信側は `PacketReader+0xc8` で呼び出します。

    IV[0..3]  = u32be( crc32(ten bytes) )
    IV[3]     = overwritten with (packet.source_variable_id & 0xFF)
    IV[4..11] = the eight-byte header nonce, copied from packet+0x1b

`0x1719204` の CRC は通常の CRC32 (`0xEDB88320`) です。対象は、network id (リトルエンディアン。アドバタイズメントの +0x00 から、
ネットワークオブジェクトの +0x450 を経由して取得) と、ステーションレコードの送信元 MAC です。
送信元 MAC は、復号しているパケットからは復元できません。

## Local Protocol の解読

参照キャプチャのパケットはすべて、同じメッセージを運びます:

    presence 0x7f  flags 0x11  size 121  protocol 36  port 0  destination 0

これは Local Protocol の `0x11` 更新セッションで、
すべてのステーションが確認応答を返すまで、100 ms ごとに再ブロードキャストされます:

    local message header  version 1, type 0x11, size 73
    sequence id           4
    network id            8b4a3b22        random, not the advertisement's network id
    host variable id      11bac90d        the same value as the packet header's source variable id
    host constant id      0000 48f1 2022 9beb
    allow participating   1
    node 0                169.254.54.1:12345          the console
    node 1                169.254.54.2:12345   01     the client
    nodes 2-7             empty, marked 0xff
    host migration state  0

9 バイトのノードスロットが 8 個 (部屋の 8 席)、続いて 1 バイトです。host constant id はリトルエンディアンで読み、
LDN の規則 (`mac[2] << 56 | mac[4] << 48 | mac[5] << 40 | mac[3] << 32 | mac[1] << 24 | mac[0] << 16`) で展開すると、
スキャンした MAC `48:f1:eb:20:9b:22` になります。
Pia のメッセージヘッダーはビッグエンディアン、Local Protocol のフィールドはリトルエンディアンで、その中のローカルアドレスは再びビッグエンディアンです。
プレゼンスバイト 0x7F は、Pia 5.27 から 6.30 のどのフィールドにも対応しない 3 つのビットを立てます。

### 確認応答 (ack)

20 バイトの ack は [Pia 層](pia.md#the-local-protocol-0x24) にあります。動作するフレーミングは、ホスト自身が使うもの、
つまりパケットの `dst_var` 0 とメッセージの宛先 0 のブロードキャストです。ユニキャストのフレーミングは未検証です。
最初の ack で更新は止まります。キャプチャでは、42 回の更新が 100 ms 間隔で送られ、ack は最後の更新から 34 ms 後に届きました。
その後、本体は何も送らず、ステーションもそれ以上何も送りませんでした。

## mesh への参加

3 回のハンドシェイクと ack の規則は [Pia 層](pia.md#joining-a-mesh) にあります。
BDSP のアドレスは次のとおりです:

| 内容 | 場所 |
|---|---|
| station プロトコルの受信ディスパッチャー | `0x0154e848`、テーブル `0x3e6b38f` |
| 接続要求のデシリアライザー | `0x0154ebd0` |
| 内部エラーから結果への変換 | `0x0154f5e8` |
| ID からプロトコルバージョンを引く処理 | `0x0159b850`。未登録の ID では 0 を返す |
| mesh メッセージの ack ID を読む | `0x01542db8` (`size - 4`、ビッグエンディアン) |
| ack を送る (8 バイト、0x14 上) | `0x01550324`、`mov w3, #8` |
| 参加リクエストのハンドラー (ホスト側) | `0x0154b790`、`0x0154b868` |
| 参加レスポンスのハンドラー | `0x0154b984`、`0x0154b9a4` |

`0x01550324` は `session + 0xa0` にある MeshStationProtocol に属します。参加レスポンスのハンドラーは、
参加リクエストを送ったあとに `JoinMeshJob` が `MeshProtocol + 0x128` に格納するポインター (`0x0154e5c4`) です。

本体は 9 つのプロトコルを登録します ([実測](#measurement-methods)):

    0x14 Station v2   0x18 Mesh v3      0x1c SyncClock v0
    0x24 Local v0     0x58 RTT v3       0x68 Unreliable v1
    0x7c Reliable v3  0x94 Session v1   0xa4 MonitoringData v0

バージョン 0 の 4 つは、バージョンの問い合わせでは未登録のものと区別できません。

ステーション 2 つの mesh に対する参加レスポンスです:

    stations 2, host index 0, joiner index 1, max_active 8, update counter 0
    station 0   the console
    station 1   the client: its own station location and ids, read back

1 秒以内に、本体は RTT (0x58) と信頼性 (0x7c) のトラフィックを送ります。
結果 7 は、variable ID がすでに自身のステーションの 1 つであることを意味します。

### Session プロトコル (0x94)

`nn::pia::session::SessionProtocol` (1.3.0 `main`、vtable `0x4b5da50`) は共同セッション機能を担いますが、
LDN では動作しません。Pia のセッション起動処理は、
設定バイト (GOT `0x4c4b850` の +0x38) が設定されていない限りこれを生成 [`0x157c66c`] し、session+0xC8 に格納して、
他のステーションごとに `transport::ReliableSlidingWindow` を 1 つずつ渡します [`0x1581938`]。
そのディスパッチャーのハンドラー [`0x15820b8`、ジャンプテーブル `0x3e6b986`] は、共同セッションのジョブ (session+0x70) が null の場合にすべて戻ります。
LDN では常に null です。`LdnNetworkFactory` のスロット 61 [`0x16b30ac`] が null を返し、
ほかの格納処理も null だけです (`0x157c618`、`0x157d0e8`)。
Lan と Nex のファクトリーは `LanMatchJointSessionJob` と `NexMatchJointSessionJob` を生成します。

そのウィンドウは、ゲームストリームの `ReliableSlidingWindow` [コンストラクター `0x159dab4`] で、
2 スロットのリング、プロトコル 0x94、ポート 1 を使います (`0x159de54(window, 2, 2, 0x94000001)`)。
受信はスロット 10 [`0x1581c98`] で行われます。ウィンドウより前で session+0x70 を読む処理はないため、
有効な信頼性 0x94 メッセージ(最初の 1 通は `is initialized` のフラグ付き、
シーケンスが base に 2 を足した値より小さいもの) は確認応答を返されたあと、ハンドラーで破棄されます。0x94 はまだキャプチャも送信もされていません。

## ホスティング

ユニオンルームに入った本体は、自分の部屋を開く前に、空席のある部屋があればそこに参加します
(`matchingMode` `IlcaNetSessionInitMode.Random`、`localRandomMatchmakeHostWaitTime` 25、`localRandomMatchmakeTimeUp` 270)。
`bin/bdsp_host.py` は部屋を 1 つホストし、ユニオンルームに入った本体はそれに参加します。ホスト側の処理は `pokeldn/bdsp/host.py` にあります。
製品版の部屋のアドバタイズメントは次のとおりです:

    LDN protocol         1 (AES-CTR advertisement)
    frame version        4
    security mode        1
    scene_id             4352 (0x1100)
    app_version          199
    accept policy        ALL, 1/8
    application_data     17 bytes, the Pia header with a fresh network id and session
                         parameter, then one zero byte

ホストが送るものは、いずれも製品版ホストのものをバイト単位で再現したものです (`tests/test_bdsp_host.py`):

| 段階 | メッセージ | フレーミング |
|---|---|---|
| ステーションが接続する | Local Protocol の更新セッション。確認応答があるまで 100 ms ごと。参加者はランク 1 のノード 1 | ブロードキャスト、`dst_var` 0 |
| 接続要求 | 承認された接続レスポンス (949 バイト)。タイプ 2 のリクエストのレイアウトで、ホストの 9 つのプロトコル、ステーションの位置情報、広告したネットワーク ID、PlayerInfo 1 つ (ゼロ埋め)、最後に ack ID を含む | `dst_var` 0、フラグ 0x01 |
| mesh 参加リクエスト | リクエストの ack ID に対するステーションの ack と、続けて参加レスポンス (ステーション 2 つ、`max_active` 8、参加者自身の位置バイトをエントリに含む) | `dst_var` 0 |
| 参加レスポンスの確認応答 | 信頼性ウィンドウ上の `NetJoinData`、シーケンス 1、フラグ 0x0F | 参加者の `dst_var` |
| 以降 | 1 秒ごとの mesh 更新 (556 バイト)、RTT リクエスト、0x68 上の `NetCharacterStateData{0, 0}` | `dst_var` 1、宛先 0xFFFFFFFF |

ホストのステーション位置情報は公開アドレスを持たず、NAT フィールドはゼロで、36 バイトです。参加者のものは両方を持ち、40 バイトです。
ホストのステーションエントリーは、インデックスと参加順序がどちらも 0 で、参加者は 1 です。

参加者は、確認応答を受けた参加レスポンスのあと、約 1 秒ごとに Sync Clock (0x1C) のリクエストを送ります。
ホストは、リクエストのティックと mesh のクロック (ミリ秒) で応答します。
Sync Clock のリクエストに応答が返らない参加者は、画面に "communication en cours" と表示されたまま、
認証解除と再接続を繰り返します。離脱は最初のリクエストの約 10 秒後に起きました。タイムアウトの値は読めていません。
応答が返ると、製品版ホストと同じく `NetJoinData` を送り、0x04 と 0x23 を要求します。
そこからは部屋が対称になり、接近、あいさつ、[交換](bdsp_trade.md) は、本体を参加者としても変更なく動きます。

## 離脱

部屋を離れる本体は、参加者としては Pia の mesh 離脱を、ホストとしてはホスト移行の離脱を実行します。
どちらも mesh プロトコルの信頼性ウィンドウ (0x18、ポート 1) を、シーケンス 1 の信頼性ヘッダーの下で開き、
ポート 0 上で信頼できない方法で送られる応答を待ちます。製品版の Pia は各応答を 2 回送り、
2 回目は別のパケットで送ります。mesh のディスパッチャーは `0x0154ac94` です (ジャンプテーブル `0x3e6b25a`、
タイプ 0x40 から 0x4A は `0x3e6b35c` の背後にあります)。

| 本体 | 送信 | 受け取るべき応答 | 待ちを終わらせるハンドラー |
|---|---|---|---|
| 参加者、`LeaveMeshJob` | `04 <own index>` (離脱リクエスト) | ホストからの `08 <host index>` | `0x0154baf8`。job+0x7c をクリア |
| 参加者、その後 | ステーション切断リクエスト、0x14 上の `03` | `04` | |
| ホスト、`LeaveWithHostMigrationJob` | `44 <host index> <new host index>` を各ステーションに 1 つずつ | 全ステーションからの `48 <station index>` | `0x0154b068`。job+0x6e[index] をクリア |

ホストの離脱リクエストのハンドラー `0x0154b9e0` は、サイズが 2 でないもの、インデックス 0xFD、および自分自身のインデックスを拒否します。
応答は `0x0154c860` を通して返し (`08` とホスト自身のステーションインデックス)、そのステーションを削除します。
`LeaveMeshJob::WaitLeaveResponse` `0x0155fd2c` は、
応答がフラグをクリアするか期限が過ぎると次に進みます。`LeaveWithHostMigrationJob::WaitMigrationResponse` `0x015607bc` は、
存在するいずれかのステーションのフラグが立っている間は待ちます。離脱したステーションは自分のフラグをクリアします。

応答を返さなかった場合の実測:

| 本体 | 最初のメッセージ | その後 | 消失 |
|---|---|---|---|
| ホスト付きの部屋を離れる参加者 (4 回のキャプチャ) | 離脱リクエストを 0.125 秒ごとに 4.9 から 5.0 秒間 | 切断リクエストを 0.5 秒ごとに、3.6 秒から送信 (1 回のキャプチャは最後まで続いた) | そのキャプチャでは、最初のリクエストの 9.0 秒後に認証解除 |
| ステーション 1 つが参加した部屋をホストが離れる (4 回のキャプチャ) | 移行開始を 0.125 秒ごとに 4.9 秒間 | 更新セッション (シーケンス +1、移行状態 1) を 0.11 秒ごとに 10 秒間。続けて Local Protocol 0x13 (ホスト移行の開始) を 0.3 秒ごとに 10 秒間 | 最初の移行開始から 25.3 秒後にネットワークが閉じる |

応答が返った場合 (`08 00` を 2 回と `04`)、離脱する製品版の参加者は、
離脱リクエストの 0.06 秒後に切断リクエストを 1 回送り、その 0.15 秒後に認証解除されました。
応答が返った場合 (`48 01`、更新セッションの ack、0x13 での離脱)、部屋を離れる製品版のホストは、
移行開始の 0.11 秒後と 0.45 秒後に 0x13 を送り (2 回の実行)、ネットワークを閉じました。

0x13 のフェーズは `LocalDestroyNetworkJob::WaitUntilAllClientsDisconnection` `0x016b95b8` です。
前回の送信から 301 ms ごとに 0x13 を送り直します (`job+0x68`)。次のいずれかが最初に起きるとフェーズが終わります。
リクエストのキャンセルバイトが立つ (`WaitForCancel` `0x016b9330` へ進む)。
`0x016b014c` が返す存在するステーションの数が 1 になる (`0x016b9614`)。
前の状態から 10000 ms を超えて経過する (`WaitUntilAllClientsReceiveUpdateSessionMessage` `0x016b936c`、`job+0x70` に格納された値、
`0x016b9644`)。後の 2 つは `StartDestroyNetwork` `0x016b950c` に進み、ネットワークを閉じます。
離脱するステーションがあると、フェーズは次の更新で終わります。残るステーションがあると、フェーズは 10 秒間ずっと保持されます。前の状態は、
`[[protocol+0x4f0]+0x5c]` が設定されたとき (`0x016b0888`)、または 10001 ms 後に抜けます。

`pokeldn.bdsp.session.answer_departure` が両方の応答を組み立てます。`bin/bdsp_host.py` は、
離脱する参加者とその切断リクエストに応答します。`bin/bdsp_connect.py` は、移行開始に応答し、その後のすべての更新セッションに ack を返し、
0x13 でネットワークから離脱します (`--no-leave-on-host-migration` を付けると、ホスト移行時に離脱せず残ります)。

交換ボックスを離れても、離脱メッセージは送られません。
ボックスを閉じるコールバック `TradeSelectPokeModel$$CheckComplete` [1.3.0 main 0x1c26810] は`NetDataCurrentFlowCancelData{0}` (0x25、
`SendCancel` 0x1c26bf0) を送り、
`UnionTradeManager$$Cancel` [0x1c33780] は `NetCharacterStateData{0}` を送ります。両方とも 1 つのパケットにまとめられ、相手の応答は待ちません。

## 実測の方法

- 本体は、接続リクエストに対して、プロトコル数が自分のものと一致する場合にのみ応答します。数を走査すると 9 になります。
  未登録のプロトコルはバージョン 0 を想定するため、バージョン 1 を当てると失敗します。
  二分探索で、どのプロトコルのバージョンも読めます。
- 信号は応答でなければなりません。無応答は、パケットが失われたことと同じくらい頻繁に起きます。
  信頼性ウィンドウは、受け入れたデータに確認応答を返すので、データを送ってシーケンス ID だけを走査します。
