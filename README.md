# FurryBox 每日签到

自动完成 [my.furry.box](https://my.furry.box/wallet) 每日签到，并通过 Telegram Bot 推送结果。支持 Supabase Token 自动刷新，刷新后的 Token 持久化到 GitHub Gist，实现真正的无人值守。

---

## ✨ 特性

- ✅ 每日定时签到（GitHub Actions）
- 🔄 Access Token 过期自动刷新（Supabase `grant_type=refresh_token`）
- 💾 刷新后的 Token 自动写回 Gist，跨运行持久化
- 📨 签到结果推送到 Telegram（成功 / 已签到 / 失败均通知）
- 🎯 `ALREADY_SIGNED_IN` 视为任务完成，不会让 Actions 变红叉
- 🔍 执行前校验 JWT 有效期，提前 5 分钟触发刷新

---

## 📁 项目结构

```
.
├── checkin.py
├── requirements.txt
├── README.md
└── .github
    └── workflows
        └── checkin.yml
```

---

## 🔐 Secrets 配置清单

进入仓库 **Settings → Secrets and variables → Actions → New repository secret**，逐个添加以下变量。

### 一、基础签到（必填）

| Secret 名称 | 必填 | 说明 | 获取方式 |
|---|---|---|---|
| `FURRYBOX_ACCESS_TOKEN` | ✅ | 首次登录后的 access_token（JWT） | 见下方「获取 Access Token」 |
| `FURRYBOX_REFRESH_TOKEN` | ✅ | 首次登录后的 refresh_token | 同上 |
| `FURRYBOX_APIKEY` | ✅ | Supabase publishable key | 固定值：`sb_publishable_KWmBOppH3DRoJf4-g9UCwQ_qLUZojRR` |

### 二、Token 持久化（必填）

| Secret 名称 | 必填 | 说明 | 获取方式 |
|---|---|---|---|
| `GIST_ID` | ✅ | 存放刷新后 Token 的 Gist 的 ID | 见下方「创建 Gist」 |
| `GH_TOKEN` | ✅ | 有 `gist` 读写权限的 GitHub PAT | 见下方「创建 PAT」 |

### 三、Telegram 通知（可选，但强烈推荐）

| Secret 名称 | 必填 | 说明 | 获取方式 |
|---|---|---|---|
| `TG_BOT_TOKEN` | ⭕ | Telegram Bot Token | 见下方「创建 TG Bot」 |
| `TG_CHAT_ID` | ⭕ | 接收消息的 chat id（个人或群） | 同上 |

> 不配置 `TG_*` 时，脚本会自动跳过通知，签到功能不受影响。

---

## 📖 详细获取步骤

### 1. 获取 Access Token / Refresh Token

1. 浏览器登录 [https://my.furry.box](https://my.furry.box)
2. 按 `F12` 打开 DevTools → **Network** 面板
3. 点击签到按钮或刷新页面
4. 找到任意一条 `pwfcgdrorwpdzatxavdt.supabase.co` 的请求
5. 从 **Response** 中复制以下两个字段：
   - `access_token`（很长的 `eyJ...` JWT）
   - `refresh_token`（**必须是完整值**，不能截断）
6. 分别填入 `FURRYBOX_ACCESS_TOKEN` 和 `FURRYBOX_REFRESH_TOKEN`

> ⚠️ 填入 `access_token` 时**不要**带 `Bearer ` 前缀。
> ⚠️ `refresh_token` 必须是完整字符串，截断会导致刷新失败。

### 2. 创建 Gist

1. 打开 [https://gist.github.com](https://gist.github.com)
2. **Filename** 填 `furrybox.json`
3. **Content** 填 `{}`（不能为空，否则无法创建）
4. 点击 **Create secret gist**
5. 从浏览器地址栏复制 Gist ID：

   ```
   https://gist.github.com/你的用户名/a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6
                                        ↑ 这一段就是 GIST_ID
   ```

6. 填入 Secret `GIST_ID`

### 3. 创建 PAT（Personal Access Token）

**Classic Token（推荐，配置简单）**

1. 打开 [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. 点击 **Generate new token (classic)**
3. 填写 Note，Expiration 按需设置
4. **只勾选 `gist` 权限**（不需要其他任何权限）
5. 生成后立即复制 `ghp_...`，填入 Secret `GH_TOKEN`

**Fine-grained Token（如果使用此类）**

1. 在 token 设置页 → **Account permissions**
2. 找到 **Gists** → 设置为 **Read and write**
3. 否则写回 Gist 会返回 `404 Not Found`

### 4. 创建 Telegram Bot（可选）

1. 找 `@BotFather`，发送 `/newbot`，按提示创建
2. 获得 `BOT_TOKEN`（形如 `123456789:AAHxxxx...`），填入 `TG_BOT_TOKEN`
3. 给你的 bot 发一条 `/start` 消息
4. 浏览器访问：

   ```
   https://api.telegram.org/bot<你的BOT_TOKEN>/getUpdates
   ```

5. 在返回 JSON 中找到 `"chat":{"id":123456789,...}`，填入 `TG_CHAT_ID`
6. 如需推送到群组：把 bot 拉入群组，群内发一条消息后再调 `getUpdates`，群 chat id 通常是负数（如 `-1001234567890`）

---

## 🚀 部署步骤

### 1. Fork 或克隆仓库

```bash
git clone https://github.com/你的用户名/furrybox-checkin.git
cd furrybox-checkin
```

### 2. 添加上述所有 Secrets

按「Secrets 配置清单」在仓库 Settings 中配置完毕。

### 3. 首次手动触发

进入仓库 **Actions** 页面：

1. 选择 **FurryBox Daily Signin**
2. 点击 **Run workflow**
3. 查看日志，成功时会出现以下关键行：

   ```
   ✅ 刷新成功，新 exp 剩余 3599 秒
   💾 Token 已写回 Gist
   HTTP 409
   Body: {"error":"ALREADY_SIGNED_IN","message":"今天已经签到过了"}
   📨 TG 已发送
   ```

### 4. 确认定时任务

工作流默认每天 **UTC 02:00（北京时间 10:00）** 自动运行，可在 `checkin.yml` 的 `cron` 表达式中调整。

---

## 🔄 工作原理

```
┌─────────────────────────────────────────┐
│ GitHub Actions 定时触发                 │
└─────────────────┬───────────────────────┘
                  ▼
        ┌─────────────────────┐
        │ 读取 Gist / Secrets │
        │ 获取 Token          │
        └─────────┬───────────┘
                  ▼
        ┌─────────────────────┐
        │ 检查 JWT 剩余时间   │
        │ < 300 秒则刷新      │
        └─────────┬───────────┘
                  ▼
        ┌─────────────────────┐
        │ 刷新（如需）        │
        │ Supabase 标准端点   │
        └─────────┬───────────┘
                  ▼
        ┌─────────────────────┐
        │ 新 Token 写回 Gist  │
        └─────────┬───────────┘
                  ▼
        ┌─────────────────────┐
        │ POST 签到请求       │
        │ {"action":"signin"} │
        └─────────┬───────────┘
                  ▼
        ┌─────────────────────┐
        │ 推送结果到 Telegram │
        └─────────────────────┘
```

### 关键设计

- **Token 持久化必要性**：Supabase 的 refresh_token 每次刷新都会轮换（旧值失效），所以必须把新 Token 写回 Gist，否则第二天会用已失效的旧 refresh_token 导致刷新失败。
- **409 处理**：签到接口返回 `{"error":"ALREADY_SIGNED_IN"}` 是正常情况，脚本视为任务完成，退出码为 0。
- **通知失败不阻塞**：Telegram 发送失败不会影响签到结果和 Actions 状态。

---

## ⚠️ 常见问题

### Q1: 刷新返回 `invalid_grant` / `Invalid Refresh Token`

**原因**：`refresh_token` 已被轮换或截断。

**解决**：重新登录 `my.furry.box`，按「获取 Access Token」步骤重新抓取完整的一对 Token，更新两个 Secret。

### Q2: Gist 写回返回 `404 Not Found`

**原因**：`GIST_ID` 错误，或 `GH_TOKEN` 没有 gist 权限。

**排查**：

```bash
curl -H "Authorization: token 你的GH_TOKEN" \
     https://api.github.com/gists/你的GIST_ID
```

- `200` → 配置正确
- `404` → Gist ID 或权限问题
- `401` → Token 无效或填错

### Q3: 签到返回 401 / 403

**原因**：`FURRYBOX_ACCESS_TOKEN` 无效且刷新失败。

**解决**：检查 `FURRYBOX_REFRESH_TOKEN` 和 `FURRYBOX_APIKEY` 是否正确，必要时重新抓取 Token。

### Q4: Telegram 消息未收到

**排查顺序**：

1. 确认 `TG_BOT_TOKEN` 和 `TG_CHAT_ID` 都有值
2. 浏览器访问 `https://api.telegram.org/bot<TOKEN>/getMe` 验证 bot 有效
3. 确认已给 bot 发送过 `/start`（个人）或 bot 已加入群组
4. 查看 Actions 日志中的 `📨 TG 已发送` 或 `⚠️ TG 失败`

### Q5: Actions 定时任务延迟

GitHub Actions 的 `schedule` 在高负载时段可能延迟数分钟到数十分钟，属于平台行为，不影响功能。

---

## 🛡️ 安全建议

- **不要**把任何 Token / Cookie 硬编码在代码或提交到仓库
- 所有敏感信息一律通过 GitHub Secrets 注入
- 首次配置完成后，建议**退出 `my.furry.box` 重新登录**，使旧 Token 失效，避免已泄露的 Token 被滥用
- `GH_TOKEN` 只授予 `gist` 权限，遵循最小权限原则
- Gist 使用 **secret gist**（非公开）

---

## 📊 运行结果示例

### ✅ 签到成功

```
✅ FurryBox 签到成功
本次奖励：5 boxes
当前余额：? boxes
```

### ℹ️ 今日已签到

```
ℹ️ FurryBox
今日已签到，无需重复操作。
```

### ❌ 签到失败

```
❌ FurryBox 签到失败
错误：UNAUTHORIZED
提示：Token 可能已失效，请重新抓包
```

---

## 📄 License

仅供学习交流，请勿用于商业用途。使用本脚本产生的任何后果由使用者自行承担。
