# Design — Magpie 懸浮面板復刻（MyGo）

## 1. Architecture Overview

產品形態 B：**Magpie 插件（JS/Bun）＋ MyGo 桌面浮窗**。兩者獨立進程，透過 Magpie 本機 HTTP API 與檔案系統協作；插件負責被 Magpie 發現、取得／啟動二進位；MyGo 負責 tray、浮窗與資料呈現。

```text
magpie-status/
├── app/                          # MyGo 桌面程式
│   ├── main.go                   # app lifecycle、single instance、tray、window
│   ├── internal/
│   │   ├── magpie/               # Magpie API adapter（唯一對外接口）
│   │   ├── panel/                # 資料快取、輪詢、離線退避
│   │   ├── avatar/               # 狀態機、優先序、fallback、圖片庫
│   │   ├── settings/             # config.json 讀寫與遷移
│   │   └── platform/             # autostart、hotkey、screen、file dialog
│   ├── src/                      # Vite + TypeScript 前端
│   │   ├── tabs/                 # usage / agents / actions / settings / avatar
│   │   ├── components/
│   │   └── mygo.ts               # 由 mygo generate 產生
│   ├── resources/                # icon、內建佔位圖
│   ├── mygo.config.ts
│   └── go.mod
├── plugin/                       # Magpie 插件（npm 包）
│   ├── package.json
│   ├── index.js                  # host 入口：下載、驗證、spawn
│   ├── page/                     # 沙箱設定／狀態頁
│   └── README.md
├── .github/workflows/release.yml # 建置 Windows binary + SHA256SUMS
└── docs/
```

## 2. Component Boundaries

### 2.1 MyGo App

| 模組 | 責任 | 不做的事 |
|---|---|---|
| `main.go` | `RequestSingleInstanceLock`、tray、建立 panel、全域快捷鍵、`SetOpenAtLogin` | 不直接解析 Magpie JSON |
| `internal/magpie` | 探索 port、`/api/hello`、讀 state／quotas／usage／models、SSE、model pick | 不碰 UI 狀態 |
| `internal/panel` | 定期刷新、快取、離線退避、事件廣播 | 不決定圖片 |
| `internal/avatar` | route event → 7 態、agent 優先序、fallback、圖片檔案管理 | 不發 HTTP |
| `internal/settings` | `config.json` schema、版本遷移、預設值 | 不寫 Magpie 設定 |
| 前端 | 5 個 tab、主題、互動 | 不直接呼叫 Magpie API |

### 2.2 Magpie Plugin

| 檔案 | 責任 |
|---|---|
| `index.js` | host hook：`ensureBinary()`、`launch()`、`status()`；處理 options |
| `page/index.html` + `page/app.js` | 顯示狀態、啟動按鈕、binary 路徑設定；用 `magpie.options` 儲存 |
| `package.json` | `magpie.page`、`magpie.actions`、`magpie.icon`、`main` |

插件頁面是沙箱 iframe，`connect-src 'none'`，**不能直接呼叫 MyGo app**；它只透過 bridge 讀 `usage:summary`／`providers:list` 與讀寫 plugin options。App 的啟動由 host 在頁面被服務時（`plugin.Warm`）執行；app 本身也會用 `SetOpenAtLogin` 常駐，避免依賴插件宿主。

## 3. Contracts

### 3.1 Magpie API Adapter

介面（Go）：

```go
type Client interface {
    Discover(ctx context.Context) (Endpoint, error) // settings.json port → 3425
    Hello(ctx context.Context) (Hello, error)
    State(ctx context.Context) (State, error)       // internal /api/state
    Quotas(ctx context.Context) ([]Quota, error)    // /v1/magpie/quotas
    Usage(ctx context.Context, period string) (Usage, error)
    Models(ctx context.Context) ([]Model, error)    // /v1/models
    PickModel(ctx context.Context, agentID, model, effort string) error
    RouteEvents(ctx context.Context, cursor string) (<-chan RouteEvent, error) // SSE
}
```

