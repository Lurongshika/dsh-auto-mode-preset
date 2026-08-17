# Adaptive Mode (自适应模式) — DSH Agent Preset

English | [中文](README.zh.md)

A [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) agent preset that mounts **every mode's real capability up front** — `standard`'s full native tool set, the Code Mode SDK (`run_code`), and the live cordis toolset — and, **before each task and whenever the work changes**, briefly analyzes the task and picks the most appropriate discipline from `standard` / `minimal` / `code` / `cordis`, declares the choice, and follows it.

Because all four capabilities are already mounted, switching mid-task is just choosing different tools — no re-mount, no interruption.

## How it works

The persona turns each new task into a brief routing decision. Before acting, classify the task and pick a discipline in this order:

| Order | Discipline | When to choose | Capability |
|---|---|---|---|
| 1 | **cordis** | Authoring / debugging / reviewing the harness itself: a plugin, a preset, `cordis.patch.yml`, or a skill | Live runtime inspection + plugin mount/dispose (`tool-cordis`), plus the bundled authoring skills |
| 2 | **minimal** | One shell command or one small file edit | Use only `bash` and `str_replace_editor` |
| 3 | **code** | Multi-step, deterministic, scriptable work (batch rewrite / generation / data movement) | Code Mode SDK (`run_code`): write one TypeScript program and run it once |
| 4 | **standard** | Everything else (cross-file browse + edit + run + search + plan) | Full native tool set, used normally |

Before acting, print a one-line declaration:

```
[preset: <name>] <one-line reason> — <next action>
```

Example: `[preset: minimal] single-file tweak, one edit — locate the file, then edit.`

When unsure, pick `standard`. Do not re-run the routing for every sub-step of one task. If you misjudged, upgrade to `standard` and say so in one line. **Never stop to wait for a manual switch.**

The declaration and every reply follow the language the user writes in (the discipline names `standard` / `minimal` / `code` / `cordis` stay as identifiers; only the reason and next action are translated).

## Directory layout

```
adaptive/
  preset.yml           # display name / description / order (order: 0 sorts first)
  agent.cordis.yml     # composition: standard + Code Mode SDK (both) + cordis toolset + authoring skills
  skills/
    editing-cordis-compositions/SKILL.md
    cordis-plugin-development/SKILL.md
```

## Install / configure

### 1. Install locally

```sh
mkdir -p ~/.dsh/.agent-presets
git clone <your-repo-url> ~/.dsh/.agent-presets/adaptive
# or: copy this repository's contents into ~/.dsh/.agent-presets/adaptive/
```

The directory name is the preset id (here `adaptive`) and must match `[a-z0-9][a-z0-9-]*`.

### 2. Select the preset

**Option A — Web UI**: Settings → General → Agent Preset, choose "自适应模式 / Adaptive Mode".

**Option B — set as default** (new sessions use it automatically): add to `~/.dsh/settings.yaml`:

```yaml
agent-presets:
  default: adaptive
```

### 3. Verify

Create a new session and send a task; if the persona is active, a `[preset: ...]` declaration line appears before it starts working.

## Technical notes

- A preset's tool schema is fixed at mount time: `dsh-agent-presets` composes it **once at session creation** and it cannot be re-mounted mid-conversation (swapping tools mid-log would leave logged calls the new composition cannot make).
- Adaptive Mode works around that by mounting **everything up front**: `code` (Code Mode SDK, `mode: both`) and `cordis` (`tool-cordis`) are real capabilities, so switching mid-task is choosing tools, not re-mounting.
- `minimal` is the one behavioral discipline — a preset cannot hide already-mounted tools, so it is "use only bash + str_replace_editor".
- `tool-cordis` is a **trust boundary, not a sandbox**: a session here can inspect the live runtime and mount/dispose plugins, i.e. it is shell-equivalent. Remove that row if you do not want self-modification.
- To switch the *mounted composition* based on the first message **before a session starts**, you would write a host plugin that takes over `AgentPresets.mount(agentCtx, id)` (the agent factory's `setup` hook) — an invasive change to the web session-creation path, outside this preset's scope.

## Customize

Copy `agent.cordis.yml` and edit the `persona` row's `text`. `{{model}}` and `{{cwd}}` resolve at mount time to the current model and working directory.
