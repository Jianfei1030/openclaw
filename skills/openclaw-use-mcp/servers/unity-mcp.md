# Unity MCP (CoplayDev) 配置指南

## 概述

通过 MCP 协议让 OpenClaw 控制 Unity Editor，支持场景管理、GameObject 操作、组件编辑、动画控制、物理模拟等 40+ 个工具。

**官方仓库**: https://github.com/CoplayDev/unity-mcp

**适用版本**: Unity 2021.3 LTS+ / 2022.3+ / Unity 6

---

## 安装步骤

### 1. Unity 端安装

**方式一：Package Manager（推荐）**

1. Unity → Window → Package Manager
2. 点击 `+` → Add package from git URL
3. 输入：
   ```
   https://github.com/CoplayDev/unity-mcp.git?path=/MCPForUnity#main
   ```
4. 等待安装完成

**方式二：OpenUPM**
```bash
openupm add com.coplaydev.unity-mcp
```

### 2. 启动 MCP Server

1. Unity → Window → MCP for Unity
2. 点击 **Start Server**
3. 确认显示 🟢 **"Connected ✓"**
4. 记下 HTTP 地址（默认 `http://localhost:8080`）

**传输方式选择**:
- **HTTP** (推荐): 适合 OpenClaw 远程连接
- **stdio**: 适合本地 CLI 工具

### 3. OpenClaw 端配置

**添加 MCP 配置**:

```bash
openclaw mcp set unity-mcp '{"url": "http://localhost:8080/mcp"}'
```

或手动编辑 `~/.openclaw/openclaw.json`:

```json5
{
  "mcp": {
    "servers": {
      "unity-mcp": {
        "url": "http://localhost:8080/mcp"
      }
    }
  }
}
```

**重启 Gateway**:
```bash
openclaw gateway restart
```

### 4. 验证连接

> **注意：server name 以本机实际配置为准。** 本工作区当前实测名称是 **`unityMCP`**（不是 `unity-mcp`）。先跑 `mcporter list` 或查看 `~/.cursor/mcp.json` / mcporter 配置确认名称，再调用。

```bash
# 列出工具
mcporter list unityMCP --schema

# 测试创建 Cube
mcporter call unityMCP.manage_gameobject action=create name="TestCube" primitive_type="Cube"
```

---

## 核心工具分类

### 场景管理 (manage_scene)

| 动作 | 说明 |
|------|------|
| `load` | 加载场景 |
| `create` | 创建新场景 |
| `save` | 保存当前场景 |
| `get_info` | 获取场景信息 |

### GameObject 管理 (manage_gameobject)

| 动作 | 说明 |
|------|------|
| `create` | 创建 GameObject（支持 primitive_type: Cube/Sphere/Capsule/Cylinder/Plane/Quad） |
| `modify` | 修改属性（position/rotation/scale/name/active） |
| `delete` | 删除 GameObject |
| `duplicate` | 复制 |

### 组件管理 (manage_components)

| 动作 | 说明 |
|------|------|
| `add` | 添加组件（如 Rigidbody, BoxCollider） |
| `remove` | 移除组件 |
| `set_property` | 设置组件属性 |
| `get_properties` | 获取组件属性列表 |

### 动画控制 (manage_animation)

| 动作前缀 | 说明 |
|----------|------|
| `animator_*` | Animator 控制（play, crossfade, set 参数） |
| `controller_*` | AnimatorController 管理 |
| `clip_*` | AnimationClip 创建和编辑 |

### 物理控制 (manage_physics)

| 动作 | 说明 |
|------|------|
| `raycast` | 射线检测 |
| `overlap_sphere` | 球形碰撞检测 |
| `add_force` | 添加力（Rigidbody） |
| `set_time_scale` | 设置时间缩放 |

### Camera 控制 (manage_camera)

| 动作 | 说明 |
|------|------|
| `screenshot` | 截图（支持 batch='surround' 六面环绕） |
| `set_position` | 设置相机位置 |
| `set_target` | 设置跟随目标 |
| `set_orthographic` | 切换正交/透视 |

### 材质管理 (manage_material)

| 动作 | 说明 |
|------|------|
| `create` | 创建材质 |
| `set_color` | 设置颜色 |
| `set_texture` | 设置贴图 |
| `apply_to_renderer` | 应用到 MeshRenderer |

### 代码执行 (execute_code)

直接执行 C# 代码：
```csharp
var cube = GameObject.CreatePrimitive(PrimitiveType.Cube);
cube.name = "DynamicCube";
return cube.name;
```

---

## 使用示例

### 创建并配置物体

```bash
# 创建红色 Cube
mcporter call unity-mcp.manage_gameobject \
  action=create \
  name="RedCube" \
  primitive_type="Cube" \
  position="[0,2,0]"

# 添加 Rigidbody
mcporter call unity-mcp.manage_components \
  action=add \
  target="RedCube" \
  component_type="Rigidbody"

# 创建红色材质并应用
mcporter call unity-mcp.manage_material \
  action=create \
  name="RedMat" \
  shader="Standard" \
  properties='{"color": "[1,0,0,1]"}'
```