- Port 來源：`%APPDATA%\magpie\settings.json`（Windows）或 `$XDG_CONFIG_HOME/magpie/settings.json` 的 `port`，預設 `3425`。只讀不寫。
- `GET /api/hello` 驗證 `{"name":"magpie"}`。
- 外部程式 loopback 不需 gateway key；**不攜帶任何 key**。
- `State`／`PickModel` 走視窗內部 `/api`，列為不穩定介面：集中在 adapter，版本或欄位缺失時降級。
- `PickModel` 首選 `POST /api/agents/pick/{id}`；若 body 形狀在 PoC 不匹配，改用 CLI `magpie <agent> model <value>`（文件中承諾的介面）。
- SSE：`GET /v1/magpie/route-events?after=<cursor>`；斷線用 `Last-Event-ID` 或重新抓 snapshot。

### 3.2 App Config Schema

`%AppData%\MagpiePanel\config.json`：

```json
{
  "version": 1,
  "magpie": { "port": 3425 },
  "window": {
    "alwaysOnTop": true,
    "width": 440,
    "height": 640,
    "stateKey": "magpie-panel-main"
  },
  "hotkey": "Alt+Shift+M",
  "autostart": true,
  "avatar": {
    "defaultSet": "claude",
    "agentOrder": ["claude", "codex", "gemini", "opencode"],
    "sets": {
      "claude": {
        "idle": "claude/idle.gif",
        "routing": "claude/routing.webp",
        "waiting": "claude/waiting.gif",
        "streaming": "claude/streaming.gif",
        "fallback": "claude/fallback.png",
        "failed": "claude/failed.png",
        "offline": "claude/offline.png"
      }
    }
  }
}
```

- 圖片實際存放：`%AppData%\MagpiePanel\avatars\<agent>\<state>.<ext>`；`sets` 只存相對路徑。
- 使用者選檔後**複製**到上述目錄，不直接引用外部路徑（避免檔案被移動）。
- 遷移：`version` 小於目前版本時先備份 `config.json.bak` 再升級。

### 3.3 Avatar State Contract

```text
offline   Magpie 不可達（hello 失敗 / SSE 中斷）
idle      active == 0
routing   active > 0 && phase == started（尚未送出 attempt）
waiting   phase == attempt && ttft_ms == 0（等待首 token）
streaming phase == responding（ttft_ms > 0）
fallback  attempt > 1 || fallback == true（優先於 waiting/streaming 顯示，持續 2s）
failed    failed == true || status >= 400（顯示 3s 後回到 active/idle）
```

- 事件 reducer 維護 `map[routeID]routeActivity`；每次事件重算畫面狀態。
- Agent 選擇：
  1. 有 active route → 取 `agentOrder` 中最靠前者。
  2. 全部 idle → 最後有事件的 agent。
  3. 尚無事件 → `defaultSet`。
- 圖片解析：`sets[agent][state]` → `sets[defaultSet][state]` → `resources/avatar-placeholder.<ext>`。
- GIF／WebP／PNG 由 WebView2 原生支援；單檔上限 20 MB，超過拒絕並提示。

### 3.4 Plugin Host Contract

```js
// index.js
export const MagpiePanel = async ({ magpie }) => ({
  // host 被載入時確保 binary 存在（不阻塞 Magpie 啟動）
  config: async () => { scheduleEnsure(); },
  tool: {},
})
```

- `ensureBinary()`：
  1. `options.binaryPath` 有值 → 直接使用（驗證存在與副檔名）。
  2. `%APPDATA%\MagpiePanel\bin\<version>\magpie-panel.exe` 存在 → 使用。
  3. 否則 `GET https://api.github.com/repos/<owner>/magpie-panel/releases/latest` → 找 `magpie-panel-windows-amd64.zip` 與 `SHA256SUMS` → 下載、驗 SHA256、解壓、原子 rename。
  4. 保留上一版目錄；下載失敗不清舊版。
