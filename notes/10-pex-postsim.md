# 10 — PEX 提取与后仿

## 目标

[[09-tutorial-layout-lvs]] 把版图 LVS 跑过了，本章把版图里**真实存在但 schematic 里没有**的寄生 R/C 提出来灌进 SPICE，再跑一次 AC 看性能下降多少。

整本笔记到这里完成"模拟设计全流程"知识闭环：

```
原理图 (xschem)
    │
    └──netlist──> schematic.spice ─── 仿真 (08) ── 性能
                                          │
                                          ↓
                                       LVS (09)
                                          │
                                          ↓
版图 (magic)  ──extract+pex (本章)──> pex.spice ─ 后仿 (本章) ── 真实性能
```

所有命令本机实测过（magic 8.3、ngspice-36、sky130A）。

## 一、PEX 是什么、提取什么

**Parasitic Extraction** = 从版图几何尺寸算出实际硅上的寄生 R/C 网络，作为补充元件接到原 schematic 网表上。

| 寄生类型 | 来源 | 量级（sky130 典型） |
| --- | --- | --- |
| **C_coupling** | 同层 / 跨层金属间互容、metal-substrate | fF–pF 量级 |
| **C_overlap / C_fringe** | 器件接触 / via 附近 | 0.1–1 fF |
| **R_metal** | metal 线段的 sheet 电阻 × 长度/宽度 | mΩ–Ω 量级 |
| **R_via** | via 的接触电阻 | 几 Ω 一个 via |
| **R_diff / R_poly** | 扩散 / 多晶电阻（**也是器件**，不算"寄生"）| 100 Ω-kΩ |

模拟设计通常**先看电容**——R 在低频信号路径上影响小；做时钟树 / 高速开关电路才必须做 R+C 全提。

## 二、magic 的 PEX 流程（cap-only，最常用）

参考 [[09-tutorial-layout-lvs]]，已经在 `~/work/ota_5t/` 准备好 `local_inv_1.mag`（PDK inv_1 的本地副本）。

```bash
cd ~/work/ota_5t

PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole \
        -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load local_inv_1
extract all                     ;# 跑普通 extract，建 .ext
ext2spice lvs                   ;# LVS 友好预设
ext2spice cthresh 0.001         ;# 提取阈值：低于 0.001 fF 的 cap 不写出
ext2spice                       ;# 实际生成 .spice
quit -noprompt
EOF
```

`ext2spice cthresh <pF>` 是关键开关。默认 cthresh=0 是 "infinite"（不提取任何 cap）。设成一个小数（如 0.001 = 1 aF）就开始提 cap。再大一点（如 0.01 = 10 aF）能滤掉太小的"噪音"。

**实测产出 `local_inv_1.spice`**（与 [[09]] 里的纯 LVS 网表对比）：

```spice
* 纯 LVS 网表（7 行）
.subckt sky130_fd_sc_hd__inv_1 A VGND VNB VPB VPWR Y
X0 Y A VGND VNB sky130_fd_pr__nfet_01v8     ad=0.169 pd=1.82 as=0.169 ps=1.82 w=0.65 l=0.15
X1 Y A VPWR VPB sky130_fd_pr__pfet_01v8_hvt ad=0.26  pd=2.52 as=0.26  ps=2.52 w=1    l=0.15
.ends
```

```spice
* PEX 网表（22 行：器件 + 15 个 cap）
.subckt local_inv_1 A VGND VNB VPB VPWR Y
X0 Y A VGND VNB sky130_fd_pr__nfet_01v8     ad=0.169 pd=1.82 as=0.169 ps=1.82 w=0.65 l=0.15
X1 Y A VPWR VPB sky130_fd_pr__pfet_01v8_hvt ad=0.26  pd=2.52 as=0.26  ps=2.52 w=1    l=0.15
C0  Y    VPWR  0.12759f      ; 输出到 VDD 屏蔽 cap
C1  VGND Y     0.09986f      ; 输出到 VSS 屏蔽 cap
C2  VPWR A     0.04571f
C3  VGND A     0.04768f
C4  VPB  VPWR  0.06649f
C5  VGND VPB   0.01319f
C6  Y    A     0.04760f      ; ★ Miller cap（输入/输出耦合）
C7  VPB  Y     0.01774f
C8  VPB  A     0.04506f
C9  VGND VPWR  0.03401f
C10 VGND VNB   0.24421f      ; ★ 衬底耦合（最大）
C11 Y    VNB   0.09610f
C12 VPWR VNB   0.20582f      ; ★ 衬底耦合
C13 A    VNB   0.13301f
C14 VPB  VNB   0.33898f      ; ★ 最大：PMOS body 到 NMOS body
.ends
```

