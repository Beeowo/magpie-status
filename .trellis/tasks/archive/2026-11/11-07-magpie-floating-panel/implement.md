# Implement — Magpie 懸浮面板復刻（MyGo）

## Preconditions

- Go 1.27+、Bun、WebView2（Windows 11 內建）。
- 本機已安裝並可執行 Magpie（`GET http://127.0.0.1:3425/api/hello` 有回應）。
- 可用 GitHub repo 發佈 Releases（MVP 可先用本機檔案模擬下載）。
- MyGo CLI 可用：`go run github.com/egoist/mygo/cmd/mygo@latest doctor`。

## Ordered Checklist

### Step 0 — 專案骨架

- [ ] 建立 `app/`：`go run github.com/egoist/mygo/cmd/mygo@latest init app`（web frontend 模板）。
- [ ] 建立 `plugin/`（npm 包）與 `.github/workflows/release.yml` 佔位。
- [ ] `app/mygo.config.ts`：name `Magpie Panel`、identifier、version `0.1.0`、icon。
- [ ] 確認 `go version`、`bun --version`、`mygo doctor` 全數通過。
- [ ] 根目錄 README 記錄建置與執行方式。

### Step 1 — 風險 PoC（先做，失敗就改路線）

- [ ] **MyGo 視窗 PoC**：tray icon、frameless、AlwaysOnTop、可縮放、最小尺寸、`StateKey`、`GlobalShortcut`、`SetOpenAtLogin`。
- [ ] **Magpie 探索 PoC**：讀 `settings.json` 的 port → `/api/hello` → 記錄 `/api/state`、`/v1/magpie/quotas`、`/v1/magpie/usage`、`/v1/models` 的實際 JSON 形狀。
- [ ] **模型切換 PoC**：驗證 `POST /api/agents/pick/{id}` 的 body；若不匹配，改用 `magpie <agent> model <value>` CLI，記錄結果。
- [ ] **SSE PoC**：連 `/v1/magpie/route-events`，實際觸發一次模型請求，確認 `attempt_started`／`first_content`／`request_finished` 事件。
- [ ] **插件 spawn PoC**：在 Magpie 插件 host 內用 `Bun.spawn` 啟動一個測試 exe，確認可行與 detach 行為。
- [ ] Gate：以上全部通過，或將失敗項寫入 design.md 的 fallback 並經使用者確認。

### Step 2 — App 資料層

- [ ] `internal/settings`：config schema、預設值、版本遷移、`.bak`。
- [ ] `internal/magpie`：`Discover`、`Hello`、`State`、`Quotas`、`Usage`、`Models`、`PickModel`、`RouteEvents`。
- [ ] `internal/panel`：首屏載入、5s／15s 輪詢、離線退避（2s→30s）、事件廣播。
- [ ] bound methods + typed events，前端可取得資料與錯誤。
- [ ] 單元測試：port 解析、JSON 解析、退避曲線、錯誤映射。

### Step 3 — 浮窗外殼與四個基礎 tab

- [ ] 主視窗：frameless、標題列自繪、拖曳區、縮放邊界、關閉即隱藏。
- [ ] tray 選單：顯示／隱藏、永遠置頂、開機啟動、離開。
- [ ] Tab 1 用量／quota：卡片、重置時間、今日／7d／30d。
- [ ] Tab 2 Agents：列表、目前模型、切換模型／reasoning。
- [ ] Tab 3 快速操作：開啟 Magpie 主視窗、叫原生面板、刷新、更新提示。
- [ ] Tab 4 設定：主題、語言、置頂、開機啟動、快捷鍵、Magpie port。
- [ ] 主題：`prefers-color-scheme` + `light-dark()`；zh-TW／en 文案。
- [ ] 視覺比對：與 Magpie `?mode=panel` 截圖並排檢查。

### Step 4 — 「形象」tab

- [ ] 7 態狀態機（offline／idle／routing／waiting／streaming／fallback／failed）。
- [ ] route-events reducer：`map[routeID]routeActivity`、事件合併、fallback／failed 顯示停留。
- [ ] Agent 優先序：`agentOrder`；同時活躍取最高優先；全 idle 取最後活躍 agent。
- [ ] 圖片庫：`ChooseAvatar(agent, state)` → `mygo.Dialog` 選檔 → 複製到 `avatars/` → 回傳 URL。
- [ ] `avatar://` custom scheme：只服務 avatars 目錄，拒絕 path traversal。
- [ ] 圖片解析 fallback：agent 圖 → 預設套 → 內建佔位圖。
- [ ] 格式與大小驗證：webp／png／gif；單檔 ≤ 20 MB。
- [ ] 前端 `<img>` 切換不閃爍（預載下一張、保留尺寸）。

