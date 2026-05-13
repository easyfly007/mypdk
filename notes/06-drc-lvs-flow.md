# 06 — DRC / LVS 流程实战

## 目标

把"画完版图之后到底要跑哪几个命令、用哪几个文件、出什么报告"具体化：

- **两套独立的 DRC 引擎**——magic 内置 DRC 与 KLayout DRC，各自的命令、覆盖范围、用法场景；
- **两条 LVS 路径**——magic → netgen（开源默认）和 KLayout LVS（独立、更接近签核）；
- 配套检查：**天线、密度、填充**怎么跑；
- 报告应该长什么样、常见的"假阴性 / 假阳性"。

参考 [[02-pdk-structure]] 关于这些文件在 `libs.tech/{magic,klayout,netgen}/` 下的位置，[[05-rc-entrypoints.md]] 关于工具入口。

## 物理签核四件套

模拟版图完工后，一般按下面顺序跑：

```text
┌─────────────────┐   ┌──────────────┐   ┌────────────┐   ┌──────────────┐
│ ① DRC （规则）  │ → │ ② Antenna    │ → │ ③ Density  │ → │ ④ LVS        │
│  Design Rules   │   │  天线效应    │   │  密度填充  │   │  网表 ↔ 版图 │
└─────────────────┘   └──────────────┘   └────────────┘   └──────────────┘
                                         (插入 dummy)
                                                                ↓
                                                          ⑤ PEX 提取（笔记 0X）
```

四件套都通过才算"clean"。

## 一、Magic 内置 DRC

### 1.1 规则来自哪里

DRC 规则全部塞在 `libs.tech/magic/sky130A.tech` 文件里。它一个文件 6101 行，包含从 `tech / planes / types / contact / connect / drc / extract / cifoutput / cifinput` 等 ~17 个区段。其中 DRC 区段包含约 **211 条规则**：

```text
spacing     84 条      间距规则（同层 / 跨层）
width       45 条      最小线宽
surround    45 条      环绕（如 well 包 active 边距）
edge4way    27 条      四方向边缘规则
area        10 条      最小面积
```

如果想看原始规则文本（适合理解某个 DRC 报错的来源），在 magic 启动后：

```tcl
:tech section drc | head -100      ;# 输出 sky130A.tech 的 drc 段开头
```

### 1.2 交互式 DRC（推荐画版图边画边用）

```tcl
:drc check          ;# 强制重检整个 cell（默认增量检查，编辑时自动跑）
:drc count          ;# 打印总错误数
:drc count total    ;# 包含所有层级子 cell 的错误总数
:drc list why       ;# 列出当前 box 范围内所有违反的规则名 + 描述
:drc find           ;# 把视图跳到下一个 DRC 错误位置
```

实战节奏：画 → 看右下角红色块（magic 实时高亮 DRC 错） → `drc find` 跳过去 → `drc list why` 看是什么规则 → 修正 → 重复。

### 1.3 批处理 DRC（签核前用）

```bash
$PDK_ROOT/$PDK/libs.tech/magic/run_standard_drc.py <layout_name>
```

`<layout_name>` 可以是 `.mag` 也可以是 `.gds`。脚本会启动 magic batch 模式，跑完所有规则（**排除天线和密度**——这两个有独立脚本），生成 `<layout_name>_drc.txt`，里面格式：

```text
sky130_fd_pr__nfet_01v8 cell:
    Active spacing < 0.27um (rule SK_M.AC.5):
        x1 y1 x2 y2 (bbox)
        ...
    M1 width < 0.14um (rule SK_M.M1.1):
        ...
```

### 1.4 天线检查

```bash
$PDK_ROOT/$PDK/libs.tech/magic/check_antenna.py <layout_name>
```

输出 `<layout_name>_antenna.txt`。天线规则衡量"长 metal 走线在 etch 阶段累积的电荷会不会击穿栅极"。模拟设计 manual layout 时违例不常见，但顶层布线 / 长 signal trace 要小心。

### 1.5 密度检查 + 填充生成

```bash
# 先生成 dummy
$PDK_ROOT/$PDK/libs.tech/magic/generate_fill.py <layout_name>
# 再检查密度
$PDK_ROOT/$PDK/libs.tech/magic/check_density.py <gds_file>
```

**顺序很重要**：先 `generate_fill.py` 把 dummy 填进去，再 `check_density.py` 验证密度落在 foundry 要求的窗口（典型 30%~70%）。如果手画的小芯片密度太低，必须填 dummy。

## 二、KLayout DRC

