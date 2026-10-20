---
title: プラグイン
---
# プラグイン

プラグインは、pokeldn を変更せずにツール、ゲーム、ラジオを追加できます。独自の LDN ネットワークに参加したり、ホストしたりすることもできます。`plugin.toml` を含む Python ファイルのフォルダーで、1つの `.pokeplugin` ファイルにまとめて配布され、プレイヤーはアプリのプラグインページから追加します。
プロジェクトはサードパーティのプラグインを配布、審査、サポートしません。

関連コード: `pokeldn/plugins.py`（ローダーと API）、`pokeldn/app/plugin_store.py`（パッケージ）、`pokeldn/ldn/radios.py`（ラジオの検索）、`gui/views/plugins.py`（ページ）。

出発点にできる例が2つあります。`examples/plugins/hello` は最小のプラグインで、2つのファイルに1つのツールがあります。`examples/plugins/test_kit` は API のすべての部分を使っています。

## プラグインのインストール

1. サイドメニューの一番下にある**プラグイン**を開きます。
2. **Browse** で、フォローしているソースに掲載されたプラグインの **Install** を押します。または `.pokeplugin` ファイルを **Installed** タブにドロップするか、**Add a plugin** を押して選びます。
3. プラグインの内容とリスクを読み、確認のチェックを入れてから **Install** を押します。

プラグインのツールはすぐにゲームページに表示され、プラグインのマークと「Plugin: ...」の表示が付きます。各プラグインのカードには、オン・オフを切り替えるスイッチと、メニューがあります。メニューからツールを開いたり、新しいファイルから更新したり、フォルダーを表示したり、ページを開いたり、削除したりできます。インストールと更新のたびに、リスクの画面が再び表示されます。プラグインのフォルダーに手動でコピーしたプラグインには「Added by hand」と表示され、初めてオンにするときに同じ同意が求められます。

起動中にアプリを閉じたり、固まらせたりするプラグインは、次回の起動時にオフになり、アプリがその名前を表示します。プラグインを削除するときは、そのプラグインが保存したデータも削除するかどうかを尋ねられます。

プラグインは pokeldn の中で Python として動き、ファイル、鍵、ネットワーク、ボードへのアクセス権は pokeldn 本体と同じです。パッケージ内のものは、インストールされてオンにされるまで実行されません。

### ソース

ソースは、誰かが公開しているプラグインの一覧（JSON）です。プラグインページの **Sources** タブでソースをフォローします。**Browse** には、フォローしているソースが指定するすべてのプラグインが並び、**Install**、**Update to N**、**Installed** のいずれかと削除ボタンが付きます。アプリは最初から `Decryptu/pokeldn-decrypt-plugins` をフォローしており、このソースは削除できません。ほかのソースを削除しても、そのソースからインストールしたプラグインは残ります。

| Sources に入力した内容 | 読み込む一覧 |
|---|---|
| `OWNER/REPO` または `https://github.com/OWNER/REPO` | `https://raw.githubusercontent.com/OWNER/REPO/HEAD/pokeldn-source.json` |
| `.json` で終わる HTTPS アドレス | そのファイル |
| それ以外の HTTPS アドレス | そのフォルダー内の `pokeldn-source.json` |

```json
{
  "schema": 1,
  "name": "My plugins",
  "homepage": "https://github.com/me/my-plugins",
  "plugins": [
    {"id": "my-plugin", "name": "My plugin", "version": "1.0.0", "api": "1.6",
     "description": "...", "author": "me",
     "download": "https://github.com/me/my-plugins/releases/download/my-plugin-1.0.0/my-plugin-1.0.0.pokeplugin",
     "sha256": "64 hex digits", "size": 46302,
     "icon": "https://.../icon.png", "homepage": "https://..."}
  ]
}
```

`id`、`version`、HTTPS の `download`、パッケージの `sha256` は必須です。どれかが欠けている項目や、id が重複している項目は除外されます。`schema` が別の値の一覧は受け付けません。ダウンロードした SHA-256 が一覧の値と一致しない場合は、読み込む前に拒否されます。パッケージの id が項目の id と一致しない場合は、リスクの画面より前に拒否されます。一覧があっても作者が確かだとは言えません。リスクの画面と署名の確認は、ファイルからインストールする場合と同じように行われます。

アプリは起動時（設定の「Updates」がオンのとき）と、ページの更新ボタンを押したときに、フォローしている各ソースを読み込みます。直近の一覧は `plugins/.source-cache/` に保存され、ソースが応答しないときはそれを表示します。フォローしているソースの一覧は `plugins/.sources.json` にあります。インストール済みのプラグインは取得元のソースを記録し、そのソースから更新を受け取ります。ファイルからインストールしたプラグインは、その一覧に載っている最初のソースから更新を受け取り、どのソースにも載っていなければ、自身の `updates` のアドレスから受け取ります。

`scripts/plugin_source.py` は、アプリと同じ方法で各パッケージを読み込み、パッケージから一覧を書き出します。

```bash
python scripts/plugin_source.py my-plugin-1.0.0.pokeplugin -o pokeldn-source.json --name "My plugins" \
    --download "https://github.com/me/my-plugins/releases/download/{id}-{version}/{file}"
```

すでにより高いバージョンで掲載されている項目は残されます。

### ガードレール

明らかな害を防ぐために、2つのチェックがあります。どちらもサンドボックスではありません。これらを回避するように書かれたプラグインは回避できてしまいます。

インストール画面は、パッケージの Python ファイルを読み、その動作を一覧にします。対象は、ネットワークの利用、ほかのプログラムの起動、ファイルの削除、ネイティブコードの呼び出し（`ctypes`、`cffi`）、実行時に作られるコードの実行（`exec`、`eval`、`compile`、`marshal`）、`prod.keys` の指定、読み取れないコンパイル済みファイルの同梱です。一覧が空の場合は「見つからなかった」という意味であり、プラグインが安全だという意味ではありません。

プラグインのコードが呼び出しスタック上にある間は、アプリ内でも、どの実行でも、監査フック（`pokeldn/plugin_guard.py`）が `PermissionError` で拒否します。

| 拒否 | 許可 |
|---|---|
| プラグインのフォルダー外にあるファイルの削除、空にする、移動、上書き | 同じ操作でも、プラグイン自身のフォルダー、データのフォルダー、ライブラリ、セッションのフォルダー、ログのフォルダー、一時フォルダー内での操作 |
| `rm`、`del`、`rd`、`Remove-Item`、`shred`、`dd`、`format`、`mkfs`、`diskutil`、`diskpart`、`sudo`、`su` など同類のプログラムを、直接またはシェルコマンドの中で起動すること。`-delete` または `--no-preserve-root` を付けたコマンド | ほかのプログラムの起動 |
| | どこにでも新しいファイルを作ること |

