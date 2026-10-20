---
title: アダプター
parent: Hardware and setup
nav_order: 3
---

# Wi-Fi アダプター

Linux ホストは、AP 対応の Wi-Fi カードを root 権限で直接操作できます。これは
[ESP32 無線](hardware_esp32.md) の代わりとなる経路です。この経路は旧来のもので、今後は開発しません。

| 症状 | 原因 |
|---|---|
| AP 起動時にハイパーバイザー経由の USB 切断が起きる | USB モード切り替え (後述) |
| ホストが無反応になる | `accept_decrypted_ccmp` が未設定 |
| `dmesg` に `failed to get tx report from firmware` と出る | ホスト側の終了処理 |

## ldnrs 経由の Windows USB Wi-Fi

Windows では、外付け USB Wi-Fi アダプターを
[ldnrs 0.0.3](https://github.com/unlimitedcoder2/ldnrs/releases/tag/0.0.3) 経由で使えます。ldnrs は別途インストールし、
そのアダプターウィザードを完了させ、デーモンを起動したままにしてください。pokeldn では **Plugins** を開き、
**Radio** の下にある **USB Wi-Fi (Windows, ldnrs)** を選びます。Switch 用の鍵は引き続き pokeldn の Settings から読み込まれます。

ldnrs のウィザードは、選んだアダプターを WinUSB に割り当て、必要な Linux ドライバーの
ファームウェアをダウンロードします。WinUSB がアダプターを占有している間、そのアダプターは通常の
Windows Wi-Fi としては使えなくなります。インターネット接続には別の接続を使ってください。アダプターを
Windows のネットワークに戻すには、デバイスマネージャーで元のドライバーを復元します。互換性は ldnrs における
アダプターのドライバーとファームウェアに依存します。下の Linux カードの測定結果は、Windows での互換性を示すものではありません。

PowerShell では、ゲームの通常のランチャーを実行する前にバックエンドを選びます。

```powershell
$env:POKELDN_RADIO = 'ldnrs:'
python bin/swsh_gift_host.py --keys C:\Switch\prod.keys --species 25 --level 25
```

引数が空の場合は、ローカルパイプ `\\.\pipe\ldnd` が選ばれます。コロンの後ろに別の
ローカルパイプ名を指定することもできます。デーモンは、選ばれた鍵をそのローカルパイプ経由で受け取ります。無線、LDN の
アソシエーション、暗号化はデーモンが担当します。pokeldn は、既存のユーザー空間 IPv4 スタックを通して Ethernet フレームを運びます。
スキャン、参加、ホスト化、アドバタイズの更新は、
[デーモンプロトコル 7](https://github.com/unlimitedcoder2/ldnrs/blob/0.0.3/docs/daemon-protocol.md) を使います。
デーモンが存在しない場合、アダプターの準備ができていない場合、またはプロトコルのバージョンが異なる場合は、エラーになります。

アソシエーションの待ち時間は、デーモンでは 70 秒、ランチャーでは少なくとも 90 秒です。上流の
[RTL8822BU の測定値](https://github.com/unlimitedcoder2/ldnrs/blob/0.0.3/clients/python/ldnd/compat.py) では、
ドライバーの再構成が 2 回あり、それぞれ約 14.5 秒かかります。ゲームのハンドシェイクはアソシエーションの後に始まります。
デーモン 1 つで同時に扱えるセッションは 1 つだけです。pokeldn はセッションが終わると raw チャンネルとネットワークを閉じますが、
外部のデーモンは起動したままにします。パケットキャプチャには、ゲームの UDP トラフィックが残ります。
ESP32 のカウンターと、同梱ライブラリの Wi-Fi 管理フックは適用されません。

オフラインの検査では、スキャン、参加、ホストの各リクエストを、ldnrs 0.0.3 の Python クライアントが生成した
テストベクターと比べます。また、スクリプト化したデーモンを相手に、その raw Ethernet チャンネル経由で UDP をやり取りします。
Windows のアダプターと市販の本体は、このバックエンドではまだ試験していません。

## 検証済みのカード

| モデル | 種類 | ドライバー | 検証したホストでの結果 |
|---|---|---|---|
| TP-Link Archer T3U (`2357:012d`) | 外付け USB | `rtw88_8822bu` | 安定。基準となるアダプター |
| ALFA AWUS036ACHM | 外付け USB | `mt76x0u` | 安定 |
| Realtek RTL8821CE | 内蔵 PCIe | `rtw88_8821ce` | 安定 |
| AMD RZ616 | 内蔵 M.2 | `mt7921e` | 速度は約半分。終了前にデッドロックすることがある |
| MT7601U | 外付け USB | `mt7601u` | 標準ドライバーには AP モードがない。固定版の `mt7601u-ap` DKMS モジュールが必要 ([Raspberry Pi ホスト](hardware_raspberry_pi.md)) |
| Intel AX200 | 内蔵 M.2 | `iwlwifi` | 失敗。IP アドレスを割り当てられない |
| Atheros AR9271 | 外付け USB | `ath9k_htc` | 失敗。通常は IP アドレスを割り当てられない |

## ホストのモード

`--skip-encryption` は、LDN の Python 側の CCMP 処理を省き、CCMP は mac80211 またはハードウェアが 1 回だけ適用するようにします。
通信は無線区間で暗号化されたままです。Archer T3U (`config/host.toml` の `skip_encryption = true`)
と ALFA AWUS036ACHM ではこの設定が必要です。

`--accept-decrypted-ccmp` は、ハードウェアで復号された平文の周りに CCMP ヘッダーと MIC を残す
監視用ドライバー向けです。Archer T3U の `rtw88_8822bu` がこれに当たり、平文を `ldn-tap` に渡す前に MIC を取り除きます。
ALFA は標準のフレームを渡すため、このオプションは無効のままにしてください。

| アダプター | ホストの設定 |
|---|---|
| TP-Link Archer T3U | 追跡対象の `config/host.toml` プロファイル。Wi-Fi 用のフラグは付けない |
| ALFA AWUS036ACHM | `--phy phyN --skip-encryption --no-accept-decrypted-ccmp` |

`phy = "auto"` は、`rtw88_8822bu` の USB `2357:012d` デバイスが 1 台だけあるときにだけ解決されます。0 台または複数台の場合は失敗します。
明示的な `--phy phyN` は常に優先されます。番号は再列挙のたびに変わるため、一度限りのデバッグ用にだけ使ってください。
起動時には、検出されたプロファイル、送信と受信のモード、
そして指定されたフラグが検証済みのプロファイルと異なる場合の警告が表示されます。

## TP-Link Archer T3U

`rtw88_8822bu` は kernel 6.11 以降メインラインに含まれており、この USB ID も登録されています。カーネルがこれをバインドしていることを確認してください。

```bash
modinfo rtw88_8822bu | grep -i 2357
lsusb | grep -i 2357
iw dev
```

### アダプター自体を NetworkManager の管理から外す

NetworkManager は、アダプターのホットプラグされたインターフェース (`wlx...`) を取り込み、バックグラウンドでスキャンを開始します。
インターフェースが現れた直後にチャンネルが変わり、無線が停止します (測定値で 11 秒)。

```text
rtw88_8822bu 3-1:1.0: write register 0x81c failed with -71
usb 3-1: USB disconnect, device number 2
```

アダプターと LDN のインターフェースを除外します。

```text
# /etc/NetworkManager/conf.d/zz-ldn-unmanaged.conf
[keyfile]
unmanaged-devices=interface-name:ldnclient;interface-name:ldn;interface-name:ldn-mon;interface-name:ldn-tap;interface-name:wlx*;interface-name:wlan*
```

このファイルは必ず最後に並ぶ必要があります (`zz-`)。一部のディストリビューションは、後から
`unmanaged-devices=none` を設定するファイルを同梱しているためです。NetworkManager を再読み込みしてください。
`nmcli device status` でアダプターが `unmanaged` と表示されることを確認し、`NetworkManager --print-config | grep unmanaged` で除外リストを確認します。
リストに `ldnclient` がないと、NetworkManager は参加時に作られたインターフェースを掴み、wpa_supplicant をそれに向けてしまいます。
その結果、参加は `[Errno 114] Match already configured` で失敗します。

### ドライバーの USB 3 モード切り替えを無効にする

`rtw88_usb` は既定で `switch_usb_mode=Y` になっており、ドライバーの読み込み後に
アダプターを USB 3 モードで再列挙します。デバイスをパススルーしているハイパーバイザーは切断を検知し、LDN のインターフェースが消えます。
ホストはホスト化の初期段階 (測定で約 1 秒) で停止します。その直前にドライバーのエラーは出ません。

```text
RuntimeError: 802.11 beacon injector stopped: [Errno 100] Network is down
```

このパラメーターの説明にも、USB 3 モードは LDN が使う 2.4 GHz 帯に干渉しうると書かれています。

```text
# /etc/modprobe.d/rtw88-ldn.conf
options rtw88_usb switch_usb_mode=N
options rtw88_core disable_lps_deep=Y
```

その後、アダプターを抜き差ししてください。電源が切れるまでアダプターは USB 3 モードのままです。
抜き差し後は high speed で 1 回だけ接続されます (切り替えが有効な場合は、high speed と SuperSpeed の 2 回列挙されます)。

```bash
cat /sys/bus/usb/devices/*/speed     # 480, not 5000
cat /sys/bus/usb/devices/*/version   # 2.10, not 3.00
```

### 実行前にアダプターのインターフェースを停止する

起動中の管理対象インターフェースは、無線のチャンネルを保持します。`transport.free_radio()` は
それを停止させますが、素の `tools/ldn/ldn_scan.py` は停止させません。

```text
OSError: [Errno 16] Device or resource busy
```

```bash
sudo ip link set wlxXXXXXXXXXXXX down
```

ベースのインターフェースを再び有効にする kill スクリプトがあると、次回の起動は同じエラーで失敗します。
カーネルの `iw scan` 用にそのエラーを出すものは、引き渡す前にインターフェースを再度停止させなければなりません。

### 検証

`./scripts/preflight_pi.sh` は、任意の Linux ホストで実行できます。
