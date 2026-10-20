---
title: コード署名ポリシー
nav_order: 12
---

# コード署名ポリシー

無償のコード署名は [SignPath.io](https://about.signpath.io) が提供し、証明書は [SignPath Foundation](https://signpath.org) によるものです。

デスクトップアプリの Windows 版である `pokeldn-windows-x64.zip` は、
[リリース](https://github.com/Decryptu/pokeldn/releases) ページにあり、このリポジトリのソースからビルドしたバイナリに Authenticode 署名が付いています。対象は `pokeldn.exe`、
PKHeX ヘルパーの `pokeldn-pkhex.exe`、Unicorn ライブラリの `unicorn.dll` です。
このリポジトリの `v*` タグから、
[release.yml](https://github.com/Decryptu/pokeldn/blob/main/.github/workflows/release.yml) を使って GitHub Actions がこれらをビルドします。
SignPath は、そのビルドが生成したファイルだけに署名し、他の場所でビルドされたものには署名しません。署名対象の設定は
[.github/signpath/artifact-configuration.xml](https://github.com/Decryptu/pokeldn/blob/main/.github/signpath/artifact-configuration.xml) にあります。
同梱されている第三者のファイルを変更せずに含めている場合は、そのファイルの発行元の署名がそのまま残るか、署名のない状態になります。

## チームの役割

| 役割 | メンバー |
|---|---|
| コミッターとレビュアー | [Decryptu](https://github.com/Decryptu) |
| 承認者 | [Decryptu](https://github.com/Decryptu) |

リリースごとの署名要求は、すべて手作業で承認されます。他の人によるコントリビューションは、
コミッターがレビューするプルリクエストとしてのみ、リポジトリに入ります。

## プライバシーポリシー

このプログラムは、利用者またはインストール・運用する人が特に要求しない限り、
ネットワーク上の他のシステムへ情報を送ることはありません。このプログラムが行う要求は次のとおりです。

| いつ | 送信先 | 送信される内容 |
|---|---|---|
| 起動時、Settings で Updates が有効なとき (既定で有効) | GitHub API と、インストール済みの各プラグインの更新先アドレス | 最新リリースの要求。プレイヤーやそのゲームに関する情報は含みません |
| プレイヤーがラジオのファームウェアやイベントのレイドデータをダウンロードするとき | GitHub | ダウンロード要求 |
| プレイヤーが Trade (Online) や GTS を使うとき | 公開の Nostr リレー ([オンライントレード](online.md)) | 暗号化された交換イベント (差し出すポケモンを含む) |
| プレイヤーがプラグインをインストール・更新するとき、またはプラグインのソースを閲覧するとき | そのプラグインまたはソースのアドレス | ダウンロード要求 |

GitHub へのリクエストには GitHub のプライバシーステートメントが適用されます。各リレーには、そのリレーの運営者のポリシーが適用されます。
本アプリは、解析情報もクラッシュレポートも送信しません。
