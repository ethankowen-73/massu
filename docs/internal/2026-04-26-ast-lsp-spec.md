# Plan 3b — AST + LSP Introspection Spec (Internal Design Doc)

**Status**: Phase 0 deliverable (AUDITED — implementation-ready)
**Scope**: Internal design doc for massu maintainers authoring first-party adapters.
**Audience**: Future massu contributors. Public-facing guide ships in Phase 9 to `massu.ai/docs`.
**Plan reference**: `/Users/ekoultra/hedge/docs/plans/2026-04-26-massu-3b-ast-lsp-introspection.md`

---

## 1. Vendor Decisions (LOCKED — Phase 0)

| Concern | Decision | Rationale |
|---|---|---|
| AST engine | `web-tree-sitter@^0.26.8` | One binding, 18+ languages, incremental re-parse, error-resilient, WASM keeps install footprint predictable |
| LSP client base | `vscode-languageserver-protocol@^3.17.5` | Reference TypeScript types + JSON-RPC layer for the LSP spec; battle-tested |
| WASM grammar packaging | **Strategy A** (download-on-first-use, SHA-256 verified, cached at `~/.massu/wasm-cache/`) | Strategy B (peer-dep packages) breaks zero-touch install |
| Native grammar bundling | **REJECTED** | Native deps break npm portability across platforms |
| `tree-sitter-wasms` umbrella package | **BANNED** | Plan line 122 explicit ban — Strategy A precludes it |

### Strategy A vs Strategy B (rationale)

**Strategy A — Download on first use**:
- Tarball stays small (`@massu/core` ~ 1MB, budgeted ≤ 5MB ceiling)
- User pays one-time cost when they first introspect a given language
- SHA-256 manifest hardcoded in source — tampered downloads refuse to load
- Cache at `~/.massu/wasm-cache/<lang>-<sha>.wasm` — survives reinstalls

**Strategy B — Peer-dep `@massu/grammars-<lang>` packages** (REJECTED for v1):
- Forces 18 peer-dep declarations onto user `package.json`
- Breaks Plan #2's zero-touch install promise
- Switching to B in the future requires a new ADR

---

## 2. Adapter Contract

Lives at `packages/core/src/detect/adapters/types.ts`.

```ts
export interface CodebaseAdapter {
  /** Stable adapter id, e.g. "python-fastapi" */
  id: string;
  /** Languages this adapter consumes */
  languages: TreeSitterLanguage[];
  /** Returns true if any signal in the codebase suggests this adapter applies */
  matches(signals: DetectionSignals): boolean;
  /** Sample N files, run AST queries, return extracted conventions */
  introspect(files: SourceFile[], rootDir: string): Promise<AdapterResult>;
}

export interface AdapterResult {
  conventions: Record<string, unknown>;  // becomes detected.<adapter.id> in massu.config.yaml
  provenance: Provenance[];              // for each field: which file/line/query produced it
  confidence: 'high' | 'medium' | 'low' | 'none'; // 'none' returns null; 'low' triggers warning
}
```

### Type Origins

All types are LOCAL to `packages/core/src/detect/adapters/types.ts` — none are re-exported from `web-tree-sitter` (that package exposes `Language` as a class, not a string union).

| Type | Origin | Definition |
|---|---|---|
| `TreeSitterLanguage` | Local string-literal union | `'python' \| 'typescript' \| 'javascript' \| 'swift' \| 'rust' \| 'go' \| 'ruby' \| 'php' \| 'java' \| 'kotlin' \| 'elixir' \| 'erlang' \| 'csharp' \| 'cpp' \| 'haskell' \| 'ocaml'` |
| `SourceFile` | Local interface | `{ path: string; content: string; language: TreeSitterLanguage; size: number; }` |
| `DetectionSignals` | Local interface | `{ packageJson?: Record<string, unknown>; pyprojectToml?: Record<string, unknown>; gemfile?: string; cargoToml?: Record<string, unknown>; goMod?: string; presentDirs: Set<string>; presentFiles: Set<string>; }` |
| `Provenance` | Local interface | `{ field: string; sourceFile: string; line: number; query: string; }` |

