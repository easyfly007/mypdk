# 03 — 器件库 sky130_fd_pr：MOS / 电阻 / 电容 / 二极管

## 目标

把 `sky130_fd_pr/` 里的器件按"族"看明白：

- 命名规则怎么读（碰到 `pfet_g5v0d10v5` 不要发懵）；
- 每族给哪几个 SPICE 文件、xschem 符号长什么样；
- 端口、参数（W / L / nf / mult / area / mismatch）大概有什么；
- 哪几个是日常模拟设计真正会用到的，其余的什么时候才考虑。

参考自 [[02-pdk-structure]] —— SPICE 模型在 `libs.ref/sky130_fd_pr/spice/`，符号在 `libs.tech/xschem/sky130_fd_pr/`。

## 命名规则速读

```text
sky130_fd_pr__<class>_<voltage>[_<variant>][_<modifier>][_<geom>]
                │       │         │           │             │
                │       │         │           │             └── rf_* 把固定 W/L/M 编进名字：W3p00L0p15M04 = W=3.00µm L=0.15µm M=4
                │       │         │           └── 比如 _no_rs、_iso、_withptap、_lvt、_hvt、_nvt、_zvt
                │       │         └── 二级变种：esd / rf / special
                │       └── 电压档（V）：01v8 / 03v3 / 05v0 / g5v0d10v5 / g5v0d16v0 / 11v0 / 20v0
                └── 器件类：nfet / pfet / res / cap / diode / npn / pnp / ind
```

数字里 `p` 是小数点：`01v8 = 1.8V`、`g5v0d10v5 = 5V gate / 10.5V drain`、`W2p00L0p15 = W=2.00 µm L=0.15 µm`。

**Vt 后缀**：
- 无后缀 = standard / regular Vt
- `lvt` = low-Vt（更快、漏电更高）
- `hvt` = high-Vt（更慢、漏电低，pfet 才有）
- `nvt` = native（接近 0V 阈值，主要 03v3/05v0/20v0 上有）
- `zvt` = zero-Vt（20V 上才有）

## MOSFET

电压族总览（关注前两族就够日常模拟 90%）：

| 器件 | 用途 | Vt 变种 | 备注 |
| --- | --- | --- | --- |
| `nfet_01v8` / `pfet_01v8` | **核心 1.8V CMOS**，模拟主战场 | rvt / lvt / hvt（pfet 三档；nfet 只有 rvt+lvt） | L_min ≈ 0.15 µm，W_min ≈ 0.42 µm |
| `nfet_g5v0d10v5` / `pfet_g5v0d10v5` | **5V IO / 中等电源**，cascode/电源管理 | — | LDMOS：5V gate / 10.5V drain |
| `nfet_03v3_nvt` | 3.3V native，开关用 | nvt | 没有对应的 03v3 标准管 |
| `nfet_05v0_nvt` | 5V native | nvt | — |
| `nfet_g5v0d16v0` / `pfet_g5v0d16v0` | 16V 高压 LDMOS | — | — |
| `nfet_20v0` / `pfet_20v0` | 20V 高压 LDMOS | nvt / zvt / nvt_iso / reverse_iso | — |

**端口固定四端**：`d g s b`，body 永远显式接出（不像有些 PDK 自动接衬底）。

**子电路参数**（所有 MOS 通用，从 `*.pm3.spice` 的 `.subckt` 行抓出来）：

| 参数 | 含义 | 单位 / 默认 |
| --- | --- | --- |
| `l`, `w` | 单指沟道长度 / 宽度 | µm，默认 1 |
| `nf` | 指数（finger 数）| 默认 1 |
| `mult` | 并联倍率（整器件复制）| 默认 1 |
| `ad`, `as` | drain / source 扩散区面积 | µm² |
| `pd`, `ps` | drain / source 扩散区周长 | µm |
| `nrd`, `nrs` | drain / source 串联电阻方块数 | 默认 0 |
| `sa`, `sb`, `sd` | well-proximity（边距）| µm |

实际 W = `w * nf * mult`。布尺寸时**永远显式写 nf**，仅靠加大 W 不分指对版图和提取都不利。

### 三个修饰前缀（看到名字别慌）