呼び出しスタック上にプラグインがないコードは、pokeldn 自身のものを含めて拒否されません。ただし、プラグインから呼び出された pokeldn のコードは拒否の対象になります。拒否された操作が必要なプラグインは、その操作をプレイヤー自身に行うよう求めます。

## プラグインの作り方

### フォルダー構成

    my-plugin/
        plugin.toml        what the plugin is
        plugin.py          register(api): what it adds
        hello.py           a tool's script
        icon.png           optional, shown on the Plugins page and the Games page
        mylib_common.py    optional, any module the scripts share

### plugin.toml

```toml
id = "my-plugin"            # required: lowercase letters, digits, dots, dashes, underscores
name = "My plugin"          # shown in the app
version = "1.0.0"           # compared on update: 1.2.0 updates 1.0.0
author = "Your name"
description = "One or two sentences shown on the install screen and the plugin's card."
homepage = "https://example.org/my-plugin"   # optional, opened from the plugin's menu
updates = "https://github.com/you/my-plugin" # optional, where newer versions are published (Updates)
api = "1.1"                 # required: the plugin API it needs (see Versions); 1 means 1.0
icon = "icon.png"           # optional; a square PNG, 96 x 96 looks sharp
library = "My files"        # optional (1.7): a folder the player sees, see A library
library_sort = "name"       # optional (1.8): its order until the player picks one, see A library
module = "plugin"           # optional; the module holding register()
exclude = ["tests"]         # optional; folders and files pack_plugin.py leaves out of the package
```

### plugin.py

`register(api)` は、プラグインがオンになったときに、アプリ内と、アプリが開始するすべての実行で呼ばれます。

```python
from pokeldn.plugins import Field, Game, Tool


def register(api):
    api.add_tool("frlg", Tool(
        "my-hello",                                   # a key no other tool uses
        "Hello",                                      # the name on the Games page
        "hello.py",                                   # the script, relative to the plugin folder
        "Says hello.",                                # the line under the tool's name
        ("Nothing to do on the console.", "Press Start."),   # the console steps the app lists
        fields=(Field("--name", "Your name", default="Red"),),
        fixed=("--trainer", "{ot}"),                  # always passed; {ot} is the trainer name in Settings
    ))
```

### ツールのスクリプト

アプリはスクリプトをインポートして詳細タブ用のオプションを読み取り、その後セッションのフォルダーで独立した子プロセスとして実行します。スクリプトは `build_parser()` を定義し、作業は `if __name__ == "__main__"` の中で行います。トップレベルの処理はインポート時に実行されるため、ゲームページが壊れます。

```python
import argparse


def build_parser():
    parser = argparse.ArgumentParser()
    parser.add_argument("--name")
    parser.add_argument("--trainer")
    return parser


if __name__ == "__main__":
    args = build_parser().parse_args()
    print(f"Hello {args.name}, from {args.trainer}'s pokeldn.")
```

