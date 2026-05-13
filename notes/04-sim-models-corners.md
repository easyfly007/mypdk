# 04 — 仿真模型与 corner 体系

## 目标

把 sky130 给 ngspice 用的整套模型 + corner 机制串起来：

- testbench 里只写一行 `.lib …/sky130.lib.spice tt`，里面到底链了什么？
- 主 corner（tt/ss/ff/sf/fs/leak/wafer）+ R+C corner（ll/hh/hl/lh）+ mismatch (`_mm`) 各是什么？
- BSIM binned model 是怎么实现的，63 个 bin 怎么覆盖 W/L 平面？
- 两个开关 `mc_mm_switch` / `mc_pr_switch` 各管什么，做 Monte Carlo 时怎么用？

下文路径都基于 `~/.volare/sky130A/libs.tech/ngspice/`（按 [[02-pdk-structure]] 的分层）。

## 入口：sky130.lib.spice

testbench 里通常只见到：

```spice
.lib "$PDK_ROOT/sky130A/libs.tech/ngspice/sky130.lib.spice" tt
```

这个文件就是个**派发表**——里面定义了一长串 `.lib <name> ... .endl <name>` 段，按需要的 corner 名拉一段进来。结构高度规则：

```spice
.lib tt
.param mc_mm_switch=0
.param mc_pr_switch=0
.include "corners/tt.spice"                       ← MOSFET 模型（按工艺 corner）
.include "r+c/res_typical__cap_typical.spice"     ← 无源 corner
.include "r+c/res_typical__cap_typical__lin.spice" ← 无源 corner 的线性化
.include "corners/tt/specialized_cells.spice"     ← SRAM/latch 特殊 binning
.endl tt
```

每个 corner 都是这四件套（MOSFET + 无源 + 特殊 cell + 两个 MC 开关默认 0）。

## 三个独立的"corner 维度"

sky130 把变化拆成三层**正交**维度，每层都可以单独选：

| 维度 | 控制对象 | 取值 | 命名出现位置 |
| --- | --- | --- | --- |
| **MOSFET 过程 corner** | nFET / pFET 的速度（vth / mobility / tox 联动）| tt / ss / ff / sf / fs / leak / wafer | corner 名的"基"部分 |
| **R + C 过程 corner** | 电阻方块值、电容密度 | typical / low / high 各取一种 | 复合 corner 名的第二段 |
| **失配 / 过程 MC 开关** | 是否启用 Monte Carlo 随机数 | mc_mm_switch / mc_pr_switch ∈ {0,1} | `_mm` 后缀 / `mc` 专用 lib |

实际暴露给你的 `.lib` 名是这三个维度的笛卡儿积里被打包出来的常用组合。

### 维度 1 — MOSFET 过程 corner

| Lib | nFET | pFET | 用途 |
| --- | --- | --- | --- |
| `tt` | typical | typical | 中心点 |
| `ss` | slow | slow | 速度最差 / 功耗最低 |
| `ff` | fast | fast | 速度最好 / 漏电最高 |
| `sf` | slow | fast | 反相器拉低慢、拉高快 → 跳变点偏低 |
| `fs` | fast | slow | 反相器拉低快、拉高慢 → 跳变点偏高 |
| `leak` | 高泄漏边角 | 高泄漏边角 | 漏电分析专用 |
| `wafer` | 全 wafer 同向漂 | 全 wafer 同向漂 | 系统 calibration 用，**不用作签核** |

R+C 在以上 7 个 lib 里固定走 typical 档（`r+c/res_typical__cap_typical.spice`）。

### 维度 2 — R + C 过程 corner

`libs.tech/ngspice/r+c/` 下文件：

```text
res_typical__cap_typical.spice         R typ / C typ
res_low__cap_low.spice                 R low  / C low
res_high__cap_high.spice               R high / C high
res_high__cap_low.spice                R high / C low   ← 极端反相
res_low__cap_high.spice                R low  / C high  ← 极端反相
*__lin.spice                           对应的线性化版本（一般和主文件同时 include）
```

为了方便，sky130 把"MOS typical × R+C 4 个非 typ 组合"也打包成了 4 个独立 lib：

| Lib | MOS | R | C |
| --- | --- | --- | --- |
| `ll` | tt | low | low |
| `hh` | tt | high | high |
| `hl` | tt | high | low |
| `lh` | tt | low | high |

模拟设计常跑 `tt / ss / ff / sf / fs / ll / hh` 这 7 个；RC 反相极端 `hl / lh` 用于时序 RC 树的极限边角。

### 维度 3 — 失配 / 过程 MC

两个开关：

```spice
.param mc_mm_switch=0   * 默认关，置 1 → 启用器件失配（同 die 内随机）
.param mc_pr_switch=0   * 默认关，置 1 → 启用过程随机（die-to-die / wafer）
```

它们的作用方式：器件模型文件里凡是失配 / 过程相关的项都写成

