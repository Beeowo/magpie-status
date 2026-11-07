# Integration Contracts

External contracts this repo depends on that are not obvious from the code alone.
Treat each as a compatibility boundary: changing a dependency's version or shape
means re-verifying the anchors below.

## Magpie gateway (loopback only)

- Discover the port with `GET /api/hello`; default is `127.0.0.1:3425`. The port
  can be overridden in Magpie's `settings.json` (`$XDG_CONFIG_HOME/magpie/`,
  `~/.config/magpie/`, or `os.UserConfigDir()`); `app/internal/magpie` reads it.
- Read-only data endpoints used by the panel: `/v1/models`,
  `/v1/magpie/quotas`, `/v1/magpie/usage?period=today|7d|30d`,
  `/v1/magpie/usage/requests?limit=8`.
- Route events are two-step: fetch `/v1/magpie/route-events/snapshot` for the
  current active requests and `cursor`, then stream
  `/v1/magpie/route-events?after=<cursor>` (SSE, `event: route`). A `409`
  `snapshot_required` means the cursor is stale: re-snapshot before retrying.
  Never invent a cursor.
- Keep every request on loopback. Never send Magpie keys to another host and
  never log prompts, accounts, or secrets.

## Agent and model control

- Magpie's GUI internal API is auth-gated; do not try to call it.
- Use the CLI instead: `magpie ls`, `magpie models <agent>`,
  `magpie <agent> <model>`, `magpie <agent> effort <level>` (some agents use
  `thinking` instead of `effort`). Display names are accepted.
- Validate a model against `magpie models <agent>` before switching, and run
  commands with a hidden console (`CREATE_NO_WINDOW`) on Windows.

## Avatar state and fallback

- Seven states: `offline`, `idle`, `routing`, `waiting`, `streaming`,
  `fallback`, `failed`. Each agent may have a full set; `avatar.defaultSet`
  supplies missing images.
- Resolution order: agent image for the state, then default-set image, then the
  built-in placeholder. All lookups go through `app/internal/avatar`; never read
  arbitrary paths (normalize and reject traversal).
- Images live under the user data directory and are served through the
  `avatar://` scheme (`http://avatar.localhost/...` on Windows). Cache-bust
  with the file mtime.

## Window, tray, and lifecycle

- The panel is a frameless, transparent, always-on-top MyGo window with
  `StateKey` persistence, a minimum size, and tray `OnClick` toggling. Closing
  hides instead of quitting; quitting is explicit.
- The app holds `RequestSingleInstanceLock`; a second launch focuses the
  existing window.
- Global hotkey registration is transactional: if the new hotkey fails, restore
  the previous one and report the error.
- After changing the Magpie port or binary, reconnect the data client instead of
  restarting the app.

## Plugin host and page

- The plugin page runs in a sandboxed iframe (`connect-src 'none'`); it cannot
  open windows or fetch. Everything reaches it through `_magpie/bridge.js`
  (`magpie.options.get/set`, `magpie.read("usage:summary")`).
- The Bun host starts the panel binary. Lookup order: `binaryPath` option, NSIS
  install directory, downloaded versions, then GitHub Releases.
- Only download over HTTPS, verify SHA256 against `SHA256SUMS` before
  extraction, stage the result, and atomically move it into
  `bin/<version>/`. Keep the previous version when anything fails.
- Page options are limited to about one write per second; the page merges the
  current options before writing.
- `magpie.options.set` restarts the host. The host starts again only when the
  page HTML is served, so the page reloads itself after a launch request.
- `launchAt` is honored only when fresh (two minutes), so stale values do not
  launch the app on every host reload.

## Release artifact contract

- A `v*` tag triggers `.github/workflows/release.yml`.
- `magpie-panel-windows-amd64.zip` contains `magpie-panel.exe` at the root and
  `resources/` beside it.
- `SHA256SUMS` uses `<lowercase hex>  <filename>` lines and covers every
  published file except itself.
- The NSIS installer is published as `*-setup-*.exe` and is never used by the
  plugin's updater.
