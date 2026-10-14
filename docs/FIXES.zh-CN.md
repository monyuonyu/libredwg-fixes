[English](FIXES.md) | [日本語](FIXES.ja.md) | **简体中文**

> 本文译自 [FIXES.md](FIXES.md)（英文版为原文）。

# 每一项修复的说明

本页逐一说明 `fixes` 分支上的 116 个修复提交，它们建立在上游标签 `0.14.8601` 之上。
对每项修复，说明哪里出错、用户看到的现象、哪张图纸可以看出问题、做了什么修改，并附上提交链接。
简短的概要见 [README](../README.zh-CN.md)。

## 如何发现这些问题

- **独立的读取库。** 149 张图纸（R13 至 R2018）分别用 LibreDWG 和独立的 MIT 许可读取库
  [ACadSharp](https://github.com/DomCR/ACadSharp) 3.8.0 读取，并逐个图元、逐条表记录、逐个头变量进行比较。
  每一处差异都追查到了位级别。有时是 LibreDWG 错了，有时是成文的规范错了（ODA 的
  "Open Design Specification for .dwg files"，下文引用为 "ODA 20.4.x"）。
  规范与 AutoCAD 生成的文件不一致时，以文件为准。
- **AutoCAD 自己保存的图纸。** AutoCAD 将同一张图纸保存为了所有版本：`sample_AC1014` …
  `sample_AC1032`（ACadSharp 的样例）和 `example_r14` … `example_2018`（LibreDWG 的测试数据）。
  把其中一个转换为另一个版本，再与 AutoCAD 自己保存的该版本比较，就能看出 AutoCAD 在转换时做了什么。
- **往返测试。** 每张图纸都以 R2000、R2004、R2010、R2013 和 R2018 写出，包括直接写出
  （`dwgrewrite`）和经由 DXF（先 `dwg2dxf` 再 `dxf2dwg`），再用 ACadSharp 读回。途中丢失或改变的任何内容都是错误。

图纸名称沿用测试集中的名称。`sample_AC10xx` 和 `sample_base` 是 ACadSharp 的样例；`example_*`、`gh109_1`、
`gh44-error`、`gh209_1`、`2000/TS1`、`2004/Line` 和 `2018/Dynblocks` 位于 LibreDWG 的 `test/test-data`。
字段名和函数名均为 LibreDWG 中的名称（主要在 `src/dwg.spec`、`src/decode.c`、`src/encode.c`、`src/in_dxf.c`、
`src/out_dxf.c`）。DWG 类型：B 为位，BS/BL/BD 为位编码的 short/long/double，RC/RS/RL 为原始的 char/short/long，
TV/TU 为 8 位和 UTF-16 文本，H 为句柄。

### 提交给上游的拉取请求

读取、写出 R2000 以及 DXF 方面的大部分修复，都已以拉取请求的形式提交给
[LibreDWG](https://github.com/LibreDWG/libredwg)。这些提交已变基到上游 `master` 上，其间有些被拆分或合并，
因此这里的一个提交可能对应多个 PR 的各一部分。

| PR | 主题 |
|---|---|
| [#1440](https://github.com/LibreDWG/libredwg/pull/1440) | 亚洲代码页：双字节字符的首字节 |
| [#1441](https://github.com/LibreDWG/libredwg/pull/1441) | JSON 输出：不输出 NaN |
| [#1442](https://github.com/LibreDWG/libredwg/pull/1442) | 读取方面的修复（类、颜色、AuxHeader、ACIS、MTEXT、LAYER、EED 等） |
| [#1443](https://github.com/LibreDWG/libredwg/pull/1443) | R2018 多行 ATTRIB/ATTDEF |
| [#1444](https://github.com/LibreDWG/libredwg/pull/1444) | 从较新版本写出 R2000 |
| [#1445](https://github.com/LibreDWG/libredwg/pull/1445) | DXF 导入与导出 |
| [#1446](https://github.com/LibreDWG/libredwg/pull/1446) | TABLE、TABLECONTENT、TABLESTYLE |
| [#1447](https://github.com/LibreDWG/libredwg/pull/1447) | 未知对象作为代理对象 |

用于写出 R2004 至 R2018 的 62 个提交（从 [`1eb484a6`](https://github.com/monyuonyu/libredwg-fixes/commit/1eb484a6b53445a2f9be6d2a8e09d512b674d485) 开始）尚未提交给上游：
上游 LibreDWG 只能写出 R2000 及更早版本。

## 目录

1. [类、文件头和其他文件段](#1-类文件头和其他文件段)
2. [颜色](#2-颜色)
3. [文本和代码页](#3-文本和代码页)
4. [扩展图元数据（EED）](#4-扩展图元数据eed)
5. [MTEXT、ATTRIB 和 ATTDEF](#5-mtextattrib-和-attdef)
6. [TABLE 和 TABLESTYLE](#6-table-和-tablestyle)
7. [ACIS 实体（3DSOLID、REGION、BODY）](#7-acis-实体3dsolidregionbody)
8. [代理对象和未知对象](#8-代理对象和未知对象)
9. [其他图元和对象](#9-其他图元和对象)
10. [DXF 导入](#10-dxf-导入)
11. [DXF 导出](#11-dxf-导出)
12. [从较新版本写出 R2000](#12-从较新版本写出-r2000)
13. [写出 R2004 至 R2018](#13-写出-r2004-至-r2018)
14. [工具和测试](#14-工具和测试)

---

## 1. 类、文件头和其他文件段

### 1.1 R2004 及以后的类条目：`dwg_version` 和 `maint_version` 是 BL

- **原因：** ODA 规范（10.2）把类条目的这两个字段都列为 BS，`read_2004_section_classes` 以及 R2007 的读取函数
  `read_2007_section_classes` 都照此实现。而 AutoCAD 写的是 BL（ACadSharp 也按 BL 读取）。BS 和 BL 只有在值小于 256
  时才一致。编码器（`encode_classes`）同样写的是 BS。另外，第二个未知 long 被再次存入了 `unknown_1`，而不是 `unknown_2`。
- **表现：** 在 AutoCAD 的 R2004 样例中，类 508 MLEADERSTYLE 的 `maint_version` 大于 255。按 BS 读取丢失了 16 位，
  其后的所有类都成了乱码（编号 9974，dxfname 为 "W'6..."），编号 509 及以上的类的所有图元（MULTILEADER、MESH、
  WIPEOUT、IMAGE、PDFUNDERLAY 等）都变成了 UNKNOWN_OBJ。写成 R2004 的图纸中，第一个类之后的所有类都发生错位，
  自定义类的对象读回时类型无效（"Invalid class 523"）。
- **图纸：** `sample_AC1018`。
- **修改：** 两个字段都按 BL 读写；第二个 long 存入 `unknown_2`。该样例现在有 353 个图元与 ACadSharp 一致。
- **提交：** [`19c7ab68`](https://github.com/monyuonyu/libredwg-fixes/commit/19c7ab689d0583a341b69867ea70d879ff97417e)（读取，位于 PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)），
  [`b25ba83f`](https://github.com/monyuonyu/libredwg-fixes/commit/b25ba83f095951ad1f2ddd3b0598775ab8ce4da3)（写出，尚未提交给上游）。

### 1.2 类的数量上限定得过低

- **原因：** 为防止过大的内存分配（CWE-400），解码器最多只允许 `100 + size / sizeof(Dwg_Class)` 个类。
  类名较短的合法图纸会超过这个数。
- **表现：** 一张写成 R2000 的 AEC 图纸有 478 个类，而上限只允许 385 个，因此后面那些类的对象全部无法读取。
- **修改：** 一个类条目至少有 13 位（BS 编号、BS 标志、三个带 BS 长度的 TV、B、BS 项目类 ID），
  所以 `size` 字节的段最多容纳 `size*8/13` 个条目。内存用量仍与输入成线性关系。R2004 及以后的类段也同样处理。
- **提交：** [`4ff2ed83`](https://github.com/monyuonyu/libredwg-fixes/commit/4ff2ed83ccafa44fa85620ab1b2621dc681248bc) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 1.3 R2018 的 AuxHeader：维护版本号是 RL

- **原因：** 在 R2018 的 AuxHeader 中，`maint_version`、`maint_version_1` 和 `maint_version_2` 都是 RL。
  LibreDWG 把它们各按 RS 读取，并把三个高半部分（6 字节）当作末尾的 `zero_18` 读取，
  因此 `maint_version` 之后的所有内容都错位了 2 字节。
- **表现：** `minus_1` 为 0xffff0000，TDCREATE/TDUPDATE 是乱码，R2018 图纸写成 R2000 时带上了这些错误日期，
  ACadSharp 无法读取。
- **图纸：** AutoCAD 将同一张图纸保存为 R2013 和 R2018，AuxHeader 分别为 123 和 129 字节；
  多出的 6 字节就是三个维护版本号的高半部分。
- **修改：** 自 R2018 起，在每个维护版本号之后紧接着读取其高半部分。现在 `minus_1` 为 −1，
  日期和 HANDSEED 与文件头一致。
- **提交：** [`c7efdf98`](https://github.com/monyuonyu/libredwg-fixes/commit/c7efdf987d7d3817456f94b789cc7345e558975f) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 1.4 R2007 及以后的文件头：没有 `unknown_54` … `unknown_57`

- **原因：** 自 R14 起的所有版本中，头变量在 SHADOWPLANELOCATION 之后都有四个 BS `unknown_54` … `unknown_57`。
  但自 R2007 起，字符串流恰好从那里开始。
- **表现：** 这四个值是从字符串中读出的（总是 1、180、36874、32843），写回时成了 56 位的多余数据。
- **图纸：** AutoCAD 的所有 R2007 及以后的图纸（`example_2007` … `example_2018`、`sample_AC1021` …
  `sample_AC1032`）。
- **修改：** 这四个字段只用于 R14 至 R2004（`src/header_variables.spec`）。
- **提交：** [`1a42025a`](https://github.com/monyuonyu/libredwg-fixes/commit/1a42025ad7e5294c05853c27372cd5ccb086a4dd) — 尚未提交给上游。

### 1.5 FileDepList：功能名称是 T32 字符串

- **原因：** R2004 的 FileDepList 段中，功能名称与文件名一样，每个字符串都由 RL 长度和字符组成，不带结尾的 NUL
  （`02000000 09000000 "Acad:Text" 0f000000 "Acad:PlotConfig" 01000000 ...`）。而功能名称被当作位编码的 TV 读取。
- **表现：** 所有功能名称都读成了空，段的其余部分发生错位；写回后成了一个损坏的段。
- **图纸：** `2004/Line`。
- **修改：** 与文件字符串一样，用 `FIELD_T32` 读写功能名称；`bit_write_T32` 与 AutoCAD 一样写出不带 NUL 的 8 位形式。
  `2004/Line` 写成 R2004 后，FileDepList 与 AutoCAD 的逐字节一致。
- **提交：** [`5fa2061b`](https://github.com/monyuonyu/libredwg-fixes/commit/5fa2061b838b24177d32bc6af74ea45224e9f1d3) — 尚未提交给上游。

为 R2000 写出的 AuxHeader 见 [12.4](#124-auxheaderr2000-同样是-123-字节)；为 R2004 及以后版本写出的类条目、文件头和段大小见 [13](#13-写出-r2004-至-r2018)。

## 2. 颜色

### 2.1 图元颜色：RGB 在透明度之前

- **原因：** 在图元颜色（ENC，ODA 2.11）中，标志 0x8000 表示“下一个值是 RGB”，标志 0x2000 表示“后跟一个透明度 BL”，
  顺序就是如此。`common_entity_data.spec` 却先读取了透明度。
- **表现：** 同时具有真彩色和透明度的图元，两者被调换，结果完全没有真彩色，只剩颜色索引：LINE 9D4 得到
  rgb 0x01000000 和透明度 0xc2803dc2（只有索引 183）。
- **图纸：** `sample_AC1032`。现在 rgb 为 #803dc2，透明度为 ByBlock，与 ACadSharp 读取的结果相同；
  该样例中有 14 个图元发生了变化。
- **修改：** 先读 RGB，再读透明度。
- **提交：** [`1c4dcc35`](https://github.com/monyuonyu/libredwg-fixes/commit/1c4dcc35a975e52d734e37a662c5c4222335bd1c) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 2.2 R2007 及以后：配色簿句柄被读了两次，导致丢失图层

- **原因：** 对于 R2007 及以后版本，带标志 0x40 的图元颜色的 DBCOLOR 句柄由 `dwg_decode_common_entity_handle_data`
  读取（"deferred from common_entity_data"）。但 `common_entity_data.spec` 也从同一个句柄流读取了它，
  因此延后的那次读取消耗掉了下一个句柄，也就是图层。
- **表现：** 使用配色簿颜色的图元最终落在图层 NULL 上（显示为 [0, 0]）。
- **图纸：** `sample_AC1021` … `sample_AC1032` 中句柄为 99F 的 CIRCLE（应为图层 16）。
- **修改：** 解码 R2007 及以后版本时跳过前面那次读取。
- **提交：** [`f18f7216`](https://github.com/monyuonyu/libredwg-fixes/commit/f18f721636605498a7afa29410f9dcbd83b27e00) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 2.3 写入 R2000 的颜色：ByBlock、AutoCAD 的 ACI 后备值、ACI 调色板

- **原因：** `bit_downconvert_CMC` 把 R2004 及以后的图元颜色（ENC）当作其他带方法字节的颜色来处理。
  而 ENC 没有方法字节；其索引表示 ByBlock（0）或 ByLayer（256）。真彩色被简化为以其 RGB 的低字节作为索引。
  此外，ACI 调色板 `rgb_palette` 的大多数条目与 AutoCAD 不同（颜色 8 是 65,65,65）。
- **表现：** 在 R2000 输出中，每个 ByBlock 图元都变成了 ByLayer，真彩色得到的 ACI 索引与其颜色毫无关系。
- **图纸：** AutoCAD 自己将同一批图纸保存为 R2000 的版本；AutoCAD 写出的是与真彩色一起保存的 ACI 索引
  （#803dc2 对应 183）。
- **修改：** 保留 ByBlock 和 ByLayer；真彩色保留与之一起保存的 ACI 索引（AutoCAD 的后备值），
  否则用最接近的 ACI（`dwg_nearest_color_index`）；调色板采用 AutoCAD 显示的颜色（ezdxf 的值；ACadSharp 在 256 个中有
  251 个一致）。`example_2018` 写成 R2000 后，全部 222 个图元都与 AutoCAD 保存的 R2000 一致。
- **提交：** [`ecad64a8`](https://github.com/monyuonyu/libredwg-fixes/commit/ecad64a88311475a4a1c5234d3c7dc56c948efba) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 2.4 DXF：真彩色（420–429）按 24 位 RGB

- **原因：** DXF 组码 420 是纯 RGB 值（AutoCAD 写出 3355443）。`out_dxf` 却用 `%li` 写出了带 0xC2 方法字节的
  DWG 值（3258135347）。
- **表现：** 420 的值不是 RGB；在 Windows 上（32 位 `long`）甚至是负数，因此 Linux 和 Windows 上的 `dwg2dxf`
  写出的 DXF 不同。
- **修改：** 取低 24 位，用 int32 格式写出。现在 Linux 和 Windows 写出的 DXF 相同。
- **提交：** [`8f1650ad`](https://github.com/monyuonyu/libredwg-fixes/commit/8f1650ad30a8c7364b590c339e12dbec91d479ba) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 2.5 DXF：DIMSTYLE 的颜色 176–178 是颜色索引

- **原因：** `dxf_CMC` 把 DXF 组码在 90 及以上的所有颜色都按 int32 RGB 写出，而这种写法只适用于 90–99。
- **表现：** R2004 及以后图纸的 DIMCLRD、DIMCLRE 和 DIMCLRT 被写成 `176 -1037268965`，读回时变成了别的颜色；
  真彩色还被写成了组码 534。
- **修改：** RGB 只用于 90–99，420 真彩色只用于 62–67；其余与 AutoCAD 一样写颜色索引。
- **提交：** [`96b39361`](https://github.com/monyuonyu/libredwg-fixes/commit/96b393619e800e438e465e3687973ed9a5b21d28) — 尚未提交给上游。

### 2.6 R13–R2000 升级到 R2004 及以后：AutoCAD 保存在 EED 中的真彩色

- **原因：** AutoCAD 把带真彩色的图元保存为 R2000 或更早版本时，会把真彩色保存在 APPID 为 ACAD 的 EED 中
  （"RTTcAl"，70 为标志和索引，71 为 RGB，以及颜色名称），读取该图纸时再恢复。LibreDWG 忽略了它。
- **表现：** `sample_AC1015` 写成 R2004 后，有 21 个图元的颜色与 AutoCAD 自己保存的 `sample_AC1018` 不同。
- **修改：** 从 R2004 之前的版本写出 R2004 及以后版本时，由 `upconvert_true_colors` 恢复这些颜色。
- **提交：** [`ce5e98f8`](https://github.com/monyuonyu/libredwg-fixes/commit/ce5e98f882ec54d26185959a34a031ea1f96f04a) — 尚未提交给上游。

DXF 中的 TABLESTYLE 颜色见 [6.3](#63-tablestyler2010-及以后的所有单元格样式r2007-的行样式)。

## 3. 文本和代码页

### 3.1 亚洲代码页：双字节字符的首字节

- **原因：** `bit_utf8_to_TV` 把双字节字符的首字节写成了 `wc1 >> 4` 而不是 `wc1 >> 8`：0xB8F4（U+9694 的 GBK 编码）
  变成了 8F F4。在 3 字节 UTF-8 的分支中，代码页里的单字节字符（CP932 的半角片假名）被加上了 0x00 首字节。
- **表现：** 所有从 UTF-8 转换为此类代码页的中文、日文或韩文字符串（例如通过 `dwg_add_u8_input`、JSON 导入，
  或 DXF 转 R2000）都成了乱码，而半角片假名字符会截断字符串。单元测试期望的是 시험 在 CP949 中的错误字节
  （DC C3 7E E8）；正确的字节是 BD C3 C7 E8，与 Python 和 Windows 的转换结果相同。
- **修改：** 改为 `>> 8`，单字节字符不加首字节；测试改为期望 CP949 的真实字节（该 PR 还增加了 ANSI_932 的测试）。
- **提交：** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504)（部分）、[`45e6b50f`](https://github.com/monyuonyu/libredwg-fixes/commit/45e6b50f615aa7346827f71a26de67dc3a8f0eae) — PR
  [#1440](https://github.com/LibreDWG/libredwg/pull/1440)。

### 3.2 R2007 及以后的 DXF 转 R2000：把 UTF-8 文本转换为图纸代码页

- **原因：** R2007 及以后的 DXF 是 UTF-8，但 `dxf_read_pair` 和 dynapi 原样复制 TV 字符串，
  因此 R2000 的 DWG 中写入的是原始的 UTF-8 字节。
- **表现：** R2007 及以后的 DXF 转出的 R2000 中文本是乱码。
- **图纸：** `gh109_1`（代码页 ANSI_936）。
- **修改：** 输出 R2000 时转换为图纸的代码页，代码页中没有的字符与 AutoCAD 一样写成 `\U+XXXX`；
  MTEXT 的文本块也同样处理（仅限目标为 R2007 及以后的情况）。
- **提交：** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504)（部分） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 3.3 DXF 转 R2007 及以后：名称只剩第一个字符

- **原因：** R2007 及以后的编码器把 DXF 中的字符串当作 UTF-8 处理（`bit_write_T`），但当目标版本为 R2007 及以后时，
  `in_dxf` 和 dynapi 把其中大部分做成了 UTF-16（TU）字符串。再按 UTF-8 写出时，它们在第一个 0 字节处被截断。
  主要修复之后，`add_HATCH` 和 `add_MULTILEADER` 对渐变颜色名称和 MULTILEADER 的默认文本仍然这样处理。
- **表现：** 名称只剩第一个字符（块 ALIGNMENT_PARAMETER 变成 "A"，EED 的 APPID 也是如此），
  ACadSharp 丢失了这些块中的图元。DXF 转 R2010–R2018 时，59,883 个图元丢失了 11,007 个，改变的值有 47,757 个。
- **修改：** `dxf_read_file` 按 R2004 读取 DXF（这样字符串保持 UTF-8），然后再设回 R2007 及以后的版本；
  HATCH 和 MULTILEADER 依据 DWG 的版本进行判断。
- **提交：** [`f8cb428f`](https://github.com/monyuonyu/libredwg-fixes/commit/f8cb428f7ae7850f9bd9d3810264aea75fb00445)、[`061d4164`](https://github.com/monyuonyu/libredwg-fixes/commit/061d41647cdfdbaf000d2dd4d04c3a9aa6f79002) — 尚未提交给上游。

写入 R2000 的 UTF-16 EED 字符串见 [4.3](#43-写入-r2000-的-r2007-及以后的-eed-字符串)；R2004 及以后文件的字符串编码见 [13.10](#1310-r2004-及以后文件中的字符串t16tu-和字符串流)。

## 4. 扩展图元数据（EED）

### 4.1 EED 图层（1003）：大端序的句柄，DXF 中是名称

- **原因：** EED 代码 3（DXF 1003）是 LAYER 句柄，与图元句柄（代码 5）一样按大端序存储。却被按小端序读取。
  在 DXF 中，1003 是图层的*名称*，但 `out_dxf` 写出的是句柄编号，而 `in_dxf` 完全没有实现 1003。
- **表现：** AutoCAD 的 `sample_AC1015` 中，LAYER app_layer 8F5 存为 `00..00 08 F5`；却被读成了 0xF508000000000000。
- **修改：** 解码器和编码器都按大端序处理；DXF 写出图层名称，导入时把名称解析为对应的图层。
- **提交：** [`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8)（部分） — 大端序位于 PR
  [#1442](https://github.com/LibreDWG/libredwg/pull/1442)，DXF 名称位于 PR
  [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 4.2 DXF 的 EED 二进制数据（1004）被解码了两次

- **原因：** `dxf_read_binary` 已经把 1004 的十六进制转换成了字节；`add_eed` 又解码了一次。
- **表现：** 出现 "Invalid hex string member"，ASCII DXF 中所有对象的 EED 二进制数据都丢失了。
- **修改：** 只解码一次。
- **提交：** [`3d495876`](https://github.com/monyuonyu/libredwg-fixes/commit/3d49587605124359218d042037bd1fa2e5fdd80d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 4.3 写入 R2000 的 R2007 及以后的 EED 字符串

- **原因：** 处理 UTF-16 EED 字符串的 R13–R2007 分支用 `bit_write_RS` 写出代码页（ANSI_1252 即 30，变成了 0x1E00；
  它应是大端序），并且长度是 UTF-16 字符数，而不是转换后字符串的长度（U+00FF 以上的字符会变成 `\U+XXXX`）。
  缓冲区检查的条件写反了，还会发生下溢。
- **表现：** R2013 图纸写成 R2000 后，ACadSharp 无法读取其中的 STYLE（字体名称在 EED 中）。
- **修改：** 代码页按大端序，使用转换后的长度，缓冲区检查改正。
- **提交：** [`438e9817`](https://github.com/monyuonyu/libredwg-fixes/commit/438e98179131cba1be9732e42cec26cd4b8faa54) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

## 5. MTEXT、ATTRIB 和 ATTDEF

### 5.1 R2018 的 MTEXT：只有边框位时也有背景字段

- **原因：** 当 `bg_fill_flag` 只设置了位 0x10（文字边框，R2018 及以后）时，背景字段（比例、颜色、透明度）同样存在。
  旧的判断 `version <= R_2018 ? 1 : 16` 对有边框但无填充的 R2018 文件跳过了这些字段。
- **表现：** 这种 MTEXT 后面的所有字段都读错了。
- **修改：** 与 ACadSharp 一样，检查位 0x01，自 R2018 起也检查位 0x10。
- **提交：** [`3ab77f4d`](https://github.com/monyuonyu/libredwg-fixes/commit/3ab77f4d123107fd17bddd725865dd1e1dad9039)（部分） — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 5.2 R2018 多行 ATTRIB/ATTDEF：嵌入的 MTEXT

- **原因：** 当 `mtext_type` 为 2 或 4 时，"all fields of an embedded MTEXT object are written, starting from the Entmode"
  （ODA 20.4.4）。LibreDWG 对这个嵌入部分的布局与 AutoCAD 的文件不符。根据在 AutoCAD 的 R2018 样例上的实测以及
  ACadSharp 的读取方式：公共图元部分包含所有者句柄、反应器、xdic、`has_ds_data`、完整的 ENC 颜色（RGB 在透明度之前），
  以及按 ltype、material、shadow、plotstyle 顺序排列的标志及其句柄；MTEXT 中 `extents_height` 在 `extents_width` 之前，
  背景字段在位 0x10 时也存在，还有 R2018 的注释和分栏字段；ATTRIB/ATTDEF 的 STYLE 句柄位于嵌入 MTEXT 的句柄之前；
  MTEXT 之后是 `annotative_data_size`，若不为 0，则后跟数据、注册应用程序和一个 BS。
- **表现：** MTEXT 的图层指向了 STYLE Standard，ATTRIB 的图层也是错的。
- **图纸：** `sample_AC1032`（ATTDEF 796、ATTRIB 79D）。现在数据正好结束于字符串流处，所有句柄都指向 LAYER 0、
  LTYPE ByLayer、MATERIAL ByLayer 和 STYLE Standard。在 22 个 ACadSharp 样例上与 ACadSharp 的值差异：10 → 0。
- **修改：** 采用上述布局，为这些句柄新增结构体字段；在 JSON 中这些字段写成 `mtext.<field>`
  （以前像 `ins_pt` 这样的扁平键会与 ATTRIB 自身的键冲突）。GEOPOSITIONMARKER 使用同一个宏，并保留其原有的尾部字段；
  这部分未经验证，因为没有样例。
- **提交：** [`a1c9eba4`](https://github.com/monyuonyu/libredwg-fixes/commit/a1c9eba4034cd71f53533a1e134c43337c5d15aa) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443)。

### 5.3 R2018 多行属性写成较早版本

- **原因：** R2018 之前没有多行属性。AutoCAD 保存的 R2010 和 R2013 中，属性文字在 `text_value` 中；
  R2018 中 `text_value` 为空，文字只在嵌入的 MTEXT 中。写出较早版本时保留了空的 `text_value`。
- **表现：** 在 R2013 及更早版本的输出中，多行属性丢失了文字。
- **图纸：** `sample_AC1024`、`sample_AC1027` 对照 `sample_AC1032`。
- **修改：** 当 `text_value`（ATTDEF 为 `default_value`）为空时，使用去掉格式代码后的 MTEXT 文字。
  未能复现的部分：AutoCAD 去掉哪些代码、`\P` 变成什么（这里变成空格），以及 AutoCAD 在扩展字典中保存的往返数据。
- **提交：** [`41f07038`](https://github.com/monyuonyu/libredwg-fixes/commit/41f0703840f8d81ee349f58e389199f1c73e023f) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443)。

### 5.4 R2010 及以后的 ATTRIB：没有 `keep_duplicate_records`

- **原因：** 在 AutoCAD 的 R2010、R2013 和 R2018 文件中，ATTRIB 的数据流在 `lock_position_flag` 之后立即结束。
  只有 ATTDEF 在提示文字之前有重复的版本 RC（这里即 `keep_duplicate_records`）；ODA 20.4.4 和 ACadSharp 的说法一致。
- **表现：** 读取时出现 "Invalid ATTRIB.keep_duplicate_records 64/65/72"，写出时多出一个字节。
- **图纸：** `sample_AC1024`、`sample_AC1027`、`sample_AC1032`。
- **修改：** 自 R2010 起不再出现在 DWG 中；DXF 和 JSON 中保留。
- **提交：** [`f3432746`](https://github.com/monyuonyu/libredwg-fixes/commit/f343274668fd1e997a47a4f8b83832e22ec26813) — PR [#1443](https://github.com/LibreDWG/libredwg/pull/1443)。

### 5.5 DXF 导入：ATTRIB/ATTDEF 中嵌入的 MTEXT，MTEXT 的分栏部分

- **原因：** `in_dxf` 既不读取 R2018 多行 ATTRIB/ATTDEF 在 `<101, Embedded Object>` 之后嵌入的 MTEXT，
  也不读取 MTEXT 的注释/分栏部分，其中的 40（矩形宽度）覆盖了文字高度。
- **表现：** `dxf2dwg` 处理 `sample_AC1032` 时失败（"Invalid DXF code 46"）；36 个 MTEXT 的高度变成了 0。
- **修改：** 把这两部分读入各自的字段。
- **提交：** [`69d49925`](https://github.com/monyuonyu/libredwg-fixes/commit/69d499258e99d7f12ca58555fd5ada3a001f6504)（部分） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 5.6 R13/R14 的 MTEXT 写成 R2000 及以后：行距为 1，而不是 0

- **原因：** R13/R14 的 MTEXT 没有行距字段；写出 R2000 及以后版本时写成了 0。
- **表现：** 每个 MTEXT 的各行都叠在了一起（`sample_AC1014` 和 `example_r14` 写成 R2000 时的 55 个 MTEXT）。
- **修改：** 使用默认值，行距样式 1，行距系数 1.0。
- **提交：** [`6b26f48d`](https://github.com/monyuonyu/libredwg-fixes/commit/6b26f48deb27612c4013109522f92cc67e41ce9c) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 5.7 MTEXT 的换行写成 `\P`

- **原因：** 较新图纸的 MTEXT 中可能包含单纯的 LF 换行。AutoCAD 在 R2000 DWG 和 DXF 中把它们写成 `\P`；
  LibreDWG 写出的 DXF 中却是 `^J`。
- **表现：** DXF 读取程序把 `^J` 当作文字保留了下来。
- **图纸：** `sample_AC1018` … `sample_AC1032`，对照 AutoCAD 保存的 R2000。
- **修改：** 对 R2000 DWG 和 DXF，由 `dwg_fixup_mtext_linebreaks` 把 LF 转为 `\P`。
- **提交：** [`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40)（部分） — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 5.8 DXF 导出：SEQEND 之后的 ATTRIB 和 VERTEX

- **原因：** 对于 R13–R2000，`out_dxf` 把 INSERT 的属性和 POLYLINE 的顶点当作“下一个对象”来查找。
  但属性之间可能夹着一个 SEQEND。
- **表现：** DXF 丢失了 SEQEND 之后的所有属性。
- **图纸：** `sample_AC1015`（ATTRIB 79D、SEQEND 79E、ATTRIB 79F、7A0）。
- **修改：** 沿着图元链接（下一个图元的句柄）查找。
- **提交：** [`77fec52b`](https://github.com/monyuonyu/libredwg-fixes/commit/77fec52b4abc92612188613f9cb7a8192bfb6957) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 5.9 属性标记：除空格和 `!` 以外的任意字符

- **原因：** `dwg_is_valid_tag` 只接受字母、数字、`$`、`_` 和 `-`；AutoCAD 接受除空格和 `!` 以外的任意字符。
  仅限 ASCII 的那个版本还接受空标记。
- **表现：** ATTDEF 的标记 `SPACEOBJECTS:NUMBER` 被拒绝，经由 DXF 时丢失。
- **图纸：** `aec_objects`。
- **修改：** 采用 AutoCAD 的规则；空标记视为无效。
- **提交：** [`f9b9f144`](https://github.com/monyuonyu/libredwg-fixes/commit/f9b9f14442df3966abda9e1d856c61745a1679b4) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

## 6. TABLE 和 TABLESTYLE

### 6.1 TABLE/TABLECONTENT：布局，以及在普通构建中可读取

- **原因：** TABLE 和 TABLECONTENT 只在调试构建中编译，而且其布局在很多地方是错的。以每个表格的数据必须结束的位置为准
  进行了核对（R2000/R2004 在句柄流处，R2007 及以后在 bitsize − 17 − 字符串数据大小处）：
  - TABLE 在所有版本中都派生自 INSERT；R2010 及以后的形式同样以插入数据和句柄开头，然后是 RC、H、BL、B（R2010）
    或 BL（R2013 及以后）、表格内容、表格样式句柄、BS、3BD 以及分割数据（ODA 20.4.96.2，ACadSharp `readTableEntity`）。
  - TABLECONTENT（20.4.97）：列有自定义数据集合、单元格样式 ID 和 BD 宽度；单元格有自己的单元格样式替代；
    行高是 BD；表格单元格样式替代位于合并范围之前；几何标志非零时有一个单元格几何数据；表格样式句柄位于末尾。
  - R2010 之前的单元格：文本字符串只在 R2007 之前、且 344 句柄为 0 时存在；块单元格有一个 has-attdefs 位和一个 BS 计数；
    单元格对齐方式是 BS 而不是 RS；0x10 有自己的文字样式句柄；R2007 中每个单元格都有 BL + 值；
    表格替代位 0x2000 和 0x40000 之后没有额外的句柄。
  - 值（20.4.99）：只有标志位 0x01（而不是 0x03）时才为空；自 R2007 起，字符串值是 BL 字节数加 UTF-16LE 字节，
    而不是 TV。
  - 通用的 REPEAT 大小检查拒绝了 R2010 之前的单元格。
  - 在 `-v3` 跟踪输出中，`strrplc` 遇到很长的嵌套字段名时触发断言。
- **表现：** 在普通构建中完全不读取表格；在调试构建中数据越过了其末尾。
- **图纸：** `sample_AC1014` … `sample_AC1032`、`sample_base`、`example_r14` … `example_2018`、
  `2000/TS1`。
- **修改：** 采用上述布局；TABLE 和 TABLECONTENT 从 DEBUGGING 移到 UNSTABLE，因此在普通构建中也会读取。
  句柄指向预期的对象。
- **提交：** [`1d92022d`](https://github.com/monyuonyu/libredwg-fixes/commit/1d92022d6a0c4af2c927794e2689f512cf107894) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.2 TABLECONTENT：R2007 之前的 kGeneral 值，计数检查

- **原因：** kGeneral（512）值也出现在 R14–R2004 的 TABLECONTENT 对象中（AutoCAD 2008 的往返数据），
  与 R2007 起的形式一样是 BL 字节数 + 字节；读取程序在那里停了下来。REPEAT 的大小检查拿 `count * sizeof(struct)`
  与可用的位数比较。
- **表现：** 出现 "kGeneral <r2007"，合法表格的最后几行被拒绝。
- **修改：** 在所有版本中读取 kGeneral；按每个元素的最小位数进行检查。测试集中全部 24 个表格和 9 个 TABLECONTENT
  对象都在应有的位置结束，全部 24 个表格逐字段与 ACadSharp 一致。
- **提交：** [`b9780ac7`](https://github.com/monyuonyu/libredwg-fixes/commit/b9780ac7a0b3b1c995fbc37f17a3d798901db081) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.3 TABLESTYLE：R2010 及以后的所有单元格样式，R2007 的行样式

- **原因：** R2010 及以后：在表格单元格样式之后，单元格样式 _TITLE、_HEADER 和 _DATA 以数组形式出现，每个前面有一个
  BL 索引（与 ACadSharp `readTableStyle` 相同）；但只读取了第一个。R2007：行样式在边框之后有值的数据类型、单位类型和
  格式字符串（20.4.101.1），这些没有读取；`format_string` 是 T 而不是 TU。写出 R2000 时，编码器在写出页边距之后才转换
  表格样式，因此页边距为 0。DXF 导入把 TABLESTYLE 的颜色当作调色板 RGB，把边框线宽当作图元线宽索引。
- **表现：** `example_2018` 的 TABLESTYLE 0x29B 在第 310.4 位结束，剩下 6688 个未知位；`example_2007` 和
  `sample_AC1021` 的第 1、2 行是乱码。
- **修改：** 读取整个数组；重写 `downconvert_TABLESTYLE`（数据行、标题行和表头行样式分别取自 _DATA、_TITLE 和
  _HEADER，页边距和流向取自表格单元格样式，“无”背景写成无背景的 ACI 7，边框不可见标志取反）；DXF 颜色按表格颜色处理
  （ACI 7 为 0xc3000007），线宽按原值，420–427 按 24 位 RGB；在 JSON 中 `ovr` 是数组。现在 `example_2000` …
  `example_2018` 的 DXF TABLESTYLE 与 AutoCAD 的 DXF 相同，R2000 输出也与 AutoCAD 保存的 R2000 相同。
- **提交：** [`6a3f2d0c`](https://github.com/monyuonyu/libredwg-fixes/commit/6a3f2d0cbab6e9a5748e030d64824362c17abe14) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.4 R2010 及以后的表格以旧格式写入 R2000/R2004

- **原因：** 自 R2010 起，表格把单元格保存在表格内容中（20.4.97）；而 R2000 和 R2004 需要旧的单元格数据（20.4.96.1）。
  之前没有任何转换。
- **表现：** R2010 及以后图纸中的表格在 R2000 输出中丢失。AutoCAD 在其 R2000 文件中保留了 ACAD_TABLE
  （`sample_AC1015`、`example_2000`）。
- **修改：** 由 `downconvert_TABLE` 构建旧数据：行、列、宽度、高度、合并范围、单元格文字（格式化后的值 302，否则为字符串）、
  块单元格，以及每个内容的文字高度、对齐方式和颜色替代。边框、行/列/表格单元格样式以及字段不做转换。
  `flag_for_table_value` 取自 R2010 及以后的一个取值相同的未知 BS；认为两者是同一字段只是推测。
  `sample_AC1024/1027/1032`、`sample_base` 和 `example_2010/2013/2018` 的 R2000 输出与 AutoCAD 的 R2000 文件有相同的
  11 个表格。
- **提交：** [`7139a423`](https://github.com/monyuonyu/libredwg-fixes/commit/7139a423964adff8eef80da0cb1d6daa192e07c0) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.5 R2007 的表格写入 R2000/R2004：单元格文字

- **原因：** R2007 把单元格文字保存在值中（UTF-16），`text_value` 为空。AutoCAD 保存 R2000 时优先写出格式化后的值文字（302）。
- **表现：** 写成 R2000 后，每个单元格都丢失了文字；数字丢失了格式。
- **图纸：** `sample_AC1021` 对照 AutoCAD 的 `sample_AC1015`。
- **修改：** 单元格文字取自值，优先使用格式化后的文字（"0.500000"、"$100.00"、"CELL FORMAT 1"）。
  现在这些表格与 AutoCAD 保存的 R2000 相同。
- **提交：** [`4f819653`](https://github.com/monyuonyu/libredwg-fixes/commit/4f8196538d897b971893cd585d38a1d47e2371a7)、[`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb)（部分） — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.6 ACAD_TABLE 的 DXF 导入与导出

- **原因：** 表格不会写入 DXF，也无法导入。当 TABLE 变为 UNSTABLE 时（[`d1f8fb4c`](https://github.com/monyuonyu/libredwg-fixes/commit/d1f8fb4c7e729e577e4d005e699a495f490f01a8)），`in_dxf` 遇到未知的 ACAD_TABLE
  组码会失败，`out_dxfb` 则用 DWG 字段写出表格；因此那个提交先让 DXF/DXB 的表格保持禁用。
- **表现：** 经由 DXF（`dwg2dxf`、`dxf2dwg`）时丢失了所有表格（测试集中共 18 个）。
- **修改：** AutoCAD 在所有 DXF 版本中都以 R2010 之前的形式写出 ACAD_TABLE（INSERT 部分；342 343 11 90 91 92 93–96；
  行高 141 在列宽 142 之前；每个单元格 171–176、177、178、145，然后在 R2007 之前是文字 1，自 R2007 起是值；
  R2010 及以后再加上 280 0）。`dxf_ACAD_TABLE` 按此写出（R2010 及以后的表格与写 R2000 时一样转换），
  `add_ACAD_TABLE` 按此读取。`example_2000/2007/2018` 的输出除句柄、匿名块名以及 R2018 的标志 90 和 0x40000
  单元格标志位以外，与 AutoCAD 的 DXF 相同。单元格替代和表格替代沿用 DWG 字段的顺序；没有任何 AutoCAD 样例包含它们，
  因此其 DXF 顺序只是推测。之后写出代码移到了 `out_dxf_table.inc`，使二进制 DXF（`dwg2dxf -b`）也能写出表格。
- **提交：** [`d1f8fb4c`](https://github.com/monyuonyu/libredwg-fixes/commit/d1f8fb4c7e729e577e4d005e699a495f490f01a8)、[`56cd3c4f`](https://github.com/monyuonyu/libredwg-fixes/commit/56cd3c4fe139abf172696e8ca31093db1936b3df)、[`0f3a1ac7`](https://github.com/monyuonyu/libredwg-fixes/commit/0f3a1ac7e14820b28ecaf16841ad588b439bc51f) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.7 按 AutoCAD 的布局写出 DXF 的 TABLESTYLE

- **原因：** LibreDWG 在 DXF 中写出的是 R2010 及以后的单元格样式布局（TABLEFORMAT_BEGIN …）。AutoCAD 在所有版本中都写出
  R2007 的布局：三个行样式（7 140 170 62 63 283 [90 91 1] 以及 6 条边框），R2010 及以后前面再加 280 0。
  R13/R14 的 DXF 则丢弃了 TABLESTYLE。
- **表现：** ACadSharp 无法读取 TABLESTYLE，表格经由 DXF 时丢失了表格样式；`sample_AC1014` 有表格及其 TABLESTYLE，
  它们丢失了样式名 STANDARD。
- **修改：** 所有版本都使用 R2007 的布局；图纸中有 TABLESTYLE 时，R13/R14 的 DXF 也写出它。
- **提交：** [`61f6d7ec`](https://github.com/monyuonyu/libredwg-fixes/commit/61f6d7ecaad2cda7d54a5857fa2aceaac79c3a33)、[`e1835040`](https://github.com/monyuonyu/libredwg-fixes/commit/e18350406d9dd7784bf7c547bcd9ad9786b164ac) — PR [#1446](https://github.com/LibreDWG/libredwg/pull/1446)。

### 6.8 从较早版本和从 DXF 写入 R2010 及以后的表格

- **原因：** R2010 及以后的 TABLE 需要表格内容；R2000–R2007 表格的单元格，以及从任何 DXF 读取的表格
  （`in_dxf` 读取的是 R2010 之前的形式）的单元格，都没有转换。来自 DXF 的 R2007 单元格值没有 `data_size`。
- **表现：** R14–R2007 图纸中的 13 个表格写成 R2010 及以后时没有行、列和单元格，水平方向为 0；
  R2010 及以后的 DXF 写成 R2010 及以后时表格为空（测试集中 11 个）；ACadSharp 拒绝了大小为 0 的三维点值
  （"Could not read ACAD_TABLE"，即 `sample_AC1021` 经由 DXF 时的表格）。
- **修改：** 新增 `upconvert_TABLE`，即 `downconvert_TABLE` 的逆过程（列、行、每个单元格一个内容、替代、合并范围、
  水平方向、一个分割行范围），也适用于 DXF 输入；设置二维/三维点值的大小，没有数据的日期写成其文字。
- **提交：** [`7fb484ed`](https://github.com/monyuonyu/libredwg-fixes/commit/7fb484ed0d13655ed22dce0ab8678c170339a142)、[`03c45bf5`](https://github.com/monyuonyu/libredwg-fixes/commit/03c45bf52eff38e84da496b20f4bc508e034dd95)、[`3ec9d114`](https://github.com/monyuonyu/libredwg-fixes/commit/3ec9d1148f71d8a2a0fd07fbe044f33ef17d39c5) — 尚未提交给上游。

### 6.9 从较早版本写出 R2007 及以后时的内存泄漏

- **原因：** `upconvert_TABLESTYLE` 创建的 `ovr` 数组（源版本的释放规范不知道它），以及 `add_DUMMY_eed` 的 UTF-16 名称，
  始终没有被释放。
- **表现：** 把 `example_2000` 写成 R2018 时 LeakSanitizer 报告泄漏。
- **修改：** 两者都释放。
- **提交：** [`10a1a752`](https://github.com/monyuonyu/libredwg-fixes/commit/10a1a752e71debbe348e500c2da9daced8b1dec1) — 尚未提交给上游。

## 7. ACIS 实体（3DSOLID、REGION、BODY）

### 7.1 读取：自 R2013 起没有 ACIS 空位，R2007 及以后的未知 BL

- **原因：** 自 R2013 起，SAB 存放在 AcDsPrototype_1b 段中，图元以线框数据开头；LibreDWG 却先读了一个 ACIS 空位，
  使所有内容错位一位。此外，ODA 20.4.41 在空位之后列有 "R2007+: Unknown BL"，这个字段没有读取。
- **表现：** AutoCAD 的 R2018 3DSOLID D6A 有 1274288069 条素线；现在它有点 (31.69, −220.11, 2.19) 和 4 条素线，
  与 ACadSharp 读取的结果相同。这个乱码的 3DSOLID 还破坏了 R2000 输出：ACadSharp 丢失了模型空间中位于它之后的 9 个图元。
- **图纸：** `sample_AC1032`。
- **修改：** 自 R2013 起没有空位；新增字段 `unknown_2007`。149 张图纸中全部 107 个 ACIS 图元现在都正好结束于其 bitsize
  （以前是 100 个）。有一个测试按地址比较修订 GUID 数组，这永远不会相等；以前因为 GUID 读错，从未执行到那里。
  现在改为比较内容。
- **提交：** [`faa9c595`](https://github.com/monyuonyu/libredwg-fixes/commit/faa9c59576683f2931117b4cd6edb54b4adc7ca2)、[`aad393ec`](https://github.com/monyuonyu/libredwg-fixes/commit/aad393ec206b1116694db8ae7877874207530709) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 7.2 写出：空位、历史句柄、材质

- **原因：**
  - 编码器对 R2013 及以后仍然写出 ACIS 空位（[`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)）。
  - R2013 及以后 3DSOLID 的历史句柄（H 350）既不读取也不写出（AutoCAD 的 R2018 3DSOLID 的句柄流中剩下 26 位）
    （[`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)）。
  - 它还作为一个多余的空句柄被写入了 R2004 和 R2000 的 DWG；而 AutoCAD 的 R2004 图纸中没有它（[`81d14b4c`](https://github.com/monyuonyu/libredwg-fixes/commit/81d14b4c8e628be99b3deabcce0ef3e773c9b580)）。
  - 对于 R2007 及以后，它只在有 SAB 数据时写出，因此 R14–R2004 图纸中的 SAT 3DSOLID 缺少它（[`d7da6276`](https://github.com/monyuonyu/libredwg-fixes/commit/d7da62767fb3af37edb5cce653ac7115e36917df)）。
  - AutoCAD 的 R2007–R2010 3DSOLID 和 REGION 在空位之后只有那个未知 BL：`num_materials` 读走了这个 BL，
    而未知 BL 又从句柄流中读了 34 位；并且历史句柄只存在于 3DSOLID 中，REGION 中没有（[`d59ae195`](https://github.com/monyuonyu/libredwg-fixes/commit/d59ae195c6d89b813ae3a90e1a14b9ceb00dbb76)）。
- **表现：** R2013 及以后的图纸重新写成 R2013 及以后时，ACadSharp 无法读取其中的 3DSOLID 和 REGION
  （丢失了 5 个 3DSOLID 和 4 个 REGION）；SAT 3DSOLID 写成 R2010 时，ACadSharp 读到了句柄之外
  （"Index was outside the bounds of the array"）；`example_2010` 的 3DSOLID 多出了 34 位。
- **图纸：** `example_2004`、`sample_AC1018`、`example_2010`、`example_2013`、`example_2018`。
- **修改：** 与 AutoCAD 相同：自 R2013 起没有空位；历史句柄用于所有 R2007 及以后的 3DSOLID，且只用于 3DSOLID；
  DWG 中不写材质。`example_2010`、`example_2013` 和 `example_2018` 重新写成相同版本时，其中的 3DSOLID 和 REGION
  现在与 AutoCAD 的字节相同。
- **提交：** [`81d14b4c`](https://github.com/monyuonyu/libredwg-fixes/commit/81d14b4c8e628be99b3deabcce0ef3e773c9b580)、[`f74671e2`](https://github.com/monyuonyu/libredwg-fixes/commit/f74671e29512976fb1bf3823f15b022412b5865f)、[`d7da6276`](https://github.com/monyuonyu/libredwg-fixes/commit/d7da62767fb3af37edb5cce653ac7115e36917df)、[`d59ae195`](https://github.com/monyuonyu/libredwg-fixes/commit/d59ae195c6d89b813ae3a90e1a14b9ceb00dbb76) — 尚未提交给上游。

### 7.3 R2013 及以后的 AcDs 段：按句柄读取 SAB

- **原因：** R2013 及以后的 REGION、3DSOLID、BODY 和 SURFACE 的 SAB 位于 AcDb:AcDsPrototype_1b 段的数据记录中。
  LibreDWG 搜索 "ACIS BinaryFile"，并按顺序把数据交给带有 `has_ds_data` 的图元。但 AutoCAD 的 R2018 文件中是
  "ASM BinaryFile4"，而且记录的顺序与图元的顺序不同。段索引的检查使用的是结构体大小 16，而不是条目的 12 字节。
- **表现：** AutoCAD 的 R2018 文件中的 ACIS 数据一个都没有读到（`example_2018`、`sample_AC1032`）；
  SAB 可能分配给错误的图元（`example_2013`）；`sample_base` 的 AcDs 被拒绝。
- **修改：** 从数据索引中按句柄取出数据记录，也包括 blob 页中的记录。
- **提交：** [`eb3b2553`](https://github.com/monyuonyu/libredwg-fixes/commit/eb3b25530662bec56bebb0d045d614aff48be9be) — 尚未提交给上游。

### 7.4 R2013 及以后的 AcDs 段：写出

- **原因：** AcDs 段无法编码（`acds.spec` 被禁用）。
- **表现：** R2013 及以后的图纸重新写成 R2013 及以后时丢失了全部 ACIS 数据；R2004–R2010 的图纸写成 R2013 及以后时也是如此。
- **修改：** 首先，保留解压后的段，从 R2013 及以后写到 R2013 及以后时原样写出（[`60d4a3e0`](https://github.com/monyuonyu/libredwg-fixes/commit/60d4a3e07f870ea31c94f4990f7b60c0ab51c141)）。
  然后按照 AutoCAD 的 `example_2018` 的样子写出该段：按句柄组织的 SAB 记录（超过 0x40000 字节时放在 blob01 页中，
  与 ODA 规范一致）、相同的模式和搜索索引，并给图元标上 `has_ds_data`（[`b06ca869`](https://github.com/monyuonyu/libredwg-fixes/commit/b06ca8699845d096d972be21469c42f3ca787a68)）。
- **提交：** [`60d4a3e0`](https://github.com/monyuonyu/libredwg-fixes/commit/60d4a3e07f870ea31c94f4990f7b60c0ab51c141)、[`b06ca869`](https://github.com/monyuonyu/libredwg-fixes/commit/b06ca8699845d096d972be21469c42f3ca787a68) — 尚未提交给上游。

### 7.5 SAT 与 SAB 互相转换

- **原因：** R13–R2000 的 DWG 和 DXF 中，ACIS 数据是 SAT 文本，而 R2013 及以后的 AcDs 只有 SAB 记录。
  另外，旧的面向 DXF 的 SAB 转 SAT 转换（`dwg_convert_SAB_to_SAT1`）用 `%g`（6 位有效数字）写出 double，
  把 long 写成指针 `$n`，逻辑值写错（用 F 代替 forward_v 和 I），头字符串被拆成碎片（"A utodesk A utoCA D"），
  结束标记被截断，版本被改变，没有历史属性，而且对 "ASM BinaryFile4" 什么也不输出。
- **表现：** 这些图纸中的实体写成 R2013 及以后时是空的；DXF 中的 SAT 无法读取，几何形状也发生了变化。
- **修改：** 用一套 ACIS 记录的语法来补充 SAT 所缺少的值类型（long 和整数值的 double 看起来一样，逻辑值是单词），
  这套语法是根据 AutoCAD 对同一批实体写出的 SAT 和 SAB 整理的。`dwg_convert_SAT_to_SAB` 和 `dwg_convert_SAB_to_SAT`
  使用它，double 用能精确表示的最短形式。测试集中全部 15 个 SAT 实体转为 SAB 再转回时得到相同的 SAT，
  全部 83 个 SAB 实体转为 SAT 再转回时得到相同的字节。
- **提交：** [`d207b237`](https://github.com/monyuonyu/libredwg-fixes/commit/d207b2379e50b0163ba401a5639e2fc58c1a0871)、[`2ab21c75`](https://github.com/monyuonyu/libredwg-fixes/commit/2ab21c7563a9f8d75596f55ab4ea1906f0a9050e) — 尚未提交给上游。

### 7.6 DXF：SAT 的行和转义的 `^`

- **原因：** SAT 在 DXF 中以组码 1 的行写出，超过 255 个字符的行在组码 3 中续写。LibreDWG 把 DWG 中加密的 4096 字节块
  逐行写出，但块可能在任意位置把行切断。读取时，1 行的换行符被放在了它的 3 续行之前。加密后的 `A` 是 `^`，DXF 把它写成
  `^ `；导入时在 `acis_data` 中做了反转义，但 `encr_sat_data`（R13–R2010 的 DWG 使用的正是它）中没有。
- **表现：** 位于块末尾的行变成两行，下一个字符被丢掉，于是数字被破坏（在 `example_2000` 中 −0.8944579735636958 变成了
  −0.89 和 44579735636958）；由 DXF 生成的 DWG 中，SAT 的每个 A 都变成了 "A "，并且丢失了结尾。
- **修改：** `dxf_sat_lines` 写出所有块的行，超过 255 个字符的部分放入 3，`^` 写成 `^ `；`add_3DSOLID_encr`
  把换行符放在正确的位置，并对两个副本都做反转义。
- **提交：** [`94a6cf90`](https://github.com/monyuonyu/libredwg-fixes/commit/94a6cf906aa7bd5749254bae8c849216c4627819)、[`bff66044`](https://github.com/monyuonyu/libredwg-fixes/commit/bff660446e6496299f05b67bd6fd353645799e4e) — 尚未提交给上游。

### 7.7 DXF 导入：修订 GUID 的字段名

- **原因：** 修订 GUID 被设置为 `revision_guid[39]` 和 `revision_bytes[9]`，而这两个名称 dynapi 并不认识。
- **表现：** 出现 "Invalid 3DSOLID field"。
- **修改：** 使用真实的字段名。
- **提交：** [`da463d6d`](https://github.com/monyuonyu/libredwg-fixes/commit/da463d6d34deddbfcb3c107a1582bed3d65a4384)（部分） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

## 8. 代理对象和未知对象

### 8.1 未知对象保留其类名

- **原因：** LibreDWG 无法解码的图元和对象全都被命名为 UNKNOWN_ENT / UNKNOWN_OBJ。
- **表现：** JSON 中只显示 UNKNOWN_ENT，看不出对象是什么。
- **修改：** 与其他读取库一样，以其类名作为 dxfname（AEC_WALL、TCH_WALL 等）。
- **提交：** [`1935be09`](https://github.com/monyuonyu/libredwg-fixes/commit/1935be09e0ad6e1cac637427804eb02741afeada)（部分） — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.2 需要时添加 ACDBPLACEHOLDER 类

- **原因：** 不支持的对象会写成占位对象，但 R2007 及以后图纸的 DXF 中没有 ACDBPLACEHOLDER 类。
- **表现：** 每个不支持的对象（TABLEGEOMETRY，以及写入 R2000 时的 MATERIAL）都变成了 DUMMY，并报告 ERROR。
- **修改：** 需要时由编码器添加该类。
- **提交：** [`ebeebb13`](https://github.com/monyuonyu/libredwg-fixes/commit/ebeebb139ac7e446df77e17486c59c6baa3f693d) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.3 未知对象写成 ACAD_PROXY_ENTITY / ACAD_PROXY_OBJECT

- **原因：** 从 R2004 及以后的 DWG 写出 R2000 时，未知的图元和对象（AEC_*、TCH_* 等）变成了厚度为 1e25 的 POINT 或
  PLACEHOLDER。此外，代理的 spec 在图形格式之后又读写了一份图形数据（ODA 和 ACadSharp 中都没有），没有解码所有对象 ID，
  而且 `bit_write_bits` 以错误的顺序写出末尾的位。
- **表现：** 其他应用程序的自定义对象丢失了数据。
- **修改：** 与 AutoCAD 一样把它们写成代理对象（ODA 20.4.90/29）：类编号、源图纸的图形格式、代理图形、直到句柄流为止的
  数据位以及对象自身的句柄；对于 R2007 及以后的源，数据保留字符串流。公共句柄会与解码得到的句柄进行核对；
  不一致时仍采用原来的后备处理。对于 `gh109_1`（R2013，TCH_*），26 个代理对象在类、图形格式（0x6A001F）、
  数据位和字符串位以及对象 ID 上都与 CAD 写出的 `gh109_1.dxf` 一致。
- **提交：** [`d9004510`](https://github.com/monyuonyu/libredwg-fixes/commit/d9004510df29de8f15efbb4c7017d15efb59ef0c) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.4 DXF：未知对象和代理对象

- **原因：** `dwg2dxf` 跳过了未知的图元和对象。图形格式被计算为 `>> 8`，而不是 `version | maint << 16`；
  相对的代理对象 ID（6、8、10、12）没有转换为绝对值。
- **表现：** 经由 DXF（`dwg2dxf`、`dxf2dwg`）时丢失了它们。
- **修改：** 按 AutoCAD 的布局写出它们以及 PROXY_* 对象（90 498/499，91 类编号，95 图形格式，70，AcDbProxyEntity 中的
  图形 92/160 + 310，自 R2007 起数据为 161/162 + 310/311，否则为 93 + 310，对象 ID 330/340/350/360，94 0）；
  `in_dxf` 将其读回（`add_PROXY`）。对于 `gh109_1`，DXF 中全部 26 个代理对象都与 CAD 写出的 DXF 一致，
  只有 8 个的图形除外（我们的取自 DWG）；`gh109_1.dxf` 经过 `dxf2dwg` 得到的代理对象与经由 DWG 的结果相同。
- **提交：** [`5885ae72`](https://github.com/monyuonyu/libredwg-fixes/commit/5885ae72ccb15e2b0768f922ce3d1c705be64cfd) — PR [#1447](https://github.com/LibreDWG/libredwg/pull/1447)。

### 8.5 目标为 R2004 和 R2010 及以后时也生成代理对象

- **原因：** 代理转换只在目标为 R2000 时执行。当源为 R2007 及以后、目标为 R2010 及以后时，代理数据的末尾是原对象的
  字符串流，而不是代理对象自己的。
- **表现：** `aec_objects`（R2018）中的 AEC_WALL 在 R2004 中变成了 POINT，在 R2010 和 R2013 中变成了 POINT 和占位对象。
- **修改：** 目标版本比源版本旧时一律生成代理对象；`encode_proxy_data` 把数据中的字符串放入代理对象自己的字符串流。
- **提交：** [`a970cfee`](https://github.com/monyuonyu/libredwg-fixes/commit/a970cfeecbc350f21b88f60c22205dbb69db445e)、[`5243d4f5`](https://github.com/monyuonyu/libredwg-fixes/commit/5243d4f5610125fdc3bb2d741ba70d21e12cc32b) — 尚未提交给上游。

在 AEC 图纸上发现的类数量上限问题见 [1.2](#12-类的数量上限定得过低)。

## 9. 其他图元和对象

### 9.1 DIMENSION_ANG2LN：点 10 和点 16 被调换

- **原因：** R13 及以后先存储点 16（2RD），然后是 13、14、15，最后是点 10（3BD）（ODA 20.4.27）。
  而 2RD 被读入了 `def_pt`，最后的 3BD 被读入了 `xline2end_pt`。
- **表现：** `dwg2dxf` 把圆弧上的点写成组码 10，把第二条尺寸界线的端点写成组码 16，DXF 读取程序把标注圆弧画在了错误的位置。
- **图纸：** `sample_AC1032`，句柄 516。
- **修改：** 把相同的位读入与相同 DXF 组码对应的字段，并按 DXF 的顺序写出 13–16；在 JSON 中 `xline2end_pt` 保持三维，
  因为 `in_json` 期望的是三维点（否则会出现 "JSON 3DPOINT must be ARRAY of size 3"，导入的其余部分也会失败）。
- **提交：** [`3d450c21`](https://github.com/monyuonyu/libredwg-fixes/commit/3d450c21651ed381fe5ba101aaa7c8e353dca8f7)、[`fee66f28`](https://github.com/monyuonyu/libredwg-fixes/commit/fee66f28e31bb1ca27813672fb665a0278a45a58) — 不在 PR 中：上游 `master` 此后已在自己的提交
  [9bcd0586](https://github.com/LibreDWG/libredwg/commit/9bcd058671439d3d86342012d882805a45d8916e)
  中修正了同样的调换。

### 9.2 R2013 的 RENDERSETTINGS

- **原因：** R2013 的特例处理跳过了 `class_version`，并在 `display_index` 之后读取 `has_predefined`。
  AutoCAD 的 R2013 预设有 `class_version` 2，之后就是 R2018 的布局，`has_predefined` 位于末尾。
- **表现：** 每个 R2013 的 RAPIDRTRENDERSETTINGS 都错位了 10 位（渲染级别和过滤器大小成了乱码）；
  `dwgread -O json` 甚至输出了 "-nan"（见 [14.1](#141-json不输出-nan)）。
- **图纸：** `gh44-error`、`gh109_1`。
- **修改：** R2013 采用 R2018 的布局。
- **提交：** [`3cfd5030`](https://github.com/monyuonyu/libredwg-fixes/commit/3cfd5030d85ff792550b62ffb1b16eba07ce6c77) — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 9.3 R13/R14 的 LAYER：关闭、打印、线宽

- **原因：** R13/R14 的解码器用“颜色 < 0”覆盖了关闭位，但 AutoCAD 的 R14 图纸保持颜色为正值并设置该位。
  另外，R13/R14 没有打印标志和线宽；写出 R2000 及以后版本时写成了 0。
- **表现：** 关闭的图层被读成打开；在 R2000 输出中，所有图层都不可打印，线宽为 0.00 mm。
- **修改：** 关闭状态由该位或负的颜色值决定；对 R2000 及以后使用 AutoCAD 的默认值，即可打印，线宽为 DEFAULT（31）。
- **提交：** [`4b815379`](https://github.com/monyuonyu/libredwg-fixes/commit/4b815379f222c25641ed9124c511db55fc7c1ea0) — 关闭状态位于 PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)，打印和线宽位于
  PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 9.4 R13/R14 的图元：自身的线型，线宽

- **原因：** 解码器只为 ByLayer 设置 `ltype_flags`，对使用自身线型的图元没有设为 3。R13/R14 没有图元线宽；
  R2000 输出写成了 0。
- **表现：** 使用自身线型（CONTINUOUS）的图元在 R2000 输出中变成了 ByLayer，线宽为 0.00 mm。
- **图纸：** `example_r14`、`sample_AC1014`，对照 AutoCAD 保存的 R2000：线型差异已经消失。
- **修改：** 自身线型时 `ltype_flags` 为 3；线宽为 ByLayer（29）。
- **提交：** [`597c0da6`](https://github.com/monyuonyu/libredwg-fixes/commit/597c0da69fbf26c80db4ac4634e4ef884c002e0c) — 线型位于 PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)，
  线宽位于 PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 9.5 LEADER：R2010 及以后的布局

- **原因：** 注释偏移（`endptproj`）在 R2010 及以后同样存在，而框的高度和宽度只到 R2007 为止，与 ACadSharp 的读取方式相同。
  LibreDWG 对这个 3BD 只读了 2 个 BD。
- **表现：** 数据超出了 6 位；`2000/TS1` 的 LEADER 写成 R2013 和 R2018 时丢失。
- **图纸：** `sample_AC1032`（其 LEADER 现在正好结束）。
- **修改：** `endptproj` 自 R13c3 起存在，`box_height`/`box_width` 到 R2007 为止。
- **提交：** [`9e5e27b3`](https://github.com/monyuonyu/libredwg-fixes/commit/9e5e27b33d9c622b1e6865ebfc7fc49515ea26b7) — 尚未提交给上游。

### 9.6 BLOCKSCALEACTION：连接点

- **原因：** AcDbBlockScaleAction 在 AcDbBlockActionWithBasePt 的 2 个连接点之后还有 3 个连接点（`conn_pts[5]`，
  `in_dxf` 使用 2–4），但它们被读入了 0–2。
- **表现：** UpdatedBaseX 和 UpdatedBaseY 被 Scale 和 XScale 覆盖。
- **修改：** 使用连接点 2–4。
- **提交：** [`a8f2deb1`](https://github.com/monyuonyu/libredwg-fixes/commit/a8f2deb148e2550913e32de9fd37c7412e1b1052) — 尚未提交给上游。

## 10. DXF 导入

### 10.1 LWPOLYLINE 的宽度和标志；关闭的图层

- **原因：** `in_dxf` 只在第一个顶点有 40/41 时才分配宽度，把恒定宽度标志设为 4 而不是 32，用 70 覆盖了标志，
  还丢掉了闭合多段线的 plinegen；`out_dxf` 只在有凸度时才写出宽度。关闭图层在 DXF 中的负颜色值被写入 DWG，成了索引 65529。
- **表现：** `sample_AC1015` 的 `dwg2dxf` 输出（只有最后一个顶点有宽度）无法读回（"Invalid DXF code 41 for LWPOLYLINE"）；
  ACadSharp 拒绝了该图层颜色。
- **修改：** 任何顶点上的宽度都读取，使用标志 32，保留 DWG 的标志；关闭的图层与 AutoCAD 写出的一样，带关闭位且颜色为正值。
- **提交：** [`57cda29e`](https://github.com/monyuonyu/libredwg-fixes/commit/57cda29e049ce9995721b6e90e1e614cc11caa6b) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.2 SPATIAL_FILTER

- **原因：** SPATIAL_FILTER 的所有 double 都使用组码 40（前裁剪 z 以及两个各含 12 个 double 的矩阵）；
  通用导入程序无法区分它们。
- **表现：** 裁剪顶点成了乱码，逆矩阵被 NaN 填满，DXF 转 DWG 后 ACadSharp 无法读取。
- **修改：** 专用的读取函数。
- **提交：** [`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8)（部分） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.3 HATCH：位于 (0,0) 的种子点

- **原因：** 在边界路径之后，值为 0.0 的 10/20 被当作标高，而不是种子点。
- **表现：** 位于原点的种子点被丢弃，`num_seeds` 从 1 变成了 0（`sample_AC10xx` 经由 DXF）。
- **修改：** 在路径之后出现时作为种子点。
- **提交：** [`da463d6d`](https://github.com/monyuonyu/libredwg-fixes/commit/da463d6d34deddbfcb3c107a1582bed3d65a4384)（部分） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.4 `*Model_Space` 保留其 DXF 句柄；VX_CONTROL

- **原因：** object[0]（`*Model_Space`）在读取 TABLES 之前就以句柄 0x1F 创建了。编码器给新增的 VX_CONTROL
  分配句柄 B 时，没有检查它是否空闲。
- **表现：** `gh209_1`（`*Model_Space` 为 2，LAYER 表为 1F）经由 DXF 后出现了重复句柄、两个模型空间，
  以及 14 个位于错误图层上的图元；它的 DICTIONARY 是 B。
- **修改：** 快速扫描 ASCII DXF，取出其 `*Model_Space` BLOCK_RECORD 的句柄；VX_CONTROL 只在 B 空闲时才使用 B。
- **提交：** [`f7880bf9`](https://github.com/monyuonyu/libredwg-fixes/commit/f7880bf993cc0926d66de6041ae95baf45661a4d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.5 表名：先精确匹配，再忽略大小写匹配

- **原因：** `dwg_find_tablehandle` 忽略大小写匹配名称，并取第一个匹配项。
- **表现：** 其他软件生成的图纸可能同时有 STANDARD 和 Standard；`gh109_1` 中引用 STANDARD 的 DIMENSION
  在 DXF 导入后变成了 Standard。
- **修改：** 先进行精确匹配。
- **提交：** [`eedf8e37`](https://github.com/monyuonyu/libredwg-fixes/commit/eedf8e37d597c5324b246a6dc9edc1084a87947a) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.6 DXF 中缺少的头变量

- **原因：** DXF 可以省略任何头变量；缺少的头变量保持为 0，即使 0 是无效值。
- **表现：** FreeCAD 的 DXF 没有 $TEXTSIZE 和 $DIMTXT，因此 FreeCAD 的 DWG 导出（`dxf2dwg`）得到的文字高度为 0，
  而 ACadSharp 会拒绝这样的值。
- **修改：** 对 TEXTSIZE、DIMTXT（acad，或 MEASUREMENT 为 1 时的 acadiso）、LTSCALE、CELTSCALE、DIMLFAC、DIMTFAC、
  DIMALTF 使用 AutoCAD 的默认值。
- **提交：** [`98acd4a3`](https://github.com/monyuonyu/libredwg-fixes/commit/98acd4a314d8d2cf7c9525c66ea4b900a140b54a) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.7 SHAPE：名称不是形编号

- **原因：** dynapi 把 `style_id` 作为 DXF 2，通用导入程序把字符串指针当作编号保存了下来。
- **表现：** 每次运行、每个平台上都是一个随机数；Linux 和 Windows 上的 `dxf2dwg` 写出的 DWG 不同。
- **修改：** 保持为 0；不读取 SHX 文件就无法得到这个编号。
- **提交：** [`0f204cfc`](https://github.com/monyuonyu/libredwg-fixes/commit/0f204cfcfca9dd153bfa59dc9dc998453e00507d) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 10.8 所属的顶点和属性使用绝对句柄

- **原因：** `dxf_postprocess_SEQEND` 把 POLYLINE_PFACE/MESH 的 `vertex[]` 和 INSERT/MINSERT 的 `attribs[]`
  做成了相对软指针，写入 R2004 及以后时成为相对代码 6 和 10。
- **表现：** ACadSharp 把它们读成了别的句柄，DXF 写成 R2004 及以后时丢失了顶点和属性（测试集中 160 个图元）。
- **修改：** 与 AutoCAD 一样使用绝对句柄。
- **提交：** [`1b1a0c62`](https://github.com/monyuonyu/libredwg-fixes/commit/1b1a0c629952aa23c6a9fbdb4e563c7e91836844) — 尚未提交给上游。

### 10.9 `dwgwrite` 按目标版本读取 DXF

- **原因：** `dwgwrite` 在 `header.version` 未知的情况下读取 DXF，因此 `in_dxf` 构建出的内容与 `dxf2dwg` 时不同。
- **表现：** AutoCAD 的 `example_2018` 的 DXF，222 个图元中只保留了 91 个（写成 R2004 和 R2018 时都是如此）。
- **修改：** 与 `dxf2dwg` 一样，在读取之前设置要写出的版本（或 R2000）。
- **提交：** [`8ba7693c`](https://github.com/monyuonyu/libredwg-fixes/commit/8ba7693c14a60fc55d1eee1a70fb551ee821f230) — 尚未提交给上游。

DXF 导入方面的其他修复：EED 1003/1004（[4.1](#41-eed-图层1003大端序的句柄dxf-中是名称)、
[4.2](#42-dxf-的-eed-二进制数据1004被解码了两次)），UTF-8 文本
（[3.2](#32-r2007-及以后的-dxf-转-r2000把-utf-8-文本转换为图纸代码页)、
[3.3](#33-dxf-转-r2007-及以后名称只剩第一个字符)），嵌入的 MTEXT
（[5.5](#55-dxf-导入attribattdef-中嵌入的-mtextmtext-的分栏部分)），属性标记
（[5.9](#59-属性标记除空格和--以外的任意字符)），表格（[6.6](#66-acad_table-的-dxf-导入与导出)），
SAT（[7.6](#76-dxfsat-的行和转义的-)），代理对象（[8.4](#84-dxf未知对象和代理对象)），
DIMSTYLE 和 SPLINE 的默认值（[13.16](#1316-dimstyler2007-及以后变量的-autocad-默认值)、
[13.17](#1317-spliner2013-及以后的标志节点参数和方案)）。

## 11. DXF 导出

### 11.1 HATCH/MPOLYGON：图案名称与 DWG 中一致

- **原因：** 实体填充总是写成 "SOLID"，丢掉了名称中表示孤岛样式的后缀。
- **表现：** 经由 DXF 后 "SOLID,_O" 变成了 "SOLID"。
- **修改：** 按 DWG 中保存的样子写出带孤岛样式后缀的名称（"SOLID,_O"）；只有没有名称的实体填充才写成 SOLID。
- **提交：** [`1935be09`](https://github.com/monyuonyu/libredwg-fixes/commit/1935be09e0ad6e1cac637427804eb02741afeada)（部分） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 11.2 R13/R14 图纸保留较新的类的图元

- **原因：** 由较新版本 AutoCAD 保存的 R14 DWG 可能包含 MULTILEADER、WIPEOUT、MESH、PDFUNDERLAY 和 ARC_DIMENSION 类；
  `out_dxf` 在输出 R13/R14 DXF 时跳过了它们。
- **表现：** DXF 把它们全部丢掉了。
- **修改：** 只在把较新的图纸转换为 R13/R14 DXF 时才跳过。
- **提交：** [`649637f8`](https://github.com/monyuonyu/libredwg-fixes/commit/649637f8a4a98102115f7747787dd7883178650c) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 11.3 Standard/ByLayer/ByBlock 的拼写只在从 R13/R14 转换时更改

- **原因：** `out_dxf` 对所有图纸都改写名称的拼写，例如 STANDARD → Standard。
- **表现：** `gh109_1`（R2013）有一个 DIMSTYLE STANDARD，而其 DIMENSION 在 DXF 中引用的是 Standard。
- **修改：** 只在从 R13/R14 转换时改写拼写。
- **提交：** [`fc66341d`](https://github.com/monyuonyu/libredwg-fixes/commit/fc66341dacc1b48750fbf947daa9e4c0301470a6) — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

### 11.4 R2004 及以后图纸执行 `dwg2dxf --as r2000` 时不输出图元

- **原因：** `get_first_owned_entity` / `get_next_owned_block_entity` 使用了由 `dwg_fixup_BLOCKS_entities` 添加的
  引用，而这些引用尚未解析。
- **表现：** 对 R2004 及以后的图纸执行 `dwg2dxf --as r2000` 时，不输出任何图元。
- **修改：** 按句柄查找块中的图元。
- **提交：** [`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb)（部分） — PR [#1445](https://github.com/LibreDWG/libredwg/pull/1445)。

DXF 导出方面的其他修复：真彩色（[2.4](#24-dxf真彩色420429按-24-位-rgb)），DIMSTYLE 颜色
（[2.5](#25-dxfdimstyle-的颜色-176178-是颜色索引)），EED 1003
（[4.1](#41-eed-图层1003大端序的句柄dxf-中是名称)），MTEXT 的 `\P`
（[5.7](#57-mtext-的换行写成-p)），SEQEND 之后的 ATTRIB/VERTEX
（[5.8](#58-dxf-导出seqend-之后的-attrib-和-vertex)），表格和表格样式
（[6.6](#66-acad_table-的-dxf-导入与导出)、[6.7](#67-按-autocad-的布局写出-dxf-的-tablestyle)），
ACIS（[7.5](#75-sat-与-sab-互相转换)、[7.6](#76-dxfsat-的行和转义的-)），代理对象
（[8.4](#84-dxf未知对象和代理对象)）。

## 12. 从较新版本写出 R2000

有了这些修复，`example_2004` … `example_2018` 直接写成或经由 DXF 写成 R2000 后，全部 222 个图元都与 AutoCAD 自己保存的
R2000 一致；两组图纸合计 3,405 个图元中有 3,258 个一致（[`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40)）。其余差异来自 AutoCAD 重新编号的句柄和匿名块、
R14 无法保存的线宽，以及未做的三种转换（堆叠分数代码、动态块参照、拟合点样条转为控制点）。

### 12.1 链接 POLYLINE 的 VERTEX 和 INSERT 的 ATTRIB

- **原因：** R2004 及以后把属于另一个图元的图元保存在句柄列表中（`vertex[]`、`attribs[]`）；R13–R2000 则保存在链表中
  （所有者中有 first/last，每个所属图元中有 prev/next）。`dwg_fixup_BLOCKS_entities` 只为块头中的图元构建了链表。
- **表现：** R2018 图纸写成 R2000 后，first/last 顶点和属性为 0，顶点也没有链接：把 AutoCAD 的 R2018 样例重新写成 R2000 后，
  ACadSharp 在 10 个 VERTEX 和 4 个 ATTRIB 中一个也找不到（"Leading entity with handle 0 not found"）。
- **修改：** 按照 AutoCAD 的写法（`sample_AC1015`）从列表构建链接：first/last 为绝对代码 4 的软指针（相对代码 6 是合法的，
  但 ACadSharp 会相对于句柄 0 去解析它并崩溃），prev/next 为相对的。
- **提交：** [`89191ebe`](https://github.com/monyuonyu/libredwg-fixes/commit/89191ebe07c103dfdf91f56fe74556d573486389) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.2 MLINESTYLE：线型编号

- **原因：** 自 R2018 起，MLINESTYLE 中的各条线按句柄引用其线型，之前则按编号引用（是一个 union）。
  写出 R2000 时把句柄指针当作编号写了出去。
- **表现：** ACadSharp 报错 "Linetype not assigned, index 29712"。
- **修改：** 转换句柄：ByLayer 为 32767，ByBlock 为 32766，否则为其在 LTYPE_CONTROL 条目中的位置。
  AutoCAD 对同一张图纸保存的 R2000 和 R2018 与此一致（ByLayer 句柄 0x15 对应编号 32767）。
- **提交：** [`a1670d7a`](https://github.com/monyuonyu/libredwg-fixes/commit/a1670d7a3b25ed0cc22a4cb3a8939cdac43cb671) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.3 LWPOLYLINE：R2010 之前没有顶点 ID 标志

- **原因：** R2010 及以后的 LWPOLYLINE 写成 R2000 时保留了标志 1024（顶点 ID）。
- **表现：** ACadSharp 读取了一个并不存在的 `num_vertexids`，并丢失了点。
- **图纸：** `2018/Dynblocks`。
- **修改：** R2010 之前清除该标志。
- **提交：** [`c2e85bb6`](https://github.com/monyuonyu/libredwg-fixes/commit/c2e85bb62181e2d04578d3f3387756ced6f44590) — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.4 AuxHeader：R2000 同样是 123 字节

- **原因：** ODA 第 27 章对所有版本都列出了末尾的 RL `zero_7` 和 `zero_8`，但它们只在 R2004 及以后才写出。
- **表现：** `dwgrewrite` 生成的 R2000 文件中 AuxHeader 为 115 字节（AutoCAD 的是 123 字节，`sample_AC1015`），
  ACadSharp 抛出 EndOfStreamException。
- **修改：** 所有版本都写出它们。
- **提交：** [`3ab77f4d`](https://github.com/monyuonyu/libredwg-fixes/commit/3ab77f4d123107fd17bddd725865dd1e1dad9039)（部分） — PR [#1442](https://github.com/LibreDWG/libredwg/pull/1442)。

### 12.5 R13/R14 的名称按 AutoCAD 2000 的写法

- **原因：** R13/R14 的名称用大写拼写；AutoCAD 把同一张图纸保存为 R2000 时，使用 ByBlock、ByLayer、Continuous、Standard
  以及字典键 Standard。
- **图纸：** `example_r14` 对照 `example_2000`。
- **修改：** 从 R13/R14 写出 R2000 及以后版本时使用这些名称；MLINESTYLE 保留 STANDARD。
- **提交：** [`65ae3982`](https://github.com/monyuonyu/libredwg-fixes/commit/65ae39822d496dc21265df6fadbe6cd871c5c0fb)（部分） — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

### 12.6 渐变填充

- **原因：** R2000 没有渐变填充。
- **表现：** 渐变 HATCH 保持为 ByLayer，与 AutoCAD 保存的 R2000 不同。
- **图纸：** `sample_AC1015`、`example_2000`（AutoCAD 保存的 R2000）。
- **修改：** 与 AutoCAD 一样，把渐变 HATCH 变为使用其第一个渐变颜色的实体填充。
- **提交：** [`358cb644`](https://github.com/monyuonyu/libredwg-fixes/commit/358cb64439f23dd79739ff337942ebc938cfcd40)（部分） — PR [#1444](https://github.com/LibreDWG/libredwg/pull/1444)。

R2000 输出方面的其他修复：颜色（[2.3](#23-写入-r2000-的颜色byblockautocad-的-aci-后备值aci-调色板)），
EED 字符串（[4.3](#43-写入-r2000-的-r2007-及以后的-eed-字符串)），R2018 多行属性
（[5.3](#53-r2018-多行属性写成较早版本)），MTEXT
（[5.6](#56-r13r14-的-mtext-写成-r2000-及以后行距为-1而不是-0)、[5.7](#57-mtext-的换行写成-p)），
表格（[6.4](#64-r2010-及以后的表格以旧格式写入-r2000r2004)、
[6.5](#65-r2007-的表格写入-r2000r2004单元格文字)），代理对象
（[8.3](#83-未知对象写成-acad_proxy_entity--acad_proxy_object)），R13/R14 的图层和
图元（[9.3](#93-r13r14-的-layer关闭打印线宽)、[9.4](#94-r13r14-的图元自身的线型线宽)）。

## 13. 写出 R2004 至 R2018

上游 LibreDWG 只能写出 R2000 及更早版本。有了这些提交，LibreDWG 可以写出 R2004、R2010、R2013 和 R2018；
全部 149 张图纸在每个版本下，无论直接写出还是经由 DXF，经 ACadSharp 往返都没有丢失。这些提交都尚未提交给上游。

### 13.1 R2004 文件容器，以及 LZ77 压缩

- **原因：** R2004 容器（也是 R2010 及以后的外框）写出后没有任何读取程序能够接受：段映射和页映射被列为数据段
  （ACadSharp 因重复的空段名而失败），页的大小错误，Preview 是一个 1.3 MB 的页，系统页没有对齐，文件头的尾部也被覆盖了。
- **修改：** 新增 `encode_r2004_container`，按照 AutoCAD 自己的 R2004 文件写出：空段在最前，ID 按段映射的顺序递减；
  压缩的段放在 0x7400 的页中，其他段各放在一个页中，页大小为其大小向上取整到 0x80（Preview 为 0x400）；
  页头带有大小、64 位起始偏移和校验和；页按 0x20 对齐并用魔术序列填充；段映射和页映射位于数据页之后；
  文件头位于 0x80，并在末尾重复一份；还有 AppInfo 和 AppInfoHistory 的地址。新增 LZ77 压缩器 `r2004_compress`
  （ODA 4.7，使用哈希链）；其单元测试经由 `decompress_R2004_section` 做往返验证。
- **提交：** [`1eb484a6`](https://github.com/monyuonyu/libredwg-fixes/commit/1eb484a6b53445a2f9be6d2a8e09d512b674d485)、[`9a0474f3`](https://github.com/monyuonyu/libredwg-fixes/commit/9a0474f3679c1283b96fffc04197c3a3e0bb34e2)。

### 13.2 R2004 及以后的输入：页数不是段数

- **原因：** 解码后的 R2004 及以后 DWG 的 `header.sections` 是其页数（`sample_AC1018` 中为 56）。
- **表现：** `dwg_sections_init` 报错 "Too many sections"，什么也没有写出。
- **修改：** 输出 R2004 及以后版本时将其重置为 6。
- **提交：** [`7cb0849e`](https://github.com/monyuonyu/libredwg-fixes/commit/7cb0849e1bcc8a77e55ae2b2f9fd21770c3ff8b8)。

### 13.3 按 AutoCAD 的写法写出文件头

- **原因：** AutoCAD 的 R2004 及以后文件在 0x0C 处为 3；R2018 文件在 0x0B 处也有维护版本号，0x15–0x17 处为 0
  （测试集中全部 37 张 AC1032 图纸都是如此）。LibreDWG 重新写出的 R2018 在 0x0B 处为 0x1D，在 0x16 处为应用程序版本 0x21；
  由较早版本生成的 R2004 及以后文件在 0x0C 处为 1，在 0x11 处为源文件的应用程序版本。
- **修改：** 在各版本之间都与 AutoCAD 相同；R2018 不写应用程序版本。
- **提交：** [`649dd383`](https://github.com/monyuonyu/libredwg-fixes/commit/649dd3838c46e3e92277702fef3dfadfb879bd59)。

### 13.4 R2010 和 R2013：维护版本号大于 3

- **原因：** R2010 和 R2013 的文件头段和类段，只有在维护版本号大于 3 时，其大小之后才有 4 字节的 hsize。
  ACadSharp 从 0x0B 处的字节取得维护版本号，LibreDWG 则从 0x12 处的字节取得。R2018 的源在 0x12 处为 0，在 0x0B 处得到了 0x6d。
- **表现：** ACadSharp 期望有 hsize，因而无法读取文件头（EndOfStreamException，例如 `sample_base__empty` 写成 R2010 时）。
- **修改：** 对于 R2010 和 R2013，与 AutoCAD 的文件一样，把两处都提高到大于 3。
- **提交：** [`79c07845`](https://github.com/monyuonyu/libredwg-fixes/commit/79c07845985f416afe2af4a3fc09036050f0e919)。

### 13.5 R2010 及以后的文件头和类：段大小不含 hsize

- **原因：** R2010 及以后的文件头段和类段的大小把其后 4 字节的 hsize 也算了进去；而 AutoCAD 中类段的 hsize 为 0，而不是字节大小。
- **表现：** CRC 和结束哨兵位于大小所指位置之前 4 字节处（ACadSharp："Invalid section sentinel found in AcDb:Header"）。
- **修改：** 新增 `encode_patch_RLsize_hsize`；类段的 hsize 为 0。
- **提交：** [`ceeaca39`](https://github.com/monyuonyu/libredwg-fixes/commit/ceeaca39aa0f23085a1498bc9880c36c6c250333)。

### 13.6 保留类版本 0

- **原因：** AutoCAD 对许多类写出的类版本为 0；编码器却把它替换成了图纸的版本。
- **图纸：** `2004/Line`（EXACXREFPANELOBJECT、SOLID_BACKGROUND 变成了 31.8）。
- **修改：** 只有源为 R2000 及更早版本（没有类版本）或来自 DXF/JSON 时，才使用图纸的版本。
- **提交：** [`e9563c45`](https://github.com/monyuonyu/libredwg-fixes/commit/e9563c453ac021a08906a38d6c7028252116017c)。

### 13.7 Template、Classes 和 AcDbObjects 的开头和结尾与 AutoCAD 相同

- **原因：** AcDb:Template 的说明、类段的结尾以及第一个对象之前的 4 个字节都与 AutoCAD 的文件不一致。
- **修改：** 空的 Template 说明在 R2004 中为 RS 0（共 4 字节），自 R2007 起为 RS 1 加结尾的 0（6 字节）；
  R2004 及以后的类段在哨兵之后以 8 个零字节结束，并计入段大小（R13–R2000 仍为 RL 0x0dca）；R2004 及以后的 AcDbObjects
  与 AutoCAD（以及 ACadSharp）一样以 RL 0x0dca 开头，而不是 0。
- **提交：** [`beb122ae`](https://github.com/monyuonyu/libredwg-fixes/commit/beb122aed566b71c27cdafe1acb810fb736657d3)、[`4738d852`](https://github.com/monyuonyu/libredwg-fixes/commit/4738d85246bf7b3f01553cec5c81513e689879df)、[`7caa82f8`](https://github.com/monyuonyu/libredwg-fixes/commit/7caa82f84092580a14bc2679c4bac1e583dc9fdc)。

### 13.8 Preview：END 哨兵和图像地址

- **原因：** 解码后的 R2000 预览缺少 END 哨兵，而它只在输出 R13–R2000 时才写出。预览中的头部和图像起始地址是文件地址，
  却直接从源文件复制而来。
- **表现：** 读回 R2004 输出时出现 "Preview overflow 13208 + 119 > 13311"。
- **修改：** 内容中缺少哨兵时一律写出；`thumbnail_rebase` 把地址移到新的位置。
- **提交：** [`30a1f7a1`](https://github.com/monyuonyu/libredwg-fixes/commit/30a1f7a11d5170dc93f57c8bac3961eb5560990b)。

### 13.9 从 0x2F0 开始的对象类型

- **原因：** `bit_write_BOT` 把 256 及以上的所有类型都写成 RC + 0x1F0，而且其第三个分支是代码 1 加 RS。
- **表现：** 编号从 0x2F0（752）开始的类在 R2010 及以后得到了错误的类型。
- **修改：** 与 AutoCAD 和 `bit_read_BOT` 一致：0x1F0–0x2EF 用代码 1，其他用代码 2 加 RS。
- **提交：** [`78fe37ba`](https://github.com/monyuonyu/libredwg-fixes/commit/78fe37ba18e9581ce347546460ae162225d49b0b)。

### 13.10 R2004 及以后文件中的字符串：T16、TU 和字符串流

- **原因：**
  - 在 R2007 之前，空的 T16 字符串是长度 1 加 NUL（AutoCAD 的 SummaryInfo 中为 `01 00 00`）；LibreDWG 写的是 `00 00`
    （[`754d999d`](https://github.com/monyuonyu/libredwg-fixes/commit/754d999d6a521a0ef329530068b40f525b9080f9)）。
  - 自 R2007 起，RS 长度包含结尾的 0，空字符串为 RS 1 加 0；从较早版本写出时长度不含 0，而 R2007 及以后的源还把 UTF-16
    写入了 R2000/R2004 文件（[`ce66db95`](https://github.com/monyuonyu/libredwg-fixes/commit/ce66db95cfd3ac20b3855d8c35ce31dfda9e6adf)）。
  - AutoCAD 写出的 TU 字符串不带结尾的 0（`aec_objects` 的 NOD：69 个名称，字符串流少 69 × 16 位）
    （[`011c3181`](https://github.com/monyuonyu/libredwg-fixes/commit/011c3181df8591028bc1c83e38b349e35303f8ad)，测试 [`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478)）。
  - 字符串流在其 `has_strings` 位之前的大小被强制转换为 RS，因此字符串超过 32,767 位时高位部分出错（[`632dfd5c`](https://github.com/monyuonyu/libredwg-fixes/commit/632dfd5cc46018f8ca7a4014e5610b67a6e64ef1)）。
- **表现：** SummaryInfo 的字节不同；字符串流很大时（`aec_objects` 和 `gh44-error` 写成 R2018 时的类），
  ACadSharp 从错误的位置读取类。
- **修改：** 与 AutoCAD 一致；`bit_write_strings_size` 写出 RS 大小，大的值带 0x8000 和高位部分。
  `2004/Line` 写成 R2004 后，SummaryInfo 与 AutoCAD 的逐字节一致。
- **提交：** [`754d999d`](https://github.com/monyuonyu/libredwg-fixes/commit/754d999d6a521a0ef329530068b40f525b9080f9)、[`ce66db95`](https://github.com/monyuonyu/libredwg-fixes/commit/ce66db95cfd3ac20b3855d8c35ce31dfda9e6adf)、[`011c3181`](https://github.com/monyuonyu/libredwg-fixes/commit/011c3181df8591028bc1c83e38b349e35303f8ad)、[`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478)、[`632dfd5c`](https://github.com/monyuonyu/libredwg-fixes/commit/632dfd5cc46018f8ca7a4014e5610b67a6e64ef1)。

### 13.11 所属的图元、属性和顶点以列表形式保存

- **原因：** R2004 及以后的 BLOCK_HEADER 中没有 first/last 图元链接，INSERT/MINSERT 中没有 first/last 属性，
  POLYLINE 中没有 first/last 顶点，取而代之的是 `num_owned` 和句柄列表。这些列表是空的。
- **表现：** `sample_AC1015` 写成 R2004 后丢失了全部 355 个图元。
- **修改：** `upconvert_owned_lists` 按对象顺序把图元收集到其所有者下（不含 BLOCK、ENDBLK 和 SEQEND），
  ATTRIB 收集到其 INSERT 下，VERTEX 收集到其 POLYLINE 下。
- **提交：** [`2098043b`](https://github.com/monyuonyu/libredwg-fixes/commit/2098043bbaaa4d2ad816a76209e757bc019b6144)。

### 13.12 PLOTSETTINGS、LAYOUT：着色打印的默认值

- **原因：** R2000 没有着色打印字段；R2004 及以后得到的自定义 DPI 为 0。
- **表现：** ACadSharp 拒绝该值（"The valid shade plot DPI values range from 100 to 23767"），因此所有 LAYOUT 都丢失了。
- **修改：** 分辨率级别 2 和 300 DPI，即 AutoCAD 的 R2004 图纸（`sample_AC1018`）中的值。
- **提交：** [`653fea5e`](https://github.com/monyuonyu/libredwg-fixes/commit/653fea5ebdf0e3e7edb2a306165ed075f9fd9fa0)。

### 13.13 较早版本的 XRECORD 数据

- **原因：** 自 R2007 起 xdata 字符串是 UTF-16，但写出时受源文件 `xdata_size` 的限制。
- **表现：** 从 R2000 转换时，前几个字符串之后的值被截掉；ACadSharp 读取这些 xrecord 时失败
  （"Arithmetic operation resulted in an overflow"，`sample_AC1015` 写成 R2010 时有 8 个）。
- **修改：** 跨版本时允许最多两倍的大小；大小会重新计算。
- **提交：** [`5300c579`](https://github.com/monyuonyu/libredwg-fixes/commit/5300c579ff3943543011c6de2dd4e3b70df36471)。

### 13.14 DWG 输入中的 MATERIAL 句柄

- **原因：** 写出 R2007 及以后版本时，图元、LAYER 以及 3DSOLID、REGION 和 BODY 的材质句柄被丢弃，
  理由是 MATERIAL 对象会写成占位对象。但只有来自 DXF 时才是这样。
- **表现：** R2018 写成 R2018 时，所有图层都丢失了材质。
- **修改：** 只对 DXF 输入丢弃它们。
- **提交：** [`036da95e`](https://github.com/monyuonyu/libredwg-fixes/commit/036da95ef1a810ef1c60826b0aa535ba1d9ae65d)。

### 13.15 AcDbBlockElement：版本按读取到的值

- **原因：** 对于 R2007 及以后，`be_major` 和 `be_minor` 被写成 33 和 29；AutoCAD 的 R2018 图纸中是 33 和 329。
- **修改：** 按从 R2007 及以后的 DWG 读取到的值写出。
- **提交：** [`eac6288a`](https://github.com/monyuonyu/libredwg-fixes/commit/eac6288a16743092f005cecbddfd848d007d8739)。

### 13.16 DIMSTYLE：R2007 及以后变量的 AutoCAD 默认值

- **原因：** 从较早版本或从 DXF 转换时，DIMJOGANG、DIMFXL、DIMALTMZF 和 DIMMZF 为 0，DIMTFILLCLR 没有颜色方法。
  第一次修复只涵盖了较旧的 DWG；而从 DXF 转换时，`dat.from_version` 就是目标版本。
- **表现：** ACadSharp 拒绝 DIMJOGANG 为 0（"must be in range of 5 to 90 degrees"），丢弃了这样的标注样式，
  标注改用了 Standard（往返到 R2010 及以后时有 63 个）。
- **修改：** 使用 AutoCAD 的默认值：DIMFXL 1、DIMJOGANG 45°、DIMTFILLCLR ByBlock、DIMALTMZF 和 DIMMZF 100，
  在 DIMSTYLE 和头变量中都设置，来自 DXF 时也一样。
- **提交：** [`16871d28`](https://github.com/monyuonyu/libredwg-fixes/commit/16871d2898ba823f957abae68a73c7114be2e50a)、[`c720b36d`](https://github.com/monyuonyu/libredwg-fixes/commit/c720b36d04579353ac62ebda61b43dde5f379f72)。

### 13.17 SPLINE：R2013 及以后的标志、节点参数和方案

- **原因：** 从较早版本转换时 `splineflags` 为 8 或 9，`knotparam` 为 0；从 DXF 转换时两者都为 0。
  ACadSharp 依据这两个值判断方案，因此控制点样条看起来像拟合点样条。LibreDWG 还写出了解码得到的方案（拟合点样条为 2）；
  而在测试集中全部 591 个 R2013 及以后的样条中，AutoCAD 写的都是 1。
- **表现：** 出现 "Failed to read ReadBitDouble"，样条丢失（测试集中来自 DXF 的有 32 个）。
- **修改：** 与 AutoCAD 一致：控制点样条的 `splineflags` 为 0（闭合时为 4），`knotparam` 为 15；拟合点样条分别为 9 和 0；
  方案 BL 为 1。
- **提交：** [`09d03974`](https://github.com/monyuonyu/libredwg-fixes/commit/09d03974d1cd3423d68f088bcfdcc7dd85787d29)、[`ab05c9d1`](https://github.com/monyuonyu/libredwg-fixes/commit/ab05c9d13e521420f001422af986c9a303d5833e)、[`f3322488`](https://github.com/monyuonyu/libredwg-fixes/commit/f332248859806fe1609bb05e4e845ffd1a8629a9)。

### 13.18 途中发现、同样影响 R2000 的问题

- **`bit_write_UMC`** 只检查了 8 组中的 4 组是否为前导零，因此每个值至少占 4 字节（偏移 1 被写成 `81 80 80 00`，
  AutoCAD 写的是 `01`）；`2004/Line` 的对象映射是 1,174 字节，而不是 674 字节（[`691d4598`](https://github.com/monyuonyu/libredwg-fixes/commit/691d4598e5c03d21047caac959358f8df2ba21ad)）。它还像有符号 MC 那样，
  在最后一个字节设置了 0x40 时多加一个 0 字节；在 R2010 及以后，每个对象前的句柄流大小因此多了一个字节
  （[`40d6e568`](https://github.com/monyuonyu/libredwg-fixes/commit/40d6e568c035ba2af2c37dd45bc581f1e9835912)）。现在使用最短形式。
- **`bit_write_DD`** 把与默认值相差在 1e-12 以内的值写成了默认值：一条 LINE 的终点与起点只差 1 ulp（`example_2004`），
  读回时变成了起点。现在只有逐位相同的值才算默认值；`example_2004` 和 `sample_AC1018` 中的 LINE、LWPOLYLINE 和 3DFACE
  写成 R2004 后与 AutoCAD 的对象逐位一致（[`57cbcdd9`](https://github.com/monyuonyu/libredwg-fixes/commit/57cbcdd91fb0cb99437098919fd66508606e3b5a)）。
- **对象映射：** 当最后一个句柄正好填满一个分块（超过 2030 字节）时，在最后一个大小为 2 的分块之前留下了 2 个保留字节的零，
  ACadSharp 读过了末尾（`sample_AC1032` 写成 R2000 时）（[`d56fd195`](https://github.com/monyuonyu/libredwg-fixes/commit/d56fd19555ec86e6366a6c0a3b3497d2b8c8acbb)）。
- **SPATIAL_INDEX：** 其格式尚不清楚；`aec_objects` 的 `bindata_size` 为 50,396,167，解码器拒绝了它，
  编码器却写出了同样数量的零字节，得到一个 51 MB 的对象段。现在无效的大小一律视为 0（[`5398b8a6`](https://github.com/monyuonyu/libredwg-fixes/commit/5398b8a6c63a4e43815114dbbfe0df1dd2c2db12)）。
- **提交：** [`691d4598`](https://github.com/monyuonyu/libredwg-fixes/commit/691d4598e5c03d21047caac959358f8df2ba21ad)、[`40d6e568`](https://github.com/monyuonyu/libredwg-fixes/commit/40d6e568c035ba2af2c37dd45bc581f1e9835912)、[`57cbcdd9`](https://github.com/monyuonyu/libredwg-fixes/commit/57cbcdd91fb0cb99437098919fd66508606e3b5a)、[`d56fd195`](https://github.com/monyuonyu/libredwg-fixes/commit/d56fd19555ec86e6366a6c0a3b3497d2b8c8acbb)、[`5398b8a6`](https://github.com/monyuonyu/libredwg-fixes/commit/5398b8a6c63a4e43815114dbbfe0df1dd2c2db12)。

R2004 及以后方面的其他修复：类条目（[1.1](#11-r2004-及以后的类条目dwg_version-和-maint_version-是-bl)），
文件头（[1.4](#14-r2007-及以后的文件头没有-unknown_54--unknown_57)），FileDepList
（[1.5](#15-filedeplist功能名称是-t32-字符串)），来自 EED 的真彩色
（[2.6](#26-r13r2000-升级到-r2004-及以后autocad-保存在-eed-中的真彩色)），DXF 字符串
（[3.3](#33-dxf-转-r2007-及以后名称只剩第一个字符)），表格
（[6.8](#68-从较早版本和从-dxf-写入-r2010-及以后的表格)、[6.9](#69-从较早版本写出-r2007-及以后时的内存泄漏)），
ACIS（[7.2](#72-写出空位历史句柄材质) – [7.6](#76-dxfsat-的行和转义的-)），
代理对象（[8.5](#85-目标为-r2004-和-r2010-及以后时也生成代理对象)），LEADER
（[9.5](#95-leaderr2010-及以后的布局)），BLOCKSCALEACTION（[9.6](#96-blockscaleaction连接点)），
DXF 句柄和 `dwgwrite`（[10.8](#108-所属的顶点和属性使用绝对句柄)、
[10.9](#109-dwgwrite-按目标版本读取-dxf)）。

## 14. 工具和测试

### 14.1 JSON：不输出 NaN

- **原因：** JSON 中没有 NaN。`VALUE_RD` 只在发布构建中替换 NaN，而且不处理无穷大；向量中的 double
  （`FIELD_VECTOR_N`，例如 SPATIAL_FILTER 的矩阵）直接用 `fprintf` 输出。
- **表现：** 对于读错的 RAPIDRTRENDERSETTINGS，`dwgread -O json` 输出了 "-nan"，整个输出因此无法读取。
- **图纸：** `gh44-error`。
- **修改：** NaN 和无穷大的 double 写成 0.0，在 `_VALUE_RD` 中对所有构建统一检查，向量也使用它。
- **提交：** [`c86190fb`](https://github.com/monyuonyu/libredwg-fixes/commit/c86190fbcc0db292aa09e9592e24bdca4b8513ef)、[`ed01cb5c`](https://github.com/monyuonyu/libredwg-fixes/commit/ed01cb5c6d4d4805319d22447e47a1a9c18152e8)（部分） — PR [#1441](https://github.com/LibreDWG/libredwg/pull/1441)。

### 14.2 `dwgrewrite`、`dxf2dwg`、`dwgwrite`：R2010、R2013 和 R2018 是有效的输出版本

- **修改：** 帮助文本和手册页把 r2010、r2013 和 r2018 列为有效的输出版本（r2007 仍按 r2010 写出），
  `dwgwrite` 也不再只接受 r2000。测试集中全部 149 张图纸写成 R2010、R2013 和 R2018 后经 ACadSharp 往返都没有丢失，
  经由 DXF 写成 R2004 至 R2018 时也是如此（剩余的改变值，R2000/R2004 为 53 个，R2010 及以后为 66 个，全部位于 TABLE 中）。
- **提交：** [`416c3d22`](https://github.com/monyuonyu/libredwg-fixes/commit/416c3d222396b1419a4dc6edf27e97969135d61a)、[`1b613fcb`](https://github.com/monyuonyu/libredwg-fixes/commit/1b613fcbd8b4a04f44b1bc9d2ea591a43d346c04) — 尚未提交给上游。

其他测试方面的修改与相应的修复放在一起：[`45e6b50f`](https://github.com/monyuonyu/libredwg-fixes/commit/45e6b50f615aa7346827f71a26de67dc3a8f0eae)（[3.1](#31-亚洲代码页双字节字符的首字节)），
[`aad393ec`](https://github.com/monyuonyu/libredwg-fixes/commit/aad393ec206b1116694db8ae7877874207530709)（[7.1](#71-读取自-r2013-起没有-acis-空位r2007-及以后的未知-bl)），
[`9a0474f3`](https://github.com/monyuonyu/libredwg-fixes/commit/9a0474f3679c1283b96fffc04197c3a3e0bb34e2)（[13.1](#131-r2004-文件容器以及-lz77-压缩)），
[`2110248b`](https://github.com/monyuonyu/libredwg-fixes/commit/2110248b026ac8afbd994fb8cae52dbfc19c3478)（[13.10](#1310-r2004-及以后文件中的字符串t16tu-和字符串流)）。
