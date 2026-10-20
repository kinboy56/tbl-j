---
title: 無線レイヤー
nav_order: 2
has_children: true
---

# 無線レイヤー

Switch は近くのコンソールと **LDN** (Nintendo の無線通信) で通信し、その上で
[Pia](pia.md) (Nintendo のピアツーピアのセッション用ミドルウェア) を使います。どちらもシステムライブラリです。
タイトルによって Pia のバージョンとペイロードが異なります。

## 2つの秘密情報

LDN のパスフレーズは 802.11 の関連付けを認証します。長さは 16-64 バイトで、
(`nn::pia::local::LdnBackgroundProcessJob` が検査)、そのまま使われます。ブリリアントダイヤモンドは ASCII 文字列を
`nn::ldn::CreateNetwork` にそのまま渡します。16 バイトの Pia ゲームキーから、
すべてのデータグラムを暗号化するセッション鍵が導出されます。

| タイトル | LDN パスフレーズ | Pia ゲームキー |
|---|---|---|
| ブリリアントダイヤモンド・シャイニングパール | `WirelessStrongCryptoKey2021` (27 バイト、生の値) | `cryptoKeyDataSeed` から導出。[BDSP](bdsp_session.md) を参照 |
| ソード・シールド | `W3GoSMEn7RIIUQ89rzqBHGhGferRNb7K18ZBq2aNuj8Us9RO9Q9JYyGOZlLy8MYL` (64 バイト、生の値) | `p1frXqxmeCZWFv0X` |

スカーレット・バイオレットと Pokemon LEGENDS アルセウスは、ソード・シールドのパスフレーズとゲームキーを使います
([スカーレット・バイオレット](sv.md), [Pokemon LEGENDS アルセウス](pla.md))。NintendoClients wiki にある Pokemon LEGENDS アルセウス向けの
`HGHG` という表記は誤りです。

## 検出

アドバタイズメントの読み取りには `prod.keys` だけが必要です。`tools/ldn/ldn_scan.py` で任意のセッションの
`local_communication_id`、`scene_id`、バージョン、チャンネル、受け入れポリシー、参加人数、
アプリケーションデータを表示できます。タイトルが見られるのは自分の LDN プロトコルのアドバタイズメントだけです (1: `master_key_00` の下の AES-CTR、
3: `master_key_12` の下の AES-GCM) ので、ホストはタイトルのプロトコルを
(`HostTransport(protocol=...)`) でコピーします。

| タイトル | プロトコル | アドバタイズメント形式 |
|---|---|---|
| GBA アプリ (ファイアレッド・リーフグリーン)、comm id `0x01006fa0233f8000` | 3 | 3、AES-GCM |
| ソードのふしぎなおくりもの画面、comm id `0x0100abf008968000` | 1 | 2、AES-CTR |
| Pokemon LEGENDS アルセウスのローカル交換、comm id `0x01001f5010dfa000` | 1 | 4 |

ホストのコンソールは、ビーコンを 11 Mbit/s の DSSS で、アドバタイズメントのアクションフレームを HT MCS 3、20 MHz
(OFDM) で送信します。スカーレット・バイオレットの通信交換の検索も、ソードのふしぎなおくりもの画面も同じです。DSSS のみの受信機は
アクションフレームを取りこぼします。

## アドバタイズメントのアプリケーションデータ

Pia 5 のレイアウト (シャイニングパール由来):

    +0x00  4  network id                     random per session
    +0x04  4  CRC32 of the user password     0 when the room has no password
    +0x08  1  system communication version
    +0x09  1  header size                    16
    +0x0a  2  padding
    +0x0c  4  session parameter              random per session; seeds the Pia session key
    +0x10     application data

キャプチャは、そのセッション自身のアドバタイズメントでしか復号できません。別のセッションのものでは、エラーなしに失敗します。

Pia 6.16 から 6.41 は、システムプロパティブロックを使い、セッション鍵とネットワーク ID を
SSID から導出します:

    +0x00  2  system property data size      0x5C
    +0x02  1  system communication version   21 for 6.16-6.30, 22 for 6.39-6.41
    +0x03  2  application communication version
    +0x05  16 user password
    +0x15  1  is player limit enabled
    +0x16  1  number of players
    +0x17  4  player name size
    +0x1b  1  player name encoding           1 = UTF-8, 2 = UTF-16
    +0x1c  64 player name
    +0x5c     application data