### 2.1 两个入口

```text
libs.tech/klayout/drc/sky130A.lydrc       ← GUI 用：XML 包装，注册到 Tools → DRC 菜单
libs.tech/klayout/drc/sky130A_mr.drc      ← 命令行用：多线程纯 Ruby，可批量跑
```

两个**底层规则一致**（都来自 SkyWater periphery / layers rules），只是包装不同。

### 2.2 命令行批处理

```bash
klayout -b \
        -rd input=my_layout.gds \
        -rd report=sky130_drc.txt \
        -r $PDK_ROOT/$PDK/libs.tech/klayout/drc/sky130A_mr.drc
```

- `-b` = batch mode
- `-rd key=val` = 给 DRC 脚本传变量（脚本里 `if $input ... source($input)`）
- `-r script.drc` = 运行该脚本
- 报告默认输出 `report` 参数指定的文件；不指定就在 GDS 同目录写 `sky130_drc.txt`。

### 2.3 规则分组开关

`sky130A_mr.drc` 文件头部有几个布尔开关：

```ruby
FEOL = false   # front-end-of-line：active/poly/contact 层
BEOL = false   # back-end-of-line：所有金属、via
OFFGRID = false # 0.005 µm 网格 / 角度
SEAL = false   # 密封环规则
FLOATING_MET = false  # 悬浮 metal 检查
SRAM_EXCLUDE = true   # 跳过 SRAM macro 内部
```

默认全 `false` 意味着脚本"啥都不查"。**实际用之前必须按需打开**，例如版图签核应该开 `FEOL = BEOL = OFFGRID = SEAL = true`。在脚本里直接改，或者命令行 `-rd FEOL=true` 传入（脚本顶部要写 `FEOL = (defined? $FEOL) ? $FEOL : false`）。

### 2.4 GUI 模式

打开 GDS，菜单 **Tools → DRC → drc_scripts → SKY130 DRC**（来自 `sky130A.lydrc` 的 `menu-path`）。运行后错误会作为 markers 标在版图上，双击跳转。

## 三、Magic + Netgen LVS（开源默认路径）

### 3.1 三步走

```bash
# Step 1：在 magic 里从版图提取
magic -d XR -T sky130A -rcfile $PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc <<'EOF'
load my_cell
extract all
ext2spice lvs
ext2spice
quit -noprompt
EOF
# 会生成 my_cell.ext / my_cell.spice（提取出来的 SPICE 网表）

# Step 2：准备原理图网表（从 xschem 导出，或手写）
#         得到 my_cell_schematic.spice

# Step 3：用 netgen 比对
netgen -batch lvs \
       "my_cell.spice          my_cell" \
       "my_cell_schematic.spice my_cell" \
       $PDK_ROOT/$PDK/libs.tech/netgen/sky130A_setup.tcl \
       lvs_report.out
```

`netgen lvs` 的两个参数分别是 `"网表 顶层 cell 名"`，**前者是版图提取的，后者是原理图导出的**——顺序无所谓，netgen 会找匹配。

### 3.2 ext2spice 的关键开关

```tcl
ext2spice lvs           ;# LVS 友好模式：保留 cell 边界、不把器件展开
ext2spice scale off      ;# 与 magicrc 里设的一致，避免 .option scale 双倍叠加
ext2spice hierarchy on   ;# 保留层次（默认）
ext2spice subcircuit top off  ;# 顶层不包成 subckt
ext2spice global VPWR VGND VSUBS  ;# 全局节点声明
ext2spice short resistor  ;# 把短路电阻当短路处理（默认）
```

`ext2spice lvs` 是个 alias，等同于把上述 LVS 推荐设置一次性打开。

### 3.3 sky130A_setup.tcl 干了什么

netgen 自己**不知道** sky130 的器件名。`sky130A_setup.tcl` 这个 200 行的脚本告诉它：

```tcl
# 1. 把所有 sky130 标准器件登记成"器件单元"（subcircuit，不展开内部）
foreach dev {sky130_fd_pr__nfet_01v8 sky130_fd_pr__pfet_01v8 ...} {
    property "-circuit1 $dev" parallel enable     ;# 允许并联合并
    permute "-circuit1 $dev" 1 2                  ;# 允许 source/drain 互换
}

# 2. 电阻两端可对调
foreach res {sky130_fd_pr__res_high_po_*} {
    property "-circuit1 $res" parallel enable
    permute "-circuit1 $res" 1 2
}

# 3. 处理 "library__cellname" 风格的命名差异（Electric 等工具产生）

# 4. 把 fill / decap / diode 等 "并联多个相同器件" 当作单个看
```