### Step 5 — Magpie 插件

- [ ] `plugin/package.json`：`magpie.page`、`magpie.actions`、`magpie.icon`、`main`。
- [ ] `index.js`：`ensureBinary()`（local override → installed → GitHub Releases 下載＋SHA256）、`launch()`（detached spawn）、狀態寫入 options。
- [ ] `page/`：狀態（Magpie 連線、app 版本、binary 路徑）、啟動／重啟、選擇本機 binary、開啟資料夾。
- [ ] 插件頁只使用 bridge 允許的 API；不嘗試 `fetch`。
- [ ] 沙箱測試：`HOME=$sb magpie plugin add ./plugin`、`magpie plugin --json`。
- [ ] 失敗路徑：下載失敗、SHA 不符、binary 不存在、Magpie 未執行——皆有明確訊息。

### Step 6 — 打包與發佈

- [ ] `bun run build` 產出 Windows exe 與 installer。
- [ ] GitHub Actions：Windows runner 建置、上傳 zip、產生 `SHA256SUMS`、建立 draft release。
- [ ] 插件 npm 發佈（先 `0.1.0-beta`）。
- [ ] 更新檢查：插件比較 `installedVersion` 與 Releases `tag_name`；新版本下載到新目錄，成功後才切換。
- [ ] README：安裝、卸載、資料夾位置、已知限制（SmartScreen）。

### Step 7 — 驗收

- [ ] 全項 AC1–AC8 手動驗收（見 prd.md）。
- [ ] 乾淨機器測試：只裝 Magpie → `magpie plugin add` → 首次下載 → 啟動 → 浮窗可用。
- [ ] Magpie 關閉／重啟／port 變更：app 顯示離線並自動恢復。
- [ ] 多螢幕拔除：視窗回到可見區域。
- [ ] 連續跑 10 分鐘 SSE，無記憶體暴增、無 panic。

## Validation Commands

```bash
# 環境
go version
bun --version
go run github.com/egoist/mygo/cmd/mygo@latest doctor

# App
cd app
go test ./...
go vet ./...
bun run build          # frontend + mygo build
go run .               # 開發執行

# Magpie API
curl -s http://127.0.0.1:3425/api/hello
curl -s http://127.0.0.1:3425/v1/magpie/quotas
curl -s http://127.0.0.1:3425/v1/models
curl -N "http://127.0.0.1:3425/v1/magpie/route-events?after=0:0"

# 插件沙箱
sb=$(mktemp -d)
HOME=$sb XDG_CONFIG_HOME=$sb/.config magpie plugin add ./plugin
HOME=$sb XDG_CONFIG_HOME=$sb/.config magpie plugin --json

# 打包
bun run build -- -platform windows/amd64
sha256sum build/windows-amd64/*
```

## Risky Files / Rollback Points

| 檔案／區域 | 風險 | 回滾 |
|---|---|---|
| `plugin/index.js` | 下載／spawn／路徑處理 | 保留上一版 binary；`binaryPath` 覆蓋；插件可停用 |
| `app/internal/magpie/*` | 依賴 Magpie 內部 API | adapter 集中；CLI fallback；版本檢查 |
| `app/internal/avatar/*` | 狀態機錯誤、圖片遺失 | 預設套＋佔位圖；config `.bak` |
| `app/internal/settings/*` | schema 遷移壞檔 | 寫入前備份、原子寫入 |
| `.github/workflows/release.yml` | 發佈錯誤 | draft release 人工檢查後才 publish |
| `app/mygo.config.ts` | identifier／version 變更 | 首發前固定 identifier |

## Review Gates

1. **PoC Gate（Step 1）**：API 形狀、SSE、spawn、MyGo 視窗全部確認；否則回 design 調整。
2. **UI Gate（Step 3）**：與 Magpie 原生面板並排截圖，主要版面／配色一致。
3. **Avatar Gate（Step 4）**：7 態逐一切換、fallback 鏈、Agent 優先序、GIF 動畫正常。
4. **Install Gate（Step 6）**：乾淨機器完整安裝流程；下載失敗可恢復。
5. **Final Gate（Step 7）**：AC1–AC8 全數通過，才進入 `finish-work`。

## Definition of Done

- `prd.md` 的 AC1–AC8 全部勾選。
- `go test ./...`、`go vet ./...`、前端 typecheck／build 全綠。
- Windows exe ＋ installer 可產出，SHA256SUMS 可驗證。
- 插件可安裝、可停用、可重新啟動 app。
- README 與 LICENSE（Magpie MIT  attribution）齊備。
