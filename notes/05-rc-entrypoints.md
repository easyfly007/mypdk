# 05 — magicrc / xschemrc 入口配置

## 目标

把"启动一个 sky130 工作目录要做什么"具体化：

- 哪几个环境变量必须导出（`PDK_ROOT` / `PDK` / `MAGTYPE` / `XSCHEM_USER_LIBRARY_PATH`）；
- 工作目录里要不要放 `.magicrc` / `xschemrc`，放哪种最干净；
- magic 启动时 `sky130A.magicrc` 实际做了什么（tech load → 设备生成器 → addpath）；
- xschem 启动时 `xschemrc` 实际做了什么（XSCHEM_LIBRARY_PATH → 选 PDK 变体 → fet_drc 检查）；
- netgen / klayout 是怎么找配置的，不需要单独 rc 但有自己的入口文件。

参考 [[02-pdk-structure]] 关于这些文件在 `libs.tech/` 下的位置，以及 [[04-sim-models-corners]] 关于 `.lib` 引用。

## 必备环境变量

写进 `~/.bashrc`（或 `~/.zshrc`）：

```bash
export PDK_ROOT=$HOME/.volare
export PDK=sky130A
# 可选：让 magic 默认用抽象视图，编辑 nfet 实例时不展开 well/contact
export MAGTYPE=maglef
# 可选：把自己的 xschem 库放进 XSCHEM_LIBRARY_PATH
export XSCHEM_USER_LIBRARY_PATH=$HOME/code/mypdk/xschem_lib
```

| 变量 | 谁用 | 含义 | 推荐值 |
| --- | --- | --- | --- |
| `PDK_ROOT` | magic, xschem, netgen, OpenLane | Volare 的根（含 `sky130A` 符号链接的那一层）| `$HOME/.volare` |
| `PDK` | xschem, OpenLane | 工艺变体名 | `sky130A` |
| `MAGTYPE` | magic | 加载 cell 时用 mag 还是 maglef 视图 | `maglef`（顶层版图实例化别人时）/ `mag`（编辑 cell 内部时）|
| `XSCHEM_USER_LIBRARY_PATH` | xschem | 用户私有符号库目录，会被 append 到 `XSCHEM_LIBRARY_PATH` | 你自己项目的 xschem 库 |
| `NETGEN_COLUMNS` | netgen | 报告宽度（一行多少字符）| 通常不设，默认 80 |

**`PDK_ROOT` 层级**：Volare 把"当前激活版本"做成符号链接（`~/.volare/sky130A → volare/sky130/versions/<hash>/sky130A`），所以 `PDK_ROOT` 指 `~/.volare` 就行，工具会拼出 `${PDK_ROOT}/${PDK}/libs.tech/...`。别按老教程指到 `volare/sky130/build/<hash>`，那是 Volare 内部目录，跨版本会断。

## 工作目录的最小布置

推荐每个项目目录长这样：

```text
my_design/
├── .magicrc            ← 一行 source 指向 PDK 自带 magicrc（不要复制全文）
├── xschemrc            ← 同上
├── schematics/         ← .sch 文件
├── layouts/            ← .mag 文件
├── sim/                ← testbench .sp 与仿真输出
└── lvs/                ← LVS 工作目录
```

### `.magicrc`（项目级，2 行）

```tcl
# Source the SkyWater sky130A magicrc, then any local overrides.
source $env(PDK_ROOT)/$env(PDK)/libs.tech/magic/sky130A.magicrc
# 项目自定义（可选）：例如 addpath ./layouts
```

magic 启动时优先读当前目录的 `.magicrc`，再读 `~/.magicrc`，最后才考虑 `-rcfile` 指定的文件。**不要 `cp sky130A.magicrc .`** —— PDK 升级时复制副本就脱节了。

### `xschemrc`（项目级，2 行）

```tcl
# Source the SkyWater xschemrc, then anything project-local.
source $env(PDK_ROOT)/$env(PDK)/libs.tech/xschem/xschemrc
# 例如：覆盖默认启动窗口
set XSCHEM_START_WINDOW {}
```

xschem 启动时找 `./xschemrc → ~/.xschem/xschemrc → 系统 xschemrc`，找到第一个就用。

## sky130A.magicrc 拆解（共 80 行）

