# Magpie 懸浮面板復刻（MyGo）

## Goal

Magpie 執行時，使用者能隨時看到並操作一個與「點擊 Magpie tray icon 後彈出的面板」高度相似的常駐懸浮窗；可自由調整大小與位置，不必反覆點 tray icon。

## Background / Confirmed Facts

- **F1 — Magpie 插件沒有原生浮窗／tray 能力。** 插件是 npm 包，跑在 Bun 宿主，UI 只能是 `magpie.page`（Magpie 視窗側邊欄的沙箱 iframe）、`magpie.panels`（嵌在 Agents／Usage／Settings／provider 頁內）或 `magpie.actions`（選單項目）。頁面 CSP 為 `connect-src 'none'`，可讀資料僅 `usage:summary`、`providers:list`。
  證據：<https://usemagpie.ai/docs/zh/plugins>、`internal/gui/pluginpage.go`、`internal/plugin/page.go`。
- **F2 — Magpie 內建 tray 面板不是可縮放視窗。** 它是 Wails v3 視窗：frameless、AlwaysOnTop、半透明、`DisableResize: true`、`HideOnFocusLost`、Escape 隱藏，高度由頁面內容回報（`FitPanel`）決定，寬度固定；URL 為 `/?mode=panel`。
  證據：`internal/gui/app.go`（`panelOptions`、`fitPanel`、`togglePanelNow`）。
- **F3 — Magpie 本體用 Wails v3，不是 MyGo。** `go.mod` 依賴 `github.com/wailsapp/wails/v3 v3.0.0-beta.24`。
- **F4 — MyGo 具備所需能力。** MyGo（`/egoist/mygo`，v0.6.x）支援：`NewTray`（`OnClick`／`OnRightClick`）、`Panel`／`AlwaysOnTop`／`Frameless`／`Transparent`／`Vibrancy`／`CornerRadius` 視窗、預設可縮放（`DisableResize` 可關）、`Min/MaxWidth/Height`、`StateKey` 記住視窗位置與大小、`FitContent`、webview 或 Go 原生 UI、typed TypeScript IPC。桌面能力另含 `App.SetOpenAtLogin`（Windows Run entry）、`RequestSingleInstanceLock`、`GlobalShortcut.Register`、`mygo.Dialog`（系統檔案對話框）、`App.Path(PathUserData)`、`mygo.Protocol.HandleFunc`（自訂 scheme 供應圖片）、`mygo build`（Windows exe＋NSIS installer）。
  證據：Context7 `/egoist/mygo`；MyGo docs（windows.md、menus.md、launchers.md、app.md、native.md、distribution.md、frontend.md）。
- **F5 — Magpie 有本機 HTTP API 可供外部程式讀取。** 閘道監聽 `127.0.0.1:3425`（或 `settings.json` 的 `port`）；`GET /api/hello` 確認身分；`/v1/models`、`/v1/magpie/quotas`、`/v1/magpie/usage`、`/v1/magpie/usage/requests` 供本機程式使用；loopback 不需 key，外部呼叫可用 `magpie-<app-id>` 計入 Usage。
  證據：`docs/integrating.md`。
- **F6 — Magpie 為 MIT 授權。** 可重製／改編其 HTML／CSS 作視覺復刻，需保留版權聲明。
  證據：`LICENSE`（Copyright (c) 2026 yetone）。
- **F7 — Magpie 有現成的路由即時事件流，適合驅動「形象」狀態。** `GET /v1/magpie/route-events/snapshot` 回傳目前進行中的請求與 cursor；`GET /v1/magpie/route-events?after=<cursor>` 以 SSE 串流事件。事件含 `kind`（`attempt_started`、`first_content`、`attempt_finished`、`request_finished`、`route_changed`、`request_dropped`）、`active`（目前進行中數量）、`route.phase`（`started`／`attempt`／`responding`／`attempt_done`／`done`／`dropped`）、`agent`、`provider`、`model`、`attempt`、`fallback`、`failed`、`ttft_ms`。
  證據：`internal/gateway/route_events.go`。
- **F8 — 插件宿主可啟動外部程式，且開啟插件頁時會喚醒宿主。** 插件跑在 Bun；宿主程式本身用 `Bun.spawn` 管理子進程；插件頁的 HTML 被服務時會呼叫 `plugin.Warm` 啟動宿主。
  證據：`internal/plugin/host.go`、`internal/plugin/host_tree_test.go`、`internal/gui/pluginpage.go`（`pageHost`、`warmHost`）。

