# mypdk — Claude 工作约定

## 工作流

- **每完成一个步骤的任务**（典型粒度：一篇 `notes/NN-*.md` 或一处明确闭合的修改），**立即 `git commit` 并 `git push`**。不要把多个步骤攒在一起再交。
  - commit message 用中文，遵循已有 commit 风格（首行短摘要 + 空行 + 1–2 句"为什么/做了什么"）。
  - push 到 `origin/main`；如果 upstream 未设，用 `git push -u origin main`。
- 写笔记同时更新根 `README.md` 目录区，让其与 `notes/` 同步（已完成章节移出"后续会扩展"列表）。

## 笔记风格

参考 `notes/01-install-sky130.md` 和 `notes/02-pdk-structure.md`：

- 中文为主，专业术语保留英文（PDK / DRC / LVS / corner / mismatch / Vt …）。
- 每篇开头一段"目标"。
- 多用对照表（尤其是命名、视图、电压族这种"分类"内容）。
- 命令、路径、文件名用 ```` ``` ```` 代码块。
- 结尾给"几个容易踩的坑"小结 + "下一步"指引到下一章。

## 用户视角

- 读者是模拟 IC / 版图工程师，MOS / 小信号 / 基本版图已熟，**不需要再讲基础**。
- 最终目标是借助 sky130 学会**基于 PDK 的模拟电路设计全流程**：器件 → 原理图 → 仿真（含 corner / MC） → 版图 → DRC / LVS / PEX → 后仿 → 签核。

## 本机环境关键事实

- sky130A 通过 Volare 装在 `~/.volare/sky130A`（符号链接），open_pdks hash `0fe599b2afb6708d281543108caf8310912f54af`。
- `PDK_ROOT` / `PDK` 当前**未写入 shell 配置**；做实验前再加。
- 已安装 EDA：magic / klayout / xschem / ngspice / netgen / volare。
