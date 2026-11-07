# MyGo framework — research notes

日期：2026-11-07
來源：Context7 `/egoist/mygo`、<https://github.com/egoist/mygo> docs。

## 基本

- Go 1.27+；web frontend 需 Bun；Windows 需 WebView2（Win11 內建）。
- `mygo init` 產生 Go module + Vite/TS frontend；`bun run dev`、`bun run build`。
- `mygo build` 產出：Windows `App.exe` + NSIS installer；macOS `.app` + dmg；Linux binary + deb。
- 無 cgo，可交叉編譯：`mygo build -platform windows/amd64`。

## 視窗

- `mygo.NewWindow(WindowOptions{...})`：
  - `Panel`、`AlwaysOnTop`、`Frameless`、`Transparent`、`Vibrancy`、`CornerRadius`；
  - `Width/Height`、`MinWidth/MinHeight`、`MaxWidth/MaxHeight`；
  - `DisableResize`（預設可縮放；frameless 仍可從邊緣縮放）；
  - `StateKey` 記住位置與大小；
  - `FitContent` 隨內容調整；
  - `HideOnBlur` 可在 panel 上使用（本專案不自動隱藏）。
- 事件：`OnClose`、`OnClosed`、`OnFocus`、`OnBlur`、`OnShow`、`OnHide`、`OnResize`、`OnMove`。
- `win.Page().LoadURL`、`LoadHTML`、`LoadFile`；`mygo.Protocol.HandleFunc` 自訂 scheme。

## 桌面能力

- Tray：`mygo.NewTray(TrayOptions{Icon, IconIsTemplate, ToolTip, Menu})`；`OnClick`、`OnRightClick`。
- 全域快捷鍵：`mygo.GlobalShortcut.Register("Alt+Shift+M", fn)`。
- 開機啟動：`mygo.App.SetOpenAtLogin(true)`、`App.OpenAtLogin()`、`App.WasOpenedAtLogin()`。
- Single instance：`mygo.App.RequestSingleInstanceLock()`、`OnSecondInstance`。
- 檔案對話框：`mygo.Dialog`（系統對話框；方法會 block，從 bound method 或 goroutine 呼叫）。
- 路徑：`mygo.App.Path(mygo.PathUserData)` 等。
- 螢幕：`mygo.Screen`（顯示器、工作區、游標所在螢幕）。

## 前端

- bound methods 由 `mygo generate` 產生 `src/mygo.ts`（typed）。
- events：`mygo.NewEvent[T](name)`；前端 `events.name.on(...)`。
- `mygo-runtime` 提供 `currentWindow`、`isMyGo()`、`onFileDrop`、`preferences()`、locale。
- 深色模式：`prefers-color-scheme`、`color-scheme: light dark`、`vibrancy`。
- 自訂 scheme 的頁面可呼叫 Go；第三方 HTML 要用 `PageOptions.Untrusted`。

## 對本專案的含義

- 浮窗用 web frontend（HTML/CSS 最容易復刻 Magpie 面板）。
- 使用者圖片用 `avatar://` custom scheme 從 `PathUserData` 供應。
- 選檔用 `mygo.Dialog`；複製檔案到 app data。
- 開機啟動、single instance、全域快捷鍵、多螢幕 clamp 都有原生 API。
