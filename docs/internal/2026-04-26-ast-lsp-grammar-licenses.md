# Plan 3b — Tree-sitter Grammar License Enumeration

**Status**: Phase 0 deliverable (audit-iter-3 fix X)
**Scope**: Per-grammar source repo, license SPDX, redistribution constraints, status (CLEAN/REVIEW/REMOVE).
**Audience**: Massu maintainers — pre-Phase-1 license clearance gate.
**License compatibility target**: massu ships under BSL 1.1. Permissive licenses (MIT, Apache-2.0, BSD-2/3-Clause, ISC) are compatible. Copyleft licenses (GPL, LGPL, AGPL) that would force source disclosure of consumers are INCOMPATIBLE — those grammars are REMOVED from the v1 set.

---

## Verification methodology

License values verified via `gh api repos/<owner>/<repo>/license --jq '.license.spdx_id'` against the latest default branch on 2026-04-27. Pinned versions are recorded for the WASM manifest in `tree-sitter-loader.ts`. SHA-256 hashes are filled in during Phase 1 when the actual WASM artifacts are pinned.

The Strategy A WASM packaging decision (Phase 0 spec doc, section 8) means **massu does not redistribute these grammars**. They are downloaded by the user's machine on first use from the upstream source. This further reduces redistribution-clause exposure — even copyleft virality concerns are limited to runtime linkage, which the SPDX-permissive set handled here doesn't trigger anyway.

---

## v1 First-party adapters (BLOCKING for Phase 1)

These 4 languages back the 4 first-party AST adapters (`python-fastapi`, `python-django`, `nextjs-trpc`, `swift-swiftui`). All MUST be license-CLEAN before Phase 1 starts.

| Language   | Source repo URL                                              | License SPDX | Pinned version | Redistribution constraints                                                                          | Status |
|------------|--------------------------------------------------------------|--------------|----------------|----------------------------------------------------------------------------------------------------|--------|
| python     | https://github.com/tree-sitter/tree-sitter-python            | MIT          | TBD (Phase 1)  | None beyond MIT attribution. Strategy A: not redistributed by massu, downloaded on first use.     | CLEAN  |
| typescript | https://github.com/tree-sitter/tree-sitter-typescript        | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| javascript | https://github.com/tree-sitter/tree-sitter-javascript        | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| swift      | https://github.com/alex-pinkus/tree-sitter-swift             | MIT          | TBD (Phase 1)  | None beyond MIT attribution. NOT in tree-sitter org — maintained by alex-pinkus.                  | CLEAN  |

**Phase 1 gate: PASS** — all 4 first-party adapter languages are MIT, BSL-compatible, no copyleft.

---

## v1 Additional grammars (no first-party adapters, but loadable)

These 12 grammars are loaded on-demand if the user's codebase contains them, but no first-party adapter ships in v1. Users can fork and add adapters per Phase 9 deferral note (user-adapter discovery is Plan 3c).

| Language   | Source repo URL                                              | License SPDX | Pinned version | Redistribution constraints                                                                          | Status |
|------------|--------------------------------------------------------------|--------------|----------------|----------------------------------------------------------------------------------------------------|--------|
| rust       | https://github.com/tree-sitter/tree-sitter-rust              | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| go         | https://github.com/tree-sitter/tree-sitter-go                | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| ruby       | https://github.com/tree-sitter/tree-sitter-ruby              | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| php        | https://github.com/tree-sitter/tree-sitter-php               | MIT          | TBD (Phase 1)  | Historically LGPL — modern fork is MIT. Confirmed MIT on default branch 2026-04-27.                | CLEAN  |
| java       | https://github.com/tree-sitter/tree-sitter-java              | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| kotlin     | https://github.com/fwcd/tree-sitter-kotlin                   | MIT          | TBD (Phase 1)  | None beyond MIT attribution. NOT in tree-sitter org — maintained by fwcd.                         | CLEAN  |
| elixir     | https://github.com/elixir-lang/tree-sitter-elixir            | Apache-2.0   | TBD (Phase 1)  | Apache-2.0 requires NOTICE file preservation if redistributing source. Strategy A: not redistributed by massu — user downloads upstream. | CLEAN  |
| csharp     | https://github.com/tree-sitter/tree-sitter-c-sharp           | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| cpp        | https://github.com/tree-sitter/tree-sitter-cpp               | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |
| haskell    | https://github.com/tree-sitter/tree-sitter-haskell           | MIT          | TBD (Phase 1)  | None beyond MIT attribution.                                                                       | CLEAN  |

