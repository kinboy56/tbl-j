---
title: Raspberry Pi ホスト
parent: Hardware and setup
nav_order: 4
---

# Raspberry Pi 4 ふしぎなおくりものホスト

64 ビットの Raspberry Pi 4 に TP-Link Archer T3U / AC1300 (`2357:012d`, `rtw88_8822bu`) を接続すると、
ふしぎなおくりものをホストできます。パッチ済みの LDN 実装は `vendor/LDN` です。パッチを当てていない PyPI パッケージは、
Pi にインストールしないでください。アダプターのカーネル設定は [アダプター](hardware_adapters.md) にあります。

デプロイでは、コミット済みの Git オブジェクトを SSH エイリアス経由で Pi 上のベアリポジトリへ送ります。GitHub は使わず、
`rsync` も使わず、無視されるファイル (参照用リポジトリ、`.venv`、キャプチャ、Pokemon のファイル、
Switch の鍵) は送りません。

## 初回デプロイ

デスクトップ PC で、SSH エイリアスと Pi のログインユーザーを指定して、次のコマンドを実行します。

```bash
git add -A
git commit -m "Prepare Raspberry Pi deployment"
./scripts/deploy_pi.sh --host pi-ldn --user PI_USER
```

`deploy_pi.sh` は、未コミットの変更があるデスクトップ側のチェックアウトを拒否します。その後、設定とドキュメントのテストを実行し、
必要なら `/home/PI_USER/repos/pokeldn.git` を作成してから、現在のコミットを `deploy` ブランチへ push し、
`/home/PI_USER/pokeldn` を作成するか fast-forward します。強制リセットは行わず、未コミットのファイルがある Pi 側のチェックアウトも拒否します。
SSH エイリアスですでにリモートユーザーを指定している場合は、次のようにパスを明示してください。

```bash
./scripts/deploy_pi.sh --host pi-ldn \
  --path /home/PI_USER/pokeldn \
  --repo /home/PI_USER/repos/pokeldn.git
```

続けて Pi を初期セットアップします。

```bash
ssh pi-ldn
cd ~/pokeldn
./scripts/setup_pi.sh
```

`setup_pi.sh` には 64 ビットの Raspberry Pi OS (`aarch64`) と Python 3.11 以降が必要です (Bookworm は 3.11、Trixie は 3.13)。
`requirements.txt` から仮想環境を作成し、システムツールをインストールします (`--no-apt` を付けるとスキップします)。
また、`ldnclient`、`ldn`、`ldn-mon`、`ldn-tap` を NetworkManager の管理対象から除外します
(`--no-networkmanager` を付けるとスキップします)。SSH は Ethernet か内蔵 Wi-Fi で接続してください。ホストが
TP-Link アダプターを専有するためです。

## 設定と Switch の鍵

Git で管理している `config/host.toml` は、TP-Link のライブプロファイルです。

```toml
[host]
live = true
skip_encryption = true
accept_decrypted_ccmp = true

[ldn]
phy = "auto"
```

Git で無視される `config/host.local.toml` には、鍵ファイルの絶対パスを書きます (ホストは `sudo` の下で動作します)。
`prod.keys` のインストール方法は [Pi 上の Switch 鍵](hardware_switch_keys.md) を参照してください。

## 検証と実行

TP-Link アダプターを接続した状態で、次を実行します。

```bash
cd ~/pokeldn
./scripts/preflight_pi.sh
./scripts/run_mystery_gift.sh
```

preflight (事前確認) は読み取り専用です。Python と同梱の LDN パッケージを確認し、`config/host.toml` と
任意の `config/host.local.toml` を読み込みます。TP-Link の USB ID と `rtw88_8822bu`、AP モードとモニターモードへの対応を確認し、
NetworkManager の除外設定と、鍵ファイルのモードが `0600` であることも確認します。`skip_encryption` や `accept_decrypted_ccmp` を無効にする TP-Link の呼び出しは、
`run_mystery_gift.sh` 経由で渡された引数も含めて拒否します。
Debian の `sbin` パスは preflight 自身が追加するため、
シェルから実行しても、1 行の SSH コマンドから実行しても同じように動作します。

