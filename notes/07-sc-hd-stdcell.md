# 07 — 标准单元库 sky130_fd_sc_hd 简介

## 目标

前面六章都在讲 `sky130_fd_pr`（模拟基本器件）。最后一章看看数字标准单元库 `sky130_fd_sc_hd`：

- 这个库长什么样、441 个 cell 怎么分类、命名规则；
- "fd_sc_hd" 几个字母分别什么意思，sky130 还有几个并列的标准单元库；
- 模拟 IC 设计师**什么时候**会真正用到它（不是为了做数字综合）；
- 9 种视图（cdl/gds/lef/lib/mag/maglef/spice/techlef/verilog）在数字流程里分别给谁用。

参考 [[02-pdk-structure]] 里 libs.ref 的 9 视图表，[[03-fd-pr-devices]] 里 fd_pr 的视图（只 5 种）做对照。

## 库的命名拆开看

```text
sky130_fd_sc_hd
       │   │   └── hd  = high density（高密度、低电压、面积优先）
       │   └────── sc  = standard cells
       └────────── fd  = foundry data（直接来自 SkyWater 原始数据，非派生）
```

sky130A 一共带 7 个标准单元库：

| 库 | 主打 | 何时用 |
| --- | --- | --- |
| **sky130_fd_sc_hd** | 高密度，1.8 V，**默认** | 数字逻辑主战场 |
| sky130_fd_sc_hdll | High-Density Low-Leakage | 漏电敏感的低功耗块 |
| sky130_fd_sc_hs | High-Speed | 速度优先（更大单元）|
| sky130_fd_sc_ms | Medium-Speed | 折中 |
| sky130_fd_sc_ls | Low-Speed | 慢速、最低功耗 |
| sky130_fd_sc_lp | Low-Power | 已废弃，被 hdll 取代 |
| **sky130_fd_sc_hvl** | High-Voltage Logic（3.3 V）| 电平转换 / 数模混合接口 |

本章只详细讲 `hd`，其它库结构同构。

## 文件视图（9 种全有）

```text
sky130_fd_sc_hd/
├── cdl/        1 个文件      LVS 用的"干净"网表，所有 cell 拼一起
├── gds/        1 个 .gds     所有 cell 的版图打包成一个 GDS
├── lef/        2 个 .lef     抽象版图（macro footprint + pin）
├── lib/        18 个 .lib    Liberty 时序模型，按 PVT corner 分文件
├── mag/        441 个 .mag   Magic 原生版图（完整层）
├── maglef/     441 个 .mag   Magic 抽象版图（只 pin + obs）
├── spice/      5 个 .spice   Efabless 补加的几个特殊 cell（ef_sc_hd 前缀）
├── techlef/    6 个 .tlef    technology LEF（metal stack 规则）
└── verilog/    4 个 .v       Verilog 行为模型 + 黑盒接口
```

各视图分工已在 [[02-pdk-structure]] 给过对照表，这里补 `fd_sc_hd` 特有的几点：

### lib/（Liberty 时序文件）

18 个文件名编码 `corner_温度_电压`：

```text
sky130_fd_sc_hd__ff_n40C_1v95.lib       FF corner, -40°C, 1.95V
sky130_fd_sc_hd__ff_100C_1v65.lib       FF corner, 100°C, 1.65V
sky130_fd_sc_hd__ss_n40C_1v28.lib       SS corner, -40°C, 1.28V
sky130_fd_sc_hd__tt_025C_1v80.lib       TT corner, 25°C, 1.80V  ← 中心点
sky130_fd_sc_hd__tt_100C_1v80.lib       TT corner, 100°C, 1.80V
*_ccsnoise.lib                          带 noise 数据的高级版（少数 corner 才有）
```

文件内容是 Liberty 格式：

```liberty
library ("sky130_fd_sc_hd__tt_025C_1v80") {
    delay_model : "table_lookup";
    time_unit : "1ns";
    voltage_unit : "1V";
    operating_conditions ("tt_025C_1v80") {
        voltage : 1.8;  process : 1.0;  temperature : 25.0;
    }
    cell ("sky130_fd_sc_hd__inv_1") {
        pin (A) { direction : input; capacitance : ... }
        pin (Y) { direction : output;
            timing () { ... look-up table ... }
        }
    }
    ...
}
```

