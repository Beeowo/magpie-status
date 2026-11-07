# Magpie local API — research notes

日期：2026-11-07
來源：`yetone/magpie` repo（`docs/integrating.md`、`internal/gateway/*`、`internal/gui/*`）。

## 探索與身分

- 閘道監聽 `127.0.0.1:3425`，或 `settings.json` 的 `port`。
- 設定檔位置：`~/.config/magpie/settings.json`（Windows 使用者目錄下同路徑；`$XDG_CONFIG_HOME/magpie`）。
- `GET /api/hello` → `{"name":"magpie","version":"…"}`。
- 本機 loopback 不需 gateway key；外部程式不應攜帶 key。

## 文件化外部端點

| 端點 | 用途 |
|---|---|
| `GET /v1/models` | 模型目錄（`id`、`display_name`、`owned_by`、`reasoning`、`context_window` 等） |
| `GET /v1/magpie/quotas` | 訂閱額度 |
| `GET /v1/magpie/usage` | 用量摘要 |
| `GET /v1/magpie/usage/requests` | 逐筆請求 |
| `GET /v1/magpie/route?session=` | 單一 session 的路由（需 session id） |
| `GET /v1/magpie/route-events/snapshot` | 進行中請求 + cursor |
| `GET /v1/magpie/route-events?after=<cursor>` | SSE 路由事件 |

## 路由事件（形象 tab 的資料源）

`internal/gateway/route_events.go`：

- `routeActivity` 欄位：`id`、`time`、`agent`、`provider`、`model`、`attempt`、`fallback`、`phase`、`status`、`failed`、`ms`、`ttft_ms`。
- phase：`started`、`attempt`、`responding`、`attempt_done`、`done`、`dropped`。
- event kind：`attempt_started`、`attempt_finished`、`first_content`、`request_finished`、`route_changed`、`request_dropped`。
- 事件含 `active`（目前進行中數量）與 `cursor`。
- Cursor 失效時回 `409`／`reset`，需重抓 snapshot。

## 視窗內部端點（非公開契約）

- `GET /api/state`：Magpie 視窗主要狀態（agents、providers 等）。
- `POST /api/agents/{action}/{id}`：agent 操作；`Pick` 用於模型選擇。
- `GET /api/settings`、`POST /api/settings`：設定。
- 這些端點由桌面視窗使用，無 key；外部 loopback 客戶端通常可讀，但欄位可能隨版本變動。

## 建議

- 文件化端點作為主要資料源。
- `/api/state`／`/api/agents/pick` 隔離在 adapter；Phase 2 驗證實際形狀，必要時改用 CLI（`magpie <agent> model <value>`）。
- 所有請求只打 `127.0.0.1`，不轉發、不記錄敏感內容。
