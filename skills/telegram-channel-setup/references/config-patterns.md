# Telegram Channel Setup Reference

## 1. Decide the target shape

- **Single default bot, safest DM onboarding**: `botToken` + `dmPolicy: "pairing"`
- **Single-owner durable DM access**: `dmPolicy: "allowlist"` + numeric `allowFrom`
- **Group enabled for one allowed group**: add `channels.telegram.groups.<groupId>`
- **Any member in allowed group may talk**: per-group `groupPolicy: "open"`
- **Only specific people in group may talk**: per-group `allowFrom` or top-level `groupAllowFrom`
- **Multiple Telegram bots/accounts**: use `channels.telegram.accounts.<accountId>` + `bindings[]`
- **China/VPS/network issues**: add `channels.telegram.proxy`; consider `network.autoSelectFamily: false`
- **Public ingress required**: webhook mode (`webhookUrl` + `webhookSecret`)

## 2. Base config patterns

### A. Default bot with DM pairing

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

Verification:
```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

### B. Durable DM allowlist for one owner

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "allowlist",
      allowFrom: ["8734062810"],
    },
  },
}
```

Notes:
- `allowlist` requires at least one sender ID.
- `open` requires `allowFrom: ["*"]`.
- Prefer numeric Telegram user IDs over `@username`.

### C. Allow one specific group; anyone in that group may talk

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

Telegram-side requirement:
- If `requireMention: false`, disable BotFather privacy mode or make the bot a group admin.

### D. Allow one specific group; only specific members may talk

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

Use this when sender control belongs only to one group. Use top-level `groupAllowFrom` when the same sender rule should apply to many groups.

### E. Multi-account Telegram setup

```json5
{
  channels: {
    telegram: {
      enabled: true,
      defaultAccount: "main-agent-id",
      accounts: {
        "main-agent-id": {
          botToken: "123:main",
          dmPolicy: "pairing",
        },
        "new-agent-id": {
          botToken: "456:child",
          dmPolicy: "pairing",
          groupPolicy: "open",
          groupAllowFrom: ["8734062810"],
          streaming: "partial",
        },
      },
    },
  },
}
```

Typical binding for the second account:

```json5
{
  agentId: "new-agent-id",
  match: { channel: "telegram", accountId: "new-agent-id" }
}
```

Rules:
- If 2+ accounts exist, set `defaultAccount` explicitly.
- Account-level `allowFrom` / `groupAllowFrom` override inherited values.
- Named accounts do not inherit `accounts.default.allowFrom` / `groupAllowFrom`.

### F. Proxy for unstable direct Telegram API access

```json5
{
  channels: {
    telegram: {
      proxy: "socks5://user:password@proxy-host:1080",
    },
  },
}
```

Use when logs show errors such as:
- `TypeError: fetch failed`
- `Network request for 'getUpdates' failed!`

Optional IPv4-biased fallback:

```json5
{
  channels: {
    telegram: {
      network: {
        autoSelectFamily: false,
      },
    },
  },
}
```

### G. Webhook mode

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "replace-me",
      webhookPath: "/telegram-webhook",
      webhookHost: "127.0.0.1",
      webhookPort: 8787,
    },
  },
}
```

Notes:
- Long polling is the default; use webhook only when you really need public ingress.
- If public URL and local listener differ, place a reverse proxy in front.

## 3. How to find IDs

### Telegram user ID
Safest path:
1. DM the bot
2. Run `openclaw logs --follow`
3. Read `from.id`

Alternative:
```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

### Telegram group ID
Read `chat.id` from logs or Bot API updates.
Group / supergroup IDs are negative, e.g. `-1001234567890`.

## 4. Verification checklist

- Bot token saved in config or env
- Gateway restarted with user approval when required
- DM message reaches the bot
- Pairing code appears and can be approved when using pairing
- `allowFrom` / `groupAllowFrom` entries are numeric IDs
- Allowed groups live under `channels.telegram.groups`
- `requireMention` matches intended UX
- Privacy mode / admin status matches `requireMention`
- Multi-account routing points to the expected `accountId` / `agentId`

## 5. Common mistakes

### Mistake: put group IDs in `groupAllowFrom`
Wrong:
```json5
groupAllowFrom: ["-1001234567890"]
```

Right:
- put `-1001234567890` under `channels.telegram.groups`
- put user IDs under `groupAllowFrom`

### Mistake: expect pairing approvals to unlock groups
Wrong assumption: DM pairing approval lets the user trigger the bot in groups.

Reality:
- pairing is DM-only
- for groups, configure `groupPolicy`, `groups`, and `groupAllowFrom` / per-group `allowFrom`

