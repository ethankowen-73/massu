# Watcher Spec — `massu watch` Daemon (Plan 3a)

**Created**: 2026-04-27 (Phase 0 of `/Users/ekoultra/hedge/docs/plans/2026-04-26-massu-3a-watcher-daemon.md`)
**Scope**: Layer B (file-watcher daemon) + Layer C (quiescence detector) + Phase 6 (auto-trigger refresh wiring). No AST parsers, no third-party adapter loading (those land in 3b/3c).

---

## 1. Watch surface

| Always watched | Conditionally watched | Excluded |
|----|----|----|
| `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `Gemfile`, `*.csproj`, `mix.exs`, `requirements*.txt`, `setup.py` | `paths.*_source` from `massu.config.yaml`, OR fallback safe-default globs (`src/**`, `app/**`, `apps/**`, `packages/**`, `lib/**`, `cmd/**`) when `paths.*_source` absent | `node_modules/`, `.venv/`, `target/`, `build/`, `dist/`, `.git/`, `.massu/`, `.claude/` |

Daemon root: `git rev-parse --show-toplevel` (NOT `process.cwd()`).
Default-fallback discovery emits one stderr line at startup: `[massu] watching default globs (paths.*_source unset): src/, app/, apps/, packages/, lib/, cmd/`.

---

## 2. Watcher engine

- **Engine**: chokidar (`^3.6.0`)
- **Why 3.x not 5.x**: chokidar v5 requires `node: ">=20.19.0"` per its package metadata; massu's `engines.node` is `">=18.0.0"` (verified `packages/core/package.json:60`). Bumping to v5 is a breaking change for downstream Node-18 consumers. v4 is also Node-14+ compatible but v3.6.0 is the most conservative pin matching the ecosystem of `proper-lockfile@^4.x`. **Decision**: pin `chokidar@^3.6.0`. Future maintainer who bumps `engines.node` to `>=20.19.0` may upgrade to v5 in lockstep.
- **Types**: chokidar bundles its own TypeScript declarations since v3.4.0 (verified `npm view chokidar types` → `./index.d.ts`). **DO NOT add `@types/chokidar`** — it would install a deprecated stub that conflicts with the bundled types.
- **Critical chokidar option**: `awaitWriteFinish: false` (explicit). Reason: the Layer-C lockfile-detector is the canonical source of "wait for writer to settle" logic; allowing chokidar's own writeFinish to fire after our 3s debounce already elapsed would race the storm-detection counter.

---

## 3. Quiescence algorithm (Layer C)

```
debounce_ms        = 3000   (configurable via watch.debounce_ms)
storm_threshold    = 50     (>50 events in 1000ms → 30s wait)
deep_storm_threshold = 500  (>500 events in 10000ms → 120s wait)
hard_timeout_ms    = 300000 (5 minutes — abort and apply with warning)
lockfile_window_ms = 500    (mtime delta on *.lock means "still being written")
```

Hard stops (refuse to apply):
- `.git/MERGE_HEAD` exists
- `.git/REBASE_HEAD` exists
- `.git/CHERRY_PICK_HEAD` exists

Manual override: `npx massu watch --apply-now` — bypass quiescence, apply immediately (CI use).

### Overlapping-fire mutex (iter-1 + iter-2)

A `runningRefresh` boolean mutex guards the daemon's `fireRefresh()` so two refresh cycles cannot overlap (which would otherwise concurrently call `runConfigRefresh` + `installAll` + `writeStateAtomic` against the same project — race city). When a second refresh attempt arrives while the first is still in flight:

1. **Stderr observability (iter-1)**: the daemon writes `[massu] refresh skipped (previous refresh still running)` so users investigating "why didn't a refresh fire?" can see the skip instead of silent suppression.
2. **Re-arm the debounce (iter-2 correctness fix)**: the skip ALSO calls `scheduleDebounce(debounceMs)` so the deferred refresh definitely fires after the first releases. Without this re-arm the debounce timer was consumed by the skipped attempt and the second refresh would be silently lost until an unrelated event re-armed it. Tests in `__tests__/watch/quiescence.test.ts` cover both behaviors.

### Shutdown semantics (iter-6)

`handle.stop()` clears timers, closes chokidar, and sets `stopped=true`. It does **NOT** await an in-flight `fireRefresh()` for the following reason: every file op the refresh issues is already protected by atomic-rename (`runConfigRefresh`, `installAll` per-file, `writeStateAtomic`) or append-only JSONL (`appendRefreshLog`, where readers tolerate trailing partial lines). If SIGINT/SIGTERM kills the process partway through `installAll`, the atomic-rename guarantees prevent torn writes; the next watcher run completes any remaining files (installAll is idempotent) and the install-lock auto-releases via its 30s stale timeout. Two alternative designs were tried in iter-6 and rejected:

- **Promise-tracking (`inFlightRefresh = fireRefresh()` + `await pending` in stop)**: chained `.then` handlers add microtasks that interact poorly with vitest's `advanceTimersByTimeAsync` in tests where mock `onQuiescent` returns a forever-pending promise. The iter-2 deferred-fire test breaks.
- **Polling (`while (runningRefresh) { ... await sleep }` in stop)**: under fake timers, `setTimeout` is mocked but `Date.now` may be too — the loop hangs.

**Leak guard (iter-6, kept)**: `scheduleDebounce` short-circuits with `if (stopped) return;` so an in-flight refresh that hits a re-arm path (transient error / lockfile mid-write / git mid-op) DURING shutdown does not leak a `setTimeout`. This is the actual leak-prevention concern that IS reachable; the corresponding test (`scheduleDebounce no-ops after stop()`) lives in `__tests__/watch/quiescence.test.ts`. `pushEvent` already had a `stopped` guard from inception.

---

## 4. State file

Path: `<projectRoot>/.massu/watch-state.json`

```json
{
  "schema_version": 1,
  "lastFingerprint": "<sha256 hex>",
  "lastRefreshAt": "<ISO-8601>",
  "lastError": null,
  "daemonPid": 12345,
  "startedAt": "<ISO-8601>",
  "tickedAt": "<ISO-8601>"
}
```

### Schema versioning

- `MIN_SUPPORTED_SCHEMA_VERSION = 1`, `MAX_SUPPORTED_SCHEMA_VERSION = 1` (both = 1 at ship; bump MAX when migrators added).
- Missing field on read → archive to `.massu/watch-state.v0.bak.json` and rebuild from current state.
- `version > MAX` → refuse to start; emit user-friendly error (see VR-USER-ERROR-MESSAGES item 2 of the plan).
- `version < MIN` → run inline migrators in `state.ts`. Migration map `STATE_MIGRATORS: Record<from, (oldState) => newState>` covers every pair `[from..MAX-1]`. At ship, the map is empty.

### Atomic writes

`writeStateAtomic(path, value)`:

```
1. write `<path>.<pid>.<counter>.tmp` with JSON.stringify       # iter-1
2. fsyncSync the tmp fd
3. renameSync(tmp, path)
```

Why `.<pid>.<counter>.tmp` (iter-1 fix): two concurrent writers in the same project (e.g. the foreground daemon plus a `massu watch --apply-now` racing against it, or back-to-back tick + refresh writes within the same process) MUST never share the same temp file. The pid disambiguates across processes; the per-process counter disambiguates back-to-back writes in the same process. POSIX `rename(2)` is atomic per (target) so the final state file always reflects exactly one successful writer — never a torn write. Tests cover (a) sequential-50-write convergence with no straggler `.tmp` files (`state.test.ts`) and (b) `kill -9` at the three points across a refresh cycle.

---

## 5. CLI surface

```
npx massu watch                  # start (registers via claude-bg)
npx massu watch --foreground     # run in current terminal (used by claude-bg internally)
npx massu watch --status         # show running watcher info
npx massu watch --stop           # kill via claude-bg
npx massu watch --apply-now      # bypass quiescence, apply immediately
npx massu refresh-log [N=10]     # show last N auto-refresh events with diffs
```

Plus on the existing `massu config refresh` subcommand:

```
npx massu config refresh --yes   # bypass interactive confirm prompt (alias: -y)
                                 # mirrors --skip-commands plumbing
```

---

## 6. claude-bg decision (binary)

- **Decision**: Option (b) — `claude-bg` is a peer dependency. `massu watch` invokes it via `spawnSync('claude-bg', ['start', '--name', ..., '--port', '0', '--', 'npx', '@massu/core', 'watch', '--foreground', '--root', toplevel])`. If `which claude-bg` returns nothing or spawn returns ENOENT, the wrapper emits a user-friendly error (see VR-USER-ERROR-MESSAGES item 1 of the plan) and exits 1.
- **Why not (a)**: embedding a minimal bg-process registry in `packages/core/src/bg/` would (i) add 4-7h Phase-2 work (per plan estimates), (ii) require keeping the embedded version in sync with `~/.claude/bin/claude-bg` over time, (iii) duplicate the SessionEnd-hook integration that `claude-bg` already provides. The peer-dep path lets us re-use the existing infrastructure that all Hedge consumers already have. Massu's audience overlaps with Hedge's `claude-bg`-installed cohort; the user-friendly error is the docs link.

---

## 7. installAll concurrency

- **Decision**: Option (A) per iter-3 G3-A9 — pass `skipCommands: true` from the watcher's `runConfigRefresh` invocation so the recursive `installAll` call inside `runConfigRefresh` is skipped. The watcher's own outer `installAll(projectRoot)` call (Phase 6 step 5) acquires `<projectRoot>/.massu/installAll.lock` via `proper-lockfile.lockSync`. Single install per refresh, no reentrancy.
- **Why not (B)**: process-level held-lock reentrancy counter (Map<lockPath, count>) is more code, harder to reason about under crashes, and the use case it would solve (callers nesting `installAll`) is exactly what Option A eliminates. Reject.
- **Sync API choice**: `proper-lockfile.lockSync()` + `unlockSync()` so `installAll(projectRoot: string): InstallAllResult` keeps its synchronous signature. CLI dispatcher and the `runConfigRefresh` recursive call site remain unchanged.
- **Block-up-to-30s policy (iter-3 third pass, G3-iter3-2)**: Plan §190 demands "second caller blocks up to 30s, then bails with stderr `installAll already running (PID=X) — try again in <N>s`". `proper-lockfile.lockSync()` REJECTS `retries > 0` in its sync API (verified `node_modules/proper-lockfile/lib/adapter.js`: throws `Cannot use retries with the sync api`). Therefore `lib/installLock.ts` implements a **manual retry loop**: try `lockSync` with `retries: 0`; on `ELOCKED`/`EBUSY`, sleep for `pollIntervalMs` (default 100ms) and re-try until either acquisition succeeds or `blockMs` (default 30s) elapses. Sleep uses `Atomics.wait()` against a `SharedArrayBuffer` for portable synchronous sleep. Tests cover (a) acquire-after-release within block window and (b) bail at deadline.
- **PID disclosure**: `withInstallLock` writes `<lockPath>.pid` containing its own PID after acquisition and removes it on release. Contenders read this sidecar in their `InstallLockBusyError` constructor so the user-facing message satisfies plan §243's `(PID=X)` field; on read failure the message degrades to `(PID=unknown)` rather than throwing.
- **Lock-dir creation**: `mkdirSync(dirname(lockPath), { recursive: true })` MUST run before `lockSync` (per iter-3 G3-A11 — `.massu/` may not exist on a fresh repo). The wrapper at `lib/installLock.ts` does this unconditionally.
- **Cross-platform**: catch both `ELOCKED` (POSIX) and `EBUSY` (Windows) and surface as the same user-facing error.
- **Bail-immediately escape hatch**: callers may pass `{ retries: 0 }` to skip the manual retry loop and bail on first contention (used by tests that want deterministic immediate failures).

---

## 8. Sleep/wake handling (no `power-monitor` package)

`power-monitor` does NOT exist on npm (`npm view power-monitor` returns 404). Use a **tick-gap heuristic** in the daemon's main loop:

```
every 10s:  state.tickedAt = new Date().toISOString(); writeStateAtomic(...);
            if (now - lastTickEpoch > 30_000ms):
                stderr.write('[massu] tick gap detected (likely sleep/wake), reconciling\n');
                forceReconciliation();   # fireRefresh -> onQuiescent -> runDetection
            lastTickEpoch = now;
```

`forceReconciliation()` calls into the same `fireRefresh()` path that the debounce timer uses (clearing the debounce timer first), so a fresh `runDetection()` + fingerprint compare runs unconditionally. This catches any stack change that occurred during sleep — even if chokidar dropped the underlying file events. The implementation does NOT re-add globs to chokidar (`chokidar.add(currentGlobs)`); chokidar re-establishes its FSEvents/inotify subscriptions on its own after wake on macOS/Linux, and the manual fingerprint-compare provides the correctness safety-net regardless.

Tests: synthetic clock-jump test that mocks `Date.now()` advancing by 5min between two ticks; assert reconciliation fires exactly once.

---

## 9. PID liveness probe (cross-platform)

`hooks/session-start.ts` (Phase 6) needs to know if the daemon is still alive before suppressing the drift banner. New helper `lib/pidLiveness.ts`:

| OS | Mechanism | Edge cases |
|----|---|---|
| POSIX (macOS/Linux) | `process.kill(pid, 0)` | ESRCH → dead, EPERM → alive (different user / locked-down process), other errors → bubble up |
| Windows | `spawnSync('tasklist', ['/FI', `PID eq ${pid}`, '/NH'])` and grep for the PID in stdout | best-effort; document the limitation in JSDoc |

Returns `true | false`. Unit tests cover ESRCH (synthetic dead PID), EPERM (synthetic mock), and a Windows-stub test path.

---

## 10. Test plan summary (full set in Phase 2 / Phase 3 / Phase 6)

| Test | Layer | Acceptance |
|---|---|---|
| 100 rapid file writes | Layer C | 1 refresh fires after 3s debounce |
| Lockfile mid-write | Layer C | debounce extends another 3s |
| `npm install` storm (5000 changes / 30s) | Layer C | 1 refresh AFTER install completes + 3s |
| `git checkout` deep storm | Layer C | 2-min wait |
| `.git/MERGE_HEAD` present | Layer C | NEVER applies |
| Process restart | Layer B | reads state file, resumes |
| `kill -9` mid-refresh | Layer B | atomic state write — no corruption at any of 5 points across cycle |
| Tick-gap (sleep/wake) | Layer B | synthetic Date.now jump → 1 forced reconciliation |
| Concurrent refresh + watcher | Phase 6 | exactly one wins per file via `installAll.lock` |
| End-to-end: pyproject.toml mutation | Phase 6 | `.claude/commands/massu-scaffold-router.md` becomes FastAPI variant within 10s |

---

## 11. Watch scope (iter-8)

`watch.scope` (`paths` | `full`) gates the chokidar watch surface:

- `paths` (default) — watch only the declared `paths.*` + `framework.languages.*.source_dirs`, falling back to safe-default globs (`src/**`, `app/**`, `apps/**`, `packages/**`, `lib/**`, `cmd/**`) when both are empty. Bounded watch surface, recommended for huge (>10K-file) repos.
- `full` — watch the entire project root (`'**'`) bounded by the same `DEFAULT_EXCLUSIONS` set. Manifest files are still watched. Opt-in for users on small/medium repos who want every file under toplevel to count. NOT recommended for huge repos — chokidar may peg CPU.

The schema field landed in iter-7 but went unwired through to `deriveWatchGlobs`; iter-8 wires it. Tests in `__tests__/watch/paths.test.ts` cover both scopes.

## 12. Second-daemon refusal (iter-8)

Plan §256 risk #6 demands that a second `massu watch --foreground` on the same toplevel must refuse to start. Iter-8 implements `checkConflictingDaemon(root)` (exported from `commands/watch.ts`) which reads `watch-state.json`, checks the recorded `daemonPid` against `isPidAlive`, and returns a non-null actionable message when another live daemon owns the root. `runForeground` short-circuits on a non-null return with exit 1. Tests live in `__tests__/watch/conflicting-daemon.test.ts` (5 cases: missing state, dead pid, self-pid, live pid, corrupt state).

Without this guard two daemons would each write watch-state.json + refresh-log on every quiescence cycle; the install-lock prevents data corruption but the user gets bursty `installAll already running` stderr spew. The pre-flight peek catches the common case before any side effects.

---

**Hard prerequisite gate (verified at session start 2026-04-27)**:

```
$ ls /Users/ekoultra/massu/packages/core/src/commands/template-engine.ts        # exit 0
$ ls /Users/ekoultra/massu/packages/core/src/detect/codebase-introspector.ts    # exit 0
$ grep -nE 'skipCommands\?:\s*boolean' .../config-refresh.ts                    # 1 match (line 66)
$ grep -cE 'pickVariant' .../install-commands.ts                                 # 8 matches (≥3)
$ bash scripts/massu-pattern-scanner.sh                                          # exit 0
```

All five gates green. Implementation starts at Phase 2.