```spice
... + MC_MM_SWITCH * AGAUSS(0, σ, 1) ...
... + MC_PR_SWITCH * AGAUSS(0, σ, 1) ...
```

`AGAUSS(0, σ, 1)` 是 ngspice 在仿真启动时（或 monte carlo 每次迭代）抽一个 N(0, σ) 的随机数。开关为 0 → 整个项归零 → 纯 typical。开关为 1 → 引入随机扰动。

**预打包的 mismatch lib**（共 9 个）：

```text
tt_mm / ss_mm / ff_mm / sf_mm / fs_mm    ← 5 个工艺 corner × mm
ll_mm / hh_mm / hl_mm / lh_mm            ← 4 个 R+C corner × mm
```

每个 `_mm` 库只设 `mc_mm_switch=1, mc_pr_switch=0` —— **只开器件失配，不开过程随机**。

**过程 MC 专用库 `mc`**：

```spice
.lib mc
.param mc_mm_switch=0
.param mc_pr_switch=1
.include "parameters/critical.spice"   ← 一堆 .param X = MC_PR_SWITCH * AGAUSS(0,σ,1)
.include "parameters/montecarlo.spice" ← 推导参数（如 camimc = f(ic_cap)...）
.endl mc
```

注意 `.lib mc` **不包含 MOS 模型本身**——它只把"过程随机的种子参数"和"对种子的依赖公式"灌进来。要做 process MC，需要**同时**拉一个普通 corner（`tt` 一般作中心点）：

```spice
* 工艺 + 失配 + 过程 MC，三个都开
.lib "sky130.lib.spice" tt
.param mc_mm_switch=1
.param mc_pr_switch=1
* 注意：用 .lib mc 也只是把 critical / montecarlo 这两个文件拉进来；
* 也可以直接显式 .include 它们，效果一样。
.lib "sky130.lib.spice" mc
```

## MOSFET binned model 的实现

打开 `corners/tt.spice`：里面是一长串 `.include`，把 1.8V/3.3V/5V/16V/20V × n/p × Vt 变种 × ESD 变种 …… 各 corner 模型卡全拉进来：

```spice
.include "../../../libs.ref/sky130_fd_pr/spice/sky130_fd_pr__nfet_01v8__tt.pm3.spice"
.include "../../../libs.ref/sky130_fd_pr/spice/sky130_fd_pr__nfet_01v8__mismatch.corner.spice"
.include "../../../libs.ref/sky130_fd_pr/spice/sky130_fd_pr__nfet_01v8_lvt__tt.corner.spice"
...
.include "tt/nonfet.spice"      ← 这个 corner 下的电阻/电容/二极管模型
.include "../all.spice"         ← 所有 corner 都用的"非 corner 相关"模型
.include "tt/rf.spice"          ← RF 固定器件
```

以 `nfet_01v8` 为例，包含两类文件：

```text
sky130_fd_pr__nfet_01v8__tt.corner.spice    1474 行：63 个 bin 的 `_diff_NN` 参数（每个参数的"该 bin 相对 typical 的偏移"）
                                            末尾 .include "sky130_fd_pr__nfet_01v8__tt.pm3.spice"
sky130_fd_pr__nfet_01v8__tt.pm3.spice       数千行：63 张 `.model sky130_fd_pr__nfet_01v8__model.<N> nmos ...` 卡片
                                            每张卡描述一个 L×W 区间的 BSIM4 参数（参数值 = typical 值 + `_diff_NN`）
```

ngspice 按器件实例的 W/L 自动选 bin（BSIM4 的 lmin/lmax/wmin/wmax 选择规则）。

外面给你看到的"统一器件" `sky130_fd_pr__nfet_01v8 d g s b` 是 `sky130_fd_pr__nfet_01v8.pm3.spice` 里的 wrapper subckt，内部 `msky130_fd_pr__nfet_01v8 ... sky130_fd_pr__nfet_01v8__model ...` —— 它指的 model 名（不带 `.NN` 后缀）会被 binning 机制自动展开。

**为什么这样设计**：把"corner 偏移 (`_diff_NN`)"和"基础 BSIM 卡片 (`.model …__model.NN`)"分开，更换 corner 只需要替换 `_diff_NN` 那批 `.param`，bin 模型卡片本体不动。可读性 + 维护性都更好。

### nonfet.spice / all.spice / specialized_cells.spice

跟 corner 走、但不是 MOSFET 的东西：

```text
corners/tt/nonfet.spice         电阻、电容、二极管 在这个 corner 下的参数（res_high_po 的 rsheet、cap_mim 的 camimc 等）
corners/tt/specialized_cells.spice   SRAM bitcell / latch 的 special_* 器件 binning
all.spice                       不随 corner 变的东西：二极管 .model 卡、BJT、电感、变容、VPP 电容 wrapper
```

