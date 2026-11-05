---
name: discord-channel-config
description: OpenClaw Discord 频道配置全流程指南：从创建 Discord Bot 到接入 OpenClaw，实现 Discord DM/Server 通信 + Claude Code ACP 持久会话。
---

# Discord Channel Config 技能

## 功能定位
完整记录 OpenClaw 接入 Discord 频道的配置流程，包括 Discord 应用创建、Bot 配置、OAuth2 邀请链接生成、OpenClaw 频道配置、以及 Claude Code ACP 持久会话方案。

---

## 前置条件

- **OpenClaw** v2026.3.13+
- **Node.js** + **npm**（用于安装 Bun）
- **Bun** 1.3.11+（Discord 插件依赖）
- **Discord 账号**（需要登录 Discord Developer Portal）
- **Claude Code** v2.1.80+（如果需要 ACP 持久会话）
- **claude.ai 订阅**（仅 Claude Code 官方频道需要；OpenClaw Discord channel 不需要）

---

## 操作指南

### 1. 创建 Discord 应用和 Bot

#### 1.1 创建应用
1. 打开 [Discord Developer Portal](https://discord.com/developers/applications)
2. 登录你的 Discord 账号
3. 点击 **New Application**，输入名称（如 `OpenClaw`）
4. 同意 Terms of Service → Create

#### 1.2 配置 Bot 页面
1. 左侧栏点击 **Bot**
2. 注意 **Username** 和 **Discriminator**（如 `bot1485878220900405268#1142`）
3. 记录 **App ID**（从 General Information 页面获取，如 `1485878220900405268`）

#### 1.3 开启 Privileged Gateway Intents
在 Bot 页面滚动到 **Privileged Gateway Intents**，开启：
- ✅ **Message Content Intent**（必须 — 读取消息内容）
- ✅ **Server Members Intent**（推荐 — 用户名/角色匹配）
- ⬜ Presence Intent（可选 — 状态更新，一般不需要）

点击 **Save Changes**（绿色横幅提示 "Bot successfully updated!"）。

#### 1.4 获取 Bot Token
1. 在 Bot 页面找到 **Token** 区域
2. 点击 **Reset Token**（首次创建实际是生成 token，不是重置）
3. 在弹出的确认对话框点击 **Yes, do it!**
4. 通过 Cloudflare Turnstile 验证（需**用户手动**长按按钮）
5. **复制并保存 Token** — 这是唯一的显示机会，token 只能看一次

> ⚠️ Token 是敏感信息，不要在聊天中直接发送。用环境变量或直接写入 OpenClaw 配置。

**Browser 自动化注意**：Discord Developer Portal 的 Token 显示在特殊 DOM 元素中，常规 JS 选择器无法提取（非标准 input/text 元素）。需要用户手动复制。

---

### 2. 生成 OAuth2 邀请链接

#### 2.1 进入 OAuth2 页面
左侧栏点击 **OAuth2** → **URL Generator**

#### 2.2 选择 Scopes
勾选：
- ✅ `bot` — 基本 bot 功能
- ✅ `applications.commands` — 斜杠命令支持

#### 2.3 选择 Bot Permissions
勾选以下权限（按需选择，推荐最小权限）：

| 权限 | 说明 | 是否必须 |
|------|------|----------|
| View Channels | 查看频道 | ✅ 必须 |
| Send Messages | 发送消息 | ✅ 必须 |
| Read Message History | 读取历史消息 | ✅ 必须 |
| Embed Links | 嵌入链接 | ✅ 推荐 |
| Attach Files | 附加文件 | ✅ 推荐 |
| Add Reactions | 添加表情反应 | 可选 |
| Send Messages in Threads | 线程中发消息 | 可选（ACP 持久会话需要） |
| Create Public/Private Threads | 创建线程 | 可选（ACP 持久会话需要） |

> 💡 如果想快速全选，勾选 **Administrator**（权限值 64），但生产环境建议最小权限。

#### 2.4 获取邀请链接
页面底部的 **Generated URL** 就是邀请链接，格式如：
```
https://discord.com/oauth2/authorize?client_id=1485878220900405268&permissions=64&integration_type=0&scope=bot+applications.commands
```

#### 2.5 添加 Bot 到服务器
1. 浏览器打开邀请链接
2. 选择目标服务器（如果没服务器，先在 Discord 里创建一个）
3. Continue → Authorize
4. 完成 Captcha 验证

> 💡 Discord 现在支持 **先 DM 再加服务器**（2024年更新），但 bot 需要先通过 OAuth2 添加到某个服务器后才能被 DM。

---

### 3. 获取 Discord IDs

#### 3.1 开启开发者模式
Discord → User Settings（齿轮图标）→ Advanced → **Developer Mode** ✅

#### 3.2 获取 Server ID
右键服务器图标（左侧栏）→ **Copy Server ID**

#### 3.3 获取 User ID
右键自己的头像 → **Copy User ID**

---

### 4. 配置 OpenClaw Discord Channel

#### 4.1 编辑配置文件
直接编辑 `~/.openclaw/openclaw.json`，在 `channels` 下添加：

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "你的BOT_TOKEN",
      "dmPolicy": "pairing",
      "groupPolicy": "disabled",
      "streaming": "partial",
      "allowFrom": ["你的USER_ID"]
    }
  }
}
```

**配置项说明**：

| 字段 | 说明 |
|------|------|
| `enabled` | 是否启用，设为 `true` |
| `token` | Bot Token（第 1.4 步获取） |
| `dmPolicy` | `pairing`（首次 DM 需配对确认）/ `open`（直接回复） |
| `groupPolicy` | `disabled`（不响应群聊）/ `open`（响应所有群消息） |
| `streaming` | `partial`（流式响应，推荐） |
| `allowFrom` | Discord User ID 列表（**必须是字符串**，用引号包裹） |

#### 4.2 添加 Agent 绑定
在 `bindings` 数组中添加 Discord 绑定：

```json
{
  "agentId": "main",
  "match": {
    "channel": "discord"
  }
}
```

#### 4.3 常见配置错误

**错误 1**：Discord IDs 必须是字符串
```
❌ "allowFrom": [1173321850529517729]
✅ "allowFrom": ["1173321850529517729"]
```

**错误 2**：`groupAllowFrom` 不是 Discord channel 的有效字段（Telegram 用法）
```
❌ "groupAllowFrom": [1173321850529517729]  // Discord 不支持
✅ "allowFrom": ["1173321850529517729"]     // 正确写法
```

#### 4.4 重启 Gateway
```bash
openclaw gateway restart
```

---

### 5. 测试连接

1. 在 Discord 里找到你的 bot（服务器成员列表或搜索）
2. 点击 bot 头像 → **Message**，打开私聊
3. 发送一条消息
4. 首次会收到 **配对码**（pairing code），确认后完成配对
5. 再发一条消息，应该能收到回复

---

### 6. Claude Code ACP 持久会话（Discord Thread）

#### 6.1 前提条件
- Discord channel 已配置并正常工作
- Claude Code ACP agent 已在 OpenClaw 中配置

#### 6.2 创建 Thread 并启动 Claude Code
1. 在 Discord server 中右键一个文本频道 → **Create Thread**
2. 给 thread 起个名字（如 `coding-session`）
3. 通过 OpenClaw 发送 spawn 命令：

```
sessions_spawn runtime="acp" agentId="claude-code" thread=true mode="session" task="你的编码任务"
```

4. Claude Code 会绑定到这个 Discord thread
5. 在同一个 thread 里持续对话，上下文不丢失

> ⚠️ **重要区别**：
> - **Telegram**：ACP 只支持 `mode="run"`（一次性），不支持 `thread=true`
> - **Discord**：ACP 支持 `mode="session"` + `thread=true`（持久会话）
> - 这是 Discord 相比 Telegram 的最大优势 — **Claude Code 持久编码会话**

---

### 7. Claude Code 官方频道方案（不推荐，需要付费订阅）

Claude Code 官方也有 Telegram/Discord 频道集成（research preview），但：
- 需要 **claude.ai 登录认证**（Pro/Max 订阅）
- 百炼 API key / 免费模式**不支持**
- 仅支持 Claude 模型，不能切换其他模型

**结论**：如果已有 claude.ai 订阅，官方方案简单直接；否则 OpenClaw Discord channel 是更好的选择。

---

### 8. DM vs Server 区别

| 功能 | DM（私聊） | Server（群聊） |
|------|-----------|---------------|
| 隐私 | 仅你和 bot 可见 | 群成员可见 |
| 配对 | 首次 DM 自动配对 | 需设置 groupPolicy |
| 响应 | 直接回复 | @ 提问者回复 |
| ACP 持久会话 | ❌ 不支持 | ✅ 通过 Thread 支持 |
| 消息流 | 线性对话 | 可用 Thread 组织任务 |

---

### 9. 故障排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Bot 无响应 | Gateway 未重启 | `openclaw gateway restart` |
| Bot 无响应 | Token 无效 | 重新生成 Token 并更新配置 |
| `--channels ignored` (Claude Code) | 未登录 claude.ai | 需要 claude.ai 订阅 |
| Config invalid: `groupAllowFrom` | Discord 不支持此字段 | 改用 `allowFrom` |
| Config invalid: IDs must be strings | User ID 没加引号 | `"allowFrom": ["1173321850529517729"]` |
| Cloudflare Turnstile 验证失败 | 浏览器自动化无法过 CAPTCHA | 需用户手动完成 |
| Token 无法通过 DOM 提取 | Discord 使用特殊渲染 | 需用户手动复制 |
| Bot 无法被 DM | 未添加到任何服务器 | 用 OAuth2 链接添加 bot 到服务器 |

---

## 参考链接

- [OpenClaw Discord 文档](https://docs.openclaw.ai/channels/discord)
- [Discord Developer Portal](https://discord.com/developers/applications)
- [Discord OAuth2 文档](https://discord.com/developers/docs/topics/oauth2)
- [Claude Code 频道文档](https://code.claude.com/docs/en/channels)

---

*最后更新：2026-03-24 | 来源：虎子配置 Discord channel 实战经验*
