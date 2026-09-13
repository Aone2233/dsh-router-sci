# router-sci — 科研计算 Agent 预设

面向 **核工程粉体 / 多相流仿真** 的 DeepSeek Harness (dsh) agent preset。

在 `router-standard` 的渐进披露路由之上，替换 persona 为中文科研助手，
并把 **WSL ↔ Windows 双环境工具路由**、**科研写作纪律**、**跨会话续接**
写进系统提示，让 agent 在 LAMMPS / Fluent DPM / LaTeX 混合工作流里
按项目约定行事。

## 它解决什么问题

跑仿真研究时的典型摩擦：

- **环境分裂** — LAMMPS 和 Python 在 WSL，Fluent 在 Windows，agent 经常
  在错误的一侧执行，或者用 9P 路径（`\\wsl.localhost\...`）跑重 IO 导致极慢
- **数值来源失守** — 颗粒物性（密度、弹性模量、表面能）被随手编造
- **写作纪律松散** — 报告结构每次都不一样，图表风格不统一
- **跨会话断片** — 新会话不知道上一轮做到哪、哪些是未决项

这个预设把上述约定固化成 persona，并配一套渐进披露的阶段机制。

## 核心内容

### 1. 双环境工具路由

```
LAMMPS / Python / LaTeX  →  WSL 侧（经 git bash 执行 wsl -d <DISTRO>）
Fluent                   →  Windows 侧（批跑脚本）
重 IO（dump / .trn）      →  必须在 WSL 原生目录内
9P 路径（UNC / 符号链接）  →  只做轻量读写与 dsh 交互
```

### 2. 领域背景注入

- **微观**：LAMMPS 分子动力学模拟颗粒碰撞，恢复系数 `e(v)`，附着/反弹判据，
  附着临界速度幂律模型 `v_crit = A·d_p^B`
- **宏观**：Fluent DPM 气固两相流沉积，UDF 壁面边界条件，
  P2P/P2W 恢复系数 profile
- **仿真-理论对照**：物性参数只取文献或报告值，结论须相互印证

### 3. 科研写作纪律

报告七段结构（时间线 / 问题发现 / 参数空间 / 敏感性分析 / 物理机制 /
工程意义 / 后续计划），图表遵循统一出版风格（STIX 数学字体、300 dpi、11pt）。

### 4. 跨会话续接

开工前读 `task_plan.md` / `findings.md` / `progress.md`，完成后回写。

## 安装

### 前置

- [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）
- 无需额外安装——本仓库自带全部 bootstrap 与执行器文件
  （见下方「与上游的关系」）。它们直接以相对路径被 `agent.cordis.yml` 引用，
  不经过 npm 安装。

### 步骤

```bash
# 1. 克隆
git clone https://github.com/<你的账号>/dsh-router-sci.git
cd dsh-router-sci

# 2. 复制到 dsh 的 agent-presets 目录（dsh 只扫一级子目录）
#    Windows:
xcopy /E /I . "%USERPROFILE%\.dsh\.agent-presets\router-sci"
#    Linux / macOS / WSL:
cp -r . ~/.dsh/.agent-presets/router-sci

# 3. 填占位符（见下）
# 4. 生效（新会话挂载新一代）
```

## 必填占位符

本仓库的 `agent.cordis.yml` 是**模板**，persona 中的机器相关路径以占位符给出，
使用前请替换为你的真实环境：

| 占位符 | 含义 | 示例 |
|---|---|---|
| `<PROJECT_NAME>` | 项目名 | `MyThesis` |
| `<WSL_DISTRO>` | WSL 发行版名 | `Ubuntu` |
| `<WSL_PROJECT_ROOT>` | WSL 侧项目根（绝对路径） | `/home/you/MyThesis` |
| `<WIN_PROJECT_LINK>` | Windows 侧符号链接路径 | `C:\MyThesis` |
| `<ANSYS_INSTALL>` | ANSYS 安装路径 | `D:\ANSYS2024R2` |

> `{{model}}` 与 `{{cwd}}` 由 dsh 运行时解析，**保持原样不要替换**。

## 与上游的关系

本预设构建在 [dsh-routing-suite](https://github.com/yjh051108/dsh-routing-suite)
的 `router-standard` 之上，沿用其渐进披露路由机制与 Git Bash shell seam。
下列文件来自该上游项目（MIT）：

```
gitbash-executor.mjs
router-bootstrap.mjs
router-bootstrap-v34.mjs
router-bootstrap-v34.selftest.mjs
router-core.mjs
router-core-v34.mjs
```

本仓库的原创部分为：

```
agent.cordis.yml   （persona、工具路由约定、科研写作纪律 —— 该文件整体
                    改编自 DeepSeek Harness Standard 预设的 composition，
                    其余插件行沿用上游；只有上述内容为原创）
preset.yml         （预设元数据）
README.md / NOTICE （文档与许可说明）
```

请注意 `agent.cordis.yml` **同时**包含派生的插件行与原创的 persona，
两部分的归属不同，详见 [NOTICE](NOTICE)。

若上游发布新版本，可自行同步上述代码文件。

注意 `router-core.mjs`（无版本别名）与 `router-core-v34.mjs`（版本快照）
在本仓库中内容相同，且都包含上游 `39ee0a0` 引入的 `sessionEvents` 辅助。
但上游仓库自身的别名文件尚未跟进该修复（其 `router-core.mjs` 仍是旧版，
缺 `sessionEvents`），因此**从上游同步时请以 `-v34` 快照为准**，
不要用上游的别名覆盖本仓库的别名，以免两个 core 文件出现版本漂移。

## 自定义

改 persona 就直接编辑 `agent.cordis.yml` 里的 `persona.config.prefix`。
阶段划分与工具解锁顺序在 `router-bootstrap.mjs` 的 `STAGES` 常量中。

改完需要绕 ESM 缓存才能让新会话加载新代码：

```bash
# 递增 agent.cordis.yml 里 router-bootstrap-v34.mjs?v=N 的 N
# 然后对新会话生效
```

## 已知限制

- **技能目录不可见**：`router` 系列预设把 `skill` 工具排除在工具面之外
  （上游设计取舍：控制面工具会扰动推理轨迹）。本预设沿用该取舍，
  因此 `~/.agents/skills` 下的技能在会话中不可用。
- **POSIX bash**：`agent.cordis.yml` 已包含 `tool-bash-posix` 行以支持
  在 Linux/macOS/WSL 侧直接运行 dsh；Windows 侧走 `gitbash-shell` 组。
- **依赖 dsh 版本**：persona schema 随 dsh 演进（`text` → `prefix`）。
  本仓库已适配 dsh `0.1.5-rc.1` 的 `@deepseek-ai/dsh-persona` schema。

## License

MIT — 见 [LICENSE](LICENSE)。
上游衍生内容的版权归原作者，见 [NOTICE](NOTICE)。
