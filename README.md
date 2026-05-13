# mypdk — sky130 PDK 学习笔记

记录通过开源 sky130 PDK 学习 PDK（Process Design Kit）的过程与总结。

## 视角

模拟 IC / 版图为主，已经熟悉 MOS 器件、小信号、基本版图流程。重点是搞清楚 sky130 这个开源 PDK 怎么组织、各文件给哪个工具用、corner / DRC / LVS 怎么接入。

## 工具栈

机器上已经装好的：

- `magic` — 版图编辑 + 提取 + DRC（`/usr/local/bin/magic`）
- `klayout` — 另一套版图查看 / DRC 引擎（`/usr/bin/klayout`）
- `xschem` — 原理图编辑（`/usr/local/bin/xschem`）
- `ngspice` — 电路仿真（`/usr/bin/ngspice`）
- `netgen` — LVS（在 magic 中调）
- `volare` — sky130 PDK 安装 / 版本管理工具

## 目录

- [notes/](notes/) — 概念笔记
  - [01 — 安装 sky130 PDK（Volare 方式）](notes/01-install-sky130.md)
  - [02 — PDK 文件结构地图（libs.tech vs libs.ref）](notes/02-pdk-structure.md)

后续会扩展：

- 03 — 器件库 sky130_fd_pr：MOS、电阻、电容、二极管
- 04 — 仿真模型与 corner 体系
- 05 — magicrc / xschemrc 入口配置
- 06 — DRC / LVS 流程实战
- 07 — 标准单元库 sky130_fd_sc_hd 简介
