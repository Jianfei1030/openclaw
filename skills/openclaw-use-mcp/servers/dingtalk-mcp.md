# 钉钉 MCP 配置指南

## 概述

通过 MCP 协议让 OpenClaw 调用钉钉企业 API，支持通讯录查询、日程管理、待办任务、工作通知等。

**官方文档**: <https://open.dingtalk.com/document/ai-dev/dingtalk-server-api-mcp-overview>
**ModelScope**: <https://www.modelscope.cn/mcp/servers/@open-dingtalk/dingtalk-mcp>
**PyPI**: <https://www.npmjs.com/package/dingtalk-mcp>（通过 npx 运行）

---

## 前置准备

### 1. 获取钉钉应用凭证

复用已有钉钉企业内部应用（即钉钉机器人的应用）的 AppKey/AppSecret：
- `DINGTALK_Client_ID` = AppKey（Client ID）
- `DINGTALK_Client_Secret` = AppSecret（Client Secret）

无需额外申请。

### 2. 开通权限

在钉钉开放平台 → 应用 → 权限管理页面，按需开通以下权限：

| 模块 | 所需权限点 |
|------|-----------|
| 通讯录 | `qyapi_get_member`, `qyapi_addresslist_search` |
| 部门管理 | `qyapi_get_department_list`, `qyapi_get_department_member` |
| AI 表格 | `Notable.Base.Read/Write.All` |
| 日程管理 | `Calendar.Event.Read/Write` |
| 待办任务 | `Todo.Todo.Read/Write` |
| 机器人消息 | `qyapi_robot_sendmsg` |
| 企业荣誉 | `OrgCulture.Honor.Read/Write` |
| 工作通知 | `qyapi_base` |
| 签到管理 | `qyapi_base` |
| 服务窗 | `OfficialAccount.*` |
| 应用管理 | `qyapi_microapp_manage`, `qyapi_get_microapp_list` |
| TB 项目管理 | `Project.*` |
| 日志管理 | `qyapi_report_*` |

⚠️ **权限开通后需重新发布应用版本**，可能需要几分钟才生效。

### 3. 环境要求

- Node.js >= 16.0.0（`npx` 会自动下载 `dingtalk-mcp`）

---

## 配置步骤

### 方式一：mcporter CLI（推荐，无需重启）

在 `config/mcporter.json` 中添加：

```json
{
  "mcpServers": {
    "dingtalk": {
      "command": "npx",
      "args": ["-y", "dingtalk-mcp@latest"],
      "env": {
        "DINGTALK_Client_ID": "你的AppKey",
        "DINGTALK_Client_Secret": "你的AppSecret",
        "ACTIVE_PROFILES": "ALL"
      }
    }
  }
}
```

⚠️ **注意**：不要使用 `mcporter config add dingtalk 'command=...'` 命令——它会把整串当作一个字符串写入 JSON。必须手动编辑 JSON。

### 方式二：.mcp.json（项目级）

在工作区根目录 `.mcp.json` 中添加相同配置。

### 方式三：openclaw.json（Gateway 原生，需重启）

```json5
{
  "mcp": {
    "servers": {
      "dingtalk": {
        "command": "npx",
        "args": ["-y", "dingtalk-mcp@latest"],
        "env": {
          "DINGTALK_Client_ID": "你的AppKey",
          "DINGTALK_Client_Secret": "你的AppSecret",
          "ACTIVE_PROFILES": "ALL"
        }
      }
    }
  }
}
```

### ACTIVE_PROFILES 配置

- `"ALL"` = 启用全部模块
- 按需启用：`"dingtalk-calendar,dingtalk-tasks,dingtalk-contacts"`

---

## 支持的能力

| 模块 | Profile 名称 | 可用工具示例 |
|------|-------------|------------|
| 👥 通讯录 | `dingtalk-contacts` | searchUser, getUserDetailByUserId, getUserIdByMobile, getUserIdByUnionId, getDepartmentUsersByDepId |
| 🏢 部门管理 | `dingtalk-department` | getDepartmentDetail, listSubDepartments, searchDepartment, listSubDepartmentIds |
| 📝 AI 表格 | `dingtalk-notable` | 完整的 CRUD 操作 |
| 📅 日程管理 | `dingtalk-calendar` | createEvent, getCalendarView, updateEvent, deleteEvent, getAttendees |
| ✅ 待办任务 | `dingtalk-tasks` | queryTasks, createTask, updateTask, deleteTask |
| 🤖 机器人消息 | `dingtalk-robot-send-message` | sendDINGMessageByRobot, recallDINGMessage |
| 🎖️ 企业荣誉 | `dingtalk-honor` | grantHonor, queryOrgHonors, createOrgHonor |
| 📧 工作通知 | `dingtalk-notice` | sendNotice |
| ⏰ 签到管理 | `dingtalk-checkin` | getUserCheckinRecords, getDepartmentCheckinRecords |
| 📋 应用管理 | `dingtalk-app-manage` | listAllApp, listAllInnerApps |
| 📝 日志管理 | `dingtalk-report` | 日志查询 |
| 🎯 TB 项目管理 | `dingtalk-teambition` | 项目管理 |

---

## 验证方法

```bash
# 快速验证认证是否通过
mcporter call dingtalk.currentDateTime

# 查看部门信息（验证通讯录）
mcporter call dingtalk.getDepartmentDetail dept_id=1

# 查看企业内部应用列表
mcporter call dingtalk.listAllInnerApps

# 搜索通讯录
mcporter call dingtalk.searchUser queryWord="关键词"
```

### 获取用户 unionId（用于日程/待办操作）

```bash
# 1. 获取根部门所有成员
mcporter call dingtalk.getDepartmentUsersByDepId dept_id=1

# 2. 用 userId 获取详细信息（包含 unionId）
mcporter call dingtalk.getUserDetailByUserId userid="你的userId"
```

---

## 接入经验（2026-04-20 实测踩坑）

1. **mcporter config add 命令格式坑**: `mcporter config add dingtalk 'command=...'` 会把整串当作一个字符串写入 JSON。**必须手动编辑 JSON**，将 `command`、`args`、`env` 分开写。

2. **权限需要逐一开通**: 钉钉应用的权限是逐个申请的，不是一次性全部开通。常见遗漏：
   - 通讯录搜索需要 `qyapi_addresslist_search`（不只是 `qyapi_get_member`）
   - 应用列表需要 `qyapi_get_microapp_list`

3. **权限开通后需重新发布应用版本**，可能需要几分钟才生效。

4. **工具调用参数**: 注意工具名和参数名，例如 `queryTasks` 的第一个参数是 `unionId`（不是 `operatorId`）。

5. **验证方式**: `mcporter call dingtalk.currentDateTime` 即可快速验证认证是否通过。

6. **用户身份信息**: 通过 `getDepartmentUsersByDepId(dept_id=1)` 可获取根部门所有成员的 userId，再通过 `getUserDetailByUserId` 获取 unionId，用于日程/待办等需要用户身份的操作。
