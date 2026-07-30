# ModForge Deploy & Drift Tracking

Part of the [7DTD Modding Knowledgebase](README.md). How modman decides what a
deploy needs (game shutdown or not) and the failure modes when that tracking
goes wrong. Learned debugging a "deploy script is not working" report on the
Elevator mod (2026-07).

## The kind counters drive the shutdown preflight

`.modforge-state.json` holds three monotonic counters (`restart` / `xui` /
`assets`). Deploy status = source counters vs the copy of the state file in
the deployed folder. Two writers bump them:

- **modman's source watcher** (`source_watch.rs` → `deploy_status::classify_in_mod`),
  live fs events;
- **the per-mod Claude Stop hook** (`.claude/modforge-bump-id.cjs`), a
  git-status-based safety net at end of each Claude turn.

The deploy modal **skips the game-shutdown preflight when only `xui`/`assets`
drift is pending**. That makes misclassification dangerous, not just cosmetic.

## Failure mode: deploy fails with robocopy exit 8/9 while the game runs

A loaded mod DLL is **memory-mapped** by the running game. Overwriting it
fails with Windows error **1224 "user-mapped section open"** — note a plain
`[IO.File]::Open(path, 'Open', 'ReadWrite', 'None')` **succeeds** on such a
file, so an open-for-write probe is NOT a valid lock test; only the actual
overwrite fails. robocopy retries then reports exit ≥ 8 (9 = "some copied,
some failed") and `deploy.ps1` throws.

So: if code changed but the drift shows `assets`-only, modman deploys without
shutting the game down → robocopy hits 1224 on the DLL → deploy "doesn't
work". Fix the tracking, or shut the game down (`mcp__modman__shutdown_game`)
before deploying.

## Failure mode: Stop hook can't find git → everything becomes `assets`

Hook processes spawned from the modman GUI can carry a minimal `PATH` where a
bare `git` (or `node`) doesn't resolve. The original hook swallowed the error
and bumped `assets` as the "least disruptive" fallback — which is exactly
wrong for `.cs` changes, because it lets the deploy skip the game shutdown
(see above). Symptom signature: `assets` counter +1 per Claude turn while
`restart` never moves despite `.cs` edits.

Hardened hook (Elevator's `.claude/modforge-bump-id.cjs`, worth porting to
the modman template):

- try `git`, then `C:\Program Files\Git\cmd\git.exe` / `bin\git.exe`;
- on total failure, write the reason + `PATH` to
  `.claude/modforge-bump-id.log` and bump **restart** (over-flagging costs
  one unnecessary restart; under-flagging breaks deploys);
- git OK + clean tree → bump nothing (previously bumped `assets` for every
  no-change turn).

## Reproducing a modforge deploy outside modman

`deploy.rs` runs `powershell.exe -NoProfile -NonInteractive -ExecutionPolicy
Bypass -File deploy.ps1` with cwd = mod root and env injected: `GameDir`,
`SEVEN_DTD_CLIENT`, `MODFORGE_MOD_DIR`, `MODFORGE_MOD_NAME`, `MODFORGE_MOD_ID`,
`MODMAN_MCP_URL`/`MODMAN_MCP_TOKEN`. Deploy output streams to the UI only —
there is no log file — so to see a script error, run the script with those
env vars set. On this machine the game is at
`D:\SteamLibrary\steamapps\common\7 Days To Die` (NOT the `C:\Program Files
(x86)` default the scripts fall back to).

## Open modman issues spotted (2026-07)

- The source watcher did not bump `restart` for `.cs` edits during a live
  session (counters only moved via the Stop hook fallback) — why the watcher
  missed the events is unverified; the Rust classifier itself is correct.
- The deploy flow could detect robocopy 1224/exit≥8 on a `.dll` and offer
  "shut down the game and retry" instead of failing.

## Deploy queue UI bookkeeping

The batch modal intentionally keeps completed mod IDs in `deployBatch.modIds`
so successful logs remain available until the modal closes. Do not use that
array alone as the toolbar's pending count: record successful IDs separately
(`deployCompleted`) and exclude them from the badge, resetting that list when
the batch closes or a fresh batch starts. Re-enqueueing the same mod during an
open batch must remove it from `deployCompleted` so the new request runs and
counts again. Completed IDs must override every contribution to the badge,
including cached yellow/red drift status, not only `deployBatch.modIds`.
On success, explicitly refresh `deployStatusGet` because the status event and
its asynchronous frontend handler can race the deploy-exit event. This fixed
successful deploys appearing stuck in the queue (2026-07-22).

## `deployable` lives in the repo, not just SQLite (2026-07-28)

The per-mod **Deployable** toggle (Mod Settings) is stored in **`modforge.json`
at the mod root**, committed with the mod, and mirrored into the `mods.deployable`
column. SQLite is per-machine, so a second ModForge install cloning the same repo
would otherwise fall back to the column default (`1`).

- `src-tauri/src/mod_config.rs` owns the file: `read_deployable`,
  `write_deployable` (preserves unknown keys — `version` is also read from this
  file by `mods::compute_mod_version`), `write_initial` (new scaffolds).
- `mods::list_mods` calls `mod_config::sync_deployable_from_disk` first, so a
  value another instance committed applies as soon as the repo is pulled.
  **The config file wins over the local DB whenever it has a `deployable` key.**
- Mods with no `modforge.json` keep their local value — the file is only seeded
  when Mod Settings is saved, or at `create_mod` scaffold time. Existing mods
  need one save to start sharing the setting.
- `modforge.json` is in the ignore lists of **both** classifiers
  (`deploy_status::classify_relative` and the Claude Stop hook's `classify`),
  so flipping the toggle does not bump a drift counter. New scaffolds also
  exclude it from the `deploy.ps1` robocopy.
- A malformed/non-object `modforge.json` makes the write fail rather than
  clobber the file; the toggle still saves locally and the error says so.
