# libredwg-fixes

Bug fixes for [GNU LibreDWG](https://github.com/LibreDWG/libredwg) 0.14.8601, found by reading the
same drawings with an independent implementation and comparing with AutoCAD's own saves.
The fixes are on the branch `fixes` (the default branch), on top of the upstream tag `0.14.8601`.
They are being offered upstream as pull requests; once they are merged this fork has done its job.

[日本語の説明は下にあります](#日本語)

## What was checked

- **Reading** — 149 drawings (R13 to R2018, AutoCAD's ACadSharp samples and LibreDWG's own test
  data) read with LibreDWG and with [ACadSharp](https://github.com/DomCR/ACadSharp) 3.8.0, an
  independent MIT-licensed reader, and compared entity by entity, table record by table record and
  header variable by header variable. Result: no value or type differences left (59,883 entities,
  2,936 table records, 30,412 header variables), except things ACadSharp does not read.
- **Writing R2000** — every drawing written as R2000, directly (`dwgrewrite --as r2000`) and via DXF
  (`dwg2dxf` then `dxf2dwg`), and read back with ACadSharp: nothing lost, no type changes; custom
  entities of other applications are kept as proxies as AutoCAD does.
- **Against AutoCAD itself** — AutoCAD saved the same drawings in every version (sample_AC1014 …
  AC1032, example_r14 … example_2018). Writing the newer ones as R2000 and comparing with AutoCAD's
  own R2000 save: example_2004 … 2018 are equal in all 222 entities; overall 3,258 of 3,405
  entities are equal. The rest are handles and anonymous block numbers AutoCAD renumbers on every
  save, lineweights R14 cannot store, and three AutoCAD conversions not done (stacked fraction
  codes, dynamic block references, fit-point splines as control points).
- **Writing R2004, R2010, R2013, R2018** (new: upstream writes R2000 and older only) — every
  drawing written in each version, directly and via DXF (`dxf2dwg --as r2018`), and read back with
  ACadSharp: no drawing unreadable, nothing lost, no type changes. Compared with AutoCAD's own save
  in the same version, R2018: 3,208 of 3,405 entities equal (the same kinds of rest as above, plus
  TABLE cells). ACIS solids are kept: SAT/SAB of 107 solids in 19 drawings byte-identical or
  token-identical after writing, also from pre-R2013 drawings and DXF into the R2013+ AcDs section.
- `make check` passes.

## What was fixed (116 commits)

Reading
- R2004+ class section: `dwg_version`, `maint_version` are BL (the spec says BS); R2018 AuxHeader:
  the maintenance versions are RL.
- Entity colors: RGB before transparency; R2007+ color book handle read twice (lost the layer).
- MTEXT: R2018 background fill with only the frame bit.
- 3DSOLID/REGION/BODY since R2013: no leading ACIS empty bit; the R2007+ unknown BL.
- R2018 multi-line ATTRIB/ATTDEF: the embedded MTEXT; no `keep_duplicate_records` in R2010+ ATTRIB.
- TABLE/TABLECONTENT (were debugging-only) and R2010+ TABLESTYLE (only the first of three cell
  styles was read); R2013 RENDERSETTINGS; R13/R14 LAYER off, R13/R14 entity linetypes.
- EED layer (1003) handle is big-endian.
- Asian codepages: the lead byte of two-byte characters (`>> 4` instead of `>> 8`), single-byte
  characters of CP932 cut the string.

Writing R2000 from newer versions
- Colors: ByBlock entities became ByLayer; true colors keep the ACI index stored with them; the ACI
  palette as AutoCAD (and ezdxf) has it.
- EED strings: codepage big-endian and the converted length; AuxHeader size; VX_CONTROL handle.
- VERTEX/ATTRIB links, MLINESTYLE linetype index, LWPOLYLINE vertex id flag, tables in the old
  format, gradient hatches, MTEXT `\P`, R13/R14 names, lineweights and plot flags.
- Custom entities and objects as ACAD_PROXY_ENTITY / ACAD_PROXY_OBJECT instead of POINT/DUMMY.

DXF
- TABLE import/export, TABLESTYLE in the R2007 layout, proxies, SPATIAL_FILTER, embedded MTEXT,
  EED 1003/1004, HATCH seeds and names, LWPOLYLINE widths, true colors (420) as 24-bit RGB,
  UTF-8 text of R2007+ DXF converted for R2000, `*Model_Space` handle, header defaults,
  exact name matches, ATTRIB/VERTEX after a SEQEND, R14 drawings keeping their newer classes,
  `dwg2dxf --as r2000` of R2004+ drawings writing no entities.

Writing R2004 to R2018 (62 commits)
- R2004 file container as AutoCAD writes it: page and section maps, encrypted header, page
  checksums, an LZ77 compressor; the R2010+ object format with its string stream; R2013+ AcDs
  data section with the SAB of the ACIS entities (SAT to SAB and back).
- Upgrading older drawings: owned entity lists, PLOTSETTINGS/LAYOUT, DIMSTYLE, SPLINE, LEADER,
  TABLE content, XRECORD, 3DSOLID history handles, true colors saved by AutoCAD in EED.
- Fixes found on the way that also affect R2000: `bit_write_UMC` shortest form, `bit_write_DD`
  rounding, object map chunks, FileDepList strings, SPATIAL_INDEX writing 51 MB of zeros,
  `dwgwrite`/`dxf2dwg` reading the DXF before the target version was set.

Each commit message says what was wrong, how it showed, and which drawing shows it.

## Build

The same as upstream LibreDWG (see `README` and `INSTALL`):

```sh
git clone https://github.com/monyuonyu/libredwg-fixes.git
cd libredwg-fixes
sh ./autogen.sh
./configure --disable-bindings
make
make check
```

## License

GNU GPL v3 or later, as LibreDWG. All changes are offered to the upstream project.

---

## 日本語

[GNU LibreDWG](https://github.com/LibreDWG/libredwg) 0.14.8601 の不具合を直した版です（ブランチ `fixes`）。
同じ図面を独立した実装（ACadSharp）で読んだ結果や、AutoCAD 自身が版を変えて保存した図面と突き合わせて、
食い違いを一つずつ直しました。直しは本家にプルリクエストとして送っていて、取り込まれたらこの版の役目は終わりです。

- 読み取り: 149 図面（R13〜R2018）で ACadSharp と図形・表・ヘッダ変数を突き合わせ、値の違いは 0
- R2000 への書き出し（DWG のまま／DXF 経由）: 消える図形 0、独自アプリの図形は AutoCAD と同じくプロキシとして残す
- **R2004・R2010・R2013・R2018 での書き出し**（本家は R2000 まで）: 各版に書いて読み直し、読めない図面 0・消える図形 0。3D ソリッド（ACIS）も保たれる
- AutoCAD 自身の R2000 保存との比較: example_2004〜2018 は 222 図形すべて一致、全体で 3,258 / 3,405 図形が一致
- 日本語・中国語など 2 バイト文字の変換の誤りも直しています
- この版は、日本語化した FreeCAD（[freecad-ja-unified](https://github.com/monyuonyu/freecad-ja-unified)）の DWG の読み書きに使っています
