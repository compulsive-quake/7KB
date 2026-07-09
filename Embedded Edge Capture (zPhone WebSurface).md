# Embedded Edge Capture (zPhone WebSurface)

zPhone renders its React webapp by launching a hidden `msedge.exe --app=<file:///url>` window offscreen (`--window-position=-32000,-32000`), capturing it with Windows Graphics Capture (WGC), and blitting the frames onto an NGUI `UITexture`. Input goes the other way over CDP (`Input.dispatchMouseEvent` / `dispatchKeyEvent`) via the DevTools websocket (`--remote-debugging-port=0`, port discovered from `<user-data-dir>/DevToolsActivePort`).

Key files (zPhone): `src/web/EdgeHost.cs` (process + WGC + flags), `src/web/WebSurfaceController.cs` (XUi controller, UV crop), `src/web/WebSurfaceInput.cs` (NGUI collider → CDP coords), `src/web/WebSurfaceOverlay.cs` (fullscreen IMGUI draw), `src/web/ZphoneBridge.cs` / `CdpClient.cs` (CDP transport).

## Gotcha: Edge chrome height is NOT constant — infobars break fixed-offset cropping

WGC captures the **whole window**, including Edge's chrome above the page:

- The `--app=` mode "Web App Frame Toolbar" (title strip with min/max/close), ~40 px. `WindowFinder.RemoveTitleBar` (WS_CAPTION strip) is ignored by recent Edge for app windows, so it must be UV-cropped out of the texture instead.
- **Any infobar Edge decides to inject under that strip** — e.g. the "It seems that your browser data on this device…" notification (seen 2026-07), sync nags, etc. These appear even with `--no-first-run --disable-sync --disable-features=msImplicitSignIn,EdgeSyncSetup,…`, and there is no documented reliable `--disable-features` flag for all of them (the auto-import one is only controllable via the `AutoImportAtFirstRun` *policy*, which is registry-wide and marks the user's real Edge "managed" — don't).

**Symptom when chrome height is assumed constant:** the page content shifts down inside the captured texture, so the displayed UI and the click mapping disagree — the user must aim ~a banner-height *above* a button to hit it.

**Fix (implemented 2026-07):** measure instead of guessing. Infobars are browser chrome — they shrink `window.innerHeight` and fire a `resize`. The React app reports `window.innerWidth/innerHeight` to the host over the bridge (`viewport` action) on bridge-up and on every resize; the host then:

1. computes `chromePx = capturedTexture.height − reportedInnerHeight × DEVICE_SCALE` and sets the `UITexture.uvRect` crop from that (re-checked each frame in the bind loop, applied only on change) — `EdgeHost.ChromeUvFraction(tex)`;
2. maps widget → page CSS coordinates using the *reported* viewport, not compile-time constants (`WebSurfaceInput.ReportedPageW/H`). CDP mouse coords are viewport-relative, so this alone re-aligns clicks.

Result: any banner appearing or disappearing mid-session self-corrects; the banner itself never shows in-game because it's cropped with the rest of the chrome (content is slightly vertically compressed while a banner is up — 16:9 widget showing a shorter viewport — which is barely visible and beats broken clicks).

## Other Edge-host gotchas (already handled in EdgeHost.cs, keep in mind for new hosts)

- `--user-data-dir` must be **unique per spawn attempt** — reusing a dir makes Edge relay to the prior window and exit, tripping "process exited" respawn loops.
- Cap respawns (cooldown + budget); `EnsureRunning` is called every frame.
- Assign Edge to a Job Object with `KILL_ON_JOB_CLOSE` so a game crash can't orphan it.
- Disable Chromium occlusion/background throttling flags or the offscreen window stops painting.
- `--allow-file-access-from-files` needed for a Vite bundle loaded from `file:///`.
- WGC frames are top-row-first; NGUI path uses `Flip.Vertically`, IMGUI path uses a negative-height UV rect. Chrome rows land at the **low end of V** — crop with `yMin = chromeFraction`.
