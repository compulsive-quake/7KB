# ModForge Claude Model Selector

How the per-mod Claude model + effort picker works. Since 2026-07-27 the model
list is **fetched from Anthropic at runtime**, so a new model release no longer
needs a ModForge release.

## How it works

- `src-tauri/src/claude_models.rs` exposes `claude_models_list(force)` →
  `GET https://api.anthropic.com/v1/models?limit=100`. Returns per model: `id`,
  `display_name`, `max_input_tokens`, `max_tokens`, and
  `capabilities.effort.{low,medium,high,xhigh,max}.supported`.
- ModForge stores a **full model id** per mod (`mods.claude_model`), e.g.
  `claude-opus-5`, and an effort level (`mods.claude_effort`). `"default"` on
  either means "don't pass the flag". Everything else goes to the CLI verbatim
  as `claude --model <id> --effort <level>`.
- Backend validation is a **character-class check** (`is_valid_claude_model`),
  not an allowlist — an allowlist would reintroduce the per-release sync.
- Results are cached to `%APPDATA%\modforge\claude-models.json` (6 h TTL).
  Degradation ladder: fresh API → stale cache → small built-in list, with
  `source`/`error` surfaced in the picker so a fallback is visible, not silent.

### Auth: reuse Claude Code's own OAuth token

The endpoint needs credentials, but ModForge users sign in through the `claude`
CLI, not an API key. Resolution order:

1. `ANTHROPIC_API_KEY` → `x-api-key` header
2. `ANTHROPIC_AUTH_TOKEN` → `Authorization: Bearer`
3. **`~/.claude/.credentials.json` → `claudeAiOauth.accessToken`** (an
   `sk-ant-oat01-…`) → `Authorization: Bearer` **plus the required
   `anthropic-beta: oauth-2025-04-20` header**

(3) is the normal path and is verified working with a Max subscription (scopes
include `user:inference`). `claudeAiOauth.expiresAt` is **epoch milliseconds** —
check it before use or you just earn a 401. Note the whole file is also where
MCP OAuth tokens live; only read the `claudeAiOauth` key.

Verify locally with the ignored smoke test, which prints the live table:

```
cd src-tauri && cargo test --lib claude_models -- --ignored --nocapture
```

## Effort has no live-switch path

The CLI's control-request protocol (the JSON lines ModForge writes to claude's
stdin) supports these subtypes — dumped from the 2.1.220 binary:

```
file_suggestions  mcp_authenticate  get_context_usage  rename_session
set_model  set_max_thinking_tokens  mcp_oauth_callback_url
set_permission_mode  mcp_status  mcp_reconnect  set_color  get_usage
```

**There is no `set_effort`.** Effort is read once, from the `--effort` launch
flag. So changing model or permission mode mid-session is a control request,
but changing effort requires a respawn: `claude_stream_set_effort` returns
whether a session is live, and ChatInput then calls
`resetStream(modId, mod.lastSessionId)` — the same restart-with-`--resume` path
the sessions modal's Resume button uses, so the transcript carries over.

## CLI flag facts (verified 2.1.220, 2026-07-27)

- `--model` accepts a bare alias (`opus`) **or** a full id (`claude-opus-5`).
- `--effort` takes `low | medium | high | xhigh | max`.
- **Unsupported combinations do not error.** `--model claude-haiku-4-5 --effort high`
  (Haiku reports no effort capability) and `--model claude-opus-4-6 --effort xhigh`
  (4.6 predates xhigh) both run fine — the CLI ignores the flag. So a stale
  effort value saved against a different model can't break a spawn; no clamping
  logic is needed.
- An unknown model fails gracefully with a message, not a crash:
  `There's an issue with the selected model (…). It may not exist or you may not have access to it.`

## Gotcha: stale installed builds black-screen on unknown model values

The dev build and the installed build share one SQLite DB
(`%APPDATA%\modforge\modforge.sqlite`). Under the **old** hardcoded design, a
dev build persisting a value the installed release didn't know (e.g. `fable`)
made `MODEL_META[model].label` throw in ChatInput, React unmounted the tree, and
the installed app showed a **solid black screen**. Seen 2026-06-11 with v1.0.20.

The dynamic picker is immune by construction — every lookup falls back
(`resolveSelected` uses the raw id when the catalog has no entry;
`EFFORT_LABELS[v] ?? v`), and a stored-but-unlisted model still renders its own
row so it can't silently vanish while remaining what claude actually runs. Keep
that property when editing the picker. Diagnosing this class of crash in an
installed build: [[Debugging Installed ModForge Builds]] (Debugging Installed ModForge Builds.md).

## Gotcha: WebView2 CDP did not attach

`WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS=--remote-debugging-port=9223` did **not**
open a CDP port against the ModForge dev build on 2026-07-27 (Windows 11,
WebView2 current) — the port stayed closed whether the var was exported through
`npm run tauri` or set in PowerShell around a direct `Start-Process` of
`dist/cargo-target/debug/modman.exe`. Fall back to the vite + headless-Chrome +
`__TAURI_INTERNALS__` mock recipe in
[[Verifying Tauri UIs Headlessly (CDP + IPC mock)]] (Verifying Tauri UIs Headlessly (CDP + IPC mock).md),
which needs the whole boot IPC surface stubbed but doesn't depend on WebView2
honoring the flag.

Also: killing a dev `modman.exe` takes the harness-owned `npm run tauri` task
down with it (exit 127), and the leftover vite keeps port 1420 bound — free it
before relaunching or the next `tauri dev` dies with `Port 1420 is already in use`.

## Sync points (what's left)

Nothing per-model. Only if the **CLI's effort vocabulary** changes:
`EFFORT_ORDER` in `claude_models.rs`, `VALID_CLAUDE_EFFORTS` in `mods.rs`,
`CLAUDE_EFFORT_ORDER` + `ClaudeEffort` in `src/types.ts`, and `EFFORT_LABELS`
in `ChatInput.tsx`.

Last updated 2026-07-27: replaced the hardcoded alias list with the live
`/v1/models` catalog, added per-mod effort, documented the OAuth-token reuse,
the missing `set_effort` control request, and the CLI's tolerance of
unsupported model/effort pairs.
