# 09 — 实战 2：版图与 DRC / LVS（在 magic 中跑通整条链）

## 目标

[[08-tutorial-ota-sim]] 拿到一个仿真过的电路，现在让它落到硅上。本章不画图（版图本身是 GUI 强交互工作），重点是**跑通工具链**：

- magic 批处理模式怎么用（无 GUI 也能 extract / DRC）；
- `extract all` → `ext2spice` 实际产生什么；
- netgen LVS 命令的精确形式（端口顺序、ports 写法）；
- 一个最常见的 LVS 坑：`M` 元件 vs `X` 子电路；
- 全部命令本机实测过（magic 8.3、netgen、ngspice-36），输出贴在下面。

## 一、5T-OTA 版图原则（用文字代替画图）

实际打开 magic 画时遵循这些约定：

| 决策 | 推荐 | 为什么 |
| --- | --- | --- |
| 输入对 M1/M2 | **公共质心 (common centroid)** 摆放，2 个或 4 个 finger | 抵消版图梯度（doping / temperature / stress）造成的 offset |
| 镜像负载 M3/M4 | 同方向并排，**dummy** 各 1 个夹两端 | 消除边缘效应，给 mismatch [[03-fd-pr-devices]] 提到的 Pelgrom 模型最佳条件 |
| 尾源 M5 + 镜像 M6 | 同方向并排 + dummy | 同上 |
| Guard ring | M5 周围加一圈 nwell tap → VDD，输入对周围加 psub tap → VSS | 抑制衬底耦合；数模混合中尤其重要 |
| 走线 | li1 走 cell 内部信号；metal1 走电源；输出 vout 走 metal2 | 减少 li1 寄生电阻，避免 IR drop |

[[03-fd-pr-devices]] 里所有 MOS 都是 4 端器件，body 必须显式走到 well tap —— 画版图时**永远留 tap 位置**。

