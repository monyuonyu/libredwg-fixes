**English** | [日本語](FIXES.ja.md) | [简体中文](FIXES.zh-CN.md)

# Every fix, explained

This page explains the 116 fix commits of the branch `fixes`, which sit on top of the upstream tag
`0.14.8601`. For each fix it says what was wrong, how it showed to a user, which drawing shows it,
and what was changed, with a link to the commit. The [README](../README.md) has the short summary.

## How the fixes were found

- **An independent reader.** 149 drawings (R13 to R2018) were read with LibreDWG and with
  [ACadSharp](https://github.com/DomCR/ACadSharp) 3.8.0, an independent MIT-licensed reader, and
  compared entity by entity, table record by table record and header variable by header variable.
  Every difference was traced to the bits. Sometimes LibreDWG was wrong, sometimes the written
  specification (the ODA "Open Design Specification for .dwg files", quoted as "ODA 20.4.x").
  Where the spec and AutoCAD's files disagree, the files win.
- **AutoCAD's own saves.** AutoCAD saved the same drawing in every version: `sample_AC1014` …
  `sample_AC1032` (ACadSharp's samples) and `example_r14` … `example_2018` (LibreDWG's test data).
  Converting one of them to another version and comparing with AutoCAD's own save of that version
  shows what AutoCAD does when it converts.
- **Round trips.** Every drawing was written as R2000, R2004, R2010, R2013 and R2018, directly
  (`dwgrewrite`) and via DXF (`dwg2dxf`, then `dxf2dwg`), and read back with ACadSharp. Anything lost
  or changed on the way is a bug.

Drawing names are those of the test corpus. `sample_AC10xx` and `sample_base` are ACadSharp's
samples; `example_*`, `gh109_1`, `gh44-error`, `gh209_1`, `2000/TS1`, `2004/Line` and
`2018/Dynblocks` are in LibreDWG's `test/test-data`. Field and function names are LibreDWG's
(mostly `src/dwg.spec`, `src/decode.c`, `src/encode.c`, `src/in_dxf.c`, `src/out_dxf.c`). DWG
types: B bit, BS/BL/BD bit-coded short/long/double, RC/RS/RL raw char/short/long, TV/TU 8-bit and
UTF-16 text, H handle.

### Upstream pull requests

Most fixes for reading, for writing R2000 and for DXF are offered to
[LibreDWG](https://github.com/LibreDWG/libredwg) as pull requests. Their commits were rebased onto
upstream `master`, and some were split or merged on the way, so one commit here can correspond to
parts of several PRs.

| PR | Topic |
|---|---|
| [#1440](https://github.com/LibreDWG/libredwg/pull/1440) | Asian codepages: the lead byte of two-byte characters |
| [#1441](https://github.com/LibreDWG/libredwg/pull/1441) | JSON output: no NaN |
| [#1442](https://github.com/LibreDWG/libredwg/pull/1442) | Reading fixes (classes, colors, AuxHeader, ACIS, MTEXT, LAYER, EED, ...) |
| [#1443](https://github.com/LibreDWG/libredwg/pull/1443) | R2018 multi-line ATTRIB/ATTDEF |
| [#1444](https://github.com/LibreDWG/libredwg/pull/1444) | Writing R2000 from newer versions |
| [#1445](https://github.com/LibreDWG/libredwg/pull/1445) | DXF import and export |
| [#1446](https://github.com/LibreDWG/libredwg/pull/1446) | TABLE, TABLECONTENT, TABLESTYLE |
| [#1447](https://github.com/LibreDWG/libredwg/pull/1447) | Unknown objects as proxies |

The 62 commits for writing R2004 to R2018 (from [`1eb484a6`](https://github.com/monyuonyu/libredwg-fixes/commit/1eb484a6b53445a2f9be6d2a8e09d512b674d485) on) are not yet offered upstream:
upstream LibreDWG writes R2000 and older only.

## Contents

1. [Classes, header and other file sections](#1-classes-header-and-other-file-sections)
2. [Colors](#2-colors)
3. [Text and codepages](#3-text-and-codepages)
4. [Extended entity data (EED)](#4-extended-entity-data-eed)
5. [MTEXT, ATTRIB and ATTDEF](#5-mtext-attrib-and-attdef)
6. [TABLE and TABLESTYLE](#6-table-and-tablestyle)
7. [ACIS solids (3DSOLID, REGION, BODY)](#7-acis-solids-3dsolid-region-body)
8. [Proxies and unknown objects](#8-proxies-and-unknown-objects)
9. [Other entities and objects](#9-other-entities-and-objects)
10. [DXF import](#10-dxf-import)
11. [DXF export](#11-dxf-export)
12. [Writing R2000 from newer versions](#12-writing-r2000-from-newer-versions)
13. [Writing R2004 to R2018](#13-writing-r2004-to-r2018)
14. [Tools and tests](#14-tools-and-tests)

---

## 1. Classes, header and other file sections

### 1.1 R2004+ class entries: `dwg_version` and `maint_version` are BL

- **What was wrong:** the ODA spec (10.2) lists both fields of a class entry as BS, and
  `read_2004_section_classes` and the R2007 reader `read_2007_section_classes` followed it. AutoCAD
  writes BL (ACadSharp reads BL too). BS and BL agree only for values below 256. The encoder
  (`encode_classes`) also wrote BS. The second unknown long was stored in `unknown_1` a second time
  instead of `unknown_2`.
- **How it showed:** in AutoCAD's R2004 sample class 508 MLEADERSTYLE has a `maint_version` above
  255. The BS read lost 16 bits, every following class was garbage (number 9974, dxfname "W'6..."),
  and every entity of a class numbered 509 or higher (MULTILEADER, MESH, WIPEOUT, IMAGE,
  PDFUNDERLAY, ...) became UNKNOWN_OBJ. A drawing written as R2004 had every class after the first
  shifted, and objects of custom classes got invalid types when read back ("Invalid class 523").
- **Drawing:** `sample_AC1018`.
- **What was changed:** read and write both fields as BL; keep the second long in `unknown_2`. The
  sample now matches ACadSharp for 353 entities.
- **Commits:** [`19c7ab68`](https://github.com/monyuonyu/libredwg-fixes/commit/19c7ab689d0583a341b69867ea70d879ff97417e) (reading, in PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)),
  [`b25ba83f`](https://github.com/monyuonyu/libredwg-fixes/commit/b25ba83f095951ad1f2ddd3b0598775ab8ce4da3) (writing, not yet offered upstream).

### 1.2 The number of classes was bounded too low

- **What was wrong:** to guard against huge allocations (CWE-400) the decoder allowed at most
  `100 + size / sizeof(Dwg_Class)` classes. Valid drawings with short class names have more.
- **How it showed:** an AEC drawing written to R2000 has 478 classes; the bound allowed 385, so all
  objects of the later classes were unreadable.
- **What was changed:** a class entry has at least 13 bits (BS number, BS flags, three TV with a BS
  length, B, BS item class id), so a section of `size` bytes holds at most `size*8/13` entries.
  Memory stays linear in the input. The same for the R2004+ classes section.
- **Commit:** [`4ff2ed83`](https://github.com/monyuonyu/libredwg-fixes/commit/4ff2ed83ccafa44fa85620ab1b2621dc681248bc) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447).

### 1.3 R2018 AuxHeader: the maintenance versions are RL

- **What was wrong:** in the R2018 AuxHeader `maint_version`, `maint_version_1` and
  `maint_version_2` are RL. LibreDWG read each as RS and the three upper halves (6 bytes) as
  `zero_18` at the end, so everything after `maint_version` was shifted by 2 bytes.
- **How it showed:** `minus_1` was 0xffff0000, TDCREATE/TDUPDATE were garbage, and R2018 drawings
  written as R2000 carried the garbage dates, which ACadSharp could not read.
- **Drawing:** AutoCAD's R2013 and R2018 saves of the same drawing have 123- and 129-byte
  AuxHeaders; the 6 bytes are the upper halves of the three maintenance versions.
- **What was changed:** since R2018 the upper half is read right after each maintenance version.
  Now `minus_1` is −1, and the dates and HANDSEED match the
  header.
- **Commit:** [`c7efdf98`](https://github.com/monyuonyu/libredwg-fixes/commit/c7efdf987d7d3817456f94b789cc7345e558975f) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442).

### 1.4 R2007+ header: no `unknown_54` … `unknown_57`

- **What was wrong:** the header variables had four BS `unknown_54` … `unknown_57` after
  SHADOWPLANELOCATION in all versions since R14. Since R2007 the string stream starts right there.
- **How it showed:** the four values were read from the strings (always 1, 180, 36874, 32843) and
  written back as 56 bits of extra data.
- **Drawings:** all R2007+ drawings of AutoCAD (`example_2007` … `example_2018`, `sample_AC1021` …
  `sample_AC1032`).
- **What was changed:** the four fields only for R14 to R2004 (`src/header_variables.spec`).
- **Commit:** [`1a42025a`](https://github.com/monyuonyu/libredwg-fixes/commit/1a42025ad7e5294c05853c27372cd5ccb086a4dd) — not yet offered upstream.

### 1.5 FileDepList: the feature names are T32 strings

- **What was wrong:** the R2004 FileDepList section has an RL length and the characters of every
  string, without a terminating NUL, for the feature names as for the file names
  (`02000000 09000000 "Acad:Text" 0f000000 "Acad:PlotConfig" 01000000 ...`). The feature names
  were read as bit-coded TV.
- **How it showed:** all feature names came out empty and the rest of the section was shifted;
  written back it was a broken section.
- **Drawing:** `2004/Line`.
- **What was changed:** read and write the features with `FIELD_T32` like the file strings;
  `bit_write_T32` writes the 8-bit form without the NUL, as AutoCAD. `2004/Line` written as R2004
  now has AutoCAD's FileDepList byte for byte.
- **Commit:** [`5fa2061b`](https://github.com/monyuonyu/libredwg-fixes/commit/5fa2061b838b24177d32bc6af74ea45224e9f1d3) — not yet offered upstream.

The AuxHeader written for R2000 is in [12.4](#124-auxheader-123-bytes-for-r2000-too); the class
entries, header and section sizes written for R2004+ are in [13](#13-writing-r2004-to-r2018).

## 2. Colors

### 2.1 Entity colors: the RGB comes before the transparency

- **What was wrong:** in an entity color (ENC, ODA 2.11) flag 0x8000 says "next value is the RGB"
  and flag 0x2000 "followed by a transparency BL", in that order. `common_entity_data.spec` read
  the transparency first.
- **How it showed:** an entity with both a true color and a transparency had them swapped, so it
  had no true color at all, only its index: LINE 9D4 got rgb 0x01000000 and transparency
  0xc2803dc2 (index 183 only).
- **Drawing:** `sample_AC1032`. Now rgb is #803dc2 and the transparency ByBlock, as ACadSharp reads
  them; 14 entities of that sample changed.
- **What was changed:** the RGB first, then the transparency.
- **Commit:** [`1c4dcc35`](https://github.com/monyuonyu/libredwg-fixes/commit/1c4dcc35a975e52d734e37a662c5c4222335bd1c) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442).

### 2.2 R2007+: the color book handle was read twice and the layer was lost

- **What was wrong:** for R2007+ the DBCOLOR handle of an entity color with flag 0x40 is read by
  `dwg_decode_common_entity_handle_data` ("deferred from common_entity_data"). But
  `common_entity_data.spec` read it too, from the same handle stream, so the deferred read
  consumed the next handle, the layer.
- **How it showed:** an entity with a color book color ended up on layer NULL (shown as [0, 0]).
- **Drawings:** a CIRCLE in `sample_AC1021` … `sample_AC1032`, handle 99F (layer 16 expected).
- **What was changed:** skip the early read when decoding R2007+.
- **Commit:** [`f18f7216`](https://github.com/monyuonyu/libredwg-fixes/commit/f18f721636605498a7afa29410f9dcbd83b27e00) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442).

### 2.3 Colors written to R2000: ByBlock, AutoCAD's ACI fallback, the ACI palette

- **What was wrong:** `bit_downconvert_CMC` treated the R2004+ entity colors (ENC) like the other
  colors with a method byte. An ENC has no method; its index says ByBlock (0) or ByLayer (256). A
  true color was reduced to the low byte of its RGB as the index. And the ACI palette `rgb_palette` differed
  from AutoCAD's in most entries (color 8 was 65,65,65).
- **How it showed:** every ByBlock entity became ByLayer in R2000 output, and true colors got an
  ACI index unrelated to their color.
- **Drawings:** AutoCAD's own R2000 saves of the same drawings; AutoCAD writes the ACI index stored
  with a true color (#803dc2 with 183).
- **What was changed:** keep ByBlock and ByLayer; a true color keeps the ACI index stored with it
  (AutoCAD's fallback), else the nearest ACI (`dwg_nearest_color_index`); the palette as AutoCAD
  shows it (the values of ezdxf; ACadSharp agrees in 251 of 256). `example_2018` written as R2000
  is now equal to AutoCAD's R2000 save in all 222 entities.
- **Commit:** [`ecad64a8`](https://github.com/monyuonyu/libredwg-fixes/commit/ecad64a88311475a4a1c5234d3c7dc56c948efba) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 2.4 DXF: true colors (420–429) as 24-bit RGB

- **What was wrong:** DXF group 420 is the plain RGB (AutoCAD writes 3355443). `out_dxf` wrote the
  DWG value with the 0xC2 method byte (3258135347), with `%li`.
- **How it showed:** the 420 value was not the RGB; on Windows (32-bit `long`) it was even
  negative, so Linux and Windows `dwg2dxf` wrote different DXF.
- **What was changed:** the low 24 bits, written with int32 formats. Linux and Windows now write
  the same DXF.
- **Commit:** [`8f1650ad`](https://github.com/monyuonyu/libredwg-fixes/commit/8f1650ad30a8c7364b590c339e12dbec91d479ba) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 2.5 DXF: the DIMSTYLE colors 176–178 are color indexes

- **What was wrong:** `dxf_CMC` wrote every color with a DXF code of 90 or more as the int32 RGB,
  which is meant for 90–99 only.
- **How it showed:** DIMCLRD, DIMCLRE and DIMCLRT of R2004+ drawings were written as
  `176 -1037268965` and read back as other colors, and a true color was written as a 534 group.
- **What was changed:** the RGB only for 90–99, the 420 true color only for 62–67; otherwise the
  color index, as AutoCAD.
- **Commit:** [`96b39361`](https://github.com/monyuonyu/libredwg-fixes/commit/96b393619e800e438e465e3687973ed9a5b21d28) — not yet offered upstream.

### 2.6 R13–R2000 to R2004+: the true colors AutoCAD keeps in EED

- **What was wrong:** when AutoCAD saves a true color entity as R2000 or older, it keeps the true
  color in EED of the APPID ACAD ("RTTcAl", 70 flags and index, 71 the RGB, the color name) and
  restores it when it reads the drawing. LibreDWG ignored it.
- **How it showed:** `sample_AC1015` written as R2004 had 21 entities with other colors than
  AutoCAD's own `sample_AC1018`.
- **What was changed:** `upconvert_true_colors` restores these colors when writing R2004+ from
  before R2004.
- **Commit:** [`ce5e98f8`](https://github.com/monyuonyu/libredwg-fixes/commit/ce5e98f882ec54d26185959a34a031ea1f96f04a) — not yet offered upstream.

TABLESTYLE colors in DXF are in [6.3](#63-tablestyle-all-r2010-cell-styles-r2007-row-styles).

## 3. Text and codepages

### 3.1 Asian codepages: the lead byte of two-byte characters

- **What was wrong:** `bit_utf8_to_TV` wrote the lead byte of a two-byte character as
  `wc1 >> 4` instead of `wc1 >> 8`: 0xB8F4 (GBK for U+9694) became 8F F4. In the 3-byte UTF-8 branch
  single-byte characters of the codepage (CP932 half-width katakana) got a 0x00 lead byte.
- **How it showed:** every Chinese, Japanese or Korean string converted from UTF-8 to such a
  codepage (for example by `dwg_add_u8_input`, JSON import, or DXF to R2000) was garbled, and a half-width katakana
  character cut the string. The unit test expected the buggy bytes for 시험 in CP949 (DC C3 7E E8);
  the right bytes are BD C3 C7 E8, as Python and Windows convert it.
- **What was changed:** `>> 8`, and single bytes without a lead byte; the test expects CP949's real
  bytes (the PR also adds ANSI_932 tests).
- **Commits:** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504) (part), [`45e6b50f`](https://github.com/monyuonyu/libredwg-fixes/commit/45e6b50f615aa7346827f71a26de67dc3a8f0eae) — PR
  [#1440](https://github.com/LibreDWG/libredwg/pull/1440).

### 3.2 R2007+ DXF to R2000: convert the UTF-8 text to the drawing codepage

- **What was wrong:** R2007+ DXF is UTF-8, but `dxf_read_pair` and dynapi copied TV strings as they
  are, so an R2000 DWG got raw UTF-8 bytes.
- **How it showed:** garbled text in R2000 output of an R2007+ DXF.
- **Drawing:** `gh109_1` (codepage ANSI_936).
- **What was changed:** for R2000 output convert to the drawing codepage, with `\U+XXXX` for
  characters it does not have, as AutoCAD; the same for MTEXT text chunks (only for R2007+ targets).
- **Commit:** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504) (part) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 3.3 DXF to R2007+: names cut to their first character

- **What was wrong:** the R2007+ encoder takes the strings of a DXF as UTF-8 (`bit_write_T`), but
  `in_dxf` and dynapi made UTF-16 (TU) strings of most of them when the target version was R2007+.
  Written as UTF-8 again, they were cut at the first 0 byte. `add_HATCH` and `add_MULTILEADER`
  still did this for the gradient color names and the MULTILEADER default text after the main fix.
- **How it showed:** names were cut to their first character (the block ALIGNMENT_PARAMETER as
  "A", the APPIDs of the EED), and ACadSharp lost the entities in these blocks. DXF to
  R2010–R2018 lost 11,007 of 59,883 entities, with 47,757 changed values.
- **What was changed:** `dxf_read_file` reads the DXF as for R2004 (so the strings stay UTF-8) and
  then sets the R2007+ version again; HATCH and MULTILEADER decide by the version of the DWG.
- **Commits:** [`f8cb428f`](https://github.com/monyuonyu/libredwg-fixes/commit/f8cb428f7ae7850f9bd9d3810264aea75fb00445), [`061d4164`](https://github.com/monyuonyu/libredwg-fixes/commit/061d41647cdfdbaf000d2dd4d04c3a9aa6f79002) — not yet offered upstream.

UTF-16 EED strings written to R2000 are in [4.3](#43-r2007-eed-strings-written-to-r2000); the
string encodings of R2004+ files are in [13.10](#1310-strings-of-r2004-files-t16-tu-and-the-string-stream).

## 4. Extended entity data (EED)

### 4.1 EED layer (1003): a big-endian handle, a name in DXF

- **What was wrong:** EED code 3 (DXF 1003) is the LAYER handle, stored big-endian like the entity
  handle (code 5). It was read as little-endian. In DXF, 1003 is the layer *name*, but `out_dxf`
  wrote the handle number, and `in_dxf` did not implement 1003 at all.
- **How it showed:** AutoCAD's `sample_AC1015` has `00..00 08 F5` for the LAYER app_layer 8F5; it
  was read as 0xF508000000000000.
- **What was changed:** big-endian in the decoder and encoder; DXF writes the layer name and the
  import resolves the name to the layer.
- **Commit:** [`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8) (part) — big-endian in PR
  [#1442](https://github.com/LibreDWG/libredwg/pull/1442), the DXF name in PR
  [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 4.2 DXF EED binary (1004) decoded twice

- **What was wrong:** `dxf_read_binary` already turns the hex of 1004 into bytes; `add_eed` decoded
  it again.
- **How it showed:** "Invalid hex string member", and the EED binary data of every object of an
  ASCII DXF was lost.
- **What was changed:** decode once.
- **Commit:** [`3d495876`](https://github.com/monyuonyu/libredwg-fixes/commit/3d49587605124359218d042037bd1fa2e5fdd80d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 4.3 R2007+ EED strings written to R2000

- **What was wrong:** the R13–R2007 branch for UTF-16 EED strings wrote the codepage with
  `bit_write_RS` (ANSI_1252, 30, became 0x1E00; it is big-endian), and the length was the number of
  UTF-16 characters, not of the converted string (characters above U+00FF become `\U+XXXX`). The
  buffer check was inverted and underflowed.
- **How it showed:** ACadSharp could not read the STYLE of an R2013 drawing written as R2000 (the
  font name is in the EED).
- **What was changed:** codepage big-endian, the converted length, a correct buffer check.
- **Commit:** [`438e9817`](https://github.com/monyuonyu/libredwg-fixes/commit/438e98179131cba1be9732e42cec26cd4b8faa54) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

## 5. MTEXT, ATTRIB and ATTDEF

### 5.1 R2018 MTEXT: background fields also with only the frame bit

- **What was wrong:** the background fields (scale, color, transparency) are also present when only
  bit 0x10 (text frame, R2018+) of `bg_fill_flag` is set. The old test
  `version <= R_2018 ? 1 : 16` skipped them for R2018 files with a frame but no fill.
- **How it showed:** every following field of such an MTEXT was misread.
- **What was changed:** check bit 0x01, or bit 0x10 since R2018, as ACadSharp does.
- **Commit:** [`3ab77f4d`](https://github.com/monyuonyu/libredwg-fixes/commit/3ab77f4d123107fd17bddd725865dd1e1dad9039) (part) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442).

### 5.2 R2018 multi-line ATTRIB/ATTDEF: the embedded MTEXT

- **What was wrong:** for `mtext_type` 2 or 4 "all fields of an embedded MTEXT object are written,
  starting from the Entmode" (ODA 20.4.4). LibreDWG's layout of that embedded part did not match
  AutoCAD's files. Measured on AutoCAD's R2018 sample and as ACadSharp reads it: the common entity
  part has the owner handle, reactors, xdic, `has_ds_data`, a full ENC color (RGB before the
  transparency) and the flags in the order ltype, material, shadow, plotstyle, with their handles;
  the MTEXT has `extents_height` before `extents_width`, the background fields with 0x10 too, and
  the R2018 annotation and column fields; the STYLE handle of the ATTRIB/ATTDEF comes before the
  embedded MTEXT handles; after the MTEXT come `annotative_data_size` and, if not 0, the data, the
  registered application and a BS.
- **How it showed:** the MTEXT layer pointed to the STYLE Standard and the ATTRIB layer was wrong.
- **Drawing:** `sample_AC1032` (ATTDEF 796, ATTRIB 79D). Now the data ends exactly at the string
  stream and all handles point to LAYER 0, LTYPE ByLayer, MATERIAL ByLayer and STYLE Standard. Value
  differences against ACadSharp on the 22 ACadSharp samples: 10 → 0.
- **What was changed:** the layout above, new struct fields for the handles; in JSON the fields are
  written as `mtext.<field>` (before, flat keys like `ins_pt` clashed with the ATTRIB's).
  GEOPOSITIONMARKER uses the same macro and keeps its former trailing fields; it is unverified, there
  is no sample.
- **Commit:** [`a1c9eba4`](https://github.com/monyuonyu/libredwg-fixes/commit/a1c9eba4034cd71f53533a1e134c43337c5d15aa) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443).

### 5.3 R2018 multi-line attributes written to earlier versions

- **What was wrong:** there are no multi-line attributes before R2018. AutoCAD's R2010 and R2013
  saves have the attribute text in `text_value`; R2018 has an empty `text_value` and the text only
  in the embedded MTEXT. Writing an earlier version kept the empty `text_value`.
- **How it showed:** multi-line attributes lost their text in R2013 and older output.
- **Drawings:** `sample_AC1024`, `sample_AC1027` against `sample_AC1032`.
- **What was changed:** with an empty `text_value` (`default_value` for ATTDEF), use the MTEXT text
  with the formatting codes stripped. Not reproduced: which codes AutoCAD strips, what `\P`
  becomes (here a space), and AutoCAD's xdictionary round-trip data.
- **Commit:** [`41f07038`](https://github.com/monyuonyu/libredwg-fixes/commit/41f0703840f8d81ee349f58e389199f1c73e023f) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443).

### 5.4 R2010+ ATTRIB: no `keep_duplicate_records`

- **What was wrong:** in AutoCAD's R2010, R2013 and R2018 files the data stream of an ATTRIB ends
  right after `lock_position_flag`. Only ATTDEF has the repeated version RC (here
  `keep_duplicate_records`) before the prompt; ODA 20.4.4 and ACadSharp agree.
- **How it showed:** "Invalid ATTRIB.keep_duplicate_records 64/65/72" on reading, and an extra byte
  on writing.
- **Drawings:** `sample_AC1024`, `sample_AC1027`, `sample_AC1032`.
- **What was changed:** not in the DWG since R2010; kept for DXF and JSON.
- **Commit:** [`f3432746`](https://github.com/monyuonyu/libredwg-fixes/commit/f343274668fd1e997a47a4f8b83832e22ec26813) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443).

### 5.5 DXF import: the embedded MTEXT of ATTRIB/ATTDEF, the MTEXT columns part

- **What was wrong:** `in_dxf` did not read the embedded MTEXT of R2018 multi-line ATTRIB/ATTDEF
  after `<101, Embedded Object>`, nor the annotation/column part of MTEXT, whose 40 (rectangle
  width) overwrote the text height.
- **How it showed:** `dxf2dwg` failed on `sample_AC1032` ("Invalid DXF code 46"); 36 MTEXT got the
  height 0.
- **What was changed:** read both parts into their own fields.
- **Commit:** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504) (part) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 5.6 R13/R14 MTEXT written as R2000+: line spacing 1, not 0

- **What was wrong:** R13/R14 MTEXT has no line spacing fields; writing R2000+ wrote them as 0.
- **How it showed:** the lines of every MTEXT collapsed onto each other (55 MTEXT in `sample_AC1014`
  and `example_r14` written as R2000).
- **What was changed:** the defaults, line spacing style 1 and factor 1.0.
- **Commit:** [`6b26f48d`](https://github.com/monyuonyu/libredwg-fixes/commit/6b26f48deb27612c4013109522f92cc67e41ce9c) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 5.7 MTEXT line breaks as `\P`

- **What was wrong:** MTEXT of newer drawings can contain plain LF line breaks. AutoCAD writes them
  as `\P` in R2000 DWG and in DXF; LibreDWG wrote DXF with `^J`.
- **How it showed:** DXF readers kept `^J` as text.
- **Drawings:** `sample_AC1018` … `sample_AC1032`, against AutoCAD's R2000 save.
- **What was changed:** `dwg_fixup_mtext_linebreaks` turns LF into `\P` for R2000 DWG and DXF.
- **Commit:** [`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40) (part) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 5.8 DXF export: ATTRIB and VERTEX after a SEQEND

- **What was wrong:** for R13–R2000 `out_dxf` found the attributes of an INSERT and the vertices of
  a POLYLINE as "the next object". But a SEQEND can lie between the attributes.
- **How it showed:** the DXF lost every attribute after the SEQEND.
- **Drawing:** `sample_AC1015` (ATTRIB 79D, SEQEND 79E, ATTRIB 79F, 7A0).
- **What was changed:** follow the entity links (next entity handles).
- **Commit:** [`77fec52b`](https://github.com/monyuonyu/libredwg-fixes/commit/77fec52b4abc92612188613f9cb7a8192bfb6957) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 5.9 Attribute tags: any character but space and `!`

- **What was wrong:** `dwg_is_valid_tag` accepted only letters, digits, `$`, `_` and `-`; AutoCAD
  accepts any character except space and `!`. The ASCII-only variant accepted the empty tag.
- **How it showed:** the ATTDEF tag `SPACEOBJECTS:NUMBER` was rejected and lost via DXF.
- **Drawing:** `aec_objects`.
- **What was changed:** AutoCAD's rule; the empty tag is invalid.
- **Commit:** [`f9b9f144`](https://github.com/monyuonyu/libredwg-fixes/commit/f9b9f14442df3966abda9e1d856c61745a1679b4) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

## 6. TABLE and TABLESTYLE

### 6.1 TABLE/TABLECONTENT: the layout, readable in normal builds

- **What was wrong:** TABLE and TABLECONTENT were only built in debugging builds, and their layout
  was wrong in many places. Checked against where the data of each table must end (R2000/R2004 at
  the handle stream, R2007+ at bitsize − 17 − string data size):
  - TABLE derives from INSERT in all versions; the R2010+ variant also starts with the insert data
    and handles, then RC, H, BL, B (R2010) or BL (R2013+), the table content, the table style
    handle, BS, 3BD and the break data (ODA 20.4.96.2, ACadSharp `readTableEntity`).
  - TABLECONTENT (20.4.97): columns have a custom data collection, the cell style id and a BD
    width; cells have their own cell style overrides; row height is BD; the table cell style
    overrides come before the merged ranges; one cell geometry if the geometry flags are non-zero;
    the table style handle at the end.
  - Pre-R2010 cells: the text string only before R2007 and only if the 344 handle is 0; block cells
    have a has-attdefs bit and a BS count; cell alignment is BS, not RS; 0x10 is its own text style
    handle; the R2007 BL + value for every cell; no extra handles after table override bits 0x2000
    and 0x40000.
  - Values (20.4.99): empty only with flag bit 0x01 (not 0x03); since R2007 a string value is a BL
    byte count plus UTF-16LE bytes, not a TV.
  - The generic REPEAT size check rejected the pre-R2010 cells.
  - `strrplc` asserted on the long nested field names in the `-v3` trace.
- **How it showed:** tables were not read at all in normal builds; in debugging builds the data
  ran past its end.
- **Drawings:** `sample_AC1014` … `sample_AC1032`, `sample_base`, `example_r14` … `example_2018`,
  `2000/TS1`.
- **What was changed:** the layout above; TABLE and TABLECONTENT moved from DEBUGGING to UNSTABLE,
  so they are read in normal builds. Handles point to the expected objects.
- **Commit:** [`1d92022d`](https://github.com/monyuonyu/libredwg-fixes/commit/1d92022d6a0c4af2c927794e2689f512cf107894) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446).

### 6.2 TABLECONTENT: kGeneral values before R2007, the count checks

- **What was wrong:** kGeneral (512) values also occur in R14–R2004 TABLECONTENT objects (AutoCAD
  2008 round-trip data), as a BL byte count + bytes like since R2007; the reader stopped there. The
  REPEAT size checks compared `count * sizeof(struct)` with the available bits.
- **How it showed:** "kGeneral <r2007", and the last rows of valid tables were rejected.
- **What was changed:** read kGeneral in all versions; check with a minimal bit size per element.
  All 24 tables and 9 TABLECONTENT objects of the corpus end where they should, and all 24 tables
  match ACadSharp field by field.
- **Commit:** [`b9780ac7`](https://github.com/monyuonyu/libredwg-fixes/commit/b9780ac7a0b3b1c995fbc37f17a3d798901db081) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446).

### 6.3 TABLESTYLE: all R2010+ cell styles, R2007 row styles

- **What was wrong:** R2010+: after the table cell style come the cell styles _TITLE, _HEADER and
  _DATA as an array, each after a BL index (as ACadSharp `readTableStyle`); only the first was read.
  R2007: the row styles have the value data type, unit type and format string after the borders
  (20.4.101.1), which were not read; `format_string` is T, not TU. Writing R2000, the encoder
  converted the table style after the margins were written, so they were 0. DXF import took TABLESTYLE colors as palette RGB and border
  lineweights as entity lineweight indexes.
- **How it showed:** `example_2018` TABLESTYLE 0x29B ended at bit 310.4 with 6688 unknown bits; rows 1
  and 2 of `example_2007` and `sample_AC1021` were garbage.
- **What was changed:** read the whole array; `downconvert_TABLESTYLE` rewritten (the data, title
  and header row styles from _DATA, _TITLE and _HEADER, margins and flow direction from the table
  cell style, a "none" background as ACI 7 without background, the border invisibility inverted);
  DXF colors as table colors (ACI 7 is 0xc3000007), lineweights as given, 420–427 as 24-bit RGB;
  in JSON `ovr` is an array. The DXF TABLESTYLE of `example_2000` … `example_2018` now equals
  AutoCAD's DXF, and the R2000 output equals AutoCAD's R2000 save.
- **Commit:** [`6a3f2d0c`](https://github.com/monyuonyu/libredwg-fixes/commit/6a3f2d0cbab6e9a5748e030d64824362c17abe14) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446).

### 6.4 R2010+ tables written to R2000/R2004 in the old format

- **What was wrong:** since R2010 a table keeps its cells in the table content (20.4.97); R2000 and
  R2004 need the old cell data (20.4.96.1). Nothing converted it.
- **How it showed:** tables of R2010+ drawings were lost in R2000 output. AutoCAD keeps ACAD_TABLE
  in its R2000 files (`sample_AC1015`, `example_2000`).
- **What was changed:** `downconvert_TABLE` builds the old data: rows, columns, widths, heights,
  merged ranges, cell texts (the formatted value 302, else the string), block cells, and the text
  height, alignment and color overrides per content. Borders, row/column/table cell styles and
  fields are not converted. `flag_for_table_value` is taken from an R2010+ unknown BS with the same
  values; that it is the same field is a guess. The R2000 output of `sample_AC1024/1027/1032`,
  `sample_base` and `example_2010/2013/2018` has the same 11 tables as AutoCAD's R2000 files.
- **Commit:** [`7139a423`](https://github.com/monyuonyu/libredwg-fixes/commit/7139a423964adff8eef80da0cb1d6daa192e07c0) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446).

### 6.5 R2007 tables written to R2000/R2004: the cell text

- **What was wrong:** R2007 keeps the cell text in the value (UTF-16), with `text_value` empty.
  AutoCAD's R2000 save writes the formatted value text (302) first.
- **How it showed:** written as R2000 every cell lost its text; numbers lost their format.
- **Drawings:** `sample_AC1021` against AutoCAD's `sample_AC1015`.
- **What was changed:** the cell text from the value; the formatted text first ("0.500000",
  "$100.00", "CELL FORMAT 1"). The tables now equal AutoCAD's R2000 save.
- **Commits:** [`4f819653`](https://github.com/monyuonyu/libredwg-fixes/commit/4f8196538d897b971893cd585d38a1d47e2371a7), [`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb) (part) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446).

### 6.6 DXF import and export of ACAD_TABLE

- **What was wrong:** tables were not written to DXF and could not be imported. When TABLE became
  UNSTABLE ([`d1f8fb4c`](https://github.com/monyuonyu/libredwg-fixes/commit/d1f8fb4c7e729e577e4d005e699a495f490f01a8)), `in_dxf` failed on the unknown ACAD_TABLE groups and `out_dxfb` wrote
  tables with the DWG fields; that commit first kept DXF/DXB tables disabled.
- **How it showed:** the DXF way (`dwg2dxf`, `dxf2dwg`) lost every table (18 in the corpus).
- **What was changed:** AutoCAD writes ACAD_TABLE to every DXF version in the pre-R2010 form (the
  INSERT part; 342 343 11 90 91 92 93–96; row heights 141 before column widths 142; per cell 171–176,
  177, 178, 145, then the text 1 before R2007 or the value since R2007; R2010+ adds 280 0).
  `dxf_ACAD_TABLE` writes this (R2010+ tables converted as for R2000), `add_ACAD_TABLE` reads it.
  The output for `example_2000/2007/2018` is AutoCAD's DXF except handles, the anonymous block name
  and, for R2018, flag 90 and the 0x40000 cell flag bit. Cell and table overrides follow the DWG
  field order; no AutoCAD sample has any, so their DXF order is a guess. Then the writer moved to
  `out_dxf_table.inc` so binary DXF (`dwg2dxf -b`) writes tables too.
- **Commits:** [`d1f8fb4c`](https://github.com/monyuonyu/libredwg-fixes/commit/d1f8fb4c7e729e577e4d005e699a495f490f01a8), [`56cd3c4f`](https://github.com/monyuonyu/libredwg-fixes/commit/56cd3c4fe139abf172696e8ca31093db1936b3df), [`0f3a1ac7`](https://github.com/monyuonyu/libredwg-fixes/commit/0f3a1ac7e14820b28ecaf16841ad588b439bc51f) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446).

### 6.7 DXF TABLESTYLE in AutoCAD's layout

- **What was wrong:** LibreDWG wrote the R2010+ cell style layout (TABLEFORMAT_BEGIN …) to DXF.
  AutoCAD writes the R2007 layout in all versions: the three row styles (7 140 170 62 63 283
  [90 91 1] and 6 borders), R2010+ with a leading 280 0. R13/R14 DXF dropped TABLESTYLE.
- **How it showed:** ACadSharp could not read the TABLESTYLE, and tables lost their table style via
  DXF; `sample_AC1014` has tables and their TABLESTYLE, and they lost the style name STANDARD.
- **What was changed:** the R2007 layout for all versions; TABLESTYLE in R13/R14 DXF when the
  drawing has it.
- **Commits:** [`61f6d7ec`](https://github.com/monyuonyu/libredwg-fixes/commit/61f6d7ecaad2cda7d54a5857fa2aceaac79c3a33), [`e1835040`](https://github.com/monyuonyu/libredwg-fixes/commit/e18350406d9dd7784bf7c547bcd9ad9786b164ac) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446).

### 6.8 Tables written to R2010+ from earlier versions and from DXF

- **What was wrong:** the R2010+ TABLE needs the table content; the cells of R2000–R2007 tables, and
  of tables read from any DXF (`in_dxf` reads the pre-R2010 form), were not converted. R2007 cell
  values from DXF have no `data_size`.
- **How it showed:** the 13 tables of the R14–R2007 drawings had no rows, columns and cells as
  R2010+, and the horizontal direction was 0; the tables of R2010+ DXFs written as R2010+ were empty
  (11 of the corpus); ACadSharp rejected a 3D point value of size 0 ("Could not read ACAD_TABLE",
  the table of `sample_AC1021` via DXF).
- **What was changed:** `upconvert_TABLE`, the reverse of `downconvert_TABLE` (columns, rows, one
  content per cell, overrides, merged ranges, horizontal direction, one break row range), also for
  DXF input; the size of 2D/3D point values set, a date without data written as its text.
- **Commits:** [`7fb484ed`](https://github.com/monyuonyu/libredwg-fixes/commit/7fb484ed0d13655ed22dce0ab8678c170339a142), [`03c45bf5`](https://github.com/monyuonyu/libredwg-fixes/commit/03c45bf52eff38e84da496b20f4bc508e034dd95), [`3ec9d114`](https://github.com/monyuonyu/libredwg-fixes/commit/3ec9d1148f71d8a2a0fd07fbe044f33ef17d39c5) — not yet offered upstream.

### 6.9 Memory leaks when writing R2007+ from earlier versions

- **What was wrong:** the `ovr` array made by `upconvert_TABLESTYLE`, which the free spec of the
  source version does not know, and the UTF-16 name of `add_DUMMY_eed` were never freed.
- **How it showed:** LeakSanitizer on `example_2000` written as R2018.
- **What was changed:** free both.
- **Commit:** [`10a1a752`](https://github.com/monyuonyu/libredwg-fixes/commit/10a1a752e71debbe348e500c2da9daced8b1dec1) — not yet offered upstream.

## 7. ACIS solids (3DSOLID, REGION, BODY)

### 7.1 Reading: no ACIS empty bit since R2013, the R2007+ unknown BL

- **What was wrong:** since R2013 the SAB lives in the AcDsPrototype_1b section and the entity
  starts with the wireframe data; LibreDWG read an ACIS empty bit first, shifting everything by one
  bit. And ODA 20.4.41 lists "R2007+: Unknown BL" after the empty bit, which was not read.
- **How it showed:** AutoCAD's R2018 3DSOLID D6A had 1274288069 isolines; it now has the point
  (31.69, −220.11, 2.19) and 4 isolines, as ACadSharp reads it. The garbage 3DSOLID also broke
  R2000 output: ACadSharp lost the 9 entities after it in model space.
- **Drawing:** `sample_AC1032`.
- **What was changed:** no empty bit since R2013; the new field `unknown_2007`. All 107 ACIS
  entities of the 149 drawings now end exactly at their bitsize (100 before). A test compared the
  revision GUID arrays by address, which never matches; it was never reached before because the
  GUID was misread. It now compares the contents.
- **Commits:** [`faa9c595`](https://github.com/monyuonyu/libredwg-fixes/commit/faa9c59576683f2931117b4cd6edb54b4adc7ca2), [`aad393ec`](https://github.com/monyuonyu/libredwg-fixes/commit/aad393ec206b1116694db8ae7877874207530709) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442).

### 7.2 Writing: the empty bit, the history handle, materials

- **What was wrong:**
  - The encoder still wrote the ACIS empty bit for R2013+ ([`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)).
  - The history handle (H 350) of R2013+ 3DSOLIDs was neither read nor written (26 bits left in the
    handle stream of AutoCAD's R2018 3DSOLIDs) ([`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)).
  - It was written into R2004 and R2000 DWGs too, as an extra null handle; AutoCAD's R2004 drawings
    have none ([`81d14b4c`](https://github.com/monyuonyu/libredwg-fixes/commit/81d14b4c8e628be99b3deabcce0ef3e773c9b580)).
  - For R2007+ it was written only with SAB data, so SAT 3DSOLIDs of R14–R2004 drawings lacked it
    ([`d7da6276`](https://github.com/monyuonyu/libredwg-fixes/commit/d7da62767fb3af37edb5cce653ac7115e36917df)).
  - AutoCAD's R2007–R2010 3DSOLIDs and REGIONs have only the unknown BL after the empty bit:
    `num_materials` read that BL and the unknown BL then read 34 bits of the handle stream; and the
    history handle is only in 3DSOLIDs, not REGIONs ([`d59ae195`](https://github.com/monyuonyu/libredwg-fixes/commit/d59ae195c6d89b813ae3a90e1a14b9ceb00dbb76)).
- **How it showed:** ACadSharp could not read the 3DSOLIDs and REGIONs of R2013+ drawings rewritten
  as R2013+ (5 3DSOLIDs and 4 REGIONs lost); the SAT 3DSOLIDs as R2010 made ACadSharp read past the
  handles ("Index was outside the bounds of the array"); the 3DSOLID of `example_2010` was 34 bits
  too long.
- **Drawings:** `example_2004`, `sample_AC1018`, `example_2010`, `example_2013`, `example_2018`.
- **What was changed:** as AutoCAD: no empty bit since R2013; the history handle for all R2007+
  3DSOLIDs only; no materials in the DWG. The 3DSOLIDs and REGIONs of `example_2010`,
  `example_2013` and `example_2018` rewritten as the same version are now AutoCAD's bytes.
- **Commits:** [`81d14b4c`](https://github.com/monyuonyu/libredwg-fixes/commit/81d14b4c8e628be99b3deabcce0ef3e773c9b580), [`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f), [`d7da6276`](https://github.com/monyuonyu/libredwg-fixes/commit/d7da62767fb3af37edb5cce653ac7115e36917df), [`d59ae195`](https://github.com/monyuonyu/libredwg-fixes/commit/d59ae195c6d89b813ae3a90e1a14b9ceb00dbb76) — not yet offered upstream.

### 7.3 The R2013+ AcDs section: reading the SAB by handle

- **What was wrong:** the SAB of R2013+ REGIONs, 3DSOLIDs, BODYs and SURFACEs is in the data
  records of the AcDb:AcDsPrototype_1b section. LibreDWG searched for "ACIS BinaryFile" and gave
  the data to the entities with `has_ds_data` in their order. AutoCAD's R2018 files have "ASM
  BinaryFile4" instead, and the records are not in the order of the entities. The segment index
  check used the struct size 16, not the 12 bytes of an entry.
- **How it showed:** none of the ACIS data of AutoCAD's R2018 files was read (`example_2018`,
  `sample_AC1032`); the SAB could go to the wrong entity (`example_2013`); the AcDs of
  `sample_base` was rejected.
- **What was changed:** take the data records of the data index with their handles, also from blob
  pages.
- **Commit:** [`eb3b2553`](https://github.com/monyuonyu/libredwg-fixes/commit/eb3b25530662bec56bebb0d045d614aff48be9be) — not yet offered upstream.

### 7.4 The R2013+ AcDs section: writing it

- **What was wrong:** the AcDs section could not be encoded (`acds.spec` was disabled).
- **How it showed:** R2013+ drawings rewritten as R2013+ lost all their ACIS data; R2004–R2010
  drawings written as R2013+ too.
- **What was changed:** first, keep the decompressed section and write it as is from R2013+ to
  R2013+ ([`60d4a3e0`](https://github.com/monyuonyu/libredwg-fixes/commit/60d4a3e07f870ea31c94f4990f7b60c0ab51c141)). Then write the section as AutoCAD's `example_2018` has it: the SAB records
  by handle (in blob01 pages over 0x40000 bytes, as the ODA spec), the same schemas and the search
  index, and mark the entities with `has_ds_data` ([`b06ca869`](https://github.com/monyuonyu/libredwg-fixes/commit/b06ca8699845d096d972be21469c42f3ca787a68)).
- **Commits:** [`60d4a3e0`](https://github.com/monyuonyu/libredwg-fixes/commit/60d4a3e07f870ea31c94f4990f7b60c0ab51c141), [`b06ca869`](https://github.com/monyuonyu/libredwg-fixes/commit/b06ca8699845d096d972be21469c42f3ca787a68) — not yet offered upstream.

### 7.5 SAT to SAB and back

- **What was wrong:** R13–R2000 DWGs and DXFs have the ACIS data as SAT text, but the R2013+ AcDs
  has only SAB records. And the old SAB to SAT conversion for DXF (`dwg_convert_SAB_to_SAT1`) wrote
  doubles with `%g` (6 digits), longs as pointers `$n`, wrong logicals (F instead of forward_v and
  I), header strings broken into pieces ("A utodesk A utoCA D"), a cut end marker, a changed
  version, no history attributes, and nothing for "ASM BinaryFile4".
- **How it showed:** solids of these drawings written as R2013+ were empty; the SAT in DXF was not
  readable and the geometry changed.
- **What was changed:** a grammar of the ACIS records gives the value types the SAT lacks (a long
  and an integral double look the same, logicals are words), taken from AutoCAD's SAT and SAB of the
  same solids. `dwg_convert_SAT_to_SAB` and `dwg_convert_SAB_to_SAT` use it, with the shortest exact
  doubles. All 15 SAT bodies of the corpus convert to SAB and back to the same SAT, all 83 SAB bodies
  to SAT and back to the same bytes.
- **Commits:** [`d207b237`](https://github.com/monyuonyu/libredwg-fixes/commit/d207b2379e50b0163ba401a5639e2fc58c1a0871), [`2ab21c75`](https://github.com/monyuonyu/libredwg-fixes/commit/2ab21c7563a9f8d75596f55ab4ea1906f0a9050e) — not yet offered upstream.

### 7.6 DXF: SAT lines and the escaped `^`

- **What was wrong:** the SAT is written to DXF in group 1 lines, and a line over 255 characters
  continues in group 3. LibreDWG wrote the encrypted 4096-byte blocks of the DWG line by line, but
  the blocks split lines anywhere. Reading, the newline of a 1 line was put before its 3
  continuations. The encrypted `A` is `^`, which DXF writes as `^ `; on import it was unescaped in
  `acis_data` but not in `encr_sat_data`, which is what R13–R2010 DWGs get.
- **How it showed:** a line at the end of a block became two lines and the next character was
  dropped, so numbers broke (−0.8944579735636958 became −0.89 and 44579735636958 in
  `example_2000`); the SAT of DWGs made from DXF had "A " for each A and lost its end.
- **What was changed:** `dxf_sat_lines` writes the lines of all blocks, over 255 in 3, with `^ `
  for `^`; `add_3DSOLID_encr` puts the newline right and unescapes both copies.
- **Commits:** [`94a6cf90`](https://github.com/monyuonyu/libredwg-fixes/commit/94a6cf906aa7bd5749254bae8c849216c4627819), [`bff66044`](https://github.com/monyuonyu/libredwg-fixes/commit/bff660446e6496299f05b67bd6fd353645799e4e) — not yet offered upstream.

### 7.7 DXF import: the revision GUID field names

- **What was wrong:** the revision GUID was set as `revision_guid[39]` and `revision_bytes[9]`,
  names dynapi does not know.
- **How it showed:** "Invalid 3DSOLID field".
- **What was changed:** the real field names.
- **Commit:** [`da463d6d`](https://github.com/monyuonyu/libredwg-fixes/commit/da463d6d34deddbfcb3c107a1582bed3d65a4384) (part) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

## 8. Proxies and unknown objects

### 8.1 Unknown objects keep their class name

- **What was wrong:** entities and objects LibreDWG cannot decode were all named UNKNOWN_ENT /
  UNKNOWN_OBJ.
- **How it showed:** JSON said only UNKNOWN_ENT, not what the object is.
- **What was changed:** their class name as dxfname (AEC_WALL, TCH_WALL, ...), as other readers do.
- **Commit:** [`1935be09`](https://github.com/monyuonyu/libredwg-fixes/commit/1935be09e0ad6e1cac637427804eb02741afeada) (part) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447).

### 8.2 The ACDBPLACEHOLDER class when needed

- **What was wrong:** unsupported objects are written as placeholders, but the DXF of R2007+
  drawings has no ACDBPLACEHOLDER class.
- **How it showed:** every unsupported object (TABLEGEOMETRY, MATERIAL to R2000) became a DUMMY,
  with an ERROR.
- **What was changed:** the encoder adds the class when needed.
- **Commit:** [`ebeebb13`](https://github.com/monyuonyu/libredwg-fixes/commit/ebeebb139ac7e446df77e17486c59c6baa3f693d) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447).

### 8.3 Unknown objects written as ACAD_PROXY_ENTITY / ACAD_PROXY_OBJECT

- **What was wrong:** writing R2000 from an R2004+ DWG, unknown entities and objects (AEC_*, TCH_*,
  ...) became a POINT with thickness 1e25 or a PLACEHOLDER. Also the proxy spec read and wrote a
  second copy of the graphics after the drawing format (ODA and ACadSharp have none), did not decode
  all object ids, and `bit_write_bits` wrote the trailing bits in the wrong order.
- **How it showed:** custom objects of other applications lost their data.
- **What was changed:** write them as proxies, as AutoCAD (ODA 20.4.90/29): the class number, the
  drawing format of the source, the proxy graphics, the data bits up to the handle stream and the
  object's own handles; for R2007+ sources the data keeps the string stream. The common handles are
  checked against the decoded ones; on a mismatch the old fallback stays. For `gh109_1` (R2013,
  TCH_*) the 26 proxies match the CAD-written `gh109_1.dxf` in class, drawing format (0x6A001F),
  data and string bits and object ids.
- **Commit:** [`d9004510`](https://github.com/monyuonyu/libredwg-fixes/commit/d9004510df29de8f15efbb4c7017d15efb59ef0c) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447).

### 8.4 DXF: unknown objects and proxies

- **What was wrong:** `dwg2dxf` skipped unknown entities and objects. The drawing format was
  computed as `>> 8` instead of `version | maint << 16`; relative proxy object ids (6, 8, 10, 12)
  were not made absolute.
- **How it showed:** the DXF way (`dwg2dxf`, `dxf2dwg`) lost them.
- **What was changed:** write them and the PROXY_* objects in AutoCAD's layout (90 498/499, 91 class
  number, 95 drawing format, 70, graphics 92/160 + 310 in AcDbProxyEntity, data 161/162 + 310/311
  from R2007 on or 93 + 310, object ids 330/340/350/360, 94 0); `in_dxf` reads them back
  (`add_PROXY`). For `gh109_1` the DXF matches the CAD-written DXF in all 26 proxies, except the
  graphics of 8 (ours are those of the DWG); `gh109_1.dxf` through `dxf2dwg` gives the same proxies
  as the DWG way.
- **Commit:** [`5885ae72`](https://github.com/monyuonyu/libredwg-fixes/commit/5885ae72ccb15e2b0768f922ce3d1c705be64cfd) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447).

### 8.5 Proxies also for R2004 and R2010+ targets

- **What was wrong:** the proxy conversion only ran for R2000 targets. For R2010+ targets from an
  R2007+ source the proxy data ended with the string stream of the original object instead of
  the proxy's own.
- **How it showed:** an AEC_WALL of `aec_objects` (R2018) became a POINT in R2004, and POINTs and
  placeholders in R2010 and R2013.
- **What was changed:** proxies for any target older than the source; `encode_proxy_data` puts the
  strings of the data into the proxy's string stream.
- **Commits:** [`a970cfee`](https://github.com/monyuonyu/libredwg-fixes/commit/a970cfeecbc350f21b88f60c22205dbb69db445e), [`5243d4f5`](https://github.com/monyuonyu/libredwg-fixes/commit/5243d4f5610125fdc3bb2d741ba70d21e12cc32b) — not yet offered upstream.

The bound on the number of classes, found on an AEC drawing, is in [1.2](#12-the-number-of-classes-was-bounded-too-low).

## 9. Other entities and objects

### 9.1 DIMENSION_ANG2LN: points 10 and 16 swapped

- **What was wrong:** R13+ stores the 16 point (2RD), then 13, 14, 15, and the 10 point (3BD) (ODA
  20.4.27). The 2RD was read into `def_pt` and the last 3BD into `xline2end_pt`.
- **How it showed:** `dwg2dxf` wrote the arc point as group 10 and the end of the second extension
  line as group 16, and DXF readers drew the dimension arc in the wrong place.
- **Drawing:** `sample_AC1032`, handle 516.
- **What was changed:** read the same bits into the fields of the same DXF codes and write 13–16 in
  DXF order; in JSON `xline2end_pt` stays 3D, since `in_json` expects a 3D point (else "JSON 3DPOINT
  must be ARRAY of size 3" and the rest of the import failed).
- **Commits:** [`3d450c21`](https://github.com/monyuonyu/libredwg-fixes/commit/3d450c21651ed381fe5ba101aaa7c8e353dca8f7), [`fee66f28`](https://github.com/monyuonyu/libredwg-fixes/commit/fee66f28e31bb1ca27813672fb665a0278a45a58) — not in a PR: upstream `master` has since fixed the same
  swap in its own commit
  [9bcd0586](https://github.com/LibreDWG/libredwg/commit/9bcd058671439d3d86342012d882805a45d8916e).

### 9.2 R2013 RENDERSETTINGS

- **What was wrong:** the R2013 special case skipped `class_version` and read `has_predefined`
  after `display_index`. AutoCAD's R2013 presets have `class_version` 2 and then the R2018 layout,
  with `has_predefined` at the end.
- **How it showed:** every R2013 RAPIDRTRENDERSETTINGS was shifted by 10 bits (garbage render levels
  and filter sizes); `dwgread -O json` even printed "-nan" (see [14.1](#141-json-no-nan)).
- **Drawings:** `gh44-error`, `gh109_1`.
- **What was changed:** the R2018 layout for R2013.
- **Commit:** [`3cfd5030`](https://github.com/monyuonyu/libredwg-fixes/commit/3cfd5030d85ff792550b62ffb1b16eba07ce6c77) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442).

### 9.3 R13/R14 LAYER: off, plot, lineweight

- **What was wrong:** the R13/R14 decoder overwrote the off bit with "color < 0", but AutoCAD's R14
  drawings keep the color positive and set the bit. And R13/R14 have no plot flag and lineweight;
  writing R2000+ wrote them as 0.
- **How it showed:** off layers were read as on; in R2000 output every layer was not plottable,
  with 0.00 mm lines.
- **What was changed:** off is the bit or a negative color; for R2000+ AutoCAD's defaults, plot and
  lineweight DEFAULT (31).
- **Commit:** [`4b815379`](https://github.com/monyuonyu/libredwg-fixes/commit/4b815379f222c25641ed9124c511db55fc7c1ea0) — off in PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442), plot
  and lineweight in PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 9.4 R13/R14 entities: own linetype, lineweight

- **What was wrong:** the decoder set `ltype_flags` only for ByLayer, not 3 for an entity with its
  own linetype. R13/R14 have no entity lineweight; R2000 output wrote 0.
- **How it showed:** entities with their own linetype (CONTINUOUS) became ByLayer in R2000 output,
  and lines were 0.00 mm.
- **Drawings:** `example_r14`, `sample_AC1014`, against AutoCAD's R2000 saves: the linetype
  differences are gone.
- **What was changed:** `ltype_flags` 3 for an own linetype; lineweight ByLayer (29).
- **Commit:** [`597c0da6`](https://github.com/monyuonyu/libredwg-fixes/commit/597c0da69fbf26c80db4ac4634e4ef884c002e0c) — linetype in PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442),
  lineweight in PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 9.5 LEADER: the R2010+ layout

- **What was wrong:** the annotation offset (`endptproj`) is also in R2010+, and box height and width
  only until R2007, as ACadSharp reads it. LibreDWG read 2 BD for the 3BD.
- **How it showed:** the data overshot by 6 bits; the LEADER of `2000/TS1` written as R2013 and R2018
  was lost.
- **Drawing:** `sample_AC1032` (its LEADER now ends exactly).
- **What was changed:** `endptproj` since R13c3, `box_height`/`box_width` until R2007.
- **Commit:** [`9e5e27b3`](https://github.com/monyuonyu/libredwg-fixes/commit/9e5e27b33d9c622b1e6865ebfc7fc49515ea26b7) — not yet offered upstream.

### 9.6 BLOCKSCALEACTION: its connection points

- **What was wrong:** AcDbBlockScaleAction has 3 connection points after the 2 of
  AcDbBlockActionWithBasePt (`conn_pts[5]`, `in_dxf` uses 2–4), but they were read into 0–2.
- **How it showed:** UpdatedBaseX and UpdatedBaseY were overwritten with Scale and XScale.
- **What was changed:** connection points 2–4.
- **Commit:** [`a8f2deb1`](https://github.com/monyuonyu/libredwg-fixes/commit/a8f2deb148e2550913e32de9fd37c7412e1b1052) — not yet offered upstream.

## 10. DXF import

### 10.1 LWPOLYLINE widths and flags; off layers

- **What was wrong:** `in_dxf` allocated the widths only when the first vertex had 40/41, set the
  constant width flag 4 instead of 32, overwrote the flags with 70 and dropped plinegen of closed
  polylines; `out_dxf` wrote widths only with bulges. The negative DXF color of an off layer was
  written into the DWG as index 65529.
- **How it showed:** the `dwg2dxf` output of `sample_AC1015` (widths only on the last vertex) could
  not be read back ("Invalid DXF code 41 for LWPOLYLINE"); ACadSharp rejected the layer color.
- **What was changed:** widths from any vertex, flag 32, DWG flags kept; off layers have the off
  bit and a positive color, as AutoCAD writes them.
- **Commit:** [`57cda29e`](https://github.com/monyuonyu/libredwg-fixes/commit/57cda29e049ce9995721b6e90e1e614cc11caa6b) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 10.2 SPATIAL_FILTER

- **What was wrong:** SPATIAL_FILTER has all its doubles as group 40 (front clip z and both
  12-double matrices); the generic importer could not tell them apart.
- **How it showed:** the clip vertices were garbled, the inverse matrix filled with NaN, and
  ACadSharp could not read it after DXF to DWG.
- **What was changed:** a dedicated reader.
- **Commit:** [`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8) (part) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 10.3 HATCH: a seed point at (0,0)

- **What was wrong:** after the boundary paths, 10/20 of 0.0 were taken as the elevation, not as a
  seed point.
- **How it showed:** a seed at the origin was dropped and `num_seeds` 1 became 0
  (`sample_AC10xx` via DXF).
- **What was changed:** after the paths they are seed points.
- **Commit:** [`da463d6d`](https://github.com/monyuonyu/libredwg-fixes/commit/da463d6d34deddbfcb3c107a1582bed3d65a4384) (part) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 10.4 `*Model_Space` keeps its DXF handle; VX_CONTROL

- **What was wrong:** object[0] (`*Model_Space`) was created with handle 0x1F before the TABLES were
  read. The encoder gave an added VX_CONTROL the handle B without checking that it was free.
- **How it showed:** `gh209_1` (`*Model_Space` 2, LAYER table 1F) got duplicate handles, two model
  spaces and 14 entities on the wrong layer via DXF; its DICTIONARY is B.
- **What was changed:** a quick scan of the ASCII DXF takes the handle of its `*Model_Space`
  BLOCK_RECORD; VX_CONTROL gets B only if free.
- **Commit:** [`f7880bf9`](https://github.com/monyuonyu/libredwg-fixes/commit/f7880bf993cc0926d66de6041ae95baf45661a4d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 10.5 Table names: an exact match before one ignoring case

- **What was wrong:** `dwg_find_tablehandle` matched names ignoring case and took the first.
- **How it showed:** drawings of other software can have both STANDARD and Standard; the DIMENSION
  of `gh109_1` referring to STANDARD got Standard after DXF import.
- **What was changed:** an exact match first.
- **Commit:** [`eedf8e37`](https://github.com/monyuonyu/libredwg-fixes/commit/eedf8e37d597c5324b246a6dc9edc1084a87947a) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 10.6 Header variables missing from the DXF

- **What was wrong:** DXF may omit any header variable; they stayed 0 even where 0 is invalid.
- **How it showed:** FreeCAD's DXF has no $TEXTSIZE and $DIMTXT, so FreeCAD's DWG export
  (`dxf2dwg`) had text heights 0, which ACadSharp rejects.
- **What was changed:** AutoCAD's defaults for TEXTSIZE, DIMTXT (acad or acadiso with MEASUREMENT
  1), LTSCALE, CELTSCALE, DIMLFAC, DIMTFAC, DIMALTF.
- **Commit:** [`98acd4a3`](https://github.com/monyuonyu/libredwg-fixes/commit/98acd4a314d8d2cf7c9525c66ea4b900a140b54a) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 10.7 SHAPE: the name is not the shape number

- **What was wrong:** dynapi has `style_id` as DXF 2, and the generic importer stored the string
  pointer as the number.
- **How it showed:** a random number on every run and platform; Linux and Windows `dxf2dwg` wrote
  different DWGs.
- **What was changed:** keep 0; the number cannot be found without reading the SHX file.
- **Commit:** [`0f204cfc`](https://github.com/monyuonyu/libredwg-fixes/commit/0f204cfcfca9dd153bfa59dc9dc998453e00507d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 10.8 Owned vertices and attributes as absolute handles

- **What was wrong:** `dxf_postprocess_SEQEND` made the `vertex[]` of POLYLINE_PFACE/MESH and the
  `attribs[]` of INSERT/MINSERT relative soft pointers, written to R2004+ as relative codes 6 and 10.
- **How it showed:** ACadSharp read them as other handles and lost the vertices and attributes of
  DXFs written as R2004+ (160 entities of the corpus).
- **What was changed:** absolute handles, as AutoCAD.
- **Commit:** [`1b1a0c62`](https://github.com/monyuonyu/libredwg-fixes/commit/1b1a0c629952aa23c6a9fbdb4e563c7e91836844) — not yet offered upstream.

### 10.9 `dwgwrite` reads a DXF for the target version

- **What was wrong:** `dwgwrite` read the DXF with an unknown `header.version`, so `in_dxf` built it
  differently than for `dxf2dwg`.
- **How it showed:** of AutoCAD's `example_2018` as DXF, only 91 of 222 entities were kept (as R2004
  and R2018).
- **What was changed:** set the version to write (or R2000) before reading, as `dxf2dwg`.
- **Commit:** [`8ba7693c`](https://github.com/monyuonyu/libredwg-fixes/commit/8ba7693c14a60fc55d1eee1a70fb551ee821f230) — not yet offered upstream.

Also DXF import: EED 1003/1004 ([4.1](#41-eed-layer-1003-a-big-endian-handle-a-name-in-dxf),
[4.2](#42-dxf-eed-binary-1004-decoded-twice)), UTF-8 text
([3.2](#32-r2007-dxf-to-r2000-convert-the-utf-8-text-to-the-drawing-codepage),
[3.3](#33-dxf-to-r2007-names-cut-to-their-first-character)), embedded MTEXT
([5.5](#55-dxf-import-the-embedded-mtext-of-attribattdef-the-mtext-columns-part)), attribute tags
([5.9](#59-attribute-tags-any-character-but-space-and-)), tables ([6.6](#66-dxf-import-and-export-of-acad_table)),
SAT ([7.6](#76-dxf-sat-lines-and-the-escaped-)), proxies ([8.4](#84-dxf-unknown-objects-and-proxies)),
DIMSTYLE and SPLINE defaults ([13.16](#1316-dimstyle-autocads-defaults-of-the-r2007-variables),
[13.17](#1317-spline-flags-knot-parameter-and-scenario-for-r2013)).

## 11. DXF export

### 11.1 HATCH/MPOLYGON: the pattern name as in the DWG

- **What was wrong:** a solid fill was always written as "SOLID", dropping the island style suffix
  of its name.
- **How it showed:** "SOLID,_O" became "SOLID" via DXF.
- **What was changed:** write the name with its island style suffix ("SOLID,_O") as stored in the
  DWG; only a solid fill without a name is written as SOLID.
- **Commit:** [`1935be09`](https://github.com/monyuonyu/libredwg-fixes/commit/1935be09e0ad6e1cac637427804eb02741afeada) (part) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 11.2 R13/R14 drawings keep their newer class entities

- **What was wrong:** an R14 DWG saved by newer AutoCAD can have MULTILEADER, WIPEOUT, MESH,
  PDFUNDERLAY and ARC_DIMENSION classes; `out_dxf` skipped them for R13/R14 DXF.
- **How it showed:** the DXF dropped them all.
- **What was changed:** skip them only when converting newer drawings to R13/R14 DXF.
- **Commit:** [`649637f8`](https://github.com/monyuonyu/libredwg-fixes/commit/649637f8a4a98102115f7747787dd7883178650c) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 11.3 Standard/ByLayer/ByBlock spelling only when converting from R13/R14

- **What was wrong:** `out_dxf` respelled names like STANDARD → Standard for all drawings.
- **How it showed:** `gh109_1` (R2013) has a DIMSTYLE STANDARD, and its DIMENSIONs referred to
  Standard in DXF.
- **What was changed:** respell only when converting from R13/R14.
- **Commit:** [`fc66341d`](https://github.com/monyuonyu/libredwg-fixes/commit/fc66341dacc1b48750fbf947daa9e4c0301470a6) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

### 11.4 `dwg2dxf --as r2000` of R2004+ drawings wrote no entities

- **What was wrong:** `get_first_owned_entity` / `get_next_owned_block_entity` used the refs added
  by `dwg_fixup_BLOCKS_entities`, which were unresolved.
- **How it showed:** `dwg2dxf --as r2000` of R2004+ drawings wrote no entities.
- **What was changed:** find the block entities by their handles.
- **Commit:** [`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb) (part) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445).

Also DXF export: true colors ([2.4](#24-dxf-true-colors-420429-as-24-bit-rgb)), DIMSTYLE colors
([2.5](#25-dxf-the-dimstyle-colors-176178-are-color-indexes)), EED 1003
([4.1](#41-eed-layer-1003-a-big-endian-handle-a-name-in-dxf)), MTEXT `\P`
([5.7](#57-mtext-line-breaks-as-p)), ATTRIB/VERTEX after a SEQEND
([5.8](#58-dxf-export-attrib-and-vertex-after-a-seqend)), tables and table styles
([6.6](#66-dxf-import-and-export-of-acad_table), [6.7](#67-dxf-tablestyle-in-autocads-layout)),
ACIS ([7.5](#75-sat-to-sab-and-back), [7.6](#76-dxf-sat-lines-and-the-escaped-)), proxies
([8.4](#84-dxf-unknown-objects-and-proxies)).

## 12. Writing R2000 from newer versions

With these fixes `example_2004` … `example_2018` written as R2000, directly or via DXF, equal
AutoCAD's own R2000 save in all 222 entities; over both drawing groups 3,258 of 3,405 entities are
equal ([`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40)). The rest are AutoCAD's renumbered handles and anonymous blocks, lineweights
R14 cannot store, and three conversions not done (stacked fraction codes, dynamic block references,
fit-point splines to control points).

### 12.1 VERTEX of POLYLINEs and ATTRIB of INSERTs linked

- **What was wrong:** R2004+ keep the entities owned by another entity in a handle list (`vertex[]`,
  `attribs[]`); R13–R2000 in a linked list (first/last in the owner, prev/next in each owned
  entity). `dwg_fixup_BLOCKS_entities` built that only for the entities of block headers.
- **How it showed:** an R2018 drawing written as R2000 had first/last vertex and attrib 0 and
  unlinked vertices: in AutoCAD's R2018 sample rewritten as R2000, ACadSharp found none of 10 VERTEX
  and 4 ATTRIB ("Leading entity with handle 0 not found").
- **What was changed:** build the links from the lists, as AutoCAD writes them (`sample_AC1015`):
  first/last as absolute code 4 soft pointers (a relative code 6 is legal, but ACadSharp resolves it
  against handle 0 and crashes), prev/next relative.
- **Commit:** [`89191ebe`](https://github.com/monyuonyu/libredwg-fixes/commit/89191ebe07c103dfdf91f56fe74556d573486389) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 12.2 MLINESTYLE: the linetype index

- **What was wrong:** since R2018 the lines of an MLINESTYLE refer to their linetype by handle,
  before by index (a union). Writing R2000 wrote the handle pointer as the index.
- **How it showed:** ACadSharp failed with "Linetype not assigned, index 29712".
- **What was changed:** convert the handle: 32767 ByLayer, 32766 ByBlock, else the position in the
  LTYPE_CONTROL entries. AutoCAD's R2000 and R2018 saves of the same drawing agree (ByLayer handle
  0x15 is index 32767).
- **Commit:** [`a1670d7a`](https://github.com/monyuonyu/libredwg-fixes/commit/a1670d7a3b25ed0cc22a4cb3a8939cdac43cb671) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 12.3 LWPOLYLINE: no vertex id flag before R2010

- **What was wrong:** R2010+ LWPOLYLINEs written as R2000 kept the flag 1024 (vertex ids).
- **How it showed:** ACadSharp read a `num_vertexids` that is not there and lost the points.
- **Drawing:** `2018/Dynblocks`.
- **What was changed:** clear the flag before R2010.
- **Commit:** [`c2e85bb6`](https://github.com/monyuonyu/libredwg-fixes/commit/c2e85bb62181e2d04578d3f3387756ced6f44590) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 12.4 AuxHeader: 123 bytes for R2000 too

- **What was wrong:** ODA chapter 27 lists the trailing RLs `zero_7` and `zero_8` for every version,
  but they were written only since R2004.
- **How it showed:** an R2000 file from `dwgrewrite` had a 115-byte AuxHeader (AutoCAD's is 123
  bytes, `sample_AC1015`), and ACadSharp threw EndOfStreamException.
- **What was changed:** write them for all versions.
- **Commit:** [`3ab77f4d`](https://github.com/monyuonyu/libredwg-fixes/commit/3ab77f4d123107fd17bddd725865dd1e1dad9039) (part) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442).

### 12.5 R13/R14 names as AutoCAD 2000 writes them

- **What was wrong:** R13/R14 spell names in capitals; AutoCAD's R2000 save of the same drawing
  uses ByBlock, ByLayer, Continuous, Standard and the dictionary key Standard.
- **Drawings:** `example_r14` against `example_2000`.
- **What was changed:** those names when writing R2000+ from R13/R14; MLINESTYLE keeps STANDARD.
- **Commit:** [`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb) (part) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

### 12.6 Gradient hatches

- **What was wrong:** R2000 has no gradient fills.
- **How it showed:** gradient HATCHes stayed ByLayer, unlike AutoCAD's R2000 save.
- **Drawings:** `sample_AC1015`, `example_2000` (AutoCAD's R2000 saves).
- **What was changed:** a gradient HATCH becomes a solid fill in its first gradient color, as
  AutoCAD does.
- **Commit:** [`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40) (part) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444).

Also for R2000 output: colors ([2.3](#23-colors-written-to-r2000-byblock-autocads-aci-fallback-the-aci-palette)),
EED strings ([4.3](#43-r2007-eed-strings-written-to-r2000)), R2018 multi-line attributes
([5.3](#53-r2018-multi-line-attributes-written-to-earlier-versions)), MTEXT
([5.6](#56-r13r14-mtext-written-as-r2000-line-spacing-1-not-0), [5.7](#57-mtext-line-breaks-as-p)),
tables ([6.4](#64-r2010-tables-written-to-r2000r2004-in-the-old-format),
[6.5](#65-r2007-tables-written-to-r2000r2004-the-cell-text)), proxies
([8.3](#83-unknown-objects-written-as-acad_proxy_entity--acad_proxy_object)), R13/R14 layers and
entities ([9.3](#93-r13r14-layer-off-plot-lineweight), [9.4](#94-r13r14-entities-own-linetype-lineweight)).

## 13. Writing R2004 to R2018

Upstream LibreDWG writes R2000 and older only. With these commits LibreDWG writes R2004, R2010, R2013 and R2018;
all 149 drawings round trip through ACadSharp in each version, directly and via DXF, without
losses. None of them is offered upstream yet.

### 13.1 The R2004 file container, with LZ77 compression

- **What was wrong:** the R2004 container (also the frame of R2010+) was written so that no reader
  accepted it: the section map and page map were listed as data sections (ACadSharp failed on
  duplicate empty section names), the pages had wrong sizes, the Preview was a page of 1.3 MB,
  system pages were unaligned and the tail of the file header was clobbered.
- **What was changed:** `encode_r2004_container`, written as AutoCAD's own R2004 files: the empty
  section first, ids counting down in section map order; compressed sections in 0x7400 pages, others
  in one page of their size rounded up to 0x80 (Preview 0x400); page headers with size, 64-bit
  start offset and checksums; pages aligned to 0x20 and padded with the magic sequence; section map
  and page map after the data pages; the file header at 0x80, repeated at the end; the AppInfo and
  AppInfoHistory addresses. A new LZ77 compressor `r2004_compress` (ODA 4.7, with hash chains); its
  unit test round trips through `decompress_R2004_section`.
- **Commits:** [`1eb484a6`](https://github.com/monyuonyu/libredwg-fixes/commit/1eb484a6b53445a2f9be6d2a8e09d512b674d485), [`9a0474f3`](https://github.com/monyuonyu/libredwg-fixes/commit/9a0474f3679c1283b96fffc04197c3a3e0bb34e2).

### 13.2 R2004+ input: the page count is not the section count

- **What was wrong:** `header.sections` of a decoded R2004+ DWG is its number of pages (56 in
  `sample_AC1018`).
- **How it showed:** `dwg_sections_init` failed with "Too many sections" and nothing was written.
- **What was changed:** reset it to 6 for R2004+ output.
- **Commit:** [`7cb0849e`](https://github.com/monyuonyu/libredwg-fixes/commit/7cb0849e1bcc8a77e55ae2b2f9fd21770c3ff8b8).

### 13.3 The file header as AutoCAD writes it

- **What was wrong:** AutoCAD's R2004+ files have 3 at 0x0C; R2018 files have the maintenance
  version also at 0x0B and 0 at 0x15–0x17 (all 37 AC1032 drawings of the corpus). LibreDWG's R2018
  rewrite had 0x1D at 0x0B and the application version 0x21 at 0x16; R2004+ files made from earlier
  versions had 1 at 0x0C and the source's application version at 0x11.
- **What was changed:** as AutoCAD across versions; no application versions for R2018.
- **Commit:** [`649dd383`](https://github.com/monyuonyu/libredwg-fixes/commit/649dd3838c46e3e92277702fef3dfadfb879bd59).

### 13.4 R2010 and R2013: maintenance versions above 3

- **What was wrong:** the header and classes sections of R2010 and R2013 have a 4-byte hsize after
  their size only with a maintenance version above 3. ACadSharp takes it from the byte at 0x0B,
  LibreDWG from the one at 0x12. An R2018 source has 0 at 0x12 and got 0x6d at 0x0B.
- **How it showed:** ACadSharp expected the hsize and could not read the header
  (EndOfStreamException, e.g. `sample_base__empty` as R2010).
- **What was changed:** raise both to above 3 for R2010 and R2013, as AutoCAD's files have.
- **Commit:** [`79c07845`](https://github.com/monyuonyu/libredwg-fixes/commit/79c07845985f416afe2af4a3fc09036050f0e919).

### 13.5 R2010+ header and classes: the section size without the hsize

- **What was wrong:** the size of the R2010+ header and classes sections counted the 4-byte hsize
  after it; AutoCAD's hsize of the classes is 0, not the byte size.
- **How it showed:** the CRC and end sentinel were 4 bytes before where the size points to
  (ACadSharp: "Invalid section sentinel found in AcDb:Header").
- **What was changed:** `encode_patch_RLsize_hsize`; the classes hsize 0.
- **Commit:** [`ceeaca39`](https://github.com/monyuonyu/libredwg-fixes/commit/ceeaca39aa0f23085a1498bc9880c36c6c250333).

### 13.6 Class version 0 kept

- **What was wrong:** AutoCAD writes the class version 0 for many classes; the encoder replaced it
  with the drawing's version.
- **Drawing:** `2004/Line` (EXACXREFPANELOBJECT, SOLID_BACKGROUND became 31.8).
- **What was changed:** use the drawing's version only from R2000 and older (which have no class
  versions) or from DXF/JSON.
- **Commit:** [`e9563c45`](https://github.com/monyuonyu/libredwg-fixes/commit/e9563c453ac021a08906a38d6c7028252116017c).

### 13.7 Template, Classes and AcDbObjects begin and end as AutoCAD

- **What was wrong:** the AcDb:Template description, the end of the classes section and the 4 bytes
  before the first object did not match AutoCAD's files.
- **What was changed:** an empty Template description is RS 0 (4 bytes in all) for R2004 and RS 1
  with the terminating 0 (6 bytes) since R2007; R2004+ classes end with 8 zero bytes after the
  sentinel, counted in the section size (the RL 0x0dca stays for R13–R2000); R2004+ AcDbObjects start
  with the RL 0x0dca, not 0, as AutoCAD (and ACadSharp).
- **Commits:** [`beb122ae`](https://github.com/monyuonyu/libredwg-fixes/commit/beb122aed566b71c27cdafe1acb810fb736657d3), [`4738d852`](https://github.com/monyuonyu/libredwg-fixes/commit/4738d85246bf7b3f01553cec5c81513e689879df), [`7caa82f8`](https://github.com/monyuonyu/libredwg-fixes/commit/7caa82f84092580a14bc2679c4bac1e583dc9fdc).

### 13.8 Preview: the END sentinel and the image addresses

- **What was wrong:** a decoded R2000 preview lacks the END sentinel, which was only written to
  R13–R2000 output. The header and image start addresses in the preview are file addresses, but
  were copied from the source.
- **How it showed:** reading back R2004 output: "Preview overflow 13208 + 119 > 13311".
- **What was changed:** write the sentinel whenever the content lacks it; `thumbnail_rebase` moves
  the addresses to the new place.
- **Commit:** [`30a1f7a1`](https://github.com/monyuonyu/libredwg-fixes/commit/30a1f7a11d5170dc93f57c8bac3961eb5560990b).

### 13.9 Object types from 0x2F0

- **What was wrong:** `bit_write_BOT` wrote every type of 256 or more as RC + 0x1F0, and its third
  branch had code 1 with an RS.
- **How it showed:** classes numbered from 0x2F0 (752) got a wrong type in R2010+.
- **What was changed:** as AutoCAD and `bit_read_BOT`: code 1 for 0x1F0–0x2EF, code 2 with an RS for
  the others.
- **Commit:** [`78fe37ba`](https://github.com/monyuonyu/libredwg-fixes/commit/78fe37ba18e9581ce347546460ae162225d49b0b).

### 13.10 Strings of R2004+ files: T16, TU and the string stream

- **What was wrong:**
  - Before R2007 an empty T16 string is length 1 and the NUL (AutoCAD's SummaryInfo has `01 00 00`);
    LibreDWG wrote `00 00` ([`754d999d`](https://github.com/monyuonyu/libredwg-fixes/commit/754d999d6a521a0ef329530068b40f525b9080f9)).
  - Since R2007 the RS length counts the terminating 0 and an empty string is RS 1 with the 0; from
    earlier versions the length was written without the 0, and R2007+ sources wrote UTF-16 into
    R2000/R2004 files too ([`ce66db95`](https://github.com/monyuonyu/libredwg-fixes/commit/ce66db95cfd3ac20b3855d8c35ce31dfda9e6adf)).
  - AutoCAD writes TU strings without a terminating 0 (the NOD of `aec_objects`: 69 names, 69 × 16
    bits less string stream) ([`011c3181`](https://github.com/monyuonyu/libredwg-fixes/commit/011c3181df8591028bc1c83e38b349e35303f8ad), test [`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478)).
  - The string stream size before its `has_strings` bit was cast to RS, so with more than 32,767 bits
    of strings the high part was wrong ([`632dfd5c`](https://github.com/monyuonyu/libredwg-fixes/commit/632dfd5cc46018f8ca7a4014e5610b67a6e64ef1)).
- **How it showed:** different SummaryInfo bytes; with large string streams (the classes of
  `aec_objects` and `gh44-error` as R2018) ACadSharp read the classes from a wrong place.
- **What was changed:** as AutoCAD; `bit_write_strings_size` writes the RS size, with 0x8000 and the
  high bits for large ones. `2004/Line` written as R2004 has AutoCAD's SummaryInfo byte for byte.
- **Commits:** [`754d999d`](https://github.com/monyuonyu/libredwg-fixes/commit/754d999d6a521a0ef329530068b40f525b9080f9), [`ce66db95`](https://github.com/monyuonyu/libredwg-fixes/commit/ce66db95cfd3ac20b3855d8c35ce31dfda9e6adf), [`011c3181`](https://github.com/monyuonyu/libredwg-fixes/commit/011c3181df8591028bc1c83e38b349e35303f8ad), [`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478), [`632dfd5c`](https://github.com/monyuonyu/libredwg-fixes/commit/632dfd5cc46018f8ca7a4014e5610b67a6e64ef1).

### 13.11 Owned entities, attributes and vertices as lists

- **What was wrong:** R2004+ have no first/last entity links in the BLOCK_HEADER, no first/last
  attrib in INSERT/MINSERT and no first/last vertex in POLYLINE, but `num_owned` and a list of
  handles. They were empty.
- **How it showed:** `sample_AC1015` written as R2004 lost all its 355 entities.
- **What was changed:** `upconvert_owned_lists` collects the entities by their owner in object order
  (without BLOCK, ENDBLK and SEQEND), ATTRIBs to their INSERT, VERTEXes to their POLYLINE.
- **Commit:** [`2098043b`](https://github.com/monyuonyu/libredwg-fixes/commit/2098043bbaaa4d2ad816a76209e757bc019b6144).

### 13.12 PLOTSETTINGS, LAYOUT: shade plot defaults

- **What was wrong:** R2000 has no shade plot fields; R2004+ got a custom DPI of 0.
- **How it showed:** ACadSharp rejects it ("The valid shade plot DPI values range from 100 to
  23767"), so every LAYOUT was lost.
- **What was changed:** resolution level 2 and 300 DPI, the values of AutoCAD's R2004 drawings
  (`sample_AC1018`).
- **Commit:** [`653fea5e`](https://github.com/monyuonyu/libredwg-fixes/commit/653fea5ebdf0e3e7edb2a306165ed075f9fd9fa0).

### 13.13 XRECORD data of earlier versions

- **What was wrong:** xdata strings are UTF-16 since R2007, but writing was bounded by the
  `xdata_size` of the source.
- **How it showed:** from R2000 the values after the first strings were cut off; ACadSharp failed on
  these xrecords ("Arithmetic operation resulted in an overflow", 8 in `sample_AC1015` as R2010).
- **What was changed:** across versions allow up to twice the size; the size is recalculated.
- **Commit:** [`5300c579`](https://github.com/monyuonyu/libredwg-fixes/commit/5300c579ff3943543011c6de2dd4e3b70df36471).

### 13.14 MATERIAL handles of DWG input

- **What was wrong:** writing R2007+, the material handles of entities, of LAYER and of 3DSOLID,
  REGION and BODY were dropped, because MATERIAL objects are written as placeholders. That is only
  so from DXF.
- **How it showed:** R2018 to R2018 lost the material of every layer.
- **What was changed:** drop them only for DXF input.
- **Commit:** [`036da95e`](https://github.com/monyuonyu/libredwg-fixes/commit/036da95ef1a810ef1c60826b0aa535ba1d9ae65d).

### 13.15 AcDbBlockElement: the version as read

- **What was wrong:** `be_major` and `be_minor` were written as 33 and 29 for R2007+; AutoCAD's
  R2018 drawings have 33 and 329.
- **What was changed:** write them as read from an R2007+ DWG.
- **Commit:** [`eac6288a`](https://github.com/monyuonyu/libredwg-fixes/commit/eac6288a16743092f005cecbddfd848d007d8739).

### 13.16 DIMSTYLE: AutoCAD's defaults of the R2007+ variables

- **What was wrong:** from an earlier version or from DXF, DIMJOGANG, DIMFXL, DIMALTMZF and DIMMZF
  were 0 and DIMTFILLCLR had no color method. The first fix covered older DWGs only; from DXF
  `dat.from_version` is the target version.
- **How it showed:** ACadSharp rejects DIMJOGANG 0 ("must be in range of 5 to 90 degrees"), dropped
  such dimension styles, and the dimensions got Standard (63 in the round trips to R2010+).
- **What was changed:** AutoCAD's defaults: DIMFXL 1, DIMJOGANG 45°, DIMTFILLCLR ByBlock, DIMALTMZF
  and DIMMZF 100, in DIMSTYLE and in the header variables, also from DXF.
- **Commits:** [`16871d28`](https://github.com/monyuonyu/libredwg-fixes/commit/16871d2898ba823f957abae68a73c7114be2e50a), [`c720b36d`](https://github.com/monyuonyu/libredwg-fixes/commit/c720b36d04579353ac62ebda61b43dde5f379f72).

### 13.17 SPLINE: flags, knot parameter and scenario for R2013+

- **What was wrong:** from an earlier version `splineflags` was 8 or 9 and `knotparam` 0; from DXF
  both were 0. ACadSharp takes the scenario from these, so a control point spline looked like a fit
  point spline. LibreDWG also wrote the decoded scenario (2 for fit point splines); AutoCAD writes 1
  in all 591 R2013+ splines of the corpus.
- **How it showed:** "Failed to read ReadBitDouble" and the spline was lost (32 from DXF in the
  corpus).
- **What was changed:** as AutoCAD: control points with `splineflags` 0 (4 if closed) and
  `knotparam` 15, fit points with 9 and 0; the scenario BL is 1.
- **Commits:** [`09d03974`](https://github.com/monyuonyu/libredwg-fixes/commit/09d03974d1cd3423d68f088bcfdcc7dd85787d29), [`ab05c9d1`](https://github.com/monyuonyu/libredwg-fixes/commit/ab05c9d13e521420f001422af986c9a303d5833e), [`f3322488`](https://github.com/monyuonyu/libredwg-fixes/commit/f332248859806fe1609bb05e4e845ffd1a8629a9).

### 13.18 Found on the way, also affecting R2000

- **`bit_write_UMC`** checked only 4 of 8 groups for leading zeros, so every value took at least 4
  bytes (offset 1 was `81 80 80 00`, AutoCAD writes `01`); the object map of `2004/Line` was 1,174
  bytes instead of 674 ([`691d4598`](https://github.com/monyuonyu/libredwg-fixes/commit/691d4598e5c03d21047caac959358f8df2ba21ad)). It also added a 0 byte when the last byte had 0x40 set, as
  for the signed MC; in R2010+ the handle stream size before each object grew by a byte
  ([`40d6e568`](https://github.com/monyuonyu/libredwg-fixes/commit/40d6e568c035ba2af2c37dd45bc581f1e9835912)). Now the shortest form.
- **`bit_write_DD`** wrote values within 1e-12 of the default as the default: a LINE end 1 ulp off
  its start (`example_2004`) came back as the start. Now only the bitwise same value is the default;
  LINE, LWPOLYLINE and 3DFACE of `example_2004` and `sample_AC1018` written as R2004 are AutoCAD's
  objects bit for bit ([`57cbcdd9`](https://github.com/monyuonyu/libredwg-fixes/commit/57cbcdd91fb0cb99437098919fd66508606e3b5a)).
- **Object map:** when the last handle filled a chunk (over 2030 bytes), 2 reserved bytes stayed as
  zeros before a final chunk of size 2, and ACadSharp read past the end (`sample_AC1032` as R2000)
  ([`d56fd195`](https://github.com/monyuonyu/libredwg-fixes/commit/d56fd19555ec86e6366a6c0a3b3497d2b8c8acbb)).
- **SPATIAL_INDEX:** its format is not understood; `aec_objects` has a `bindata_size` of 50,396,167,
  which the decoder rejected but the encoder wrote as that many zero bytes, a 51 MB object section.
  An invalid size is now 0 ([`5398b8a6`](https://github.com/monyuonyu/libredwg-fixes/commit/5398b8a6c63a4e43815114dbbfe0df1dd2c2db12)).
- **Commits:** [`691d4598`](https://github.com/monyuonyu/libredwg-fixes/commit/691d4598e5c03d21047caac959358f8df2ba21ad), [`40d6e568`](https://github.com/monyuonyu/libredwg-fixes/commit/40d6e568c035ba2af2c37dd45bc581f1e9835912), [`57cbcdd9`](https://github.com/monyuonyu/libredwg-fixes/commit/57cbcdd91fb0cb99437098919fd66508606e3b5a), [`d56fd195`](https://github.com/monyuonyu/libredwg-fixes/commit/d56fd19555ec86e6366a6c0a3b3497d2b8c8acbb), [`5398b8a6`](https://github.com/monyuonyu/libredwg-fixes/commit/5398b8a6c63a4e43815114dbbfe0df1dd2c2db12).

Also for R2004+: class entries ([1.1](#11-r2004-class-entries-dwg_version-and-maint_version-are-bl)),
header ([1.4](#14-r2007-header-no-unknown_54--unknown_57)), FileDepList
([1.5](#15-filedeplist-the-feature-names-are-t32-strings)), true colors from EED
([2.6](#26-r13r2000-to-r2004-the-true-colors-autocad-keeps-in-eed)), DXF strings
([3.3](#33-dxf-to-r2007-names-cut-to-their-first-character)), tables
([6.8](#68-tables-written-to-r2010-from-earlier-versions-and-from-dxf), [6.9](#69-memory-leaks-when-writing-r2007-from-earlier-versions)),
ACIS ([7.2](#72-writing-the-empty-bit-the-history-handle-materials) – [7.6](#76-dxf-sat-lines-and-the-escaped-)),
proxies ([8.5](#85-proxies-also-for-r2004-and-r2010-targets)), LEADER
([9.5](#95-leader-the-r2010-layout)), BLOCKSCALEACTION ([9.6](#96-blockscaleaction-its-connection-points)),
DXF handles and `dwgwrite` ([10.8](#108-owned-vertices-and-attributes-as-absolute-handles),
[10.9](#109-dwgwrite-reads-a-dxf-for-the-target-version)).

## 14. Tools and tests

### 14.1 JSON: no NaN

- **What was wrong:** JSON has no NaN. `VALUE_RD` replaced NaN only in release builds, and not
  infinity; doubles in vectors (`FIELD_VECTOR_N`, e.g. the SPATIAL_FILTER matrices) were printed
  with `fprintf` directly.
- **How it showed:** `dwgread -O json` printed "-nan" for a misread RAPIDRTRENDERSETTINGS and the
  whole output became unreadable.
- **Drawing:** `gh44-error`.
- **What was changed:** NaN and infinite doubles are written as 0.0, one check in `_VALUE_RD` for all
  builds, used for the vectors too.
- **Commits:** [`c86190fb`](https://github.com/monyuonyu/libredwg-fixes/commit/c86190fbcc0db292aa09e9592e24bdca4b8513ef), [`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8) (part) — PR [#1441](https://github.com/LibreDWG/libredwg/pull/1441).

### 14.2 `dwgrewrite`, `dxf2dwg`, `dwgwrite`: R2010, R2013 and R2018 are valid

- **What was changed:** the help texts and man pages list r2010, r2013 and r2018 as valid output
  versions (r2007 is still written as r2010), and `dwgwrite` not only r2000. All 149 drawings of the
  corpus round trip through ACadSharp as R2010, R2013 and R2018 without losses, and as DXF written
  as R2004 to R2018 (the remaining changed values, 53 as R2000/R2004 and 66 as R2010+, are all in
  TABLEs).
- **Commits:** [`416c3d22`](https://github.com/monyuonyu/libredwg-fixes/commit/416c3d222396b1419a4dc6edf27e97969135d61a), [`1b613fcb`](https://github.com/monyuonyu/libredwg-fixes/commit/1b613fcbd8b4a04f44b1bc9d2ea591a43d346c04) — not yet offered upstream.

The other test changes are with their fixes: [`45e6b50f`](https://github.com/monyuonyu/libredwg-fixes/commit/45e6b50f615aa7346827f71a26de67dc3a8f0eae) ([3.1](#31-asian-codepages-the-lead-byte-of-two-byte-characters)),
[`aad393ec`](https://github.com/monyuonyu/libredwg-fixes/commit/aad393ec206b1116694db8ae7877874207530709) ([7.1](#71-reading-no-acis-empty-bit-since-r2013-the-r2007-unknown-bl)),
[`9a0474f3`](https://github.com/monyuonyu/libredwg-fixes/commit/9a0474f3679c1283b96fffc04197c3a3e0bb34e2) ([13.1](#131-the-r2004-file-container-with-lz77-compression)),
[`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478) ([13.10](#1310-strings-of-r2004-files-t16-tu-and-the-string-stream)).