## Requirements

### R1 — 常駐懸浮窗
- 提供自己的 tray icon；點擊切換浮窗顯示／隱藏。
- 浮窗為 frameless、AlwaysOnTop、無邊框、可拖曳、可縮放；不因失焦自動關閉（與 Magpie 原生面板不同）。
- 關閉按鈕只隱藏，程式常駐；記住視窗大小與位置。
- 需有最小尺寸限制，避免版面壞掉。
- **開機自動啟動**（登入後常駐，可從設定關閉）。
- **預設永遠置頂**（可從設定或 tray 選單切換）。
- **全域快捷鍵**開關浮窗（預設例如 Alt+Shift+M，可自訂）。
- **多螢幕記憶位置**：記住所在螢幕；螢幕拔除後自動回到可見區域。

### R2 — 視覺復刻
- 版面、字級、間距、圓角、半透明材質、明暗主題，盡量貼近 Magpie `?mode=panel` 的實際外觀。
- 支援跟隨系統明暗主題；Magpie 面板的語言（zh-TW／zh／en）至少支援繁中與英文。

### R3 — 資料來源
- 從 Magpie 本機 API 讀取真實資料（用量、quota／allowance、providers、models），不用假資料。
- 每 N 秒輪詢或事件刷新；Magpie 未執行時顯示明確離線狀態，不崩潰。
- 只讀 loopback；不把 Magpie 的 key／帳號／對話內容帶出或寫入日誌。

### R4 — 面板功能（MVP：四項全包）
- 用量／quota 儀表。
- Agent／模型控制：列出 agent、顯示目前模型、切換模型與 reasoning。
- 快速操作：開啟 Magpie 主視窗、叫出原生 tray 面板、手動刷新、更新提示。
- 主題與偏好：明暗跟隨系統、語言、開機啟動、置頂開關。

### R4.1 — 「形象」tab（本次新增，已決策 7 態＋依 Agent 分套）
- 新增第五個 tab「形象」，頁面只有一個圖像顯示元件（可含少量狀態文字），用來做路由狀態的動態顯示。
- 支援 `webp`、`png`、`gif`（含動畫）等常見格式。
- **7 個狀態**：離線、待機、路由中、等待首 token、串流中、切換備援、失敗。
- **依 Agent 分套**：每個 agent（claude／codex／gemini…）可有自己的一整套 7 態圖片。
- **Fallback 機制**：使用者可指定「預設套」（例如 Claude 套）；某 agent 沒設定圖片，或某狀態缺圖時，依序退回「該 agent 的圖 → 預設套對應圖 → 內建佔位圖」。
- **Agent 優先序**：使用者可自訂 agent 排序；多個 agent 同時活躍時，只顯示優先序最高的那個 agent 的狀態圖。
- **全部待機時**：顯示「最後活躍的 agent」那一套（解讀為最近一次有事件的 agent；若該 agent 缺圖則用預設套）。
- 狀態資料來源優先採用 F7 的 route-events SSE；Magpie 未連線時進入「離線」狀態。
- 圖像預設以 `object-fit: contain` 完整顯示、背景透明；是否改為 cover／填色留待第二輪設定。

### R5 — 與 Magpie 的整合形式（已決策：B）
- 使用者選擇 **B：Magpie 插件（JS）＋ MyGo 浮窗二進位**。
- Magpie 插件負責：在 Magpie 內提供安裝入口／設定頁／選單動作、取得並啟動 MyGo 浮窗程式、回報狀態。
- MyGo 程式負責：tray icon、常駐可縮放浮窗、讀取 Magpie 本機 API、記住視窗狀態。
- 插件宿主是 Bun 進程，具備 spawn 外部程式的能力（`internal/plugin` 測試即以 `Bun.spawn` 啟動子進程）；此假設在 Phase 2 以最小 PoC 驗證。
- 二進位取得方式已定為 GitHub Releases 自動下載（見 R6）。

