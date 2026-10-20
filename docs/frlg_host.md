---
title: ホスト実装
parent: FireRed and LeafGreen
nav_order: 7
---

# ホスト実装

`bin/frlg_trade_host.py` は FireRed/LeafGreen の Direct Corner リーダーで、Switch は host の
LDN ネットワークに参加し、Pia がピアセッションを確立して、Reliable がエミュレートした親 RFU リンクを運びます。
そのリンクはリーダー側の交換ステートマシンで終わります。`bin/frlg_mg_host.py` は activity より下の
部分をすべて再利用します。

## 構成要素と担当範囲

| 構成要素 | 担当範囲 |
|---|---|
| `TradeRunConfig` | 変更不可の実行設定: `TrainerProfile`, `TradePlan`, `LdnConfig`, `HostOptions` または `JoinerOptions` |
| `HostApplication` | リソースの順序制御: 検証、トランスポートの起動、ビーコンの注入、イベントループ、中断、後始末。activity のフックがメッセージと保存を提供するため、交換とふしぎなおくりものが同じループを共有する |
| `HostTransport` | LDN の AP、仮想インターフェース、参加者イベント、UDP :12345。Pia の解析はしない |
| `HostPeerProtocol` | ピア 1 台分の Pia 状態: Net、Session の受け入れ、RTT、パケット ID、nonce の選択、暗号化、フレーミング、Reliable のバッチ処理。`OutboundDatagram` を出力する |
| `HostSession` | Reliable + `RFULeader` + activity エンジン (`engine=` で交換かふしぎなおくりものを選ぶ)。ソケットは持たず、オフラインの end-to-end テストが使う境界 |
| `HostTradeEngine` | リーダーの部屋入室、手持ちとカードの交換、選択、確認、アニメーションと保存のバリア、キャンセル、部屋からの退出、クローズ後の猶予。`HostTradeTiming` がフレーム数に名前を付ける |
| `BeaconInjector` | 専用の raw モニターソケットとワーカースレッド |

データグラムは Switch, `HostTransport`, `HostPeerProtocol`, `HostSession`, `RFULeader`,
`HostTradeEngine` の順に流れ (子のコマンド行が入り、親の行が出る)、同じ経路で戻ります。タイマーの期限は
ピアプロトコルから得るため、Net/Session の再試行、RTT プローブ、再送、~59.727 Hz のティックは
ポーリングの遅延に依存しません。

## 起動とセッションの確立

1. CLI は `TradeRunConfig` を `HostApplication` に渡し、`HostApplication` は非アクティブな Direct Corner
   ネットワークと定期的なビーコンを起動します。
2. Switch が参加すると `on_participant_joined()` が Pia Net の接続要求を送り、ACK が届くまで再送します。
3. Switch は Net の ACK と Session の参加要求で応答し、host は Session の更新を送ります。
   続けてユニキャストの参加応答を送り (診断用の設定でこの組の順序を逆にできる)、Switch が ACK を返します。
4. RTT と Reliable/RFU のトラフィックが始まり、application-data プロパティがアクティブに設定されます。

不正な Pia、不正なパディング、認証の失敗、Reliable のタイルが不完全なもの、
Session の ID が食い違うものは、ログに記録したうえで無視します。Session の要求は、固定 ID、Pia の変数、
送信元 IP、暗号化ヘッダーがすべて現在の LDN ピアと一致するときだけ受け入れます。

## 交換と部屋退出のライフサイクル

```mermaid
stateDiagram-v2
    [*] --> PlayerExchange
    PlayerExchange --> RoomEntry: LinkPlayer and trainer card
    RoomEntry --> PartyExchange: seat route and entry barriers
    PartyExchange --> Selection: party, mail, ribbons
    Selection --> Confirmation: Switch selects; leader offers configured slot
    Confirmation --> Animation: both confirm
    Animation --> Save: trade committed
    Save --> PartyExchange: another configured trade
    Save --> MenuExit: final party refresh
    MenuExit --> RoomExit: both cancel; five-second field wait
    RoomExit --> CloseGrace: Switch confirms room exit
    CloseGrace --> Disconnect: fifteen seconds of peer traffic
    Disconnect --> [*]
```

最後の交換の後、host は Switch の交換メニューを待ちます。プレイヤーは CANCEL を選び、
YES で確定します。host は `BOTH_CANCEL_TRADE` をすぐに返します。host は待機バリアを終えると、
Switch が先に退出しない限り 5 秒待ってから部屋を出ます。そして Switch がクローズを確定
(`READY_CLOSE_LINK`) した後も、通常のピアトラフィックを 15 秒間保ち、その後 RFU の切断をキューに入れます。
コンソールの Session 離脱要求には種別 4 で応答します
([frlg_link.md](frlg_link.md)、Pia セッションからの離脱)。LDN の離脱イベントが届くと、
ピアへの出力をただちに止めます。

## 終了とクリーンアップ

切断の後、受信した Pokemon があれば保存し、ビーコンのワーカーを止めて join し、
ソケットとネットワークを閉じ、LDN の vif を片付けます。同じクリーンアップは、正常終了、`KeyboardInterrupt`、
一部確保した後の起動失敗、ビーコンワーカーの失敗のいずれでも実行されます。受信した Pokemon の保存は
キャプチャのログ出力とは独立しています。

