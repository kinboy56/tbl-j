---
title: Switch タイトルのリバースエンジニアリング
nav_order: 3
---

# Switch タイトルのリバースエンジニアリング

市販の Switch ゲームのコードを、展開せずに読む手順です。対象は Unity/IL2CPP のタイトル (BDSP、NSP 計 7.3 GB) と、
ネイティブ C++ のタイトル (ソード・シールド、13.3 GB の XCI) です。

## 作業の順序

1. 手元にある定数を使って、公開されているソースを検索します。コード検索では、要約表からリンクされていない
   ゲームごとのページも含めて NintendoClients wiki の中を探せます:

       gh search code "<a constant, a field name, a class name>" --limit 20
       gh api repos/kinnay/NintendoClientsWiki/contents --jq '.[].name'
       gh api repos/kinnay/NintendoClientsWiki/contents/<Page>.md --jq .content | base64 -d

   要約表には導出された値が載っており、
   ゲームごと・Pia のバージョンごと・アプリケーションデータの各ページには規則が載っています。
2. ゲームのコードが既にダンプされていないか探します。BDSP では C# による再実装 (`TeamLumi/opendpr`) のほうが
   IL2CPP の出力より速く読めます。ゲームキーで検索すると、サードパーティ製のクライアントが見つかります
   (ソード・シールドは LAN モードのクライアントが 4 つです)。
3. 1 つのビルドの実行ファイルとメタデータを取り出し、先にすべてに名前を付けます。
4. バイナリを読んで転写を検証し、書かれていないもの (BDSP の `cryptoKeyDataSeed` と
   そのバージョン規則) を見つけます。
5. キー空間の検索は、その後にします。入力の 1 つを誤った値に固定すると、確信を持った否定の結果が出てしまうためです。

Pia LDN のタイトルを復号するには、パスフレーズ、`cryptoKeyDataSeed`、ローカル通信バージョン、
アドバタイズメント、送信者ごとの MAC が必要です ([Pia 層](pia.md))。

## 実行ファイルの取り出し

### NSP から

NSP は NCA をまとめた PFS0 アーカイブです。実行ファイルは数 GB ある NCA の末尾近くにあります。

1. PFS0 のヘッダーを直接解析します。hactool の `--listfiles` が示すオフセットは、データの基点からの相対値です。
2. `<rights id>.tik` から title key を復号します (rights id は NSP のファイル名)。
   暗号化された鍵は `+0x180`、rights id は `+0x2A0` にあり、いずれも絶対位置です。
   最終バイトは鍵の世代 *n* で、`titlekek_{n-1}` を使います。hactool は暗号化された鍵を `--titlekey` で受け取ります。
   `xci_read.py` は ticket を自分で読みます。
3. スパースファイルを作ります。NCA の先頭を `dd` でコピーし、`truncate -s` で実サイズに伸ばし、
   必要な範囲を `dd ... seek_bytes conv=notrunc` で書き込みます。ディスク上は 109 MB ですが、2.7 GB のファイルとして扱われ、
   hactool の境界チェックを通ります (`du -h` は実際のディスク使用量を示しますが、`ls -l` は示しません)。

### XCI から

`tools/switch/xci_read.py` は HFS0 パーティション (または NSP の PFS0) をたどり、NCA ヘッダーをその場で復号します
(`header_key` による AES-128-XTS、ビッグエンディアンのセクタ tweak)。
そのうえで title id、コンテンツ種別、鍵の世代、各セクションのオフセット、カウンタ、鍵を表示します:

    ./.venv/bin/python tools/switch/xci_read.py <the.xci> --keys prod.keys --type Program

カートリッジの NCA は rights id がゼロで ticket を持ちません。ボディ鍵は key area のスロット 2 にあり、
`key_area_key_application_<max(crypto_type, crypto_type2) - 1>` で暗号化されています。
アップデートの exefs は Program NCA のセクション 0 で、通常の CTR PartitionFS です:

    ./.venv/bin/python tools/switch/xci_read.py <the.xci> --nca <id> --exefs 0 --extract main

