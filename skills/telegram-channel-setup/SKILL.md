---
name: telegram-channel-setup
slug: telegram-channel-setup
version: 1.1.0
description: Configure or troubleshoot OpenClaw Telegram channels, including bot token setup, DM pairing or allowlists, groups and topics, multi-account routing, proxy/webhook/network settings, SSRF whitelist fixes for mainland China environments, and verification.
---

# Telegram Channel Setup

Use this skill for **OpenClaw Telegram channel** work only.

## Workflow

1. **Classify the request first**
   - New default Telegram bot
   - Existing bot repair / migration
   - DM-only access
   - Group/topic access
   - Multi-account / multi-agent routing
   - Network / proxy / webhook troubleshooting

2. **Inspect current state before editing**
   - Read `C:/Users/jf/AppData/Local/nvm/v24.14.0/node_modules/openclaw/docs/channels/telegram.md` for authoritative behavior.
   - Read `references/config-patterns.md` for compact snippets and common pitfalls.
   - Inspect current config with `gateway` `config.get` before patching.
   - If the change is narrow, inspect only the relevant subtree when practical; otherwise use the Telegram docs because channel schema lookup may be shallow.

3. **Choose the access model explicitly**
   - DM: `dmPolicy = pairing | allowlist | open | disabled`
   - Groups: `groupPolicy = open | allowlist | disabled`
   - Allowed groups go under `channels.telegram.groups`
   - Allowed users inside groups go under `groupAllowFrom` or per-group `allowFrom`
   - Mention gating is controlled by `requireMention`

4. **Apply config with the safest mechanism**
   - Prefer `gateway config.patch` for partial edits.
   - Use `gateway config.get` before patching arrays or nested account maps.
   - For multi-account Telegram setups, configure `channels.telegram.accounts.<accountId>` and, when needed, add a routing rule in `bindings[]`.
   - If a CLI command is required, preserve the exact command text for user approval.

5. **Treat restart-triggering changes as sensitive**
   Before any config change that restarts Gateway, do all of the following:
   - Check active work with `sessions_list` and/or `subagents list`
   - Validate current service state (`openclaw gateway status`, `openclaw status`, and `openclaw gateway validate` when available)
   - Report findings and ask for explicit restart approval

6. **Verify end-to-end**
   - DM the bot and confirm inbound delivery
   - For pairing flows, run `openclaw pairing list telegram` and approve the code
   - For allowlist flows, confirm numeric Telegram user IDs are present
   - For groups, confirm chat ID, `groups` entry, `groupPolicy`, `requireMention`, and Telegram privacy/admin settings
   - For routing, confirm the correct account or binding is selected

7. **Close with a concise handoff**
   Report:
   - Config paths changed
   - Whether restart was required / approved / completed
   - Verification result
   - Remaining manual Telegram-side steps (BotFather, privacy mode, re-add bot, etc.)

## Hard rules

- Pairing is **DM-only**. Never assume group access from previous pairing approvals.
- `groupAllowFrom` stores **Telegram user IDs**, not group IDs.
- Negative Telegram chat IDs belong under `channels.telegram.groups`.
- If multiple Telegram accounts exist, make default routing explicit with `channels.telegram.defaultAccount` or `channels.telegram.accounts.default`.
- If `requireMention = false`, Telegram privacy mode must allow the bot to see all group messages; disabling privacy mode or granting admin may be necessary.
- If logs show `fetch failed` / `getUpdates` issues, prefer `channels.telegram.proxy`; on broken IPv6/WSL2 paths, consider `channels.telegram.network.autoSelectFamily = false`.
- For Telegram/WhatsApp channels, prefer running Gateway on Node rather than Bun.

## Bundled reference

Read `references/config-patterns.md` for:
- DM, group, multi-account, proxy, and webhook snippets
- Verification checklist
- Common mistakes and troubleshooting shortcuts
- **ACP topic binding exit issue** (2026-04-11): `/acp close` must run outside bound topic

---

## 中国大陆环境配置指南（新增）

### 快速配置流程

#### 1. 创建 Telegram Bot

1. 在 Telegram 中搜索 **@BotFather**
2. 发送 `/newbot` 按提示创建
3. 保存 Bot Token（格式：`123456789:ABCdefGHIjklMNOpqrsTUVwxyz`）

#### 2. 配置 openclaw.json

**注意：代理配置需根据机器环境决定。**
- **可直连 Telegram 的 Windows 主机**：无需代理，不要配置 `proxy` 字段。
- **其他机器（如 Mac/国内服务器）**：若直连被墙，则配置 `proxy` 指向本地代理。

