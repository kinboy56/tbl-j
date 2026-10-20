---
title: Pi への Switch keys の配置
parent: Hardware and setup
nav_order: 2
---

# Raspberry Pi への Switch keys のインストール

`prod.keys` は、ライブ LDN ホストに必要な、ご自身の Switch の認証情報です。これをコミットしない、
設定に貼り付けない、キャプチャに含めない、またはデスクトップの仮想環境を
Pi にコピーしない。デプロイでも転送されません。`/home/PI_USER/.switch/prod.keys` に置き、Pi の
無視対象の `config/host.local.toml` でこの絶対パスを指定します。ホストは `sudo` の下で動くため、`~` は
ログインアカウントに解決されないからです。次のとおりです。

```toml
[ldn]
keys_path = "/home/PI_USER/.switch/prod.keys"
```

## Pi でローカルにインストールする

Pi のログインユーザーとしてインストーラーを実行します (`sudo` の下では `$SUDO_USER` として再起動します)。作成されるのは
`~/.switch` (モード `0700`) です。ファイルはモード `0600` でインストールされ、内容は表示されません。再実行時は
同一のファイルならそのまま残し、権限を修復します。ソースは絶対パスで指定する通常ファイルで、
ログインユーザーが読み取れる必要があります。

```bash
cd ~/pokeldn
./scripts/install_switch_keys.sh --source /absolute/path/to/prod.keys
```

`--stdin` を使うと、ステージング用のコピーを作らずに SSH 経由で鍵を流し込めます。

```bash
# On the desktop; pi-ldn is your SSH alias.
ssh pi-ldn 'cd ~/pokeldn && ./scripts/install_switch_keys.sh --stdin' < "$HOME/.switch/prod.keys"
```

## SSH エイリアス経由の SCP

一時コピーは `/tmp` のような共有ディレクトリではなく非公開のディレクトリに置き、
作業後にコピーを削除してください (`scp -p` はローカルのモードを保持します)。

```bash
# On the desktop.
ssh pi-ldn 'install -d -m 700 "$HOME/.frlg-ldn-provision"'
scp -p "$HOME/.switch/prod.keys" pi-ldn:.frlg-ldn-provision/prod.keys
ssh pi-ldn 'cd ~/pokeldn && \
  ./scripts/install_switch_keys.sh --source "$HOME/.frlg-ldn-provision/prod.keys" && \
  rm -f "$HOME/.frlg-ldn-provision/prod.keys" && \
  rmdir "$HOME/.frlg-ldn-provision"'
```

どちらの方法でも、次のコマンドで確認します。

```bash
ssh pi-ldn 'stat -c "%a %U %n" "$HOME/.switch" "$HOME/.switch/prod.keys"'
# Expected: 700 PI_USER /home/PI_USER/.switch
#           600 PI_USER /home/PI_USER/.switch/prod.keys
```