注意点: FS ヘッダーの `fs_type` は +0x2、ハッシュ種別は +0x3 にあります
(hactool の構造体では両者の名前が入れ替わっています)。8 バイトのセクションカウンタは逆順で使います。

### RomFS から

NCA のセクションは、タイトル鍵を使った AES-128-CTR で暗号化されています
(カウンタはセクションの CTR 値と `offset >> 4` のビッグエンディアン)。そのため、どの範囲も単独で復号できます。`tools/switch/romfs_read.py` は、
コンテナから直接 4.2 GB の RomFS をたどり、grep し、展開します。ヘッダーのサイズフィールドが `0x50` でなければ例外を投げます。
誤った鍵・オフセット・カウンタは、この時点で最初に表面化します。`prod.keys` に 34 桁の 16 進数の行があると hactool は中止し、
テーブルの欠けたスパースファイルに `--listromfs` を使うとセグフォルトします。

## 最初にすべてに名前を付ける

市販の Unity タイトルには、独立した 2 つの命名レイヤーがあります:

- IL2CPP のメタデータ (C# 側): `global-metadata.dat` (romfs の `Data/Managed/Metadata`) と、
  同じビルドの実行ファイルから、Il2CppDumper がすべてのクラスとメソッドの RVA を得られます。
- C++ RTTI (ネイティブ側): `type_info` の名前ポインタと vtable の `type_info` ポインタは、通常のリロケーションです。
  `tools/switch/rtti_names.py` は、BDSP で 279 個の `nn::pia` クラスと 2269 個の仮想メソッドを、
  ソード・シールドで 252 個と 2036 個を見つけます。

所有クラス (`LocalProtocol` または `LanProtocol`) を見ると、ほぼ同じ形の LAN 版とは別の、
LDN のセッション鍵導出であることがわかります。空きのある vtable は多重継承のグループで、
その空きは次のサブオブジェクトの offset-to-top と typeinfo に当たります。

## メタデータにしかない定数

`[Serializable]` の通常の C# クラス (`ScriptableObject` ではないもの) が持つ定数の `byte[]` の既定値は、
コンストラクタが `<PrivateImplementationDetails>` の static フィールドに対して `RuntimeHelpers.InitializeArray` で書き込みます。
このバイト列は `global-metadata.dat` のフィールド既定値テーブルにしか存在しません。
コンストラクタの ADRP/LDR ペアは、メタデータの使用スロットを指し、そのスロットは
`Field$<PrivateImplementationDetails>.<HEX>` に解決されます (`<HEX>` はデータの SHA-1)。値はフィールド名で読み、
ハッシュで検証します。BDSP の 16 バイトのゲーム鍵シードもこの形で格納されています。
ネイティブ C++ の定数は rodata にあり、クロスリファレンスで見つけます。

## ダンプしたバージョンを確認する

ベースゲームとアップデートは、別々の NSP になっています。BDSP のベースは 24 個のネットワークメッセージクラスを定義し、1.3.0 は 65 個を定義します。
`PosData` は `Vector3 pos, short rotY` から `ushort posX, ushort posZ, short rotY` に変わり (16 バイトから 6 バイトへ)、
`JoinData` は 2 つのフィールドが増えてアラインメントが失われました。そのため、ベースのダンプのレイアウトでキャプチャをデコードすると、
もっともらしいが意味のない結果になります。確認は次の方法で行えます:

- 1 つのバージョンにしかないクラスを `strings global-metadata.dat | grep` で探します。BDSP のベースには
  `NetDataTradeStandbyData` があり (アップデートで削除された)、
  `NetPlayerNameData` はありません (1.3.0 で追加されたクラスです。1.3.0 のメタデータは 12,496,504 バイトです)。
- キャプチャした長さを構造体のサイズで割ります。72 バイトのポイントのリストは、6 バイトの要素が 12 個であり、
  16 バイトの要素ではありえません。

定数は残ります (BDSP のベースのメタデータにあるキーシードは、アップデート済みの本体からのパケット 674 個すべてを復号します)。
構造に関わるもので、キャプチャで確認されていないものは読み直してください。

## ゲームアップデートの RomFS を読む

アップデートの RomFS は、ベースゲームの RomFS にパッチを当てる BKTR セクションです。末尾にある 2 つのテーブルは、セクションの通常の CTR で暗号化されており、
オフセットをキーとする 0x4000 バイトのバケットに分かれています (hactool `bktr.c`)。この 2 つのテーブルが全バイトを決めます。
リロケーションテーブルは、仮想範囲をアップデートのセクションまたはベースの RomFS に対応づけます (`is_patch`)。
サブセクションテーブルは、アップデートの物理範囲それぞれに、セクションカウンタのバイト 4..8 を置き換えるカウンタ値を与えます。
`tools/switch/bktr_read.py` は両方を組み合わせて、シーク可能な 1 つのセクションにします:

    ./.venv/bin/python tools/switch/bktr_read.py UPDATE.nsp --base BASE.nsp \
        --extract /Data/Managed/Metadata/global-metadata.dat

ベースは、アップデートのタイトルの Program NCA のうち、CTR の RomFS セクションを持つものです。XCI のアップデートパーティションには
システムタイトルの Program NCA が入っており、最初に見つかる Program NCA はそのうちの 1 つになります。

## 呼び出し元を探す

イメージ内のワードを走査し、`bl` (`100101` と符号付き 26 ビットのワードオフセット) と、末尾呼び出しの `b` (`000101`) を探します。
1 行の C# の転送メソッドは末尾呼び出しにコンパイルされます。BDSP の `ANetData<SelectData>$$SendReliableData` と
`ANetData<TransitionData>$$SendReliableData` は、`bl` による呼び出し元がなく、`b` による呼び出し元が 9 個あります。
それらはいずれもユニオンルームのコンテキストメニューにあります。どこからも分岐されていないメソッドは、
ADRP/ADD のペア (`arm64_xref.py`) を通じて `Action` として登録されたデリゲートかもしれません。
`TradeStateModel` の `WriteSaveData`、`FirstSave`、`SendTradeState` がその例です。

注意点: `arm64_xref.function_start` は `udf` のパディングを逆にたどるため、格納命令を直前の関数のものとして扱うことがあります
(プロローグを確認してください)。関数の範囲を「入口から N バイト後まで」と決めると、`ret` を越えて次の本体に入り込みます。
`mov w0, #0x80` は ARM64 では ORR のイミディエイトであり、バイトパターンの走査では見えません。
そのため、定数は逆アセンブラでデコードしてください。

## nnSdk への呼び出し

nnSdk への呼び出しは、シンボルを指定する JUMP_SLOT リロケーションで埋められた GOT スロットを経由します。
そのスロットをクロスリファレンスし (`tools/switch/nso_imports.py`)、続けて 1 つだけの PLT スタブの呼び出し元を数えます。

## ツール

読み取りには `pokeldn.switch` (コンテナと NCA、RomFS、NSO、Trinity パック) を使い、ツールはその上のコマンドラインになっています。
タイトル固有のファイルが必要なプログラムは、ユーザーのダンプからそれを通して読みます。
上で挙げた `xci_read.py`、`romfs_read.py`、`bktr_read.py`、`rtti_names.py` のほかに、すべてオフラインで動くツールがあります:

    tools/switch/nso_read.py     an NSO's three segments decompressed (pure-Python LZ4) at their
                                 memory offsets: a file offset is an address
    tools/switch/nso_relocs.py   MOD0 -> dynamic -> relative relocations, RELA and RELR. Vtable slots
                                 fill at load time, so "who points here" is a relocation question;
                                 modern titles use RELR, where a RELA-only reader finds nothing
    tools/switch/arm64_xref.py   ADRP(+ADD|+LDR) cross-references, BL call graph, function starts
    tools/switch/arm64_dis.py    capstone window disassembly
