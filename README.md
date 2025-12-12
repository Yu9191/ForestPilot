# ForestPilot 

## 本地开发
```bash
npm i
npm run dev
```

## Wrangler 使用前：必须先登录（一次即可）

> 第一次使用 Wrangler 部署 Cloudflare Workers 前，需要先登录 Cloudflare 账号，否则 `wrangler deploy` 会报权限/账号相关错误。

### 浏览器授权登录
在项目根目录执行：

```bash
npx wrangler login
```

登录后会弹出浏览器窗口进行 Cloudflare 授权，成功后回到终端即可。

## 部署到 Cloudflare Workers
```bash
npx wrangler deploy
```

部署成功后会输出 Worker 访问地址（URL），用浏览器打开即可使用。

---

## 配置说明

### 1) 授权白名单（wxid）
- 目前授权wxid白名单在 `src/config.js` 的 `ALLOWED_USERS`（写死在代码里）。
- 你要增删授权用户：需要改代码并重新部署。

> 说明：前端会缓存微信号到 localStorage（方便下次自动进入），但每次都会请求后端校验：
> - 进入门禁：`/api/auth_license`
> - 创建房间：`/api/create`（后端二次校验）
>
> 所以：即使用户本地缓存了微信号，只要你后端把他移出白名单并重新部署，他也无法继续创建/使用。

### 2) Telegram 推送（推荐用 Secret）
- `TG_BOT_TOKEN`：Bot Token（建议用 Cloudflare Secret 保存）
- `TG_CHAT_ID`：接收消息的 chatId（vars）
- `TG_ENABLED`：是否启用 TG 推送（vars，true/false）
- `TG_DEBUG_START`：是否推送 start 接口完整返回（vars，调试用，默认 false）

设置 Bot Token（只需一次）：
```bash
npx wrangler secret put TG_BOT_TOKEN
```

### 3) wrangler.toml（关键配置示例）
```toml
name = "forestpilot"
main = "src/index.js"
compatibility_date = "2025-12-12"

[vars]
TG_ENABLED = "true"
TG_CHAT_ID = "5565918727"
TG_DEBUG_START = "false"

[[durable_objects.bindings]]
name = "ROOMS"
class_name = "RoomsDO"

[[migrations]]
tag = "v1"
new_sqlite_classes = ["RoomsDO"]
```

---

## 使用流程（用户）
1. 打开网页
2. 输入授权微信号（通过才进入）
3. 选择 Forest 账号并登录
4. 选择树种 / 专注时长 / 目标人数
5. 点击创建房间后：可以关闭网页，去 App 等待（后台会自动监听并发车）

---

## 后台监听策略（Durable Object）

- DO 每 5 秒获取一次房间详情（检查人数变化）
- 当 `participants >= target_count`：调用 `/start`（POST，不行则 PUT）
- 为避免“HTTP 200 但实际未开始”的误判：会做二次确认（再 GET 检查房间状态）
- 2 分钟内一直 0 人加入：停止监听并清理任务（`storage.delete("task")`）

### 清理规则（你关心的：SQ/存储会不会一直堆？）
- 发车确认成功：会 `delete("task")`
- 2 分钟无人加入：会 `delete("task")`
- 前端点“停止”：会调用 `/api/do_stop`，DO 也会 `delete("task")`

---

## 调试：推送 start 接口完整信息到 TG（可控开关）

默认关闭：
- `TG_DEBUG_START = "false"`

需要排查时开启：
- `TG_DEBUG_START = "true"`

开启后会推送：
- 调用 start 的 method（POST/PUT）
- endpoint
- HTTP status
- response body（自动截断）
- 当时人数/目标人数/重试次数

---

## 常见问题

### 1) 为什么 TG 显示 start(status=200) 但 App 里没开始？
- 可能是 start 接口“受理成功”但开始状态有延迟。
- ForestPilot 已做“开始状态二次确认”，并支持开启 `TG_DEBUG_START` 查看 start 返回与确认过程。

### 2) 用户关闭网页后后台还会监听吗？
会。监听运行在 Durable Object 的 alarm 里，不依赖前端页面。

### 3) 免费 Cloudflare 账户能用 Durable Objects 吗？
可以使用（但有用量/限制）。超量会受限，建议关注 Cloudflare 控制台用量。
