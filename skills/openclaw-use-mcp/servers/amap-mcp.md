# 高德地图 MCP 配置指南

## 概述

通过 MCP 协议让 OpenClaw 调用高德地图 API，支持地理编码、路线规划、POI 搜索、天气查询等功能。

**官方文档**: <https://lbs.amap.com/api/mcp-server/gettingstarted>

**Key 获取**: <https://lbs.amap.com/api/mcp-server/create-project-and-key>

**支持的传输方式**:
- Streamable HTTP（官方推荐，最简单）
- Node.js stdio（`npx @amap/amap-maps-mcp-server`）

---

## 接入经验（2026-04-20 实测）

### 方案：通过 mcporter CLI 接入（无需改 openclaw.json，无需重启）

**步骤 1：获取 Key**
访问 <https://lbs.amap.com/api/mcp-server/create-project-and-key> 创建项目并获取 Key。

**步骤 2：添加服务**
```bash
mcporter config add amap --url "https://mcp.amap.com/mcp?key=你的Key"
```

**步骤 3：验证**
```bash
mcporter list amap --schema
```

**步骤 4：测试调用**
```bash
mcporter call amap.maps_geo address="上海市人民广场"
```

> 实测返回：`{"location": "121.475233,31.228818", "district": "黄浦区", ...}`

---

## 完整工具列表

### 地理编码（地址 ↔ 经纬度）

| 工具 | 功能 | 示例 | 参数 |
|------|------|------|------|
| `maps_geo` | 地址 → 经纬度 | `address="北京市朝阳区"` | `address`, `city` |
| `maps_regeocode` | 经纬度 → 地址 | `location="116.48,39.99"` | `location` |

### 路线规划

| 工具 | 功能 | 参数 |
|------|------|------|
| `maps_direction_bicycling` | 骑行路线（经纬度版） | `origin`, `destination` |
| `maps_direction_walking` | 步行路线（经纬度版） | `origin`, `destination` |
| `maps_direction_driving` | 驾车路线（经纬度版） | `origin`, `destination` |
| `maps_direction_transit_integrated` | 公交路线（经纬度版） | `origin`, `destination`, `city` |
| `maps_direction_bicycling_by_address` | 骑行路线（地址版）⭐ | `origin_address`, `destination_address` |
| `maps_direction_walking_by_address` | 步行路线（地址版）⭐ | `origin_address`, `destination_address` |
| `maps_direction_driving_by_address` | 驾车路线（地址版）⭐ | `origin_address`, `destination_address` |
| `maps_direction_transit_integrated_by_address` | 公交路线（地址版）⭐ | `origin_address`, `destination_address` |

### 位置服务

| 工具 | 功能 | 参数 |
|------|------|------|
| `maps_ip_location` | IP 定位 | `ip` |
| `maps_weather` | 天气查询 | `city` |
| `maps_distance` | 距离测量 | `origins`, `destination` |

### POI 搜索

| 工具 | 功能 | 参数 |
|------|------|------|
| `maps_text_search` | 关键词搜索 | `keywords`, `city`, `types` |
| `maps_around_search` | 周边搜索 | `location`, `keywords`, `radius` |
| `maps_search_detail` | POI 详情 | `id` |

---

## 调用示例（通过 mcporter）

### 地址转经纬度
```bash
mcporter call amap.maps_geo address="北京市朝阳区阜通东大街6号" city="北京"
```

### 查询天气
```bash
mcporter call amap.maps_weather city="上海"
```

### 驾车路线规划
```bash
mcporter call amap.maps_direction_driving_by_address \
  origin_address="北京市朝阳区" \
  destination_address="北京市海淀区"
```

### 周边 POI 搜索
```bash
mcporter call amap.maps_around_search \
  location="121.475233,31.228818" \
  keywords="餐厅" \
  radius="1000"
```

### 在 OpenClaw 对话中调用

由于是通过 mcporter CLI 接入的，需要在对话中让我通过 `exec` 调用：

> 用户："帮我查一下从人民广场到外滩的步行路线"
>
> 我：`exec mcporter call amap.maps_direction_walking_by_address origin_address="上海市人民广场" destination_address="上海外滩"`

---

## 故障排查

### 工具调用返回错误

- 检查 API Key 是否有效
- 高德 Key 类型必须是 **Web 服务** 类型
- 查看高德控制台配额使用情况

### mcporter 找不到 amap 服务

```bash
# 确认配置
mcporter config list

# 确认服务在线
mcporter list
```

### 网络问题

```bash
# 测试高德 API 连通性
curl "https://restapi.amap.com/v3/geocode/geo?address=北京&key=你的Key"
```

---

## 配置示例（mcporter.json）

```json
{
  "mcpServers": {
    "amap": {
      "url": "https://mcp.amap.com/mcp?key=你的Key"
    }
  }
}
```

配置文件路径：`~/.openclaw/workspace/config/mcporter.json`

---

## 相关链接

- [高德地图 MCP 官方文档](https://lbs.amap.com/api/mcp-server/gettingstarted)
- [高德地图 Web 服务 API](https://lbs.amap.com/api/webservice/summary)
- [高德 Key 申请](https://lbs.amap.com/api/mcp-server/create-project-and-key)