**你几乎不需要改这个文件**。极个别情况下，自定义 subckt 没被识别，可以在自己的 `~/.netgenrc` 里 append 额外的 `permute` / `property` 指令。

### 3.4 LVS 报告解读

`lvs_report.out` 末尾会给一个状态总结：

```text
Final result:
Netlists match uniquely.       ← 完美：网络拓扑 + 器件 + pin 全对
```

或者：

```text
Circuits do not match.
Devices: M1 (sky130_fd_pr__nfet_01v8) in circuit 1, no match in circuit 2.
Nets: N123 in circuit 1, no match in circuit 2.
Property errors: w=2.0u vs 1.8u for M3
```

常见三类错误：
1. **Devices don't match** — 器件类型不对（pfet 画成 nfet 之类）。
2. **Nets don't match** — 拓扑不一致（少一根线 / 多一根线）。
3. **Property errors** — 拓扑对了，但 `w / l / nf / mult` 不对——找到对应实例改尺寸。

## 四、KLayout LVS（独立路径）

`libs.tech/klayout/lvs/` 下有完整一套，Efabless 2022 年贡献：

```text
sky130.lvs        ← Ruby DSL 写的 LVS 规则（提取规则 + 比对规则）
sky130.lylvs      ← KLayout GUI LVS 用的 XML 配置
run_lvs.py        ← Python CLI 封装
README.md         ← 用法文档
```

### 4.1 命令行用法

```bash
python3 $PDK_ROOT/$PDK/libs.tech/klayout/lvs/run_lvs.py \
        --design=my_cell.gds \
        --net=my_cell_schematic.spice \
        --report=lvs_report.lyrdb \
        --output_netlist=extracted.spice \
        --thr=8 \
        --run_mode=deep \
        --set_verbose
```

关键参数：
- `--run_mode=deep`（推荐）vs `flat`（小电路）vs `tiling`（大芯片，分块）
- `--thr=N` 多线程
- `--set_combine` 自动合并并联器件
- `--set_purge` 清理悬空节点
- 输出 `.lyrdb` 文件可以拖回 KLayout GUI 查看高亮的错误

### 4.2 与 magic+netgen 的核心差异

| 维度 | magic + netgen | KLayout LVS |
| --- | --- | --- |
| 提取器 | magic（专门为 magic 的内部数据库优化） | KLayout（直接读 GDS） |
| 比对器 | netgen（C 库 + Tcl 扩展） | KLayout DSL（Ruby） |
| 速度 | 中等 | 多线程下更快 |
| 接近签核 | 不完全 | 更接近 commercial Calibre 的行为 |
| GUI 集成 | magic 内部 marker | KLayout markers + .lyrdb |
| 优势场景 | 边画边验证小 cell | 顶层 / 数模混合验证 |

实践建议：**单元级开发用 magic+netgen 快速迭代，顶层签核用 KLayout LVS 二次确认**。两套都过才放心。

## 五、必要 hookup：版图、原理图、网表三方一致

LVS 最容易踩的坑是"三方实际不指同一个东西"：

```text
xschem (.sch)  ──netlist──>  schematic.spice    ←┐
                                                  ├── netgen 比对
magic (.mag)   ──extract──>  extracted.spice    ←┘
                       │
                       └──>  .gds  ──klayout──>  KLayout LVS（直接读 GDS）
```

约束：

1. **顶层 cell 名**在三处必须一致（`.sch` 文件名、`.mag` cell 名、网表里的 `.subckt` 名）。
2. **pin 名**要对齐。xschem 用 `i_pin` / `o_pin` / `io_pin` 符号定义 port；magic 用 `port make` 命令打 port。
3. **VPWR / VGND / VSUBS** 在 magic 里设过 `set VDD VPWR` 等，xschem 默认也是这套名字，**不要改**。

## 六、典型工作流示例（OTA 单元）