VR check: after Phase 1 completes, `grep -nE 'export (interface\|type) (CodebaseAdapter\|AdapterResult\|TreeSitterLanguage\|SourceFile\|DetectionSignals\|Provenance)' packages/core/src/detect/adapters/types.ts` MUST return ≥6 hits.

### Per-field confidence (not per-adapter)

A single weak field MUST NOT poison the rest of the adapter's output. The provenance trail (file path, line, query name) is written to `detected.<adapter>._provenance` so the user can audit any extracted value.

### Confidence semantics

| Level | Meaning | Behavior |
|---|---|---|
| `high` | Single canonical match, query produced exactly one result | Field written, no warning |
| `medium` | Multiple matches but all agree | Field written, no warning |
| `low` | Multiple matches, ambiguous; OR single match in test/example file | Field written, warning to stderr |
| `none` | No matches, OR adapter timed out, OR adapter threw | Field returns `null`; field omitted from `detected.<adapter>` block |

---

## 3. Query Authoring Guide

Adapters use Tree-sitter S-expression queries instead of regex. A single query can replace dozens of regex patterns.

### Example: FastAPI auth dependency

```scheme
;; Catches Depends(get_current_user), Depends(require_tier_or_guardian), etc.
;; Regardless of import alias, line wrapping, or formatting.
(call
  function: (attribute
    object: (identifier) @_depends
    attribute: (identifier) @_attr (#eq? @_attr "Depends"))
  arguments: (argument_list
    (identifier) @auth_dep))
(#eq? @_depends "Depends")
```

### Authoring checklist

1. **Anchor the query on the unambiguous node** — pick the specific function/attribute the framework defines, not a user-overridable identifier.
2. **Always use predicate constraints** (`#eq?`, `#match?`) — bare captures match too liberally.
3. **Capture into named groups** — `@auth_dep` is the field that ends up in `detected.<adapter>.auth_dep`.
4. **Write a fixture FIRST** — a 5-line synthetic file with the pattern, run the query in the Tree-sitter playground, verify capture before committing.
5. **Include adversarial fixtures** — a file with no match (returns 0 captures), a file with the pattern in a comment/string (must not capture), a file with the pattern under conditional imports.
6. **Time-box** — every query MUST complete within 2s on a single 5MB file. Larger files are pre-skipped at the runner layer.

### Query-helpers wrapper

`query-helpers.ts` exposes:

- `compileQuery(language: Language, source: string): Query` — caches compiled queries per `(language, source)` tuple
- `runQuery(query: Query, tree: Tree): Match[]` — returns named captures with line numbers
- `safeRunQuery(query, tree, timeoutMs): Match[] | 'timeout'` — wrapped with the 2s deadline

Adapters consume the helpers, never the raw `web-tree-sitter` API. This keeps the surface area minimal and testable.

---

## 4. LSP Integration

Lives at `packages/core/src/lsp/`.

### Method subset

The LSP client implements the minimum subset required for adapter enrichment:

| Method | Purpose |
|---|---|
| `initialize` | Capability handshake, capture `ServerCapabilities` |
| `textDocument/documentSymbol` | Per-file symbol tree — used to refine AST captures |
| `workspace/symbol` | Cross-file symbol search — resolve aliases (e.g. `Depends` → `fastapi.Depends`) |
| `textDocument/definition` | Jump-to-def — disambiguates user-defined vs framework-defined symbols |
| `shutdown` | Clean termination |

### Per-server method-support matrix

Not every LSP implements the full subset uniformly. The client gracefully handles `MethodNotFound` (`error.code: -32601`) per-method-per-server and downgrades that single capability without aborting the connection.