- **`special_*`**（如 `special_nfet_01v8`、`special_nfet_pass`）：W < 0.42 µm 的"窄沟道" binning，专门给 SRAM bitcell / latch 用。手写模拟电路里基本碰不到。
- **`esd_*`**（如 `esd_nfet_01v8`）：等价于普通管，但 W 走 binning 之外的"ESD 区间"。打 ESD pad 或者高功率器件做大小写时用。
- **`rf_*`**（如 `rf_nfet_01v8_aM04W5p00L0p15`）：**fixed-geometry** RF 测量提取的器件。W/L/M 直接编在文件名里，不能自由扫尺寸；用于精确 RF 仿真。整个 `rf_*` 族占了 spice/ 一半的文件数，但日常基带模拟不用。

### 每个 MOS 对应的文件层次

以 `nfet_01v8` 为例，文件组：

```text
sky130_fd_pr__nfet_01v8.pm3.spice                ★ wrapper subckt（端口 + 参数声明）
sky130_fd_pr__nfet_01v8__tt.corner.spice         tt corner 的 .model 卡（BSIM 参数 × 63 个 bins）
sky130_fd_pr__nfet_01v8__{ss,ff,sf,fs}.corner.spice  其它 corner
sky130_fd_pr__nfet_01v8__leak.corner.spice       leakage corner
sky130_fd_pr__nfet_01v8__mismatch.corner.spice   Monte Carlo 失配参数
sky130_fd_pr__nfet_01v8__subvt_mismatch.corner.spice  亚阈失配
sky130_fd_pr__nfet_01v8__tt_correln/p.corner.spice  n/p 关联（用于全片仿真）
sky130_fd_pr__nfet_01v8__wafer.corner.spice      wafer-level 变化（全片同向漂）
sky130_fd_pr__nfet_01v8.pm3.spice                与上同名（pm3 = parasitic model 3 端封装）
sky130_fd_pr__nfet_01v8__tt.pm3.spice            corner × pm3 组合
```

**职责分层**：
- `*.pm3.spice` 定义 `.subckt sky130_fd_pr__nfet_01v8 d g s b`，内部实例化 `msky130_fd_pr__nfet_01v8 ... sky130_fd_pr__nfet_01v8__model ...`，把对外的子电路接口和内部 BSIM 器件接口分开。
- `*__<corner>.corner.spice` 提供 `.model sky130_fd_pr__nfet_01v8__model nmos level=... ...`，因为 sky130 用 binned model，里面通常是 63 段（L × W 的 bin matrix）。
- 你自己**不直接 include 这些文件**，都是被 `libs.tech/ngspice/sky130.lib.spice` 串起来的。

### xschem 符号（一个器件 → 多个符号）

`libs.tech/xschem/sky130_fd_pr/` 下经常一个器件对应几张符号：

| 符号 | 用途 |
| --- | --- |
| `nfet_01v8.sym` | 标准 4 端 nfet（body 显式画出来）|
| `nfet3_01v8.sym` | "3 端"版本（body 内部默认接 VGND/VPWR）|
| `nfet_01v8_nf.sym` | 暴露 `nf` 指数参数的版本（方便扫指数）|
| `nfet_01v8_esd.sym` | ESD 版 |
| `nfet_01v8_lvt.sym`, `nfet_01v8_lvt_nf.sym` | LVT 对应两种 |

按需选用，不要混着画（同一张图里 nfet3 和 nfet 不要乱配，body 接错很难查）。

## 电阻

| 器件 | 类型 | sheet R | 用途 |
| --- | --- | --- | --- |
| `res_high_po` | 高电阻 poly | ~ 317 Ω/sq | 模拟核心电阻（线性度好、温漂中等）|
| `res_xhigh_po` | 超高电阻 poly | ~ 2000 Ω/sq | 大阻值偏置 / 启动电路 |
| `res_generic_nd` | N+ 扩散 | ~ 120 Ω/sq | 一般偏置（占面积小，但温/电压系数差）|
| `res_generic_pd` | P+ 扩散 | — | 同上 P 型 |
| `res_iso_pw` | 隔离 P-well | — | 想避免衬底耦合时 |

每个 poly 电阻有 **可变 W** 版本（`res_high_po`，参数 w/l）和 **固定 W** 版本（`res_high_po_0p35 / 0p69 / 1p41 / 2p85 / 5p73` µm，只剩 l 可调）。**强烈推荐用固定 W 版本** —— foundry 只对这几个 W 做了 binning 与 mismatch 数据，自由 W 版会用插值近似。

