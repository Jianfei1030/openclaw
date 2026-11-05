# Unreal Engine MCP (UEEditorMCP) 配置指南

## 概述

通过 MCP 协议让 OpenClaw 控制 Unreal Engine 编辑器，支持蓝图创建、材质编辑、UMG 设计、关卡管理等 141+ 个动态动作。

**插件仓库**: https://github.com/yangskin/UEEditorMCP

**核心能力**:
- 7 个固定 MCP 工具（接口永远不变）
- 141 个动态动作（蓝图、材质、UMG、图节点、增强输入等）
- 搜索 → 模式 → 执行 三步发现式工作流
- TCP 持久连接（127.0.0.1:55558）
- 仅 Editor 模块，打包时自动排除

**适用版本**: UE 5.7+

---

## 前置环境

### 1. 安装 Unreal Engine 5.7+

**Epic Games Launcher 安装**:
1. 下载 [Epic Games Launcher](https://store.epicgames.com/)
2. Library → Engine Versions → UE 5.7 → Install
3. 默认路径：`C:\Program Files\Epic Games\UE_5.7\`
4. 磁盘需求：约 **50-70GB**

**验证安装**:
```powershell
Test-Path "C:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealEditor.exe"
```

### 2. 安装 Visual Studio 2022/2026

必须安装 **C++ 桌面开发工作负载**:
- ✅ 使用 C++ 的桌面开发
- ✅ MSVC v143 - VS 2022 C++ x64/x86 生成工具
- ✅ Windows 10/11 SDK
- ✅ C++ CMake tools for Windows

> ⚠️ VS2026 版本警告可忽略：`Visual Studio 2026 compiler is not a preferred version`

### 3. 创建 UE C++ 项目

UEEditorMCP 是 C++ 插件，需要 **C++ 项目**（不是纯蓝图项目）。

1. Epic Launcher → UE 5.7 → Launch
2. Games → 选择 **C++**（不是 Blueprint）
3. 设置项目路径（建议不含中文和空格）
4. 点击 Create Project

---

## 安装步骤

### 第一步：Clone 插件

```powershell
cd <你的UE项目路径>\Plugins
git clone https://github.com/yangskin/UEEditorMCP.git
```

> 中国大陆需代理：
> ```powershell
> git config --global http.proxy http://127.0.0.1:7897
> ```

**验证结构**:
```
Plugins\UEEditorMCP\
├── UEEditorMCP.uplugin
├── Source\           ← C++ 源码
├── Python\           ← Python 服务器
├── setup_mcp.ps1
└── README.md
```

### 第二步：关闭 UE 编辑器

编译前必须**完全关闭** UE，否则 Live Coding 会阻止编译：
```powershell
Get-Process | Where-Object { $_.ProcessName -like "*UnrealEditor*" } | Stop-Process -Force
```

### 第三步：编译插件

```powershell
& "<UE安装路径>\Engine\Build\BatchFiles\Build.bat" `
  <项目名>Editor Win64 Development `
  "<.uproject路径>" -waitmutex
```

**示例**:
```powershell
& "C:\Program Files\Epic Games\UE_5.7\Engine\Build\BatchFiles\Build.bat" `
  TestUE_MCPEditor Win64 Development `
  "C:\UE\TestUE_MCP\TestUE_MCP.uproject" -waitmutex
```

**成功标志**:
```
[XX/XX] Link [x64] UnrealEditor-UEEditorMCP.dll
Result: Succeeded
```

### 第四步：运行 setup 脚本

```powershell
cd <你的UE项目路径>\Plugins\UEEditorMCP
.\setup_mcp.ps1
```

脚本自动完成：
1. 找到 UE 内置 Python 3.11
2. 创建虚拟环境 `Python/.venv`
3. 安装 MCP 包（离线 wheel，无需联网）
4. 生成 `.vscode/mcp.json`

### 第五步：启动 UE 编辑器

双击 `.uproject` 启动项目。

**验证插件加载**:
- Output Log 搜索 `MCP` 或 `UEEditorMCP`
- 确认没有加载错误
- 插件自动在 `127.0.0.1:55558` 启动 TCP 服务器

### 第六步：OpenClaw 配置

**方式一：stdio 传输（推荐）**

```json5
{
  "mcp": {
    "servers": {
      "ue-mcp": {
        "command": "python",
        "args": [
          "<项目路径>\\Plugins\\UEEditorMCP\\Python\\ue_editor_mcp\\server_unified.py"
        ],
        "cwd": "<项目路径>"
      }
    }
  }
}
```

**方式二：TCP 直连**

```json5
{
  "mcp": {
    "servers": {
      "ue-mcp": {
        "url": "tcp://127.0.0.1:55558"
      }
    }
  }
}
```

**重启 Gateway**:
```bash
openclaw gateway restart
```

---

## 7 个 MCP 工具

| # | 工具 | 用途 |
|---|------|------|
| 1 | `ue_ping` | 测试连接，返回 `{pong: true}` |
| 2 | `ue_actions_search` | 按关键字/标签搜索动作 |
| 3 | `ue_actions_schema` | 获取动作的输入模式和示例 |
| 4 | `ue_actions_run` | 执行单个带参数的动作 |
| 5 | `ue_batch` | 批量执行（单次最多 50 个动作）⭐ |
| 6 | `ue_resources_read` | 读取嵌入式文档 |
| 7 | `ue_logs_tail` | 追踪近期日志 |

---

## 动作域覆盖（141 个）

| 域 | 数量 | 说明 |
|----|------|------|
| `blueprint.*` | 11 | 蓝图增删改查、组件、接口 |
| `node.*` | 19 | 蓝图图节点创建 |
| `graph.*` | 18 | 图连线、检视、注释、补丁 |
| `variable.*` | 8 | 变量增删改查 |
| `function.*` | 4 | 函数创建与管理 |
| `dispatcher.*` | 4 | 事件分发器管理 |
| `widget.*` | 21 | UMG 控件 + MVVM |
| `material.*` | 16 | 材质创建、编辑、编译诊断 |
| `component.*` | 4 | 组件属性、网格体、事件绑定 |
| `editor.*` | 18 | 关卡 Actor、视口、资产、日志 |
| `layout.*` | 3 | 节点自动布局 |
| `input.*` | 4 | 增强输入系统 |
| `macro.*` | 1 | 宏管理 |
| `batch.*` | 1 | 批量命令执行 |

---

## 使用示例

### 搜索动作

```bash
mcporter call ue-mcp.ue_actions_search keyword="blueprint"
```

### 执行单个动作

```bash
mcporter call ue-mcp.ue_actions_run \
  action_id="blueprint.create" \
  params='{"name": "BP_Player", "parent_class": "Character"}'
```

### 批量执行（推荐）⭐

```bash
mcporter call ue-mcp.ue_batch \
  actions='[
    {"action_id": "blueprint.create", "params": {"name": "BP_Player", "parent_class": "Character"}},
    {"action_id": "variable.create", "params": {"blueprint_name": "BP_Player", "variable_name": "Speed", "variable_type": "Float"}},
    {"action_id": "blueprint.compile", "params": {"blueprint_name": "BP_Player"}}
  ]'