| スクリプト | アプリ |
|---|---|
| 1行を出力する | セッションの出力に表示する |
| `[done] trade N complete` を出力する（`pokeldn.ldn.show_done()`） | キューの交換を1件完了にする |
| 終了コード 0 で終わる | セッションを完了として終える |
| 別の終了コードで終わるか、例外を発生させる | セッションを失敗として表示し、トレースバックを出力に表示する |
| `KeyboardInterrupt` を受け取る | プレイヤーが停止を押したことになる。15秒以内に正常に終わらなければ、プロセスは終了される |
| ポップアップを開く（`pokeldn.popup`、1.2） | アプリが表示しているページの上に表示し、プレイヤーの回答をスクリプトに返す |
| 複数の選択肢から1つを選ばせる（`pokeldn.popup.choose`、1.7） | 選択肢ごとにボタンを表示し、押されたボタンの番号を返す |
| 進捗を報告する（`pokeldn.progress.show(done, total, title, unit)`、1.7） | 行の代わりに、セッションパネルに速度と残り時間付きのバーを描画する（[進捗](gui.md#progress)） |
| ステータスカードを表示する、またはライブラリのファイルに印を付ける（`pokeldn.panel`、1.8） | ツールのページに描画し、実行後も残す（[ステータスカード](#status-cards-18)） |

ポップアップはプレイヤーを待ちません。`acknowledge` と `confirm` はすぐにそのIDを返し、回答は後からコールバックとして届きます。コールバックはいつでも、別のスレッドで呼ばれる可能性があります。スクリプトは自身の状態を保持し、回答が届いたときに反応します。回答が来るまで止まることはありません。次のように書きます。

```python
import queue
from pokeldn import popup

answers = queue.Queue()
popup.set_dispatcher(answers.put)       # callbacks wait for the loop below instead of running on their own thread
asked = popup.confirm("Trade the shiny?", "Your partner offers a shiny Gholdengo.", "Trade", "Skip",
                      on_agree=lambda: print("trading"), on_disagree=lambda: print("skipping"),
                      tone="warning")
while not finished():                   # the script's own work goes on meanwhile
    try:
        answers.get(timeout=0.5)()
    except queue.Empty:
        pass
popup.cancel(asked)                     # no use for the answer any more: the pop-up closes, no callback fires
```

共有モジュールには、ほかのパッケージが使わない名前を付けてください（`utils` ではなく `mylib_common`）。プラグインのフォルダーは Python の検索パスに追加されるためです。

### プラグインがインポートできるもの

リリース版のアプリは独自の Python 3.13 を含み、`pip` は含みません。プラグインは次をインポートできます。

| 対象 | 提供元 |
|---|---|
| 標準ライブラリ | 同梱。ただし `multiprocessing`、`tkinter`、`curses`、`venv`、`ensurepip`、`profile`、`cProfile`、`pstats`、`trace`、`compileall`、`modulefinder`、`pyclbr`、`tabnanny` と、ほかのプラットフォーム用のモジュールは除く（`scripts/pack_app.py`、`PLUGIN_STDLIB`） |
| `trio`、`pycryptodome`（`Crypto`）、`zstandard`、`pyserial`（`serial`）、`websockets`、`certifi`、`bleak`、`unicorn`、`flet` | pokeldn 自身の依存関係。`requirements.txt` に記載のバージョン |
| `ldn` と `pokeldn` | pokeldn 自身。安定しているのは下の「API」の表だけ |
| 純粋な Python のモジュールなら何でも | プラグインのフォルダーに同梱 |

コンパイル済みのコードを含むパッケージ（`numpy`、`Pillow`）は、パッケージがインストールされているソースからの実行でのみ動きます。
公開する前に、リリース版のアプリでプラグインをテストしてください。プラグインテストキットの自己テストも、そこで同じように動きます。

### フィールド

`Field` は、ツールの基本タブにある1つの入力欄です。アプリは、その値をフラグの後ろに渡します。

| `kind` | 入力欄 | 渡し方 |
|---|---|---|
| `text`（既定） | テキストボックス | `--flag VALUE` |
| `number` | 数値ボックス | `--flag VALUE` |
| `choice` | `choices=((value, label), ...)` のドロップダウン | `--flag VALUE` |
| `switch` | オン・オフのスイッチ | オンのとき `--flag` |
| `file` | ファイル選択。`exts=("bin",)` | `--flag PATH` |

### プラグインが保持するもの

更新するとプラグインのフォルダーは置き換えられます。設定、キャッシュ、そのほか長く残すべきものは、`register` から `api.data`、またはツールのスクリプトから `pokeldn.plugins.data_folder("my-plugin")` に保存してください。データフォルダーの `plugin-data/<id>` に保存され、更新後も残ります。削除時に消えるのは、プレイヤーが求めた場合だけです。

### ライブラリ（1.7）

プレイヤーが管理するファイル（バックアップ、ダウンロードなど）を保存するプラグインは、`plugin.toml` でライブラリ名を指定します。`library = "My files"` と書くと、`Documents/pokeldn/My files` になります（`POKELDN_DATA` 配下のデータフォルダーにある `My files`）。`register` の `api.library`、またはツールのスクリプトの `pokeldn.plugins.library_folder("my-plugin")` で取得でき、初回使用時に作成されます。

| ライブラリ | |
|---|---|
| ガードレール | データフォルダーと同じく、書き込み、置き換え、名前の変更、削除を許可する |
| インストール画面 | ライブラリ名を表示する |
| プラグインの削除 | ライブラリとその中のすべてのファイルは残す |
| ゲームページ | プラグインのツールの下にファイルを一覧表示する。プレイヤーはファイルを追加（ボタンまたはドロップ）、名前の変更、削除ができる。`.` で始まり `.part` で終わる名前は除外される。そのような名前でファイルを書き、完成したものに置き換えてください |
| 並び順 | 名前、日付、サイズのいずれかで昇順・降順を選べる。プレイヤーが選んだ設定はプラグインごとに保持される。それまでは `plugin.toml` の `library_sort`（1.8: `name`、`name-desc`、`date`、`date-desc`、`size`、`size-desc`）に従い、指定がなければ `date-desc`（新しい順）になる。名前は自然順で並ぶ。数字の連続は数値として比べ、大文字と小文字は区別しない。そのため `v65536` は `v131072` より前に、`20.5.0` は `23.0.1` より前に並ぶ |
| ファイルのメモ | `pokeldn.panel.annotate`（1.8、[ステータスカード](#status-cards-18)） |
| 名前 | 英字、数字、空白、ドット、ハイフン、アンダースコアを使って40文字まで。pokeldn 自身のフォルダー（`Bank`、`Saves`、`Received` など）、Windows のデバイス名、読み込み済みの別のプラグインのライブラリ名は拒否される |

`help=` は入力欄の下に1行を表示し、`default=` は既定値を入れ、`hidden=True` にすると詳細タブに移します。組み込みツールが使うフィールドと、`Field` のその他の仕様は、`pokeldn/app/catalog.py` にあります。API 1.5 で `Field.identity` が追加されました。これは、ラベルが翻訳される位置指定フィールドのための、安定した設定キーです。ツールの `fixed` 引数には、`{ot}`、`{tid}`、`{sid}`、`{language}`、`{gender}`（設定のトレーナー情報）、`{received}`（Received フォルダー）、`{stamp}`（実行の開始時刻）を指定できます。1.7 からは `{keys}`（設定で指定された `prod.keys` ファイル）も使えます。パーサーが `--keys` を受け取るスクリプトは、トークンがなくてもそのファイルのパスを受け取ります。

### ステータスカード（1.8）

実行すると、ツールのページの上部、入力欄の上にカードが残り、ライブラリのファイルにメモが付きます。`pokeldn.panel` は `[panel]` の行として出力するため、ソースからでもリリース版のアプリでも同じように動きます。アプリは実行の終了後、カードをデータフォルダーの `plugin-panels/<id>.json` に保存し、プラグインと一緒に削除します。アプリの外では、行が出力されるだけです。

```python
from pokeldn import panel, plugins

icon = plugins.data_folder("my-plugin") / "icons" / "0100abcd.png"
panel.show("console", "Pokémon: Let's Go, Pikachu!", [
    "Group\tcapito27's Switch · 1 of 8 consoles",
    "System version\t23.0.1",
], image=icon, chips=[("Version", "1.0.2")], tone="ok")
panel.annotate("Game 1.0.2.nsp", state="ok", text="Every signature checks", image=icon)
```

| 呼び出し | アプリに表示されるもの |
|---|---|
| `panel.show(id, title, lines=(), *, image=None, chips=(), tone="info")` | カード `id`。同じ `id` で最後に表示したカードを置き換える。1行は平文、または2列にする `label<TAB>value`。チップは `(label, value)` の組。`tone` は `info`、`ok`、`warning`、`error` のいずれかで、タイトルの横にアイコンを表示する |
| `panel.clear(id)` | カードを削除する |
| `panel.annotate(name, *, state="ok", text="", image=None)` | ライブラリのファイル `name` に印を付ける。`state` は `ok`、`bad`、`checking` のいずれか。その下に1行のテキストを表示し、ファイルアイコンの代わりに画像を表示する |
| `panel.unannotate(name)` | メモを削除する |

| 上限 | |
|---|---|
| カード id | 小文字の英字、数字、`-`、`_` を使って1～32文字 |
| プラグインごとのカード数 | 8枚。それを超える新しい id は破棄される |
| カード | タイトルは80文字まで。行は12行まで（1行200文字まで）。チップは6個まで（片側40文字まで） |
| 画像 | プラグインのデータフォルダーまたはライブラリ内にある、2MB 以下の PNG または JPEG。それ以外は描画されない |
| メモ | ファイル名に紐づく。ファイルの名前を変えたり削除したりすると、メモも消える |

上限を超える呼び出しは、スクリプトで `ValueError` を発生させます。アプリが読み取れない `[panel]` の行は、セッションの出力に表示されます。

### 独自のゲーム

```python
api.add_game(Game("my-game", "My game", "Mine", "", (tool_one, tool_two)), icon="icon.png")
```

`Game(key, name, short, doc, tools)` です。ゲームの項目には、組み込みゲームの下にツールが順番に並びます。`api.add_tool("my-game", tool)` で追加できます。

### ラジオ

ラジオは、ESP32 ボードの代わりに無線メッセージを中継します。`api.add_radio(scheme, install, label)` を呼ぶと、プラグインページの **Radio** の下に表示されます。そのラジオで開始された実行は、最初の無線呼び出しの前に `install(argument, log)` を呼びます。

```python
import contextlib
from ldn import wlan


def install(argument, log):
    @contextlib.asynccontextmanager
    async def factory():
        yield MyFactory()           # the interface of ldn.wlan.Factory
    wlan.set_factory(factory)


def register(api):
    api.add_radio("myradio", install, "My radio")
```

`pokeldn/ldn/esp32_wlan.py` は、ボードのファクトリーであり、カーネルインターフェースを持たないラジオを作るときの手本です。アプリは、引数を空にした `POKELDN_RADIO=myradio:` としてプラグインのラジオを実行します。コマンドラインでは、コロンの後ろに続く文字列が引数になります。

### LDN ネットワーク（1.7）

`pokeldn.ldn.network` は、プラグインのために LDN ネットワークに参加またはホストし、ラジオの IPv4 スタックを渡します。カーネルインターフェースを持たないラジオが必要で、ESP32 ボードまたは ldnrs が該当します。

```python
from pokeldn.ldn import network

with network.join(COMM_ID, keys=args.keys, password=PASSPHRASE, name="My game") as net:
    net.add_protocol(6, lambda src, dst, segment: segments.put((src, segment)))   # TCP, say
    while not net.send_ip(6, net.host_ip, first_segment):    # False until ARP finds the host
        time.sleep(0.1)
```

| 名前 | 内容 |
|---|---|
| `join(comm_id, *, keys, password, nickname, app_version, name, log, timeout)` | `comm_id` を広告しているネットワークだけを探して参加する。`name` は、プレイヤーのメッセージでの呼び名です。応答がない場合は `RuntimeError` を発生させる |
| `host(comm_id, *, keys, password, nickname, application_data, scene_id, app_version, max_participants, channel, protocol, log, timeout)` | ゲーム機が参加できるネットワークを開く |
| `Network.ip`、`.mac`、`.host_ip`、`.broadcast`、`.ssid`、`.comm_id`、`.application_data` | アドレスと広告の情報。参加中はホストのもの、ホスト中は自身のもの |
| `Network.participants` | ホストしているネットワークに参加している端末。それぞれ `.index`、`.ip`、`.mac`、`.name` を持つ |
| `Network.add_protocol(number, handler)`、`.remove_protocol(number)` | `handler(src_ip, dst_ip, payload)` は、自分宛てに届いた番号 `number` の再構成済み IPv4 パケットをすべて、ラジオのループ上で受け取る。キューに入れてすぐ戻ること。UDP（17）はスタック側が保持するため、ここでは扱えない。handler で発生した例外はログに記録され、そのパケットは破棄される |
| `Network.send_ip(protocol, dst_ip, payload)` | IPv4 パケットを1つ送る。1500バイトで分割される。`dst_ip` のアドレスが不明な間は、ARP 要求を送ったうえで `False` を返す |
| `Network.udp_socket(port)` | ネットワーク上の UDP ソケット。`sendto`、`recvfrom`、`setblocking`、`settimeout`、`fileno`、`close` が使える |
| `Network.set_application_data(data)` | ホストしているネットワークの新しい広告情報 |
| `Network.error`、`.close()` | ネットワークが終了した理由（稼働中は空）。ネットワークを退出する。`Network` はコンテキストマネージャーです |
| `pokeldn.ldn.radio_lost()` | ラジオがプロセスから離れた理由。空の場合もある。その後、実行はプレイヤーの停止と同じ `KeyboardInterrupt` で終わる。この関数で両者を区別できる |

ボード経由の転送は約 0.5 MB/s で動きます。`pokeldn.progress.show` で進捗を報告してください。

### バトルサーバー

バトルサーバーは、ホスト用ランチャーのためにレイドバトルを動かします。これにより、ゲーム機とプログラムのポケモンが1つのバトルで戦います（[バトルの提供](sv_raid.md#serving-the-battle)）。対応しているのはスカーレット・バイオレット（`"sv"`）だけです。`api.add_battle_server(game, name, factory, label)` で登録します。`bin/sv_host.py --raid-serve NAME` の実行（または、読み込まれた最初のものを使う `--raid-serve` 単独の実行）は、ゲーム機がバトルのブートストラップを送ってきたら、ネットワークのループとは別の場所で `factory(bootstrap, partners)` を呼びます。`bootstrap` はバトルのブートストラップのバイト列、`partners` は2人の NPC パートナーの 0x158 バイトのパーティ情報です。返されるオブジェクトは次のとおりです。

| メンバー | 内容 |
|---|---|
| `start(now)` | `now` 秒時点でのサーバーの最初のフレーム。バトルを動かしたフレーム数を返す |
| `run(until)` | 毎秒30フレームで `until` 秒まで進める（最低1フレーム）。バトルを動かしたフレーム数を返す |
| `t` | サーバーの時計（秒） |
| `receive(slot, message)` | バトルのスロット `slot` から届いた端末のメッセージ。32バイトのヘッダーの後に本文が続く |
| `sent(slot)` | 前回の呼び出し以降に `slot` 宛てに溜まったメッセージ。それぞれ32バイトのヘッダーの後に本文が続く |

スロット 0 はプログラムのポケモン、1 はボス、2 はゲーム機、3 と 4 はパートナーです。どのメンバーからでも例外が出ると、サーバーの提供は止まります。その場合、ゲーム機はそのまま自分のバトルで戦います。

```python
def register(api):
    api.add_battle_server("sv", "mine", MyServer, "My raid battle")
```

### パッケージ化とテスト

```bash
python scripts/pack_plugin.py my-plugin --key ~/my-plugin-signing.pem
```

上のコマンドは、署名付きの `my-plugin-1.0.0.pokeplugin` を書き出し、インストール前にアプリと同じ方法で検査します。隠しファイルと、`plugin.toml` の `exclude = ["tests", "tools"]` が指定するフォルダーは、パッケージに含まれません。`plugin.toml` が最上位、または最上位のフォルダー1つの中にあれば、どの zip でも使えます。そのフォルダーの外に出るパスとシンボリックリンクは含めないでください。

開発中はパッケージを作らずに済ませられます。フォルダーを `POKELDN_PLUGINS` に指定すると、プラグインページの設定に関係なく、そのプロセスと、そこから開始された実行でプラグインが有効になります。

```bash
POKELDN_PLUGINS=~/my-plugin python -m pokeldn --plugins
POKELDN_PLUGINS=~/my-plugin python -m pokeldn my-hello -- --name Blue
POKELDN_PLUGINS=~/my-plugin python gui/main.py
```

`--plugins` は見つかったすべてのプラグインと、読み込みに失敗した理由を一覧表示します。`register` で例外を発生させるプラグイン、別の API バージョンを対象とするプラグイン、既に使われているキーを登録しようとするプラグインは、何も追加しません。そのカードにはエラーが表示され、ほかのプラグインは引き続き読み込まれます。また、プラグインは `pokeldn.plugins` グループのエントリポイントで `register` を含むモジュールを指定した Python パッケージとして配布することもできます。

### 署名

`--key FILE` は、作者の Ed25519 鍵でパッケージに署名します。鍵は初回に `FILE` へ作成され、鍵のフィンガープリント（`ABCD-EFGH-IJKL-MNOP`）を表示します。パッケージには `plugin.sig` が含まれます。これは公開鍵と、プラグインの id、バージョン、そのほかすべてのファイルの SHA-256 に対する署名です。

| アプリが確認する内容 | 表示 |
|---|---|
| 署名がファイルと一致しない | ファイルを拒否する。作者の署名後に変更されたため |
| 署名済みで、これまで見たことのないプラグイン | フィンガープリントを表示し、そのプラグイン id の鍵を記憶する |
| 記憶している鍵で署名された更新 | 「Signed by the same author」 |
| 別の鍵で署名された更新、または以前の版が署名されていたのに署名のない更新 | 赤い警告を表示し、インストールを続けるには2回目の同意が必要 |
| 署名されたことがなく、署名のないプラグイン | 「Not signed」 |

鍵ファイルは非公開にし、バックアップを取っておいてください。鍵を失うと、以降のすべての版が別の作者のものに見えてしまいます。また、鍵を持つ人は誰でも作者として署名できます。フィンガープリントはプラグインのページで公開し、プレイヤーがインストール画面に表示されるものと比べられるようにしてください。最初のインストールでは作者が誰かは証明できません。署名によって証明できるのは、以降の版が同じ作者から来たことだけです。

### 更新

`plugin.toml` の `updates` は、新しい版の公開先を指定します。

| `updates` | アプリが読み取るもの |
|---|---|
| `https://github.com/OWNER/REPO` | 最新のリリース。そのタグ（`v1.3.0` または `1.3.0`）と、最初の `.pokeplugin` アセット |
| それ以外の HTTPS アドレス | JSON ファイル `{"version": "1.3.0", "url": "https://.../my-plugin-1.3.0.pokeplugin", "page": "..."}` |

設定の「Updates」がオンの場合、アプリは起動時と、プラグインページの更新ボタンを押したときに、インストール済みの各プラグインのアドレスに問い合わせます。新しい版があると、プラグインメニューの項目が緑色になり、そのプラグインのカードに **Update to 1.3.0** が表示されます。これを押すとファイルをダウンロードし、署名の確認付きのインストール画面を開きます。プレイヤーが同意しなければ何もインストールされません。HTTP は、リリースをローカルでテストするため `127.0.0.1` と `localhost` の場合のみ受け付けます。

## 言語パック（1.5）

言語パックは、データだけを含むフォルダー形式のプラグインです。`name` は設定の App language に表示される名前です。`locale` は言語タグです。保存された選択は、`id` によってプラグインを特定します。

```toml
id = "community-zh-hans"
name = "简体中文"
version = "1.0.0"
api = "1.5"
locale = "zh-Hans"
strings = "strings.toml"
```

`strings.toml` は `plugin.toml` と同じ場所に置きます。TOML がフラットなテーブルとして読めるように、ドットを含むキーはそれぞれ引用符で囲んでください。

```toml
"bank.bank_676c47" = "宝可梦银行"
```

このファイルには文字列の値だけを書きます。パックには Python モジュールも `register` 関数も必要ありません。インストール、署名、作者の鍵、削除、`updates` は、ほかのプラグインと同じパッケージの経路を使います。不正な TOML、文字列以外の値、不正な書式の指定は、インストール前に拒否されます。インストール済みのパックが壊れている場合は、読み込みエラーを表示し、英語に戻ります。

パックはプラグインページでインストールし、設定の App language でパックを選んでから、pokeldn を再起動してください。パックを選ぶと、そのプラグインが有効になります。英語はいつでも使えます。パックがない、無効になっている、または削除された場合は、英語が使われます。トレーナーの言語設定は、アプリが生成するポケモンの言語を引き続き決めます。

すべての参照には、インラインの英語の既定値があります。キーがない場合は既定値を使い、使われないキーは効果がありません。`{value0}` のような名前付きプレースホルダーは、翻訳の中で位置を移してもかまいません。名前、変換、書式指定は変えないでください。`{{` と `}}` は、そのまま波かっことして出力されます。プレースホルダーを書式化できない翻訳は、その参照に限って英語の既定値を使います。

```bash
python scripts/extract_strings.py
python scripts/extract_strings.py --check
python scripts/pack_plugin.py examples/plugins/pseudo -o pseudo.pokeplugin
```

抽出ツールは `pokeldn/app/data/strings.en.toml` をキー順に並べて書き出します。テストは、この内容がソースと一致するかを確認し、英語の既定値が2つあるキーを拒否します。文言を変えても、キーは変わりません。テンプレートは別のプラグインリポジトリにコピーし、値を翻訳してください。省略したキーは保守が不要です。`examples/plugins/pseudo` の部分的な擬似ロケールは、選んだ文字列を角かっこで囲み、ほかの文字列はフォールバックの確認のために英語のまま残します。サンプルはデスクトップ向けパッケージから除外されます。

### 追加のリソース（1.9）

API 1.9 のパックは、`plugin.toml` と同じ場所にある次のオプションのファイルとフォルダーを指定できます。1.5 のパックも引き続き有効です。インストール、署名、更新は、`strings.toml` と同様にこれらも対象にします。いずれもプラグインのフォルダーの外のパスを指定することはできません。

```toml
api = "1.9"
messages = "messages.toml"
templates = "templates.toml"
terms = "terms.toml"
help = "help.toml"
documents = "docs"
font_family = "Noto Sans CJK SC"
```

| 項目 | キー | 値 |
|---|---|---|
| `messages` | モジュールの英語テキスト。そのまま使い、波かっこも文字どおりに扱う | その翻訳 |
| `templates` | 名前付きフィールドを含む英語のメッセージ。例: `"Could not open {path}: {reason}"` | 同じフィールドを含む翻訳 |
| `terms` | ゲーム用語（1～200文字、最大10000語）。例: `"Squirtle"` | その翻訳 |
| `help` | `"TOOL-KEY:--option"`。1つのツールのフラグ | そのフラグの説明（詳細タブに表示される） |
| `documents` | `guide.md` と、`docs/` のページを名前どおりに収めたフォルダー | |
| `font_family` | 同梱またはインストール済みのフォント名（100文字まで） | アプリのテキストフォント。CJK のフォールバックは残る |

`strings.toml` は、引き続き `t(key, English)` を通してアプリ自身の操作部分を扱います。ほかの表は、ゲームモジュール、PKHeX、プラグインが生成する英語テキストを翻訳します。対象は、操作部分に表示されるラベル、概要、エラー、配布とレイドの説明などです。メッセージは、まず `messages` で全体を探し、次に `templates` と照合し、次に文ごとに扱い、最後に単語の境界で `terms` の各項目を置き換えて扱われます。項目がない場合は英語を表示します。

テンプレートの元の文は、2048文字以内に、名前付きフィールドを1～8個含みます。隣り合う2つのフィールドの間には、固定の文字列が必要です。変換、書式指定、属性アクセスは含められません。照合は正規表現を使わない文字列の一致で、元の文が長いものから順に行われ、対象は4096文字以内のメッセージです。取り込まれた値はそのまま表示されます。ただし、フィールド名が `term` で始まる場合はゲーム用語として、`message` で始まる場合はメッセージとして、最大4段階まで翻訳されます。

PKHeX は、パックがその言語を持つ場合（`ja`、`fr`、`it`、`de`、`es`、`ko`、`zh-Hans`、`zh-Hant`。`zh` に `TW`、`HK`、`MO` が付く場合は `zh-Hant` として扱う）、種族、技、道具、特性、性格、ボールの名前をそのロケールで表示し、適法性レポートもそのロケールで出します。Flet 自身のダイアログとファイル選択も、そのロケールに従います。

文書リーダーとその検索は、パックにページがあればそのページを、なければ英語のページを読みます。翻訳ページは、見出しの数が英語のページと同じであれば英語ページの見出しアンカーを引き継ぎます。そうでない場合は、見出しの前に `<a id="NAME"></a>` の行を置くとアンカーを指定できます。front matter の `title` を指定すると、サイドバーのページ名が変わります。ページの階層自体は英語版のものです。

レコード、ファイル、ほかのアプリが読み取るものは英語のままです。ファイル名、Bank とキューの概要、GTS の一覧、設定、セッションログ（その `[app]` 行は翻訳して表示）、ラジオのスクリプトの出力、`[done]` 行が該当します。設定にあるトレーナーの言語は、アプリが生成するポケモンの言語を引き続き決めます。

## プラグインテストキット

`scripts/pack_plugin.py` でパッケージ化した `examples/plugins/test_kit` は、次を追加します。

| 場所 | ツール | 内容 |
|---|---|---|
| Plugin Test Kit | セルフテスト | API のバージョン、自身の読み込み、インポート、すべてのフィールド種別、トレーナートークン、セッションフォルダー、データフォルダー（更新後も残る実行回数カウンター）、ライブラリとそのファイルへのメモ、ラジオを確認する。それぞれ PASS または FAIL を表示し、ページに「Last self-test」カードを残す |
| Plugin Test Kit | 模擬セッション | 交換の手順と、交換ごとに `[done]` 行を出力する。停止を押すと正常に終了する |
| Plugin Test Kit | わざと失敗 | 例外を発生させるか、終了コード 3 で終わる |
| ファイアレッド・リーフグリーン | プラグインからのあいさつ | 組み込みツールの中にある、プラグインのツール |
| ラジオ | テストキットのラジオ | インストールされ、何も中継しない。これで開始したセッションには、その旨が表示される |

## バージョン

プラグイン API はメジャーバージョンとレベルを持ち、`1.1` のように書きます。プラグインの `api` は、必要とする最低のバージョンを指定します。アプリは自身のバージョンをプラグインページと `python -m pokeldn --plugins` で表示します。

| API への変更 | アプリの動作 |
|---|---|
| 追加 | レベルを上げる。より高いレベルを必要とするプラグインは「Update pokeldn」と表示して拒否する |
| 削除または意味の変更 | メジャーバージョンを上げる |
| 古いメジャーバージョンを引き続き提供 | `pokeldn/plugins.py` の `DEPRECATED` に、それを削除するリリースを指定する。プラグインは読み込まれ、そのカードとインストール画面にその旨が表示される |
| 古いメジャーバージョンの提供を終了 | `API_OLDEST` より古いメジャーは通らず、プラグインは「Ask its author for a newer version」と表示して拒否される |

プラグイン自身の `version` は、更新、再インストール、ダウングレードを表示するために、インストール済みの版と比べられるだけです。

| API | 追加内容 |
|---|---|
| 1.0 | `add_tool`、`add_game`、`add_radio`、`folder`、`version` |
| 1.1 | `api.data`、`api.level`、`pokeldn.plugins.data_folder(plugin_id)` |
| 1.2 | `pokeldn.popup` |
| 1.3 | プログラム向けインターフェース（ビルダー、配布ファイル、PKHeX、ボード） |
| 1.4 | Bank の取り込み、記録、移行先、オファー、交換の完了 |
| 1.5 | 言語パックのプラグインと文字列の参照 |
| 1.6 | `add_battle_server` |
| 1.7 | `pokeldn.ldn.network`、`pokeldn.ldn.radio_lost`、`library` と `api.library`、`pokeldn.popup.choose`、`pokeldn.progress`、`{keys}` トークン |
| 1.8 | `pokeldn.panel`、`library_sort` |
| 1.9 | 言語パックのメッセージ、テンプレート、用語、ツールのヘルプ、ドキュメント、フォントの設定 |

## API バージョン 1

| 名前 | 内容 |
|---|---|
| `api.add_tool(game, tool)` | キーが `game` のゲームの下に `Tool` を追加する。`frlg`、`lgpe`、`swsh`、`bdsp`、`pla`、`sv`、`za`、またはプラグインが追加したもの |
| `api.add_game(game, icon="")` | ツールを含むゲームの項目を追加する |
| `api.add_radio(scheme, install, label="")` | ラジオを追加する |
| `api.add_battle_server(game, name, factory, label="")` | レイドバトルのサーバーを追加する（1.6） |
| `api.folder` | プラグインのフォルダー（`Path`）。Python パッケージの場合は None |
| `api.version`、`api.level` | アプリが動作しているメジャーバージョンとレベル（`1` と `8`） |
| `api.data` | 更新後も残るフォルダー。初回使用時に作成される（1.1） |
| `pokeldn.plugins.data_folder(plugin_id)` | ツールのスクリプトから同じフォルダーを取得する（1.1） |
| `api.library`、`pokeldn.plugins.library_folder(plugin_id)` | `plugin.toml` の `library` で指定された、プレイヤーから見えるフォルダー。初回使用時に作成される（1.7、[ライブラリ](#a-library-17)） |
| `Tool`、`Field`、`Game` | アプリのカタログの型。`pokeldn.plugins` からインポートする |
| `pokeldn.ldn.show_done()` | 交換を完了にする行を出力する |
| `pokeldn.popup.acknowledge(title, text="", *, id, label="OK", on_ack, tone)` | ボタンが1つのポップアップ。戻り値はそのID（1.2、[ポップアップ](gui.md#pop-ups)） |
| `pokeldn.popup.confirm(title, text, agree, disagree, *, id, on_agree, on_disagree, on_dismiss, tone)` | 呼び出し元が指定する「同意」「不同意」ボタンを持つポップアップ。Esc または外側のクリックで閉じる。戻り値はそのID（1.2） |
| `pokeldn.popup.choose(title, text, options, *, id, on_choice, on_dismiss, tone)` | 2～8個の選択肢ボタンを持つポップアップ。押された選択肢は `on_choice(index)` で受け取る。Esc または外側のクリックで `on_dismiss` が呼ばれる。戻り値はそのID（1.7） |
| `pokeldn.progress.show(done, total, title, unit="bytes")` | セッションパネルの進捗バー。`unit` は `bytes` または `items`（1.7） |
| `pokeldn.panel.show(id, title, lines=(), *, image, chips, tone)`、`clear(id)`、`annotate(name, *, state, text, image)`、`unannotate(name)` | ツールのページのステータスカードと、ライブラリのファイルへのメモ（1.8、[ステータスカード](#status-cards-18)） |
| `pokeldn.ldn.network` | IPv4 スタックを使った LDN ネットワークへの参加とホスト（1.7、[LDN ネットワーク](#ldn-networks-17)） |
| `pokeldn.ldn.radio_lost()` | ラジオが実行から離れた理由。空の場合もある（1.7） |
| `pokeldn.popup.cancel(id)`、`available()`、`set_dispatcher(dispatch)` | ポップアップを閉じ、そのコールバックを破棄する。アプリがポップアップを表示できるかどうか。コールバックが実行される場所（1.2） |

この表と次の表が、サポートされる範囲です。プラグインが pokeldn からインポートするそれ以外のものは、どのリリースでも変わる可能性があります。

## プログラム向けインターフェース（1.3、1.4 で Bank を追加、1.5 で言語パック）

プラグイン、または pokeldn のチェックアウトを元にした別のプログラム（タッチスクリーンのフロントエンドやスクリプトなど）は、アプリと同じコードで配布ファイルやポケモンを作り、ボードを管理できます。これらの名前は API メジャーバージョン 1 の中で同じ意味を保ちます。`tests/test_plugin_surface.py` がすべての名前を解決できることを確かめます。

| 名前 | 内容 |
|---|---|
| `python -m pokeldn --list`、`python -m pokeldn TOOL -- ARGS` | 全ツールのキー。また、設定のトレーナーと `--` の後の引数を使ってツールを1つ実行する |
| `pokeldn.app.i18n.t(key, default, **fmt)`、`select(plugin_id)` | インラインの英語の既定値と名前付きの書式を使う文字列の参照。プラグイン id で有効な言語パックを選ぶ。id を空にすると英語を選ぶ（1.5） |
| `pokeldn.app.catalog.GAMES` | 組み込みゲーム。各 `Game` の `.tools` は `Tool` を保持する。`bin/` 以下のスクリプトとそのフィールド。アプリが渡すフラグは名前を保つ |
| `pokeldn.pokemon.SERVICE.species(game)`、`.names(game, kind)` | 種族と、そのゲームが持つ `moves`、`items`、`bag`、`held`、`balls` を `{"id", "name"}` の一覧で返す |
| `pokeldn.pokemon.SERVICE.options(game, species, trainer)` | その種族のオファーが持てるフォルム、性格、特性、道具、ボール |
| `pokeldn.pokemon.SERVICE.make(game, species, trainer, level, shiny, nickname, version, options)` | そのゲームで正当なポケモン。`options` は性格、特性、性別、持ち物、ボール、なつき度、技、個体値、努力値を受け取る。戻り値はその値と、書き出したパスの `"file"` |
| `pokeldn.pokemon.SERVICE.paste(game, text, trainer)` | `text` 内の Showdown 形式のセット。それぞれ `make` の値と `errors` を持つ |
| `pokeldn.pokemon.BuilderError` | PKHeX が拒否するか見つからない場合に、プレイヤー向けの文で発生する |
| `pokeldn.pokemon.NATURES` | 25 個の性格名。各レコードが保存するインデックス順 |
| `pokeldn.gifts.load(path, game=)`、`pokeldn.gifts.save(path, gift)` | `.pokegift`、`.wc3`、`.wc8` の読み書き（[ふしぎなおくりものファイル](gifts.md)） |
| `pokeldn.swsh.gift_builder.KINDS`、`PRESETS`、`PRESET`、`OUTFITS` | ソード・シールドのビルダーの配布種別 `(key, label, icon)`、プリセット（`.key`、`.label`、`.group`、`.summary`、`.state`）、衣装 |
| `pokeldn.swsh.gift_builder.blank(**changes)`、`record(state)`、`compile(state)`、`describe(state, name)` | フォームの状態、その720バイトの封印済みレコード、`gifts.save` に渡す配布データ、プレイヤー向けの行。不正な状態は、フィールド名を示す `ValueError` を発生させる。ポケモンの状態は `nature`（0～24）を受け取る（省略するとゲームが決める） |
| `pokeldn.swsh.events.load()`、`by_key()`、`gift(key)` | ソード・シールドの公式イベントカードと、1件ごとの配布データ |
| `pokeldn.frlg.gift.builder.KINDS`、`PRESETS`、`PRESET`、`GIVERS`、`CARTRIDGES` | ファイアレッド・リーフグリーンのビルダーの種別、プリセット、カードを渡す人物、ゲーム機のコードが対象にできるカートリッジ |
| `pokeldn.frlg.gift.builder.blank()`、`compile(state)`、`describe(state, name)` | ふしぎなおくりものカード、ふしぎなニュース、ゲーム機のコードについても同様 |
| `gui.board.ports()`、`identify(port, blink=)`、`detect_chip(port)` | シリアルポートと、ボードが動かしているもの（`.firmware`、`.firmware_version`、`.protocol`、`.current`）。プロトコルはこの pokeldn のもの |
| `gui.board.download_firmware(say=)`、`image_for(kind, chip)`、`image_version(path)`、`flash(port, kind=)` | リリースのイメージを SHA-256 で検査したもの、チップに合うイメージ、そのバージョン。チップを確認したうえで書き込む |
| `pokeldn.app.bank.library()` | 既定の Bank フォルダー（`Path`）。一覧表示では作成されない（1.4） |
| `pokeldn.app.bank.Entry`、`Imported`、`ImportedPokemon` | 以下で説明する Bank の結果の型（1.4） |
| `pokeldn.app.bank.Imported.count(state)` | その状態の取り込み結果の数（1.4） |
| `pokeldn.app.bank.entries(folder=)`、`find(entry_id, folder=)`、`data(entry)` | 記録を新しい順に返す。1件の記録、または `None`、およびそのネイティブのバイト列（1.4） |
| `pokeldn.app.bank.import_file(path, trainers, folder=)` | ポケモンファイルまたはセーブデータを取り込む。元のファイルは変更しない。戻り値は `Imported`（1.4） |
| `pokeldn.app.bank.deposit(game, source, info, folder=)` | 受け取ったファイルのコピーと PKHeX の検査結果を Bank に預ける。戻り値は `Entry`。すでに預けてあるバイト列の場合は `None`（1.4） |
| `pokeldn.pokemon.SERVICE.check(game, path)` | 受け取ったネイティブのレコードを PKHeX で検査する。戻り値は、適法性の判定を含む `bank.deposit` 用の `info` 辞書（1.4） |
| `pokeldn.app.bank.destinations(entry, trainer)`、`offer(entry, game, trainer)` | ゲームごとの変換の可否と、1つのゲームへの変換済みオファー（1.4） |
| `pokeldn.app.bank.edit(entry, fields)`、`remove(entry_id, folder=)` | ニックネーム、レベルの引き上げ、技、持ち物を編集する。戻り値は更新された `Entry`。記録とそのメタデータを明示的に削除する（1.4） |
| `pokeldn.app.bank.TradeSession(offers, folder=)` | 1回の実行のための、順序付きオファー辞書のスナップショット。Bank ID のないオファーも含む（1.4） |
| `pokeldn.app.bank.TradeSession.advance(count)` | 累積の交換完了数を反映する。戻り値は `.completed`（処理済みの最大数）。同じ数や古い数の場合、記録は変更されない（1.4） |
| `pokeldn.app.received.trades_done(line)` | ランチャーの `[done] trade N complete` 行にある累積数。ない場合は `None`（1.4） |

ランチャーの実行は、アプリからでも `python -m pokeldn` からでも、1つのプロセスです。停止するには、その PID に SIGINT を送り、15秒後に SIGTERM を送ってください。名前でランチャーを探して終了させないでください（`pkill -f`）。ほかのプログラムが開始した実行まで終わってしまいます。

## Bank 連携（1.4）

外部の Python フロントエンドは、デスクトップ版の Bank と同じモジュールである `pokeldn.app.bank` を使います。オファーのキューとプロセスの管理は、そのフロントエンドが行います。アプリの設定、キューのヘルパー、Bank のメタデータファイルは内部用のままです。変換には既存の PKHeX のヘルパーを使います。ソースからインストールした場合は、`dotnet build -c Release services/pkhex` でビルドしてください。

`Entry` は `id`、`path`、`game`、`species_id`、`shiny`、`summary`、`legal`、`tracker`、`created`、`origin`、`home_tracker` を持ちます。`game` はカタログのキー、`created` は秒単位の Unix 時刻、`path` はネイティブのレコードです。`home_tracker` は、[HOME トラッカーの設定](gui.md#the-home-tracker-setting) で説明されている `empty`、`made`、または空文字列です。

`Imported` は `name`、`save`、`pokemon` を持ち、`count(state)` も使えます。各 `ImportedPokemon` は `where`、`species`、`species_id`、`shiny`、`state`、`game`、`detail` を持ちます。状態は `banked`、`already`、`refused` のいずれかです。拒否はレコードごとに行われ、1匹のポケモンが拒否されても、セーブデータ内の他のレコードの取り込みは妨げられません。`trainers` は、ゲームのキーからトレーナー辞書（`ot`、`tid`、`sid`、`language`、`gender`）への対応表です。ポケモンのビルダーが受け取るものと同じ形式です。取り込み済みのレコードは、その後に交換で手放したものも含めて記憶されます。

`destinations` は、ゲームをキーとする辞書を返します。各値は `ok`、`reason`、`home_tracker` を含みます。`offer` は `file`、`summary`、`species`、`shiny`、`legal`、`encounter`、`moves`、`report`、`home_tracker`、`bank` を含む辞書を返します。`file` は用意されたネイティブのレコードのファイル名です。`bank` は元の Bank エントリーの ID です。オファーを用意しても、元の Bank のレコードはそのまま残ります。オファーのファイルは、その実行が終わるまで保管し、ランチャーの `--offer-file` 引数に渡してください。Bank のオファーは、識別情報を保つために `--fresh-pid` を付けずに実行してください。デスクトップ版のコマンドビルダーはこれを自動で行います。`python -m pokeldn TOOL` は既定値を組み立てるだけで、Bank のオファー辞書を受け取りません。そのため、Bank の交換を起動するフロントエンドは、カタログのツールのスクリプトを直接使います。

```python
from pokeldn.app import bank, received

trainer = {"ot": "ASH", "tid": 1111, "sid": 2222, "language": 2, "gender": 0}
games = ("frlg", "lgpe", "swsh", "bdsp", "pla", "sv", "za")
result = bank.import_file("main", {game: trainer for game in games}, folder="tradebox-bank")
entry = next(iter(bank.entries(folder="tradebox-bank")), None)
if entry is None:
    raise ValueError("No Pokemon available in the Bank.")
routes = bank.destinations(entry, trainer)
if not routes["swsh"]["ok"]:
    raise ValueError(routes["swsh"]["reason"])
offer = bank.offer(entry, "swsh", trainer)
session = bank.TradeSession([offer], folder="tradebox-bank")

def on_launcher_line(line):
    count = received.trades_done(line)
    if count is not None:
        session.advance(count)
```

交換ごとに `TradeSession` を1つ作り、実行中のオファーの順序を正確に渡してください。`bank` キーを持たない、組み立て済みのオファーも含めます。渡すのは確定した交換完了数だけで、プロセスの終了コードや、キューに入った交換の数は渡さないでください。失敗または停止した実行では、完了していないレコードが Bank に残ります。累積数は途中のメッセージを飛ばすことがあります。スナップショットしたオファー以外に生成された交換は、追加のレコードを削除せずに数えられます。`advance` は負の数や整数でない数を `ValueError` で拒否します。進捗は、レコードの削除が成功するたびに進むため、`OSError` が出ても同じ数で再試行できます。

ファイルアクセスのエラーは `OSError` を発生させます。PKHeX の失敗、および拒否された変換や編集は `pokeldn.pokemon.BuilderError` を発生させます。`deposit` は `pokeldn.pokemon.SERVICE.check` の検査データを受け取り、追加の適法性チェックは行いません。ユーザーが指定したセーブデータとポケモンファイルは `import_file` から取り込みます。`folder` を受け取るメソッドは、パスを受け取るか、既定では `library()` を使います。`data`、`edit`、`destinations`、`offer` は `Entry` が持つパスを使います。ヘルパーの出力とフロントエンドのほかのアプリケーションファイルも分けたい場合は、pokeldn をインポートする前に `POKELDN_DATA` を設定してください。

Bank の操作は同期的です。PKHeX の処理は、フロントエンドの UI スレッドの外で実行してください。Bank フォルダーに同時に書き込めるのは1つの書き手だけです。別々のアプリからの同時の取り込み、編集、交換はサポートされません。デスクトップ版のアプリが動いているときは、TradeBox 用に別のフォルダーを使ってください。外部のキューは完了したオファーを自分で破棄します。`TradeSession` が変更するのは Bank だけです。
