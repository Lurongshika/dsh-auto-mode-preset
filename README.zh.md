# Adaptive Mode（自适应模式）— DSH Agent Preset

[English](README.md) | 中文

一个 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 的 Agent Preset：**预先挂载每一种模式的真实能力**——`standard` 的完整原生工具集、Code Mode SDK（`run_code`）、以及实时 cordis 工具集；并在**每次任务开始时、以及任务中工作发生变化时**，简短分析任务，从 `standard` / `minimal` / `code` / `cordis` 中选出最合适的纪律，声明并执行。

因为四种能力都已挂载，任务中切换纪律只是「改用哪组工具」，无需重挂载、不会中断。

## 工作原理

Persona 把每个新任务变成一次简短的路由决策。动手前先分类，再按下面的判定顺序选择纪律：

| 判定顺序 | 纪律 | 何时选用 | 能力 |
|---|---|---|---|
| 1 | **cordis** | 创作 / 调试 / 审查 harness 自身：插件、preset、`cordis.patch.yml`、skill | 实时运行时检查 + 插件挂载/卸载（`tool-cordis`），外加随附的创作技能 |
| 2 | **minimal** | 一条 shell 命令或一处小文件编辑 | 只用 `bash` 与 `str_replace_editor` |
| 3 | **code** | 多步、确定性、可脚本化（批量改写 / 生成 / 数据搬运） | Code Mode SDK（`run_code`）：写一个 TypeScript 程序一次跑完 |
| 4 | **standard** | 其余一切（跨文件浏览 + 编辑 + 运行 + 检索 + 规划） | 完整原生工具集正常使用 |

动手前先输出一行声明：

```
[preset: <name>] <一句话理由> — <下一步行动>
```

例：`[preset: minimal] 单文件小改，只需编辑一处 — 先定位目标文件再改。`

不确定时选 `standard`；不要为每个子步骤重复触发路由；若误判，升级到 `standard` 并一句话说明；**绝不停下等待手动切换**。

声明与回复会跟随用户所使用的语言（纪律名 `standard` / `minimal` / `code` / `cordis` 保持为标识符，只翻译理由与下一步）。

## 目录结构

```
adaptive/
  preset.yml           # 显示名 / 描述 / 排序（order: 0 排最前）
  agent.cordis.yml     # 组合：standard + Code Mode SDK（both）+ cordis 工具集 + 创作技能
  skills/
    editing-cordis-compositions/SKILL.md
    cordis-plugin-development/SKILL.md
```

## 安装 / 配置方法

### 1. 安装到本机

```sh
mkdir -p ~/.dsh/.agent-presets
git clone <你的仓库地址> ~/.dsh/.agent-presets/adaptive
# 或者：把本仓库全部内容复制到 ~/.dsh/.agent-presets/adaptive/
```

目录名就是 preset id（这里是 `adaptive`），必须匹配 `[a-z0-9][a-z0-9-]*`。

### 2. 选择该 preset

**方式 A — Web UI**：设置 → 通用 → Agent Preset，选择「自适应模式 / Adaptive Mode」。

**方式 B — 设为默认**（新会话自动使用）：在 `~/.dsh/settings.yaml` 增加：

```yaml
agent-presets:
  default: adaptive
```

### 3. 验证

新建会话后发一条任务，若 persona 生效，动手前会出现一行 `[preset: ...]` 声明。

## 技术说明

- 一个 preset 的工具 schema 在挂载时即固定：`dsh-agent-presets` 在**会话创建时组合一次**，会话中无法重挂载（中途换工具会让已记录的工具调用在新组合里无法复现）。
- 自适应模式的对策是**把一切都预先挂载**：`code`（Code Mode SDK，`mode: both`）与 `cordis`（`tool-cordis`）是真实能力，所以任务中切换就是「改用哪组工具」，而非重挂载。
- `minimal` 是唯一的「行为纪律」——preset 无法隐藏已挂载的工具，所以它只是「只用 bash + str_replace_editor」。
- `tool-cordis` 是**信任边界，不是沙箱**：使用该预设的会话可以检查实时运行时、挂载/卸载插件，等价于 shell 权限。如不想要自我修改能力，删掉这一行即可。
- 若要在**会话开始前**按首条消息切换「挂载的组合」，需要写一个 host 插件接管 `AgentPresets.mount(agentCtx, id)`（会话工厂的 `setup` 钩子），这属于对 Web 会话创建路径的侵入式改动，不在本预设范围内。

## 自定义

复制 `agent.cordis.yml`，改 `persona` 行的 `text` 即可。`{{model}}` 与 `{{cwd}}` 会在挂载时解析为当前模型与工作目录。
