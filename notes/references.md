# 参考资料（外部）

整理学 sky130 PDK / 一般 PDK / 模拟全流程时可以拉来对照的外部资源。**所有链接在 2026 年 5 月经过一次网络核验**；后续若有失效，按关键词重新搜即可。

## 一、公开可获得的真实工艺 PDK（实硅可流片）

| PDK | 节点 / 工艺 | License | GitHub / 主页 | 备注 |
| --- | --- | --- | --- | --- |
| **SkyWater SKY130** | 130 nm 体硅 CMOS（hybrid 0.18/0.13 µm）| Apache 2.0 | [google/skywater-pdk](https://github.com/google/skywater-pdk) · [RTimothyEdwards/open_pdks](https://github.com/RTimothyEdwards/open_pdks) · [efabless/volare](https://github.com/efabless/volare) | 本仓库的主角；MPW 已多次流片 |
| **GlobalFoundries GF180MCU** | 180 nm bulk CMOS（MCU 工艺）| Apache 2.0 | [google/gf180mcu-pdk](https://github.com/google/gf180mcu-pdk) | Google + GF 2022 开放；适合学高压 / I/O |
| **IHP SG13G2** | 130 nm SiGe BiCMOS（含 HBT）| Apache 2.0 | [IHP-GmbH/IHP-Open-PDK](https://github.com/IHP-GmbH/IHP-Open-PDK) · [文档](https://github.com/IHP-GmbH/IHP-Open-PDK-docs) | 真正可流片，做 RF / 模拟首选；[2025 年 4 月已有公开 tape-out](https://github.com/IHP-GmbH/TO_Apr2025) |

这三个是当前（2026 中）**唯一可以"拿到 GDS、提交流片"的开源 PDK**。Tiny Tapeout 在 Efabless 关停后已转向 [IHP shuttle](https://tinytapeout.com/)。

### 三者怎么挑

- **学模拟 / 版图基础**：SKY130（社区最大、教程最多）。
- **学高压 / 电源管理**：GF180MCU（5V 工艺更接近工业用）。
- **学 RF / BJT / SiGe HBT**：IHP SG13G2（自带 SiGe HBT，是 sky130 没有的）。

## 二、FinFET 工艺的 PDK 现状

**好消息**：开源 FinFET PDK 存在。
**坏消息**：**全部是 _predictive_（学术预测）PDK**——基于公开论文与器件物理参数搭建的模型，**没有任何一家真实代工厂把 FinFET PDK 完全开源**。流片几乎都需要 NDA（TSMC University FinFET Program、Intel 16 等 shuttle 计划）。

### Predictive FinFET PDK（仅供学习 / 仿真，不能流片）

| PDK | 节点 | 维护方 | License | 仓库 |
| --- | --- | --- | --- | --- |
| **ASAP7** | 7 nm FinFET（quad / dual fin）| ASU + ARM | BSD-3-Clause | [The-OpenROAD-Project/asap7](https://github.com/The-OpenROAD-Project/asap7) |
| **ASAP5** | 5 nm FinFET | ASU | BSD-3-Clause | [The-OpenROAD-Project/asap5](https://github.com/The-OpenROAD-Project/asap5) |
| **FreePDK15** | 15 nm FinFET | NCSU | 学术开放 | 论文 + PDK：[Bhanushali & Davis 2015](https://dl.acm.org/doi/10.1145/2717764.2717782) |
| **FreePDK3** | 3 nm（GAA / nanosheet 预测）| NCSU | 学术 | [ncsu-eda/FreePDK3](https://github.com/ncsu-eda/FreePDK3) |
| **ASAP3 / ASU 3nm** | 3 nm 预测（含 GAA 变体）| ASU | 学术 | [asap.asu.edu](https://asap.asu.edu/) |

**理解 "predictive" 的含义**：
- 器件模型来自学术论文（典型是 Predictive Technology Model, PTM），并不对应任何实际硅片测量数据；
- DRC / LVS 规则按公开文献假设，**没有 foundry 背书**；
- 可以做学术研究 / 写论文 / 跑后端工具链（OpenROAD 等都集成），但**不能拿出去流片**。

### 半开放 / 受限访问的真实 FinFET PDK（需 NDA）

| 工艺 | 提供方 | 访问方式 |
| --- | --- | --- |
| TSMC 16FFC | TSMC University FinFET Program（2023 起）| 高校 NDA |
| TSMC 7nm | 同上 | 高校 NDA + Muse Semiconductor 等中介 |
| Intel 16 | Intel Foundry Services | 商业 / 学术 shuttle |

**给学习者的建议**：FinFET 物理（窄沟道效应、量子限制、layout-dependent effects、多 Vt 体系）和器件操作可以通过 ASAP7 学；真要做工业级 FinFET 设计，目前还是只能走商业 PDK 路径。

### FinFET vs FD-SOI（容易混）

- **GF 22FDX** 是 22 nm **FD-SOI**（全耗尽 SOI），平面工艺，*不是 FinFET*。
- 真实工业流程里 28 nm 以下绝大多数是 FinFET（除 22FDX/12FDX）。

## 三、工具官方文档

| 工具 | 主页 / 文档 |
| --- | --- |
| Magic | [opencircuitdesign.com/magic](http://opencircuitdesign.com/magic/) — 顶层文档与教程 |
| Netgen | [opencircuitdesign.com/netgen](http://opencircuitdesign.com/netgen/) |
| Xschem | [xschem.sourceforge.io](https://xschem.sourceforge.io/stefan/xschem_man/) — Stefan Schippers 写的手册，sky130 集成有[专门一篇](https://xschem.sourceforge.io/stefan/xschem_man/tutorial_xschem_sky130.html) |
| Ngspice | [ngspice.sourceforge.io](https://ngspice.sourceforge.io) · [PDF 手册](https://ngspice.sourceforge.io/docs/ngspice-manual.pdf) |
| KLayout | [klayout.de/doc.html](https://www.klayout.de/doc.html) |
| OpenROAD | [theopenroadproject.org](https://theopenroadproject.org/) · [Wikipedia 综述](https://en.wikipedia.org/wiki/OpenROAD_Project) |
| OpenLane 2 | [chipfoundry/openlane2](https://github.com/chipfoundry/openlane2)（替代已废弃的 efabless/OpenLane） |
| LibreLane | OpenLane 的"二代"重写，FOSSi 接管：见 [eenewseurope 报道](https://www.eenewseurope.com/en/an-openlane-successor-for-open-source-chip-design/) |

## 四、综合教程 / 课程

| 资源 | 形式 | 备注 |
| --- | --- | --- |
| [Zero-to-ASIC Course (Matt Venn)](https://zerotoasiccourse.com/) | 付费课程 + 免费 YouTube | 数字 ASIC 入门 + Analog 子课程，sky130 全程 |
| [Tiny Tapeout](https://tinytapeout.com/) | 学生 / 业余流片项目 | 集成全套工具链；当前走 IHP shuttle |
| [UCSC VLSI-DA Chip Tutorials](https://vlsida.github.io/chip-tutorials/) | 网站 | sky130 + ASAP7 多个例子 |
| [StefanSchippers/xschem_sky130](https://github.com/StefanSchippers/xschem_sky130) | GitHub | xschem 的 sky130 符号库源码 + 示例 |
| [bluecmd/learn-sky130](https://github.com/bluecmd/learn-sky130) | GitHub | 个人学习笔记，从画 inverter 开始 |
| [HRITAM-MITRA/Installation-of-open-source-analog-design-tools-with-sky130-PDK](https://github.com/HRITAM-MITRA/Installation-of-open-source-analog-design-tools-with-sky130-PDK) | GitHub | 工具安装+ PDK 配置 |
| [Analog IC Design with Open-Source Tools (Valpo)](https://scholar.valpo.edu/cgi/viewcontent.cgi?article=2230&context=cus) | 论文 / 课程材料 | 学术综述 |
| Stefan Schippers 的 xschem + ngspice 视频 | YouTube | 实操演示 |

## 五、教科书（按学习路径）

### 入门基础

- **Razavi, _Design of Analog CMOS Integrated Circuits_, 2nd ed. (McGraw-Hill, 2017)** — 模拟入门第一本，[出版社页](https://www.mheducation.com/highered/product/Design-of-Analog-CMOS-Integrated-Circuits-Razavi.html) · [TOC PDF](https://img.electronicdesign.com/files/base/ebm/electronicdesign/document/2024/09/66e85910f080fd44de86fbd0-razavi_toc.pdf)
- **Allen & Holberg, _CMOS Analog Circuit Design_, 3rd ed. (Oxford, 2012)** — 与 Razavi 互为补充，例题更工程化
- **Sedra & Smith, _Microelectronic Circuits_** — 电子学基础，PDK 不直接讲但器件部分是必备

### 进阶（系统 / 噪声 / 失配）

- **Gray, Hurst, Lewis & Meyer, _Analysis and Design of Analog Integrated Circuits_, 5th ed. (Wiley, 2009; 6th ed. 2024)** — "工业圣经"，BJT + CMOS 并重，噪声章节最值得
- **Sansen, _Analog Design Essentials_ (Springer, 2006)** — 1500+ 张幻灯片浓缩；[Internet Archive 也有 _Design of Analog Integrated Circuits and Systems_](https://archive.org/details/sansen-design-of-analog-integrated-circuits-and-systems)
- **Johns & Martin, _Analog Integrated Circuit Design_, 2nd ed. (Wiley, 2011)** — 加拿大学派代表作，**ΔΣ ADC / DAC 章节最系统**

### 版图专题

- **Alan Hastings, _The Art of Analog Layout_, 2nd ed. (Pearson, 2005)** — 模拟版图圣经，匹配 / well proximity / antenna / latch-up 都讲透
- **Dan Clein, _CMOS IC Layout: Concepts, Methodologies, and Tools_ (Newnes, 2000)** — 数字 / 模拟混合版图，从规则到工具脚本

### PDK / 器件物理

- **BSIM 模型手册** — UC Berkeley [BSIM Group](https://bsim.berkeley.edu/) 出，所有 binning / 噪声模型方程的权威来源
- **Tsividis & McAndrew, _Operation and Modeling of the MOS Transistor_, 3rd ed. (Oxford, 2011)** — MOS 物理本体；想读懂 BSIM 参数为什么这么取必看
- **Razavi, _Fundamentals of Microelectronics_** — 如果 BJT 部分基础不够，从这本回炉

## 六、必读论文 / 经典文献

| 主题 | 引用 | 为什么读 |
| --- | --- | --- |
| 失配 | M. Pelgrom, A. Duinmaijer, A. Welbers, "Matching properties of MOS transistors," *JSSC* 1989 | sky130 mismatch 模型背后的理论框架 |
| 噪声 | A. Abidi, "High-frequency noise measurements on FETs with small dimensions," *TED* 1986 | 1/f / thermal 噪声 |
| FinFET | C. Hu et al., "FinFET—a self-aligned double-gate MOSFET" 系列 *TED* 1999 起 | FinFET 物理原典 |
| 工艺 corner | E. Sicard 等 *Microelectronics Reliability* 综述 | 工艺 corner / Monte Carlo 方法论 |

ASAP7 / ASAP5 / FreePDK15 的发布论文（[ASAP7 IBM J. R&D 2017](https://pages.hmc.edu/harris/research/asap7.pdf) · [ASAP5 SSE 2022](https://www.sciencedirect.com/science/article/pii/S0026269222001148) · [FreePDK15 ISPD 2015](https://dl.acm.org/doi/10.1145/2717764.2717782)）也都值得通读，能看懂"预测 PDK 是怎么从公开论文拼出来的"。

## 七、社区 / 论坛

| 平台 | 用途 |
| --- | --- |
| [open-source-silicon.dev](https://web.open-source-silicon.dev/) | 开源芯片综合社区，sky130 / IHP / Tiny Tapeout 都聊 |
| [SemiWiki](https://semiwiki.com/) | 工业行情 / 工艺新闻；学不到工具但能了解产业上下文 |
| `#xschem`、`#open_pdks` 标签下的 GitHub Issues | 实际报错最快的解答途径 |
| sky130 / OpenLane Slack（Efabless 关停后需重找入口）| 同上 |

## 八、关键参考的去向（2026 年 5 月）

为了避免过时，列几条**结构性变化**，碰到老文章里提到这些时心里有数：

- **Efabless 已于 2025 年关停**（[Hackster.io 报道](https://www.hackster.io/news/open-source-silicon-project-tiny-tapeout-hits-trouble-as-efabless-shuts-its-doors-9ac7fab1649d)）。它的 chipIgnite shuttle 不再继续。Tiny Tapeout 已切换到 IHP。
- **OpenLane**（v1）已停止主线开发，efabless/OpenLane 仓库归档。继承者：[OpenLane 2](https://github.com/chipfoundry/openlane2) 和 FOSSi 的 LibreLane。
- **Sky130 由 [CHIPS Alliance / Linux Foundation 接管](https://opensource.googleblog.com/2023/11/open-source-pdks-joining-linux-foundation-chips-alliance.html)**，治理权从 Google + SkyWater 转给社区。

## 后续

学习路径建议（结合上述资源 + 本仓库笔记 01–05）：

1. **第 1 周**：装好 sky130 + 工具链，看完 Razavi 第 1–6 章，跟一遍 Zero-to-ASIC 的 inverter 例子。
2. **第 2–3 周**：本仓库 06 / 07 + Hastings _Art of Analog Layout_ Ch1–3，自己画一个 nfet + 跑 DRC / LVS 一遍。
3. **第 4–6 周**：跟 Stefan Schippers 的 xschem-sky130 tutorial，做一个 OTA：原理图 → 仿真（含 corner / MC，对照本仓库 04） → 版图 → DRC / LVS / PEX → 后仿。
4. **想看 FinFET**：装 ASAP7，把同一个 OTA 在 7nm 重做一遍，对比 sky130 上的尺寸 / 偏置差异。
5. **想流片**：等下一轮 IHP shuttle（[Tiny Tapeout](https://tinytapeout.com/) 通告），或者上 OpenROAD 的 chipalliance 项目。
