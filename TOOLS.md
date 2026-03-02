# TOOLS.md - Local Notes

Skills define _how_ tools work. This file is for _your_ specifics — the stuff that's unique to your setup.

## What Goes Here

Things like:

- Camera names and locations
- SSH hosts and aliases
- Preferred voices for TTS
- Speaker/room names
- Device nicknames
- Anything environment-specific

## Examples

```markdown
### Cameras

- living-room → Main area, 180° wide angle
- front-door → Entrance, motion-triggered

### SSH

- home-server → 192.168.1.100, user: admin

### TTS

- Preferred voice: "Nova" (warm, slightly British)
- Default speaker: Kitchen HomePod
```

## Why Separate?

Skills are shared. Your setup is yours. Keeping them apart means you can update skills without losing your notes, and share skills without leaking your infrastructure.

---

## Cron Jobs — 定时提醒

**主人的 Telegram chat ID：`8071307039`**

### 发送到 Telegram 的提醒必须这样创建（否则消息只在内部处理，不会推送）

```json
{
  "sessionTarget": "isolated",
  "payload": { "kind": "agentTurn", "message": "提醒内容" },
  "delivery": { "mode": "announce", "channel": "telegram", "to": "8071307039" }
}
```

### 常见错误（不要这样做）

```json
{
  "sessionTarget": "main",
  "payload": { "kind": "systemEvent", "text": "..." }
}
```
这种配置只注入 agent 内部上下文，**不会发任何消息给主人**。

### 规则

- 发 Telegram 提醒 → `isolated` + `agentTurn` + `delivery`
- agent 内部任务（不需要通知主人）→ `main` + `systemEvent`
- `to` 字段只用裸 chat ID `"8071307039"`，不要加 `"telegram:"` 前缀

### ⚠️ 时区换算（必须遵守）

服务器时区是 Asia/Shanghai（UTC+8），`schedule.at` 字段必须是 **UTC 时间**（Z 结尾）。

**创建定时任务前，必须先用 `date -u` 获取当前 UTC 时间，再加上延迟计算目标时间。**

示例：主人说"5分钟后提醒我"，当前 UTC 是 14:20：
```
at: "2026-02-23T14:25:00.000Z"   ✅ 正确（UTC + 5分钟）
at: "2026-02-23T22:25:00.000Z"   ❌ 错误（把北京时间当成了UTC，差8小时）
```

---

Add whatever helps you do your job. This is your cheat sheet.

---

## 滴答清单（Dida365）OpenAPI 集成

### App 注册信息
- **Client ID**：`Mh4vQq12OmUrF5Uxz8`
- **Client Secret**：存储在 `/root/.openclaw/.env` → `DIDA365_CLIENT_SECRET`
- **Access Token**：存储在 `/root/.openclaw/.env` → `DIDA365_ACCESS_TOKEN`（授权后写入）
- **API Base**：`https://api.dida365.com`
- **Auth Base**：`https://dida365.com`

### OAuth2 授权流程（Authorization Code）

#### Step 1 — 引导授权
```
GET https://dida365.com/oauth/authorize
  ?scope=tasks:read%20tasks:write
  &client_id=Mh4vQq12OmUrF5Uxz8
  &state=<随机字符串>
  &redirect_uri=<你配置的回调地址>
  &response_type=code
```

#### Step 2 — 拿到 code
回调 URL 带回 `?code=xxx&state=xxx`

#### Step 3 — code 换 access_token
```bash
curl -X POST 'https://dida365.com/oauth/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -u 'Mh4vQq12OmUrF5Uxz8:$DIDA365_CLIENT_SECRET' \
  --data-urlencode 'code=YOUR_CODE' \
  --data-urlencode 'grant_type=authorization_code' \
  --data-urlencode 'scope=tasks:read tasks:write' \
  --data-urlencode 'redirect_uri=YOUR_REDIRECT_URI'
```
响应里取 `access_token`，写入 `.env`。

#### Step 4 — 调用 API
所有请求加 Header：
```
Authorization: Bearer $DIDA365_ACCESS_TOKEN
```

### 常用 API

| 功能 | Method | Path |
|------|--------|------|
| 获取指定任务 | GET | `/open/v1/project/{projectId}/task/{taskId}` |
| 获取项目任务列表 | GET | `/open/v1/project/{projectId}/data` |
| 获取所有项目 | GET | `/open/v1/project` |
| 创建任务 | POST | `/open/v1/task` |
| 更新任务 | POST | `/open/v1/task/{taskId}` |
| 完成任务 | POST | `/open/v1/project/{projectId}/task/{taskId}/complete` |
| 删除任务 | DELETE | `/open/v1/project/{projectId}/task/{taskId}` |

### 任务状态字段
- `status: 0` = 未完成
- `status: 2` = 已完成
- `completedTime` = 完成时间（ISO 8601）

### 调用脚本
```bash
node /root/.openclaw/skills/dida365/dida365.mjs tasks:completed
```
