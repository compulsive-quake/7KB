# Mod Runtime Memory Attribution

7DTD loads XML/Harmony/modlet code into the same Unity process, so OS process monitors can report memory for `7DaysToDie.exe` or the dedicated server, but cannot accurately split that heap by loaded mod.

For ModForge, process-level telemetry is still useful: `src-tauri/src/game.rs` already exposes `GameProcessInfo.memoryBytes` through `sysinfo` for client/server/crash-handler rows.

Per-mod runtime memory needs cooperative in-game instrumentation:

- Cheap estimate: sample total process memory before/after deploy/relaunch combinations and flag deltas. This is noisy because Unity caches assets and GC timing is nondeterministic.
- Better for managed Harmony mods: ship a shared diagnostic helper that each mod can call around known allocation points, emit counters to the 7debug/log channel, and let ModForge attribute those counters by mod ID.
- Better for assets: have each mod optionally report loaded bundle names, texture/audio/mesh counts, and approximate byte sizes when it loads/unloads assets.
- Deep profiling: use Unity/CLR profiling APIs or snapshots for investigation, but do not expect low-overhead always-on exact per-mod RAM accounting from outside the process.

Practical UI recommendation: show exact per-process RAM in Game Instances, plus an optional "mod memory signals" panel that shows cooperative estimates/counters and labels them as approximate.