| LSP Server | initialize | documentSymbol | workspace/symbol | definition | Notes |
|---|---|---|---|---|---|
| Pyright | yes | yes | yes | yes | Full coverage |
| typescript-language-server | yes | yes | yes | yes | Full coverage |
| rust-analyzer | yes | yes | yes | yes | Full coverage |
| gopls | yes | yes | yes | yes | Full coverage |
| sourcekit-lsp | yes | yes | partial | yes | `workspace/symbol` is indexer-state-dependent — empty result is INCONCLUSIVE, not negative |

On `initialize`, the client captures the server's `ServerCapabilities` and short-circuits any method whose `*Provider` capability is false/absent — the request is never sent.

### Connection lifecycle

1. Read `massu.config.yaml.lsp.servers[]`
2. For each server with `command`, spawn child process with stdin/stdout pipes (no shell — argv array form, see Phase 3.5 #4)
3. Send `initialize`, capture capabilities, send `initialized` notification
4. Per-adapter enrichment: send method requests in parallel, validate every response against Zod schema
5. On any failure (timeout, validation, MethodNotFound, garbage payload, oversize): degrade silently, keep AST result
6. On graceful shutdown: send `shutdown` then `exit`, kill process if no response within 2s

### Failure mode contract

| Mode | Behavior | Stderr line? |
|---|---|---|
| LSP timeout (5s) | Skip enrichment for that field, keep AST | yes (info) |
| LSP returns garbage | Reject via Zod, fallback to AST | yes (warning) |
| LSP version mismatch | Skip that field, keep others | yes (warning) |
| LSP unreachable | Skip server entirely, all-AST fallback | yes (info) |
| `enabled: true` AND `servers: []` AND `autoDetect.viaPortScan: false` | Log info, proceed AST-only | yes (info) |

---

## 5. Fallback Ordering

Per-field resolution order:

```
AST adapter (high/medium/low confidence)
    -> if 'none', try LSP enrichment
    -> if LSP unavailable or returns nothing, fall back to regex (regex-fallback.ts)
    -> if regex returns nothing, field is null
```

When BOTH AST and regex produce a value for the same field: **AST wins**. Regex is the last-resort fallback. The provenance trail records both results so an auditor can see what the regex would have said.

LSP is **enrichment**, not requirement. AST adapter results are authoritative; LSP only refines them (e.g., resolving the symbol's actual import origin).

---

## 6. Adapter Registry (Phase 1 v1)

For v1, `runner.ts` ONLY discovers first-party adapters from `packages/core/src/detect/adapters/`. User-authored adapters are **deferred to Plan 3c** (the adapter registry plan).

The `CodebaseAdapter` contract IS exported from `@massu/core` and IS pluggable in principle, but `runner.ts` does NOT load user-authored adapters from `massu.config.yaml.adapters.custom: []` or any other discovery path at v1.

If a user wants to extend the adapter set in v1, the documented path is "fork @massu/core and add an adapter file" — not config-driven plug-in.

### v1 first-party adapters

| Adapter id | File | Replaces (Plan #2 regex) |
|---|---|---|
| `python-fastapi` | `python-fastapi.ts` | `introspectPython` (FastAPI subset) |
| `python-django` | `python-django.ts` | `introspectPython` (Django subset) |
| `nextjs-trpc` | `nextjs-trpc.ts` | `introspectTypeScript` (tRPC subset) |
| `swift-swiftui` | `swift-swiftui.ts` | `introspectSwift` |

Plan #2's regex functions move VERBATIM into `regex-fallback.ts`. No regex logic changes — only the import path.

---

## 7. Integration with Existing `introspect()`

`packages/core/src/detect/codebase-introspector.ts` keeps its existing public signature:

```ts
export function introspect(result, projectRoot): DetectedConventions
```

so `detect/index.ts` is **byte-for-byte unchanged**. Internally, `introspect()` becomes a 2-tier dispatcher:

1. For each detected language, run any matching AST adapters via `runner.ts`
2. For fields the AST adapters return as `none`/null, fall back to the existing per-language regex functions (now in `regex-fallback.ts`)

`runner.ts` writes `detected.<adapter.id>` blocks ALONGSIDE (not replacing) the existing `detected.python` / `detected.swift` / `detected.typescript` blocks.

The current helper `sampleFiles` (with the `pathFilter?: (absPath, basename) => boolean` signature added by commit `ef67485`) moves with the regex helpers into `regex-fallback.ts`. All 3 call sites that pass `(absPath, name) =>` parent-dir matchers (`introspectPython`, `introspectSwift`, `introspectTypeScript`) move with it.

---

## 8. Strategy A WASM Packaging (LOCKED)

### Lifecycle

1. `tree-sitter-loader.ts` is asked for grammar `<lang>` (e.g. `python`)
2. Check `~/.massu/wasm-cache/<lang>-<sha>.wasm` — if present and SHA-256 matches manifest, load it
3. Otherwise: download from pinned URL (e.g. `https://unpkg.com/tree-sitter-<lang>-wasm@<pinned-version>/tree-sitter-<lang>.wasm`)
4. Verify SHA-256 against hardcoded manifest in source
5. If verified: write to `~/.massu/wasm-cache/<lang>-<sha>.wasm` (write-then-rename atomically) and load
6. If verification fails: REFUSE to load, emit VR-USER-ERROR-MESSAGES item-1 stderr line, degrade to regex fallback
7. If download fails AND no cached grammar exists: degrade to regex fallback (offline-tolerant)

### Manifest format

Hardcoded in `tree-sitter-loader.ts` as a constant:

```ts
const GRAMMAR_MANIFEST: Record<TreeSitterLanguage, { url: string; sha256: string; version: string }> = {
  python: { url: '...', sha256: '...', version: '...' },
  // ... 18 entries
};
```

The manifest is **never** network-fetched. Tampering the manifest requires modifying the source code, which requires a release.

### Atomicity

All cache writes use write-then-rename:

1. Write to `~/.massu/wasm-cache/<lang>-<sha>.wasm.tmp.<pid>`
2. `fs.renameSync(tmp, final)` — atomic on POSIX

This prevents partial-write corruption from concurrent introspections.

### Offline behavior

If `fs.readFile` of cache fails AND `fetch` fails:

- Emit stderr: `"AST grammar for <language> failed to load — falling back to regex introspection for files in <language>. Trace: <one-line>"`
- Continue with regex-fallback for that language
- Other languages with cached grammars are unaffected

### npm tarball constraint

`npm pack @massu/core && du -sh massu-core-*.tgz` MUST stay ≤ 5MB (5120K). Baseline at HEAD `ef67485` is 920K. The 5MB ceiling provides ~4.2MB of headroom for non-grammar additions.

ZERO files matching `*.wasm` or `node_modules/tree-sitter-*` may appear in the tarball (Strategy A forbids bundled grammars). VR-PACKAGE-SIZE in Phase 9 enforces this.

---

## 9. Adapter Registration Pattern (v1 first-party)

Each first-party adapter file follows the same shape:

```ts
// packages/core/src/detect/adapters/python-fastapi.ts
import type { CodebaseAdapter, AdapterResult, DetectionSignals, SourceFile } from './types.ts';

export const pythonFastApiAdapter: CodebaseAdapter = {
  id: 'python-fastapi',
  languages: ['python'],

  matches(signals: DetectionSignals): boolean {
    // Cheap signal check — no file IO
    return Boolean(signals.pyprojectToml?.dependencies?.fastapi)
        || Boolean(signals.presentDirs.has('routers'));
  },

  async introspect(files: SourceFile[], rootDir: string): Promise<AdapterResult> {
    // Run S-expression queries via query-helpers
    // Return AdapterResult with conventions + provenance + confidence
  },
};
```

`runner.ts` imports each adapter from a static list — no dynamic discovery.

---

## 10. Security Notes (reference Phase 3.5)

Phase 3.5 owns the deep security audit. This spec only references the four enumerated attack surfaces:

1. **AST parser DoS surface** — Tree-sitter consumes arbitrary user source. Mitigations: per-file timeout (2s), per-file size limit (5MB), error-tolerant grammar.
2. **LSP IPC trust boundary** — JSON-RPC over stdio with external process. Mitigations: Zod validation on every response, AST authoritative.
3. **WASM-grammar-load network surface** (Strategy A consequence) — Mitigations: SHA-256 manifest hardcoded, TLS-only fetch, write-then-rename, refuse on hash mismatch.
4. **Config-driven command execution** — `lsp.servers[].command` MUST be an `[argv0, ...args]` array (no shell), absolute path required (or explicit opt-in).

Full enumeration with severities, repros, and verifications lives in `docs/internal/2026-04-26-ast-lsp-security-audit.md` (Phase 3.5 deliverable).

---

## 11. Test Coverage Policy (Phase 1+4)

In addition to per-adapter fixture tests:

| Module | Test file | Coverage |
|---|---|---|
| `runner.ts` | `runner.test.ts` | Confidence merging, AST-wins rule, `'none'` drops field, per-adapter try/catch isolation |
| `tree-sitter-loader.ts` | `tree-sitter-loader.test.ts` | Fresh download + SHA verify, SHA mismatch refusal, cache hit no-fetch, offline degrades to regex |
| `query-helpers.ts` | `query-helpers.test.ts` | API surface, invalid S-expression throws typed error |
| `lsp/client.ts` | `client.test.ts` | Per-server method-support matrix (one test per LSP server) |
| `lsp/auto-detect.ts` | `auto-detect.test.ts` | `viaPortScan: false` default-off invariant runtime check |

VR check after Phase 1+4: `find packages/core/src/__tests__ -name "runner.test.ts" -o -name "tree-sitter-loader.test.ts" -o -name "query-helpers.test.ts" -o -name "client.test.ts" -o -name "auto-detect.test.ts" | wc -l` MUST return ≥5.

---

## 12. Config Schema Extension (Phase 4)

`packages/core/src/config.ts` gets a new top-level `lsp` block:

```yaml
lsp:
  enabled: false  # default
  servers:
    - language: python
      command: ["pyright-langserver", "--stdio"]  # array form, no shell
  autoDetect:
    viaPortScan: false  # default false — security-sensitive opt-in
```

Zod schema (named anchors, not line numbers — survives refactors):

1. `LSPConfigSchema` named const declared above `RawConfigSchema`. `.passthrough()`-ed.
2. `RawConfigSchema = z.object({...})` block — add `lsp: LSPConfigSchema.optional()` IMMEDIATELY AFTER `watch:` field.
3. `export interface Config { ... }` — add `lsp?: LSPConfig` IMMEDIATELY AFTER existing `watch?: WatchConfig;` field.
4. `getConfig()` final `_config = { ... }` assembly — add `lsp: parsed.lsp` propagation alongside `watch: parsed.watch,`.

Validation: `grep -nE 'lsp[?:]' packages/core/src/config.ts` MUST return ≥4 hits after Phase 4.

---

## 13. Deferred Decisions (NOT in v1)

| Decision | Deferred to | Rationale |
|---|---|---|
| User-authored adapter discovery | Plan 3c | Adapter registry needs its own design |
| Strategy B (peer-dep grammar packages) | Future ADR | Strategy A is sufficient for v1 |
| Erlang and OCaml adapter authoring | Post-v1 | Lower demand; grammars enumerated in license doc but no first-party adapters |
| Automatic LSP discovery via `lsof` port scan | Plan 3d (TBD) | Security-sensitive default — opt-in only at v1 |

---

## References

- Plan: `/Users/ekoultra/hedge/docs/plans/2026-04-26-massu-3b-ast-lsp-introspection.md`
- Grammar license enumeration: `docs/internal/2026-04-26-ast-lsp-grammar-licenses.md`
- Phase 3.5 security audit (forthcoming): `docs/internal/2026-04-26-ast-lsp-security-audit.md`
- Tree-sitter docs: https://tree-sitter.github.io/tree-sitter/
- LSP spec 3.17: https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/
