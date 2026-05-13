# 08 — 实战 1：5T-OTA 原理图与 corner / Monte Carlo 仿真

## 目标

把前 7 篇的"知识层"串成一个动手实验：

- 选定一个最小拓扑 —— **5-T OTA**（单端、PMOS 镜像负载 + NMOS 差分对 + NMOS 尾源）；
- 凭手算定一组初始尺寸；
- 用 ngspice 跑 DC op、AC bode、5 个 corner 的扫描、Monte Carlo 50 次；
- 读结果、找瓶颈、点出**下一步该改什么**。

**所有 SPICE 已实测可在本机跑通**（ngspice-36 + sky130A 已装好；见 [[01-install-sky130]]）。把下面三个文件拷到 `~/work/ota_5t/`，跑就能复现。

## 一、拓扑与手算

```text
        VDD
         │
       ┌─┴─┐    ┌─┴─┐
       │M3 │    │M4 │      ← PMOS current mirror load
       │___│    │___│
         │        │
       ┌─┴─┐    ┌─┴─┐
       │M1 │    │M2 │      ← NMOS input differential pair
       │___│    │___│
   VINP─┘   ┌────┘
            │    └────VOUT
            │
          ┌─┴─┐
          │M5 │           ← NMOS tail current source
          │___│
            │
          VSS
```

### 一阶手算（目标 40 dB 增益 / 10 MHz GBW / CL = 1 pF）

- GBW = gm₁ / (2π · CL) → gm₁ ≈ 2π × 10 MHz × 1 pF ≈ **63 µA/V**
- 工作点：取 ID₁ = ID₂ = 10 µA（尾电流 ITAIL = 20 µA）。
- gm/ID ≈ 6.3 V⁻¹ —— sky130 的 nfet_01v8 在中等反型区，这对应 Vov ≈ 0.2 V，需要 **W/L ≈ 4/0.5**。
- PMOS 镜像 W/L 偏小（W/L = 2/2）以提高输出阻抗 ro。
- 增益 A₀ = gm₁ · (ro₂ ‖ ro₄)，目标 100 V/V (40 dB)；ngspice 实测出来其实只有 23 dB —— 这就是手算 vs 仿真要双校验的原因。

参数化网表（`ota.cir`）让所有尺寸都成 `.param`：

```spice
* ota.cir
.subckt ota_5t vinp vinn vout vbias vdd vss
+ WIN=4 LIN=0.5 NFIN=2 WMR=2 LMR=2 NFMR=2 WTAIL=4 LTAIL=1 NFTAIL=2
xm1 vd1  vinp vs5 vss sky130_fd_pr__nfet_01v8  w={WIN}   l={LIN}   nf={NFIN}
xm2 vout vinn vs5 vss sky130_fd_pr__nfet_01v8  w={WIN}   l={LIN}   nf={NFIN}
xm3 vd1  vd1  vdd vdd sky130_fd_pr__pfet_01v8  w={WMR}   l={LMR}   nf={NFMR}
xm4 vout vd1  vdd vdd sky130_fd_pr__pfet_01v8  w={WMR}   l={LMR}   nf={NFMR}
xm5 vs5  vbias vss vss sky130_fd_pr__nfet_01v8 w={WTAIL} l={LTAIL} nf={NFTAIL}
.ends
```

呼应 [[03-fd-pr-devices]] 关于 MOS 端口与参数。

## 二、单 corner AC 仿真

测试台 `tb_ac.sp`：

```spice
* tb_ac.sp — 5T-OTA AC bode (tt corner)
.option scale=1u
.lib "$PDK_ROOT/sky130A/libs.tech/ngspice/sky130.lib.spice" tt
.include "ota.cir"

vdd vdd 0 1.8
vss vss 0 0

* Bias mirror reference (生成 vbias，让 M5 镜像 20 µA 尾电流)
xm6 vbias vbias vss vss sky130_fd_pr__nfet_01v8 w=4 l=1 nf=2
iref vdd vbias 20u

* Differential AC stimulus, CM = 0.9V
vcm vcm 0 0.9
vd  vd  0 dc 0 ac 0.5
ediff vinp vcm vd 0  0.5
einv  vinn vcm vd 0 -0.5

xota vinp vinn vout vbias vdd vss ota_5t
cl  vout 0 1p

.control
op
ac dec 20 1 1g
let mag_db = db(mag(v(vout)))
let ph_deg = 180/pi * cph(v(vout))
meas ac dc_gain find mag_db at=1
meas ac gbw     when mag_db=0 fall=1
meas ac pm_freq find ph_deg when mag_db=0 fall=1
let pm = pm_freq + 180
print dc_gain gbw pm
wrdata bode_tt.txt mag_db ph_deg
.endc
.end
```

