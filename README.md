# 地震预警 Bark 订阅系统

基于 Rust 长驻后端的地震预警实时推送服务。后端同时监听 Wolfx WebSocket、维护订阅、通过 Bark 推送，并内置前端页面。GeoHash 空间索引实现位置匹配。

示例: [http://eew.noctiro.moe](http://eew.noctiro.moe)

## 技术栈

* **核心**: Rust, Axum, sled (DB), tokio-tungstenite (WS)
* **前端**: 单文件 `web/index.html`，原生 JS, CartoCDN 地图
* **部署**: 单二进制内嵌前端；可选 Cloudflare Worker / Vercel / Caddy / Nginx 作为边缘 adapter

## 项目结构

```text
backend/   Rust 后端，编译产物为单一二进制
web/        前端源（唯一一份 index.html）
deploy/     部署适配器
  cloudflare-worker/   静态 + API 反代
  vercel/              rewrites 反代
  caddy/               反向代理示例
  nginx/               反向代理示例
  systemd/             守护进程示例
```

## 部署

### 方式一：单二进制（推荐）

一个二进制同时提供前端页面、API 和地震监听推送，适合 VPS / NAS / Docker / systemd。

```bash
cd backend
cp .env.example .env
set -a; . ./.env; set +a
cargo build --release
./target/release/earthquake-alert-backend
```

浏览器打开 `http://your-server:30010` 即可使用。前端已内嵌于二进制，无需额外静态文件。

### 方式二：Rust 后端 + Cloudflare Worker

Worker 仅负责托管前端页面并把 `/api/*`、`/health` 转发到后端。

```bash
cd deploy/cloudflare-worker
# 编辑 wrangler.toml，将 BACKEND_URL 改为后端地址
wrangler deploy --env production
```

### 方式三：Rust 后端 + Vercel 静态前端

`deploy/vercel/vercel.json` 把 `/api/*` 和 `/health` rewrite 到后端。把 `web/` 作为 Vercel 静态站点根目录，修改 `vercel.json` 中的 `YOUR_BACKEND_HOST` 后部署即可。

### 方式四：Rust 后端 + Caddy 反代

参见 `deploy/caddy/Caddyfile`，将域名改为自己的域名后，Caddy 会自动申请 HTTPS 证书并把整站反代到本机 `127.0.0.1:30010`。

如果希望由 Caddy 托管 `web/` 静态文件，只把 `/api/*` 和 `/health` 反代到后端，使用 `deploy/caddy/Caddyfile.static`。

### 方式五：Rust 后端 + Nginx 反代

参见 `deploy/nginx/earthquake-alert.conf`，将 `/` 反代到本机 `127.0.0.1:30010`。

### 纯静态前端跨域访问后端

把 `web/index.html` 托管到任意静态平台（GitHub Pages、对象存储等），在页面加载前设置：

```html
<script>window.EEW_API_BASE = "https://api.example.com"</script>
```

即可让前端跨域访问独立后端。注意：地震监听和推送必须由后端进程承担，纯静态前端不能单独工作。

### systemd 守护

参见 `deploy/systemd/earthquake-alert.service`，按本机路径修改后：

```bash
sudo cp deploy/systemd/earthquake-alert.service /etc/systemd/system/
sudo systemctl enable --now earthquake-alert
```

### 环境变量

在 `backend/` 下创建 `.env`，运行前导入环境变量，或通过系统环境变量配置。参见下方配置说明。

```bash
cd backend
cp .env.example .env
# 编辑 .env 修改 SERVER_PORT 或 BARK_API_URL 等
set -a; . ./.env; set +a
```


## 配置说明

### 后端环境变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `SERVER_HOST` | `0.0.0.0` | 监听地址 |
| `SERVER_PORT` | `30010` | 服务端口 |
| `DB_PATH` | `./data/earthquake.db` | 数据库路径 |
| `BARK_API_URL` | `https://api.day.app` | Bark 服务器地址 |
| `BARK_SOUND` | (空) | Bark 铃声名称，空表示使用默认 |
| `BARK_VOLUME` | `10` | Bark 推送音量 (0-10) |
| `BARK_GROUP` | `地震预警` | Bark 推送分组名 |
| `BARK_CALL` | `false` | 是否触发 Bark 通话级别推送 |
| `EEW_WEBSOCKET_URL` | `wss://ws-api.wolfx.jp/all_eew` | 地震预警 WebSocket 地址 |
| `RECONNECT_MIN_SECONDS` | `1` | 重连最小间隔秒数 |
| `RECONNECT_MAX_SECONDS` | `30` | 指数退避重连最大间隔秒数 |
| `PUSH_UPDATES` | `false` | 是否推送同一事件的后续报告 |
| `UPDATE_MIN_REPORT_GAP` | `1` | 同事件两次推送之间至少间隔的报告数 |
| `IGNORE_TRAINING` | `true` | 是否跳过演练事件 |
| `IGNORE_CANCEL` | `true` | 是否跳过取消事件 |
| `P_WAVE_KM_S` | `6.0` | P 波传播速度 (km/s) |
| `S_WAVE_KM_S` | `3.5` | S 波传播速度 (km/s) |
| `STALE_ORIGIN_SECONDS` | `600` | 发震时间超过该秒数视为过期 |
| `DEDUP_KEEP_MINUTES` | `120` | 事件去重窗口分钟数 |
| `MAX_DISTANCE_KM` | `1000` | 订阅者最大推送距离 (km)，0 表示不限制 |
| `MAX_CONCURRENT_NOTIFICATIONS` | `1000` | 并发推送上限 |
| `HTTP_POOL_SIZE` | `200` | HTTP 连接池大小 |

## 后端 API 接口

* **订阅**: `POST /api/subscribe`

订阅保存成功后，服务端会通过 Bark 向该 `bark_id` 发送一条订阅成功确认提醒。

```json
{
  "bark_id": "key",
  "location_name": "东京",
  "latitude": 35.6,
  "longitude": 139.6,
  "locations": [
    { "name": "东京", "latitude": 35.6, "longitude": 139.6 }
  ],
  "notify_bands": [
    { "min": 1, "max": 1, "level": "passive", "label": "低烈度" },
    { "min": 2, "max": 2, "level": "active", "label": "中等烈度" },
    { "min": 3, "max": 99, "level": "critical", "label": "高烈度" }
  ]
}
```

* **退订**: `DELETE /api/unsubscribe`

```json
{ "bark_id": "key" }
```

* **状态**: `GET /health`

* **统计**: `GET /api/stats`

## 致谢

* 数据源：[wolfx.jp](https://ws-api.wolfx.jp)
* 推送服务：[Bark](https://github.com/Finb/Bark)
