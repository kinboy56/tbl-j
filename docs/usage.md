---
title: コマンドラインの使い方
---

# コマンドラインの使い方

コマンドはリポジトリのルートから実行します。Switch と通信するツールには Python 3.11 以上と Switch の `prod.keys`
(既定は `~/.switch/prod.keys`、または `--keys PATH`) と、書き込み済みの ESP32 ボードが必要です。
詳細は [対応ボード](hardware_esp32.md#supported-boards) と
[ビルドと書き込み](hardware_esp32.md#building-and-flashing) を参照してください。

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
dotnet build -c Release services/pkhex
export POKELDN_RADIO=esp32:auto
python -m pokeldn --list
python -m pokeldn --radio esp32:auto swsh-host --offer-file offer.pk8
```

交換ツールには PKHeX 補助ツール用の .NET 10 SDK が必要です。同梱の LDN ライブラリは
`requirements.txt` からインストールしてください。PyPI の `ldn` パッケージで代用しないでください。Windows では
PowerShell で `.venv\Scripts\Activate.ps1` を実行して venv を有効にし、
`$env:POKELDN_RADIO = "esp32:auto"` で無線を設定します。

`esp32:auto` は接続されている USB シリアルポートが 1 つだけならそれを選びます。明示的に選ぶには `esp32:PORT` を使います。
無線の設定と診断は [無線の設定と診断](hardware_esp32.md#running) を参照してください。設定ファイルと既定の出力先は
作業ディレクトリを基準に解決されます。各エントリポイントは `--help` を受け付けます。

## ファイアレッド・リーフグリーン

本体で Direct Corner を解放し、少なくとも 2 体の `.pk3` のパーティポケモンを用意してください。

```bash
./.venv/bin/python bin/frlg_trade_join.py --live -o output.pk3 PARTY1.pk3 PARTY2.pk3   # join the Switch's trade
./.venv/bin/python bin/frlg_trade_host.py -o output.pk3 PARTY1.pk3 PARTY2.pk3          # host a Direct Corner trade
```

ホストはグループを告知し、交換を進行します。既定では `PARTY2.pk3` を提示し、受け取ったものを
`output.pk3` に書き出します。既定値は `config/host.toml` から読み、続けて
Git の管理対象外の `config/host.local.toml` から読みます。フラグはどちらより優先され、`--print-effective-config` で結果を表示します。

1. ホストを起動し、`Hosting Direct Corner` と表示されるまで待ちます。
2. Switch で Direct Corner、Join Group の順に選び、pokeldn のトレーナーを選びます。
   ホストが交換の選択が有効になったと報告するまで待ちます。
3. 交換に出すポケモンを選び、確定します。
4. 本体は交換ごとに交換メニューへ戻ります。`--trades N` (1 から 6) を指定すると、同じ接続で
   パーティの 0 から N-1 番目の枠を、交換ごとに 1 体ずつ提示します。最後の交換のあと、ホストの指示を待ち、
   **CANCEL**、**YES** の順に操作します。部屋からの退出と切断は自動で続きます。

| フラグ | 用途 |
|---|---|
| `--keys PATH` | `prod.keys` の場所 |
| `--slot N` | 提示するパーティポケモン (0 始まり) |
| `--capture FILE` | JSONL 形式の診断ログを保存 |
| `--config` / `--local-config` / `--no-local-config` | 設定の層を置き換えるか無効化 |
| `--ot NAME`, `--version firered\|leafgreen`, `--id TID[:SID]` | 実行ごとのトレーナー上書き (それぞれ 0..65535; LinkPlayer ID は `(SID << 16) \| TID`) |
| `--verbose` | パケットごとの出力。フレーム時間を計るループ内で同期的に記録されるため、`--replay` と組み合わせてのみ使う |

[`pokeldn/config.py`](../pokeldn/config.py) の `DEFAULT_TRAINER` には、フラグがないときの既定値が
入っています (性別、言語、全国図鑑)。リンクプロトコルは [リンクプロトコル](frlg_link.md) にあります。

**ユニオンルーム。** `--union-room` は中央の NPC の経路で告知します。本体は
キープアライブの待ち時間 (約 10 秒) のあと接続済みと表示します ([リンクプロトコル](frlg_link.md))。ユニオンルームの
接続では交換を 1 回行い、その後本体はフィールドに戻ります (`union_room.c:1744`)。
`--board-type normal` は提示するポケモンを交換ボードに登録します。`--union-room-chat` は
`--chat-message` / `--chat-file` と組み合わせてチャット、`--union-room-battle --battle-fight` でバトルを行います (本体にはタマゴではないレベル 30 以下のポケモンが 2 体必要です)。

```bash
./.venv/bin/python bin/frlg_trade_host.py --union-room --union-room-keepalive 120 PARTY1.pk3 PARTY2.pk3
```

**ふしぎなおくりもの。** `bin/frlg_mg_host.py` はフレンド経路で告知し、ふしぎなカードと
配信スクリプトを送ります。Switch では **ふしぎなおくりもの → ふしぎなカード → Friend** を選んで pokeldn のホストに接続し、
セーブデータでふしぎなおくりものが解放されている必要があります。`--make-artifact` は送ったバイト列の `.ram.lst` 一覧を書き出します。
カタログ、作成システム、フレンド経路の要件は [ふしぎなおくりもの](frlg_gift.md) にあります。

```bash
./.venv/bin/python -u bin/frlg_mg_host.py --gift beast-cutscene --flag-id 1005 --capture mystery-gift.jsonl
```

ビーストのカットシーンは、セーブデータの御三家に応じて変わります (フシギダネならスイクン、ゼニガメならエンテイ、ヒトカゲならライコウ)。共有の
スタンプラリーのカードは `--gift solrock-stamp` と `--gift lunatone-stamp` で、どちらの順でも指定できます。

**ふしぎなニュース。** `--news` は本体のふしぎなおくりものの 2 つ目の列 (**ふしぎなおくりもの → ふしぎなニュース → Friend**) に応答します。
444 バイトのタイトルと本文で、ハナダシティできのみの報酬を与えます。本体は
保持しているものと異なる場合にだけニュースを保持します。`--news-id N` で新しいニュースを強制できます。

```bash
./.venv/bin/python -u bin/frlg_mg_host.py --news berry --news-id 7
```

**セーブデータのバックアップと復元。** 同じフレンド経路で、128 KB のセーブデータ全体を `.sav` ファイルにコピーするか、
`.sav` を書き戻します。本体自身のセーブデータの横で、すべてのセクタを読み戻し、その後ゲームがそれを読み込んで
保存します。ここまで完了しなければ、本体のセーブデータは元のままになります。アプリでは、ふしぎなおくりもののツールの
「Your save」タブでバックアップを保持・命名し、`.sav` ファイルを読み込み・書き出しし、PKHeX でトレーナーとパーティを
編集します。バックアップは、フランス語版の市販 FireRed で約 4 分かかりました。復元は、
英語版の市販 FireRed で CONTINUE を使って読み込んだ場合で確認しました (フランス語版のセーブデータを含む)。[セーブデータのバックアップと復元](frlg_gift.md#save-backup-and-restore)

```bash
./.venv/bin/python -u bin/frlg_mg_host.py --live --save-backup backup.sav --save-resume-dir partial
./.venv/bin/python -u bin/frlg_mg_host.py --live --save-restore backup.sav
```

**本体のセーブデータ。** ふしぎなおくりもののセッションでは、本体上でネイティブの ARM コードを実行します。`save-dump` は
稼働中のセーブデータを読み出します (ひみつのID、パーティポケモンそれぞれの PID、個体値、性格)。何も書き込みません。
`flash-patch` は 1 つのフィールドを編集します。セーブセクタを読み、指定したバイトだけを変更し、チェックサムを再計算して
セクタを書き戻し、ゲームがそれを読み込むようカウンタを進めます。実際のセーブデータを編集するため、まず
[RAM スナップショットからのセクタの組み立て](frlg_rom.md#composing-a-sector-from-a-ram-snapshot) を読んでください。ペイロードは [本体上のコード](frlg_rom.md) にあります。

```bash
./.venv/bin/python -u bin/frlg_mg_host.py --buffer-script save-dump --dump-block sav2 --dump-size 64 --dump-file dump.bin
./.venv/bin/python tools/frlg/dump_read.py dump.bin --block sav2
./.venv/bin/python -u bin/frlg_mg_host.py --buffer-script flash-patch --flash-id 0 \
  --flash-patch-offset 0x00 --flash-patch-hex cac9c5bfc6bec8ff --write-unsafe
```

## Let's Go ピカチュウ・イーブイ

交換画面はホストと検索を交互に行うため、pokeldn はホストにも参加側にもなれます。どちらも種類 1 の
識別メッセージを送ります。参加側は `pokeldn.lgpe.reference` に同梱のもの、ホストは本体の
応答 (`--first echo`) かファイルを使います。どちらも 232 バイトの PB7 (`pokeldn.lgpe.pb7`。`--offer echo` は本体自身のものを返します) を提示します。
ホストの `--next-offer` と、参加側の繰り返しの `--offer` は、同じ席の交換ごとに 1 件ずつキューに入れます。
参加側の `--leave-after S` は、最初の交換ステップに応答してから S 秒後に
席を離れます。プレイヤーが交換画面を離れるのと同じ動きです。指定しなければ、席は後続の交換のために
残ります。

```bash
./.venv/bin/python bin/lgpe_host.py --seconds 600 --player-name POKELDN \
  --first echo --our-trainer 41234:12345 --offer offer.pb7
./.venv/bin/python bin/lgpe_join.py --connect --connect-seconds 300 \
  --ack-peer-clock --ack-re-announce \
  --our-trainer 41234:12345 --offer offer.pb7
```

本体では Communiquer、Communication locale、Échange を選び、リンクコードはピカチュウ ×3 を入力して、
検索画面で待機します。LAN 上のエミュレート本体には `lgpe_host.py --ip-host --our-ip IP` を使います。詳細は
[Let's Go](lgpe.md) を参照してください。

## ソード・シールド

本体では Y-Comm → Link Trade → local communication の順に選び、両方のメッセージで A を押して、検索画面で待機します。

```bash
# join the console's session and trade: the console's own party snapshot is sent back, rewritten
POKELDN_RADIO=esp32:auto ./.venv/bin/python bin/swsh_connect.py --keys PROD_KEYS \
  --preset trade --offer-slot 1 [--offer-file your.pk8 --offer-file next.pk8 --fresh-pid] \
  --save-offered received.pk8

# or host, and let the console join: its snapshot is taken from this trade
POKELDN_RADIO=esp32:auto ./.venv/bin/python bin/swsh_host.py --keys PROD_KEYS \
  --offer-file your.pk8 --fresh-pid --received received.pk8 --channel 6 --seconds 900
```

ホストは自身の局の告知を作り、参加してきた本体のライブのスナップショットを書き換えます。
ホストするとき、本体は Y-Comm → Link Trade → trade の順に進み、続く両方のメッセージで A を押して参加します。
`--received FILE` は送られた内容を保存し、`--code 12345678` は Link Code 検索用にホストします。
`--advert` と `--snapshot` は、比較用に保存済みのレコードを引き続き受け付けます。ホストと参加側はどちらも、セッション内の交換ごとに 1 つずつ、
繰り返し指定される `--offer-file` を受け付けます。プレイヤーがボックスから選び直すたびに対応します。
交換 N では、受け取った内容を `-N` 付きで書き出します。詳細は [交換](swsh_trade.md) を参照してください。

ふしぎなおくりものはセッションを必要とせず、ギフト画面がスキャンし、配布側がカードを告知します。本体では
ふしぎなおくりもの → ギフトを受け取る → ローカル無線 の順に選びます。

```bash
./.venv/bin/python bin/swsh_gift_host.py --species 25 --level 25 \
  --move1 84 --move2 45 --move3 86 --move4 98 --nickname POKELDN --ot POKELDN --seconds 300
./.venv/bin/python bin/swsh_gift_host.py --record card.wc8 --seconds 300
```

`--set FIELD=VALUE` は任意のレコードのフィールドを設定します (`shiny_type=3`、`ball=1`、`held_item=236`、`iv_hp=31`、
`gigantamax=1`)。`--ribbon N` はリボンを追加し、`--dump FILE` は無線を使わずにレコードを書き出します。道具の ID は
道具の一覧で確認されます。行のない ID は受け付けられ、そのあと道具の欄を表示するときにゲームが中断します。
詳細は [ふしぎなおくりもの](swsh_gift.md) を参照してください。

## ブリリアントダイヤモンド・シャイニングパール

pokeldn はキャラクターとして本体のユニオンルームに参加します。本体では、どのポケモンセンターでも 2F → 左の
係員 → 単純な "yes" (パスワードやグループの選択肢ではない) の順に選び、壁にかからない位置で
ユニオンルームの中で待機します。

```bash
./.venv/bin/python tools/ldn/ldn_scan.py --channels 1,6,11 --dwell 0.8      # see the session
./.venv/bin/python bin/bdsp_connect.py --channels 1,6,11 --count 9 --connect 5 --join 6 \
  --hold 420 --reliable-ack --reliable-sweep 3 --room-walk 15 --room-pattern fixed \
  --room-walk-steps 0 --join-avatar 0 --answer-requests --state 0 --recruiting 0 \
  --answer-talk --can-talk 0 --initiate-talk --initiate-delay 3 \
  --after-approach 0x06:0001000000 --trade-reply --complete-trade \
  --trade-template offer.pb8 --trade-nickname POKELDN --src-var 0x2B7F4C12
```

関連付けに失敗することがあります (`Connect failed with status code 1`)。原因を調べる前に
実行をやり直してください ([セッション](bdsp_session.md))。`--room-pattern fixed` は、本体がキャラクターの状態を尋ねてくるまで参加を送り続けます。
上限は `--room-walk` (上の例では 15) です。`--trade-template` を繰り返し指定すると、セッション内の交換ごとに 1 体のポケモンをキューに入れます。
最後のものは再び提示されます。実行のたびに新しい `--src-var` を使ってください (本体は見た ID を保持するため)。
手で止めた実行のあとは、プレイヤーがいったん部屋を出て入り直します。
`--complete-trade` は本体がセーブデータを書き込めるようにします。指定しないと、交換は最後の確認で止まります。
キャラクターが現れて歩き終えたら、Y →
通信メニュー → 交換するポケモン の順に選びます。

代わりにホストする場合は、先にホストを起動します。そのあとプレイヤーは同じ方法で部屋に入り、交換のエモートを出します (Y →
通信メニュー → 交換するポケモン):

```bash
./.venv/bin/python bin/bdsp_host.py --offer offer.pb8 --complete-trade --capture host.jsonl
```

`--offer` は、セーブデータが PID を持たない正当な PB8 でなければなりません。繰り返し指定すると、交換ごとに 1 体をキューに入れ、
最後のものは再び提示されます。`--password 00000000` は、そのパスワードで入る部屋をホストします。
詳細は [ブリリアントダイヤモンド・シャイニングパール](bdsp.md) を参照してください。

## Legends アルセウス

交換をホストする本体は、参加してくる局と一緒に実行するか、
参加が遅れたときにはホストの役割をその局に渡します ([Legends アルセウス](pla.md))。
`bin/pla_host.py` がホストし、本体はリンクコードで参加します。`bin/pla_join.py` は本体の検索に参加し、
席で交換のたびに繰り返しの `--offer` を 1 つずつ使い、ホストの役割を渡されたら引き継ぎます。

```bash
./.venv/bin/python bin/pla_host.py --code 00000000 --channel 6 --seconds 1800 \
  --session-update --sustain --clock --data-exchange --data-exchange-name POKELDN \
  --data-exchange-id 11223344 --game-channel --trade-box --trade-box-record offer.pa8 \
  --offer-out received.pa8
./.venv/bin/python bin/pla_host.py --ip-host --our-ip 172.16.86.128 --code 00000000 ...   # emulated console, no radio
```

本体: Jubilife Village の Simona → 交換 → 近くの人 → 同じ 8 桁のコード、
ポケモンを提示して確定します。ホストは提示の間にレコードファイルを読み直し、本体が交換したレコードを `--offer-out` に書き出します (後続の交換では `-2` 以降)。
`--trade-box-collect DIR` は、本体が表示または提示したレコードを、カーソルも含めてすべて保存します。
`--trade-box-record` を繰り返し指定すると、セッション内の交換ごとに 1 件のレコードをキューに入れ、最後のものは再び提示されます。
`pokeldn.pla.pokemon` は 376 バイトのゼロ埋めから
レコードを読み、書き、`build` します。`pokeldn.pla.stats` は能力値とサイズを計算します。詳細は [Legends アルセウス](pla.md) を参照してください。

## スカーレット・バイオレット

オフラインの Link Trade 検索は、検索とホストを交互に行うため、pokeldn はホスト (`bin/sv_host.py`) にも
参加 (`bin/sv_join.py`) にもなれます。

```bash
./.venv/bin/python bin/sv_host.py --seconds 240 --player-name POKELDN \
  --rtt-probe --net-property --clock --net-stations 4 --scarlet-response \
  --record-delay 0.17 --announce --announce-delay 5.25 \
  --send-at 6.00:0x7c:1:b90101b902b90280800001 \
  --trade-offer offer.hex --offer-after-open 8
```

本体: X → Poké Portal → Link Trade → offline、コードなし → 検索 の順に選びます。`--trade-offer` を繰り返し指定すると、
同じ席で交換ごとに 1 件のレコードを提示します。ワイヤレベルの要件
(識別メッセージの順序、確認応答 `lowest_pending`) は [スカーレット・バイオレット](sv.md) にあります。

どちらも、ローカルのテラレイドを実行できます。`--raid-seed` は、本体が参加するレイドをホストします (Poké Portal → テラレイドバトル、
Link Code 4970)。シードと 4 つのコンテキストフラグでボスを決め、`--raid-reward
ITEM:QUANTITY` は報酬を置き換え、`--raid-pokemon` は pokeldn のプレイヤーが連れて行くポケモンです。
`--raid-event FOLDER` は、代わりに Poké Portal News の配信からボスと報酬を取り出します。
これは Project Pokemon の [EventsGallery](https://github.com/projectpokemon/EventsGallery) のイベントで、アプリが
要求に応じてダウンロードと更新を行います。`--raid-players N` は最大 3 台の本体が 1 つのレイドに参加できるようにし、
N 台がロビーに入り、それぞれが準備完了を押したとき、またはロビーのタイマーが切れたときに開始します。
`--raid-partner TRAINER` は、本体が空けた枠ごとに 1 回まで (本体 1 台なら 2 回まで) 指定でき、その枠を埋める
NPC のパートナーを選びます。指定しない場合、本体はホストのストーリーの進行度から
これを決めます。
`bin/sv_join.py --raid-pokemon FILE` は、本体がホストするレイドに参加します。どちらの場合も、pokeldn のプレイヤーは
バトル開始とともに退出し、そのポケモンは本体のポケモンと並んで戦います。アプリの
テラレイドの 2 つのツールがフラグを扱います。詳細は [テラレイド](sv_raid.md) を参照してください。

## Legends Z-A

Link Trade の検索はホストと検索を交互に行うため、pokeldn は参加もホストもできます。本体: Link Trade →
local communication → コード 00000000 で検索 の順に選びます。参加側は席を得るまで再検索を続けます。

```bash
# host: start it first, then search on the console
POKELDN_RADIO=esp32:auto ./.venv/bin/python -u bin/za_host.py --keys prod.keys --trade-offer offer.bin --capture zh.jsonl
# join
./.venv/bin/python bin/za_join.py --channels 1,6,11 --dwell 0.35 --seconds 900 \
  --hold 450 --quiet-seat 25 --connect-timeout 6 --game --trade-offer offer.bin --offer-delay 4
```

交換ボックスでポケモンを選び、相手側の表示が出たら確定します。どちらの役割も、同じセッションで別の
提示に応答します。`--trade-offer` を繰り返し指定すると、交換ごとに 1 件のレコードをキューに入れます。終わったら
B で抜けます。本体が退出するとホストは閉じます。
`--offer-out FILE` は、本体が提示したものを保存します。LAN 上のエミュレートした本体には、
`za_host.py --ip-host --our-ip IP --comm-id ffffffffffffffff` と
`za_join.py --ip-join --host-ip IP --our-ip IP --comm-id ffffffffffffffff` を使います。提示ファイルは 354 バイトです
(9 バイトのヘッダ、344 バイトのレコード、末尾 1 バイト)。
Z-A のレイアウトはスカーレットと同じなので、`pokeldn.gen9.build` がレコードを組み立てます。詳細は [Legends Z-A](za.md) を参照してください。

## 診断

- `tools/ldn/ldn_scan.py` は検出可能な LDN ネットワークを一覧表示します。
  `esp32_sniff.py` は、2 枚目のボードを 1 つの MAC・1 つのチャンネルの電波スニッファにします。
- `POKELDN_ESP32_TRACE=FILE` はボードのシリアル通信とカウンタを記録します。
  どのエントリポイントでも `--capture FILE` は、プロトコルのトレースを JSONL で書き出します。
- [ホストの実装](frlg_host.md): コンポーネントの境界、プロトコルの流れ、タイミング、終了処理。