要点（呼应 [[04-sim-models-corners]] / [[05-rc-entrypoints]]）：

| 行 | 说明 |
| --- | --- |
| `.option scale=1u` | 必须，与 PDK 模型文件约定一致 |
| `.lib "..." tt` | tt corner，可换 ss/ff/sf/fs/ll/hh/... |
| `ediff/einv` 受控源 | 单端 vd 转差分；`vinp - vinn = vd` |
| `cph()` | ngspice 取复数相位，单位 rad；× 180/π 转 deg |
| `meas ac ... when ... fall=1` | 找 mag_db 由正到负穿越点（GBW）|
| `wrdata` | 写 bode 到文本，用 python/awk 后处理 |

**实测输出**：

```text
=== Op point ===
v(vout)       = 0.516 V        ← 不在 VDD/2，但 M4 仍在饱和 (Vds = 1.28V)
v(xota.vs5)   = 0.163 V        ← M5 漏极
v(vbias)      = 0.783 V        ← 镜像 gate
i(iref)       = -20 µA

=== AC ===
dc_gain  =  23.4 dB
gbw      =  9.80 MHz
pm       =  91.4°
```

23 dB 比目标 40 dB 低了 17 dB —— 问题在 PMOS 镜像的 ro 不够大。在 [[03-fd-pr-devices]] 提到过 sky130 的 PMOS 短沟道 ro 比想象的差。解决思路（不在本章实施，留作练习）：

1. 加大 LMR → 2u 已经设了，再加大边际收益递减。
2. 改用 cascoded mirror（多 2 个管子）→ 这才是常规模拟做法。
3. 改用 telescopic OTA。

## 三、5 corner 扫描

ngspice 没有内置的"换 corner 重跑"（`.lib` 是解析期生效，不能在 `.control` 里切）。最干净的方法是**外部 shell 循环 + sed 模板**：

```bash
#!/bin/bash
# run_corners.sh
set -e
printf "%-6s %12s %12s %10s\n" corner DC_gain GBW_Hz PM_deg
for c in tt ss ff sf fs; do
  sed "s/\" tt$/\" $c/" tb_ac.sp > _tb_$c.sp
  out=$(ngspice -b _tb_$c.sp 2>/dev/null | tail -8)
  gain=$(echo "$out" | awk '/^dc_gain/{print $3; exit}')
  gbw=$(echo  "$out" | awk '/^gbw/{print $3; exit}')
  pm=$(echo   "$out" | awk '/^pm /{print $3; exit}')
  printf "%-6s %12s %12s %10s\n" "$c" "$gain" "$gbw" "$pm"
  rm -f _tb_$c.sp
done
```

**实测结果**：

```text
corner   DC_gain     GBW          PM
tt       23.4 dB     9.80 MHz     91.4°
ss       20.0 dB     8.54 MHz     93.5°
ff       25.6 dB    10.74 MHz     90.3°
sf       -4.8 dB    (失败)        (失败)         ← ！
fs       31.3 dB     9.27 MHz     88.9°
```

**SF 崩了** —— DC 增益变负（-4.8 dB），meas 找不到 0 dB 穿越。这不是仿真 bug，是**电路本身的失效**：

- SF = 慢 nFET + 快 pFET。
- M5（尾源 NMOS）变慢 → 实际 ITAIL 远低于 20 µA。
- M3/M4（PMOS 镜像）变快 → 想拉 20 µA 进 vout 但 NMOS 输入对供不上 → 输出被推到 VDD 附近。
- 结果：M4 进了三极管区，gm 直接掉，OTA 不再是放大器。

教训：5T-OTA **对工艺 corner 对称性非常敏感**。常规修法是给 vbias 加一个对工艺鲁棒的 self-biased 启动电路（如 beta-multiplier），或者换成 telescopic / folded-cascode。

## 四、Monte Carlo（失配 50 次）

