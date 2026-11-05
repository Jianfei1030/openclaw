---
name: qq-channel-media
description: >
  QQ Channel（QQBot）图片/媒体发送指南。当用户要求在 QQ 中发送图片、截图、生成的图像时使用此技能。
  涵盖：本地图片发送、截图发送、生成图片发送、媒体路径限制等。
  触发词：发图、发图片、截图、截图发给我、在QQ里发图、生成图片发QQ。
---

# QQ Channel 媒体发送指南

## 核心规则：媒体路径限制

**QQ Bot 插件要求所有本地媒体文件必须在 `~/.openclaw/media/qqbot/` 目录内。**

任何不在此目录下的本地路径都会被拒绝，错误信息：
```
sendMedia(auto) error: Media path must be inside QQ Bot media storage
```

## 发送本地图片

### 步骤

1. **将图片复制到 QQ Bot 媒体目录**：
```bash
mkdir -p ~/.openclaw/media/qqbot/
cp /path/to/image.png ~/.openclaw/media/qqbot/
```

2. **使用 `<qqmedia>` 标签发送**：
```
<qqmedia>/Users/用户名/.openclaw/media/qqbot/image.png</qqmedia>
```

**路径必须是绝对路径**，`~` 需展开为完整路径。

### 发送截图

```bash
# macOS 截图到媒体目录
screencapture -i ~/.openclaw/media/qqbot/screenshot.png
```

然后：
```
<qqmedia>/Users/用户名/.openclaw/media/qqbot/screenshot.png</qqmedia>
```

### 发送生成的图片

用 PIL/p5.js 等生成图片后，**直接保存到媒体目录**：

```python
from PIL import Image
import os

media_dir = os.path.expanduser("~/.openclaw/media/qqbot")
os.makedirs(media_dir, exist_ok=True)
path = os.path.join(media_dir, "output.png")

img = Image.new("RGB", (400, 300), "#ffffff")
img.save(path)
print(path)  # 用于后续 <qqmedia> 发送
```

## 支持的媒体类型

| 扩展名 | 类型 |
|--------|------|
| `.jpg` `.png` `.gif` `.webp` `.bmp` | 图片 |
| `.mp4` `.mov` `.avi` `.mkv` `.webm` | 视频 |
| 其他 | 文件 |

## 内置 QQBot 技能

OpenClaw 自带三个 QQBot 技能：

| 技能 | 路径 | 用途 |
|------|------|------|
| qqbot-channel | `/opt/homebrew/lib/node_modules/openclaw/dist/extensions/qqbot/skills/qqbot-channel/SKILL.md` | 频道管理：查频道列表、子频道、成员、发帖、公告、日程 |
| qqbot-media | `/opt/homebrew/lib/node_modules/openclaw/dist/extensions/qqbot/skills/qqbot-media/SKILL.md` | 富媒体收发：`<qqmedia>` 标签用法 |
| qqbot-remind | `/opt/homebrew/lib/node_modules/openclaw/dist/extensions/qqbot/skills/qqbot-remind/SKILL.md` | 定时提醒：创建/查询/取消提醒 |

## 注意事项

- **文件大小上限 10MB**
- **路径必须是绝对路径**（以 `/` 或 `http` 开头）
- **多个媒体用多个 `<qqmedia>` 标签**
- 远程 URL（http/https）不受路径限制
- `ffmpeg` 未安装时音视频转换受限（`brew install ffmpeg`）
