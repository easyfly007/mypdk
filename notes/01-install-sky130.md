# 01 — 安装 sky130 PDK（Volare 方式）

## 目标

把 SkyWater 130nm 开源 PDK 装到本地，让 magic / xschem / ngspice / klayout 都能用上。

## 三个名词先理清楚

| 名词 | 是什么 |
| --- | --- |
| **SkyWater SKY130** | 由 SkyWater Technology 公布的 130 nm CMOS 工艺。原始仓库是 `google/skywater-pdk`，只包含"原始数据"（器件模型、GDS、文档），并不能直接被 EDA 工具使用。 |
| **open_pdks** | RTimothy Edwards 维护的"组装脚本"集合。它读取原始 SkyWater 数据，**生成**各 EDA 工具（magic / netgen / klayout / xschem / ngspice / openlane …）需要的配置文件、技术文件、cell 视图。产物就是大家常说的 `sky130A` 和 `sky130B`。 |
| **Volare** | Efabless 出的一个 Python 工具，专门用来**下载已经编译好的 open_pdks 产物**，相当于 PDK 的 "pip"。一条命令下来一个完整可用的 sky130 安装。 |

理解上的层次：

```
SkyWater 原始数据 (skywater-pdk)
        │
        ▼  open_pdks Makefile + 大量脚本
   编译/组装
        │
        ▼
  sky130A / sky130B 产物（GB 级）
        │
        ▼  Volare 把某一版打包到 CDN，用户拉取
   本地 ~/.volare/...
```

两个变体：

- **sky130A** — 含 MiM 电容、HV 器件等"完整"工艺选项，几乎所有教程默认用它。
- **sky130B** — sky130A 加上 ReRAM 选项；如果不做 ReRAM，用 sky130A 就行。

## 当前机器实际装了什么

```bash
$ volare --version
Volare v0.20.6 ©2022-2025 Efabless Corporation

$ volare path
/home/yifei/.volare

$ volare ls
["0fe599b2afb6708d281543108caf8310912f54af"]

$ cat ~/.volare/volare/sky130/current
0fe599b2afb6708d281543108caf8310912f54af

$ ls -la ~/.volare/sky130A
~/.volare/sky130A -> volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A
```

要点：

- Volare 用 **open_pdks 的 git commit hash** 来标识 PDK 版本，本机装的是 `0fe599b2…`。
- 实际数据放在 `~/.volare/volare/sky130/versions/<hash>/{sky130A,sky130B}/`，每个版本约 **2.1 GB**。
- `~/.volare/sky130A` 是符号链接，指向"当前激活版本"。切换版本时 Volare 改的就是这个链接。
- `~/.volare/volare/sky130/current` 是一个文本文件，记录当前激活的 hash。

## 从零安装的步骤

### 1. 准备依赖

Volare 是纯 Python 包，本身没什么依赖：

```bash
python3 --version    # 需要 ≥ 3.8
pip --version
```

EDA 工具（magic / xschem / ngspice / klayout）需要单独装，与 PDK 无关。可以用 apt、源码编译、或 Nix。

### 2. 安装 Volare

```bash
pip install --user volare
# 或者用 pipx，避免污染系统 Python：
# pipx install volare
```

装完 `volare` 进入 `~/.local/bin/`，确保它在 `$PATH` 里。

### 3. 下载并启用 sky130

```bash
# 列出远端可用版本（最新通常排在前面）
volare ls-remote --pdk sky130

# 下载并激活某个版本
volare enable <commit-hash>

# 或者直接拉最新（不带参数）
# volare enable
```

`enable` 做的事：

1. 从 Efabless 的 CDN 下载该版本的 tarball（zstd 压缩，~几百 MB）；
2. 解压到 `~/.volare/volare/sky130/versions/<hash>/`；
3. 更新 `current` 文件；
4. 创建 / 修改 `~/.volare/sky130A`、`~/.volare/sky130B` 符号链接。

如果只想下载不激活，用 `volare fetch`。

### 4. 验证

```bash
volare output                       # 打印当前激活版本
ls ~/.volare/sky130A/libs.tech       # 应能看到 magic, xschem, ngspice, klayout 等子目录
ls ~/.volare/sky130A/libs.ref        # 应能看到 sky130_fd_pr, sky130_fd_sc_hd 等
```

### 5. 设置环境变量

各工具是通过环境变量定位 PDK 的。**Volare 本身不动 shell**，需要自己在 `~/.bashrc` 或 `~/.zshrc` 里加：

```bash
export PDK_ROOT=$HOME/.volare
export PDK=sky130A
```

- `PDK_ROOT` 指向"Volare 根目录"（不是某个具体版本，因为符号链接已经在这层）。`sky130A.magicrc` 之类的脚本读 `$PDK_ROOT/sky130A/...`。
- `PDK` 选择变体，绝大多数情况设成 `sky130A`。

> 本机当前 `~/.bashrc` / `~/.zshrc` 里都还**没设**这两个变量。后续做实验前再加。

## 常用 Volare 命令速查

| 命令 | 作用 |
| --- | --- |
| `volare ls` | 列本地已下载版本 |
| `volare ls-remote --pdk sky130` | 列远端可用版本 |
| `volare fetch <hash>` | 下载但不激活 |
| `volare enable <hash>` | 下载（如需）并激活 |
| `volare output` | 打印当前激活版本 |
| `volare path` | 打印 Volare 根目录（默认 `~/.volare`） |
| `volare rm <hash>` | 删除某个版本 |
| `volare prune` | 删除除当前激活外的所有版本 |
| `volare build` | 从 open_pdks 源码自己编译（慢，几小时；一般不用） |

## 备选安装方式（仅了解，不推荐）

1. **手动 git clone open_pdks 然后 make** — 完全可控但需要 SkyWater 原始仓库 + 长时间编译，磁盘占用大。Volare 的 `build` 子命令本质上就是这套流程的封装。
2. **OpenLane Docker 镜像内置** — 如果只想跑数字流程，OpenLane 容器里已经带 sky130，不用单独装。
3. **Nix 包** — `nixpkgs` 里有 sky130A 的衍生包，干净但灵活性低。

模拟 / 版图学习场景，**Volare 是首选**：快、版本可控、跟主流教程一致。

## 几个容易踩的坑

- **PDK_ROOT 设错层级**：很多老教程让设 `PDK_ROOT=$HOME/.volare/volare/sky130/versions/<hash>`，那是 Volare 内部目录。新版应直接设到 `~/.volare`，让符号链接做版本切换。
- **同时设了 PDK_ROOT 和 OPEN_PDKS_ROOT**：某些 OpenLane 教程会让设 `OPEN_PDKS_ROOT`，跟 Volare 的约定冲突，**两个不要混用**。
- **2 GB 一份不是错**：每个版本完整 2.1 GB，多版本会很快吃掉磁盘。学完后用 `volare prune` 清。
- **变更版本后 magic / xschem 缓存**：xschem 会在用户目录缓存符号路径，换版本若有报错先清 `~/.xschem` 下的旧缓存。

## 下一步

- 笔记 02：进入 `~/.volare/sky130A` 内部，把 `libs.tech` 和 `libs.ref` 整个目录树梳一遍，搞清楚每个文件是给谁、做什么用的。