### 批量操作（推荐）

```bash
mcporter call unity-mcp.batch_execute \
  commands='[
    {"tool": "manage_gameobject", "params": {"action": "create", "name": "Cube1", "primitive_type": "Cube", "position": "[0,0,0]"}},
    {"tool": "manage_gameobject", "params": {"action": "create", "name": "Cube2", "primitive_type": "Cube", "position": "[2,0,0]"}},
    {"tool": "manage_gameobject", "params": {"action": "create", "name": "Cube3", "primitive_type": "Cube", "position": "[4,0,0]"}}
  ]'
```

### 截图

```bash
# 单张截图
mcporter call unityMCP.manage_camera \
  action=screenshot \
  include_image=true

# 六面环绕截图
mcporter call unityMCP.manage_camera \
  action=screenshot \
  batch="surround" \
  include_image=true
```

### 编译与 Console 验证（2026-04-20 实测可用）

> **Unity 开发任务的硬门槛**：写完脚本后不要只看文件存在。必须执行：
> 1. `refresh_unity` 触发刷新/编译
> 2. `read_console` 读取错误
> 3. 再看 Scene View / Game View 截图或实际画面

#### 1) 触发 Unity 刷新 + 请求编译

```bash
mcporter call "unityMCP.refresh_unity(mode: 'force', scope: 'all', compile: 'request', wait_for_ready: true)"
```

- `mode: 'force'`：强制刷新
- `scope: 'all'`：脚本和资源都刷新
- `compile: 'request'`：请求 Unity 编译
- `wait_for_ready: true`：等待 Unity 回到 ready 状态再返回

#### 2) 读取 Unity Console 错误

```bash
mcporter call "unityMCP.read_console(action: 'get', types: ['error'], count: 30, format: 'json')"
```

常用变体：

```bash
# 错误 + 警告
mcporter call "unityMCP.read_console(action: 'get', types: ['error','warning'], count: 50, format: 'json')"

# 包含 stacktrace
mcporter call "unityMCP.read_console(action: 'get', types: ['error'], count: 20, format: 'detailed', include_stacktrace: true)"

# 清空 console
mcporter call "unityMCP.read_console(action: 'clear')"
```

#### 3) 为什么推荐这种写法

在 Windows PowerShell 下，`mcporter call <tool> --args '{...}'` 很容易被引号和 JSON 转义坑到，导致**还没调用到 Unity MCP 就先卡在 mcporter 参数解析**。

对 `read_console` / `refresh_unity` 这类参数稍复杂的工具，优先用 **function syntax**：

```bash
mcporter call "unityMCP.read_console(action: 'get', types: ['error'], count: 30, format: 'json')"
mcporter call "unityMCP.refresh_unity(mode: 'force', scope: 'all', compile: 'request', wait_for_ready: true)"
```

这套写法已在当前机器上实测成功。

---

## 故障排查

### Unity 端问题

| 症状 | 解决 |
|------|------|
| Server 无法启动 | 检查端口 8080 是否被占用 |
| 连接断开 | 重启 MCP for Unity 窗口 |
| 工具调用超时 | 增加 `requestTimeoutMs` 到 60000 |

### OpenClaw 端问题

| 症状 | 解决 |
|------|------|
| `mcporter list` 找不到 server | 确认 `openclaw.json` 中有配置，且 Gateway 已重启；同时确认 server 名到底是 `unityMCP` 还是 `unity-mcp` |
| 连接被拒绝 | 确认 Unity MCP Server 已启动，且 URL 正确 |
| `--args` JSON 总是报解析错误 | Windows PowerShell 下优先改用 function syntax：`mcporter call "unityMCP.read_console(...)"` |
| 工具返回空 | 先执行 `refresh_unity`，再用 `read_console` 读错误 |

### 网络问题

```bash
# 测试连接
curl http://localhost:8080/mcp

# 检查端口占用
netstat -ano | findstr 8080
```

---

## 配置参考

### HTTP 传输（推荐）

```json5
{
  "mcp": {
    "servers": {
      "unity-mcp": {
        "url": "http://localhost:8080/mcp",
        "requestTimeoutMs": 30000
      }
    }
  }
}
```

### stdio 传输（备用）

如需通过 stdio 连接（不推荐，HTTP 更稳定）：

```json5
{
  "mcp": {
    "servers": {
      "unity-mcp": {
        "command": "uvx",
        "args": ["--from", "mcpforunityserver", "mcp-for-unity", "--transport", "stdio"]
      }
    }
  }
}
```

---

## 相关链接

- [CoplayDev/unity-mcp GitHub](https://github.com/CoplayDev/unity-mcp)
- [Unity MCP Asset Store](https://assetstore.unity.com/packages/tools/generative-ai/mcp-for-unity-ai-driven-development-329908)
- [MCP Protocol 文档](https://modelcontextprotocol.io/)