参考 [[references.md#五-教科书]] 的 Hastings _The Art of Analog Layout_ 第 4–6 章详细讲匹配版图。

## 二、magic 批处理流程（核心）

虽然画图需要 GUI（magic 的 `-d XR` 模式），但**所有验证（DRC / extract / LVS）都能批处理跑**。下面的演示用 sky130 自带的 `sky130_fd_sc_hd__inv_1`（标准单元库里的反相器），因为它是个**已知 clean** 的 cell，方便验证工具链是否正确。

### 2.1 启动 batch + 加载 cell

```bash
cd ~/work/ota_5t       # 工作目录
PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole \
        -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load sky130_fd_sc_hd__inv_1
puts "loaded: [cellname list top]"
quit -noprompt
EOF
```

关键 flag：

| flag | 作用 |
| --- | --- |
| `-dnull` | 不打开图形窗口（headless）|
| `-noconsole` | 不开独立 Tcl 控制台 |
| `-rcfile FILE` | 用指定 rcfile 启动（[[05-rc-entrypoints]] 已讲）|
| `<<'EOF' ... EOF` | 把 magic 命令通过 stdin 灌进去 |

**实测输出**（节选）：

```text
Sourcing design .magicrc for technology sky130A ...
Using technology "sky130A", version 1.0.493-0-g0fe599b
Cell sky130_fd_sc_hd__inv_1 read from path /home/yifei/.volare/sky130A/libs.ref/sky130_fd_sc_hd/mag
loaded: (UNNAMED) sky130_fd_sc_hd__inv_1
```

`(UNNAMED)` 是 magic 的当前 view（默认顶层是空），后面跟着加载的 cell。

### 2.2 跑 DRC

```bash
PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load sky130_fd_sc_hd__inv_1
drc check
drc catchup
puts "DRC errors: [drc count]"
quit -noprompt
EOF
```

`drc catchup` 强制把所有挂起的 incremental 检查跑完。

**实测输出**：

```text
Total DRC errors found: 0
DRC errors:
```

`drc count` 在 0 错时不输出数字（设计选择，可怜的 UX）。要严格判断"是否 clean"，用 `drc count` 返回值 + Tcl 拼装：

```tcl
set errors [drc count]
if {$errors == 0 || $errors == ""} { puts "DRC CLEAN" } else { puts "$errors errors" }
```

完整签核还应跑 [[06-drc-lvs-flow]] 里讲的 `run_standard_drc.py`（包含所有 derived 规则）。

### 2.3 Extract → SPICE

```bash
PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load sky130_fd_sc_hd__inv_1
extract all
ext2spice lvs
ext2spice
quit -noprompt
EOF
```

三步链路：

| 命令 | 做什么 | 产物 |
| --- | --- | --- |
| `extract all` | 把 cell 内部图形识别成 device 与 net | `*.ext` 文件 |
| `ext2spice lvs` | 设 LVS 友好模式（保留 cell 边界、不展开器件 subckt）| 改 ext2spice 内部 flag |
| `ext2spice` | 真正写 SPICE 文件 | `*.spice` |

**实测产出 `sky130_fd_sc_hd__inv_1.spice`**：

```spice
* NGSPICE file created from sky130_fd_sc_hd__inv_1.ext - technology: sky130A
.subckt sky130_fd_sc_hd__inv_1 A VGND VNB VPB VPWR Y
X0 Y A VGND VNB sky130_fd_pr__nfet_01v8     ad=0.169 pd=1.82 as=0.169 ps=1.82 w=0.65 l=0.15
X1 Y A VPWR VPB sky130_fd_pr__pfet_01v8_hvt ad=0.26  pd=2.52 as=0.26  ps=2.52 w=1    l=0.15
.ends
```

**注意几点**（呼应 [[03-fd-pr-devices]]）：

- 器件用 `X` 前缀（子电路实例），不是 `M`（BSIM 直调）—— [[04-sim-models-corners]] 里讲过 sky130 永远用 subckt wrapper。
- `ad / as / pd / ps` 是 magic 从版图实际算出来的扩散区面积/周长 —— 比手算精确。
- pin 顺序：`A VGND VNB VPB VPWR Y` —— magic 按 `port` 命令的顺序导出，与 Liberty / xschem 符号顺序必须**严格一致**。

### 2.4 注意：`.ext` 写到 PDK 目录的坑

实测发现：load 一个**来自 PDK 路径**的 cell 时，`extract all` 把 `.ext` 写回 PDK 的源目录 (`~/.volare/sky130A/libs.ref/sky130_fd_sc_hd/mag/`)。这会**污染 PDK**（Volare 升级时可能被覆盖）。

**正解**：把 cell 复制一份到工作目录，再 load。

```bash
cp $PDK_ROOT/$PDK/libs.ref/sky130_fd_sc_hd/mag/sky130_fd_sc_hd__inv_1.mag .
# .ext 现在会写到当前目录
```

自己手画的版图本来就在 `~/work/...` 里，不会碰到这个坑。

## 三、LVS：netgen 批处理

### 3.1 准备 "schematic" 网表

LVS 需要**两个网表对比**：版图提取的，和原理图导出的。这里我们手写一个等价的：

```spice
* inv_schematic.spice
.subckt sky130_fd_sc_hd__inv_1 A VGND VNB VPB VPWR Y
X0 Y A VGND VNB sky130_fd_pr__nfet_01v8     w=0.65 l=0.15
X1 Y A VPWR VPB sky130_fd_pr__pfet_01v8_hvt w=1.0  l=0.15
.ends
```

⚠️ **`X` 前缀必须**。下面解释为什么。

### 3.2 跑 netgen

```bash
netgen -batch lvs \
  "sky130_fd_sc_hd__inv_1.spice sky130_fd_sc_hd__inv_1" \
  "inv_schematic.spice            sky130_fd_sc_hd__inv_1" \
  $PDK_ROOT/$PDK/libs.tech/netgen/sky130A_setup.tcl \
  lvs_report.out
```

参数排布（`netgen -batch lvs` 标准 4 元）：

```text
"<netlist_1_path> <top_cell_name>"      ← 版图提取的
"<netlist_2_path> <top_cell_name>"      ← 原理图的
<setup.tcl>                             ← PDK 提供
<report.out>                            ← 输出
```

**实测输出**（最后几行）：

```text
Circuit 1 contains 2 devices, Circuit 2 contains 2 devices.
Circuit 1 contains 6 nets,    Circuit 2 contains 6 nets.

Final result:
Circuits match uniquely.
.
Logging to file "lvs_report.out" disabled
LVS Done.
```

`Circuits match uniquely` = 完美匹配（器件 + 拓扑 + 引脚都对得上）。

### 3.3 故意制造 LVS 错误看输出（M vs X 坑）

如果把 `inv_schematic.spice` 里的 `X0` / `X1` 改成 `M0` / `M1`：

```spice
M0 Y A VGND VNB sky130_fd_pr__nfet_01v8 w=0.65 l=0.15
M1 Y A VPWR VPB sky130_fd_pr__pfet_01v8_hvt w=1.0 l=0.15
```

LVS 会报：

```text
Final result:
Netlists do not match.
Port matching may fail to disambiguate symmetries.

(详细 dump 里出现 proxy 端口：proxydrain / proxygate / proxysource / proxybulk)
```

原因：SPICE 里 `M` 是 BSIM 原语，端口顺序是 `D G S B` 硬编码 4 端；`X` 是子电路实例，端口顺序由 `.subckt sky130_fd_pr__nfet_01v8` 定义。两者**不能混**——版图提取的是 `X`（子电路），schematic 也必须用 `X`。

这是新手最常踩的坑之一。

### 3.4 sky130A_setup.tcl 帮你做了什么

这个文件（[[06-drc-lvs-flow]] 已介绍）在背后给 netgen 灌了：

- 所有 `sky130_fd_pr__*` 器件名注册为"原子设备类"（不展开内部），并标 `parallel enable`（并联管子自动合并）+ `permute` 允许漏源对调。
- 处理 Electric 等工具的 `library__cellname` 命名差异。
- diode / fill cell 当作"并联多个相同"。

**结果**：你只要把器件名写对，netgen 就能识别。哪天发现 LVS 把 nfet 不认时，多半是器件名 typo（如 `sky130_fd_pr__nfet01v8` 少了下划线）。

## 四、LVS：KLayout 路径

替代路径（[[06-drc-lvs-flow]] 讲过的 KLayout）：

```bash
# 先把 .mag 转 .gds（在 magic batch 里）
PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load sky130_fd_sc_hd__inv_1
gds write inv_1.gds
quit -noprompt
EOF

# 再跑 KLayout LVS
python3 $PDK_ROOT/$PDK/libs.tech/klayout/lvs/run_lvs.py \
        --design=inv_1.gds \
        --net=inv_schematic.spice \
        --report=inv_1_lvs.lyrdb \
        --thr=4 --run_mode=deep --set_verbose
```

`run_lvs.py` 比 netgen 慢些但更接近商业 Calibre 行为；签核值得跑一遍。

## 五、把 5T-OTA 推过这条链

5T-OTA 的版图工作只能在 GUI 里完成（用 magic `magic -d XR ota_5t.mag`）。一旦画完，验证流程就是把 `sky130_fd_sc_hd__inv_1` 换成 `ota_5t`：

```bash
# DRC
PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load ota_5t
drc check
drc catchup
puts "DRC errors: [drc count]"
quit -noprompt
EOF

# Extract
PDK_ROOT=$HOME/.volare PDK=sky130A MAGTYPE=mag \
  magic -dnull -noconsole -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load ota_5t
extract all
ext2spice lvs
ext2spice
quit -noprompt
EOF

# 从 xschem 导出原理图网表（GUI 操作，得到 ota_5t_xschem.spice）

# LVS
netgen -batch lvs \
  "ota_5t.spice          ota_5t" \
  "ota_5t_xschem.spice   ota_5t" \
  $PDK_ROOT/$PDK/libs.tech/netgen/sky130A_setup.tcl \
  lvs_report.out
```

## 六、xschem 导出网表的注意点

xschem 默认导出会带 testbench wrapper（`.tran` / 电源等），LVS 比对要的是**只包含 OTA 的 `.subckt`**。做法：

1. 在 xschem 里把 OTA 单独存一个 sch（不放 testbench）。
2. **Symbol → Make Symbol** 把这个 sch 包成 symbol（这一步 LVS 比对很重要）。
3. 选 `Simulation → set 'spice' netlist mode → Netlist`，输出 `ota_5t.spice`，里面只有 `.subckt ota_5t ... .ends ota_5t`。

xschem 的 port 名要和 magic 里的 `port` 一致 —— [[06-drc-lvs-flow]] 提到的"三方对齐"。

## 七、几个本章实测过的坑

- **`drc count` 0 错时不返回数字**：用 Tcl `if {$errors == 0 || $errors == ""}` 双判。
- **`.ext` 写到 PDK 源路径**：load 来自 PDK 的 cell 时，extract 把 `.ext` 写回 PDK 目录污染只读区。先 `cp` 到工作目录再 load。
- **`X` vs `M`**：上面专门一节，新手必踩。
- **没设 MAGTYPE**：默认 `mag`，问题不大；但实例化别人的 cell 时建议显式 `MAGTYPE=maglef`，extract 速度快 5-10x（[[03-fd-pr-devices]] 与 [[05-rc-entrypoints]] 已讲）。
- **`netgen -batch lvs` 的引号必须**：两个网表参数必须 `"path cellname"` 用引号包成单个 token，否则 netgen 把 cellname 当下一个 flag 解析。
- **port 顺序混乱**：magic 里 `port make` 给 label 升格成 port 的顺序决定输出的 pin 顺序。建议**永远按字母序**打 port，xschem 那边也按字母序排，避免比对失败但 "Cell pin lists are equivalent" 这种貌似匹配实际靠 netgen 内部 permute 救回来的情况。
- **gds write 后 KLayout 看不到**：sky130 的 GDS layer map 在 `libs.tech/klayout/tech/sky130A.map` —— KLayout 用 `-t` 指定 tech 才会按这个 map 解码，否则颜色全错。
- **`extract all` 慢**：大版图（万管子级）extract 很慢，可以 `extract style ngspice(si)` 切到 SI 单位风格、或者用 magic 的 hierarchical extract（默认）减小重复工作量。

## 下一步

笔记 10：**PEX 提取与后仿** —— magic 的 `extract style ngspice(R+C)` 把寄生 R/C 包进网表，导出带寄生的 SPICE，再回到 [[08-tutorial-ota-sim]] 的 corner / MC 流程做后仿对比，看版图寄生让性能掉多少。
