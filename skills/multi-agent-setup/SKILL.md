---
name: multi-agent-setup
description: 自动化创建、配置并在同一个 Gateway 下部署新的 OpenClaw Agent。使用本技能可以快速配置多 Agent 环境（包括分配 Workspace、编写基础提示词、绑定 Telegram 频道、设置群组权限及配置 Sub-agent 调用权限）。
---

# Multi-Agent Setup (多 Agent 部署)

当你需要在一个 OpenClaw 实例下部署多个分工明确的 Agent，并实现协作或加入同一个 Telegram 群组时，使用本技能作为标准化操作流程（SOP）。

## 核心前提
- 本流程基于 **单个 Gateway + 多个 Agent** 架构（官方推荐的 Hub and Spoke 模式）。
- 所有涉及修改 `openclaw.json` (通过 `openclaw config set`) 的操作，完成后都需要执行 `openclaw gateway restart` 以生效。

## ⚠️ Gateway 重启安全规则（极重要）
在执行 `openclaw gateway restart` 前，**必须严格遵守以下流程**：
1. **检查活跃任务**：使用 `sessions_list` 确认是否有其他 Agent 正在执行任务。若有，必须等待其完成或获得用户明确授权中断。
2. **验证配置有效性**：运行 `openclaw config validate` 确认配置合法。
3. **用户最终授权**：向用户报告检查结果，并明确询问是否可以重启。**严禁在未获得许可的情况下擅自重启。**

---

## 部署流程

### Step 1: 在 Gateway 注册新 Agent
向 `agents.list` 中追加新 Agent 的基本信息，包括 `id`、`name`、`workspace`（工作目录）和默认 `model`。

**命令示例（逐个添加，不要一次性设置整个数组）：**
```bash
# 先确认当前 agents.list 长度，确定下一个可用索引
openclaw config get agents.list

# 逐个追加（每次只设一个索引）
openclaw config set "agents.list[1]" "{'id': 'new-agent', 'name': 'New Agent', 'workspace': 'F:\\Projects\\MyProject\\new-agent', 'model': 'github-copilot/gpt-5.4'}"
```

> **⚠️ 注意：** `openclaw config set` 对数组索引的操作是逐个追加的。不要试图在列表为空时直接设置高索引号（如 `[3]`），应该从 `[0]` 开始逐个添加。

### Step 2: 初始化工作区
在新 Agent 的 `workspace` 目录下创建必要的初始文件，赋予其基础身份和目标。
1. 创建目录（如果不存在）。
2. 创建并写入 `SOUL.md`（定义核心职责和行为准则）。
3. 创建并写入 `IDENTITY.md`（定义名称、角色、表情符号等元数据）。

> **提示：** 如果子 Agent 的工作区已存在（如从另一台机器迁移过来），跳过此步。

### Step 3: 配置共享技能（⭐ 推荐方案：skills.load.extraDirs）
**不要复制技能文件！** 使用 `skills.load.extraDirs` 让所有 Agent 共享主工作区的技能，只维护一份文件。

```bash
# 全局配置，所有 Agent 自动可见主工作区的技能
openclaw config set skills.load.extraDirs '["F:\\Projects\\MyProject\\skills"]'
```

**原理：**
- OpenClaw 技能加载优先级：`<workspace>/skills`（最高）→ `~/.openclaw/skills` → bundled（最低）
- `skills.load.extraDirs` 是全局配置，额外加载指定目录的技能（最低优先级）
- 每个 Agent 自己工作区下的 `skills/` 仍然有效（最高优先级），可用于放置专属技能

**各 Agent 技能来源：**
| 来源 | 优先级 | 可见性 |
|------|--------|--------|
| `<agent-workspace>/skills/` | 最高 | 仅该 Agent |
| `~/.openclaw/skills/` | 中 | 所有 Agent |
| `skills.load.extraDirs` | 最低 | 所有 Agent |
| bundled | 最低 | 所有 Agent |

> **⚠️ 重要说明：** `skills.load.extraDirs` 只支持全局配置，不支持 per-agent 覆盖。`agents.defaults.subagents` 也不支持 `allowAgents` key。

### Step 4: 配置主 Agent 的 sub-agent 调度权限
主 Agent（`main`）默认不在 `agents.list` 中，因此默认只能 spawn 自己。需要**显式将 main 加入 `agents.list[0]`** 并配置 `subagents.allowAgents`。

```bash
# 将 main 加入 agents.list[0]，并配置允许调度的子 Agent
openclaw config set "agents.list[0]" "{'id': 'main', 'name': '虎子', 'workspace': 'F:\\Projects\\MyProject', 'subagents': {'allowAgents': ['info', 'marketing', 'stock', 'ta-dev']}}"
```

> **⚠️ 常见错误：**
> - ❌ 试图在 `agents.defaults.subagents` 下设置 `allowAgents` → 会报 `Unrecognized key`
> - ❌ 在第一个子 Agent（如 `info`）的 `subagents` 上设置 `allowAgents` → 这是给 info 设置的，不是给 main 的
> - ✅ 必须显式创建 `agents.list[0]` 为 main agent

