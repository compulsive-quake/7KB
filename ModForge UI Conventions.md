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

- Codex config `service_tier = "priority"` is obsolete for current Codex builds; startup fails with `unknown variant 'priority', expected 'fast' or 'flex'`. Use `service_tier = "fast"` as the direct replacement, and have ModForge sanitize that line when it edits `~/.codex/config.toml` for project trust.
- ModForge's Codex model picker maps UI labels to Codex config overrides: `GPT-5.5` -> `model = "gpt-5.5"`, `Extra High` reasoning -> `model_reasoning_effort = "xhigh"`, and speed choices -> `service_tier = "fast"` or `"flex"`. Apply these as `codex exec -c ...` overrides instead of relying on the user's global config file.

## Knowledge Base panel

- The Knowledge Base settings must distinguish the repository URL from the local checkout folder. `kb_repo_path` is the local checkout folder used for indexing/syncing; the GitHub URL is only an input to clone.
- The Knowledge Base panel has separate local refresh and git sync actions. Refresh only re-indexes configured KB files; Sync runs `git pull --ff-only` then `git push` against the configured `kb_repo_path` and should show an in-flight status plus the git result text.
- Put Clone/Explorer/Sync next to the local `Knowledge Base Checkout Folder` path in Settings, not in the reader panel, so users can create, inspect, or fix the actual checkout before reading notes.
- Agent sessions must get the KB checkout from ModForge settings, never by guessing `../7KB`. Add configured `kb_repo_path` to Codex workspace-write roots, include it in Codex per-turn prompts, and stamp it into managed per-mod `CLAUDE.md`/`AGENTS.md` blocks.
- The Knowledge Base opens as a full-window top-right toolbar section (`section="kb"`), like Settings/Game/Mod Manager. Do not expose it as a left activity-rail side panel.
- The right dock cluster (Run panel, Action Log panel, and right activity rail) is only visible in the main Mods workspace. Hide it in top-right full-window sections such as Knowledge Base, Settings, Game, and Mod Manager.

## Installed mod manager

- The Mod Manager section should list installed client mods from `<7DTD client>/Mods`, not just registered ModForge source mods. Match installed folders back to registered mods by `ModInfo.xml` `<Name value="...">` first, then by source/folder name.
- Installed freshness should reuse ModForge deploy status semantics: green is synced, yellow is marker mismatch, red is missing/never-deployed marker, and unknown is unmanaged/unmatched.
- Per-row deploy actions in the Mod Manager should enqueue the matched registered source mod through the existing deploy queue. Unmanaged installed folders belong in a separate lower "Other Client Mods" list with no deploy action.
- Disable moves the installed folder to `<7DTD client>/mods_disabled`; delete permanently removes the installed folder with no trash/recycle step.
