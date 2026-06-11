# Claude CLI Session Resume Gotchas

Learned while fixing ModForge's "No conversation found with session ID: …" error on launch (2026-06-10).

## The trap

- In stream-json mode, the claude CLI tags **every** stdout line with a `session_id`, including the very first `system/init` event of a brand-new session.
- But the CLI only writes the conversation to disk once a turn actually happens. A session that was opened and never chatted in has a session ID that resolves to nothing.
- Passing that phantom ID to `claude --resume <id>` fails with `No conversation found with session ID: <id>` on stderr and the process exits.

## Rules for harnesses that auto-resume

1. Persist a session ID for later `--resume` only from **turn events** (`type` ∈ `assistant` / `user` / `result`), never from `system/init` or `rate_limit_event` lines.
2. Treat `--resume` failure as recoverable: detect the stderr message, clear the stored ID, and respawn without the flag. IDs can also go stale when the CLI's history is pruned externally, so the recovery path is needed even with rule 1.

## Where this lives in ModForge

- `src-tauri/src/claude_stream.rs` — stdout thread persists `last_session_id` only on turn events; stderr thread detects the resume failure, NULLs `mods.last_session_id`, and emits `claude:resume_failed:{mod_id}`.
- `src/components/StreamView.tsx` — listens for `claude:resume_failed`, calls `claudeStreamReset` + `claudeStreamOpen` (no resume id) to recover seamlessly.