- `launch()`：`Bun.spawn([exe], { detached: true, stdio: ["ignore","ignore","ignore"] })`；app 自己處理 single instance。
- 狀態回報：plugin options 寫 `lastLaunchAt`、`installedVersion`；頁面用 `magpie.options.get()` 顯示。
- 平台鍵：`windows-amd64`、`windows-arm64`、`darwin-arm64`、`darwin-amd64`、`linux-amd64`、`linux-arm64`；MVP 只發 Windows amd64。

## 4. Data Flow

```text
App 啟動
  → settings.Load()
  → magpie.Discover() → /api/hello
  → 首屏：/api/state + /v1/magpie/quotas + /v1/magpie/usage
  → 開啟 SSE /v1/magpie/route-events
  → 每 5s /api/state、每 15s quotas+usage（可設定）
  → 事件 → panel store → 前端事件（typed event）
  → avatar reducer → avatar:// URL → <img>
```

- 離線：hello 失敗 → UI `offline`；重試 2s → 4s → 8s → 30s 上限。
- Model pick：前端呼叫 bound method → adapter → 成功後刷新 state；失敗回傳錯誤訊息，前端 rollback。
- 圖片變更：前端呼叫 bound method `ChooseAvatar(agent, state)` → `mygo.Dialog` 選檔 → 複製 → 回傳 `avatar://` URL。

## 5. Compatibility & Risks

| 風險 | 影響 | 對策 |
|---|---|---|
| `/api/state`、`/api/agents/pick` 非公開契約 | Magpie 更新可能壞 | adapter 隔離、版本檢查、CLI fallback、錯誤降級 |
| 插件宿主 spawn 限制 | 無法自動啟動 app | Phase 2 PoC；失敗則 app 自帶 autostart，插件頁只做狀態／設定 |
| Windows SmartScreen | 未簽章 exe 警告 | 文件說明；後續加簽 |
| MyGo 需 Go 1.27+ | 開發環境需求 | Phase 2 先 `go version`／`mygo doctor` 檢查 |
| WebView2 缺失 | 無法顯示頁面 | Windows 11 內建；啟動時偵測並提示 |
| 大型 GIF 記憶體 | 卡頓 | 20 MB 上限、尺寸上限、錯誤提示 |
| 多螢幕拔除 | 視窗消失 | `StateKey` + 啟動時 clamp 到可見工作區 |

## 6. Security

- 只連 `127.0.0.1`，不帶 gateway key、不轉發資料。
- GitHub Releases 下載僅 HTTPS；SHA256 不符即刪除並中止。
- Plugin options 視為不可信輸入：binary 路徑必須存在且為 `.exe`／可執行檔；拒絕 `..` 路徑。
- `avatar://` handler 只服務 `%AppData%\MagpiePanel\avatars` 內檔案，拒絕 path traversal。
- 不記錄 prompts、keys、帳號或對話內容。

## 7. Rollout / Rollback

- 首發：Windows amd64，GitHub Releases `v0.1.0`，插件 npm 包同步發佈。
- 插件可隨時在 Magpie 關閉；關閉後 app 仍可獨立執行。
- 保留上一版 binary；`binaryPath` 可手動回退。
- App 不寫 Magpie 的 providers／settings；唯一的寫入是使用者主動的 model pick，可由 Magpie UI 復原。
- 卸載：MyGo installer 的 uninstaller；插件移除不清 `%AppData%\MagpiePanel`，由使用者決定。

## 8. Deferred Technical Unknowns

- `/api/state` 的精確 JSON 欄位與 `/api/agents/pick/{id}` 的 body：Phase 2 PoC 驗證；不影響產品行為。
- 插件宿主能否 `Bun.spawn` 任意外部程式：Phase 2 PoC；失敗則切換為「app 自啟動 + 插件頁狀態」。
- MyGo 版本與 Go 1.27 相容性：Phase 2 `mygo doctor`。
- Windows 程式碼簽章：首發不做，文件標明。