ngspice 没有商业 SPICE 的 `.mc` 命令，自己用 `.control` 写循环：

```spice
* tb_mc.sp
.option scale=1u
.lib "$PDK_ROOT/sky130A/libs.tech/ngspice/sky130.lib.spice" tt_mm
.include "ota.cir"

vdd vdd 0 1.8
vss vss 0 0
xm6 vbias vbias vss vss sky130_fd_pr__nfet_01v8 w=4 l=1 nf=2
iref vdd vbias 20u
vcm vcm 0 0.9
vd  vd  0 dc 0 ac 0.5
ediff vinp vcm vd 0 0.5
einv  vinn vcm vd 0 -0.5
xota vinp vinn vout vbias vdd vss ota_5t
cl  vout 0 1p

.control
let n = 50
let i = 0
let g_sum = 0
let g_sq  = 0
dowhile i < n
  reset
  ac dec 10 1 100meg
  let mag_db = db(mag(v(vout)))
  meas ac dc_gain find mag_db at=1
  let g_sum = g_sum + dc_gain
  let g_sq  = g_sq  + dc_gain^2
  printf "run %3d  %g\n" i dc_gain
  let i = i + 1
end
let mean = g_sum / n
let var  = g_sq/n - mean^2
let sd   = sqrt(var)
print mean sd
.endc
.end
```

关键：
- **`.lib ... tt_mm`** —— [[04-sim-models-corners]] 里讲的"_mm 系列"，自动设 `mc_mm_switch=1, mc_pr_switch=0`，启动失配，不开过程随机。
- **`reset`** —— 每次循环重新抽 random，否则只跑一次随机种子。
- **`g_sum / g_sq` 累加** —— ngspice 没有内置统计函数，自己算均值/方差。

**实测**（50 runs）：

```text
mean (DC gain) = 21.3 dB
σ              = 10.5 dB
```

**σ = 10 dB 对一个 21 dB 的目标 mean** —— 灾难性。原因：

1. 输入对 M1/M2 失配 → 大幅 input-referred offset。
2. offset × Av₀ 直接超出输出摆幅 → 部分样本里 OTA 工作在饱和外。
3. 5T 拓扑没有 offset cancellation 机制。

修法：
- chopper / auto-zero（系统级）；
- offset trim（trim DAC）；
- 加大 W·L（mismatch ∝ 1/√(WL)）—— 用面积换精度，是模拟版图永远的 trade-off；
- 用 folded-cascode + degeneration 改善。

## 五、本章产出的文件清单

```text
~/work/ota_5t/
├── ota.cir          ← 5T-OTA subckt（参数化）
├── tb_ac.sp         ← AC bode（单 corner）
├── tb_mc.sp         ← Monte Carlo（tt_mm，50 runs）
├── run_corners.sh   ← 5 corner sweep（外部 shell）
├── bode_tt.txt      ← AC 结果（wrdata 输出）
```

直接复制本章上面三段 SPICE + shell 即可。

## 六、几个本章遇到的坑（实测过）

- **`@xm1[id]` 不能用**：当 m1 被 wrap 在 subckt 里（用 `xm1 ...` 而不是 `m1 ...` 顶层实例），ngspice 的 `@elem[param]` 语法看不见内部。要么把器件放顶层、要么用 `print i(VX)` 类的支路电流。
- **`vp(...)` 在 ngspice 报 syntax**：商业 SPICE 里 `vp()` 是相位，ngspice 用 `cph()` 取复数相位。
- **`meas when ... fall=1` "out of interval"** 不是仿真错，是**测量量自身不存在**（如 SF corner 没穿过 0 dB）。脚本要 catch 这个，否则后续语句因 `gbw` 为空 vector 报错。
- **`.lib` 不能在 `.control` 里换**：见正文，外部 shell + sed 模板是当前 ngspice 的标准做法。
- **MC 循环不写 `reset` 只跑一次**：ngspice 的随机种子在 netlist 解析时抽一次，`reset` 重新解析才会重抽。

## 下一步

笔记 09：**实战 2：5T-OTA 版图与 DRC/LVS** —— 把本章这个电路在 magic 里画出来，跑通 [[06-drc-lvs-flow]] 里讲的 magic+netgen 与 KLayout LVS 两条线，目标 "Netlists match uniquely"。
