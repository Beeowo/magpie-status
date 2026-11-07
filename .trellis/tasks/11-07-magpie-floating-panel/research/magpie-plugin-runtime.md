# Magpie plugin runtime — research notes

日期：2026-11-07
來源：<https://usemagpie.ai/docs/zh/plugins>、`yetone/magpie` repo。

## 能力邊界

- 插件是 npm 包（或 git／本地資料夾），由 Magpie 載入 Bun 宿主；以 stdin/stdout 逐行 JSON 與主程式通訊。
- UI 只能是：
  - `magpie.page`：Magpie 視窗側邊欄的沙箱 iframe；
  - `magpie.panels`：嵌入 Agents／Usage／Settings／provider 頁；
  - `magpie.actions`：provider／agent 選單項目。
- 頁面 CSP：`sandbox allow-scripts`、`connect-src 'none'`；不能 `fetch`／WebSocket／表單。
- 頁面可讀：`usage:summary`、`providers:list`；可讀寫自己的 plugin options（每秒最多一次）。
- **沒有** tray、原生浮窗、commands、一般 views API。

## 頁面與宿主生命週期

- 只有頁面的插件啟動時不跑 Bun host（in-magpie-only）。
- 有 host 部分的插件，頁面 HTML 被服務時會 `plugin.Warm` 啟動宿主（`internal/gui/pluginpage.go` 的 `pageHost`／`warmHost`）。
- 宿主是 Bun 進程；repo 內部測試以 `Bun.spawn` 啟動子進程（`internal/plugin/host_tree_test.go`、`reap_linux_test.go`）。
- 插件被安裝／更新／設定變更時宿主會 restart。

## 安裝與測試

```bash
magpie plugin add ./my-plugin
magpie plugin add github:owner/repo
magpie plugin --json
```

沙箱測試：

```bash
sb=$(mktemp -d)
HOME=$sb XDG_CONFIG_HOME=$sb/.config magpie plugin add ./my-plugin
```

## 對本專案的含義

- 插件不能擁有浮窗，浮窗必須是獨立 MyGo 進程。
- 插件 host 負責：確保 binary、下載、驗 SHA256、detached spawn。
- 插件頁只能做狀態顯示與 options 設定；不能直接呼叫 MyGo app。
- app 應自行 `SetOpenAtLogin`，避免依賴宿主何時被喚醒。