```bash
# === 准备 ===
mkdir -p ~/work/ota_5t && cd ~/work/ota_5t
cat > .magicrc <<'EOF'
source $env(PDK_ROOT)/$env(PDK)/libs.tech/magic/sky130A.magicrc
EOF
cat > xschemrc <<'EOF'
source $env(PDK_ROOT)/$env(PDK)/libs.tech/xschem/xschemrc
set XSCHEM_START_WINDOW {}
EOF

# === 原理图 ===
xschem ota_5t.sch &
# 画完后：菜单 Simulation → set 'spice' netlist mode → Netlist 按钮
# → 输出 ota_5t.spice

# === 版图 ===
magic -d XR ota_5t.mag &
# 画完后：在 magic 命令行
#   :extract all
#   :ext2spice lvs
#   :ext2spice
# 得 ota_5t.ext / ota_5t.spice

# === DRC ===
$PDK_ROOT/$PDK/libs.tech/magic/run_standard_drc.py ota_5t
# 检查 ota_5t_drc.txt 为空

klayout -b -rd input=ota_5t.gds -rd report=ota_5t_klayout_drc.txt \
        -r $PDK_ROOT/$PDK/libs.tech/klayout/drc/sky130A_mr.drc

# === LVS（magic + netgen）===
netgen -batch lvs \
       "ota_5t.spice ota_5t" \
       "ota_5t_schematic.spice ota_5t" \
       $PDK_ROOT/$PDK/libs.tech/netgen/sky130A_setup.tcl \
       lvs_report.out

# === LVS（klayout）===
python3 $PDK_ROOT/$PDK/libs.tech/klayout/lvs/run_lvs.py \
        --design=ota_5t.gds --net=ota_5t_schematic.spice \
        --report=ota_5t_lvs.lyrdb --thr=8

# === 天线 ===
$PDK_ROOT/$PDK/libs.tech/magic/check_antenna.py ota_5t

# === 密度 / 填充（顶层才做，小 cell 跳过）===
$PDK_ROOT/$PDK/libs.tech/magic/generate_fill.py ota_5t
$PDK_ROOT/$PDK/libs.tech/magic/check_density.py ota_5t_filled.gds
```

跑完所有 `*_drc.txt` 为空 + LVS 两边都 "Netlists match uniquely" → 这个 cell 干净，可以集成。

## 七、几个容易踩的坑

- **`ext2spice scale` 双倍叠加**：`.magicrc` 里已经 `ext2spice scale off`，但用户脚本里再 `ext2spice scale on` 又开了一遍 → 提取出的器件 w/l 都大 1000 倍 → LVS 报 property error。**永远 off**。
- **KLayout DRC 默认啥都不查**：`sky130A_mr.drc` 头部 `FEOL=BEOL=OFFGRID=false`。第一次跑发现"没报错"别庆祝，先看脚本顶部开关是不是开了。
- **LVS 网表 / 版图顶层名不一致**：netgen 默认按文件里第一个 `.subckt` 当顶层。如果 xschem 导出的 `.spice` 把 testbench 放最前面 → 比对的是 testbench 而不是 DUT。导网表时关掉 testbench 包装，或用 `netgen lvs "file cellname" ...` 显式指定 cellname。
- **VPWR / VGND 大小写不对**：sky130 约定全大写 `VPWR / VGND / VSUBS`。xschem 默认 OK；自写 testbench 时用 `vdd / gnd` 会被识别成普通节点，LVS 报"pin 名不匹配"。
- **net 名 `n_` 前缀**：magic 提取出来的内部节点名以 `n_` 开头；netgen 默认对内部网络名容忍（拓扑等价就行），但报告里满屏 `n_*` 看起来吓人，不是错。
- **`run_standard_drc.py` 找不到 `.mag`**：脚本默认在当前目录或 `mag/` 子目录找。在 `~/work/foo/` 跑、但 .mag 文件叫 `bar.mag` 要写 `run_standard_drc.py bar`（**不要带 .mag 后缀**）。
- **密度检查需要 GDS，不接受 .mag**：先 magic 里 `gds write foo.gds` 导出，再 `check_density.py foo.gds`。
- **天线规则真违例时**：在长 trace 中插 antenna diode（`sky130_fd_sc_hd__diode_2`）或者中间用 via 切到上层再切回来——选哪个 magic 不会替你决定。
- **顶层 cell 没打 port**：magic 里要 `port make` 把外面接出来的 label 升格为 port，否则 LVS 看不到引脚。在 magic 命令行选中 label，然后 `port make`。
- **KLayout LVS 的 `--run_mode=flat` 慢但稳**：大设计走 deep；小 cell（< 几百器件）出现奇怪不匹配时切 flat 重跑，能排除是层次比对的 bug。

## 下一步

笔记 07：**标准单元库 sky130_fd_sc_hd 简介**——之前一直在讲 `fd_pr`（模拟器件），最后这一章顺便看看数字标准单元库长什么样、模拟设计什么时候会用到它（buffer / 时钟驱动 / 电平转换 / dummy diode 等）。