交換、対戦、ふしぎなおくりものを含むすべての host は、コンソールが LDN から離脱すると停止します
(`HostApplication.run`)。クローズが確定していなければただちに、確定していた場合は 2 秒の
整定時間 (`HOST_CLOSE_SETTLE_SECONDS`) の後に停止します。`--end-on-success` を指定すると、ふしぎなおくりものの
host は配布が成功した後の切断を送信した時点で、コンソールの離脱を待たずに停止します。`--idle-timeout` は
Switch からのトラフィックがなくなってから、指定した秒数の後に停止させます。

## トレーナープロフィールの伝播

`pokeldn.config.DEFAULT_TRAINER` が既定の識別情報です。各 CLI は実行ごとに変更不可の
`TrainerProfile` を導出し (`--ot`、`--version`、10 進数の `--id TID[:SID]`)、Gen III の名前と範囲を検証して、
すべての表示をそこから導出します。

| 表示先 | 持つ情報 |
|---|---|
| LDN の検出 | 名前と公開 TID |
| Pia Session | UTF-8 の参加者名 |
| LinkPlayer | `SID << 16 \| TID`、バージョン、言語、性別と進行状況のフラグ、Gen III の名前 |
| トレーナーカード | Gen III の名前。host は `0xFF` で埋め、参加側はネイティブの `0x00` を保つ |

## 障害時の扱い

- プリフライトは AP 対応のない無線機を拒否します。
- Linux の Wi-Fi カードでは、LDN の認証の後に AP が station に `NL80211_STA_FLAG_AUTHORIZED` を付けなければなりません。
  ドライバーのプロファイルとフラグは [アダプター](hardware_adapters.md) にあります。
- トランスポートまたはビーコンのスレッドの失敗は実行を中止し、作成済みのものを巻き戻します。
- 予期しない参加者の離脱は出力を止めます。部屋のクローズが確定した後に LDN から消えた参加者は、
  ただちにではなく 2 秒の整定時間 (`HOST_CLOSE_SETTLE_SECONDS`) の後に実行を終わらせます。
  15 秒の猶予は、参加者が残っている間だけ最後まで進みます。
- 猶予 (`HostTradeTiming.post_client_close_grace_frames`) は、RFU リーダーが UNI にいる間だけ数えます。
  コンソールが `D` を送るとリーダーは DISCONNECTED に移り (`pokeldn/gba/rfu_leader.py`)、エンジンはティックを止めます。
  実機の FireRed の host 交換 13 回では、コンソールは最初の `READY_CLOSE_LINK` の 0.1 秒後に `D` を送り、
  Pia トランスポートへの応答を続けたまま、その 0.5 から 4.0 秒後に LDN から離脱しました。
  猶予は、通常のクローズでは決して働きません。
- 出力が書かれるのは、受信が完了した Pokemon だけです。入力の `.pk3`/`.ek3` ファイルは決して
  変更しません。

## ホストの拡張

振る舞いの追加は、それを理解できる最も下の層で行います。設定は実行設定に、OS とネットワークは
アプリケーションかトランスポートに、Pia は `HostPeerProtocol` に、RFU は `RFULeader` に、Direct
Corner の判断は `HostTradeEngine` に入れます。各境界は、出力されるデータグラム、ペイロード、フレーム、
コマンド行でテストします。

本ホストは 1 台の Switch に対応します。複数のピアには、ピアごとに `HostPeerProtocol` (それぞれの Pia 変数、nonce、
パケット ID) とゲーム単位の RFU ポリシーが必要です。LDN の参加者数の上限を上げるだけでは足りません。

## ソースマップ

| ソース | 役割 |
|---|---|
| `bin/frlg_trade_host.py` | CLI、設定、エントリーポイント |
| `pokeldn/frlg/host_cli.py` | 共通のホスト CLI オプション |
| `pokeldn/frlg/link/host_app.py` | `HostApplication` |
| `pokeldn/frlg/config.py` | トレーナー、交換プラン、ふしぎなおくりもののペイロード、LDN、役割と実行設定 |
| `pokeldn/frlg/link/trade_runtime.py` | CLI のログ、手持ちの読み込み、スロットの解析、出力の保存 |
| `pokeldn/frlg/link/host_beacon.py` | 捕捉した交換ビーコン、検出の改変、`BeaconInjector` |
| `pokeldn/host_support.py` | OS 向けのサポート (sudo を考慮した鍵パスの解決) |
| `pokeldn/ldn/transport.py` | `HostTransport` |
| `pokeldn/ldn/host_pia.py` | Pia のフレーミング、`HostPeerProtocol` |
| `pokeldn/frlg/link/host_session.py` | `HostSession` |
| `pokeldn/ldn/reliable.py` | Reliable |
| `pokeldn/gba/rfu_leader.py` | `RFULeader`: 親のフレーミング、NI/UNI のハンドシェイク、エコーテーブル |
| `pokeldn/frlg/link/host_trade.py` | `HostTradeEngine`、`HostTradeTiming` |
| `pokeldn/frlg/link/linkplayer.py` | LinkPlayer とトレーナーカードのエンコーダー |
| `pokeldn/ldn/ldntrace.py` | 任意の JSONL 診断 |
