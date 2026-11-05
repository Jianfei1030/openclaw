---
name: openclaw-cron-windows
description: Windows环境下OpenClaw定时任务（Cron）的完整配置、故障排查和修复工具。使用场景：(1) 定时任务执行后收不到推送，(2) 定时任务报错，(3) 配置新的定时推送任务，(4) 时区/推送渠道配置错误修复
---

# OpenClaw Windows定时任务配置&排障技能

## 核心故障处理流程
### 步骤1：诊断问题根源
```bash
# 查看所有定时任务状态（重点看lastError字段）
openclaw cron list

# 查看指定任务详细状态（包含报错原因）
openclaw cron runs --id <job-id>
```

### 步骤2：常见问题修复
#### 🔴 问题1：多渠道配置时推送失败
**报错信息**：`Channel is required when multiple channels are configured`
**修复命令**：
```bash
openclaw cron edit <job-id> --announce --to <接收人ID> --channel <telegram|feishu>
```
示例（Telegram推送）：
```bash
openclaw cron edit 88bba6e8-b68f-4b6e-979e-8e50d9d02157 --announce --to "7156779604" --channel "telegram"
```

#### 🔴 问题2：时区配置错误（时间差8小时）
**修复命令**：
```bash
openclaw cron edit <job-id> --tz "Asia/Shanghai"
```
⚠️ 禁止使用`GMT+8`/`CST`等非IANA格式时区

#### 🔴 问题3：任务执行但无推送
检查payload类型是否正确，必须是`agentTurn`而非`systemEvent`，修复命令：
```bash
openclaw cron edit <job-id> --message "<推送内容>"
```

### 步骤3：创建定时任务规范命令
```bash
openclaw cron add --name "<任务名称>" --cron "<cron表达式>" --message "<推送内容>" --tz "Asia/Shanghai" --announce --to "<接收人ID>" --channel "<telegram|feishu>"
```
示例（每天10/14/18点工作日推送）：
```bash
openclaw cron add --name "daily-check" --cron "0 10,14,18 * * 1-5" --message "Read HEARTBEAT.md and execute daily checks" --tz "Asia/Shanghai" --announce --to "7156779604" --channel "telegram"
```

### 步骤4：测试验证
```bash
# 立即运行指定任务测试
openclaw cron run <job-id>

# 查看运行历史
openclaw cron runs --id <job-id> | Select-String "deliveryStatus"
```
预期结果：`deliveryStatus: delivered`

### 步骤5：清理临时任务
```bash
openclaw cron rm <job-id>
```