**端口**：`r0 r1 b`（两端 + body）。body 一般接 VGND / 局部 substrate node。

**xschem 还提供 `res_generic_m1..m5`、`res_generic_l1`、`res_generic_po`**：金属/poly 的"理想方块电阻"符号。这是估算寄生 IR drop 时用的辅助符号，**没有对应的 SPICE 模型**（spice/ 下找不到），foundry 不背书；只能给手算/原理图标注用。

## 电容

三大族，性格完全不同：

### 1. MIM —— 主力线性电容

```text
cap_mim_m3_1     M3 顶板 MIM（小电容板）
cap_mim_m3_2     M3 顶板 MIM（带 cap2，更大密度变种）
```

- **结构**：M3 之上的薄介质 MIM 板。
- **密度**：约 2 fF/µm²。
- **参数**：`w`, `l` 自由；`mf`（multiplier）做并联。
- **何时用**：模拟里 99% 的"线性、有限值"电容（采样、滤波、补偿网络）都用 MIM。
- **端口**：`c0 c1`（两端，没有 body —— 因为下板是 M3，上板浮空）。

### 2. MOS varactor —— 可变电容

```text
cap_var_lvt      LVT 阈值的 MOS 变容
cap_var_hvt      HVT 阈值的 MOS 变容
```

- 用 nFET 接成 G vs (S=D=B) 形成 C-V 曲线。
- **何时用**：VCO 的调谐管、PLL loop filter 微调。
- **参数**：`w l b vm`（vm = 并联倍率）。
- **端口**：`c0 c1 b`。

### 3. VPP —— 多层指/华夫格寄生电容

```text
cap_vpp_<sizeW>x<sizeL>_<layers>_<shielding>[_<extra>]
例：cap_vpp_11p5x11p7_l1m1m2m3m4_shieldpom5
   尺寸 11.5×11.7 µm，用 L1 + M1–M4 交错指，PO 与 M5 做屏蔽
```

- **结构**：金属/li 各层交错指 + 顶/底屏蔽，~ 60 种尺寸/层数/屏蔽组合。
- **密度**：约 1 fF/µm²（比 MIM 低）。
- **何时用**：你**不需要顶层 MIM**、或者面积比线性度更重要时（典型：开关电容滤波器内、低 Q 的 RC 网络）。**不要随便选一个尺寸用** —— 名字里编码了实际结构，换一个等于换器件。
- **端口**：`c0 c1 b`。

读 VPP 名字三段法：

| 段 | 含义 |
| --- | --- |
| `11p5x11p7` | 单元尺寸 11.5 µm × 11.7 µm |
| `l1m1m2m3m4` | 使用 li + M1–M4 作为指 |
| `shieldpom5` | po 和 M5 做上下屏蔽（隔离衬底 / 上层布线噪声）|
| `_noshield` | 没做屏蔽 |
| `_fingercap` / `_wafflecap` | 指状 / 华夫格几何变种 |

## 二极管

```text
diode_pd2nw_05v5         P+ → N-well，反向 5.5V
diode_pd2nw_05v5_lvt     LVT 版（更低 Vf）
diode_pd2nw_05v5_hvt     HVT 版
diode_pw2nd_05v5         P-well → N+，反向 5.5V
diode_pw2nd_05v5_lvt
diode_pw2nd_05v5_nvt
diode_pd2nw_11v0         11V 高压
diode_pw2nd_11v0
diode_*_no_rs            模型不含串联电阻（看纯结特性时用）
nwdiode_top              N-well 顶部二极管（衬底/well 隔离）
```

**注意**：MOS 模型里其实**自带** drain/source ↔ body 的结二极管（cj/cjsw/mj/mjsw 在 corner 文件里），不需要再外接 `diode_*`。`diode_*` 是当你**单独需要二极管功能**（温度传感器、bandgap、ESD 路径）时才显式用。

**端口**：`a c`（anode/cathode）。这些是 `.model d` 卡片，用 `d_xxx anode cathode <modelname> area=...`，不是 subckt。

## BJT

```text
npn_05v5_W1p00L1p00 ... npn_05v5_W5p00L5p00   # 6 个固定尺寸
npn_11v0_W1p00L1p00                            # 11V 一个固定尺寸
pnp_05v5_W0p68L0p68, pnp_05v5_W3p40L3p40       # PNP 两个固定尺寸
```