STA / 综合 / P&R 工具读这个。**模拟 SPICE 仿真不读 Liberty**，仿真要找 `cdl/` 或 `mag/` 提取出来的 SPICE。

### techlef/（technology LEF，6 个）

```text
sky130_fd_sc_hd__nom.tlef           ★ 默认（典型 metal 栈）
sky130_fd_sc_hd__min.tlef           min RC（最快）
sky130_fd_sc_hd__max.tlef           max RC（最慢，签核）
sky130_fd_sc_hd__rram__{nom,min,max}.tlef  含 ReRAM 选项时用（对应 sky130B 体系）
```

这是 [[02-pdk-structure]] 里 "techlef = 与具体 cell 无关的金属栈定义" 的实物：metal 层名、走线规则（minwidth、minspacing、via）、单位。**与 cell 无关**，所以一份 techlef 配整个 `hd` 库。`min/nom/max` 三档对应金属 RC 的 corner。

### verilog/

```text
primitives.v                  原语（UDP）：基本组合逻辑、时序原语
sky130_fd_sc_hd.v             功能模型（行为级，可仿真）
sky130_fd_sc_hd__blackbox.v   黑盒（只 module 接口，无内部逻辑）
sky130_fd_sc_hd__blackbox_pp.v   带 power pin 的黑盒
```

数字综合 / 门级仿真用前两个；P&R 工具读第三个（不需要功能、只要接口）。

### cdl / spice

`cdl/` 一个大文件包含所有 cell 的"无寄生 SPICE"，专门给 LVS 用（[[06-drc-lvs-flow]] 里 netgen 比对的标准网表就是它）。

`spice/` 下只有 5 个 `sky130_ef_sc_hd__*` 文件（**前缀是 `ef` 不是 `fd`**）—— Efabless 给 SkyWater 库追加的几个 decap / fill 单元。它们是"扩展"，不在原始 SkyWater 数据里。

## 441 个 cell 怎么分类

按功能名前缀粗分：

| 前缀 | 含义 | 典型 cell | 数量 |
| --- | --- | --- | --- |
| `inv_*` | 反相器 | inv_1, inv_2, inv_4, inv_6, inv_8, inv_12, inv_16 | 7 |
| `buf_*` | 缓冲器 | buf_1 / 2 / 4 / 6 / 8 / 12 / 16 | 7 |
| `clkinv_*` / `clkbuf_*` | 时钟专用（更对称的上升/下降）| | 5 + 5 |
| `nand2 / nor2 / and2 / or2 / xor2 / xnor2 / mux2` | 2 输入组合逻辑 | 各 ~4 种驱动 | |
| `aXXoi / oXXai` | AOI / OAI 复合门 | `a21oi_1`（2-input AND, 1-input OR, INV 输出）| 大量 |
| `dfxtp / dfrtp / dfstp` | 寄存器 | D-FF, 各种 reset/set 极性 | |
| `sdfxtp / sdfrtp / sdfstp / sedfxtp` | 扫描寄存器（带 SI / SE 引脚）| | 12 |
| `sdlclkp` | 时钟门控 latch | | 3 |
| `lpflow_*` | 低功耗专用（电源切换 / iso cell）| 34 |
| **`diode_2`** | 天线二极管（修 antenna 违例）| 单个固定尺寸 | 1 |
| **`decap_*`** | decoupling cap（电源滤波）| decap_3/4/6/8/12 | 5 |
| **`tap_*` / `tapvgnd*` / `tapvpwrvgnd*`** | well tap（拉 substrate / nwell）| 5 |
| **`fill_*`** | 空白填充 | fill_1/2/4/8 | 4 |
| **`conb_1`** | tie hi / tie lo（接固定电平）| | 1 |

最后 5 类（diode / decap / tap / fill / conb）是**模拟设计也会借用**的——下一节展开。

### 命名规则：`<功能>_<驱动强度>`

