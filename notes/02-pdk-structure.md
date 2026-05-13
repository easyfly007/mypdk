# 02 — PDK 文件结构地图（libs.tech vs libs.ref）

## 目标

打开 `~/.volare/sky130A`，把里面的目录树搞清楚：哪些文件给哪个工具用、哪些是工艺本身的"知识"、哪些是 IP 库。读完这一篇，再看任何 sky130 教程里"把 XXX 加到 magicrc"或者"include sky130.lib.spice"这种话，就不会卡。

## 顶层布局

```text
~/.volare/sky130A/
├── SOURCES                        # 一行文本：open_pdks 0fe599b2…，记录构建用的源码版本
├── .config/nodeinfo.json          # 工艺节点元数据（节点名、变体、license 等）
├── libs.tech/                     # "工艺知识"——给各 EDA 工具用的技术文件
│   ├── magic/      netgen/   klayout/   ngspice/   xschem/
│   ├── openlane/   combined/  irsim/     qflow/     xcircuit/
└── libs.ref/                      # "IP 库"——具体 cell 的多视图集合
    ├── sky130_fd_pr/              # 基础工艺器件（MOS/R/C/D）
    ├── sky130_fd_io/              # IO pad
    ├── sky130_fd_sc_hd/           # 高密度标准单元
    ├── sky130_fd_sc_hvl/          # 高压标准单元
    ├── sky130_ml_xx_hd/           # ML example
    └── sky130_sram_macros/        # SRAM 宏
```

一句话概括分工：

| | libs.tech | libs.ref |
| --- | --- | --- |
| 内容 | 工艺级"规则 + 配置"：层定义、DRC 规则、SPICE 模型、工具 rc 入口、设备生成脚本 | cell 级"数据"：每个具体单元的多种视图（gds / lef / lib / mag / spice / cdl / verilog / sym） |
| 粒度 | 整个工艺一份 | 每个 cell 一组文件 |
| 谁动 | 几乎只读，由 open_pdks 生成 | 同样只读，但量级大、按库分发 |
| 改名建议 | "tech files & tool configs" | "cell views, organized by library" |

记住一条：**`libs.tech` 教工具"怎么看 sky130"，`libs.ref` 给工具"看的具体东西"。**

## libs.tech 详解

### magic/ — Magic 版图编辑器 + DRC + 提取

```text
sky130A.magicrc          ★ 入口脚本：被 magic 启动时 source，读 PDK_ROOT 后加载下面几个文件
sky130A.tech             ★ 技术文件：6101 行，定义所有层（diff/poly/li/met1…）、DRC 规则、提取规则
sky130A-GDS.tech         GDS 层映射（layer/datatype ↔ magic 层名）
sky130A.tcl              器件生成器：执行 magic 的 PCells（如点 "nfet" → 调出参数化器件）
sky130A-BindKeys         可选快捷键绑定
run_standard_drc.py      批量跑 DRC 的封装脚本
check_antenna.py         天线规则检查
check_density.py         密度检查
generate_fill.py         填充 dummy 生成
bump_bond_generator/     bump 焊球生成器
seal_ring_generator/     密封环生成器
```

`sky130A.magicrc` 是整套 magic 流程的"龙头"。它读 `$PDK_ROOT` 决定路径，然后 `tech load sky130A.tech` 把规则加进来。所有 magic 教程都让你在工作目录放一个 `.magicrc` 指向（或复制）这个文件——本质就是这样。

### netgen/ — LVS

```text
setup.tcl                # 通用 netgen 设置
sky130A_setup.tcl        ★ sky130 的 LVS 设置：器件等价类、参数容差、忽略列表
```

LVS 时 magic 会先 extract 出 SPICE，然后调 netgen 拿 `sky130A_setup.tcl` 当规则比对。**04 / 06 章会展开。**

### klayout/ — KLayout 版图查看 / DRC / LVS

```text
tech/sky130A.lyt   ★ KLayout 工艺定义（layer 映射、显示）
tech/sky130A.lyp     图层显示样式 (颜色/填充)
tech/sky130A.map     GDS layer/datatype → 名字 的纯文本映射
drc/sky130A.lydrc    KLayout DRC 主入口（轻量、面向签核的规则集）
drc/sky130A_mr.drc   多线程版 DRC
drc/zeroarea.rb.drc  零面积形状检查
lvs/sky130.lvs       KLayout LVS 规则
lvs/sky130.lylvs     LVS GUI 配置
pymacros/            参数化器件 Python 实现（KLayout 内的 PCell）
```

magic 和 KLayout 各有一套独立的 DRC 引擎，**不互相替代**：magic 的 DRC 偏交互、增量、提取友好；KLayout 的 DRC 偏批处理、签核精度。同一份工艺，两份规则各自维护。

### ngspice/ — SPICE 仿真模型

