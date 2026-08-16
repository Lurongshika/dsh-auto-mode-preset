# Auto Mode（自动模式）— DSH Agent Preset

[English](README.md) | 中文

一个 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 的 Agent Preset：在**每次处理新任务前**先简短分析任务，自动从 `standard` / `minimal` / `code` / `cordis` 四种工作纪律中选出最合适的一种，声明选择并执行。

它基于 `standard`（完整工具集），并额外打包了 `cordis` 预设的两个创作技能，让「cordis 纪律」真正可用（而非只是口头建议）。

## 工作原理

Persona 把每个新任务变成一次简短的路由决策。动手前先分类，再按下面的判定顺序选择纪律：

| 判定顺序 | 纪律 | 何时选用 | 行为约束 |
|---|---|---|---|
| 1 | **cordis** | 创作 / 调试 / 审查 harness 自身：插件、preset、`cordis.patch.yml`、skill | 先加载 `editing-cordis-compositions`（或 `cordis-plugin-development`），遵守 host 平面 vs agent 平面 |
| 2 | **minimal** | 一条 shell 命令或一处小文件编辑 | 只用 `bash` 与 `str_replace_editor` |
| 3 | **code** | 多步、确定性、可脚本化（批量改写 / 生成 / 数据搬运） | 优先写一个 TypeScript/Node 程序一次运行，减少工具往返 |
| 4 | **standard** | 其余一切（跨文件浏览 + 编辑 + 运行 + 检索 + 规划） | 完整工具集正常使用 |

动手前先输出一行声明：

```
[preset: <name>] <一句话理由> — <下一步行动>
```

例：`[preset: minimal] 单文件小改，只需编辑一处 — 先定位目标文件再改。`

不确定时选 `standard`；不要为每个子步骤重复触发路由；若误判，升级到 `standard` 并一句话说明；**绝不停下等待手动切换**。

声明与回复会跟随用户所使用的语言（纪律名 `standard` / `minimal` / `code` / `cordis` 保持为标识符，只翻译理由与下一步）。

## 目录结构

```
auto/
  preset.yml           # 显示名 / 描述 / 排序（order: 0 排最前）
  agent.cordis.yml     # 组合：standard 完整工具集 + 自动路由 persona + 创作技能
  skills/
    editing-cordis-compositions/SKILL.md
    cordis-plugin-development/SKILL.md
```

## 安装 / 配置方法

### 1. 安装到本机

```sh
mkdir -p ~/.dsh/.agent-presets
git clone <你的仓库地址> ~/.dsh/.agent-presets/auto
# 或者：把本仓库全部内容复制到 ~/.dsh/.agent-presets/auto/
```

目录名就是 preset id（这里是 `auto`），必须匹配 `[a-z0-9][a-z0-9-]*`。

### 2. 选择该 preset

**方式 A — Web UI**：设置 → 通用 → Agent Preset，选择「自动模式（Auto Mode）」。

**方式 B — 设为默认**（新会话自动使用）：在 `~/.dsh/settings.yaml` 增加：

```yaml
agent-presets:
  default: auto
```

### 3. 验证

新建会话后发一条任务，若 persona 生效，动手前会出现一行 `[preset: ...]` 声明。

## 技术说明

- 一个 preset 由 `dsh-agent-presets` 在**会话创建时挂载一次**；会话进行中无法切换。
- 因此「自动选择」是**行为纪律**（用哪些工具、如何组织工作），不是运行时的插件重挂载。`minimal` 靠「只用 bash + str_replace_editor」来近似，`code` 靠「写一个程序一次跑」来近似。
- `cordis` 纪律是真实能力：本预设随附两个创作技能，并遵守 host / agent 两平面归属规则。
- 若要在**会话创建前**按首条消息真正切换 preset，需要写一个 host 插件接管 `AgentPresets.mount(agentCtx, id)`（会话工厂的 `setup` 钩子），这属于对 Web 会话创建路径的侵入式改动，不在本预设范围内。

## 自定义

复制 `agent.cordis.yml`，改 `persona` 行的 `text` 即可。`{{model}}` 与 `{{cwd}}` 会在挂载时解析为当前模型与工作目录。