`run_mystery_gift.sh` は最初に preflight を 1 回実行してから、短命な root のホストプロセスを監視します。各プロセスは
ランダムな TID/SID を持ち、配信後、試行の失敗後、または 5 分間 Switch の参加も Pia/RFU の通信もなかったときに再起動されます。
Ctrl-C を 1 回押すと停止します。常に `--end-on-success` (配信後のクローズのあとに終了) と、
`--idle-timeout 300` (この秒数 Switch の通信がなければ終了) を渡します。`--id` も自分で管理するため、
保存された `--id` を使って古い ID を再利用することはありません。`bin/frlg_mg_host.py` のそのほかのオプションは
そのまま転送されます。`--help` と `--print-effective-config` は、preflight も root も必要ありません。

参加した試行はそれぞれ、Git で無視される日次の台帳 `logs/mystery-gift-attempts-YYYY-MM-DD.csv` に追記されます。
列は `attempt`、`received_result` (ワンダーカードかスタンプが送られたときだけ `true`)、`time`、
`trainer_name`、`trainer_ot` (Switch の LinkPlayer ブロックにある名前と 5 桁のトレーナー ID)
です。試行が途中で失敗した場合は空欄になります。`bin/frlg_mg_host.py --attempt-log-dir logs` を指定すると、
同じ台帳を書き出せます。

イベント関連のオプションは、`--gift`、`--flag-id`、`--verbose`、`--capture`、`--ot`、`--version`、`--id` です。
`--client-ready-idle-frames N` はハードウェア試験用のタイミング診断です。通常は指定しないでください。
`--make-artifact` を付けると、`artifacts/` の下に決定的な `.ram.lst` の一覧を書き出します (`--artifact-dir DIR` で
出力先を変えられます)。一覧には、コンパイル済みの RAM スクリプトのバイト列、デコード済みの命令、チェックサム、分岐先と
メッセージの送り先、配信段階の要約が含まれます。保存済みのコマンドで無効にするには `--no-make-artifact` を使います。

```bash
./scripts/run_mystery_gift.sh --gift solrock-stamp --client-ready-idle-frames 45 \
  --capture solrock-45.jsonl
./scripts/run_mystery_gift.sh --gift worlds-xp --make-artifact
```

他のハードウェアを診断する場合を除き、`--phy`、`--adapter`、`--skip-encryption`、`--accept-decrypted-ccmp` は
TP-Link の既定値のままにしてください。ALFA の `mt76x0u` では、現在の PHY (`iw dev` で確認します。再接続すると変わります) と
`--no-accept-decrypted-ccmp` を指定してください。このとき preflight は、その PHY の AP モードとモニターモード、
および `mt76x0u` の CCMP プロファイルを確認します。

### 独自の AP ドライバーを使う MT7601U アダプター

標準の `mt7601u` ドライバーには AP モードがありません。ホストするには、バージョンを固定した `mt7601u-ap` DKMS モジュール
(`vendor/mt7601u-ap-1.0`) が必要で、Pi の ARM64 カーネル向けに Pi 上でビルドします。デスクトップ PC でビルドした `.ko` を絶対にコピーしないでください。
デスクトップ PC から、次を実行します。

```bash
./scripts/deploy_pi.sh --host pi-ldn --user PI_USER --install-mt7601u-ap
```

または Pi 上で、次を実行します。

```bash
cd ~/pokeldn
./scripts/setup_pi.sh --install-mt7601u-ap --no-networkmanager
```

この処理では `dkms` と `linux-headers-rpi-v8` をインストールし、インストール済みのすべてのカーネルに対して
ヘッダーが一致するものにモジュールを登録します (APT は再起動の前に、より新しいカーネルをインストールすることがあります)。実行中のカーネルにヘッダーがない場合は
停止します。アダプターを挿し直すか再起動してから、`iw dev` で PHY を確認し、`--phy phyN
--no-accept-decrypted-ccmp` を付けて実行してください。preflight は、読み込まれた `mt7601u` が `updates/dkms` から来ていて、
AP モードとモニターモードに対応していることを確認します。

## 変更のデプロイ

デスクトップ PC でコミットし、`./scripts/deploy_pi.sh --host pi-ldn --user PI_USER` をもう一度実行します。このデプロイが fast-forward するのはコードだけです。
依存関係のファイルか `vendor/LDN` が変わったときに、Pi は仮想環境を更新します。push 後に手動で更新するには、Pi 上で
`./scripts/update_pi.sh` を実行します。Pi 上で Git 管理のファイルを編集してはいけません。
`config/host.local.toml`、鍵、キャプチャは Pi 上だけに置き、Git では無視されたままにしてください。