通信の暗号化が有効なとき、ユーザーパスワードは 0xFE で 16 バイトまでパディングされ、
その場で AES-128-GCM でゲームキーを使って暗号化されます (タグは破棄、IV は `key[1] key[8] key[7] key[2]`)。その後、
ブロックビルダー (Pokemon LEGENDS アルセウス 1.1.1 の `0x6fac70`、[設定処理](pla.md#the-link-code-in-the-advertisement)) でコピーされます。
GCM の 1 ブロックは、鍵で決まるキーストリームとの XOR です。そのため、ゲームキーだけでパスワードが読めます。

## エミュレーター向けのホスティング

ldn_mitm モードのエミュレーターはポート 11452 で関連付けを行い、その後 LAN 上で Pia を動かします。無線は使わず、
`prod.keys` も root も不要です。`pokeldn/ldn/ldn_mitm.py` が参加し、`ldn_mitm_host.py` は
`IpHostTransport` を通してホストします。`NEEDS_RADIO = False` によって `NullBeaconInjector` が選ばれます。

    UDP  console -> host:11452   Scan          header only, unicast and broadcast
    UDP  host    -> console      ScanResp      NetworkInfo, 0x480
    TCP  console -> host:11452   Connect       NodeInfo, 0x40
    TCP  host    -> console      SyncNetwork   NetworkInfo with the console seated, held open

`nn::ldn::NetworkInfo` (0x480 バイト):

    +0x000  NetworkId          IntentId 0x10 (u64 localCommunicationId, u16, u16 sceneId, u32) then SessionId 0x10
    +0x020  CommonNetworkInfo  MAC 6, Ssid 0x22 (length byte then 0x21), s16 channel, u8 linkLevel, u8 networkType, u32
    +0x050  LdnNetworkInfo     SecurityParameter 0x10, u16 securityMode, u8 acceptPolicy, ...
    +0x066                     u8 nodeCountMax, u8 nodeCount
    +0x068                     NodeInfo[8], 0x40 each: u32 IPv4 little-endian, MAC 6, u8 nodeId,
                               u8 isConnected, 0x20 name at 0x0C, u16 localCommunicationVersion at 0x2E
    +0x26A                     u16 advertiseDataSize, then 0x180 bytes of advertise data
    +0x478                     u64 authenticationId

ノードは実際の LAN アドレスを持ちます (Pia はトンネルされません)。ノード 0 は到達可能でなければならず、リンクローカルアドレスは使えません:
コンソールはそこへ Pia を送信し、TCP 接続を開きます。エミュレーターは、ノードの MAC をそのアドレスから導出します
(172.16.86.1 の場合は `02:00:ac:10:56:01`)。ゲーム自身のノードは +0x2E に 88 を保持します (0x2C のバイトの後で整列)。これは
`ConnectImpl` が渡す値で、ゲームは NetworkInfo を送られたとおりに保持します。ファイアレッドのスキャンフィルターが比較するのは
`localCommunicationId` と `networkType` だけです (`sceneId`
0xFFFF、`ssidLength` 0)。

16 バイトのセッション ID は、3 か所で同じ 1 つの値になります: `NetworkId.SessionId`、16 進で表した `Ssid`、
そして Pia セッション鍵 `AES(game_key).encrypt(ssid)` の平文です。ネットワーク ID は
`crc32(ssid[1:16])` です ([`pokeldn/ldn/crypto.py`:114])。LDN ライブラリも同じように処理します
([vendor/LDN/ldn/__init__.py:1921, :1910])。一方を広告し、もう一方で暗号化すると、関連付けは成立し、
その後コンソールはすべてのデータグラムを破棄してタイムアウトしますが、エラーは出ません。

## ステーションのブロードキャスト

参加したコンソールは、ブロードキャストとマルチキャストを BSS へ直接送ります (DS ビットなし、アドレスはグループ、
コンソール、BSSID。CCMP グループ鍵、鍵 ID 1)。ユニキャストは to-DS (ペアワイズ鍵、鍵 ID 0) で送ります。ファイアレッドの最初の
ブロードキャストは、ホスト向けの ARP と IPv6 マルチキャストです。標準のアクセスポイントは DS ビットのない
フレームを破棄し、ARP に応答しないホストは理由 3 で認証解除されます。[ESP32](hardware_esp32.md) は
これらのフレームを加工せずそのまま転送します (`vendor/LDN/ldn/__init__.py`
`_process_data_frame` が従来の Linux 経路向けにそれらを復号します)。

## チャンネル

LDN では 5 GHz の 36/40/44/48 チャンネルも使えますが、ESP32 はこれに届きません。ファイアレッド・リーフグリーンのスキャンは
2.4 GHz のみです。ホストを作り直したコンソールはチャンネルが変わることがあり、ボードのスキャンはそれを表示します:

    POKELDN_RADIO=esp32:auto ./.venv/bin/python tools/ldn/ldn_scan.py --keys PROD_KEYS --dwell 2.5

アドバタイズメントは隣接チャンネルにも漏れます。ボードは、ホストのアドバタイズメントを自分のチャンネルの両隣でも
受信しますが、数はずっと少なく、RSSI は同じです。隣接チャンネルで参加すると、
関連付けは成立しますが、ホスト自身のチャンネル上の次のアドバタイズメントが、非互換なネットワークとしてリンクを切断します。
そのため、スキャンは最も混雑しているチャンネルを報告します。