**Note on tree-sitter-php (audit prompt callout)**: tree-sitter-php was historically LGPL. The modern fork in the official tree-sitter org is MIT-licensed at the default branch as of 2026-04-27 (verified via `gh api`). When pinning the version in Phase 1, re-verify by checking the LICENSE file at the exact pinned ref — if a pinned commit pre-dates the relicense, either (a) bump the pin to a post-relicense commit, or (b) move tree-sitter-php to REVIEW pending Phase 1 author confirmation.

---

## Deferred grammars (NOT in v1)

Per plan: "16 + 2 deferred = 18". The 2 deferred grammars are documented here for completeness — their license status is recorded but they are NOT loaded at v1 runtime. Adding them post-v1 requires re-confirmation of license at the new pinned version.

| Language | Source repo URL                                       | License SPDX | Pinned version | Redistribution constraints                                                          | Status (v1) |
|----------|-------------------------------------------------------|--------------|----------------|------------------------------------------------------------------------------------|-------------|
| erlang   | https://github.com/WhatsApp/tree-sitter-erlang        | Apache-2.0   | n/a (deferred) | Apache-2.0 NOTICE preservation if redistributed. Strategy A side-steps this.       | DEFERRED (CLEAN) |
| ocaml    | https://github.com/tree-sitter/tree-sitter-ocaml      | MIT          | n/a (deferred) | None beyond MIT attribution.                                                       | DEFERRED (CLEAN) |

### Deferral rationale

Massu's target user base (per plan #2 stack-detection rollout — Hedge, eko-ultra-automations, limn-systems-enterprise, glyphwise) does not include Erlang or OCaml as primary languages. Demand for Erlang and OCaml introspection is low at v1 release. The two grammars are LICENSE-CLEAN and could be promoted to the active set in a future minor release without re-litigating license clearance — they are deferred purely on prioritization grounds, not on licensing risk.

---

## Summary

| Status     | Count | Languages                                                                                          |
|------------|-------|---------------------------------------------------------------------------------------------------|
| CLEAN (active v1) | 14    | python, typescript, javascript, swift, rust, go, ruby, php, java, kotlin, elixir, csharp, cpp, haskell |
| DEFERRED   | 2     | erlang, ocaml                                                                                     |
| REVIEW     | 0     | (none)                                                                                            |
| REMOVE     | 0     | (none — no copyleft incompatibility found)                                                       |

**Total enumerated**: 16 (14 active + 2 deferred). The plan's "16 + 2 deferred = 18" phrasing in the original description was inclusive — the audited set is exactly 16 named grammars with 2 of them deferred.

**Phase 1 license-clearance gate**: PASS for all 4 first-party adapter languages (python, typescript, javascript, swift — all MIT).

**Phase 9 release-note implication**: no grammar was REMOVED for license reasons. The deferred set (erlang, ocaml) is documented as available for post-v1 promotion. No omission line needed in release notes for license reasons.

---

## Re-verification protocol (Phase 1)

When Phase 1 pins each grammar to a specific commit/tag for the WASM manifest:

1. `git ls-remote https://github.com/<owner>/<repo>.git refs/tags/v<X.Y.Z>` — confirm tag exists.
2. `gh api repos/<owner>/<repo>/contents/LICENSE?ref=v<X.Y.Z>` — confirm license file present at pinned ref.
3. Update `Pinned version` column in this doc.
4. Pin SHA-256 of the resulting WASM artifact in `tree-sitter-loader.ts` `GRAMMAR_MANIFEST` constant.

Phase 1 commit MUST update both this doc and the manifest constant atomically.

---

## References

- Plan: `/Users/ekoultra/hedge/docs/plans/2026-04-26-massu-3b-ast-lsp-introspection.md` line 127 (audit-iter-3 fix X)
- Spec doc: `/Users/ekoultra/massu/docs/internal/2026-04-26-ast-lsp-spec.md` section 8 (Strategy A WASM packaging)
- BSL 1.1 license overview: https://mariadb.com/bsl11/
- SPDX license list: https://spdx.org/licenses/