```json5
{
  channels: {
    telegram: {
      enabled: true,
      // 单账号模式
      botToken: "123456789:ABCdefGHIjklMNOpqrsTUVwxyz",
      dmPolicy: "pairing",
      groupPolicy: "allowlist",

      // 只有需要代理的机器才配置此字段
      // proxy: "http://proxy.example:7890",  // Clash/Surge 等代理地址

      // 多账号模式（可选）
      accounts: {
        default: {
          botToken: "123456789:ABCdef...",
          // 只有需要代理的机器才配置此字段
          // proxy: "http://proxy.example:7890",
          dmPolicy: "pairing"
        },
        stock: {
          botToken: "987654321:XYZabc...",
          // 只有需要代理的机器才配置此字段
          // proxy: "http://proxy.example:7890",
          dmPolicy: "allowlist",
          allowFrom: ["123456789"]
        }
      }
    }
  }
}
```

#### 3. 启动并配对

```bash
# 启动 Gateway
openclaw gateway

# 查看配对码
openclaw pairing list telegram

# 批准配对（在 Telegram 中给 bot 发 /start 后）
openclaw pairing approve telegram <CODE>
```

---

### Clash Verge 配置要点

1. **开启局域网连接**：
   - Clash Verge → 设置 → Clash 配置
   - 开启 `allow-lan: true`
   - 设置 `bind-address: 0.0.0.0` 或你的局域网 IP（或直接使用 127.0.0.1）
   - 重启 Clash 核心

2. **验证监听状态**：
   ```bash
   # Windows PowerShell
   netstat -ano | findstr "7897"
   # macOS / Linux
   netstat -an | grep 7890
   # 应显示 0.0.0.0:7897、局域网地址或 127.0.0.1:7897（loopback 也可用，但需修改 SSRF 白名单）
   ```

3. **获取局域网 IP**：
   ```bash
   # Windows PowerShell
   (Get-NetIPAddress -AddressFamily IPv4).IPAddress | Where-Object { $_ -notlike "127.*" }
   # macOS / Linux
   ifconfig | grep "inet " | grep -v 127.0.0.1
   # 输出示例：inet 192.0.2.10
   ```

---

### SSRF 白名单修复（关键！）

**问题**：Telegram 媒体下载被 OpenClaw SSRF 保护拦截，报错：
```
[security] blocked URL fetch (url-fetch) target=https://api.telegram.org/file/bot.../photos/file_x.jpg
reason=Blocked hostname (not in allowlist): proxy.example
```

**根因**：Telegram 插件的媒体下载 SSRF 策略硬编码只允许 `api.telegram.org`，没有考虑代理服务器 IP。

**修复方案**：

#### 方案 A：修改编译后的代码（推荐，立即生效）

**Windows 路径**: `C:\Users\<用户名>\AppData\Roaming\npm\node_modules\openclaw\dist\delivery-BOdFEcOu.js`
**macOS 路径**: `/opt/homebrew/lib/node_modules/openclaw/dist/delivery-<hash>.js`

编辑对应的 `delivery-*.js` 文件，找到 `buildTelegramMediaSsrfPolicy` 函数：

```javascript
function buildTelegramMediaSsrfPolicy(apiRoot, dangerouslyAllowPrivateNetwork) {
	const hostnames = ["api.telegram.org", "127.0.0.1", "localhost"];  // 添加 127.0.0.1 和 localhost（如果使用 loopback 代理）
	// 如果使用局域网代理，还要加上代理 IP，如 "192.0.2.10"
	let allowedHostnames;
	if (apiRoot) try {
		const customHost = new URL(apiRoot).hostname;
		if (customHost && !hostnames.includes(customHost)) {
			hostnames.push(customHost);
			allowedHostnames = [customHost];
		}
	} catch {}
	return {
		hostnameAllowlist: hostnames,
		...allowedHostnames ? { allowedHostnames } : {},
		allowRfc2544BenchmarkRange: true
	};
}
```

然后重启 Gateway：
```bash
openclaw gateway restart
```

#### 方案 B：配置 browser.ssrfPolicy（可能不生效）

在 `openclaw.json` 中添加：
```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true,
      allowedHostnames: ["127.0.0.1", "localhost", "192.0.2.10"]
    }
  }
}
```

**注意**：此配置可能不适用于 Telegram 媒体下载，因为 Telegram 有独立的 SSRF 策略函数。

---

### 故障排查速查

#### 问题 1：图片无法接收，报错 "failed to download media"

