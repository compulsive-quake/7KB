# ModForge UI Conventions

Shared conventions for the ModForge desktop app UI.

## Dialog scale

- Standard compact dialogs should match the app's small operational UI: about `w-[460px] max-w-[92vw]`, `rounded-lg`, `p-5`, `text-sm` body copy, and `text-xl` titles.
- Avoid oversized modal styling such as `text-3xl` titles, `text-lg` form text, very large radii, or wide shells unless the dialog is intentionally a large workspace.

## Prompt queue

- Prompts submitted while an agent is thinking belong only in the prompt queue UI above the prompt bar. Do not emit them into the agent feed until they actually dequeue and send.

## Deploy queue marker

- Codex deploy requests use a root-level `.modforge-deploy` trigger file in the mod folder. ModForge consumes and deletes this marker immediately after the non-recursive per-mod watcher sees it, so the file often will not remain visible in Explorer; judge success by the deploy queue/badge/modal, not by marker persistence.
- The `.modforge-deploy` watcher is the agent deploy-request channel and should stay enabled for open mods even when recursive workspace/source file watching is disabled.

## Codex launch compatibility

- Agent binary settings should keep the editable text field visible and place discovery actions beside it. Claude Code uses `settings_discover_claude_binary` to search PATH/Get-Command for `claude`; Codex uses `settings_discover_codex_binary` plus its Windows app-package fallbacks.
- Codex config `service_tier = "priority"` is obsolete for current Codex builds; startup fails with `unknown variant 'priority', expected 'fast' or 'flex'`. Use `service_tier = "fast"` as the direct replacement, and have ModForge sanitize that line when it edits `~/.codex/config.toml` for project trust.
- ModForge's Codex model picker maps UI labels to Codex config overrides: `GPT-5.5` -> `model = "gpt-5.5"`, `Extra High` reasoning -> `model_reasoning_effort = "xhigh"`, and speed choices -> `service_tier = "fast"` or `"flex"`. Apply these as `codex exec -c ...` overrides instead of relying on the user's global config file.

## Knowledge Base panel

- The Knowledge Base settings must distinguish the repository URL from the local checkout folder. `kb_repo_path` is the local checkout folder used for indexing/syncing; the GitHub URL is only an input to clone.
- The Knowledge Base panel has separate local refresh and git sync actions. Refresh only re-indexes configured KB files; Sync runs `git pull --rebase --autostash` then `git push` against the configured `kb_repo_path` and should show an in-flight status plus the git result text.
- Put Clone/Explorer/Sync next to the local `Knowledge Base Checkout Folder` path in Settings, not in the reader panel, so users can create, inspect, or fix the actual checkout before reading notes.
- Agent sessions must get the KB checkout from ModForge settings, never by guessing `../7KB`. Add configured `kb_repo_path` to Codex workspace-write roots, include it in Codex per-turn prompts, and stamp it into managed per-mod `CLAUDE.md`/`AGENTS.md` blocks.
- Do not update a mod's `CLAUDE.md` or `AGENTS.md` unless that mod is currently opened in ModForge. This prevents background or unrelated mod folders from receiving managed-agent-file edits.
- The Knowledge Base opens as a full-window top-right toolbar section (`section="kb"`), like Settings/Game/Mod Manager. Do not expose it as a left activity-rail side panel.
- Dirty Knowledge Base git status is surfaced globally: poll `kb_status`, show an amber count badge on the top-right KB toolbar icon, and show the changed-file list plus a `Commit & Push` button in the KB reader. The commit action stages all KB changes and uses the generic message `more knowledge` before pushing.
- KB `Commit & Push` must pull before committing to avoid non-fast-forward push rejections: use `git pull --rebase --autostash`, then `git add -A`, `git commit -m "more knowledge"`, then `git push`. Settings expose a KB auto-pull timer (`kb_auto_pull_enabled`, `kb_auto_pull_interval_minutes`, `kb_auto_pull_pause_unfocused`) that runs the same pull command periodically. If a KB pull/commit hits conflicts, the KB reader should offer to queue a repair prompt to the currently selected mod agent, asking it to resolve conflicts in the configured KB checkout and then commit/push the KB.
- The right dock cluster (Run panel, Terminal panel, Action Log panel, and right activity rail) is only visible in the main Mods workspace. Hide it in top-right full-window sections such as Knowledge Base, Settings, Game, and Mod Manager.

## Right activity bar panels

