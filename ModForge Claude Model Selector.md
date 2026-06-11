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

Last updated 2026-06-10: added `fable` (Fable 5 tier) and bumped Opus description 4.7 → 4.8.
