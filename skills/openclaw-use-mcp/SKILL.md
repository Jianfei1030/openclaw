---
name: openclaw-use-mcp
description: OpenClaw 调用外部 MCP 服务的完整配置指南。涵盖 openclaw.json / mcporter CLI 双路径、HTTP/SSE/stdio 传输协议、以及各类 MCP server（高德地图、Unity、小红书等）的接入方法。
---

# OpenClaw MCP 使用技能

## 概述

本技能指导如何在 OpenClaw 中配置和调用外部 MCP（Model Context Protocol）服务，使 AI Agent 能够操控 Unity、获取地图信息、执行代码等。

**核心能力**:
- 支持 HTTP / SSE / stdio / streamable-http 多种传输协议
- 通过 `openclaw mcp` 命令管理 MCP 配置（需重启 Gateway）
- 通过 `mcporter` CLI 直接调用 MCP 工具（无需重启，自动发现）
- 自动扫描 Cursor / Claude Code / VS Code 等工具的 MCP 配置
- 集成高德地图 MCP、Unity MCP、小红书 MCP 等常用服务

---

## 前置知识

### MCP 传输协议

| 协议 | 适用场景 | 配置字段 |
|------|----------|----------|
| **HTTP** | 远程 MCP server | `url` |
| **SSE** | 实时数据推送 | `url` |
| **stdio** | 本地子进程 | `command`, `args`, `env` |
| **streamable-http** | HTTP 双向流 | `url`, `transport: "streamable-http"` |

### 配置位置

OpenClaw 支持三种 MCP 配置方式：

**方式一：openclaw.json 的 `mcp.servers`（Gateway 原生，需重启）**
```json5
{
  "mcp": {
    "servers": {
      "server-name": {
        "url": "http://localhost:8080/mcp"
      }
    }
  }
}
```

**方式二：mcporter.json（CLI 专用，无需重启，推荐日常使用）**
```json5
// ~/.openclaw/workspace/config/mcporter.json
{
  "mcpServers": {
    "server-name": {
      "url": "https://example.com/mcp?key=xxx"
    }
  }
}
```

**方式三：.mcp.json（项目级，自动发现）**
```json5
// 工作区根目录下的 .mcp.json
{
  "mcpServers": {
    "server-name": { ... }
  }
}
```

### mcporter 自动发现机制

`mcporter` CLI 会自动扫描以下位置的 MCP 配置，无需手动添加到 `openclaw.json`：

| 来源 | 路径 | 说明 |
|------|------|------|
| 工作区 mcporter | `config/mcporter.json` | ⭐ 推荐，项目级 |
| 工作区 .mcp | `.mcp.json` | 项目根目录 |
| VS Code / Cursor | `~/Library/Application Support/Code/User/mcp.json` | macOS |
| Claude Desktop | `~/.claude/mcp.json` | Claude 桌面版配置 |

---

## 快速开始

### 方式 A：mcporter CLI（无需重启，推荐）

```bash
# 1. 添加 HTTP 远程 MCP Server
mcporter config add <name> --url "https://example.com/mcp?key=xxx"

# 2. 添加 stdio 本地 MCP Server
mcporter config add <name> command=npx,args=[-y,@amap/amap-maps-mcp-server],env.AMAP_MAPS_API_KEY=your_key

# 3. 查看已配置的服务
mcporter list

# 4. 查看工具详情
mcporter list <name> --schema

# 5. 调用工具
mcporter call <name>.tool_name key=value
```

### 方式 B：openclaw.json（Gateway 原生，需重启）

```bash
# 1. 配置 MCP Server
openclaw mcp set <name> '{"url": "http://localhost:8080/mcp"}'

# 2. 验证配置
openclaw mcp list

# 3. 重启 Gateway
openclaw gateway restart
```

### 对比

| | mcporter CLI | openclaw.json |
|---|---|---|
| 需要重启 Gateway | ❌ 不需要 | ✅ 需要 |
| 需要改 openclaw.json | ❌ 不需要 | ✅ 需要 |
| 自动发现已有配置 | ✅ 是（Cursor/Claude/VS Code） | ❌ 否 |
| OpenClaw 对话中自动调用 | ❌ 需通过 exec 调用 mcporter | ✅ 是 |
| 适合场景 | 快速验证、轻量使用 | 长期稳定集成 |

---

## 常用命令

### openclaw mcp 命令

| 命令 | 用途 |
|------|------|
| `openclaw mcp list` | 列出配置的 MCP servers |
| `openclaw mcp show <name>` | 查看指定 server 配置 |
| `openclaw mcp set <name> <json>` | 添加/修改配置 |
| `openclaw mcp unset <name>` | 删除配置 |

### mcporter 命令

| 命令 | 用途 |
|------|------|
| `mcporter list` | 列出所有 server 的工具 |
| `mcporter list <name> --schema` | 查看工具详情 |
| `mcporter call <name>.tool key=value` | 调用工具 |
| `mcporter config list` | 查看 mcporter 配置 |

---

## mcporter 配置指南（重点）

### mcporter 是什么