```text
sky130.lib.spice    ★ 顶层入口：定义 .lib tt / ss / ff / sf / fs / leak 等 corner
corners/            各 corner 下的 MOSFET 参数文件（tt.spice, ff.spice, …）
r+c/                电阻 + 电容 corner 组合（res_typical__cap_typical.spice 等）
parameters/         全局 .param 默认值
parasitics/         寄生模型
capacitors/         电容相关子模型
sky130_fd_pr__model__*.model.spice   # 各类器件的根 model 卡（被 corners/ 里的文件 include）
sonos*/             SONOS 非挥发器件
spinit              ngspice 启动文件片段
```

仿真时典型用法是 `.lib "/path/to/sky130.lib.spice" tt`，**04 章会详细讲 corner 体系**。

### xschem/ — 原理图

```text
xschemrc            ★ xschem 启动配置：设 XSCHEM_LIBRARY_PATH，把 sky130_fd_pr 加进来
sky130_fd_pr/       ★ 工艺器件的 .sym 符号库（nfet_01v8.sym, pfet_01v8.sym, res, cap, diode…）
sky130_stdcells/    标准单元符号
sky130_tests/       自带测试电路
stdcells/  mips_cpu/  scripts/  ...
sky130_fd_pr.patch  对上游符号库的本地补丁
```

xschem 用法：`cp xschemrc 到工作目录` 或者设 `XSCHEM_LIBRARY_PATH`，启动时它就能找到 sky130 的器件符号。**05 章会展开 xschemrc**。

### openlane/ — 数字流程

只跑模拟的话基本不碰这里。包含 OpenLane 用的标准单元配置、寄生提取规则（max/min/nom × calibre/magic/spef_extractor 三套）等。

### combined/ — 共享/兼容层

```text
sky130.lib.spice    与 ngspice/sky130.lib.spice 同名同内容（方便 ngspice 之外的工具 include）
corners/  parameters/  r+c/  rescap/  sonos*/  spinit
continuous/         "连续型"模型变体
test/               测试卡
```

可以理解为"非 ngspice 工具也用得上的那部分 SPICE 资料"被打包到 combined。日常引用走 `libs.tech/ngspice/sky130.lib.spice` 就好。

### irsim/ — 老牌开关级仿真器

```text
sky130A_{tt,ss,ff}_{nom,high,low}_{n40,27,125}.prm   # 27 个 corner × 温度 × 电压参数文件
```

模拟 IC 视角下基本用不到，IRSIM 主要给数字流程做快速时序估算。

### qflow / xcircuit — 备用工具

qflow：老牌数字流程脚本（被 OpenLane 取代）。xcircuit：老的原理图工具（被 xschem 取代）。**模拟 IC 学习路径里都可以略过。**

## libs.ref 详解

每个 IP 库都用同一套"视图（view）"目录结构。**视图 = 同一个 cell 在不同工具/不同抽象层下的表示**。

| 视图目录 | 给谁用 | 内容 | 抽象层级 |
| --- | --- | --- | --- |
| `gds/` | 任何 GDS 工具（KLayout, magic, calibre…）| 二进制版图 GDSII | 物理（签核）|
| `mag/` | magic | Magic 原生格式版图（含 well、衬底等完整层）| 物理（编辑用）|
| `maglef/` | magic | 抽象后的 Magic 版图——只保留 pin 和 obs，类似 LEF | 物理抽象 |
| `lef/` | 数字 P&R / KLayout | LEF：每个 cell 的尺寸、pin 位置、阻塞层 | 物理抽象 |
| `techlef/` | 数字 P&R | technology LEF：层名、走线规则、单位（与具体 cell 无关）| 物理抽象 |
| `spice/` | SPICE 仿真 | 提取后的子电路（含寄生）或源码网表 | 电气 |
| `cdl/` | LVS | "干净"的器件级网表（无寄生，用于 LVS）| 电气 |
| `lib/` | STA / 综合 | Liberty 时序/功耗库（按 PVT corner 分文件）| 时序模型 |
| `verilog/` | 仿真 / 综合 | 行为级 / 门级 Verilog 模型 | RTL/门级 |
| `sym/`（在 libs.tech/xschem 下） | xschem | 原理图符号 | 原理图 |

### sky130_fd_pr/ — 基础工艺器件（"primitive devices"）（75 MB）

模拟设计的"砖头"。`fd_pr` = **f**oundry **d**evice **pr**imitives。

```text
spice/     675 个 .model.spice：MOS 4 个电压档（1.8/3.3/5.0/20V）、电阻、MIM 电容、VPP 电容、变容、二极管、双极管…
mag/       290 个 .mag：器件的 magic 版图（多是固定尺寸 / 模板）
maglef/    抽象版
gds/       同上的 GDS 视图
lef/       sky130_fd_pr.lef 单文件——所有 PR 器件的 LEF
```