**全部 fixed-geometry**——尺寸编进名字，不能扫 W/L。

**端口**：`c b e s`（s = substrate）。

模拟里 sky130 BJT 用得不算多，主要场景：bandgap 参考。**PNP 只有两个尺寸**，bandgap 设计里通常用 `pnp_05v5_W3p40L3p40`（大）作大单元、`pnp_05v5_W0p68L0p68`（小）作 8 个并联的小单元。

## 电感

```text
ind_03_90         3 圈，外径 ~90 µm
ind_05_125        5 圈，外径 125 µm
ind_05_220        5 圈，外径 220 µm
```

**子电路化** —— 内部由 R/L/C lumped 元件搭出 π 模型（如 `ind_05_220.model.spice` 里的 R31/C2/L1...），近似真实螺旋的频率响应。

**端口**：`a b ct sub`（两端 + center tap + 衬底）。

只 3 个尺寸，且都不大，sky130 上做 RF 电感主要还是自己画+EM 仿真。

## ESD / RF 大类

- **`esd_*`**：与对应普通器件结构相同，但参数 binning 偏向 ESD 工况（更大 W、更宽 D-S 间距），mismatch 模型可能略不同。打 IO/ESD 时选这一族。
- **`rf_*`**：测量驱动的"测出来什么样就是什么样"模型。`rf_nfet_01v8_aM04W5p00L0p15` 这种名字 = 4 指、W=5 µm、L=0.15 µm 的固定实例。每一个都是独立 `.subckt`，配套寄生 + S 参数验证过。基带模拟时**不用碰**；RF 频段（> 1 GHz）才考虑。

## sky130_fd_pr 不包含的视图

复习 [[02-pdk-structure]] 里的 9 视图，对 fd_pr 来说**只有 `spice / gds / lef / mag / maglef`**，**没有 `lib / cdl / verilog / techlef`**。原因：

- 这些是模拟基本器件，没有时序模型（lib）。
- LVS 直接用 SPICE 网表比对，不需要单独的 cdl。
- 没有 RTL/门级行为，不需要 verilog。
- techlef 是工艺级 metal 栈定义，被独立放在标准单元库目录下（如 `sky130_fd_sc_hd/techlef/`）。

## 几个容易踩的坑

- **可变 W 电阻 vs 固定 W 电阻**：`res_high_po`（自由 W）只适合粗算；模拟设计用 `res_high_po_0p69` 之类固定 W 版本，foundry 的 mismatch / 温漂数据更准。
- **MOS 永远显式画 body**：sky130 没有"假设 nfet 衬底接 VGND"的约定，body 漏接会被 LVS 抓住，仿真也可能出现意外的 Vbs。
- **nfet vs nfet3 符号别混**：`nfet_01v8.sym` 是 4 端，`nfet3_01v8.sym` 是 3 端（body 内部默认接 VGND/VPWR）。同一张图统一用一种。
- **MIM 不要乱套尺寸**：`cap_mim_m3_1` 是基础单元；超大电容要用 `mf` 并联或者 `cap_mim_m3_2`，不要靠拉大单个 w/l 把面积撑到边界外（会撞 DRC 上限）。
- **VPP 选型靠表，不靠"差不多"**：名字编码结构本身，换一个尺寸/层数/屏蔽组合等于换器件，电容值、Q、寄生差很多。一般工艺手册里给推荐组合，照抄就行。
- **rf_\* 不能扫尺寸**：把 `rf_nfet_01v8_aM04W5p00L0p15` 拖到 xschem 里改 W 是没意义的（它不是参数化的）—— W 编码在名字里，要换 W 就换器件名。
- **diode 别叠在 MOS 上重复建模**：MOS 模型自带结二极管，外接 `diode_*` 会把同一个结算两次。

## 下一步

笔记 04：**仿真模型与 corner 体系** —— 把 `libs.tech/ngspice/sky130.lib.spice` 串出来的 corner 网络（`tt / ss / ff / sf / fs / leak / wafer` 主 corner 以及 `mismatch / subvt_mismatch / discrete` 等修饰）讲清楚，包括 Monte Carlo `mc_mm_switch` / `mc_pr_switch` 怎么开、binned model 怎么覆盖整个 L/W 平面。