整个文件分 5 段，从上到下：

```tcl
# ① grid / 全局选项
set scalefac [tech lambda]
if {[lindex $scalefac 1] < 2} { scalegrid 1 2 }
drc euclidean on            ;# DRC 用欧氏距离规则
catch {random seed}         ;# 每次启动用随机种子（GDS 不可重复）；想要 reproducible 改成 fixed
ext2spice scale off          ;# 提取时不再叠加 scale，避免和 model 文件的 .option scale=1u 撞车

# ② PDK 路径
if {[catch {set PDK_ROOT [file nativename $env(PDK_ROOT)]}]} {
    set PDK_ROOT $::env(HOME)/.volare/volare/sky130/build/0fe599b2afb6708d281543108caf8310912f54af
}
tech load $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tech
if {[tech name] != "sky130A"} {quit -noprompt}    ;# tech load 失败就直接退出

# ③ 设备生成器（PCells）
source $PDK_ROOT/sky130A/libs.tech/magic/sky130A.tcl

# ④ 网络命名约定
snap lambda
set VDD VPWR
set GND VGND
set SUB VSUBS

# ⑤ MAGTYPE + addpath（关键）
if {[catch {set MAGTYPE $env(MAGTYPE)}]} { set MAGTYPE mag }
if {[file isdir ${PDK_ROOT}/sky130A/libs.ref/${MAGTYPE}]} {
    addpath ${PDK_ROOT}/sky130A/libs.ref/${MAGTYPE}/sky130_fd_pr
    addpath ${PDK_ROOT}/sky130A/libs.ref/${MAGTYPE}/sky130_fd_io
    ... (一长串 sky130_fd_sc_* / sky130_osu_sc_* / sky130_sram_macros)
} else {
    addpath ${PDK_ROOT}/sky130A/libs.ref/sky130_fd_pr/${MAGTYPE}
    ... (兼容老布局)
}
```

**重要的几个 hook**：

- **`drc euclidean on`** —— sky130 的某些规则只在欧氏距离模式下才正确。一定不要关掉。
- **`random seed`** —— 决定 GDS 输出的某些"随机生成 ID"（如 cell 名后缀）。想做 GDS bit-by-bit 可比的 regression，把 `catch {random seed}` 改成 `random seed 12345`。
- **`ext2spice scale off`** —— magic 默认会把提取后的 SPICE 网表中的 `w/l` 乘上 `.option scale`，但 ngspice 的模型文件**自己设了 `.option scale=1u`**，两次叠加会导致器件大 1000 倍。这一行是为了避免这个坑。
- **`addpath ... ${MAGTYPE}` 的双 if-else** —— Volare 新版把 `libs.ref/<lib>/mag/` 和 `libs.ref/<lib>/maglef/` 平铺；老版有过 `libs.ref/mag/<lib>` 的扁平结构。两种布局都兼容。

**`magic::query_mylib_ip` / `magic::query_my_projects`** —— 文件末尾两行 `catch` 调用。这是 magic 留给你的"插入个人 IP / 项目目录"钩子，未定义时被 `catch` 吞掉。要扩展时在 `~/.magic_setup.tcl` 之类的文件里 `proc magic::query_my_projects {} { addpath ... }` 重定义。

## xschemrc 拆解

xschem 的 rc 比 magic 更复杂（260+ 行）。**与 sky130 相关的核心段在文件末尾几十行**，前面几百行都是通用 xschem 选项的默认值，已经被注释掉，不用动。

实际起作用的关键段：