15 个寄生电容覆盖了所有节点对的耦合，量级 10 aF – 340 aF。最大的几个都是 **VNB（衬底）相关**——这是为什么"地"在版图里要做好 guard ring 的物理证据。

## 三、加上 R 的 PEX（advanced）

要把金属线电阻也提取出来：

```tcl
extract all
extresist all                  ;# 单独跑 resistance extraction
ext2spice lvs
ext2spice extresist on         ;# 告诉 ext2spice 也写 R
ext2spice cthresh 0.001
ext2spice rthresh 0.01         ;# R 阈值（Ω）：太小的 R 不写出
ext2spice
```

`extresist` 用 CIF-based tile 算法，**慢**（万管子级几分钟到几小时），结果是把每条金属线段切成多个串联 R + 节点。版图小、信号路径短的场景可以略过 R，只做 cap-only。

实测 `extresist` 在我们这个小 inv_1 上**给了 warning "Cannot open file (UNNAMED).ext"** 但 cap 部分仍然写出来了。完整 R+C PEX 在工业流程里通常用 OpenROAD 的 `pyspef-extractor` 或商业 Calibre `xRC`，magic 内置 extresist 主要适合小 cell 单点。

## 四、ext2spice 关键开关速查

| 开关 | 默认 | 作用 |
| --- | --- | --- |
| `ext2spice lvs` | off | 一键设 LVS-friendly：subcircuit top off / global / hierarchy / short resistor / scale off |
| `ext2spice scale off` | already off in sky130 magicrc | 不再乘 `.option scale`，避免双倍叠加 |
| `ext2spice cthresh <pF>` | 0 (= ∞) | cap 阈值；**写 PEX 必须设非 0** |
| `ext2spice rthresh <Ω>` | 0 (= ∞) | R 阈值 |
| `ext2spice extresist on` | off | 配合 `extresist all` 才写 R 网络 |
| `ext2spice short resistor` | on | 把短路 R 当导线 |
| `ext2spice global` | empty | 全局节点列表，sky130 一般 `VPWR VGND VSUBS` |
| `ext2spice hierarchy on` | on | 保留层次（更小输出） |

## 五、后仿对比（inv_1 demo）

写两个 testbench，**完全一样的 stimulus**，只差 `.include` 文件不同：

```spice
* tb_inv_pre.sp  — pre-layout
.option scale=1u
.lib "$PDK_ROOT/sky130A/libs.tech/ngspice/sky130.lib.spice" tt
.include "inv_schematic.spice"          ;# 纯 LVS 网表
...
xinv A VGND VNB VPB VPWR Y sky130_fd_sc_hd__inv_1
cl Y 0 5f

* tb_inv_pex.sp  — post-layout
.option scale=1u
.lib "$PDK_ROOT/sky130A/libs.tech/ngspice/sky130.lib.spice" tt
.include "local_inv_1.spice"            ;# 带 15 个 cap 的 PEX 网表
...
xinv A VGND VNB VPB VPWR Y local_inv_1
cl Y 0 5f
```

完整 testbench（含 DC bias 在转换点 0.78V，AC sweep）：

```spice
vvdd VPWR 0 1.8
vvss VGND 0 0
vvpb VPB  0 1.8
vvnb VNB  0 0
vin  A    0 dc 0.78 ac 0.5
xinv A VGND VNB VPB VPWR Y <cellname>
cl Y 0 5f

.control
op
ac dec 20 1 100g
let mag_db = db(mag(v(Y)))
meas ac av_dc find mag_db at=1
meas ac f3db when mag_db='av_dc-3' fall=1
print av_dc f3db
.endc
```

**实测结果**：

| 量 | pre-layout | post-layout (PEX) | 变化 |
| --- | --- | --- | --- |
| DC 增益 av_dc | **19.14 dB** | **19.14 dB** | 不变 ✅ |
| -3 dB 带宽 f3db | **2.39 GHz** | **2.07 GHz** | **-13.5%** |
| 静态偏置 V_Y | 0.984 V | 0.984 V | 不变 |

解读：
- **DC 增益不变** —— cap 在直流下是开路，PEX 加的只是 cap，不影响 DC 工作点。
- **带宽降 13.5%** —— PEX cap 加到输出节点，让原始 5 fF 负载实际变成 ~5.4 fF，主极点频率相应下降 ≈ 5/5.4 = 7%，加上其他节点 RC 引起的次极点提前，总带宽降 13.5%，**与寄生 cap 量级吻合**。
- 13.5% 是这个简单 cell 的水平。**对密集走线 + 长 trace 的模拟版图，带宽损失常到 30–50%**。所以前仿要预留充足设计余量。