### R6 — 平台、打包與二進位發佈
- 首發只做 Windows 11（目前開發機）；macOS／Linux 列為後續，不在本次範圍。
- MyGo 本身跨平台，程式結構不綁定 Windows，但打包與驗收只做 Windows。
- **二進位發佈（已決策）**：插件首次啟動時從 GitHub Releases 下載對應平台的 MyGo 二進位，驗證 SHA256 後存到使用者資料夾；允許使用者手動指定本機路徑覆蓋。
- MyGo 端用 `mygo build` 產出 Windows exe 與 NSIS installer；插件端只負責取得與啟動，不重複安裝。
- 需要版本檢查與保留上一版 binary，下載失敗時可回滾。

## Acceptance Criteria

- [x] AC1：點 tray icon 可切換浮窗；浮窗失焦後仍保持顯示（除非使用者手動隱藏）。
- [x] AC2：拖曳邊緣可縮放；重啟後大小與位置還原；低於最小尺寸時被限制。
- [x] AC3：畫面中的用量／quota／providers 數值與 Magpie 原生面板一致（容許刷新延遲）。
- [x] AC4：Magpie 未啟動或 API 失敗時，顯示離線／錯誤狀態且不崩潰。
- [x] AC5：只透過 loopback 存取 Magpie；不輸出敏感資料。
- [x] AC6：視覺與 Magpie 原生面板並排比對時，主要版面與配色一致（人工驗收）。
- [x] AC7：「形象」tab 可為每個狀態指定圖片（webp／png／gif）；狀態切換時立即換圖；未指定時有預設佔位圖；圖片以 contain 完整顯示。
- [x] AC8：收到 route-events 的 `attempt_started`／`first_content`／`fallback`／`failed` 時，「形象」tab 能反映對應狀態；無請求時回到待機。
- [x] AC9：可為不同 agent 設定不同圖片套；某 agent 缺圖時自動退回使用者指定的預設套。
- [x] AC10：多個 agent 同時活躍時，顯示使用者自訂優先序最高的 agent 狀態；全部待機時顯示最後活躍 agent。

## Verification (2026-11-07)

- 自動化：`go test ./...` 27 passed / 6 packages（settings、magpie client、agents、avatar、panel、view）；`go vet ./...`；`bun run typecheck`；`go tool mygo build` 產出 `Magpie Panel.exe` 與 NSIS setup。
- 實機（built exe，Magpie 127.0.0.1:3425）：用量顯示今日 2,304 次／7.3M tokens／$8.33、7 日 5,737 次／$32.63；Grok 與 ZCode quota 卡；Agents 列出 Claude Code／Codex／OpenCode／Pi／VS Code 等與目前模型、強度；快速頁顯示 Magpie 0.1.3193、port 3425；形象頁有 agent／預設套選單與佔位圖；設定頁有主題、語言、置頂、開機啟動、快捷鍵、port。
- 開關：全域快捷鍵 Alt+Shift+M 可隱藏並還原浮窗；其他視窗取得焦點時浮窗仍保持顯示。
- 縮放：拖曳邊角由 440×640 改為 536×748，版面即時重排；正常退出後重啟，`window-state.json` 還原為 536×748 @ (1060,376)。
- 限制：tray icon 點擊未在此環境自動化，但與快捷鍵共用同一個 show/hide toggle（MyGo `Tray.OnClick`）；AC6 為人工視覺驗收，非逐像素比對。

## Out of Scope

- 修改或 fork Magpie 本體（本專案採插件＋外部 MyGo 程式）。
- Magpie 面板的 100% 功能對等（MVP 只做確認過的功能子集）。
- 雲端同步、遠端 Magpie 控制、手機版。
- 首發不含 macOS／Linux 打包。

## Key Decisions

- **D1**：產品形態 B — Magpie 插件＋MyGo 浮窗二進位。
- **D2**：MVP 四項功能全包，另加「形象」tab。
- **D3**：「形象」採 7 態、依 agent 分套、可指定預設套 fallback。
- **D4**：多 agent 同時活躍依使用者自訂優先序；全待機顯示最後活躍 agent。
- **D5**：開機自動啟動、預設置頂、全域快捷鍵、多螢幕記憶全數納入。
- **D6**：二進位由 GitHub Releases 下載，SHA256 驗證，可手動指定本機路徑。

## Evidence

- Magpie 插件文檔：<https://usemagpie.ai/docs/zh/plugins>
- Magpie repo：<https://github.com/yetone/magpie>
- 整合文檔：`docs/integrating.md`
- 面板實作：`internal/gui/app.go`、`internal/gui/pluginpage.go`
- MyGo 文檔：Context7 `/egoist/mygo`
