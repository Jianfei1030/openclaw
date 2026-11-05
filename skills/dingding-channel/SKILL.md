---
name: dingding-channel
description: 钉钉（DingTalk）Channel 部署与运维指南。覆盖从创建钉钉应用机器人到 OpenClaw 配置、部署、验证的全流程。
---

# 钉钉（DingTalk）Channel 部署技能

## 概述

通过 `@largezhou/ddingtalk` 插件，以 **Stream 模式**将钉钉企业内部机器人接入 OpenClaw。

- **插件**: `@largezhou/ddingtalk` (npm, v2.0.1)
- **消息模式**: Stream（WebSocket 长连接，无需公网 IP）
- **支持**: 私聊、群聊（@机器人）、文本/图片/音视频/文件消息
- **GitHub**: [largezhou/openclaw-dingtalk](https://github.com/largezhou/openclaw-dingtalk)

---

## 第一步：在钉钉开放平台创建应用

### 1. 登录

打开 **<https://open-dev.dingtalk.com/>**，用钉钉 App 扫码登录。确保你已加入一个企业/组织。

### 2. 创建应用

1. 点击 **应用开发** → **企业内部开发** → **创建应用**
2. 填写：
   - **应用名称**：如 `OpenClaw Bot`
   - **应用描述**：OpenClaw AI 助手
   - **应用图标**：可选，建议 240×240 像素
3. 点击 **确认创建**

### 3. 获取凭证

在应用的 **凭证与基础信息** 页面，复制：
- **Client ID**（AppKey，格式如 `dingxxxx`）
- **Client Secret**（AppSecret）

> ⚠️ 妥善保管 Client Secret，不要分享给他人。

### 4. 添加机器人能力

1. 在应用的 **添加应用能力** 页面，选择 **机器人**，点击添加
2. 填写机器人信息（名称、头像、简介）
3. **消息接收模式** 选择 **Stream 模式**（关键！）
4. 保存

### 5. 配置权限

在应用的 **权限管理** 中，确保开通：
- **企业内机器人发送消息** 权限
- **根据 downloadCode 获取机器人接收消息的下载链接**（用于接收图片）

### 6. 发布机器人

1. 点击 **版本管理与发布** → **创建版本**
2. 填入版本号、描述、应用可用范围
3. 点击 **保存** → **确认发布**
4. 发布前可点击 **调试** 进入预置测试群测试

---

## 第二步：安装 OpenClaw 插件

```bash
openclaw plugins install @largezhou/ddingtalk
```

安装位置：`~/.openclaw/extensions/ddingtalk/`

---

## 第三步：配置 Channel

### 单账号配置（最常用）

编辑 `~/.openclaw/openclaw.json`，在 `channels` 中添加：

```json
{
  "channels": {
    "ddingtalk": {
      "enabled": true,
      "clientId": "dingxxxxxxxx",
      "clientSecret": "你的ClientSecret",
      "allowFrom": ["*"]
    }
  }
}
```

### 多账号配置

```json
{
  "channels": {
    "ddingtalk": {
      "enabled": true,
      "accounts": {
        "bot-hr": {
          "name": "HR助手",
          "clientId": "dingxxxxxxxx",
          "clientSecret": "secret_1"
        },
        "bot-tech": {
          "name": "技术支持",
          "clientId": "dingyyyyyyyy",
          "clientSecret": "secret_2"
        }
      },
      "defaultAccount": "bot-hr"
    }
  }
}
```

### 配置字段说明

| 字段 | 说明 |
|------|------|
| `enabled` | 是否启用 |
| `clientId` | 钉钉 AppKey（格式 `dingxxxx`） |
| `clientSecret` | 钉钉 AppSecret |
| `allowFrom` | 白名单：`["*"]` 允许所有人，或填入钉钉用户的 `staffId` |
| `defaultAccount` | 多账号时的默认账号 |

### allowFrom 白名单

- `["*"]`：允许所有用户（默认）
- `["staffId_1", "staffId_2"]`：仅允许指定用户
- `allowFrom[0]` 同时作为主动推送消息（`openclaw send`）的默认目标

---

## 第四步：重启 Gateway 验证

```bash
openclaw gateway restart
```

验证状态：

```bash
openclaw channels status
```

期望输出：

```
- DingTalk default: enabled, configured, running, mode:stream, token:config
```

### ⚠️ 重要：日志中没有 "connected" 是正常的

DingTalk 插件代码中**只在连接失败时记录错误日志**，成功连接时不输出 "connected" 日志。这是正常行为。

- **正确的验证方式**：`openclaw channels status` 显示 `running, mode:stream`
- **错误的验证方式**：在日志中搜索 "connected"（搜不到不代表失败）

---

## 第五步：测试

1. 在钉钉 App 中找到你创建的机器人
2. 发送一条消息测试
3. 群聊中需要 **@机器人** 才会触发响应

---

## 多 Agent 路由

通过 OpenClaw 的 `bindings` 机制，可将钉钉账号绑定到不同 Agent：

```bash
# 将整个钉钉渠道绑定到默认 agent
openclaw agents bind --agent default-agent --bind ddingtalk

# 将特定账号绑定到指定 agent
openclaw agents bind --agent hr-agent --bind ddingtalk:bot-hr
```

或手动编辑 `openclaw.json` 的 `bindings`：

```json
{
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "ddingtalk",
        "accountId": "default"
      }
    }
  ]
}
```

---

## 常见问题

### Q: 日志里没有 "connected" 或 "ready" 日志，是不是没连上？

**A**: 正常。插件只记录错误日志，不记录成功连接。用 `openclaw channels status` 验证，看到 `running, mode:stream` 就是成功的。

### Q: Stream 模式和 HTTP 回调模式有什么区别？

| 模式 | 原理 | 适合场景 |
|------|------|---------|
| Stream | WebSocket 长连接，主动向钉钉拉消息 | ✅ 无公网 IP（家里电脑 / Mac） |
| HTTP 回调 | 钉钉 POST 到你的公网 HTTPS 地址 | 有公网 IP / 域名 / VPS |

**OpenClaw 插件默认使用 Stream 模式**。

### Q: 群聊收不到消息？

**A**: 群聊中必须 **@机器人** 才会触发。私聊则直接对话即可。

### Q: 机器人发布后在钉钉里搜不到？

**A**: 检查：
1. 应用可见范围是否包含目标用户
2. 机器人配置开关是否已打开
3. 是否已发布（不是保存草稿）

### Q: 如何添加第二个钉钉机器人？

```bash
openclaw channels add
```

按提示选择 DingTalk，输入新机器人的 AppKey / AppSecret。

---

## 部署检查清单

- [ ] 钉钉开放平台创建企业内部应用
- [ ] 获取 Client ID 和 Client Secret
- [ ] 添加机器人能力，选择 Stream 模式
- [ ] 开通必要权限（发送消息、下载链接）
- [ ] 发布机器人版本
- [ ] `openclaw plugins install @largezhou/ddingtalk`
- [ ] 配置 `channels.ddingtalk`（clientId / clientSecret / allowFrom）
- [ ] `openclaw gateway restart`
- [ ] `openclaw channels status` 确认 `running, mode:stream`
- [ ] 钉钉 App 测试私聊和群聊

---

## 发送图片/媒体实测结论（2026-04-25 更新）

### 支持能力清单（源码已验证，实测通过）

| 媒体类型 | 消息类型 Key | 参数 | 是否支持 | 备注 |
|---------|-------------|------|---------|------|
| 图片 | `sampleImageMsg` | `photoURL` | ✅ | 实测成功 |
| 语音 | `sampleAudio` | `mediaId`, `duration` | ✅ | 需 ffmpeg |
| 视频 | `sampleVideo` | `videoMediaId`, `picMediaId`, `duration` | ✅ | 需 ffmpeg |
| 文件/附件 | `sampleFile` | `mediaId`, `fileName`, `fileType` | ✅ | 实测 zip 附件成功；支持 xlsx/pdf/zip/doc 等 |

### ⚠️ 本地文件路径限制（关键踩坑）

OpenClaw 有安全沙箱机制，**只允许从特定目录读取本地媒体文件**。
发送图片时如果路径不在白名单内，会报：
```
LocalMediaAccessError: Local media path is not under an allowed directory
```

**默认允许的目录**（`~/.openclaw/` 下）：
- `media/` — 推荐用于主动发送图片 ✅
- `workspace/`
- `canvas/`
- `sandboxes/`
- 以及 OpenClaw 内部管理的 tmp 目录

**❌ 不允许的路径示例**：
- `/tmp/xxx.png`
- `~/projects/xxx/tmp/xxx.png`
- 任何不在上述白名单中的路径

**解决方案**：发送图片前，先将文件复制到 `~/.openclaw/media/` 目录。

### 图片发送完整流程

```
本地图片（必须在允许的目录内）
  → loadWebMedia() 加载
  → uploadMedia() 上传到钉钉 oapi.dingtalk.com/media/upload
  → 拿到 media_id
  → 构造钉钉内部 URL: oapi.dingtalk.com/media/downloadFile?access_token=xxx&media_id=xxx
  → sendImageMessage() 用 sampleImageMsg + photoURL 发送（单聊/群聊自动路由）
```

### 降级策略（插件自动处理）

- 图片发送失败 → 降级为文本 + 链接提示 `📎 附件: <url>`
- 语音/视频但无 ffmpeg → 降级为文件发送，并附带文本提示用户安装 ffmpeg
- 无 `mediaUrl` 时 → 直接跳过媒体发送步骤（提前返回）

### 钉钉 API 限制说明（官方文档确认）

| 类型 | 大小限制 | 支持格式 |
|------|---------|---------|
| 图片 | 20MB | jpg/gif/png/bmp |
| 语音 | 2MB | amr/mp3/wav |
| 视频 | 20MB | mp4 |
| 文件 | 20MB | doc/docx/xls/xlsx/ppt/pptx/zip/pdf/rar |

`access_token` 有效期 2 小时，图片 URL 内部调用钉钉域名，钉钉服务端可直接访问。

---

## 消息推送延迟优化（blockStreamingCoalesce）

### 问题
钉钉插件设置 `blockStreaming: true`（因为钉钉 API 不支持编辑已发消息），导致 LLM 回复 + Tool Call 中间结果全部缓冲，最终一次性推送 20+ 条消息，体验极差。

### 解决方案
在 `openclaw.json` 的 `channels.ddingtalk` 中添加 `blockStreamingCoalesce` 配置：

```json
{
  "channels": {
    "ddingtalk": {
      "enabled": true,
      "clientId": "...",
      "clientSecret": "...",
      "allowFrom": ["*"],
      "blockStreamingCoalesce": {
        "minChars": 1,
        "maxChars": 400,
        "idleMs": 500
      }
    }
  }
}
```

**参数说明**：
- `minChars`: 缓冲最小字符数，达到此值才可能推送（设为 1 表示不设置下限）
- `maxChars`: 缓冲最大字符数，超过此值立即推送（默认 1200，建议 400）
- `idleMs`: 空闲超时毫秒数，无新内容超过此时长即推送（默认 1000，建议 500）

**效果**：将原来"等很久然后突然炸 20 条"变成"每 400 字符或空闲 0.5 秒就推一段"，每个 Tool Call 完成后会及时推送结果。

**是否需重启**：插件有 `reload` 机制，config 动态读取，**通常不需要重启**，下条消息即生效。

**注意**：此配置在每次回复时通过 `resolveBlockStreamingCoalescing()` 从 config 读取，不影响其他 channel。

---

## 经验来源

- 2026-04-18 实测部署：插件日志没有 "connected" 成功提示，误以为连接失败；SIGUSR1 热重载未生效，需要完整重启 Gateway；`openclaw channels status` 才是正确的验证方式。
- 2026-04-25 图片发送实测：OpenClaw 有本地文件路径白名单，/tmp/ 和工作区内的 tmp/ 都不允许，必须用 `~/.openclaw/media/`；图片发送完整流程验证通过（上传 → 构造 URL → 调用 batchSend）。
- 2026-04-25 语音/视频实测：先安装 ffmpeg，再重启 Gateway 才会生效；重启后语音走 `sampleAudio`，视频走 `sampleVideo`，视频会自动带封面图。
- 2026-04-25 附件实测：zip 文件作为 `sampleFile` 成功发送，说明钉钉文件消息可用作附件通道。
- 2026-04-25 **Agent 回复中 `MEDIA:` 行不会触发钉钉媒体发送** — 在钉钉群聊中，Agent 直接在回复文本里写 `MEDIA:/path/to/file.png` 不会被钉钉插件解析，用户收不到附件。**正确方式：使用 `message` 工具的 `media` 参数发送**，例如 `{"action":"send","channel":"ddingtalk","media":"/path/to/file.png","message":"描述"}`。路径必须在 `~/.openclaw/media/` 白名单内。
- 2026-04-26 **blockStreamingCoalesce 推送优化** — 钉钉因 API 不支持编辑消息（blockStreaming: true），默认缓冲 800-1200 chars + 1000ms idle，导致一次性推送 20+ 条。配置 `blockStreamingCoalesce` 将 maxChars 降到 400、idleMs 降到 500 后，推送体验大幅改善。此配置通过 `resolveBlockStreamingCoalescing()` 动态读取，通常无需重启。