```tcl
# ① 系统库路径（位于文件前半段）
set XSCHEM_LIBRARY_PATH {}
append XSCHEM_LIBRARY_PATH :${XSCHEM_SHAREDIR}/xschem_library/devices  ;# 通用器件（电源、地、port）
append XSCHEM_LIBRARY_PATH :${XSCHEM_SHAREDIR}/xschem_library          ;# 通用示例
append XSCHEM_LIBRARY_PATH :[file dirname [info script]]               ;# xschemrc 所在目录（让 sky130 库相对路径生效）
append XSCHEM_LIBRARY_PATH :$USER_CONF_DIR/xschem_library              ;# ~/.xschem/xschem_library

# ② SKYWATER 段（文件末尾，决定 PDK 接入）
if { [info exists env(PDK_ROOT)] && $env(PDK_ROOT) ne {} } {
    set PDK_ROOT $env(PDK_ROOT)
} else {
    # 找几个 fallback 路径
}
if {[info exists env(PDK)]} { set PDK $env(PDK) } else { set PDK sky130A }
set SKYWATER_MODELS ${PDK_ROOT}/${PDK}/libs.tech/combined
set SKYWATER_STDCELLS ${PDK_ROOT}/${PDK}/libs.ref/sky130_fd_sc_hd/spice

# ③ 加入 sky130 xschem 库（关键）
set XSCHEM_START_WINDOW ${PDK_ROOT}/${PDK}/libs.tech/xschem/sky130_tests/top.sch
append XSCHEM_LIBRARY_PATH :${PDK_ROOT}/${PDK}/libs.tech/xschem

# ④ 用户路径钩子
if { [info exists ::env(XSCHEM_USER_LIBRARY_PATH) ] } {
    append XSCHEM_LIBRARY_PATH :$env(XSCHEM_USER_LIBRARY_PATH)
}
```

注意几个变量名：

| 变量 | 含义 |
| --- | --- |
| `XSCHEM_SHAREDIR` | xschem 自带的器件库目录（VDD/GND/probe 等，跟工艺无关）|
| `XSCHEM_LIBRARY_PATH` | xschem 找符号的搜索路径（":" 分隔）|
| `SKYWATER_MODELS` | 给生成的 SPICE 网表头部插入 `.include "$SKYWATER_MODELS/sky130.lib.spice"` 用 |
| `SKYWATER_STDCELLS` | 标准单元 SPICE 网表路径（数字电路混合仿真时引用）|
| `XSCHEM_START_WINDOW` | xschem 启动时打开的默认 sch；空字符串 = 不开任何文件 |

**`fet_drc` 过程**：xschemrc 里定义了一个 `proc fet_drc`，对常用 MOS 做最小尺寸检查（`w/nf < 0.42` 或 `l < 0.15` 等）。这是 xschem 在画原理图时给你做的"等价于 lint"的检查，不替代真正的 DRC，但能在画错尺寸时立刻警告。

**为什么 `SKYWATER_MODELS` 指向 `combined/` 而不是 `ngspice/`**：sky130 的 `ngspice/sky130.lib.spice` 和 `combined/sky130.lib.spice` 内容一样，xschem 选 combined 是为了在生成给商业 SPICE / 其它 simulator 的网表时也能用。

## netgen 入口

netgen 没有自己的"rc 文件"，它是被 magic 直接调起来的：

```
magic ext2spice extracted   ;# 从版图提取 SPICE
netgen -batch lvs \
       "extracted.spice cellname" \
       "schematic.spice cellname" \
       $PDK_ROOT/$PDK/libs.tech/netgen/sky130A_setup.tcl \
       lvs_report.out
```

第三个参数 `sky130A_setup.tcl` 是关键：里面列出 PDK 已知的所有器件名（`sky130_fd_pr__res_high_po_*` / `sky130_fd_pr__nfet_*` …），告诉 netgen 哪些子电路当成"器件"看（不展开内部 mosfet），哪些可以 permute（如 R 的两端）。

**你不需要复制或改它**，magic 的 `extract lvs` + `ext2spice` 流程会自动用 `$PDK_ROOT/$PDK/libs.tech/netgen/sky130A_setup.tcl`。详细 LVS 流程在 [[06-drc-lvs-flow]]。

## klayout 入口

KLayout 的工艺定义靠 `.lyt` 文件而不是 rc：

```
libs.tech/klayout/tech/
├── sky130A.lyt    ★ 主工艺描述（XML）：dbu、layer 映射、reader 选项
├── sky130A.lyp    Layer 显示样式（颜色/填充）
└── sky130A.map    GDS layer/datatype 到名字的纯文本表
```

要让 KLayout 把 sky130 当作 "Technology"，需要在 GUI 里 **File → Setup → Technologies → Add** 指向 `sky130A.lyt`；或者命令行：

```bash
klayout -t -nn $PDK_ROOT/sky130A/libs.tech/klayout/tech/sky130A.lyt your_layout.gds
```

