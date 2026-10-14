[English](FIXES.md) | **日本語** | [简体中文](FIXES.zh-CN.md)

> [FIXES.md](FIXES.md)（英語版が原文）の翻訳です。

# すべての修正の解説

このページは、本家のタグ `0.14.8601` の上に載るブランチ `fixes` の修正 116 コミットを解説する。
修正ごとに、何が誤っていたか、利用者にどう見えたか、どの図面で分かるか、何を変えたかを書き、
コミットへのリンクを付ける。短いまとめは [README](../README.ja.md) にある。

## 修正の見つけ方

- **独立した読み取り実装。** 149 図面（R13〜R2018）を LibreDWG と、独立した MIT ライセンスの読み取り実装
  [ACadSharp](https://github.com/DomCR/ACadSharp) 3.8.0 の両方で読み、
  図形ごと・表のレコードごと・ヘッダ変数ごとに比べた。
  違いはすべてビット単位まで追った。LibreDWG が誤っていることもあれば、仕様書
  （ODA の "Open Design Specification for .dwg files"。"ODA 20.4.x" の形で引用する）が誤っていることもあった。
  仕様書と AutoCAD のファイルが食い違うときは、ファイルを正とする。
- **AutoCAD 自身の保存。** AutoCAD が同じ図面を全部の版で保存したものがある。`sample_AC1014` …
  `sample_AC1032`（ACadSharp の見本）と `example_r14` … `example_2018`（LibreDWG の試験データ）。
  その一つを別の版に変換し、AutoCAD 自身によるその版の保存と比べれば、AutoCAD が変換のときに何をしているかが分かる。
- **往復。** すべての図面を R2000・R2004・R2010・R2013・R2018 で、直接（`dwgrewrite`）と
  DXF 経由（`dwg2dxf` のあと `dxf2dwg`）で書き、ACadSharp で読み直した。途中で消えたり変わったりしたものは不具合である。

図面名は試験用の図面集のもの。`sample_AC10xx` と `sample_base` は ACadSharp の見本、
`example_*`、`gh109_1`、`gh44-error`、`gh209_1`、`2000/TS1`、`2004/Line`、
`2018/Dynblocks` は LibreDWG の `test/test-data` にある。フィールド名と関数名は LibreDWG のもの
（主に `src/dwg.spec`、`src/decode.c`、`src/encode.c`、`src/in_dxf.c`、`src/out_dxf.c`）。DWG の
型：B はビット、BS/BL/BD はビット符号化の short/long/double、RC/RS/RL は生の char/short/long、TV/TU は 8 ビットと
UTF-16 の文字列、H はハンドル。

### 本家へのプルリクエスト

読み取り、R2000 への書き出し、DXF の修正のほとんどは、
[LibreDWG](https://github.com/LibreDWG/libredwg) にプルリクエストとして送ってある。そのコミットは本家の
`master` の上に載せ直し、途中で分けたりまとめたりしたものもあるので、ここの 1 コミットが
いくつかの PR の一部ずつに当たることがある。

| PR | 内容 |
|---|---|
| [#1440](https://github.com/LibreDWG/libredwg/pull/1440) | アジアのコードページ：2 バイト文字の先頭バイト |
| [#1441](https://github.com/LibreDWG/libredwg/pull/1441) | JSON の出力：NaN を出さない |
| [#1442](https://github.com/LibreDWG/libredwg/pull/1442) | 読み取りの修正（クラス、色、AuxHeader、ACIS、MTEXT、LAYER、EED など） |
| [#1443](https://github.com/LibreDWG/libredwg/pull/1443) | R2018 の複数行の ATTRIB/ATTDEF |
| [#1444](https://github.com/LibreDWG/libredwg/pull/1444) | 新しい版から R2000 への書き出し |
| [#1445](https://github.com/LibreDWG/libredwg/pull/1445) | DXF の読み込みと書き出し |
| [#1446](https://github.com/LibreDWG/libredwg/pull/1446) | TABLE、TABLECONTENT、TABLESTYLE |
| [#1447](https://github.com/LibreDWG/libredwg/pull/1447) | 不明なオブジェクトをプロキシとして |

R2004〜R2018 の書き出しの 62 コミット（[`1eb484a6`](https://github.com/monyuonyu/libredwg-fixes/commit/1eb484a6b53445a2f9be6d2a8e09d512b674d485) 以降）は本家にはまだ送っていない。
本家の LibreDWG は R2000 以前しか書けないためである。

## 目次

1. [クラス、ヘッダ、その他のファイルのセクション](#1-クラスヘッダその他のファイルのセクション)
2. [色](#2-色)
3. [文字列とコードページ](#3-文字列とコードページ)
4. [拡張図形データ（EED）](#4-拡張図形データeed)
5. [MTEXT、ATTRIB、ATTDEF](#5-mtextattribattdef)
6. [TABLE と TABLESTYLE](#6-table-と-tablestyle)
7. [ACIS のソリッド（3DSOLID、REGION、BODY）](#7-acis-のソリッド3dsolidregionbody)
8. [プロキシと不明なオブジェクト](#8-プロキシと不明なオブジェクト)
9. [その他の図形とオブジェクト](#9-その他の図形とオブジェクト)
10. [DXF の読み込み](#10-dxf-の読み込み)
11. [DXF の書き出し](#11-dxf-の書き出し)
12. [新しい版から R2000 への書き出し](#12-新しい版から-r2000-への書き出し)
13. [R2004〜R2018 への書き出し](#13-r2004r2018-への書き出し)
14. [ツールと試験](#14-ツールと試験)

---

## 1. クラス、ヘッダ、その他のファイルのセクション

### 1.1 R2004 以降のクラスのエントリ：`dwg_version` と `maint_version` は BL

- **原因:** ODA の仕様書（10.2）はクラスのエントリのこの 2 つのフィールドを BS としており、
  `read_2004_section_classes` と R2007 用の `read_2007_section_classes` もそれに従っていた。AutoCAD は
  BL で書く（ACadSharp も BL で読む）。BS と BL が一致するのは 256 未満の値だけ。エンコーダ
  （`encode_classes`）も BS で書いていた。2 つ目の不明な long は `unknown_2` ではなく
  `unknown_1` にもう一度入っていた。
- **症状:** AutoCAD の R2004 の見本では、クラス 508 の MLEADERSTYLE の `maint_version` が 255 を超える。
  BS で読むと 16 ビットを失い、以降のクラスはすべてゴミになり（番号 9974、dxfname "W'6..."）、
  番号 509 以上のクラスの図形（MULTILEADER、MESH、WIPEOUT、IMAGE、
  PDFUNDERLAY など）はすべて UNKNOWN_OBJ になった。R2004 で書いた図面は最初のクラスより後がすべて
  ずれ、独自クラスのオブジェクトは読み直すと不正な型になった（"Invalid class 523"）。
- **図面:** `sample_AC1018`。
- **変更:** 両フィールドを BL で読み書きする。2 つ目の long は `unknown_2` に入れる。
  この見本は 353 図形で ACadSharp と一致するようになった。
- **コミット:** [`19c7ab68`](https://github.com/monyuonyu/libredwg-fixes/commit/19c7ab689d0583a341b69867ea70d879ff97417e)（読み取り、PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442) に含む）、
  [`b25ba83f`](https://github.com/monyuonyu/libredwg-fixes/commit/b25ba83f095951ad1f2ddd3b0598775ab8ce4da3)（書き出し、本家にはまだ送っていない）。

### 1.2 クラスの数の上限が低すぎた

- **原因:** 巨大な確保を防ぐため（CWE-400）、デコーダはクラスを最大
  `100 + size / sizeof(Dwg_Class)` 個までしか許していなかった。クラス名の短い正しい図面はこれより多く持つ。
- **症状:** R2000 で書いた AEC の図面はクラスが 478 個あるが、上限は 385 だったので、
  後ろのクラスのオブジェクトはすべて読めなかった。
- **変更:** クラスのエントリは最低 13 ビット（BS の番号、BS のフラグ、BS の長さ付きの TV が 3 つ、B、
  BS の item class id）なので、`size` バイトのセクションに入るエントリは最大 `size*8/13` 個。
  メモリは入力に対して線形のまま。R2004 以降のクラスのセクションも同じ。
- **コミット:** [`4ff2ed83`](https://github.com/monyuonyu/libredwg-fixes/commit/4ff2ed83ccafa44fa85620ab1b2621dc681248bc) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 1.3 R2018 の AuxHeader：保守版番号は RL

- **原因:** R2018 の AuxHeader では `maint_version`、`maint_version_1`、
  `maint_version_2` は RL。LibreDWG はそれぞれを RS で読み、上位半分 3 つ（6 バイト）を末尾の
  `zero_18` として読んでいたので、`maint_version` より後がすべて 2 バイトずれていた。
- **症状:** `minus_1` が 0xffff0000 になり、TDCREATE/TDUPDATE はゴミで、R2000 で書いた R2018 の図面は
  そのゴミの日付を持ち、ACadSharp で読めなかった。
- **図面:** AutoCAD が同じ図面を R2013 と R2018 で保存したものは AuxHeader が 123 バイトと 129 バイト。
  差の 6 バイトは 3 つの保守版番号の上位半分である。
- **変更:** R2018 以降は、各保守版番号のすぐ後に上位半分を読む。
  これで `minus_1` は −1 になり、日付と HANDSEED はヘッダと一致する。
- **コミット:** [`c7efdf98`](https://github.com/monyuonyu/libredwg-fixes/commit/c7efdf987d7d3817456f94b789cc7345e558975f) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 1.4 R2007 以降のヘッダ：`unknown_54` … `unknown_57` は無い

- **原因:** ヘッダ変数では、R14 以降の全版で SHADOWPLANELOCATION の後に 4 つの BS `unknown_54` … `unknown_57`
  があることになっていた。R2007 以降はそこから文字列ストリームが始まる。
- **症状:** 4 つの値は文字列から読まれ（常に 1、180、36874、32843）、
  56 ビットの余分なデータとして書き戻されていた。
- **図面:** AutoCAD の R2007 以降の全図面（`example_2007` … `example_2018`、`sample_AC1021` …
  `sample_AC1032`）。
- **変更:** 4 つのフィールドは R14〜R2004 だけにした（`src/header_variables.spec`）。
- **コミット:** [`1a42025a`](https://github.com/monyuonyu/libredwg-fixes/commit/1a42025ad7e5294c05853c27372cd5ccb086a4dd) — 本家にはまだ送っていない。

### 1.5 FileDepList：機能名は T32 の文字列

- **原因:** R2004 の FileDepList セクションは、ファイル名と同じく機能名も、RL の長さと
  各文字列の文字（終端の NUL なし）で持つ
  （`02000000 09000000 "Acad:Text" 0f000000 "Acad:PlotConfig" 01000000 ...`）。機能名は
  ビット符号化の TV として読まれていた。
- **症状:** 機能名はすべて空になり、セクションの残りはずれた。
  書き戻すと壊れたセクションになった。
- **図面:** `2004/Line`。
- **変更:** 機能名をファイル名の文字列と同じく `FIELD_T32` で読み書きする。
  `bit_write_T32` は AutoCAD と同じく NUL なしの 8 ビットの形で書く。R2004 で書いた `2004/Line` は
  FileDepList が AutoCAD のものとバイト単位で一致するようになった。
- **コミット:** [`5fa2061b`](https://github.com/monyuonyu/libredwg-fixes/commit/5fa2061b838b24177d32bc6af74ea45224e9f1d3) — 本家にはまだ送っていない。

R2000 向けに書く AuxHeader は [12.4](#124-auxheaderr2000-でも-123-バイト) に、R2004 以降向けに書くクラスの
エントリ、ヘッダ、セクションの大きさは [13](#13-r2004r2018-への書き出し) にある。

## 2. 色

### 2.1 図形の色：RGB は透明度より前

- **原因:** 図形の色（ENC、ODA 2.11）では、フラグ 0x8000 が「次の値は RGB」、
  フラグ 0x2000 が「透明度の BL が続く」を表し、順番もこのとおり。`common_entity_data.spec` は
  透明度を先に読んでいた。
- **症状:** トゥルーカラーと透明度の両方を持つ図形はこの 2 つが入れ替わり、トゥルーカラーを
  まったく持たず色番号だけになった。LINE 9D4 は rgb 0x01000000、透明度
  0xc2803dc2 になった（色番号 183 だけ）。
- **図面:** `sample_AC1032`。いまは ACadSharp の読みと同じく rgb が #803dc2、透明度が ByBlock。
  この見本の 14 図形が変わった。
- **変更:** RGB を先に、そのあと透明度を読む。
- **コミット:** [`1c4dcc35`](https://github.com/monyuonyu/libredwg-fixes/commit/1c4dcc35a975e52d734e37a662c5c4222335bd1c) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 2.2 R2007 以降：カラーブックのハンドルを 2 回読み、画層を失っていた

- **原因:** R2007 以降、フラグ 0x40 の図形の色の DBCOLOR ハンドルは
  `dwg_decode_common_entity_handle_data` が読む（"deferred from common_entity_data"）。ところが
  `common_entity_data.spec` も同じハンドルストリームから読んでいたので、後回しの読み取りが
  次のハンドル、つまり画層を消費していた。
- **症状:** カラーブックの色を持つ図形の画層が NULL（[0, 0] と表示）になった。
- **図面:** `sample_AC1021` … `sample_AC1032` の CIRCLE、ハンドル 99F（画層は 16 のはず）。
- **変更:** R2007 以降を読むときは、先に読む方を飛ばす。
- **コミット:** [`f18f7216`](https://github.com/monyuonyu/libredwg-fixes/commit/f18f721636605498a7afa29410f9dcbd83b27e00) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 2.3 R2000 に書く色：ByBlock、AutoCAD の ACI の代替、ACI の色表

- **原因:** `bit_downconvert_CMC` は R2004 以降の図形の色（ENC）を、方式バイトを持つほかの色と
  同じに扱っていた。ENC には方式が無く、色番号が ByBlock（0）か ByLayer（256）を表す。
  トゥルーカラーは RGB の下位バイトを色番号にされていた。また ACI の色表 `rgb_palette` は
  ほとんどの項目で AutoCAD と違っていた（色 8 が 65,65,65）。
- **症状:** R2000 の出力では ByBlock の図形がすべて ByLayer になり、トゥルーカラーは色と関係の無い
  ACI の番号になった。
- **図面:** 同じ図面を AutoCAD 自身が R2000 で保存したもの。AutoCAD はトゥルーカラーと一緒に
  保存されている ACI の番号を書く（#803dc2 なら 183）。
- **変更:** ByBlock と ByLayer は保つ。トゥルーカラーは一緒に保存されている ACI の番号を保ち
  （AutoCAD の代替）、無ければ最も近い ACI（`dwg_nearest_color_index`）。色表は AutoCAD の
  表示どおり（ezdxf の値。ACadSharp とは 256 中 251 で一致）。R2000 で書いた `example_2018`
  は、AutoCAD 自身の R2000 保存と 222 図形すべてで一致するようになった。
- **コミット:** [`ecad64a8`](https://github.com/monyuonyu/libredwg-fixes/commit/ecad64a88311475a4a1c5234d3c7dc56c948efba) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 2.4 DXF：トゥルーカラー（420〜429）を 24 ビットの RGB で

- **原因:** DXF のグループ 420 はただの RGB（AutoCAD は 3355443 と書く）。`out_dxf` は
  DWG の値を 0xC2 の方式バイト付き（3258135347）で、`%li` で書いていた。
- **症状:** 420 の値が RGB になっていなかった。Windows（32 ビットの `long`）では負の値にさえなり、
  Linux と Windows の `dwg2dxf` が違う DXF を書いた。
- **変更:** 下位 24 ビットを int32 の書式で書く。Linux と Windows は同じ DXF を書くようになった。
- **コミット:** [`8f1650ad`](https://github.com/monyuonyu/libredwg-fixes/commit/8f1650ad30a8c7364b590c339e12dbec91d479ba) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 2.5 DXF：DIMSTYLE の色 176〜178 は色番号

- **原因:** `dxf_CMC` は DXF コードが 90 以上の色をすべて int32 の RGB で書いていた。
  これは 90〜99 だけのためのもの。
- **症状:** R2004 以降の図面の DIMCLRD、DIMCLRE、DIMCLRT が
  `176 -1037268965` と書かれ、読み直すと別の色になった。トゥルーカラーは 534 のグループとして書かれた。
- **変更:** RGB は 90〜99 だけ、420 のトゥルーカラーは 62〜67 だけ。それ以外は AutoCAD と同じく
  色番号。
- **コミット:** [`96b39361`](https://github.com/monyuonyu/libredwg-fixes/commit/96b393619e800e438e465e3687973ed9a5b21d28) — 本家にはまだ送っていない。

### 2.6 R13〜R2000 から R2004 以降へ：AutoCAD が EED に残すトゥルーカラー

- **原因:** AutoCAD はトゥルーカラーの図形を R2000 以前で保存するとき、トゥルーカラーを
  APPID ACAD の EED（"RTTcAl"、70 にフラグと色番号、71 に RGB、色名）に残し、
  図面を読むときに復元する。LibreDWG はこれを無視していた。
- **症状:** R2004 で書いた `sample_AC1015` は、AutoCAD 自身の `sample_AC1018` と色の違う図形が
  21 個あった。
- **変更:** R2004 より前の図面から R2004 以降を書くとき、`upconvert_true_colors` がこれらの色を
  復元する。
- **コミット:** [`ce5e98f8`](https://github.com/monyuonyu/libredwg-fixes/commit/ce5e98f882ec54d26185959a34a031ea1f96f04a) — 本家にはまだ送っていない。

DXF の TABLESTYLE の色は [6.3](#63-tablestyler2010-以降のすべてのセルスタイルr2007-の行スタイル) にある。

## 3. 文字列とコードページ

### 3.1 アジアのコードページ：2 バイト文字の先頭バイト

- **原因:** `bit_utf8_to_TV` は 2 バイト文字の先頭バイトを `wc1 >> 8` ではなく
  `wc1 >> 4` で書いていた。0xB8F4（U+9694 の GBK）が 8F F4 になった。UTF-8 の 3 バイトの分岐では、
  コードページの 1 バイト文字（CP932 の半角カタカナ）に 0x00 の先頭バイトが付いていた。
- **症状:** UTF-8 からこうしたコードページに変換した中国語・日本語・韓国語の文字列
  （たとえば `dwg_add_u8_input`、JSON の読み込み、DXF から R2000 へ）はすべて文字化けし、半角カタカナの
  文字で文字列が切れた。単体試験は CP949 の 시험 について誤ったバイト列（DC C3 7E E8）を期待していた。
  正しいのは BD C3 C7 E8 で、Python と Windows もそう変換する。
- **変更:** `>> 8` にし、1 バイト文字には先頭バイトを付けない。試験は CP949 の本当の
  バイト列を期待する（PR では ANSI_932 の試験も足している）。
- **コミット:** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504)（一部）、[`45e6b50f`](https://github.com/monyuonyu/libredwg-fixes/commit/45e6b50f615aa7346827f71a26de67dc3a8f0eae) — PR
  [#1440](https://github.com/LibreDWG/libredwg/pull/1440)。

### 3.2 R2007 以降の DXF から R2000 へ：UTF-8 の文字列を図面のコードページに変換

- **原因:** R2007 以降の DXF は UTF-8 だが、`dxf_read_pair` と dynapi は TV の文字列をそのまま
  写していたので、R2000 の DWG に生の UTF-8 のバイト列が入った。
- **症状:** R2007 以降の DXF から書いた R2000 の出力で文字化け。
- **図面:** `gh109_1`（コードページ ANSI_936）。
- **変更:** R2000 の出力では図面のコードページに変換し、そこに無い文字は AutoCAD と同じく
  `\U+XXXX` にする。MTEXT の文字の断片も同じ（R2007 以降が対象のときだけ）。
- **コミット:** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504)（一部） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 3.3 DXF から R2007 以降へ：名前が最初の 1 文字で切れていた

- **原因:** R2007 以降のエンコーダは DXF の文字列を UTF-8 として受け取る（`bit_write_T`）が、
  `in_dxf` と dynapi は対象の版が R2007 以降だと、そのほとんどを UTF-16（TU）の文字列にしていた。
  それを UTF-8 としてもう一度書くと、最初の 0 のバイトで切れた。主な修正の後も、`add_HATCH` と `add_MULTILEADER`
  はグラデーションの色名と MULTILEADER の既定の文字列で同じことをしていた。
- **症状:** 名前が最初の 1 文字で切れ（ブロック ALIGNMENT_PARAMETER が
  "A" に、EED の APPID も）、ACadSharp はそれらのブロックの図形を失った。DXF から
  R2010〜R2018 へで 59,883 図形中 11,007 を失い、値の変化は 47,757 個あった。
- **変更:** `dxf_read_file` は DXF を R2004 向けとして読み（文字列は UTF-8 のまま）、
  そのあと R2007 以降の版を設定し直す。HATCH と MULTILEADER は DWG の版で判断する。
- **コミット:** [`f8cb428f`](https://github.com/monyuonyu/libredwg-fixes/commit/f8cb428f7ae7850f9bd9d3810264aea75fb00445)、[`061d4164`](https://github.com/monyuonyu/libredwg-fixes/commit/061d41647cdfdbaf000d2dd4d04c3a9aa6f79002) — 本家にはまだ送っていない。

R2000 に書く UTF-16 の EED の文字列は [4.3](#43-r2007-以降の-eed-の文字列を-r2000-に書く) に、R2004 以降のファイルの
文字列の符号化は [13.10](#1310-r2004-以降のファイルの文字列t16tu文字列ストリーム) にある。

## 4. 拡張図形データ（EED）

### 4.1 EED の画層（1003）：ビッグエンディアンのハンドル、DXF では名前

- **原因:** EED のコード 3（DXF の 1003）は LAYER のハンドルで、図形のハンドル（コード 5）と同じく
  ビッグエンディアンで保存される。これをリトルエンディアンで読んでいた。DXF の 1003 は画層の*名前*だが、`out_dxf` は
  ハンドルの番号を書き、`in_dxf` は 1003 をまったく実装していなかった。
- **症状:** AutoCAD の `sample_AC1015` は LAYER の app_layer 8F5 を `00..00 08 F5` で持つが、
  これが 0xF508000000000000 と読まれていた。
- **変更:** デコーダとエンコーダでビッグエンディアンにした。DXF では画層名を書き、
  読み込みでは名前から画層を引く。
- **コミット:** [`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8)（一部） — ビッグエンディアンは PR
  [#1442](https://github.com/LibreDWG/libredwg/pull/1442)、DXF の名前は PR
  [#1445](https://github.com/LibreDWG/libredwg/pull/1445) に含む。

### 4.2 DXF の EED のバイナリ（1004）を 2 回デコードしていた

- **原因:** `dxf_read_binary` が 1004 の 16 進をすでにバイト列にしているのに、`add_eed` が
  もう一度デコードしていた。
- **症状:** "Invalid hex string member" が出て、ASCII の DXF のすべてのオブジェクトの EED のバイナリデータが
  失われた。
- **変更:** デコードは 1 回だけ。
- **コミット:** [`3d495876`](https://github.com/monyuonyu/libredwg-fixes/commit/3d49587605124359218d042037bd1fa2e5fdd80d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 4.3 R2007 以降の EED の文字列を R2000 に書く

- **原因:** UTF-16 の EED 文字列のための R13〜R2007 の分岐は、コードページを
  `bit_write_RS` で書いていた（ANSI_1252 の 30 が 0x1E00 になった。正しくはビッグエンディアン）。また長さは変換後の
  文字列ではなく UTF-16 の文字数だった（U+00FF を超える文字は `\U+XXXX` になる）。
  バッファの確認は条件が逆で、アンダーフローしていた。
- **症状:** R2000 で書いた R2013 の図面の STYLE を ACadSharp が読めなかった
  （フォント名が EED にある）。
- **変更:** コードページはビッグエンディアン、長さは変換後のもの、バッファの確認を正しく。
- **コミット:** [`438e9817`](https://github.com/monyuonyu/libredwg-fixes/commit/438e98179131cba1be9732e42cec26cd4b8faa54) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

## 5. MTEXT、ATTRIB、ATTDEF

### 5.1 R2018 の MTEXT：枠のビットだけでも背景のフィールドがある

- **原因:** 背景のフィールド（倍率、色、透明度）は、`bg_fill_flag` のビット 0x10
  （文字の枠、R2018 以降）だけが立っているときにもある。以前の判定
  `version <= R_2018 ? 1 : 16` は、枠があって塗りの無い R2018 のファイルでこれを飛ばしていた。
- **症状:** そうした MTEXT では、それより後のフィールドをすべて読み誤った。
- **変更:** ACadSharp と同じく、ビット 0x01、または R2018 以降ならビット 0x10 を見る。
- **コミット:** [`3ab77f4d`](https://github.com/monyuonyu/libredwg-fixes/commit/3ab77f4d123107fd17bddd725865dd1e1dad9039)（一部） — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 5.2 R2018 の複数行の ATTRIB/ATTDEF：埋め込みの MTEXT

- **原因:** `mtext_type` が 2 か 4 のとき、「埋め込みの MTEXT オブジェクトのすべてのフィールドが
  Entmode から書かれる」（ODA 20.4.4）。LibreDWG のこの埋め込み部分の配置は AutoCAD のファイルと
  合っていなかった。AutoCAD の R2018 の見本で実測し、ACadSharp の読み方とも合わせると次のとおり。図形の共通
  部分には所有者のハンドル、リアクタ、xdic、`has_ds_data`、完全な ENC の色（RGB が
  透明度より前）、そして線種、マテリアル、影、印刷スタイルの順のフラグとそれぞれのハンドルがある。
  MTEXT は `extents_height` が `extents_width` より前で、背景のフィールドは 0x10 のときにもあり、
  R2018 の注釈と段組みのフィールドを持つ。ATTRIB/ATTDEF の STYLE のハンドルは
  埋め込みの MTEXT のハンドルより前。MTEXT の後には `annotative_data_size` が来て、0 でなければデータ、
  登録アプリケーション、BS が続く。
- **症状:** MTEXT の画層が STYLE の Standard を指し、ATTRIB の画層も誤っていた。
- **図面:** `sample_AC1032`（ATTDEF 796、ATTRIB 79D）。いまはデータがちょうど文字列ストリームの
  ところで終わり、ハンドルはすべて LAYER 0、LTYPE ByLayer、MATERIAL ByLayer、STYLE Standard を指す。
  ACadSharp の見本 22 個での ACadSharp との値の違い：10 → 0。
- **変更:** 上の配置にし、ハンドル用に構造体のフィールドを新設した。JSON ではフィールドを
  `mtext.<field>` として書く（以前は `ins_pt` のような平らなキーが ATTRIB のものとぶつかっていた）。
  GEOPOSITIONMARKER も同じマクロを使い、以前の末尾のフィールドを保っている。見本が無いので
  確かめていない。
- **コミット:** [`a1c9eba4`](https://github.com/monyuonyu/libredwg-fixes/commit/a1c9eba4034cd71f53533a1e134c43337c5d15aa) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443)。

### 5.3 R2018 の複数行の属性を古い版に書く

- **原因:** R2018 より前には複数行の属性は無い。AutoCAD の R2010 と R2013 の保存では
  属性の文字列は `text_value` にある。R2018 では `text_value` が空で、文字列は埋め込みの MTEXT
  にしかない。古い版に書くと空の `text_value` のままだった。
- **症状:** R2013 以前の出力で、複数行の属性が文字列を失った。
- **図面:** `sample_AC1024`、`sample_AC1027` と `sample_AC1032` の比較。
- **変更:** `text_value`（ATTDEF では `default_value`）が空なら、MTEXT の文字列から
  書式コードを除いたものを使う。再現していないこと：AutoCAD がどのコードを除くか、`\P`
  が何になるか（ここでは空白）、AutoCAD の xdictionary の往復用データ。
- **コミット:** [`41f07038`](https://github.com/monyuonyu/libredwg-fixes/commit/41f0703840f8d81ee349f58e389199f1c73e023f) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443)。

### 5.4 R2010 以降の ATTRIB：`keep_duplicate_records` は無い

- **原因:** AutoCAD の R2010、R2013、R2018 のファイルでは、ATTRIB のデータストリームは
  `lock_position_flag` のすぐ後で終わる。プロンプトの前に繰り返しの版の RC（ここでの
  `keep_duplicate_records`）を持つのは ATTDEF だけ。ODA 20.4.4 と ACadSharp も同じ。
- **症状:** 読み取りで "Invalid ATTRIB.keep_duplicate_records 64/65/72" が出て、書き出しでは余分な 1 バイトが
  付いた。
- **図面:** `sample_AC1024`、`sample_AC1027`、`sample_AC1032`。
- **変更:** R2010 以降は DWG に含めない。DXF と JSON では保つ。
- **コミット:** [`f3432746`](https://github.com/monyuonyu/libredwg-fixes/commit/f343274668fd1e997a47a4f8b83832e22ec26813) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443)。

### 5.5 DXF の読み込み：ATTRIB/ATTDEF の埋め込みの MTEXT、MTEXT の段組みの部分

- **原因:** `in_dxf` は、R2018 の複数行の ATTRIB/ATTDEF の `<101, Embedded Object>` の後にある
  埋め込みの MTEXT も、MTEXT の注釈・段組みの部分も読んでいなかった。後者の 40（矩形の
  幅）が文字の高さを上書きしていた。
- **症状:** `sample_AC1032` で `dxf2dwg` が失敗し（"Invalid DXF code 46"）、36 個の MTEXT の
  高さが 0 になった。
- **変更:** 両方の部分をそれぞれ専用のフィールドに読む。
- **コミット:** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504)（一部） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 5.6 R2000 以降に書く R13/R14 の MTEXT：行間は 0 ではなく 1

- **原因:** R13/R14 の MTEXT には行間のフィールドが無く、R2000 以降を書くとき 0 で書いていた。
- **症状:** どの MTEXT も行が重なってつぶれた（R2000 で書いた `sample_AC1014` と
  `example_r14` で 55 個の MTEXT）。
- **変更:** 既定値、つまり行間のスタイル 1 と係数 1.0 にする。
- **コミット:** [`6b26f48d`](https://github.com/monyuonyu/libredwg-fixes/commit/6b26f48deb27612c4013109522f92cc67e41ce9c) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 5.7 MTEXT の改行は `\P` で

- **原因:** 新しい図面の MTEXT には、素の LF の改行が入ることがある。AutoCAD は R2000 の DWG と DXF では
  これを `\P` で書く。LibreDWG は DXF に `^J` で書いていた。
- **症状:** DXF を読むソフトが `^J` を文字として残した。
- **図面:** `sample_AC1018` … `sample_AC1032` と、AutoCAD の R2000 保存の比較。
- **変更:** R2000 の DWG と DXF では、`dwg_fixup_mtext_linebreaks` が LF を `\P` にする。
- **コミット:** [`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40)（一部） — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 5.8 DXF の書き出し：SEQEND の後の ATTRIB と VERTEX

- **原因:** R13〜R2000 では、`out_dxf` は INSERT の属性と POLYLINE の頂点を
  「次のオブジェクト」として探していた。しかし属性の間に SEQEND があることがある。
- **症状:** DXF で SEQEND より後の属性がすべて失われた。
- **図面:** `sample_AC1015`（ATTRIB 79D、SEQEND 79E、ATTRIB 79F、7A0）。
- **変更:** 図形のつながり（次の図形のハンドル）をたどる。
- **コミット:** [`77fec52b`](https://github.com/monyuonyu/libredwg-fixes/commit/77fec52b4abc92612188613f9cb7a8192bfb6957) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 5.9 属性のタグ：空白と `!` 以外はどの文字でもよい

- **原因:** `dwg_is_valid_tag` は英字、数字、`$`、`_`、`-` しか受け付けなかった。AutoCAD は
  空白と `!` 以外のどの文字も受け付ける。ASCII だけの版は空のタグを受け付けていた。
- **症状:** ATTDEF のタグ `SPACEOBJECTS:NUMBER` がはねられ、DXF 経由で失われた。
- **図面:** `aec_objects`。
- **変更:** AutoCAD の規則にした。空のタグは不正とする。
- **コミット:** [`f9b9f144`](https://github.com/monyuonyu/libredwg-fixes/commit/f9b9f14442df3966abda9e1d856c61745a1679b4) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

## 6. TABLE と TABLESTYLE

### 6.1 TABLE/TABLECONTENT：配置を直し、通常のビルドで読めるように

- **原因:** TABLE と TABLECONTENT はデバッグ用のビルドでしか作られず、その配置は
  多くの箇所で誤っていた。各表のデータが終わるべき位置（R2000/R2004 では
  ハンドルストリーム、R2007 以降では bitsize − 17 − 文字列データの大きさ）と照らして確かめた。
  - TABLE はすべての版で INSERT から派生する。R2010 以降の形も挿入のデータと
    ハンドルで始まり、その後に RC、H、BL、B（R2010）または BL（R2013 以降）、表の内容、表スタイルの
    ハンドル、BS、3BD、分割のデータが続く（ODA 20.4.96.2、ACadSharp の `readTableEntity`）。
  - TABLECONTENT（20.4.97）：列は独自データの集まり、セルスタイルの id、BD の
    幅を持つ。セルはそれぞれセルスタイルの上書きを持つ。行の高さは BD。表のセルスタイルの
    上書きは結合範囲より前。ジオメトリのフラグが 0 でなければセルのジオメトリが 1 つ。
    表スタイルのハンドルは末尾。
  - R2010 より前のセル：文字列は R2007 より前で、しかも 344 のハンドルが 0 のときだけ。ブロックのセルは
    has-attdefs のビットと BS の個数を持つ。セルの配置は RS ではなく BS。0x10 はそれ自身の文字スタイルの
    ハンドル。R2007 ではすべてのセルに BL ＋値がある。表の上書きのビット 0x2000 と
    0x40000 の後に余分なハンドルは無い。
  - 値（20.4.99）：空なのはフラグのビット 0x01 のときだけ（0x03 ではない）。R2007 以降、文字列の値は BL の
    バイト数＋UTF-16LE のバイト列で、TV ではない。
  - 汎用の REPEAT の大きさの確認が R2010 より前のセルをはねていた。
  - `-v3` のトレースで、入れ子の長いフィールド名に `strrplc` が assert していた。
- **症状:** 通常のビルドでは表をまったく読まなかった。デバッグ用のビルドではデータが
  終わりを越えて読まれた。
- **図面:** `sample_AC1014` … `sample_AC1032`、`sample_base`、`example_r14` … `example_2018`、
  `2000/TS1`。
- **変更:** 上の配置にした。TABLE と TABLECONTENT を DEBUGGING から UNSTABLE に移したので、
  通常のビルドで読まれる。ハンドルは期待どおりのオブジェクトを指す。
- **コミット:** [`1d92022d`](https://github.com/monyuonyu/libredwg-fixes/commit/1d92022d6a0c4af2c927794e2689f512cf107894) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.2 TABLECONTENT：R2007 より前の kGeneral の値、個数の確認

- **原因:** kGeneral（512）の値は R14〜R2004 の TABLECONTENT オブジェクトにも現れ（AutoCAD
  2008 の往復用データ）、R2007 以降と同じく BL のバイト数＋バイト列の形をとる。読み取りはそこで止まっていた。
  REPEAT の大きさの確認は `count * sizeof(struct)` を残りのビット数と比べていた。
- **症状:** "kGeneral <r2007" が出て、正しい表の最後の行がはねられた。
- **変更:** kGeneral をすべての版で読む。確認は要素ごとの最小のビット数で行う。
  図面集の表 24 個と TABLECONTENT オブジェクト 9 個はすべて終わるべき位置で終わり、表 24 個はすべて
  ACadSharp とフィールド単位で一致する。
- **コミット:** [`b9780ac7`](https://github.com/monyuonyu/libredwg-fixes/commit/b9780ac7a0b3b1c995fbc37f17a3d798901db081) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.3 TABLESTYLE：R2010 以降のすべてのセルスタイル、R2007 の行スタイル

- **原因:** R2010 以降：表のセルスタイルの後に、セルスタイル _TITLE、_HEADER、
  _DATA が配列として、それぞれ BL の番号の後に続く（ACadSharp の `readTableStyle` と同じ）。最初の 1 つしか読んでいなかった。
  R2007：行スタイルは罫線の後に値のデータ型、単位の型、書式文字列を持つ
  （20.4.101.1）が、読んでいなかった。`format_string` は TU ではなく T。R2000 を書くとき、エンコーダは
  余白を書いた後で表スタイルを変換していたので、余白は 0 になった。DXF の読み込みは TABLESTYLE の色を色表の RGB として、罫線の
  線の太さを図形の線の太さの番号として扱っていた。
- **症状:** `example_2018` の TABLESTYLE 0x29B はビット 310.4 で終わり、6688 ビットの不明なデータが残った。`example_2007` と `sample_AC1021` の
  行 1 と 2 はゴミだった。
- **変更:** 配列全体を読む。`downconvert_TABLESTYLE` を書き直した（データ・タイトル・
  見出しの行スタイルは _DATA、_TITLE、_HEADER から、余白と流れの向きは表のセルスタイルから、
  背景 "none" は背景なしの ACI 7 として、罫線の非表示は反転）。
  DXF の色は表の色として（ACI 7 は 0xc3000007）、線の太さは与えられたとおり、420〜427 は 24 ビットの RGB で読む。
  JSON では `ovr` は配列。`example_2000` … `example_2018` の DXF の TABLESTYLE は
  AutoCAD の DXF と同じになり、R2000 の出力は AutoCAD の R2000 保存と同じになった。
- **コミット:** [`6a3f2d0c`](https://github.com/monyuonyu/libredwg-fixes/commit/6a3f2d0cbab6e9a5748e030d64824362c17abe14) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.4 R2010 以降の表を古い形式で R2000/R2004 に書く

- **原因:** R2010 以降、表はセルを表の内容（20.4.97）に持つ。R2000 と
  R2004 には古い形式のセルのデータ（20.4.96.1）が要る。これを変換するものが無かった。
- **症状:** R2010 以降の図面の表が R2000 の出力で失われた。AutoCAD は R2000 のファイルにも ACAD_TABLE を
  残す（`sample_AC1015`、`example_2000`）。
- **変更:** `downconvert_TABLE` が古い形式のデータを作る：行、列、幅、高さ、
  結合範囲、セルの文字列（書式付きの値 302、無ければ文字列）、ブロックのセル、そして内容ごとの文字の
  高さ・配置・色の上書き。罫線、行・列・表のセルスタイル、
  フィールドは変換しない。`flag_for_table_value` は R2010 以降の同じ値を持つ不明な BS から取る。
  それが同じフィールドだというのは推測である。`sample_AC1024/1027/1032`、
  `sample_base`、`example_2010/2013/2018` の R2000 の出力は、AutoCAD の R2000 のファイルと同じ 11 個の表を持つ。
- **コミット:** [`7139a423`](https://github.com/monyuonyu/libredwg-fixes/commit/7139a423964adff8eef80da0cb1d6daa192e07c0) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.5 R2007 の表を R2000/R2004 に書く：セルの文字列

- **原因:** R2007 はセルの文字列を値（UTF-16）に持ち、`text_value` は空。
  AutoCAD の R2000 保存は書式付きの値の文字列（302）を優先して書く。
- **症状:** R2000 で書くと、すべてのセルが文字列を失い、数値は書式を失った。
- **図面:** `sample_AC1021` と AutoCAD の `sample_AC1015` の比較。
- **変更:** セルの文字列を値から取る。書式付きの文字列を優先する（"0.500000"、
  "$100.00"、"CELL FORMAT 1"）。表は AutoCAD の R2000 保存と同じになった。
- **コミット:** [`4f819653`](https://github.com/monyuonyu/libredwg-fixes/commit/4f8196538d897b971893cd585d38a1d47e2371a7)、[`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb)（一部） — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.6 ACAD_TABLE の DXF の読み込みと書き出し

- **原因:** 表は DXF に書かれず、読み込むこともできなかった。TABLE が
  UNSTABLE になったとき（[`d1f8fb4c`](https://github.com/monyuonyu/libredwg-fixes/commit/d1f8fb4c7e729e577e4d005e699a495f490f01a8)）、`in_dxf` は未知の ACAD_TABLE のグループで失敗し、`out_dxfb` は
  表を DWG のフィールドのまま書いた。そのコミットではまず DXF/DXB の表を無効のままにした。
- **症状:** DXF 経由（`dwg2dxf`、`dxf2dwg`）ですべての表（図面集で 18 個）が失われた。
- **変更:** AutoCAD は ACAD_TABLE をどの版の DXF にも R2010 より前の形で書く（
  INSERT の部分、342 343 11 90 91 92 93〜96、行の高さ 141 が列の幅 142 より前、セルごとに 171〜176、
  177、178、145、その後 R2007 より前は文字列 1、R2007 以降は値、R2010 以降は 280 0 が加わる）。
  `dxf_ACAD_TABLE` がこれを書き（R2010 以降の表は R2000 向けと同じく変換）、`add_ACAD_TABLE` が読む。
  `example_2000/2007/2018` の出力は、ハンドル、無名ブロックの名前、
  R2018 ではフラグ 90 とセルのフラグのビット 0x40000 を除いて AutoCAD の DXF と同じ。セルと表の上書きは DWG の
  フィールドの順に従う。AutoCAD の見本にはどれも無いので、DXF での順番は推測である。その後、書き出しを
  `out_dxf_table.inc` に移し、バイナリの DXF（`dwg2dxf -b`）も表を書くようにした。
- **コミット:** [`d1f8fb4c`](https://github.com/monyuonyu/libredwg-fixes/commit/d1f8fb4c7e729e577e4d005e699a495f490f01a8)、[`56cd3c4f`](https://github.com/monyuonyu/libredwg-fixes/commit/56cd3c4fe139abf172696e8ca31093db1936b3df)、[`0f3a1ac7`](https://github.com/monyuonyu/libredwg-fixes/commit/0f3a1ac7e14820b28ecaf16841ad588b439bc51f) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.7 DXF の TABLESTYLE を AutoCAD の配置で

- **原因:** LibreDWG は R2010 以降のセルスタイルの配置（TABLEFORMAT_BEGIN …）を DXF に書いていた。
  AutoCAD はすべての版で R2007 の配置を書く：3 つの行スタイル（7 140 170 62 63 283
  [90 91 1] と罫線 6 本）、R2010 以降は先頭に 280 0。R13/R14 の DXF は TABLESTYLE を落としていた。
- **症状:** ACadSharp が TABLESTYLE を読めず、表は DXF 経由で表スタイルを失った。
  `sample_AC1014` は表とその TABLESTYLE を持つが、スタイル名 STANDARD を失った。
- **変更:** すべての版で R2007 の配置にする。図面が TABLESTYLE を持つなら R13/R14 の DXF にも
  書く。
- **コミット:** [`61f6d7ec`](https://github.com/monyuonyu/libredwg-fixes/commit/61f6d7ecaad2cda7d54a5857fa2aceaac79c3a33)、[`e1835040`](https://github.com/monyuonyu/libredwg-fixes/commit/e18350406d9dd7784bf7c547bcd9ad9786b164ac) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.8 古い版や DXF から R2010 以降に書く表

- **原因:** R2010 以降の TABLE には表の内容が要る。R2000〜R2007 の表のセルも、
  どの DXF から読んだ表のセルも（`in_dxf` は R2010 より前の形を読む）変換していなかった。DXF から来た R2007 のセルの
  値には `data_size` が無い。
- **症状:** R14〜R2007 の図面の 13 個の表は、R2010 以降で書くと行・列・セルが無く、
  水平方向が 0 だった。R2010 以降の DXF の表を R2010 以降で書くと空だった
  （図面集で 11 個）。ACadSharp は大きさ 0 の 3D 点の値をはねた（"Could not read ACAD_TABLE"、
  DXF 経由の `sample_AC1021` の表）。
- **変更:** `downconvert_TABLE` の逆の `upconvert_TABLE`（列、行、セルごとに 1 つの
  内容、上書き、結合範囲、水平方向、分割の行範囲 1 つ）を作り、DXF の入力にも使う。
  2D/3D の点の値の大きさを設定し、データの無い日付は文字列として書く。
- **コミット:** [`7fb484ed`](https://github.com/monyuonyu/libredwg-fixes/commit/7fb484ed0d13655ed22dce0ab8678c170339a142)、[`03c45bf5`](https://github.com/monyuonyu/libredwg-fixes/commit/03c45bf52eff38e84da496b20f4bc508e034dd95)、[`3ec9d114`](https://github.com/monyuonyu/libredwg-fixes/commit/3ec9d1148f71d8a2a0fd07fbe044f33ef17d39c5) — 本家にはまだ送っていない。

### 6.9 古い版から R2007 以降を書くときのメモリリーク

- **原因:** `upconvert_TABLESTYLE` が作る `ovr` 配列（元の版の解放用 spec はこれを知らない）と、
  `add_DUMMY_eed` の UTF-16 の名前が解放されていなかった。
- **症状:** R2018 で書いた `example_2000` で LeakSanitizer が検出。
- **変更:** 両方を解放する。
- **コミット:** [`10a1a752`](https://github.com/monyuonyu/libredwg-fixes/commit/10a1a752e71debbe348e500c2da9daced8b1dec1) — 本家にはまだ送っていない。

## 7. ACIS のソリッド（3DSOLID、REGION、BODY）

### 7.1 読み取り：R2013 以降は ACIS の空ビットが無い、R2007 以降の不明な BL

- **原因:** R2013 以降、SAB は AcDsPrototype_1b セクションにあり、図形は
  ワイヤフレームのデータから始まる。LibreDWG は先に ACIS の空ビットを読み、すべてを 1 ビット
  ずらしていた。また ODA 20.4.41 は空ビットの後に "R2007+: Unknown BL" を挙げているが、これを読んでいなかった。
- **症状:** AutoCAD の R2018 の 3DSOLID D6A のアイソラインが 1274288069 本になっていた。いまは ACadSharp の
  読みと同じく点 (31.69, −220.11, 2.19) とアイソライン 4 本。ゴミの 3DSOLID は
  R2000 の出力も壊した：ACadSharp はモデル空間でその後の 9 図形を失った。
- **図面:** `sample_AC1032`。
- **変更:** R2013 以降は空ビットを読まない。新しいフィールド `unknown_2007`。149 図面の ACIS の
  図形 107 個はすべて、ちょうど bitsize で終わるようになった（以前は 100 個）。ある試験は
  リビジョン GUID の配列をアドレスで比べていて、これは決して一致しない。以前は GUID を読み誤っていたので
  そこまで到達しなかった。いまは中身を比べる。
- **コミット:** [`faa9c595`](https://github.com/monyuonyu/libredwg-fixes/commit/faa9c59576683f2931117b4cd6edb54b4adc7ca2)、[`aad393ec`](https://github.com/monyuonyu/libredwg-fixes/commit/aad393ec206b1116694db8ae7877874207530709) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 7.2 書き出し：空ビット、履歴のハンドル、マテリアル

- **原因:**
  - エンコーダは R2013 以降でも ACIS の空ビットを書いていた（[`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)）。
  - R2013 以降の 3DSOLID の履歴のハンドル（H 350）を読みも書きもしていなかった（AutoCAD の R2018 の 3DSOLID の
    ハンドルストリームに 26 ビットが残っていた）（[`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)）。
  - それを R2004 と R2000 の DWG にも余分な null ハンドルとして書いていた。AutoCAD の R2004 の図面には
    無い（[`81d14b4c`](https://github.com/monyuonyu/libredwg-fixes/commit/81d14b4c8e628be99b3deabcce0ef3e773c9b580)）。
  - R2007 以降では SAB のデータがあるときにしか書かず、R14〜R2004 の図面の SAT の 3DSOLID には欠けていた
    （[`d7da6276`](https://github.com/monyuonyu/libredwg-fixes/commit/d7da62767fb3af37edb5cce653ac7115e36917df)）。
  - AutoCAD の R2007〜R2010 の 3DSOLID と REGION は空ビットの後に不明な BL しか持たない。
    `num_materials` がその BL を読み、不明な BL がハンドルストリームの 34 ビットを読んでいた。また
    履歴のハンドルは 3DSOLID だけにあり、REGION には無い（[`d59ae195`](https://github.com/monyuonyu/libredwg-fixes/commit/d59ae195c6d89b813ae3a90e1a14b9ceb00dbb76)）。
- **症状:** R2013 以降の図面を R2013 以降で書き直すと、ACadSharp がその 3DSOLID と REGION を読めなかった
  （3DSOLID 5 個と REGION 4 個を失った）。SAT の 3DSOLID を R2010 で書くと ACadSharp が
  ハンドルの先まで読んだ（"Index was outside the bounds of the array"）。`example_2010` の 3DSOLID は 34 ビット
  長すぎた。
- **図面:** `example_2004`、`sample_AC1018`、`example_2010`、`example_2013`、`example_2018`。
- **変更:** AutoCAD と同じにした：R2013 以降は空ビットなし。履歴のハンドルは R2007 以降の
  3DSOLID すべてに、3DSOLID だけに。DWG にマテリアルは書かない。`example_2010`、
  `example_2013`、`example_2018` の 3DSOLID と REGION は、同じ版で書き直すと AutoCAD のバイト列と同じになった。
- **コミット:** [`81d14b4c`](https://github.com/monyuonyu/libredwg-fixes/commit/81d14b4c8e628be99b3deabcce0ef3e773c9b580)、[`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)、[`d7da6276`](https://github.com/monyuonyu/libredwg-fixes/commit/d7da62767fb3af37edb5cce653ac7115e36917df)、[`d59ae195`](https://github.com/monyuonyu/libredwg-fixes/commit/d59ae195c6d89b813ae3a90e1a14b9ceb00dbb76) — 本家にはまだ送っていない。

### 7.3 R2013 以降の AcDs セクション：SAB をハンドルで読む

- **原因:** R2013 以降の REGION、3DSOLID、BODY、SURFACE の SAB は、AcDb:AcDsPrototype_1b セクションの
  データレコードにある。LibreDWG は "ACIS BinaryFile" を探し、`has_ds_data` を持つ図形に
  その順番でデータを渡していた。AutoCAD の R2018 のファイルは代わりに "ASM
  BinaryFile4" を持ち、レコードは図形の順番になっていない。セグメントの番号の
  確認には、エントリの 12 バイトではなく構造体の大きさ 16 を使っていた。
- **症状:** AutoCAD の R2018 のファイル（`example_2018`、`sample_AC1032`）の ACIS データはどれも
  読まれなかった。SAB が別の図形に渡ることがあった（`example_2013`）。`sample_base` の AcDs は
  はねられた。
- **変更:** データ索引のデータレコードをハンドルとともに取る。blob のページからも取る。
- **コミット:** [`eb3b2553`](https://github.com/monyuonyu/libredwg-fixes/commit/eb3b25530662bec56bebb0d045d614aff48be9be) — 本家にはまだ送っていない。

### 7.4 R2013 以降の AcDs セクション：書き出し

- **原因:** AcDs セクションをエンコードできなかった（`acds.spec` が無効になっていた）。
- **症状:** R2013 以降の図面を R2013 以降で書き直すと、ACIS のデータをすべて失った。R2004〜R2010 の
  図面を R2013 以降で書いても同じ。
- **変更:** まず、展開したセクションを保ち、R2013 以降から R2013 以降へはそのまま書く
  （[`60d4a3e0`](https://github.com/monyuonyu/libredwg-fixes/commit/60d4a3e07f870ea31c94f4990f7b60c0ab51c141)）。次に、AutoCAD の `example_2018` と同じ形でセクションを書く：SAB のレコードを
  ハンドルで（ODA の仕様書どおり 0x40000 バイトを超えるものは blob01 のページに）、同じスキーマと検索
  索引を書き、図形に `has_ds_data` の印を付ける（[`b06ca869`](https://github.com/monyuonyu/libredwg-fixes/commit/b06ca8699845d096d972be21469c42f3ca787a68)）。
- **コミット:** [`60d4a3e0`](https://github.com/monyuonyu/libredwg-fixes/commit/60d4a3e07f870ea31c94f4990f7b60c0ab51c141)、[`b06ca869`](https://github.com/monyuonyu/libredwg-fixes/commit/b06ca8699845d096d972be21469c42f3ca787a68) — 本家にはまだ送っていない。

### 7.5 SAT から SAB へ、その逆

- **原因:** R13〜R2000 の DWG と DXF は ACIS のデータを SAT の文字列で持つが、R2013 以降の AcDs
  は SAB のレコードしか持たない。また DXF 用の古い SAB から SAT への変換（`dwg_convert_SAB_to_SAT1`）は、
  double を `%g`（6 桁）で書き、long をポインタ `$n` として書き、論理値を誤り（forward_v と
  I の代わりに F）、ヘッダの文字列をばらばらにし（"A utodesk A utoCA D"）、終端の印を切り、
  版を変え、履歴の属性を書かず、"ASM BinaryFile4" には何もしなかった。
- **症状:** これらの図面のソリッドを R2013 以降で書くと空になった。DXF の SAT は
  読めず、形状が変わった。
- **変更:** ACIS のレコードの文法を作り、SAT に欠けている値の型（long と整数値の double は
  見た目が同じ、論理値は単語）を与える。文法は同じソリッドの AutoCAD の SAT と SAB から
  取った。`dwg_convert_SAT_to_SAB` と `dwg_convert_SAB_to_SAT` がこれを使い、double は最短で正確な表記で書く。
  図面集の SAT のボディ 15 個はすべて SAB にして戻すと同じ SAT に、SAB のボディ 83 個はすべて
  SAT にして戻すと同じバイト列になる。
- **コミット:** [`d207b237`](https://github.com/monyuonyu/libredwg-fixes/commit/d207b2379e50b0163ba401a5639e2fc58c1a0871)、[`2ab21c75`](https://github.com/monyuonyu/libredwg-fixes/commit/2ab21c7563a9f8d75596f55ab4ea1906f0a9050e) — 本家にはまだ送っていない。

### 7.6 DXF：SAT の行とエスケープされた `^`

- **原因:** SAT は DXF にグループ 1 の行で書き、255 文字を超える行は
  グループ 3 に続ける。LibreDWG は DWG の暗号化された 4096 バイトのブロックを 1 行ずつ書いていたが、
  ブロックは行の途中でも分かれる。読むときは、1 の行の改行がその 3 の続きの前に置かれていた。暗号化された `A` は `^` で、
  DXF ではこれを `^ ` と書く。読み込みでは `acis_data` ではエスケープを戻していたが、R13〜R2010 の DWG が
  受け取る `encr_sat_data` では戻していなかった。
- **症状:** ブロックの終わりにある行は 2 行になり、次の文字が
  落ちたので、数値が壊れた（`example_2000` で −0.8944579735636958 が −0.89 と 44579735636958 に
  なった）。DXF から作った DWG の SAT は A ごとに "A " となり、終わりが欠けた。
- **変更:** `dxf_sat_lines` がすべてのブロックの行を書き、255 を超える分は 3 に、`^` は `^ `
  にする。`add_3DSOLID_encr` は改行を正しい位置に置き、両方のコピーのエスケープを戻す。
- **コミット:** [`94a6cf90`](https://github.com/monyuonyu/libredwg-fixes/commit/94a6cf906aa7bd5749254bae8c849216c4627819)、[`bff66044`](https://github.com/monyuonyu/libredwg-fixes/commit/bff660446e6496299f05b67bd6fd353645799e4e) — 本家にはまだ送っていない。

### 7.7 DXF の読み込み：リビジョン GUID のフィールド名

- **原因:** リビジョン GUID を `revision_guid[39]` と `revision_bytes[9]` という、
  dynapi の知らない名前で設定していた。
- **症状:** "Invalid 3DSOLID field"。
- **変更:** 本当のフィールド名にした。
- **コミット:** [`da463d6d`](https://github.com/monyuonyu/libredwg-fixes/commit/da463d6d34deddbfcb3c107a1582bed3d65a4384)（一部） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

## 8. プロキシと不明なオブジェクト

### 8.1 不明なオブジェクトはクラス名を保つ

- **原因:** LibreDWG がデコードできない図形とオブジェクトは、すべて UNKNOWN_ENT /
  UNKNOWN_OBJ という名前になっていた。
- **症状:** JSON には UNKNOWN_ENT とだけ出て、何のオブジェクトか分からなかった。
- **変更:** ほかの読み取り実装と同じく、クラス名を dxfname にする（AEC_WALL、TCH_WALL など）。
- **コミット:** [`1935be09`](https://github.com/monyuonyu/libredwg-fixes/commit/1935be09e0ad6e1cac637427804eb02741afeada)（一部） — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.2 必要なときは ACDBPLACEHOLDER のクラスを

- **原因:** 対応していないオブジェクトはプレースホルダとして書くが、R2007 以降の図面の DXF には
  ACDBPLACEHOLDER のクラスが無い。
- **症状:** 対応していないオブジェクト（TABLEGEOMETRY、R2000 への MATERIAL）はすべて DUMMY になり、
  ERROR が出た。
- **変更:** 必要なときはエンコーダがクラスを足す。
- **コミット:** [`ebeebb13`](https://github.com/monyuonyu/libredwg-fixes/commit/ebeebb139ac7e446df77e17486c59c6baa3f693d) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.3 不明なオブジェクトを ACAD_PROXY_ENTITY / ACAD_PROXY_OBJECT として書く

- **原因:** R2004 以降の DWG から R2000 を書くとき、不明な図形とオブジェクト（AEC_*、TCH_*
  など）は厚さ 1e25 の POINT か PLACEHOLDER になっていた。またプロキシの spec は図面形式の後に
  グラフィックスの 2 つ目のコピーを読み書きし（ODA と ACadSharp には無い）、すべての
  オブジェクト id をデコードせず、`bit_write_bits` は末尾のビットを逆の順で書いていた。
- **症状:** ほかのアプリの独自のオブジェクトがデータを失った。
- **変更:** AutoCAD と同じくプロキシとして書く（ODA 20.4.90/29）：クラス番号、
  元の図面形式、プロキシのグラフィックス、ハンドルストリームまでのデータのビット、そして
  オブジェクト自身のハンドル。元が R2007 以降ならデータは文字列ストリームを保つ。共通のハンドルは
  デコードしたものと照合し、合わなければ以前の代替のまま。`gh109_1`（R2013、
  TCH_*）では、26 個のプロキシが CAD の書いた `gh109_1.dxf` とクラス、図面形式（0x6A001F）、
  データと文字列のビット、オブジェクト id で一致する。
- **コミット:** [`d9004510`](https://github.com/monyuonyu/libredwg-fixes/commit/d9004510df29de8f15efbb4c7017d15efb59ef0c) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.4 DXF：不明なオブジェクトとプロキシ

- **原因:** `dwg2dxf` は不明な図形とオブジェクトを飛ばしていた。図面形式は
  `version | maint << 16` ではなく `>> 8` で計算していた。プロキシの相対的なオブジェクト id（6、8、10、12）を
  絶対値にしていなかった。
- **症状:** DXF 経由（`dwg2dxf`、`dxf2dwg`）でこれらが失われた。
- **変更:** これらと PROXY_* オブジェクトを AutoCAD の配置で書く（90 498/499、91 クラス
  番号、95 図面形式、70、AcDbProxyEntity にグラフィックス 92/160 ＋ 310、データは
  R2007 以降 161/162 ＋ 310/311、それ以外は 93 ＋ 310、オブジェクト id 330/340/350/360、94 0）。`in_dxf` はそれを読み戻す
  （`add_PROXY`）。`gh109_1` では、DXF は 26 個のプロキシすべてで CAD の書いた DXF と一致する。ただし 8 個の
  グラフィックスは除く（こちらのは DWG のもの）。`gh109_1.dxf` を `dxf2dwg` に通すと、DWG 経由と同じ
  プロキシになる。
- **コミット:** [`5885ae72`](https://github.com/monyuonyu/libredwg-fixes/commit/5885ae72ccb15e2b0768f922ce3d1c705be64cfd) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.5 R2004 と R2010 以降が対象のときもプロキシに

- **原因:** プロキシへの変換は R2000 が対象のときしか動かなかった。R2007 以降の元から R2010 以降を
  対象にすると、プロキシのデータがプロキシ自身のものではなく、元のオブジェクトの文字列ストリームで
  終わっていた。
- **症状:** `aec_objects`（R2018）の AEC_WALL は、R2004 では POINT に、R2010 と R2013 では POINT と
  プレースホルダになった。
- **変更:** 元より古いどの版が対象でもプロキシにする。`encode_proxy_data` がデータの
  文字列をプロキシの文字列ストリームに入れる。
- **コミット:** [`a970cfee`](https://github.com/monyuonyu/libredwg-fixes/commit/a970cfeecbc350f21b88f60c22205dbb69db445e)、[`5243d4f5`](https://github.com/monyuonyu/libredwg-fixes/commit/5243d4f5610125fdc3bb2d741ba70d21e12cc32b) — 本家にはまだ送っていない。

AEC の図面で見つけたクラスの数の上限は [1.2](#12-クラスの数の上限が低すぎた) にある。

## 9. その他の図形とオブジェクト

### 9.1 DIMENSION_ANG2LN：点 10 と 16 が入れ替わっていた

- **原因:** R13 以降は 16 の点（2RD）、次に 13、14、15、そして 10 の点（3BD）を保存する（ODA
  20.4.27）。2RD を `def_pt` に、最後の 3BD を `xline2end_pt` に読んでいた。
- **症状:** `dwg2dxf` は円弧の点をグループ 10 に、2 本目の補助線の端を
  グループ 16 に書き、DXF を読むソフトは寸法の円弧を違う位置に描いた。
- **図面:** `sample_AC1032`、ハンドル 516。
- **変更:** 同じビットを同じ DXF コードのフィールドに読み、13〜16 を DXF の順に書く。
  JSON では `xline2end_pt` は 3D のまま。`in_json` が 3D の点を期待するため（そうでないと "JSON 3DPOINT
  must be ARRAY of size 3" が出て、残りの読み込みが失敗した）。
- **コミット:** [`3d450c21`](https://github.com/monyuonyu/libredwg-fixes/commit/3d450c21651ed381fe5ba101aaa7c8e353dca8f7)、[`fee66f28`](https://github.com/monyuonyu/libredwg-fixes/commit/fee66f28e31bb1ca27813672fb665a0278a45a58) — PR には含めていない。本家の `master` がその後、同じ
  入れ替わりを自分のコミット
  [9bcd0586](https://github.com/LibreDWG/libredwg/commit/9bcd058671439d3d86342012d882805a45d8916e) で直したため。

### 9.2 R2013 の RENDERSETTINGS

- **原因:** R2013 の特例は `class_version` を飛ばし、`has_predefined` を
  `display_index` の後に読んでいた。AutoCAD の R2013 のプリセットは `class_version` 2 で、その後は R2018 の配置、
  `has_predefined` は末尾にある。
- **症状:** R2013 の RAPIDRTRENDERSETTINGS はすべて 10 ビットずれた（レンダーのレベルと
  フィルタの大きさがゴミ）。`dwgread -O json` は "-nan" まで出力した（[14.1](#141-jsonnan-を出さない) を参照）。
- **図面:** `gh44-error`、`gh109_1`。
- **変更:** R2013 でも R2018 の配置にする。
- **コミット:** [`3cfd5030`](https://github.com/monyuonyu/libredwg-fixes/commit/3cfd5030d85ff792550b62ffb1b16eba07ce6c77) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 9.3 R13/R14 の LAYER：非表示、印刷、線の太さ

- **原因:** R13/R14 のデコーダは非表示のビットを「色 < 0」で上書きしていたが、AutoCAD の R14 の
  図面は色を正のまま保ち、ビットを立てる。また R13/R14 には印刷のフラグと線の太さが無く、
  R2000 以降を書くとき 0 で書いていた。
- **症状:** 非表示の画層が表示として読まれた。R2000 の出力ではどの画層も印刷しない設定になり、
  線の太さは 0.00 mm だった。
- **変更:** 非表示はビットか負の色。R2000 以降では AutoCAD の既定値、印刷あり、
  線の太さ DEFAULT（31）。
- **コミット:** [`4b815379`](https://github.com/monyuonyu/libredwg-fixes/commit/4b815379f222c25641ed9124c511db55fc7c1ea0) — 非表示は PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)、印刷
  と線の太さは PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444) に含む。

### 9.4 R13/R14 の図形：独自の線種、線の太さ

- **原因:** デコーダは `ltype_flags` を ByLayer のときにしか設定せず、独自の線種を持つ図形に
  3 を設定していなかった。R13/R14 には図形の線の太さが無く、R2000 の出力は 0 と書いていた。
- **症状:** 独自の線種（CONTINUOUS）を持つ図形が R2000 の出力で ByLayer になり、
  線の太さは 0.00 mm だった。
- **図面:** `example_r14`、`sample_AC1014` と、AutoCAD の R2000 保存の比較。線種の
  違いは無くなった。
- **変更:** 独自の線種には `ltype_flags` 3。線の太さは ByLayer（29）。
- **コミット:** [`597c0da6`](https://github.com/monyuonyu/libredwg-fixes/commit/597c0da69fbf26c80db4ac4634e4ef884c002e0c) — 線種は PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)、
  線の太さは PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444) に含む。

### 9.5 LEADER：R2010 以降の配置

- **原因:** ACadSharp の読みと同じく、注釈のオフセット（`endptproj`）は R2010 以降にもあり、枠の高さと幅は
  R2007 までしかない。LibreDWG は 3BD のところを 2 つの BD として読んでいた。
- **症状:** データが 6 ビット行き過ぎた。R2013 と R2018 で書いた `2000/TS1` の LEADER は
  失われた。
- **図面:** `sample_AC1032`（その LEADER はいまちょうどで終わる）。
- **変更:** `endptproj` は R13c3 以降、`box_height`/`box_width` は R2007 まで。
- **コミット:** [`9e5e27b3`](https://github.com/monyuonyu/libredwg-fixes/commit/9e5e27b33d9c622b1e6865ebfc7fc49515ea26b7) — 本家にはまだ送っていない。

### 9.6 BLOCKSCALEACTION：接続点

- **原因:** AcDbBlockScaleAction は、AcDbBlockActionWithBasePt の 2 つの後に 3 つの接続点を持つ
  （`conn_pts[5]`、`in_dxf` は 2〜4 を使う）が、0〜2 に読んでいた。
- **症状:** UpdatedBaseX と UpdatedBaseY が Scale と XScale で上書きされた。
- **変更:** 接続点 2〜4 に読む。
- **コミット:** [`a8f2deb1`](https://github.com/monyuonyu/libredwg-fixes/commit/a8f2deb148e2550913e32de9fd37c7412e1b1052) — 本家にはまだ送っていない。

## 10. DXF の読み込み

### 10.1 LWPOLYLINE の幅とフラグ、非表示の画層

- **原因:** `in_dxf` は最初の頂点が 40/41 を持つときしか幅の領域を確保せず、
  一定幅のフラグに 32 ではなく 4 を立て、フラグを 70 で上書きし、閉じたポリラインの plinegen を
  落としていた。`out_dxf` はふくらみがあるときしか幅を書かなかった。非表示の画層の DXF の負の色は
  DWG に色番号 65529 として書かれた。
- **症状:** `sample_AC1015` の `dwg2dxf` の出力（幅は最後の頂点だけ）を
  読み戻せなかった（"Invalid DXF code 41 for LWPOLYLINE"）。ACadSharp は画層の色をはねた。
- **変更:** 幅はどの頂点からでも読み、フラグは 32、DWG のフラグは保つ。非表示の画層は AutoCAD が
  書くのと同じく、非表示のビットと正の色を持つ。
- **コミット:** [`57cda29e`](https://github.com/monyuonyu/libredwg-fixes/commit/57cda29e049ce9995721b6e90e1e614cc11caa6b) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.2 SPATIAL_FILTER

- **原因:** SPATIAL_FILTER は double をすべてグループ 40 で持つ（前方クリップの z と、
  12 個の double の行列 2 つ）。汎用の読み込みではこれらを区別できなかった。
- **症状:** クリップの頂点が崩れ、逆行列は NaN で埋まり、
  DXF から DWG にすると ACadSharp が読めなかった。
- **変更:** 専用の読み込みを作った。
- **コミット:** [`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8)（一部） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.3 HATCH：(0,0) の種点

- **原因:** 境界のパスの後、0.0 の 10/20 を種点ではなく
  高さとして扱っていた。
- **症状:** 原点の種点が落ち、`num_seeds` が 1 から 0 になった
  （DXF 経由の `sample_AC10xx`）。
- **変更:** パスの後のものは種点とする。
- **コミット:** [`da463d6d`](https://github.com/monyuonyu/libredwg-fixes/commit/da463d6d34deddbfcb3c107a1582bed3d65a4384)（一部） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.4 `*Model_Space` は DXF のハンドルを保つ、VX_CONTROL

- **原因:** object[0]（`*Model_Space`）は、TABLES を読む前にハンドル 0x1F で作られていた。
  エンコーダは追加した VX_CONTROL に、空いているか確かめずにハンドル B を与えていた。
- **症状:** `gh209_1`（`*Model_Space` が 2、LAYER の表が 1F）は DXF 経由でハンドルが重複し、
  モデル空間が 2 つになり、14 図形が違う画層に載った。その DICTIONARY は B である。
- **変更:** ASCII の DXF をざっと走査して `*Model_Space` の
  BLOCK_RECORD のハンドルを取る。VX_CONTROL は B が空いているときだけ B にする。
- **コミット:** [`f7880bf9`](https://github.com/monyuonyu/libredwg-fixes/commit/f7880bf993cc0926d66de6041ae95baf45661a4d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.5 表の名前：大文字小文字を無視する一致より完全一致を先に

- **原因:** `dwg_find_tablehandle` は大文字小文字を無視して名前を照合し、最初のものを取っていた。
- **症状:** ほかのソフトの図面は STANDARD と Standard の両方を持つことがある。`gh109_1` の
  STANDARD を参照する DIMENSION が、DXF の読み込み後に Standard を参照した。
- **変更:** 完全一致を先に探す。
- **コミット:** [`eedf8e37`](https://github.com/monyuonyu/libredwg-fixes/commit/eedf8e37d597c5324b246a6dc9edc1084a87947a) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.6 DXF に無いヘッダ変数

- **原因:** DXF はどのヘッダ変数を省いてもよい。省かれたものは、0 が不正なものでも 0 のままだった。
- **症状:** FreeCAD の DXF には $TEXTSIZE と $DIMTXT が無いので、FreeCAD の DWG の書き出し
  （`dxf2dwg`）は文字の高さが 0 になり、ACadSharp がはねた。
- **変更:** TEXTSIZE、DIMTXT（MEASUREMENT 1 なら acad または acadiso の値）、
  LTSCALE、CELTSCALE、DIMLFAC、DIMTFAC、DIMALTF に AutoCAD の既定値を与える。
- **コミット:** [`98acd4a3`](https://github.com/monyuonyu/libredwg-fixes/commit/98acd4a314d8d2cf7c9525c66ea4b900a140b54a) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.7 SHAPE：名前はシェイプの番号ではない

- **原因:** dynapi では `style_id` が DXF の 2 になっており、汎用の読み込みは文字列の
  ポインタを番号として入れていた。
- **症状:** 実行ごと・プラットフォームごとにでたらめな番号になり、Linux と Windows の `dxf2dwg` が
  違う DWG を書いた。
- **変更:** 0 のままにする。番号は SHX ファイルを読まないと分からない。
- **コミット:** [`0f204cfc`](https://github.com/monyuonyu/libredwg-fixes/commit/0f204cfcfca9dd153bfa59dc9dc998453e00507d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.8 所有する頂点と属性は絶対ハンドルで

- **原因:** `dxf_postprocess_SEQEND` は POLYLINE_PFACE/MESH の `vertex[]` と
  INSERT/MINSERT の `attribs[]` を相対のソフトポインタにしており、R2004 以降には相対コード 6 と 10 として書かれた。
- **症状:** ACadSharp はそれらを別のハンドルとして読み、R2004 以降で書いた DXF の頂点と属性を
  失った（図面集で 160 図形）。
- **変更:** AutoCAD と同じく絶対ハンドルにする。
- **コミット:** [`1b1a0c62`](https://github.com/monyuonyu/libredwg-fixes/commit/1b1a0c629952aa23c6a9fbdb4e563c7e91836844) — 本家にはまだ送っていない。

### 10.9 `dwgwrite` は対象の版として DXF を読む

- **原因:** `dwgwrite` は `header.version` が不明なまま DXF を読んでいたので、`in_dxf` は
  `dxf2dwg` のときと違う組み立て方をした。
- **症状:** DXF にした AutoCAD の `example_2018` で、222 図形中 91 しか残らなかった（R2004
  でも R2018 でも）。
- **変更:** `dxf2dwg` と同じく、読む前に書き出す版（または R2000）を設定する。
- **コミット:** [`8ba7693c`](https://github.com/monyuonyu/libredwg-fixes/commit/8ba7693c14a60fc55d1eee1a70fb551ee821f230) — 本家にはまだ送っていない。

DXF の読み込みについてはほかに：EED 1003/1004（[4.1](#41-eed-の画層1003ビッグエンディアンのハンドルdxf-では名前)、
[4.2](#42-dxf-の-eed-のバイナリ1004を-2-回デコードしていた)）、UTF-8 の文字列
（[3.2](#32-r2007-以降の-dxf-から-r2000-へutf-8-の文字列を図面のコードページに変換)、
[3.3](#33-dxf-から-r2007-以降へ名前が最初の-1-文字で切れていた)）、埋め込みの MTEXT
（[5.5](#55-dxf-の読み込みattribattdef-の埋め込みの-mtextmtext-の段組みの部分)）、属性のタグ
（[5.9](#59-属性のタグ空白と--以外はどの文字でもよい)）、表（[6.6](#66-acad_table-の-dxf-の読み込みと書き出し)）、
SAT（[7.6](#76-dxfsat-の行とエスケープされた-)）、プロキシ（[8.4](#84-dxf不明なオブジェクトとプロキシ)）、
DIMSTYLE と SPLINE の既定値（[13.16](#1316-dimstyler2007-以降の変数に-autocad-の既定値を)、
[13.17](#1317-spliner2013-以降のフラグノットのパラメータシナリオ)）。

## 11. DXF の書き出し

### 11.1 HATCH/MPOLYGON：パターン名は DWG のとおりに

- **原因:** ソリッドの塗りはいつも "SOLID" と書かれ、名前に付いた島の検出スタイルの接尾辞が
  落ちていた。
- **症状:** "SOLID,_O" が DXF 経由で "SOLID" になった。
- **変更:** 名前を DWG に保存されているとおり島の検出スタイルの接尾辞付きで書く（"SOLID,_O"）。
  名前の無いソリッドの塗りだけを SOLID と書く。
- **コミット:** [`1935be09`](https://github.com/monyuonyu/libredwg-fixes/commit/1935be09e0ad6e1cac637427804eb02741afeada)（一部） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 11.2 R13/R14 の図面は新しいクラスの図形を保つ

- **原因:** 新しい AutoCAD が保存した R14 の DWG は、MULTILEADER、WIPEOUT、MESH、
  PDFUNDERLAY、ARC_DIMENSION のクラスを持つことがある。`out_dxf` は R13/R14 の DXF ではこれらを飛ばしていた。
- **症状:** DXF でこれらがすべて落ちた。
- **変更:** 飛ばすのは、新しい図面を R13/R14 の DXF に変換するときだけにした。
- **コミット:** [`649637f8`](https://github.com/monyuonyu/libredwg-fixes/commit/649637f8a4a98102115f7747787dd7883178650c) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 11.3 Standard/ByLayer/ByBlock の綴りの変更は R13/R14 からの変換のときだけ

- **原因:** `out_dxf` は、すべての図面で STANDARD → Standard のように名前の綴りを変えていた。
- **症状:** `gh109_1`（R2013）は DIMSTYLE STANDARD を持ち、その DIMENSION が DXF では
  Standard を参照した。
- **変更:** 綴りを変えるのは R13/R14 から変換するときだけ。
- **コミット:** [`fc66341d`](https://github.com/monyuonyu/libredwg-fixes/commit/fc66341dacc1b48750fbf947daa9e4c0301470a6) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 11.4 R2004 以降の図面の `dwg2dxf --as r2000` で図形が出なかった

- **原因:** `get_first_owned_entity` / `get_next_owned_block_entity` は、
  `dwg_fixup_BLOCKS_entities` が足した参照を使っていたが、それは未解決だった。
- **症状:** R2004 以降の図面の `dwg2dxf --as r2000` で図形が一つも書かれなかった。
- **変更:** ブロックの図形をハンドルで探す。
- **コミット:** [`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb)（一部） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

DXF の書き出しについてはほかに：トゥルーカラー（[2.4](#24-dxfトゥルーカラー420429を-24-ビットの-rgb-で)）、DIMSTYLE の色
（[2.5](#25-dxfdimstyle-の色-176178-は色番号)）、EED 1003
（[4.1](#41-eed-の画層1003ビッグエンディアンのハンドルdxf-では名前)）、MTEXT の `\P`
（[5.7](#57-mtext-の改行は-p-で)）、SEQEND の後の ATTRIB/VERTEX
（[5.8](#58-dxf-の書き出しseqend-の後の-attrib-と-vertex)）、表と表スタイル
（[6.6](#66-acad_table-の-dxf-の読み込みと書き出し)、[6.7](#67-dxf-の-tablestyle-を-autocad-の配置で)）、
ACIS（[7.5](#75-sat-から-sab-へその逆)、[7.6](#76-dxfsat-の行とエスケープされた-)）、プロキシ
（[8.4](#84-dxf不明なオブジェクトとプロキシ)）。

## 12. 新しい版から R2000 への書き出し

これらの修正で、R2000 で書いた `example_2004` … `example_2018` は、直接でも DXF 経由でも、
AutoCAD 自身の R2000 保存と 222 図形すべてで一致する。両方の図面群を合わせると 3,405 図形中 3,258 が
一致する（[`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40)）。残りは、AutoCAD が振り直したハンドルと無名ブロック、R14 が保存できない
線の太さ、そしてしていない 3 つの変換（積み重ね分数のコード、ダイナミックブロックの参照、
フィット点のスプラインを制御点に）である。

### 12.1 POLYLINE の VERTEX と INSERT の ATTRIB をつなぐ

- **原因:** R2004 以降は、別の図形が所有する図形をハンドルのリスト（`vertex[]`、
  `attribs[]`）で持つ。R13〜R2000 は連結リスト（所有者に first/last、所有される各図形に prev/next）で持つ。
  `dwg_fixup_BLOCKS_entities` はそれをブロックヘッダの図形にしか作っていなかった。
- **症状:** R2000 で書いた R2018 の図面は、first/last の頂点と属性が 0 で、頂点が
  つながっていなかった。AutoCAD の R2018 の見本を R2000 で書き直すと、ACadSharp は VERTEX 10 個と
  ATTRIB 4 個のどれも見つけられなかった（"Leading entity with handle 0 not found"）。
- **変更:** AutoCAD の書き方（`sample_AC1015`）どおり、リストからつながりを作る。
  first/last は絶対のコード 4 のソフトポインタ（相対のコード 6 も規則上は正しいが、ACadSharp はそれを
  ハンドル 0 に対して解決して落ちる）、prev/next は相対。
- **コミット:** [`89191ebe`](https://github.com/monyuonyu/libredwg-fixes/commit/89191ebe07c103dfdf91f56fe74556d573486389) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.2 MLINESTYLE：線種の番号

- **原因:** R2018 以降、MLINESTYLE の線は線種をハンドルで参照し、それより前は番号で参照する（共用体）。
  R2000 を書くとき、ハンドルのポインタを番号として書いていた。
- **症状:** ACadSharp が "Linetype not assigned, index 29712" で失敗した。
- **変更:** ハンドルを変換する：ByLayer は 32767、ByBlock は 32766、それ以外は
  LTYPE_CONTROL のエントリ中の位置。同じ図面の AutoCAD の R2000 保存と R2018 保存もこれと合う（ByLayer のハンドル
  0x15 が番号 32767）。
- **コミット:** [`a1670d7a`](https://github.com/monyuonyu/libredwg-fixes/commit/a1670d7a3b25ed0cc22a4cb3a8939cdac43cb671) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.3 LWPOLYLINE：R2010 より前は頂点 ID のフラグなし

- **原因:** R2000 で書いた R2010 以降の LWPOLYLINE が、フラグ 1024（頂点 ID）を保っていた。
- **症状:** ACadSharp が存在しない `num_vertexids` を読み、点を失った。
- **図面:** `2018/Dynblocks`。
- **変更:** R2010 より前ではフラグを消す。
- **コミット:** [`c2e85bb6`](https://github.com/monyuonyu/libredwg-fixes/commit/c2e85bb62181e2d04578d3f3387756ced6f44590) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.4 AuxHeader：R2000 でも 123 バイト

- **原因:** ODA の 27 章は、末尾の RL `zero_7` と `zero_8` をすべての版について挙げているが、
  R2004 以降でしか書いていなかった。
- **症状:** `dwgrewrite` の R2000 のファイルは AuxHeader が 115 バイトで（AutoCAD のものは 123
  バイト、`sample_AC1015`）、ACadSharp が EndOfStreamException を投げた。
- **変更:** すべての版で書く。
- **コミット:** [`3ab77f4d`](https://github.com/monyuonyu/libredwg-fixes/commit/3ab77f4d123107fd17bddd725865dd1e1dad9039)（一部） — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 12.5 R13/R14 の名前を AutoCAD 2000 の書き方で

- **原因:** R13/R14 は名前を大文字で綴る。同じ図面の AutoCAD の R2000 保存は
  ByBlock、ByLayer、Continuous、Standard、辞書のキー Standard を使う。
- **図面:** `example_r14` と `example_2000` の比較。
- **変更:** R13/R14 から R2000 以降を書くときはこれらの名前にする。MLINESTYLE は STANDARD のまま。
- **コミット:** [`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb)（一部） — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.6 グラデーションのハッチング

- **原因:** R2000 にはグラデーションの塗りが無い。
- **症状:** グラデーションの HATCH は、AutoCAD の R2000 保存と違って ByLayer のままだった。
- **図面:** `sample_AC1015`、`example_2000`（AutoCAD の R2000 保存）。
- **変更:** AutoCAD と同じく、グラデーションの HATCH を最初のグラデーションの色のソリッドの塗りにする。
- **コミット:** [`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40)（一部） — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

R2000 の出力についてはほかに：色（[2.3](#23-r2000-に書く色byblockautocad-の-aci-の代替aci-の色表)）、
EED の文字列（[4.3](#43-r2007-以降の-eed-の文字列を-r2000-に書く)）、R2018 の複数行の属性
（[5.3](#53-r2018-の複数行の属性を古い版に書く)）、MTEXT
（[5.6](#56-r2000-以降に書く-r13r14-の-mtext行間は-0-ではなく-1)、[5.7](#57-mtext-の改行は-p-で)）、
表（[6.4](#64-r2010-以降の表を古い形式で-r2000r2004-に書く)、
[6.5](#65-r2007-の表を-r2000r2004-に書くセルの文字列)）、プロキシ
（[8.3](#83-不明なオブジェクトを-acad_proxy_entity--acad_proxy_object-として書く)）、R13/R14 の画層と
図形（[9.3](#93-r13r14-の-layer非表示印刷線の太さ)、[9.4](#94-r13r14-の図形独自の線種線の太さ)）。

## 13. R2004〜R2018 への書き出し

本家の LibreDWG は R2000 以前しか書けない。これらのコミットで、LibreDWG は R2004、R2010、R2013、R2018 を書けるようになった。
149 図面すべてが、各版で、直接でも DXF 経由でも、ACadSharp を通した往復で
何も失わない。どれも本家にはまだ送っていない。

### 13.1 R2004 のファイルの器、LZ77 の圧縮付き

- **原因:** R2004 の器（R2010 以降の枠組みでもある）は、どの読み取り実装も受け付けない形で書かれていた：
  セクションマップとページマップがデータのセクションとして並び（ACadSharp は空のセクション名の
  重複で失敗した）、ページの大きさが誤り、Preview は 1.3 MB の 1 ページで、
  システムページは整列されず、ファイルヘッダの末尾は上書きされていた。
- **変更:** AutoCAD 自身の R2004 のファイルと同じ形で `encode_r2004_container` を書いた：空の
  セクションを先頭に、id はセクションマップの順に減っていく。圧縮するセクションは 0x7400 のページに、それ以外は
  大きさを 0x80 に切り上げた 1 ページに（Preview は 0x400）。ページヘッダには大きさ、64 ビットの
  開始オフセット、チェックサム。ページは 0x20 に整列し、決まったバイト列で埋める。セクションマップ
  とページマップはデータのページの後。ファイルヘッダは 0x80 に置き、末尾にも繰り返す。AppInfo と
  AppInfoHistory のアドレス。新しい LZ77 の圧縮器 `r2004_compress`（ODA 4.7、ハッシュチェーン付き）。その
  単体試験は `decompress_R2004_section` を通して往復させる。
- **コミット:** [`1eb484a6`](https://github.com/monyuonyu/libredwg-fixes/commit/1eb484a6b53445a2f9be6d2a8e09d512b674d485)、[`9a0474f3`](https://github.com/monyuonyu/libredwg-fixes/commit/9a0474f3679c1283b96fffc04197c3a3e0bb34e2)。

### 13.2 R2004 以降の入力：ページ数はセクション数ではない

- **原因:** デコードした R2004 以降の DWG の `header.sections` はページ数である（
  `sample_AC1018` で 56）。
- **症状:** `dwg_sections_init` が "Too many sections" で失敗し、何も書かれなかった。
- **変更:** R2004 以降の出力では 6 に設定し直す。
- **コミット:** [`7cb0849e`](https://github.com/monyuonyu/libredwg-fixes/commit/7cb0849e1bcc8a77e55ae2b2f9fd21770c3ff8b8)。

### 13.3 ファイルヘッダを AutoCAD の書き方で

- **原因:** AutoCAD の R2004 以降のファイルは 0x0C に 3 を持つ。R2018 のファイルは保守版番号を
  0x0B にも持ち、0x15〜0x17 は 0（図面集の AC1032 の図面 37 個すべて）。LibreDWG の R2018 の
  書き直しは 0x0B に 0x1D、0x16 にアプリケーションの版 0x21 を持っていた。古い版から作った R2004 以降のファイルは
  0x0C に 1、0x11 に元のアプリケーションの版を持っていた。
- **変更:** 版ごとに AutoCAD と同じにした。R2018 ではアプリケーションの版を書かない。
- **コミット:** [`649dd383`](https://github.com/monyuonyu/libredwg-fixes/commit/649dd3838c46e3e92277702fef3dfadfb879bd59)。

### 13.4 R2010 と R2013：3 を超える保守版番号

- **原因:** R2010 と R2013 のヘッダとクラスのセクションは、保守版番号が 3 を超えるときだけ、大きさの後に
  4 バイトの hsize を持つ。ACadSharp はそれを 0x0B のバイトから、
  LibreDWG は 0x12 のバイトから取る。R2018 の元は 0x12 が 0 で、0x0B に 0x6d が入っていた。
- **症状:** ACadSharp は hsize を期待してヘッダを読めなかった
  （EndOfStreamException。たとえば R2010 にした `sample_base__empty`）。
- **変更:** AutoCAD のファイルと同じく、R2010 と R2013 では両方を 3 より大きくする。
- **コミット:** [`79c07845`](https://github.com/monyuonyu/libredwg-fixes/commit/79c07845985f416afe2af4a3fc09036050f0e919)。

### 13.5 R2010 以降のヘッダとクラス：hsize を含まないセクションの大きさ

- **原因:** R2010 以降のヘッダとクラスのセクションの大きさに、その後の 4 バイトの hsize を
  数えていた。AutoCAD のクラスの hsize はバイト数ではなく 0。
- **症状:** CRC と終端の印が、大きさの指す位置の 4 バイト手前にあった
  （ACadSharp："Invalid section sentinel found in AcDb:Header"）。
- **変更:** `encode_patch_RLsize_hsize`。クラスの hsize は 0。
- **コミット:** [`ceeaca39`](https://github.com/monyuonyu/libredwg-fixes/commit/ceeaca39aa0f23085a1498bc9880c36c6c250333)。

### 13.6 クラスの版 0 を保つ

- **原因:** AutoCAD は多くのクラスでクラスの版 0 を書く。エンコーダはそれを
  図面の版で置き換えていた。
- **図面:** `2004/Line`（EXACXREFPANELOBJECT、SOLID_BACKGROUND が 31.8 になった）。
- **変更:** 図面の版を使うのは、R2000 以前（クラスの版を持たない）か DXF/JSON から来たとき
  だけにする。
- **コミット:** [`e9563c45`](https://github.com/monyuonyu/libredwg-fixes/commit/e9563c453ac021a08906a38d6c7028252116017c)。

### 13.7 Template、Classes、AcDbObjects の始めと終わりを AutoCAD と同じに

- **原因:** AcDb:Template の説明、クラスのセクションの終わり、最初のオブジェクトの前の 4 バイトが
  AutoCAD のファイルと合っていなかった。
- **変更:** 空の Template の説明は、R2004 では RS 0（全体で 4 バイト）、R2007 以降では
  終端の 0 付きの RS 1（6 バイト）。R2004 以降のクラスは終端の印の後に 0 のバイトを 8 つ置いて終わり、
  それもセクションの大きさに数える（R13〜R2000 では RL 0x0dca のまま）。R2004 以降の AcDbObjects は
  AutoCAD（と ACadSharp）と同じく、0 ではなく RL 0x0dca で始まる。
- **コミット:** [`beb122ae`](https://github.com/monyuonyu/libredwg-fixes/commit/beb122aed566b71c27cdafe1acb810fb736657d3)、[`4738d852`](https://github.com/monyuonyu/libredwg-fixes/commit/4738d85246bf7b3f01553cec5c81513e689879df)、[`7caa82f8`](https://github.com/monyuonyu/libredwg-fixes/commit/7caa82f84092580a14bc2679c4bac1e583dc9fdc)。

### 13.8 Preview：END の印と画像のアドレス

- **原因:** デコードした R2000 のプレビューには END の印が無く、それは R13〜R2000 の出力にしか
  書いていなかった。プレビューの中のヘッダと画像の開始アドレスはファイル上のアドレスだが、
  元からそのまま写していた。
- **症状:** R2004 の出力を読み戻すと："Preview overflow 13208 + 119 > 13311"。
- **変更:** 中身に印が無ければいつでも書く。`thumbnail_rebase` がアドレスを新しい位置に
  移す。
- **コミット:** [`30a1f7a1`](https://github.com/monyuonyu/libredwg-fixes/commit/30a1f7a11d5170dc93f57c8bac3961eb5560990b)。

### 13.9 0x2F0 以降のオブジェクトの型

- **原因:** `bit_write_BOT` は 256 以上の型をすべて RC ＋ 0x1F0 で書き、3 つ目の
  分岐はコード 1 に RS を付けていた。
- **症状:** 0x2F0（752）以降の番号のクラスが、R2010 以降で誤った型になった。
- **変更:** AutoCAD と `bit_read_BOT` と同じく、0x1F0〜0x2EF はコード 1、それ以外は RS 付きの
  コード 2。
- **コミット:** [`78fe37ba`](https://github.com/monyuonyu/libredwg-fixes/commit/78fe37ba18e9581ce347546460ae162225d49b0b)。

### 13.10 R2004 以降のファイルの文字列：T16、TU、文字列ストリーム

- **原因:**
  - R2007 より前、空の T16 の文字列は長さ 1 と NUL である（AutoCAD の SummaryInfo は `01 00 00`）。
    LibreDWG は `00 00` と書いていた（[`754d999d`](https://github.com/monyuonyu/libredwg-fixes/commit/754d999d6a521a0ef329530068b40f525b9080f9)）。
  - R2007 以降、RS の長さは終端の 0 を数え、空の文字列は 0 付きの RS 1。古い版からは
    0 を数えない長さで書いていた。また R2007 以降の元からは R2000/R2004 のファイルにも UTF-16 を
    書いていた（[`ce66db95`](https://github.com/monyuonyu/libredwg-fixes/commit/ce66db95cfd3ac20b3855d8c35ce31dfda9e6adf)）。
  - AutoCAD は TU の文字列を終端の 0 なしで書く（`aec_objects` の NOD：名前 69 個で、文字列ストリームが
    69 × 16 ビット短い）（[`011c3181`](https://github.com/monyuonyu/libredwg-fixes/commit/011c3181df8591028bc1c83e38b349e35303f8ad)、試験 [`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478)）。
  - `has_strings` のビットの前の文字列ストリームの大きさを RS にキャストしていたので、文字列が 32,767 ビットを
    超えると上位の部分が誤っていた（[`632dfd5c`](https://github.com/monyuonyu/libredwg-fixes/commit/632dfd5cc46018f8ca7a4014e5610b67a6e64ef1)）。
- **症状:** SummaryInfo のバイト列が違った。大きな文字列ストリーム（R2018 にした
  `aec_objects` と `gh44-error` のクラス）では、ACadSharp がクラスを違う位置から読んだ。
- **変更:** AutoCAD と同じにした。`bit_write_strings_size` が RS の大きさを書き、大きいものは 0x8000 と
  上位のビットを付ける。R2004 で書いた `2004/Line` は SummaryInfo が AutoCAD のものとバイト単位で一致する。
- **コミット:** [`754d999d`](https://github.com/monyuonyu/libredwg-fixes/commit/754d999d6a521a0ef329530068b40f525b9080f9)、[`ce66db95`](https://github.com/monyuonyu/libredwg-fixes/commit/ce66db95cfd3ac20b3855d8c35ce31dfda9e6adf)、[`011c3181`](https://github.com/monyuonyu/libredwg-fixes/commit/011c3181df8591028bc1c83e38b349e35303f8ad)、[`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478)、[`632dfd5c`](https://github.com/monyuonyu/libredwg-fixes/commit/632dfd5cc46018f8ca7a4014e5610b67a6e64ef1)。

### 13.11 所有する図形、属性、頂点をリストで

- **原因:** R2004 以降は BLOCK_HEADER に first/last の図形のつながりを持たず、INSERT/MINSERT に first/last の
  属性を、POLYLINE に first/last の頂点を持たず、代わりに `num_owned` とハンドルのリストを持つ。
  これらが空だった。
- **症状:** R2004 で書いた `sample_AC1015` は 355 図形すべてを失った。
- **変更:** `upconvert_owned_lists` が図形を所有者ごとにオブジェクトの順で集める
  （BLOCK、ENDBLK、SEQEND を除く）。ATTRIB はその INSERT に、VERTEX はその POLYLINE に。
- **コミット:** [`2098043b`](https://github.com/monyuonyu/libredwg-fixes/commit/2098043bbaaa4d2ad816a76209e757bc019b6144)。

### 13.12 PLOTSETTINGS、LAYOUT：シェード印刷の既定値

- **原因:** R2000 にはシェード印刷のフィールドが無く、R2004 以降ではユーザー定義の DPI が 0 になっていた。
- **症状:** ACadSharp がそれをはね（"The valid shade plot DPI values range from 100 to
  23767"）、LAYOUT がすべて失われた。
- **変更:** AutoCAD の R2004 の図面（`sample_AC1018`）の値、解像度レベル 2 と 300 DPI
  にする。
- **コミット:** [`653fea5e`](https://github.com/monyuonyu/libredwg-fixes/commit/653fea5ebdf0e3e7edb2a306165ed075f9fd9fa0)。

### 13.13 古い版の XRECORD のデータ

- **原因:** R2007 以降、xdata の文字列は UTF-16 だが、書き出しは元の
  `xdata_size` で上限が決まっていた。
- **症状:** R2000 からだと、最初の文字列より後の値が切れた。ACadSharp はこれらの
  xrecord で失敗した（"Arithmetic operation resulted in an overflow"、R2010 にした `sample_AC1015` で 8 個）。
- **変更:** 版をまたぐときは大きさの 2 倍まで許す。大きさは計算し直す。
- **コミット:** [`5300c579`](https://github.com/monyuonyu/libredwg-fixes/commit/5300c579ff3943543011c6de2dd4e3b70df36471)。

### 13.14 DWG の入力の MATERIAL のハンドル

- **原因:** R2007 以降を書くとき、図形、LAYER、そして 3DSOLID、
  REGION、BODY のマテリアルのハンドルを落としていた。MATERIAL オブジェクトはプレースホルダとして書かれるから、という理由だが、
  そうなるのは DXF から来たときだけ。
- **症状:** R2018 から R2018 で、すべての画層のマテリアルが失われた。
- **変更:** 落とすのは DXF の入力のときだけにする。
- **コミット:** [`036da95e`](https://github.com/monyuonyu/libredwg-fixes/commit/036da95ef1a810ef1c60826b0aa535ba1d9ae65d)。

### 13.15 AcDbBlockElement：読んだとおりの版

- **原因:** `be_major` と `be_minor` を R2007 以降では 33 と 29 で書いていた。AutoCAD の
  R2018 の図面は 33 と 329 を持つ。
- **変更:** R2007 以降の DWG から読んだとおりに書く。
- **コミット:** [`eac6288a`](https://github.com/monyuonyu/libredwg-fixes/commit/eac6288a16743092f005cecbddfd848d007d8739)。

### 13.16 DIMSTYLE：R2007 以降の変数に AutoCAD の既定値を

- **原因:** 古い版や DXF から来ると、DIMJOGANG、DIMFXL、DIMALTMZF、DIMMZF
  が 0 で、DIMTFILLCLR は色の方式を持たなかった。最初の修正は古い DWG しか扱っておらず、DXF からでは
  `dat.from_version` が対象の版になっている。
- **症状:** ACadSharp は DIMJOGANG 0 をはね（"must be in range of 5 to 90 degrees"）、
  そうした寸法スタイルを落とし、寸法は Standard になった（R2010 以降への往復で 63 個）。
- **変更:** AutoCAD の既定値：DIMFXL 1、DIMJOGANG 45°、DIMTFILLCLR ByBlock、DIMALTMZF
  と DIMMZF 100。DIMSTYLE にもヘッダ変数にも、DXF から来たときにも。
- **コミット:** [`16871d28`](https://github.com/monyuonyu/libredwg-fixes/commit/16871d2898ba823f957abae68a73c7114be2e50a)、[`c720b36d`](https://github.com/monyuonyu/libredwg-fixes/commit/c720b36d04579353ac62ebda61b43dde5f379f72)。

### 13.17 SPLINE：R2013 以降のフラグ、ノットのパラメータ、シナリオ

- **原因:** 古い版から来ると `splineflags` は 8 か 9、`knotparam` は 0 で、DXF から来ると
  どちらも 0 だった。ACadSharp はこれらからシナリオを決めるので、制御点のスプラインがフィット
  点のスプラインに見えた。LibreDWG はデコードしたシナリオ（フィット点のスプラインなら 2）も書いていた。AutoCAD は
  図面集の R2013 以降のスプライン 591 個すべてで 1 を書く。
- **症状:** "Failed to read ReadBitDouble" が出てスプラインが失われた（図面集で DXF から 32 個）。
- **変更:** AutoCAD と同じにした：制御点は `splineflags` 0（閉じていれば 4）と
  `knotparam` 15、フィット点は 9 と 0。シナリオの BL は 1。
- **コミット:** [`09d03974`](https://github.com/monyuonyu/libredwg-fixes/commit/09d03974d1cd3423d68f088bcfdcc7dd85787d29)、[`ab05c9d1`](https://github.com/monyuonyu/libredwg-fixes/commit/ab05c9d13e521420f001422af986c9a303d5833e)、[`f3322488`](https://github.com/monyuonyu/libredwg-fixes/commit/f332248859806fe1609bb05e4e845ffd1a8629a9)。

### 13.18 途中で見つかったもの、R2000 にも影響するもの

- **`bit_write_UMC`** は先頭のゼロを 8 グループ中 4 つしか調べていなかったので、どの値も最低 4
  バイトになった（オフセット 1 が `81 80 80 00`。AutoCAD は `01` と書く）。`2004/Line` のオブジェクトマップは 674 バイトではなく
  1,174 バイトだった（[`691d4598`](https://github.com/monyuonyu/libredwg-fixes/commit/691d4598e5c03d21047caac959358f8df2ba21ad)）。また、符号付きの MC のように、最後のバイトに 0x40 が立っていると 0 のバイトを
  足していた。R2010 以降では各オブジェクトの前のハンドルストリームの大きさが 1 バイト増えた
  （[`40d6e568`](https://github.com/monyuonyu/libredwg-fixes/commit/40d6e568c035ba2af2c37dd45bc581f1e9835912)）。いまは最短の形で書く。
- **`bit_write_DD`** は既定値から 1e-12 以内の値を既定値として書いていた：始点から 1 ulp ずれた
  LINE の終点（`example_2004`）が始点として戻ってきた。いまはビット単位で同じ値だけを既定値とする。
  R2004 で書いた `example_2004` と `sample_AC1018` の LINE、LWPOLYLINE、3DFACE は AutoCAD の
  オブジェクトとビット単位で一致する（[`57cbcdd9`](https://github.com/monyuonyu/libredwg-fixes/commit/57cbcdd91fb0cb99437098919fd66508606e3b5a)）。
- **オブジェクトマップ：** 最後のハンドルでチャンクがいっぱいになると（2030 バイト超）、大きさ 2 の最後の
  チャンクの前に予約の 2 バイトが 0 のまま残り、ACadSharp が終わりを越えて読んだ（R2000 にした `sample_AC1032`）
  （[`d56fd195`](https://github.com/monyuonyu/libredwg-fixes/commit/d56fd19555ec86e6366a6c0a3b3497d2b8c8acbb)）。
- **SPATIAL_INDEX：** 形式が分かっていない。`aec_objects` の `bindata_size` は 50,396,167 で、
  デコーダははねていたが、エンコーダはその数だけ 0 のバイトを書き、51 MB のオブジェクトのセクションになった。
  いまは不正な大きさは 0 にする（[`5398b8a6`](https://github.com/monyuonyu/libredwg-fixes/commit/5398b8a6c63a4e43815114dbbfe0df1dd2c2db12)）。
- **コミット:** [`691d4598`](https://github.com/monyuonyu/libredwg-fixes/commit/691d4598e5c03d21047caac959358f8df2ba21ad)、[`40d6e568`](https://github.com/monyuonyu/libredwg-fixes/commit/40d6e568c035ba2af2c37dd45bc581f1e9835912)、[`57cbcdd9`](https://github.com/monyuonyu/libredwg-fixes/commit/57cbcdd91fb0cb99437098919fd66508606e3b5a)、[`d56fd195`](https://github.com/monyuonyu/libredwg-fixes/commit/d56fd19555ec86e6366a6c0a3b3497d2b8c8acbb)、[`5398b8a6`](https://github.com/monyuonyu/libredwg-fixes/commit/5398b8a6c63a4e43815114dbbfe0df1dd2c2db12)。

R2004 以降についてはほかに：クラスのエントリ（[1.1](#11-r2004-以降のクラスのエントリdwg_version-と-maint_version-は-bl)）、
ヘッダ（[1.4](#14-r2007-以降のヘッダunknown_54--unknown_57-は無い)）、FileDepList
（[1.5](#15-filedeplist機能名は-t32-の文字列)）、EED からのトゥルーカラー
（[2.6](#26-r13r2000-から-r2004-以降へautocad-が-eed-に残すトゥルーカラー)）、DXF の文字列
（[3.3](#33-dxf-から-r2007-以降へ名前が最初の-1-文字で切れていた)）、表
（[6.8](#68-古い版や-dxf-から-r2010-以降に書く表)、[6.9](#69-古い版から-r2007-以降を書くときのメモリリーク)）、
ACIS（[7.2](#72-書き出し空ビット履歴のハンドルマテリアル)〜[7.6](#76-dxfsat-の行とエスケープされた-)）、
プロキシ（[8.5](#85-r2004-と-r2010-以降が対象のときもプロキシに)）、LEADER
（[9.5](#95-leaderr2010-以降の配置)）、BLOCKSCALEACTION（[9.6](#96-blockscaleaction接続点)）、
DXF のハンドルと `dwgwrite`（[10.8](#108-所有する頂点と属性は絶対ハンドルで)、
[10.9](#109-dwgwrite-は対象の版として-dxf-を読む)）。

## 14. ツールと試験

### 14.1 JSON：NaN を出さない

- **原因:** JSON には NaN が無い。`VALUE_RD` はリリースビルドでしか NaN を置き換えず、
  無限大も置き換えていなかった。ベクトルの中の double（`FIELD_VECTOR_N`。たとえば SPATIAL_FILTER の行列）は
  `fprintf` で直接出力していた。
- **症状:** 読み誤った RAPIDRTRENDERSETTINGS で `dwgread -O json` が "-nan" を出力し、
  出力全体が読めなくなった。
- **図面:** `gh44-error`。
- **変更:** NaN と無限大の double は 0.0 と書く。確認は `_VALUE_RD` の 1 か所で、すべての
  ビルドで行い、ベクトルにも使う。
- **コミット:** [`c86190fb`](https://github.com/monyuonyu/libredwg-fixes/commit/c86190fbcc0db292aa09e9592e24bdca4b8513ef)、[`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8)（一部） — PR [#1441](https://github.com/LibreDWG/libredwg/pull/1441)。

### 14.2 `dwgrewrite`、`dxf2dwg`、`dwgwrite`：R2010、R2013、R2018 も正しい指定に

- **変更:** ヘルプの文と man ページで、r2010、r2013、r2018 を正しい出力の版として挙げる
  （r2007 は引き続き r2010 として書く）。`dwgwrite` も r2000 だけではなくなった。図面集の 149 図面すべてが
  R2010、R2013、R2018 として ACadSharp を通した往復で何も失わず、R2004〜R2018 で書いた DXF としても
  同じ（残る値の変化は R2000/R2004 で 53 個、R2010 以降で 66 個で、すべて
  TABLE の中）。
- **コミット:** [`416c3d22`](https://github.com/monyuonyu/libredwg-fixes/commit/416c3d222396b1419a4dc6edf27e97969135d61a)、[`1b613fcb`](https://github.com/monyuonyu/libredwg-fixes/commit/1b613fcbd8b4a04f44b1bc9d2ea591a43d346c04) — 本家にはまだ送っていない。

ほかの試験の変更は、それぞれの修正と一緒にある：[`45e6b50f`](https://github.com/monyuonyu/libredwg-fixes/commit/45e6b50f615aa7346827f71a26de67dc3a8f0eae)（[3.1](#31-アジアのコードページ2-バイト文字の先頭バイト)）、
[`aad393ec`](https://github.com/monyuonyu/libredwg-fixes/commit/aad393ec206b1116694db8ae7877874207530709)（[7.1](#71-読み取りr2013-以降は-acis-の空ビットが無いr2007-以降の不明な-bl)）、
[`9a0474f3`](https://github.com/monyuonyu/libredwg-fixes/commit/9a0474f3679c1283b96fffc04197c3a3e0bb34e2)（[13.1](#131-r2004-のファイルの器lz77-の圧縮付き)）、
[`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478)（[13.10](#1310-r2004-以降のファイルの文字列t16tu文字列ストリーム)）。
