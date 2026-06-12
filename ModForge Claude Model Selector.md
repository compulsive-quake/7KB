# ModForge Claude Model Selector

How the per-mod Claude model picker works and what to update when Anthropic ships a new model.

## How it works

- ModForge stores a CLI **alias** per mod (`mods.claude_model` in SQLite), not a full model ID. `"default"` means no `--model` flag (claude picks the account default); other values are passed verbatim as `claude --model <alias>`.
- Because aliases (`opus`, `sonnet`, `haiku`, `fable`) track the latest model of each tier on Anthropic's side, point releases (e.g. Opus 4.7 → 4.8) only require updating the **display descriptions**, not the values.

## Sync points when a new model/tier ships

1. `src/types.ts` — `ClaudeModel` union + `CLAUDE_MODELS` array (new alias only).
2. `src-tauri/src/mods.rs` — `VALID_CLAUDE_MODELS` (backend validation; must match types.ts or `claude_stream_set_model` rejects the pick).
3. `src/components/ChatInput.tsx` — `MODEL_META` (picker labels/descriptions; update version numbers here for point releases).
4. `src-tauri/src/claude_stream.rs` — doc comment on `claude_stream_set_model` listing the values.

Verify a new alias with `claude --model <alias> --print "Reply with just: ok"` before shipping it.

## Gotcha: stale installed builds black-screen on unknown aliases

The dev build and the installed build share the same SQLite DB (`%APPDATA%\modforge\modforge.sqlite`). If a dev build persists a new alias (e.g. `fable`) and the installed release predates the matching `MODEL_META` entry, the installed app throws `Cannot read properties of undefined (reading 'label')` in ChatInput when the mod's tab is restored at startup — React unmounts the whole tree and the app shows a **solid black screen** (the `bg-zinc-950` body). Seen 2026-06-11 with v1.0.20 installed + `fable` saved on a mod.

Fixes: ship a release containing the new alias before using it in dev, and/or keep `MODEL_META[model]` lookups fallback-safe (`?? MODEL_META.default`). To diagnose this class of crash in an installed build, see [[Debugging Installed ModForge Builds]] (Debugging Installed ModForge Builds.md).

Last updated 2026-06-11: added `fable` (Fable 5 tier), bumped Opus description 4.7 → 4.8, documented the stale-build black-screen gotcha.