mcporter 是 OpenClaw 官方的 MCP 协议翻译器 + 统一管理器，解决 OpenClaw 运行时不直接扫描外部 MCP 配置的问题。

**工作流程**：
```
用户自然语言 → OpenClaw → exec mcporter → MCP Server → 结果返回
```

### 核心命令

| 命令 | 用途 |
|------|------|
| `mcporter list` | 列出所有已发现的 MCP 服务及状态 |
| `mcporter list <name> --schema` | 查看指定服务的工具详情 |
| `mcporter call <name>.tool key=value` | 手动调用 MCP 工具（排错专用） |
| `mcporter config add <name> --url "..."` | 添加 HTTP 远程服务 |
| `mcporter config add <name> command=...` | 添加 stdio 本地服务 |
| `mcporter config list` | 查看 mcporter 配置文件路径 |
| `mcporter config remove <name>` | 删除服务配置 |

### mcporter 配置存储位置

- **工作区级**：`~/.openclaw/workspace/config/mcporter.json`（推荐）
- **系统级**：`~/.mcporter/mcporter.json`
- **项目级**：`.mcp.json`（工作区根目录）

---

## MCP Server 配置指南

### Unity MCP (CoplayDev)

详见 [servers/unity-mcp.md](./servers/unity-mcp.md)

> **适用场景**: 操控 Unity 编辑器中的 GameObject、材质、场景、Prefab、动画、物理模拟等 40+ 工具。当用户要求「在 Unity 里创建物体」「修改材质球」「调整场景灯光」「播放动画」时使用。
> **传输协议**: HTTP
> **Server 名称**: `unityMCP`
> **适用版本**: Unity 2021.3 LTS+ / 2022.3+ / Unity 6
> **前置条件**: 需在 Unity 中安装 unity-mcp 包并启动 MCP Server（默认端口 8080）

### Unreal Engine MCP (UEEditorMCP)

详见 [servers/ue-mcp.md](./servers/ue-mcp.md)

> **适用场景**: 操控 Unreal Engine 编辑器中的 Actor、蓝图创建、材质编辑、UMG 设计、关卡管理等 141+ 动态动作。当用户要求「在 UE 中创建 Actor」「创建蓝图」「打开关卡」「修改材质」时使用。
> **传输协议**: HTTP / TCP
> **Server 名称**: `ueMCP`
> **适用版本**: UE 5.7+
> **前置条件**: 需在 UE 项目中安装 UEEditorMCP 插件并编译（需 VS C++ 工具链）

### Maya MCP

详见 [servers/maya-mcp.md](./servers/maya-mcp.md)

> **适用场景**: 操控 Autodesk Maya 中的建模（多边形/曲线创建）、材质编辑、动画控制（关键帧/播放控制）、渲染等。当用户要求「在 Maya 中创建球体」「设置关键帧」「修改材质颜色」时使用。
> **传输协议**: Socket 桥接（Python MCP Server → Maya commandPort）
> **Server 名称**: `mayaMCP`
> **适用版本**: Maya 2018 (Python 2.7) / Maya 2022+ (Python 3)
> **前置条件**: 需在 Maya Script Editor 中开启 commandPort 端口（默认 7001）

### 高德地图 MCP

详见 [servers/amap-mcp.md](./servers/amap-mcp.md)

> **适用场景**: 地理编码/逆地理编码、POI 搜索、路线规划、天气查询、距离测量等地图相关操作。当用户问「查一下某地方的位置」「规划路线」「附近有什么」时使用。
> **传输协议**: HTTP (Streamable HTTP)
> **Server 名称**: `amap`
> **前置条件**: 需要高德 API Key

### 钉钉 MCP

详见 [servers/dingtalk-mcp.md](./servers/dingtalk-mcp.md)

> **适用场景**: 通讯录查询、部门管理、日程管理、待办任务、工作通知、机器人消息、AI 表格、TB 项目管理、日志管理等企业办公场景。当用户说「帮我查日程」「创建待办」「给 XXX 发通知」「搜索通讯录」时使用。
> **传输协议**: stdio (通过 npx 运行)
> **Server 名称**: `dingtalk`
> **前置条件**: 需要钉钉企业内部应用的 AppKey/AppSecret，并在开放平台开通对应权限

---

## 故障排查

### mcporter 找不到 server

- 确认 `config/mcporter.json` 中有对应配置
- 运行 `mcporter config list` 查看配置文件路径
- 检查 server 名称拼写
- 运行 `mcporter list` 看服务是否在线

### 工具调用失败

- 查看 Unity/Server 端的错误日志
- 使用 `mcporter list <name> --schema` 确认参数格式
- 检查网络连接（HTTP 模式下）

---

## 相关文件

- [servers/unity-mcp.md](./servers/unity-mcp.md) - Unity MCP 详细配置
- [servers/amap-mcp.md](./servers/amap-mcp.md) - 高德地图 MCP 配置（含完整工具列表 + 接入经验）
- [servers/dingtalk-mcp.md](./servers/dingtalk-mcp.md) - 钉钉 MCP 配置（含 Profile、权限点、实测踩坑经验）