注意 **VPP / MIM 电容**的"密度参数"（`camimc`、`cm1d`、…）放在 `parameters/montecarlo.spice` 里，会被 `mc_pr_switch` 调制。这意味着即使你只想跑 `.lib tt`，**电容值也不带过程变化**——要把电容当 corner 看，得自己再 include 一个 `r+c/` 对应的文件，或者用 `.lib hh / ll` 之类预打包。

## 常用 corner 矩阵（实战速查）

| 你要分析的 | 选哪个 lib | 备注 |
| --- | --- | --- |
| 中心点功能验证 | `tt` | 永远先跑这一个 |
| 速度上下边界（数字） | `ss`, `ff` | |
| 反相器跳变点边界 | `sf`, `fs` | |
| 漏电 / 静态功耗 | `leak`，温度也要拉到 -40 / 125 | |
| RC 网络（时钟树）延迟边界 | `ll`, `hh` 主要；`hl`, `lh` 极端检查 | |
| 偏置电流 / 镜像精度 | `tt_mm` + Monte Carlo（100~500 次）| `mc_mm_switch=1`，关 process |
| 全片散布 / Cpk | `tt` + `.lib mc` + 多次 MC | 两个开关都开 |
| 工艺 + 失配联合最差 | `ss_mm`, `ff_mm`, `sf_mm`, `fs_mm` 各 100 次 MC | "corner of corner" |

## 一个最小可跑的 testbench 模板

```spice
* OPA inverter Vout vs Vin
.option scale=1u    * 与 PDK 一致，l/w 默认按 µm 解析

* 选 corner（这里跑 typical-mm + process MC）
.lib "$PDK_ROOT/sky130A/libs.tech/ngspice/sky130.lib.spice" tt
.param mc_mm_switch=1
.param mc_pr_switch=1
.include "$PDK_ROOT/sky130A/libs.tech/ngspice/parameters/critical.spice"
.include "$PDK_ROOT/sky130A/libs.tech/ngspice/parameters/montecarlo.spice"

vdd vdd 0 1.8
vin vin 0 dc 0.9 ac 1
xm1 vout vin 0 0 sky130_fd_pr__nfet_01v8 w=1 l=0.15 nf=1
xm2 vout vin vdd vdd sky130_fd_pr__pfet_01v8 w=2 l=0.15 nf=1
cl  vout 0 10f

.control
  let runs = 100
  let i = 0
  dowhile i < runs
    reset
    op
    print v(vout) > "out.$&i.txt"
    let i = i + 1
  end
.endc

.end
```

要点：

- **`.option scale=1u`** 必须和 PDK 内 `all.spice` 设的一致；不写就要把所有 `w=1` 改成 `w=1u`。
- 引用器件**永远用 `sky130_fd_pr__<name>` 的 subckt 形式 + `x` 前缀**，不要用 `m` 直接调 BSIM model。
- ngspice 跑 MC 是手写 `.control` 循环（`dowhile`），它没有内置 `.mc` 命令；用商业 SPICE 时才会有 `.mc 100`。
- `print > "file.txt"` 把结果存盘后用 Python / awk 后处理（直方图、Cpk）。

## 几个容易踩的坑

- **scale 不一致 → 器件大 1000 倍**：`.option scale=1u` 必须显式写。忘了就会出现 W=1m 的怪现象，仿真"为什么不工作"。
- **mc_mm 和 mc_pr 弄反**：`mm` = mismatch（同 die 内）、`pr` = process（die 间）。仿镜像电流精度时只开 `mm`，仿散布率时两个都开 + 多次 MC。
- **`.lib mc` 单独用没意义**：它只把 MC 种子参数引入，**不带任何器件模型**。必须再 `.lib tt`（或任一 corner）才有 MOS / R / C 可用。
- **bin 边界 ≠ 物理边界**：63 个 bin 是模型拟合的方便，**不是器件可用范围的硬限制**。但是落在 bin 边界附近的 W/L 容易出现"两个 bin 不连续"的现象，敏感设计（offset、增益）尽量用 bin 中心附近的尺寸。
- **温度 corner 单独管**：corner lib 不含温度，温度是用 `.options TNOM=27` + `.temp -40 27 125` 单独设的。"ss 125°C"是 corner + 温度的组合。
- **R+C corner 不是免费搭配**：`hh / ll / hl / lh` 都把 MOS 固定为 tt。要"`ss` + R high + C high"就只能手动 `.include "corners/ss.spice"` + `.include "r+c/res_high__cap_high.spice"` 自己拼。
- **wafer corner ≠ ss/ff**：`wafer.spice` 描述"整片 wafer 同向漂移"用于硬件校准，**不替代** `ss/ff` 做签核。

## 下一步

笔记 05：**magicrc / xschemrc 入口配置**——把 magic 启动、xschem 启动各自要在工作目录或 shell 里准备什么讲清楚，把 `PDK_ROOT` / `PDK` / `MAGTYPE` 几个环境变量定下来。