- 数字部分越大 = 驱动越强（W 越宽、可带载越大、面积越大、漏电也越多）；
- 不是连续整数：典型只有 `1, 2, 4, 6, 8, 12, 16`；
- 部分类型只有 `1`（如 `mux4_1`），表示这个功能只有一个驱动版本。

例子：`sky130_fd_sc_hd__inv_2` = 高密度反相器，驱动 = 2（约 2 倍 minimum-size NFET 的载流能力）。

## 模拟设计什么时候用 fd_sc_hd

模拟 IC 工程师**不需要做数字综合**，但下面几个场合会直接把 `fd_sc_hd` 单元当"成品零件"塞进版图：

### 1. 时钟 / 信号 buffer

PLL、采样保持、SAR ADC 里满地都是要"驱动一组同步翻转"的时钟。手画 buffer 很浪费时间，直接 `sky130_fd_sc_hd__clkbuf_8` 或 `clkbuf_16` 就行——上升下降时间已经匹配，foundry 已表征过。

### 2. 数字接口 / control logic

如 SAR ADC 的逐次逼近控制、Σ-Δ ADC 的 decimator 简版。手画 6 个 latch 不如直接放 6 个 `dfxtp_1`。

### 3. 天线 diode

magic 报天线违例时，在长走线中插 `sky130_fd_sc_hd__diode_2`。这是**专门做这件事**的标准单元，比从 `fd_pr` 拽一个二极管又对又快。

### 4. Decoupling cap

```text
decap_3 / decap_4 / decap_6 / decap_8 / decap_12   # 数字 = 高度 unit
```

数模混合芯片里在 VDD / VSS 之间填一片 decap 是常规操作。`decap_*` cell 内部是大尺寸 MOS 接成电容，会自动跟标准单元行高对齐，**布到 cell row 缝里**很方便。

### 5. Well tap

模拟版图的"接 substrate / 接 nwell"用大块 substrate contact + nwell contact 自己拉。数字行里的 `tap_*` / `tapvgnd_1` / `tapvpwrvgnd_1` 干同样的事，但是**预画好的最小成本版本**——周期插一个就把 latch-up 风险压下去了。

### 6. Tie hi / tie lo

某些数字 IP 的输入要拉到 VDD / VSS 但不能直接短接（DRC / antenna 考虑）。`conb_1` 提供 `HI` 和 `LO` 两个输出 pin，分别就是 VDD / VSS。

### 7. 一致的行高（cell row）便于混排

`hd` 库的所有 cell 都站在同一个 `SITE unithd` 上（高度 2.72 µm，对应 7 个 metal-1 track）。混合数模时，**把模拟 cell 也设计成 2.72 µm 高**，能直接和数字行一起走自动 P&R。

## 9 种视图里模拟设计真正用到的

| 视图 | 模拟设计是否常用 |
| --- | --- |
| `mag` / `maglef` | ✅ magic 里实例化标准单元时用 maglef |
| `gds` | ✅ 最终拼版用 |
| `lef` | 数字流程才用 |
| `lib` | 数字 STA 才用，模拟不读 |
| `cdl` | LVS 时用（[[06-drc-lvs-flow]]）|
| `spice` | 几乎不用（只有 5 个 ef 文件） |
| `techlef` | 数字流程才用 |
| `verilog` | 数字综合才用 |

模拟视角看 `fd_sc_hd`，**有用的就是 mag/maglef/gds/cdl**。

## 一个 cell 的多视图对应（以 inv_2 为例）

```text
sky130_fd_sc_hd__inv_2
├── mag/inv_2.mag                 完整版图，编辑可见
├── maglef/inv_2.mag              抽象，只 pin + obs
├── gds/sky130_fd_sc_hd.gds       打包，含本 cell
├── lef/sky130_fd_sc_hd.lef       MACRO sky130_fd_sc_hd__inv_2 段
├── cdl/sky130_fd_sc_hd.cdl       .subckt sky130_fd_sc_hd__inv_2 段
├── lib/*.lib                     18 个 corner 各含 cell ("sky130_fd_sc_hd__inv_2")
└── verilog/sky130_fd_sc_hd.v     module sky130_fd_sc_hd__inv_2(A, Y, ...);
```

