---
title: ルビー・サファイア・エメラルド
nav_order: 11
nav_exclude: true
search_exclude: true
---

# ルビー・サファイア・エメラルド

ルビー・サファイア・エメラルドには Switch 版がありません。pokeldn には、
[pret/pokeemerald](https://github.com/pret/pokeemerald) と
[pret/pokeruby](https://github.com/pret/pokeruby) から用意したサポートを入れてあります。これは、Nintendo がこれらを FireRed・LeafGreen と同じ
GBA アプリケーションに収録して出荷する場合に備えたものです。このページの内容はどれも実機で計測したものではありません。すべてのアドレスと数値は GBA 英語版のものです。
FireRed の Switch 版は、コードが移動した新しいリビジョン
(`0x0A`) です。そのため Emerald の Switch 版も同じように違うはずです。

アプリは、ソースチェックアウトから実行したときにだけタイトルを表示します
([未公開のタイトル](gui.md#unreleased-titles))。このページはサイトのナビゲーションから除外されています。

## GBA アプリケーションでこれらが動くと見込まれる理由

FireRed と LeafGreen のアプリケーションには、第3世代の5つのゲームコード
(ルビー `AXV?`、サファイア `AXP?`、エメラルド `BPE?` を含む) ごとのプレイレポートテーブルがすでに入っています
([コンソール上のコード](frlg_rom.md))。FireRed と LeafGreen の12タイトルはすべて、同じ最初のローカル通信 ID
`0x01006fa0233f8000` を持っています ([ROM マップ](frlg_rom_map.md))。ランチャーのスキャンは、
ID を指定しない場合に存在する唯一のネットワークに参加します。

## 各ゲームがリンク越しにできること

| | ルビー / サファイア | エメラルド |
|---|---|---|
| ワイヤレスアダプター (RFU) のコード | なし: `pokeruby` は SIO マルチプレイヤーケーブルのみ | FireRed と同じ `link_rfu_2.c` |
| リンク交換 | ケーブル | FireRed と同じダイレクトコーナーとユニオンルーム |
| ふしぎなおくりもの (ふしぎなカード、ふしぎなニュース) | なし | FireRed と同じプロトコルと構造体 |
| ふしぎなできごと | ケーブル経由の e-Reader。"MYSTERY EVENT IS EXCITING" で有効化 | タイトルメニューからは利用不可 |

pokeldn のランチャーはワイヤレスアダプターのプロトコルを話すため、到達できるのはエメラルドです。
Switch 版がルビー・サファイアのケーブル接続をどう扱うかは不明です。

RS のケーブル通信プロトコルは、pokeldn の交換エンジンがリンクより上で使うものを
FireRed・Emerald と共有しています。`LinkPlayer` レコード (GameFreak のマジックを含む)、ブロックサイズ 200、200、100、220、40、
リンクコマンドのワード、交換の3つの200バイトのパーティブロック、220バイトのメールブロック、40バイトのリボンブロックです [pokeruby link.c, trade.c]。
RS は `LinkPlayer` に11バイトの名前を書き込みます。その位置は FireRed と Emerald が進行フラグとして読むバイトに重なるため、
両者は RS の相手に対してそれらのフラグを0にします。

## 識別子

| 項目 | ルビー | サファイア | エメラルド |
|---|---|---|---|
| ゲームコード | `AXV` + 言語 | `AXP` + 言語 | `BPE` + 言語 |
| ヘッダーのタイトル | `POKEMON RUBY` | `POKEMON SAPP` | `POKEMON EMER` |
| `gGameVersion` | 2 | 1 | 3 |
| `LinkPlayer.version` | `0x4002` | `0x4001` | `0x4003` |

言語を示す文字は `E`、`F`、`D`、`I`、`S`、`J` のいずれかで、FireRed のカートリッジと同じです。バージョンと
言語の番号は3つのデコンプすべてで同じです [include/constants/global.h:8-12]。

## ホストへの参加

広告レコードと `RfuGameData` のレイアウトは FireRed のものです [include/link_rfu.h:80-113]。
交換相手を探すとき、Emerald は広告バージョンが 3 でないホストを、
プレイヤーがチャンピオンで、ホストが `canLinkNationally` を広告している場合を除いて「準備中」として表示します [union_room.c:1266-1283]。
交換ボードも同じルールを適用します [trade.c:2514-2521]。

`--version emerald` (または `ruby`、`sapphire`) を付けてランチャーを実行すると、そのバージョンを広告し、
プロファイルから `canLinkNationally`、`hasNationalDex`、`gameClear` を広告レコード
(`[16:20]` にあるワードのビット 16、17、24) と NI のゲームデータ (`RfuGameData` のビット 7、8、9) に設定します。
FireRed と LeafGreen は、市販の本体が受け入れてきたバイトをそのまま保ち、それらのビットは0のままにします。

## 交換

通信の仕組みは FireRed と同じです。リンクコマンド、ブロック交換の順序とサイズ、トレーナーカード
(`0x60` にあるふしぎなカードの ID) です。異なるルールは2つあります。

- 全国図鑑がなければ、Emerald は相手のタマゴや、ホウエン図鑑に含まれないポケモンを拒否し、
  その後 A ボタンが押されるまで待ってから `READY_CANCEL_TRADE` を送信します [trade.c:1585-1589]。本体は全国図鑑の状態を
  `LinkPlayer.progressFlags & 0xF` で報告します。`pokeldn.rse.constants` が202件のホウエン図鑑を保持し、
  ホストは、そのような本体が拒否するオファーをログに記録します。
- GBA の Emerald は交換後の待機ラウンドを5回実行します (`trade.c` のステート 1、41、5、8 と交換メニューの初期化)。
  Switch の FireRed は6回で、ステート 43 のラウンドのあとに `svc_FinishSave` が続きます。`--save-rounds N` は、ホストが待つ回数を設定します (既定は 6)。
  同じシステムコールで保存する Switch の Emerald は6回目のラウンドを含むと見られますが、
  回数は本体を計測するまでわかりません。

バージョン 3 のとき、Emerald の本体は `0x3A` にフロンティアシンボル、`0x3C` にバトルポイントを描いた私たちのトレーナーカードを表示します。
4 か 5 のときは FireRed のカードを描きます。Emerald の受信キューは32スロットで、FireRed の20スロットに対し、
シリアル `0x0000` を `0x0002` や `0x7F7D` と並べて受け入れます。

## ふしぎなおくりもの

リンク転送、サーバーとクライアントのスクリプト、332バイトの `WonderCard`、444バイトの
`WonderNews` と `MysteryGiftSave` (`0x36C`) は、フィールドまで含めて FireRed と同じです。違うのは3点です。

`+0x0C` と `+0x10` のゲームデータには、Emerald ではギフトの種類のビットが入ります。
カードは 4 と `0x200`、ニュースは 5 と `0x201` です。FireRed は 1 とバージョンコードを送ります [mystery_gift.c:346-363]。
Emerald のチェックは、カードでビット 2 と `0x380`、ニュースでは何も要求しません [mystery_gift.c:390]。
ホストは、本体のゲームコードが示すゲームのルールを適用します。
本体のバージョンは、そのゲームコードから読み取ります。

ギフトスクリプトが指定するフラグ、変数、スペシャルは、別の番号に置かれています。pokeldn が出力するフィールドスクリプトのオペコードは
同じで、違うのは `0xC7..0xCC`、`0xD0`、`0xD3..0xD5` だけです。Emerald は `0xE3` まで動作します。

| | FireRed / LeafGreen | エメラルド |
|---|---|---|
| カードのflagId 1000 (`sReceivedGiftFlags[0]`) | `0x2A7` | `0x13A` |
| `FLAG_MYSTERY_GIFT_DONE` | `0x3D8` | `0x1E4` |
| ギフトのカーソル変数 | `0x40B6..0x40BC` | `0x40DE..0x40E4` |
| 最後のどうぐ | 374 | 376 (`ITEM_OLD_SEA_MAP`) |
| `StartLegendaryBattle` / `GetMysteryGiftCardStat` | 312 / 390 | 314 / 392 |
| `special` のインデックスの確認 | `gSpecialsEnd` で上限あり [scrcmd.c:101] | なし: 527 個のスペシャル [scrcmd.c:119] |
| SaveBlock1 内の RamScript | `0x361C` | `0x3728` |

`pokeldn.frlg.gift.gift_composer` は `GameRules` テーブルを受け取ります。`pokeldn.rse.constants.EMERALD_RULES` は
Emerald の番号でギフトをコンパイルします。ペイロードにはそのゲームが含まれ、
ホストは何かを送信する前に、別のゲームの本体を拒否します。

Emerald のふしぎなニュースには報酬がありません。ステップカウンターを増やす処理はなく、
どのスクリプトもそのきのみを読み取りません。

### ギフト

pokeemerald に含まれる公式の4つのギフトスクリプト (`data/scripts/gift_aurora_ticket.inc`、
`gift_mystic_ticket.inc`、`gift_old_sea_map.inc`、`gift_altering_cave.inc`) は、
pokeldn 独自のテキストを使って `pokeldn.rse.gifts` にオペコード単位で再構築されています。`tests/test_rse.py` はデコンプのソースを組み立て、
バイトコードを比較します。ポケモンセンター2階のふしぎなおくりもの係は、保存されたスクリプト
(`trywondercardscript`、`CableClub_EventScript_MysteryGiftMan`) を実行します。

| プリセット | カードのflagId | 内容 |
|---|---|---|
| オーロラチケット | 1000 | チケットと `FLAG_ENABLE_SHIP_BIRTH_ISLAND` |
| ミスティックチケット | 1001 | チケットと `FLAG_ENABLE_SHIP_NAVEL_ROCK` |
| ふるびたかいず | 1002 | 地図と `FLAG_ENABLE_SHIP_FARAWAY_ISLAND` |
| かわりのどうくつのうわさ | 1003 | かわりのどうくつの野生ポケモンテーブル9種のうち次のもの |
| エオンチケット | 1007 | ゲーム側の配布 [cable_club.inc:33]: チケットと `FLAG_ENABLE_SHIP_SOUTHERN_ISLAND` |
| マスターボール、ふしぎなアメ x5、なみのりピチューのタマゴ | 1004..1006 | `EMERALD_RULES` のもとで構成 |

    POKELDN_RADIO=esp32:auto ./.venv/bin/python -u bin/frlg_mg_host.py --live --version emerald \
        --gift-file old-sea-map.pokegift --keys "$HOME/Documents/Switch/23.0.0 keys/prod.keys"

Emerald のギフトファイルは `emerald` という1つのバリアントを持ちます ([ふしぎなおくりもののファイル](gifts.md))。ネイティブコード、
セーブのバックアップ、FireRed のプリセットは FireRed のアドレスを指定しているため、Emerald では拒否されます。

## セーブデータ

フラッシュのレイアウト、セクターの署名 `0x08012025`、チェックサムは FireRed と同じです [save.c]。SaveBlock1 は
`0x3D88` バイト、SaveBlock2 は `0xF2C` バイトです。パーティ数 `0x234`、パーティ `0x238`、お金 `0x490` (SaveBlock2 の `+0xAC` と XOR)、
フラグ `0x1270`、変数 `0x139C`、ゲーム統計 `0x159C`、ふしぎなおくりもの `0x322C` です。
SaveBlock2 には `filler_B20` がないため、残すネイティブのペイロードは別の置き場所が必要になります。

## pokeldn にない Emerald のリンク機能

レコードミックス (レコードコーナー)、きのみブレンダー、コンテスト、バトルタワーのリンクマルチバトル、
きのみクラッシュ、ポケモンジャンプ、ドードリオのきのみひろいが、Emerald ではワイヤレスアダプター経由で動作します (リンクグループ 12 から 21、
アクティビティ 14 と 23 から 29)。FireRed にも最後の3つが含まれています。

## 未解決の事項

- Switch 版が存在するか、そのタイトル ID、ゲームコード、ソフトウェアバージョン。
- FireRed のアプリケーションの通信 ID、シーン ID、広告のパッキングを維持するか。
- Switch 版がルビー・サファイアのケーブル接続をどう扱うか。
- Switch の Emerald の交換後の待機回数。
- Switch の Emerald のすべての ROM と RAM のアドレス。ネイティブコードのペイロードにはこれらが必要です。
- 日本語版 Emerald のふしぎなおくりもののアクティビティとカードのサイズ。
- Switch 版が、ふしぎなおくりものメニューに対する Emerald のゲーム内の制限 (`FLAG_SYS_MYSTERY_GIFT_ENABLE`、
  "LINK TOGETHER WITH ALL" で設定 [easy_chat.c:693]) を維持するか。
