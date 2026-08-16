# Auto Mode (自动模式) — DSH Agent Preset

English | [中文](README.zh.md)

A [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) agent preset that, **before every new task**, briefly analyzes the task and automatically picks the most appropriate working discipline from `standard` / `minimal` / `code` / `cordis`, declares the choice, and follows it.

It builds on `standard` (the full tool set) and additionally bundles the two authoring skills from the `cordis` preset, so the `cordis` discipline is a real capability rather than a verbal suggestion.

## How it works

The persona turns each new task into a brief routing decision. Before acting, classify the task and pick a discipline in this order:

| Order | Discipline | When to choose | Behavioral constraint |
|---|---|---|---|
| 1 | **cordis** | Authoring / debugging / reviewing the harness itself: a plugin, a preset, `cordis.patch.yml`, or a skill | Load `editing-cordis-compositions` (or `cordis-plugin-development`) first; respect the host-plane vs agent-plane split |
| 2 | **minimal** | One shell command or one small file edit | Use only `bash` and `str_replace_editor` |
| 3 | **code** | Multi-step, deterministic, scriptable work (batch rewrite / generation / data movement) | Prefer writing one TypeScript/Node program and running it once, cutting tool round-trips |
| 4 | **standard** | Everything else (cross-file browse + edit + run + search + plan) | Use the full tool set normally |

Before acting, print a one-line declaration:

```
[preset: <name>] <one-line reason> — <next action>
```

Example: `[preset: minimal] single-file tweak, one edit — locate the file, then edit.`

When unsure, pick `standard`. Do not re-run the routing for every sub-step of one task. If you misjudged, upgrade to `standard` and say so in one line. **Never stop to wait for a manual switch.**

The declaration and every reply follow the language the user writes in (the discipline names `standard` / `minimal` / `code` / `cordis` stay as identifiers; only the reason and next action are translated).

## Directory layout

```
auto/
  preset.yml           # display name / description / order (order: 0 sorts first)
  agent.cordis.yml     # composition: standard's full tool set + auto-routing persona + authoring skills
  skills/
    editing-cordis-compositions/SKILL.md
    cordis-plugin-development/SKILL.md
```

## Install / configure

### 1. Install locally

```sh
mkdir -p ~/.dsh/.agent-presets
git clone <your-repo-url> ~/.dsh/.agent-presets/auto
# or: copy this repository's contents into ~/.dsh/.agent-presets/auto/
```

The directory name is the preset id (here `auto`) and must match `[a-z0-9][a-z0-9-]*`.

### 2. Select the preset

**Option A — Web UI**: Settings → General → Agent Preset, choose "自动模式（Auto Mode）".

**Option B — set as default** (new sessions use it automatically): add to `~/.dsh/settings.yaml`:

```yaml
agent-presets:
  default: auto
```

### 3. Verify

Create a new session and send a task; if the persona is active, a `[preset: ...]` declaration line appears before it starts working.

## Technical notes

- A preset is mounted **once at session creation** by `dsh-agent-presets`; it cannot be switched mid-conversation.
- "Auto selection" is therefore a **behavioral discipline** (which tools to use, how to structure the work), not a runtime plugin re-mount. `minimal` is approximated by "only use bash + str_replace_editor", `code` by "write one program and run it once".
- The `cordis` discipline is a real capability: this preset ships the two authoring skills and respects the host / agent plane ownership rules.
- To truly switch presets **before session creation** based on the first message, you would write a host plugin that takes over `AgentPresets.mount(agentCtx, id)` (the agent factory's `setup` hook) — an invasive change to the web session-creation path, outside this preset's scope.

## Customize

Copy `agent.cordis.yml` and edit the `persona` row's `text`. `{{model}}` and `{{cwd}}` resolve at mount time to the current model and working directory.
