# ModForge Managed Agent Docs (CLAUDE.md / AGENTS.md blocks)

ModForge (modman) maintains sentinel-delimited "managed by modman" blocks at the top of each mod's `CLAUDE.md` (full MCP tool guidance, `<!-- BEGIN MODMAN -->` … `<!-- END MODMAN -->`) and `AGENTS.md` (7KB capture rule only, `<!-- BEGIN MODMAN 7KB -->` … `<!-- END MODMAN 7KB -->`). Logic lives in `ModForge/src-tauri/src/claude_md.rs`.

## When the blocks are refreshed (as of 2026-06-10)

The blocks are refreshed **only when a mod is opened**, never as a bulk sweep:

- `claude_stream_open` (`claude_stream.rs`) — Claude chat session starts for the mod
- `codex_stream_open` (`codex_stream.rs`) — Codex session starts for the mod
- `pty_open_for_mod` (`pty.rs`) — terminal session opens in the mod
- `create_mod` (`create_mod.rs`) — new mod scaffold

All four routes go through `claude_md::refresh_for_mod(db, mod_dir)` (or `update_claude_md` directly in `create_mod`), which looks up the configured 7KB path from settings and rewrites only the sentinel block, preserving user content outside it.

**Removed behavior:** ModForge used to run `claude_md::refresh_all` on app boot, rewriting CLAUDE.md/AGENTS.md in *every* registered mod. This was removed deliberately (user requirement: do not touch agent docs of mods that aren't open). Do not reintroduce a boot-time or bulk sweep — unopened mods' files must stay untouched.