- The right activity rail (`RightActivityBar.tsx`) has two groups separated by a `flex-1` spacer (WebStorm tool-window stripe layout): a top group of right-docked side panels — Run (`useRuns`), Action Log (`useActionLog`), and Game Log (`useGameLog`) — and a bottom group of full-width bottom-docked tool windows — Terminal (`useTerminal`). Each panel has its own Zustand store with `panelOpen`/`setPanelOpen`/`togglePanel`.
- Right-dock mutual exclusion: Run, Action Log, and Game Log share the right dock, so opening one closes the others (enforced at the rail click AND in `RunPanel`'s `panelOpen` effect because `startRun` opens it without going through the rail). The Terminal is on a separate bottom dock and is independent — it can be open alongside a right-docked panel; do NOT couple it into the right-dock exclusion.
- Game Log panel (`GameLogPanel.tsx`, `gameLogStore.ts`): streams the in-game console live from 7debug's `GET /api/console/stream` SSE endpoint while the panel is open (one `EventSource`, no polling; auto-reconnects when the game is down). Rows click-to-copy; header has copy-all and clear. Clear sets a seq watermark so replayed backlog stays hidden; a changed per-process `run` id in the events resets the watermarks (game restarted). Requires 7debug ≥ 1.2.0.
- Layout (`App.tsx`): everything left of the rail is a vertical `flex flex-col` stack — the section content row (with the right-docked Run/Action Log panels) on top, and `<TerminalPanel />` docked along the bottom. `<RightActivityBar />` stays full-height as a sibling on the far right. Opening the Terminal pushes the content above it up rather than stealing horizontal space.
- Right-docked panels: `shrink-0 flex ... border-l border-zinc-800` aside, 1px left drag handle resizing leftward (`window.innerWidth - clientX - 48`), width persisted to `localStorage` under `modman.<panel>PanelWidth`. Bottom-docked Terminal: `shrink-0 flex flex-col ... border-t border-zinc-800`, 1px top drag handle resizing upward (`window.innerHeight - clientY - 24`), height persisted under `modman.terminalPanelHeight`.
- The Terminal embeds a plain interactive PowerShell (xterm) cwd'd to the active mod's root. The backend (`shell_term.rs`, `ShellRegistry`) keeps one persistent shell per mod id, keyed by `mod_id`: `shell_open` is reattach-aware (returns existing scrollback, spawns fresh only when none), and events stream over `shell:data:{modId}` / `shell:exit:{modId}`. This mirrors `pty.rs` (agent PTY) but launches `pwsh`/`powershell -NoLogo` instead of Claude/Codex. The panel follows the active mod tab via `modIdFromTab(activeTab)` (falling back to `mods[0]`), same as the Run panel's `defaultModId`.

## Git commit author identity

- ModForge stores a default git identity in Settings (`git_user_name` / `git_user_email`, Git tab). When both are set, `git_commit` applies them inline via `git -c user.name=… -c user.email=… commit` (see `commit_changes` in `git_status.rs`) so commits work on any mod even when the repo/global git config has no identity. This is non-mutating — it does not write to any repo's config.
- If a commit fails with an "author identity unknown" style error (detected by `isGitIdentityError` in `LeafPaneView.tsx`) and no default is set, the commit dialog shows `GitIdentityDialog`. It offers name/email plus an "This mod" vs "All mods" toggle: "All mods" saves the ModForge default (so it never asks again), "This mod" calls the `git_set_local_identity` command which writes `user.name`/`user.email` to that repo's local git config only. On success it retries the pending commit with the same message.

## Adding a global setting (end-to-end)

A new global setting touches, in order: `Settings` struct in `src-tauri/src/settings.rs` → read it in BOTH `settings_get_raw` and `settings_get` → write it in `settings_save` (stored as a string KV row in the `settings` table) → `Settings` interface in `src/types.ts` (camelCase) → a control in the relevant `SettingsPanel.tsx` tab using `draft.<field>` / `update("<field>", …)`. The Zustand store passes the whole `Settings` object through `saveSettings`, so no store change is needed for a plain field.

## Beta feature toggles

- Settings has a `Beta` tab (before Updates in `SettingsPanel.tsx`) that collects experimental feature toggles. Beta toggles default to enabled (`bool_default_true`) so existing behavior is unchanged until a user opts out.
- `webapp_viewer_enabled` / `webappViewerEnabled`: gates the webapp review dialog, change toast (`App.tsx`), and the per-mod Webapp editor button (`LeafPaneView.tsx`). When disabled, the `/webapp-review` route in `mcp.rs` (`handle_webapp_review`) short-circuits to an `approve` decision before parking the PostToolUse hook, so agent webapp edits are auto-approved with no dialog.
- `sketchfab_assets_enabled` / `sketchfabAssetsEnabled`: gates the Sketchfab Assets toolbar button (`Toolbar.tsx`) and the `assets` section route; `App.tsx` bounces `section === "assets"` back to `mods` if the toggle is off so the user is never stranded on a blank pane.

## Installed mod manager

- The Mod Manager section should list installed client mods from `<7DTD client>/Mods`, not just registered ModForge source mods. Match installed folders back to registered mods by `ModInfo.xml` `<Name value="...">` first, then by source/folder name.
- Installed freshness should reuse ModForge deploy status semantics: green is synced, yellow is marker mismatch, red is missing/never-deployed marker, and unknown is unmanaged/unmatched.
- Per-row deploy actions in the Mod Manager are exact-mod actions: run `deploy_run` for only the matched registered source mod, do not use the shared deploy queue/batch, then refresh that installed row after `deploy:exit:{modId}`. Unmanaged installed folders belong in a separate lower "Other Client Mods" list with no deploy action.
- Disable moves the installed folder to `<7DTD client>/mods_disabled`; delete permanently removes the installed folder with no trash/recycle step.
- The Mod Manager also lists disabled mods (from `<7DTD client>/mods_disabled`) in a separate lower "Disabled Mods" table, each row with an Enable and a Delete action. Enable moves the folder back to `<7DTD client>/Mods` (via `unique_dest` so it never clobbers an existing folder); Delete permanently removes the disabled folder with no trash step. Backed by `mod_manager_list_disabled` / `mod_manager_enable_disabled` / `mod_manager_delete_disabled` in `mod_manager.rs`, using a `disabled_mod_path` path-safety guard that mirrors `installed_mod_path`. The disabled list only shows folder name + version (no deploy-status/source matching).
