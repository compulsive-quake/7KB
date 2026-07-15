# Verifying Tauri UIs Headlessly (CDP + IPC mock)

How to drive and screenshot a Tauri 2 + vite frontend end-to-end when the
dev machine's desktop is in active use (popping real windows fights the
user for focus and they'll close them mid-test). Worked out on AssettoCar
(2026-07); recipe lives in `AssettoCar/.claude/skills/verify/SKILL.md`.

## Core idea

Run only the vite dev server, load it in **headless Chrome**, and inject a
`window.__TAURI_INTERNALS__` mock **before page load** via CDP
`Page.addScriptToEvaluateOnNewDocument`. Feed the mock **real payloads
dumped from the Rust backend** (an `#[ignore]`d test that serializes probe
results to JSON) so the UI renders real data, not hand-typed fixtures.

The mock needs three members:

- `invoke(cmd, args)` — switch on command name; route `plugin:dialog|open`
  to a canned path (bypasses native dialogs entirely), record mutation
  commands' args on `window` for assertions (e.g. what `generate` was sent).
- `transformCallback(cb)` — assign `window['_' + id] = cb`, return id
  (Tauri's event system delivers via these globals).
- `metadata: { currentWebview: {label:'main'}, currentWindow: {label:'main'} }`
  — required by `getCurrentWebview()` / `onDragDropEvent`.

Simulate backend events (progress/log) by calling registered
`plugin:event|listen` handlers from the mock.

## Driving

CDP over WebSocket (`npm i ws --no-save`, Node 20 lacks global WebSocket):
`Runtime.evaluate` (with `awaitPromise`) for state assertions,
`Input.dispatchMouseEvent` for trusted clicks (needed for autoplay/user
activation), `Page.captureScreenshot` for evidence — works fully
offscreen, no focus stolen.

## Gotchas

- React StrictMode double-mounts effects; if the mock's
  `plugin:event|unlisten` is a no-op, every event renders twice. Mock
  artifact, not an app bug — real unlisten removes the first listener.
- A harness/background-task-owned `npm run tauri dev` dies when the task
  ends; `Start-Process` (detached) if the app must survive tool calls.
- Automating the real app's native file dialogs (UIA/SendKeys) is flaky
  and lands on the user's desktop — avoid; stub at the
  `plugin:dialog|open` boundary instead.
- WebView2 accepts `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS=--remote-debugging-port=9223`
  for CDP against the *real* app when the desktop IS free; same driver
  works, but native dialogs still pop real windows.
- `cargo run --example` linking a tauri lib crate can fail with
  STATUS_ENTRYPOINT_NOT_FOUND on Windows; put dev utilities in
  `#[ignore]`d integration tests instead.