### Step 5: 配置底层目录（可选，通常不需要）
OpenClaw 在首次与 Agent 交互时会自动创建 `~\.openclaw\agents\<id>\` 目录。**通常不需要手动创建。** 如果遇到问题再手动补。

```bash
# 仅在出问题时手动创建
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.openclaw\agents\new-agent-id\agent"
```

### Step 6: 绑定通讯频道（以 Telegram 为例，可选）
给新 Agent 绑定独立的 Bot Token，或选择**方案 B：不绑定独立频道**，仅通过 `sessions_spawn` 后台调度。

**方案 A（独立 Bot）：**
```bash
# 1. 在 @BotFather 创建新 Bot，获取 Token
# 2. 注册 account
openclaw config set channels.telegram.accounts.new-agent-id "{'botToken': 'YOUR_BOT_TOKEN', 'dmPolicy': 'pairing', 'groupPolicy': 'open', 'groupAllowFrom': ['<你的Telegram用户ID>'], 'streaming': 'partial'}"

# 3. 绑定路由
openclaw config set "bindings[0]" "{'agentId': 'new-agent-id', 'match': {'channel': 'telegram', 'accountId': 'new-agent-id'}}"
```

**方案 B（仅后台调度，推荐用于专用 Agent）：**
不绑定 Telegram 频道，主 Agent 通过 `sessions_spawn(agentId='new-agent-id', task='...')` 后台调度。子 Agent 完成后自动推送结果到主 Agent 的会话。

### Step 7: 首次启动引导（BOOTSTRAP.md）

新 Agent 的工作区初始化后，可以创建一个 `BOOTSTRAP.md` 文件来引导它首次启动时的自我发现。

**在新 Agent 的 workspace 下创建 `BOOTSTRAP.md`：**
```markdown
# BOOTSTRAP.md - Your Birth Certificate

This file exists to guide your first session.

1. Read SOUL.md — understand who you are
2. Read IDENTITY.md — know your name, role, emoji
3. Read USER.md — know who you're helping
4. Customize SOUL.md if needed — add your own voice
5. Read AGENTS.md (in parent workspace) — understand workspace rules
6. Once you've figured yourself out, DELETE this file
7. After deleting, greet your human in your configured persona
```

**原理：**
- AGENTS.md 约定：如果 `BOOTSTRAP.md` 存在，Agent 首次启动时必须遵循它，完成后删除
- 这是一个一次性引导机制，类似"出生证明"——只存在第一次启动时
- BOOTSTRAP.md 不存在时，Agent 按正常流程启动（读 SOUL.md + USER.md 等）

**什么时候需要 BOOTSTRAP.md：**
- ✅ 全新创建的 Agent，需要自我发现流程
- ✅ 从模板/其他机器迁移过来的 Agent，身份可能变化
- ❌ 已经运行过的 Agent，不需要

### Step 8: 最终重启与验证
1. 检查活跃任务 → 验证配置 → **向用户申请重启许可** → 获得许可后执行 `openclaw gateway restart`
2. 验证 Agent 注册：`openclaw agents list`
3. 验证技能共享：`sessions_spawn` 唤醒子 Agent，让其列出能加载的技能数量
4. 完成链路测试

---

## 实战经验（来源：2026-03-22 多 Agent 部署）

### 经验教训清单
1. **`config set` 数组操作是逐个追加的**：不要一次设多个索引，每次只设一个，确认成功后再设下一个
2. **`agents.defaults.subagents` 不支持 `allowAgents`**：会报 `Unrecognized key`，只能在 `agents.list[].subagents` 下配置
3. **主 Agent 必须显式加入 `agents.list`**：否则 `allowAgents` 无处配置
4. **底层目录（`~\.openclaw\agents\<id>\`）会自动创建**：不需要手动创建
5. **`skills.load.extraDirs` 是全局的**：无法 per-agent 配置，但作为共享技能方案足够好用
6. **子 Agent 只注入 `AGENTS.md` + `TOOLS.md`**：不含 `SOUL.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`
7. **Workspace-state.json 会自动迁移**：从另一台电脑复制过来的工作区，`workspace-state.json` 保留了 onboarding 状态
8. **禁止擅自重启 Gateway**：即使是合理的重启操作，也必须先获得用户明确授权
9. **`skills` 数组是白名单过滤器**：`agents.list[].skills` 中列出的技能名会限制该 Agent 只能加载这些技能。**清空为 `[]` 或删除该字段后，Agent 不再过滤，通过 `skills.load.extraDirs` 和默认路径自动发现全部技能。**子 Agent 如果不需要限定技能范围，建议不写 `skills` 字段，让它们通过 `extraDirs` 共享主工作区的完整技能目录。
10. **技能目录名必须与 SKILL.md 中的 `slug` 一致**：`extraDirs` 扫描时按目录名注册技能，`SKILL.md` 中的 `slug` 字段应与目录名匹配，否则可能出现目录存在但技能无法加载的情况。（已知不一致案例：`xiaohongshu-ops-skill` vs slug `xiaohongshu-ops`）
