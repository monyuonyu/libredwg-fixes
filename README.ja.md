[English](README.md) | **日本語** | [简体中文](README.zh-CN.md)

> [README.md](README.md)（英語版が原文）の翻訳です。

# libredwg-fixes

[GNU LibreDWG](https://github.com/LibreDWG/libredwg) 0.14.8601 の不具合を直した版です。同じ図面を独立した実装で読んだ結果や、
AutoCAD 自身が保存した図面と突き合わせて、食い違いを一つずつ直しました。
修正はブランチ `fixes`（既定のブランチ）にあり、本家のタグ `0.14.8601` の上に載っています。
本家にはプルリクエストとして送っていて、取り込まれたらこのフォークの役目は終わりです。
日本語化した FreeCAD（[freecad-ja-unified](https://github.com/monyuonyu/freecad-ja-unified)）の DWG の読み書きに使っています。

## 確かめたこと

- **読み取り** — 149 図面（R13〜R2018。ACadSharp の AutoCAD 製の見本と、LibreDWG 自身の試験データ）を、LibreDWG と、
  独立した MIT ライセンスの読み取り実装 [ACadSharp](https://github.com/DomCR/ACadSharp) 3.8.0 の両方で読み、
  図形ごと・表のレコードごと・ヘッダ変数ごとに比べました。結果、値や種類の違いは残っていません
  （図形 59,883、表のレコード 2,936、ヘッダ変数 30,412）。ACadSharp が読まないものは除きます。
- **R2000 への書き出し** — すべての図面を R2000 で、直接（`dwgrewrite --as r2000`）と DXF 経由（`dwg2dxf` のあと `dxf2dwg`）で書き、
  ACadSharp で読み直しました。消えたものは無く、種類の変化もありません。ほかのアプリ独自の図形は、AutoCAD と同じくプロキシとして残します。
- **AutoCAD 自身との比較** — AutoCAD が同じ図面を全部の版で保存したもの（sample_AC1014 … AC1032、example_r14 … example_2018）があります。
  新しい版のものを R2000 で書き、AutoCAD 自身の R2000 保存と比べました。example_2004 … 2018 は 222 図形すべて一致し、
  全体では 3,405 図形中 3,258 が一致しました。残りは、AutoCAD が保存のたびに振り直すハンドルと無名ブロックの番号、
  R14 には保存できない線の太さ、そして AutoCAD がしている 3 つの変換（積み重ね分数のコード、ダイナミックブロックの参照、
  フィット点のスプラインを制御点に）をしていないことによるものです。
- **R2004・R2010・R2013・R2018 への書き出し**（新規。本家は R2000 以前しか書けません） — すべての図面を各版で、
  直接と DXF 経由（`dxf2dwg --as r2018`）で書き、ACadSharp で読み直しました。読めない図面は無く、消えたものも種類の変化もありません。
  同じ版の AutoCAD 自身の保存と比べると、R2018 で 3,405 図形中 3,208 が一致しました（残りは上と同じ種類と、TABLE のセル）。
  ACIS のソリッドも保たれます。19 図面の 107 体のソリッドの SAT/SAB は、書いたあともバイト単位またはトークン単位で一致しました。
  R2013 より前の図面や DXF から、R2013 以降の AcDs セクションに書く場合も同じです。
- `make check` が通ります。

## 直したこと（116 コミット）

読み取り
- R2004 以降のクラスセクション：`dwg_version`、`maint_version` は BL（仕様書では BS）。R2018 の AuxHeader：保守版番号は RL。
- 図形の色：RGB は透明度より前。R2007 以降のカラーブックのハンドルを 2 回読んでいた（画層を失っていた）。
- MTEXT：R2018 で、枠のビットだけのときの背景の塗り。
- R2013 以降の 3DSOLID/REGION/BODY：先頭の ACIS の空ビットは無い。R2007 以降の不明な BL。
- R2018 の複数行の ATTRIB/ATTDEF：埋め込みの MTEXT。R2010 以降の ATTRIB に `keep_duplicate_records` は無い。
- TABLE/TABLECONTENT（デバッグ用だけだった）と R2010 以降の TABLESTYLE（3 つあるセルスタイルの最初しか読んでいなかった）。
  R2013 の RENDERSETTINGS。R13/R14 の LAYER の非表示、R13/R14 の図形の線種。
- EED の画層（1003）のハンドルはビッグエンディアン。
- アジアのコードページ：2 バイト文字の先頭バイト（`>> 8` のところが `>> 4`）。CP932 の 1 バイト文字で文字列が切れていた。

新しい版から R2000 への書き出し
- 色：ByBlock の図形が ByLayer になっていた。トゥルーカラーは一緒に保存されている ACI の番号を保つ。ACI の色表を AutoCAD（と ezdxf）と同じに。
- EED の文字列：コードページはビッグエンディアン、長さは変換後のもの。AuxHeader の大きさ。VX_CONTROL のハンドル。
- VERTEX/ATTRIB のつながり、MLINESTYLE の線種の番号、LWPOLYLINE の頂点 ID のフラグ、表を古い形式で、グラデーションのハッチング、
  MTEXT の `\P`、R13/R14 の名前、線の太さと印刷のフラグ。
- 独自の図形とオブジェクトは、POINT/DUMMY ではなく ACAD_PROXY_ENTITY / ACAD_PROXY_OBJECT として。

DXF
- TABLE の読み書き、R2007 の形の TABLESTYLE、プロキシ、SPATIAL_FILTER、埋め込みの MTEXT、EED 1003/1004、
  HATCH の種点と名前、LWPOLYLINE の幅、トゥルーカラー（420）を 24 ビットの RGB で、R2007 以降の DXF の UTF-8 の文字を R2000 向けに変換、
  `*Model_Space` のハンドル、ヘッダの既定値、名前の完全一致、SEQEND のあとの ATTRIB/VERTEX、R14 の図面が新しいクラスを保つこと、
  R2004 以降の図面の `dwg2dxf --as r2000` で図形が出なかったこと。

R2004〜R2018 への書き出し（62 コミット）
- AutoCAD と同じ R2004 のファイルの入れ物：ページとセクションの対応表、暗号化したヘッダ、ページのチェックサム、LZ77 の圧縮器。
  文字列ストリームを持つ R2010 以降のオブジェクトの形。ACIS の図形の SAB を入れる R2013 以降の AcDs データセクション（SAT と SAB の相互変換）。
- 古い図面を新しい版へ：所有する図形のリスト、PLOTSETTINGS/LAYOUT、DIMSTYLE、SPLINE、LEADER、TABLE の内容、XRECORD、
  3DSOLID の履歴のハンドル、AutoCAD が EED に保存したトゥルーカラー。
- 途中で見つかった、R2000 にも効く修正：`bit_write_UMC` の最短の形、`bit_write_DD` の丸め、オブジェクトマップの区切り、
  FileDepList の文字列、SPATIAL_INDEX が 51 MB のゼロを書いていたこと、`dwgwrite`/`dxf2dwg` が出力の版を決める前に DXF を読んでいたこと。

各コミットのメッセージに、何が誤っていたか、どう表に出たか、どの図面で分かるかを書いています。

## ダウンロード

[リリース](https://github.com/monyuonyu/libredwg-fixes/releases)ごとに、本家の `libredwg-0.14.8601.tar.xz` に当てるパッチを付けています。
`libredwg-fixes-patches.tar.gz`（修正ごとのファイル、`git am` 用）と `libredwg-fixes.patch`（1 つのファイル、`patch -p1` 用）です。
このリポジトリを clone しても使えます。

## ビルド

本家の LibreDWG と同じです（`README` と `INSTALL` を参照）。

```sh
git clone https://github.com/monyuonyu/libredwg-fixes.git
cd libredwg-fixes
sh ./autogen.sh
./configure --disable-bindings
make
make check
```

## ライセンス

LibreDWG と同じく GNU GPL v3 以降です。変更はすべて本家に提供しています。