```

### 读取日志

```bash
mcporter call ue-mcp.ue_logs_tail \
  source="editor" \
  category="LogBlueprint" \
  min_verbosity="Error"
```

---

## 故障排查

### 编译失败：Live Coding 活跃

**症状**: `Unable to build while Live Coding is active`
**解决**: 完全关闭 UE 编辑器，再编译

### Git clone 超时

**解决**:
```powershell
git config --global http.proxy http://127.0.0.1:7897
```

### setup_mcp.ps1 找不到 Python

**解决**:
```powershell
.\setup_mcp.ps1 -EngineRoot "C:\Program Files\Epic Games\UE_5.7"
```

### OpenClaw 连接失败

1. 确认 UE 编辑器已启动（TCP 55558 端口监听）
2. 检查 `.vscode/mcp.json` 路径是否正确
3. 确认 Python .venv 中 mcp 包已安装

### 端口 55558 冲突

```powershell
netstat -ano | findstr 55558
```

---

## 经验教训

1. **中国大陆 clone 必须用代理**，直连约 18KB/s 且频繁超时
2. **编译前必须关闭 UE 编辑器**，Live Coding 会阻止 UBT 编译
3. **setup 脚本使用离线 vendor wheels**，无需联网
4. **VS Code 必须打开项目根目录**（不是 Plugins 子目录）
5. **下载 ZIP 注意目录层级**，`Expand-Archive` 会多包一层
6. **必须先有 UE C++ 项目**，纯蓝图项目无法编译 C++ 插件
7. **VS 必须安装 C++ 桌面开发工作负载**，需要 MSVC + Windows SDK
8. **VS2026 编译警告可忽略**

---

## 相关链接

- [UEEditorMCP GitHub](https://github.com/yangskin/UEEditorMCP)