**症状**：
- 文字消息正常
- 发送图片后 bot 无响应或报错
- 日志显示 `Blocked hostname (not in allowlist): 192.0.2.10`

**排查步骤**：

1. **检查日志**：
   ```bash
   tail -50 ~/.openclaw/logs/gateway.err.log | grep -i "blocked\|telegram\|media"
   ```

2. **验证代理监听**：
   ```bash
   netstat -an | grep 7890
   # 应显示 *.7890 或局域网地址
   ```

3. **测试代理连通性**：
   ```bash
   curl -v -x http://192.0.2.10:7890 "https://api.telegram.org/bot<TOKEN>/getMe"
   ```

4. **应用 SSRF 修复**（见上方）

#### 问题 2：Telegram 完全不通，报错 "Network request failed"

**首先确认机器环境**：
- **可直连 Telegram 的 Windows 主机**：**不要配置 proxy**，直接连接 Telegram API。如果配了 proxy 反而会导致连接失败。
- **其他机器**：若直连被墙，才需要配置代理。

**可能原因（针对需要代理的机器）**：
1. 代理地址配置错误
2. Clash 未开启局域网连接
3. 防火墙阻止

**排查（针对需要代理的机器）**：
```bash
# 1. 检查 openclaw.json 配置（可直连机器不应有 proxy 字段！）
grep -A5 '"telegram"' ~/.openclaw/openclaw.json

# 2. 验证 Clash 监听
lsof -i :7890 | grep LISTEN

# 3. 测试代理
curl -x http://192.0.2.10:7890 https://www.google.com
```

#### 问题 3：配对失败

**症状**：
- `/start` 后 bot 无响应
- `openclaw pairing list telegram` 显示空

**排查**：
1. 检查 Bot Token 是否正确
2. 确认 `channels.telegram.enabled: true`
3. 查看日志：`openclaw logs --follow`

---

### 常用命令

```bash
# 查看配对状态
openclaw pairing list telegram

# 批准配对
openclaw pairing approve telegram <CODE>

# 查看 Telegram 频道状态
openclaw channels status --probe

# 查看日志
openclaw logs --follow | grep telegram

# 重启 Gateway
openclaw gateway restart
```

---

### 经验教训

1. **Clash 必须开启 `allow-lan`**：否则只能监听 `127.0.0.1`，会被 SSRF 拦截
2. **代理 IP 必须加入 SSRF 白名单**：Telegram 媒体下载走独立 SSRF 策略，需要修改源码
3. **配置修改后必须重启 Gateway**：SSRF 策略在启动时加载
4. **更新可能覆盖修改**：OpenClaw 更新后可能需要重新应用 SSRF 修复
5. **群组/topic 流程不通排查** — 优先检查对应 account 的 `groupPolicy` 是否允许目标群接入；不要只重复测试消息链路，还要同时核对群组/topic 类型、命令授权与机器人权限配置。（2026-04-11 虎子+股子 群组排查经验）
6. **ACP 持久会话退出** — 普通 Telegram DM 不适合持久 ACP，`/reset` 会重置 session，`/acp close` 在 DM 中不可靠。需要稳定多轮对话时，优先使用 Telegram topic 或 Discord thread。（2026-04-11 实测）
7. **Telegram 远程控制方案（claude-code-telegram）** — 运行 claude-code-telegram v1.3.0 时，支持多项目会话持久化，通过 `/repo <name>` 切换项目自动恢复会话记录。配置文件在项目根目录的 `.env`。已排除 OpenACP 和官方 Bridge 方案。（2026-04-10 确认）
8. **127.0.0.1 loopback 代理同样需要加白名单** — 不仅局域网 IP 会被 SSRF 拦截，`127.0.0.1` 也会被拦。代理配置为 `http://127.0.0.1:7897` 时，同样需要修改源码加 `"127.0.0.1"` 和 `"localhost"` 到白名单。（2026-04-24 实测）
9. **Telegram 代理配置按网络环境判断** — 能直接连接 Telegram API 的 Windows 主机无需 `proxy`；其他机器若直连被墙才需配置代理。配置错误会导致所有请求失败。

---

### 相关文档

- [Telegram Channel Docs](https://docs.openclaw.ai/channels/telegram.md)
- [Channel Troubleshooting](https://docs.openclaw.ai/channels/troubleshooting.md)
- [Gateway Configuration](https://docs.openclaw.ai/gateway/configuration.md)
- [Security & SSRF Policy](https://docs.openclaw.ai/gateway/security.md)
