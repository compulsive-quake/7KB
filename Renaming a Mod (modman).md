# Renaming a Mod (modman-managed)

How to rename a mod end-to-end (folder, GitHub repo, assembly, ModInfo), learned
renaming FogErasor → FogEraser on 2026-07-03.

## Checklist

1. **Contents first**: replace the old name in `ModInfo.xml` (`Name` +
   `DisplayName`), `src/*.csproj` (`AssemblyName`, `RootNamespace`), all C#
   namespaces/classes, `build.ps1`/`deploy.ps1`, `.gitignore` (the `/<Mod>.dll`
   entry), README/plan/CLAUDE.md. `deploy.ps1` derives the deploy target from
   `MODFORGE_MOD_NAME` / the folder leaf name, so it usually needs no edit
   beyond comments.
2. **File renames** via `git mv` (csproj, `<Mod>Hud.cs` etc.) so history follows.
3. **Delete stale build artifacts** (`src/bin`, `src/obj`, old `<Mod>.dll` in the
   mod root) — they carry the old assembly name and would otherwise get deployed.
4. **GitHub repo**: `gh repo rename <NewName> --yes` run inside the repo renames
   the remote repo AND updates the local `origin` URL. Gotcha: gh rewrites the
   remote in **SSH form** (`git@github.com:...`) even if it was https — run
   `git remote set-url origin https://github.com/<owner>/<NewName>` afterwards
   if the setup uses https auth. GitHub redirects the old URL indefinitely.
5. **Old deployed copy**: the game's `Mods/<OldName>` folder lingers; delete it
   or it will double-load alongside the next deploy of `Mods/<NewName>`.

## Gotcha: the folder itself cannot be renamed from inside a modman session

Windows refuses to rename a directory that is any process's current working
directory (cwd handles are opened without `FILE_SHARE_DELETE`). A modman-managed
Claude session runs as `modman.exe → cmd.exe → claude.exe → pwsh.exe` with the
**cwd of claude/cmd/pwsh all inside the mod folder**, so the rename always fails
with "being used by another process" while the session is open. File-watcher
handles (FileSystemWatcher) do NOT block renames — it's specifically the cwd.

To find the blockers, enumerate process cwds (read each PEB's
`ProcessParameters->CurrentDirectory` via `NtQueryInformationProcess` +
`ReadProcessMemory`; works unelevated for same-user processes).

**Fix**: close the Claude session (and any terminal/IDE sitting in the folder),
then `Rename-Item <old> <NewName>` from the parent dir. Scheduling a rename to
fire after session exit (schtasks watcher) is blocked by Claude's permission
classifier as unauthorized persistence — don't bother; just leave the rename as
the user's last manual step.

After the rename, modman still points at the old path — the mod must be
re-registered/re-pointed in modman (the gitignored `.modforge-id` in the folder
preserves its identity), and a fresh deploy recreates `Mods/<NewName>`.