`-t` 表示用指定 tech 打开。

DRC / LVS 入口是单独的：

```
libs.tech/klayout/drc/sky130A.lydrc        ← KLayout 内置 DRC 主入口
libs.tech/klayout/drc/sky130A_mr.drc       ← 多线程命令行版
libs.tech/klayout/lvs/sky130.lvs           ← KLayout LVS 主入口
```

详细用法在 [[06-drc-lvs-flow]]。

## 自检命令（确认环境是否就绪）

把环境变量写进 shell 配置并 `source` 后跑：

```bash
echo $PDK_ROOT $PDK $MAGTYPE
ls $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc        # 应存在
ls $PDK_ROOT/$PDK/libs.tech/xschem/xschemrc              # 应存在
ls $PDK_ROOT/$PDK/libs.tech/ngspice/sky130.lib.spice     # 应存在

# magic 最小启动测试（启动 → 看到 "Sourcing design .magicrc"）
mkdir -p ~/tmp/sky130_smoke && cd ~/tmp/sky130_smoke
cat > .magicrc <<'EOF'
source $env(PDK_ROOT)/$env(PDK)/libs.tech/magic/sky130A.magicrc
EOF
magic -d XR -T sky130A &

# xschem 最小启动测试
cat > xschemrc <<'EOF'
source $env(PDK_ROOT)/$env(PDK)/libs.tech/xschem/xschemrc
EOF
xschem &
```

如果 magic 启动后状态栏显示 `Sky130A`，xschem 启动后菜单 "File → Open" 能浏览到 `sky130_fd_pr/` 下的 `nfet_01v8.sym`，环境就 OK 了。

## 几个容易踩的坑

- **`PDK_ROOT` 末尾不要带斜杠**：`$HOME/.volare/` 写成 `$HOME/.volare` 才安全，否则路径拼成 `//`，老版 magic 会卡。
- **`.magicrc` vs `~/.magicrc` 优先级**：当前目录的 `.magicrc` 完全覆盖 `~/.magicrc`，不是叠加。两边都设了同一个 PDK 路径但版本不同，会有"为什么器件库不对"的诡异表现。
- **`MAGTYPE=mag` vs `MAGTYPE=maglef`**：编辑顶层版图时该用 `maglef`（实例化时只读 pin / obs，省内存）；编辑 cell 自己的内部时该 `mag`（要看到完整层）。在 `.magicrc` 里写死会让另一种用例不方便，更好做法是在 `~/.bashrc` 里设默认 `MAGTYPE=maglef`，临时编辑时 `MAGTYPE=mag magic ...`。
- **xschem 不自动加载 simulator**：xschem 只生成网表，仿真要么从 GUI 的 "Simulation → ngspice" 调，要么手工 `ngspice tb.spice`。**simulation 目录默认在 `~/.xschem/simulations/`**，要放到项目目录下需要在 xschemrc 里 `set local_netlist_dir 1`。
- **xschem 启动时打开 `top.sch` 卡住**：默认 `XSCHEM_START_WINDOW` 指向 `sky130_tests/top.sch`，那是 PDK 自带的大测试。第一次启动会觉得"为什么进来一堆电路图"。在项目 `xschemrc` 里写 `set XSCHEM_START_WINDOW {}` 关掉。
- **netgen / KLayout 不读 `PDK_ROOT`**：除非你显式把 `$PDK_ROOT/...` 写到调用命令里。OpenLane 之类的脚本帮你拼路径，纯手工流程要自己写。
- **复制 magicrc 到项目里 = 升级断裂**：永远 `source` 不要 `cp`。Volare 升级 PDK 后，你工作目录里的副本就停在老版本上。
- **`bespice_listen_port 2022` 是默认开的**：xschemrc 默认 `set bespice_listen_port 2022`。如果你机器有别的服务占了 2022 端口，启动 xschem 会报警告（不致命，但烦）；在项目 xschemrc 里 `set bespice_listen_port 0` 关掉。

## 下一步

笔记 06：**DRC / LVS 流程实战** —— 把 magic-DRC、magic-extract→netgen-LVS、KLayout DRC 三条线串成可跑的命令序列，并对照"为什么 sky130 同时给两套 DRC 引擎"。