## 六、把 PEX 用到 5T-OTA 后仿

把 [[09-tutorial-layout-lvs]] 里的版图流程换成 OTA：

```bash
# 1. 在 magic GUI 里画好 ota_5t.mag
# 2. PEX 提取
PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load ota_5t
extract all
ext2spice lvs
ext2spice cthresh 0.001
ext2spice
quit -noprompt
EOF
# 3. 拿 ota_5t.spice 替换 ota.cir，回到笔记 08 的流程跑 AC / corner / MC
```

预期看到：
- DC 增益基本不变（前面那条规律）；
- GBW **下降 10–30%**（输出节点 cap 增加）；
- 次极点频率下降 → PM 可能掉 5–20°，**需要补偿调整**。

这是为什么模拟设计的"后仿迭代"周期常常占整个设计周期的 30–50%——画完版图发现性能掉了，回去调尺寸或加 cap 重做。

## 七、几个常见坑

- **`cthresh 0` 拿不到 cap**：默认值是 0 但代表"无穷"——即没有 cap 小到值得提。要提就必须设一个非 0 的小值，建议 `0.001` (= 1 aF)。
- **PEX 让 LVS 失败**：PEX 网表里的 cap 节点是 schematic 没有的。**LVS 永远用 `ext2spice lvs`（不带 cthresh）的版本**，单独跑 PEX 用 `cthresh` 版。即维护两份 .spice：一份 LVS、一份 PEX。
- **Miller cap 把电路稳定性翻天**：cap 在 input/output 间形成正反馈路径（Miller 倍增），如果原电路本就 PM 紧（< 60°），PEX 后可能直接振荡。Always re-check PM。
- **scale 又双倍**：`.option scale=1u` 是给器件 W/L 的，PEX 出来的 cap 单位**已经是 fF**（带 `f` 后缀），不会被 scale。但有时候 ext2spice 输出的 cap 不带单位，会被 scale 折算——确认 cap 末尾有 `f` 后缀。
- **VNB / VPB 端口在 PEX 后悬空**：上面的 inv_1 例子里 PEX cap 把 VNB / VPB 连到很多节点。如果 testbench 不接这俩端口（让它们浮空），AC sim 会拿到错的结果。永远显式接 VNB=VGND, VPB=VPWR。
- **`extresist all` 在小 cell 报 "Cannot open .ext"**：bug 触发条件复杂，cap-only 不受影响。要做 R+C 完整 PEX，转用 OpenROAD `pyspef-extractor` 或 SPEF-based 流程。
- **PEX 文件巨大**：万管子级版图 PEX 出来的 SPICE 上 GB；ngspice 解析慢。生产环境用 SPEF (Standard Parasitic Exchange Format) + 二进制后仿工具更高效。

## 完结：从安装到后仿的全链回顾

至此 10 篇笔记串完，原 README 计划的"基于 PDK 的模拟电路设计全流程"：

| # | 内容 | 类型 |
| --- | --- | --- |
| 01 | sky130 PDK 安装（Volare）| 知识 |
| 02 | PDK 文件结构地图（libs.tech / libs.ref）| 知识 |
| 03 | 器件库 sky130_fd_pr | 知识 |
| 04 | 仿真模型与 corner 体系 | 知识 |
| 05 | magicrc / xschemrc 入口 | 知识 |
| 06 | DRC / LVS 流程 | 知识 |
| 07 | 标准单元库 sky130_fd_sc_hd | 知识 |
| 08 | **5T-OTA 仿真**（AC / corner / Monte Carlo）| 实战 |
| 09 | **版图 + DRC / LVS** | 实战 |
| 10 | **PEX + 后仿** | 实战 |
| 参考 | 外部资源 + FinFET PDK 现状 | 参考 |

下一步不应该是再写一篇笔记——应该**真的把 5T-OTA 全程跑一遍**：

1. xschem 画 OTA 原理图（手工，1–2 天）；
2. ngspice 跑 [[08]] 的 corner/MC（30 分钟）；
3. magic 画 OTA 版图（手工，3–5 天，是工作量大头）；
4. [[09]] DRC / LVS 直到 "Circuits match uniquely"；
5. 本章 PEX → 后仿 → 评估性能损失；
6. 如果性能不达标，迭代尺寸 / 版图，回 2。

然后选一个更难的电路（OTA + 输出 buffer / bandgap / SAR ADC 子电路），重复一次完整流程。

模拟 IC 是 "muscle memory"——任何一篇笔记都替代不了亲手跑过的体感。