### Mistake: bot is silent in groups
Check all of these:
- group is allowed under `channels.telegram.groups`
- `groupPolicy` is not `disabled`
- sender is allowed
- `requireMention` is satisfied
- privacy mode/admin setting matches the desired visibility
- after changing privacy mode, remove and re-add the bot

### Mistake: multi-account routing is ambiguous
If multiple Telegram accounts exist and no explicit default is set, routing can be confusing. Set `channels.telegram.defaultAccount`.

### Mistake: commands return "You are not authorized" in groups (2026-04-10)
**Scenario**: 群组里普通消息可用（Bot 会回复聊天内容），但 `/new`、`/reset`、`/status`、`/models` 等命令返回 "You are not authorized to use this command."。

**Root cause**: `groupPolicy: "open"` 只放开群消息，不自动放开群组命令；群命令仍需要 `allowFrom` 和/或 `groupAllowFrom`。

**Solution**: 多账号场景下，为 `default` 和 `stock` 等 account 添加 `allowFrom` 和 `groupAllowFrom`：

```json5
{
  channels: {
    telegram: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        "default": {
          botToken: "123:default",
          dmPolicy: "pairing",
          allowFrom: ["7156779604"],
          groupPolicy: "open",
          groupAllowFrom: ["7156779604"],
        },
        "stock": {
          botToken: "456:stock",
          dmPolicy: "pairing",
          allowFrom: ["7156779604"],
          groupPolicy: "open",
          groupAllowFrom: ["7156779604"],
        },
      },
    },
  },
}
```

**Key insight from OpenClaw docs**:
> "Common confusion: DM pairing approval does not mean 'this sender is authorized everywhere'. Pairing grants DM access only. Group sender authorization still comes from explicit config allowlists. If you want 'I am authorized once and both DMs and group commands work', put your numeric Telegram user ID in `channels.telegram.allowFrom`."

**Verification**:
1. Restart Gateway after config change
2. In group, send `/status` or `/models`
3. Should now return proper response instead of authorization error
4. 在群组里测试 `/new` 或 `/status`，应不再报未授权

## 6. Local notes inherited from prior work

- In China / unstable outbound environments, Telegram often needs an explicit proxy in `channels.telegram.proxy`.
- If `openclaw directory self --channel telegram` returns `null` and `pairing list` is empty, suspect token or network/proxy reachability first.
- Prefer Node runtime for Telegram.

### Mistake: ACP control commands cannot exit from bound topic (2026-04-11)

**Scenario**: After `/acp spawn opencode --bind here`, you cannot exit ACP mode with `/acp close` inside the bound topic. The command is treated as a normal message and routed to the ACP agent instead of OpenClaw Gateway.

**Root cause**: When a topic is bound to an ACP session via `--bind here`:
- All messages in that topic route to the ACP agent (OpenCode/Codex/Claude)
- `/acp close` is an OpenClaw Gateway command, not an ACP agent command
- The ACP agent does not process OpenClaw commands → `/acp close` is ignored
- The command never reaches Gateway, so binding is never removed

**This is a known design limitation** (not a bug per se):
- GitHub Issue #55292: "ACP persistent binding sessions do not deliver completions back to the originating topic"
- GitHub Issue #44492: "Telegram forum topic loses ACP routing after heavy bound turn"
- GitHub Issue #38675: "Stop commands don't work in Telegram topic channels"

**Solution**: Run `/acp close` in a **different conversation** (not the bound topic):

```
# In another Telegram DM or topic:
/acp close agent:opencode:acp:<session-id>

# If you don't know the session id:
/acp sessions
# Then use the session key from the list
```

**Alternative methods**:

```bash
# CLI-based exit (from computer terminal)
openclaw gateway restart  # Clears all bindings

# Or find and close specific session
openclaw acp sessions list
openclaw acp close <session-id>
```

**Key insight**:
> "ACP binding uses `--bind here` which routes all topic messages to the ACP agent. Control commands like `/acp close` must be sent from outside the bound topic to reach OpenClaw Gateway."

**Verification**:
1. In another conversation, run `/acp close <session-key>`
2. Return to original topic, send `/status`
3. Should show normal session (not `agent:opencode:acp:...`)

**Best practice**:
- Before entering ACP mode, note down the session key from the spawn response
- To exit, go to another DM/topic and run `/acp close <session-key>`
- If stuck, `openclaw gateway restart` is the reliable fallback

---

## 7. Source material

Primary references used to build this skill:
- `C:/Users/jf/AppData/Local/nvm/v24.14.0/node_modules/openclaw/docs/channels/telegram.md`
- prior local multi-agent Telegram account setup patterns
- local operating memory about Telegram proxy troubleshooting
