[English](README.md) | [日本語](README.ja.md) | **简体中文**

> 本文译自 [README.md](README.md)（英文版为原文）。

# libredwg-fixes

这是修复了 [GNU LibreDWG](https://github.com/LibreDWG/libredwg) 0.14.8601 中错误的版本。我们用独立的实现读取同一批图纸，
并与 AutoCAD 自身保存的图纸逐一对照，把发现的差异一个个修正。
修复位于 `fixes` 分支（默认分支），建立在上游标签 `0.14.8601` 之上。
这些修复正在以拉取请求的形式提交给上游；一旦被合并，这个分支的使命就完成了。
日文版 FreeCAD（[freecad-ja-unified](https://github.com/monyuonyu/freecad-ja-unified)）用它来导入和导出 DWG。

## 验证内容

- **读取** — 149 张图纸（R13 至 R2018；ACadSharp 中由 AutoCAD 生成的样例，以及 LibreDWG 自带的测试数据）分别用 LibreDWG 和
  独立的 MIT 许可读取库 [ACadSharp](https://github.com/DomCR/ACadSharp) 3.8.0 读取，并逐个图元、逐条表记录、逐个头变量进行比较。
  结果：值和类型均无差异（图元 59,883 个，表记录 2,936 条，头变量 30,412 个），ACadSharp 不读取的内容除外。
- **写出 R2000** — 每张图纸都以 R2000 写出，包括直接写出（`dwgrewrite --as r2000`）和经由 DXF（先 `dwg2dxf` 再 `dxf2dwg`），
  再用 ACadSharp 读回：没有丢失，类型没有变化；其他应用程序的自定义图元像 AutoCAD 一样保留为代理对象。
- **与 AutoCAD 本身对照** — AutoCAD 将同一批图纸保存为了所有版本（sample_AC1014 … AC1032，example_r14 … example_2018）。
  把较新的版本写成 R2000，并与 AutoCAD 自己保存的 R2000 比较：example_2004 … 2018 的全部 222 个图元一致；
  总体上 3,405 个图元中有 3,258 个一致。其余差异来自 AutoCAD 每次保存都会重新编号的句柄和匿名块编号、R14 无法保存的线宽，
  以及 AutoCAD 所做而这里未做的三种转换（堆叠分数代码、动态块参照、拟合点样条转为控制点）。
- **写出 R2004、R2010、R2013、R2018**（新增：上游只能写出 R2000 及更早版本）— 每张图纸都以各个版本写出，包括直接写出和经由 DXF
  （`dxf2dwg --as r2018`），再用 ACadSharp 读回：没有无法读取的图纸，没有丢失，类型没有变化。
  与 AutoCAD 以同一版本保存的结果比较，R2018 中 3,405 个图元有 3,208 个一致（其余差异与上面同类，另有 TABLE 单元格）。
  ACIS 实体得以保留：19 张图纸中 107 个实体的 SAT/SAB 在写出后按字节或按记号完全一致，
  从 R2013 之前的图纸或 DXF 写入 R2013 及以后版本的 AcDs 段时也是如此。
- `make check` 通过。

## 修复内容（122 个提交）

每个修复的详细说明：[docs/FIXES.zh-CN.md](docs/FIXES.zh-CN.md)

读取
- R2004 及以后的类段：`dwg_version`、`maint_version` 是 BL（规范写的是 BS）；R2018 的 AuxHeader：维护版本号是 RL。
- 图元颜色：RGB 在透明度之前；R2007 及以后的配色簿句柄被读了两次（导致丢失图层）。
- MTEXT：R2018 中只有边框位时的背景填充。
- R2013 及以后的 3DSOLID/REGION/BODY：没有开头的 ACIS 空位；R2007 及以后的未知 BL。
- R2018 多行 ATTRIB/ATTDEF：嵌入的 MTEXT；R2010 及以后的 ATTRIB 没有 `keep_duplicate_records`。
- TABLE/TABLECONTENT（以前只用于调试）和 R2010 及以后的 TABLESTYLE（三个单元格样式只读了第一个）；
  R2013 的 RENDERSETTINGS；R13/R14 图层的关闭状态、R13/R14 图元的线型。
- EED 图层（1003）句柄是大端序。
- 亚洲代码页：双字节字符的首字节（应为 `>> 8` 却写成了 `>> 4`），CP932 的单字节字符会截断字符串。

从较新版本写出 R2000
- 颜色：ByBlock 图元变成了 ByLayer；真彩色保留与之一起保存的 ACI 编号；ACI 调色板与 AutoCAD（以及 ezdxf）一致。
- EED 字符串：代码页为大端序，长度为转换后的长度；AuxHeader 的大小；VX_CONTROL 句柄。
- VERTEX/ATTRIB 的链接、MLINESTYLE 的线型编号、LWPOLYLINE 的顶点 ID 标志、以旧格式写出表格、渐变填充、MTEXT 的 `\P`、
  R13/R14 的名称、线宽和打印标志。
- 自定义图元和对象写为 ACAD_PROXY_ENTITY / ACAD_PROXY_OBJECT，而不是 POINT/DUMMY。

DXF
- TABLE 的导入导出、R2007 布局的 TABLESTYLE、代理对象、SPATIAL_FILTER、嵌入的 MTEXT、EED 1003/1004、
  HATCH 的种子点和名称、LWPOLYLINE 的宽度、真彩色（420）按 24 位 RGB、将 R2007 及以后 DXF 的 UTF-8 文本转换为 R2000 所用编码、
  `*Model_Space` 的句柄、头变量默认值、名称精确匹配、SEQEND 之后的 ATTRIB/VERTEX、R14 图纸保留较新的类、
  R2004 及以后图纸执行 `dwg2dxf --as r2000` 时不输出图元的问题。

写出 R2004 至 R2018（62 个提交）
- 与 AutoCAD 相同的 R2004 文件容器：页映射和段映射、加密的文件头、页校验和、LZ77 压缩器；
  带字符串流的 R2010 及以后的对象格式；存放 ACIS 图元 SAB 的 R2013 及以后的 AcDs 数据段（SAT 与 SAB 互相转换）。
- 将旧图纸升级到新版本：所属图元列表、PLOTSETTINGS/LAYOUT、DIMSTYLE、SPLINE、LEADER、TABLE 内容、XRECORD、
  3DSOLID 的历史句柄、AutoCAD 保存在 EED 中的真彩色。
- 过程中发现、同样影响 R2000 的修复：`bit_write_UMC` 的最短形式、`bit_write_DD` 的舍入、对象映射的分块、FileDepList 字符串、
  SPATIAL_INDEX 写出 51 MB 零字节的问题、`dwgwrite`/`dxf2dwg` 在设定目标版本之前就读取 DXF 的问题。

在一位用户的真实 AutoCAD 2018 图纸上发现的问题（6 个提交，尚未提交给上游）
- 读取：AutoCAD 省略了零页的 R2004 及以后的段（大实体的 ACIS 数据）。
- 写出 R13–R2010：大实体（64 KB 及以上）的 SAT 与 AutoCAD 一样按 4096 字节分块。
- DXF：从 $DWGCODEPAGE 起头部字符串使用图纸代码页；SummaryInfo 字符串以及布局名和字典名不再从 UTF-8 转换两次（日文乱码）。
- 从 R2007 及以后写出 R2000：头部字符串（HYPERLINKBASE、STYLESHEET、FINGERPRINTGUID 等）完整写出，而不是只剩第一个字符。

每个提交说明都写明了哪里出错、表现为什么现象、以及用哪张图纸可以看出来。

## 下载

每个[发布版本](https://github.com/monyuonyu/libredwg-fixes/releases)都附有可用于上游 `libredwg-0.14.8601.tar.xz` 的补丁：
`libredwg-fixes-patches.tar.gz`（每个修复一个文件，用于 `git am`）和 `libredwg-fixes.patch`（单个文件，用于 `patch -p1`）。
也可以直接克隆本仓库。

## 构建

与上游 LibreDWG 相同（参见 `README` 和 `INSTALL`）：

```sh
git clone https://github.com/monyuonyu/libredwg-fixes.git
cd libredwg-fixes
sh ./autogen.sh
./configure --disable-bindings
make
make check
```

## 许可证

与 LibreDWG 相同，采用 GNU GPL v3 或更高版本。所有修改都已提供给上游项目。
