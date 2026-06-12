# Debugging Installed ModForge Builds

How to read console errors from the **installed** (release) ModForge build, where devtools are compiled out (no `devtools` feature in `src-tauri/Cargo.toml`, so F12 does nothing).

## WebView2 remote debugging

WebView2 honors an env var for extra browser args. Launch the installed exe with a remote-debugging port:

```powershell
$env:WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS="--remote-debugging-port=9222"
Start-Process "C:\Program Files\Upgrayedd\Mod Forge\modman.exe"
```

Then `http://127.0.0.1:9222/json` lists debug targets. Either open the `devtoolsFrontendUrl` in a browser, or attach to `webSocketDebuggerUrl` with a small CDP script (`Runtime.enable` + `Page.reload`, listen for `Runtime.exceptionThrown` / `Runtime.consoleAPICalled`). Node 20 needs `node --experimental-websocket` for the global `WebSocket`.

Kill the app afterwards — leaving a release build running with an open debug port is undesirable.

## Facts that matter when diagnosing

- Installed path: `C:\Program Files\Upgrayedd\Mod Forge\modman.exe` (NSIS per-machine; publisher folder "Upgrayedd"). Process name: `modman`.
- Installed and dev builds share `%APPDATA%\modforge\modforge.sqlite`, so data written by a newer dev build can crash an older installed build (see [[ModForge Claude Model Selector]] — unknown model alias → black screen).
- A solid black screen usually means an uncaught render exception unmounted the React root: the empty `<body class="bg-zinc-950">` is near-black. The startup progress overlay appearing briefly on right-click → Refresh confirms the bundle loads and the crash happens during/after workspace restore (`hydrateTabs`).

Created 2026-06-11 while diagnosing the v1.0.20 black screen.