注意它**没有 lib/cdl/verilog**：因为 PR 器件是模拟基本元件，不参与数字综合，也不需要 Liberty。

对应的**原理图符号**在 `libs.tech/xschem/sky130_fd_pr/` 下（78 个 .sym），物理上分开但概念上配套。

### sky130_fd_sc_hd/ — 高密度标准单元（350 MB）

数字设计核心库。视图齐全：`cdl / gds / lef / lib / mag / maglef / spice / techlef / verilog` 一个不落。`lib/` 下 18 个 Liberty 文件覆盖 ff/ss/tt × 多种温度电压组合。

- `gds/sky130_fd_sc_hd.gds` 是**单个大文件**——所有 cell 拼一起。
- `techlef/sky130_fd_sc_hd__{min,nom,max}.tlef` 是这个标准单元库**专用的** tech LEF（与工艺 tech LEF 互补，包含 metal stack 提取参数）。
- 模拟设计中也会用：做 buffer / inverter 拉时钟、做电平转换等。

### sky130_fd_sc_hvl/ — 高压标准单元（116 MB）

5V tolerant，给跨电压域电路用。

### sky130_fd_io/ — IO pad（352 MB）

封装 pad、ESD 结构。tapeout 时才用。

### sky130_sram_macros/ — SRAM 宏（152 MB）

预编译好的 SRAM 块，按尺寸命名。

### sky130_ml_xx_hd/ — ML example（412 KB）

机器学习课程的小示例库，可忽略。

## 三个容易混的关系

### 1. libs.tech/{magic,klayout,ngspice} 和 libs.ref/* 怎么协作？

```text
启动 magic
   └─ source $PDK_ROOT/sky130A/libs.tech/magic/sky130A.magicrc
        ├─ tech load …/libs.tech/magic/sky130A.tech         ← 工艺规则
        ├─ source …/libs.tech/magic/sky130A.tcl              ← 器件生成器
        └─ addpath …/libs.ref/sky130_fd_pr/mag …             ← cell 数据来源
```

`libs.tech` 把工具配置好，`libs.ref` 把 cell 喂给它。两边缺一不可。

### 2. mag vs maglef vs lef vs techlef

- **mag**：完整版图，能编辑，含所有层 → 用于查看/修改 cell 本身。
- **maglef**：抽象，只保留对外接口 → 当你**实例化别人的 cell** 时用，省内存。
- **lef**：与 maglef 等价的标准格式 → 给非 magic 工具（P&R / KLayout）用。
- **techlef**：和具体 cell 无关，描述**金属层栈**（线宽规则、via、单位）→ 数字工具读这个建立布线模型。

### 3. sky130.lib.spice vs 单个 *.model.spice

`sky130.lib.spice` 是**入口**，里面 `.lib tt` 包一层，`.include "corners/tt.spice"` 把 MOS 参数拉进来，再 include 电阻/电容/特殊器件。**自己写 testbench 时只 `.lib sky130.lib.spice tt` 即可，不要直接 include 单个 model 文件**，否则缺 corner 参数、缺 .param 默认值，行为不可控。

## 几个容易踩的坑

- **magicrc 复制 vs 引用**：很多教程让你 `cp sky130A.magicrc .` 到工作目录。复制是历史包袱——只要 `PDK_ROOT` 设对，magic 可以直接 `-rcfile $PDK_ROOT/sky130A/libs.tech/magic/sky130A.magicrc` 启动。复制副本容易和 PDK 升级脱节。
- **maglef vs mag 选错**：编辑顶层版图实例化 nfet 时应该读 `maglef/`（抽象），如果设 `MAGTYPE=mag` 会把 nfet 内部 well/contact 全展开，慢且占内存。`MAGTYPE` 环境变量控制。
- **不要直接 include 单个 .model.spice**：见上面"三个易混"的第 3 条。
- **GDS 单层 vs 多层**：`sky130_fd_sc_hd/gds/` 是一个大 GDS 包含所有 cell；而 `sky130_fd_pr/gds/` 是按 cell 散开。读 GDS 库的脚本要兼容两种组织方式。
- **xschem 符号目录不在 libs.ref**：xschem 的 sky130_fd_pr 符号被放在 `libs.tech/xschem/sky130_fd_pr/` 而不是 `libs.ref/sky130_fd_pr/`。原因是 .sym 与"工具配置"耦合更紧（坐标系、属性约定都是 xschem 特有的）。

## 下一步

- **笔记 03**：进 `sky130_fd_pr/`，把 MOS / 电阻 / MIM / VPP / 二极管 各类器件挑代表展开看：命名规则、参数维度、`.sym` 与 `.model.spice` 怎么对应、典型尺寸约束。
- 在写 03 之前，先把 `PDK_ROOT` / `PDK` 写进 `~/.bashrc`，并启动一次 `magic -d XR` + `xschem` 做最小验证。