只有 `mag/maglef` 是**每个 cell 一个文件**，其他视图都是"一个大文件包含所有 cell"。这影响版本管理：改一个 cell 的 mag 只动一个文件；但 lib/gds/lef 改任何一个 cell 都触发整个大文件更新。

## 与 sky130_fd_io / sky130_sram_macros 的关系

完整的数字流程会用到三个层级：

```text
sky130_fd_sc_hd       ← 标准单元（这一章）
sky130_fd_io          ← I/O pad（与 die 外部接口）
sky130_sram_macros    ← SRAM 大块宏（预编译好的存储器）
```

模拟设计偶尔会用到 `sky130_fd_io` 里的 ESD 结构 / pad 单元。`sky130_sram_macros` 几乎只跟数字内存子系统相关。

## 几个容易踩的坑

- **`ef_sc_hd` vs `fd_sc_hd` 前缀**：库主体是 `fd_sc_hd`（来自 SkyWater）；`spice/` 下的 5 个文件是 `ef_sc_hd`（Efabless 后加）。LVS / 综合脚本里如果只 grep `fd_sc_hd` 会漏掉这几个 cell。
- **mag vs maglef 在顶层实例化**：[[03-fd-pr-devices]] 里 MOS 也说过——实例化 inv_2 应该读 maglef（抽象），不要 mag。`MAGTYPE=maglef` 在 [[05-rc-entrypoints]] 里设过。
- **`*_ccsnoise.lib` 不要无脑全选**：CCS noise 模型文件比普通 NLDM 大一个量级，只有几个 corner 才需要；做高精度噪声分析时再加，否则用普通 `.lib`。
- **lpflow 是低功耗专用，混用要小心**：`lpflow_*` cell 涉及多电源域（VPB / VNB / VPWR / VGND 都可能不接到默认）。普通设计里**不要随便实例化** lpflow 的东西，会破坏标准电源域假设。
- **行高一致才能混进数字行**：手画的模拟 cell 高度不对齐 `unithd` 站（2.72 µm）就只能放在芯片单独区，不能跟标准单元行混排。要混排就照 hd 站定高度（高度自定义可以是 N × 2.72 µm 倍数）。
- **Liberty `default_max_transition: 1.5` 是 ns**：默认是 1.5 ns，相当严格。要把模拟 cell 当数字 cell 用（外部跑 STA）时要写自己的 .lib，注意单位。
- **不要在 mag/ 下直接编辑**：那 441 个文件是 PDK 一部分。要改某个 cell 就先 `cp` 到自己目录、改前缀（如 `mylib__inv_custom`），不要污染 PDK。

## 全书完结 & 下一步

这是第七篇笔记。整本"基于 PDK 的模拟设计入门"路线图（README "目标全流程"）走到这里：

| 阶段 | 笔记 | 状态 |
| --- | --- | --- |
| PDK 安装与版本 | [01](01-install-sky130.md) | ✅ |
| 文件结构 | [02](02-pdk-structure.md) | ✅ |
| 模拟器件库 fd_pr | [03](03-fd-pr-devices.md) | ✅ |
| 仿真模型 + corner | [04](04-sim-models-corners.md) | ✅ |
| 工具入口 magicrc/xschemrc | [05](05-rc-entrypoints.md) | ✅ |
| DRC / LVS 流程 | [06](06-drc-lvs-flow.md) | ✅ |
| 标准单元库 | 本篇 | ✅ |
| 实战 1：原理图 + 仿真 | 08（待写） | ⏳ |
| 实战 2：版图 → 后仿 | 09（待写） | ⏳ |
| PEX 提取 + 签核 | 10（待写） | ⏳ |

"知识层"算是搭完了；后续应该转入**动手做一个具体电路**（推荐 5-T OTA 或 2-stage Miller 补偿 OTA），把 04 的 corner 仿真、06 的 DRC/LVS、本篇的 buffer/decap 都串一遍。
