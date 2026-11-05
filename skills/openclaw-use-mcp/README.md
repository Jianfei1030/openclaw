# OpenClaw MCP 使用技能

OpenClaw 调用外部 MCP 服务的完整配置指南。

## 目录

- [SKILL.md](./SKILL.md) - 主文档，通用配置方法
- [servers/](./servers/) - 各 MCP server 详细配置
  - [unity-mcp.md](./servers/unity-mcp.md) - Unity MCP (CoplayDev)
  - [amap-mcp.md](./servers/amap-mcp.md) - 高德地图 MCP

## 快速开始

```bash
# 配置 MCP server
openclaw mcp set <name> '{"url": "http://localhost:8080/mcp"}'

# 验证
openclaw mcp list

# 调用工具
mcporter call <name>.tool_name key=value
```

## 支持的 MCP Servers

| Server | 用途 | 传输方式 |
|--------|------|----------|
| [Unity MCP](./servers/unity-mcp.md) | 控制 Unity Editor | HTTP |
| [Unreal Engine MCP](./servers/ue-mcp.md) | 控制 UE 编辑器 | stdio/TCP |
| [Maya MCP](./servers/maya-mcp.md) | 控制 Maya | stdio/Socket |
| [高德地图](./servers/amap-mcp.md) | 地理编码、路线规划 | stdio/SSE |

## 添加新的 MCP Server

1. 在 `servers/` 目录下创建 `<name>-mcp.md`
2. 参考现有文档格式
3. 更新 [SKILL.md](./SKILL.md) 中的链接
4. 更新本 README 的表格
