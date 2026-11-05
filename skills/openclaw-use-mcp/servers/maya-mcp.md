# Maya MCP 配置指南

## 概述

通过 MCP 协议让 OpenClaw 控制 Autodesk Maya，支持创建几何体、修改材质、动画控制等操作。

**核心架构**：Python 3 MCP Server + Socket 桥接 → Maya commandPort (7001)

**适用版本**: Maya 2018 (Python 2.7) / Maya 2022+ (Python 3)

---

## 前置环境

### 1. Maya 端配置

在 Maya **Script Editor (Python)** 中运行：

```python
import maya.cmds as cmds

# 关闭可能存在的旧端口
try:
    cmds.commandPort(name=":7001", close=True)
except:
    pass

# 开启新端口，允许 Python 命令
cmds.commandPort(name=":7001", sourceType="python")
print("Maya Command Port opened on 7001")
```

> 💡 **自动化**：添加到 `userSetup.py` 实现启动自动开启

### 2. Python 3 环境

```bash
# 创建虚拟环境
python -m venv venv

# 激活
# Windows: .\venv\Scripts\activate
# Mac/Linux: source venv/bin/activate

# 安装依赖
pip install mcp
```

---

## MCP Server 部署

### 创建 server.py

```python
from mcp.server.fastmcp import FastMCP
import socket

mcp = FastMCP("Maya-Bridge")
MAYA_HOST = "localhost"
MAYA_PORT = 7001

def send_to_maya(command: str) -> str:
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((MAYA_HOST, MAYA_PORT))
        s.sendall((command + "\n").encode('utf-8'))
        data = s.recv(4096)
        s.close()
        return data.decode('utf-8', errors='ignore').strip()
    except Exception as e:
        return f"Error: {e}"

@mcp.tool()
def execute_python(code: str) -> str:
    """Execute Python code in Maya."""
    return send_to_maya(code)

@mcp.tool()
def create_sphere(radius: float = 1.0, name: str = "sphere") -> str:
    """Create a sphere in Maya."""
    return send_to_maya(f'cmds.polySphere(radius={radius}, name="{name}")')

@mcp.tool()
def create_cube(width: float = 1.0, height: float = 1.0, depth: float = 1.0, name: str = "cube") -> str:
    """Create a cube in Maya."""
    return send_to_maya(f'cmds.polyCube(width={width}, height={height}, depth={depth}, name="{name}")')

@mcp.tool()
def move_object(name: str, x: float = 0, y: float = 0, z: float = 0) -> str:
    """Move object to position."""
    return send_to_maya(f'cmds.move({x}, {y}, {z}, "{name}")')

@mcp.tool()
def create_material(name: str, color: list = None) -> str:
    """Create a Blinn material with color."""
    if color is None:
        color = [1, 0, 0]
    code = f'''
shader = cmds.shadingNode('blinn', asShader=True, name='{name}')
sg = cmds.sets(renderable=True, noSurfaceShader=True, empty=True, name='{name}SG')
cmds.connectAttr('%s.outColor' % shader, '%s.surfaceShader' % sg)
cmds.setAttr('%s.color' % shader, {color[0]}, {color[1]}, {color[2]}, type='double3')
shader
'''
    return send_to_maya(code)

@mcp.tool()
def assign_material(object_name: str, material_name: str) -> str:
    """Assign material to object."""
    return send_to_maya(f'''
cmds.select('{object_name}')
cmds.hyperShade(assign='{material_name}SG')
''')

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

---

## OpenClaw 配置

### stdio 传输

```json5
{
  "mcp": {
    "servers": {
      "maya-mcp": {
        "command": "python",
        "args": [
          "C:/path/to/maya-bridge/server.py"
        ],
        "cwd": "C:/path/to/maya-bridge"
      }
    }
  }
}
```

---

## 使用示例

### 创建几何体

```bash
mcporter call maya-mcp.create_sphere radius=5 name="MySphere"
mcporter call maya-mcp.create_cube width=10 height=10 depth=10 name="MyCube"
```

### 移动物体

```bash
mcporter call maya-mcp.move_object name="MySphere" x=5 y=0 z=0
```

### 材质与着色

```bash
# 创建红色材质
mcporter call maya-mcp.create_material name="RedBlinn" color="[1,0,0]"

# 赋予物体
mcporter call maya-mcp.assign_material object_name="MySphere" material_name="RedBlinn"
```

### 执行任意 Python 代码

```bash
mcporter call maya-mcp.execute_python code="cmds.polyCylinder(radius=3, height=10, name='MyCylinder')"
```

---

## 故障排查

### Maya 端问题

| 症状 | 解决 |
|------|------|
| `commandPort` 不存在 | 检查 Maya 版本，2018+ 支持 |
| 连接被拒绝 | 确认端口已开启，防火墙允许 7001 |
| 中文乱码 | Maya 2018 中文支持有限，使用英文变量名 |

### OpenClaw 端问题

| 症状 | 解决 |
|------|------|
| MCP server 无法启动 | 确认 Python 3 和 mcp 包已安装 |
| 命令无响应 | 检查 Maya 是否在前台运行 |
| 返回值异常 | `commandPort` 返回字符串形式，可能为 `None` |

### 网络问题

```bash
# 测试端口连通性
telnet localhost 7001

# PowerShell
Test-NetConnection -ComputerName localhost -Port 7001
```

---

## 安全警告

⚠️ **严禁将端口暴露给公网！** 此桥接器会执行任何收到的 Python 代码，仅限 `localhost` 使用。

---

## 相关链接

- [Maya commandPort 文档](https://help.autodesk.com/view/MAYAUL/2020/ENU/?guid=__CommandsPython_index_html)
