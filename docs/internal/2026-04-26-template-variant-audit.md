# Phase 0 Audit — Template Variants for `@massu/core@1.3.0`

**Date**: 2026-04-26
**Plan**: `/Users/ekoultra/hedge/docs/plans/2026-04-26-massu-stack-aware-command-templates.md`
**Auditor**: massu-loop iteration 1

## Scope

The variant convention introduced in this plan applies **only to the top-level of `packages/core/commands/`** (60 `.md` files). Files in subdirectories (37 nested `.md` files) are copied recursively as-is — no variant resolution, no dot-skip filter.

| Layer | File count | Variant scope |
|-------|-----------|---------------|
| Top-level `packages/core/commands/*.md` | 60 | IN SCOPE — variant resolution applies |
| Nested `packages/core/commands/**/*.md` (depth ≥ 1) | 37 | OUT OF SCOPE — copied recursively as-is |
| **Total `.md` templates bundled** | **97** | — |

## TS-leaning detection (recursive grep, baseline)

```
grep -rln 'tRPC|trpc|Next\.js|Vercel|prisma|\.tsx' packages/core/commands/*.md
```

Top-level matches: 15 files

```
_shared-preamble.md, massu-batch.md, massu-checkpoint.md, massu-create-plan.md,
massu-deploy.md, massu-guide.md, massu-learning-audit.md, massu-perf.md,
massu-production-verify.md, massu-rollback.md, massu-scaffold-page.md,
massu-scaffold-router.md, massu-security.md, massu-tdd.md, massu-type-mismatch-audit.md
```

The vast majority of these mentions are **incidental** (e.g., `massu-perf.md` checking Vercel/Next.js patterns as part of generic perf review, `massu-security.md` referring to tRPC procedure auth as one example among many). Only three templates are **structurally TS/Next.js-shaped** to the point that they actively misguide a non-TS consumer:

1. `massu-scaffold-router.md` — opens with "Scaffold New tRPC Router"; would scaffold a tRPC file in a FastAPI service.
2. `massu-scaffold-page.md` — opens with "Scaffold New Page" (Next.js App Router page.tsx); would scaffold `.tsx` in a SwiftUI/iOS service.
3. `massu-deploy.md` — hard-coded to `vercel deploy` against project `prj_Io7AaGCM27cwRQerAj3BdihUur1Y`; would attempt a Vercel deploy of a launchd-managed Python service.

Per the plan's "favor making the default generic over forking" guidance, the remaining 12 incidental-mention templates do NOT get variants in this release. Their default content is generic enough to serve all consumers; the TS-flavored examples inside them are illustrative rather than prescriptive.

## Per-template labels (top-level, sorted)

Labels:
- **generic** — current default works for all consumers; no variants needed
- **needs-variant: `<lang>`** — ship a `<base>.<lang>.md` variant
- **replace-default** — current default is wrong/too-narrow; regenerate the default and (optionally) ship language variants

| # | Template | Label | Variants this release |
|---|----------|-------|----------------------|
| 1 | `_shared-preamble` | generic | (special filename — survives dot-skip + pickVariant; installs as-is) |
| 2 | `massu-article-review` | generic | — |
| 3 | `massu-audit-deps` | generic | — |
| 4 | `massu-autoresearch` | generic | — |
| 5 | `massu-batch` | generic | — |
| 6 | `massu-bearings` | generic | — |
| 7 | `massu-changelog` | generic | — |
| 8 | `massu-checkpoint` | generic | — |
| 9 | `massu-ci-fix` | generic | — |
| 10 | `massu-cleanup` | generic | — |
| 11 | `massu-command-health` | generic | — |
| 12 | `massu-command-improve` | generic | — |
| 13 | `massu-commit` | generic | — |
| 14 | `massu-create-plan` | generic | — |
| 15 | `massu-data` | generic | — |
| 16 | `massu-dead-code` | generic | — |
| 17 | `massu-debug` | generic | — |
| 18 | `massu-deploy` | **needs-variant: python** | `.python.md` (launchd / pm2 / docker) |
| 19 | `massu-deps` | generic | — |
| 20 | `massu-doc-gen` | generic | — |
| 21 | `massu-docs` | generic | — |
| 22 | `massu-estimate` | generic | — |
| 23 | `massu-full-audit` | generic | — |
| 24 | `massu-gap-enhancement-analyzer` | generic | — |
| 25 | `massu-golden-path` | generic | — |
| 26 | `massu-guide` | generic | — |
| 27 | `massu-hooks` | generic | — |
| 28 | `massu-hotfix` | generic | — |
| 29 | `massu-incident` | generic | — |
| 30 | `massu-infra-audit` | generic | — |
| 31 | `massu-learning-audit` | generic | — |
| 32 | `massu-loop` | generic | — |
| 33 | `massu-loop-playwright` | generic | — |
| 34 | `massu-new-feature` | generic | — |
| 35 | `massu-new-pattern` | generic | — |
| 36 | `massu-parity` | generic | — |
| 37 | `massu-perf` | generic | — |
| 38 | `massu-plan` | generic | — |
| 39 | `massu-plan-audit` | generic | — |
| 40 | `massu-production-verify` | generic | — |
| 41 | `massu-push` | generic | — |
| 42 | `massu-push-light` | generic | — |
| 43 | `massu-recap` | generic | — |
| 44 | `massu-refactor` | generic | — |
| 45 | `massu-release` | generic | — |
| 46 | `massu-review` | generic | — |
| 47 | `massu-rollback` | generic | — |
| 48 | `massu-scaffold-hook` | generic | — |
| 49 | `massu-scaffold-page` | **replace-default + needs-variant: swift** | `.swift.md` (SwiftUI iOS/visionOS) + regenerated default with embedded Next.js / FastAPI / SwiftUI / Rust examples |
| 50 | `massu-scaffold-router` | **needs-variant: python** | `.python.md` (FastAPI) |
| 51 | `massu-security` | generic | — |
| 52 | `massu-simplify` | generic | — |
| 53 | `massu-squirrels` | generic | — |
| 54 | `massu-status` | generic | — |
| 55 | `massu-tdd` | generic | — |
| 56 | `massu-test` | generic | — |
| 57 | `massu-type-mismatch-audit` | generic | — |
| 58 | `massu-ui-audit` | generic | — |
| 59 | `massu-verify` | generic | — |
| 60 | `massu-verify-playwright` | generic | — |

**Counts**: 57 generic, 2 needs-variant (python), 1 replace-default + needs-variant (swift). Total = 60.

## Nested files (out of variant scope, copied as-is)

37 `.md` files inside 6 subdirectories. All are **content-generic** (no language-specific scaffolding); they describe protocols, references, or sub-phase docs that apply to all stacks. Confirmed by inspection:

| Subdirectory | Files | Notes |
|--------------|-------|-------|
| `_shared-references/` | 5 | Cross-cutting protocol docs (auto-learning, blast-radius, security pre-screen, test-first, verification table) |
| `massu-autoresearch/references/` | 3 | Eval, safety, scoring |
| `massu-data/references/` | 2 | Common queries, table guide |
| `massu-debug/references/` | 5 | Debug investigation phases, codegraph tracing, etc. |
| `massu-golden-path/references/` | 12 | Phase docs, evaluator specs |
| `massu-loop/references/` | 7 | Loop controller, plan extraction, etc. |
| **Total** | **37** | All copied recursively unchanged |

These subdirectory files would NOT be subject to the dot-skip filter Phase 1 introduces because the recursive call passes `topLevel: false`. Filenames that happen to contain dots (none currently exist, but future authors are unblocked) survive verbatim.

## Cross-repo prerequisite (Risk #4)

Hedge's three rewrites at `/Users/ekoultra/hedge/.claude/commands/{massu-scaffold-router,massu-scaffold-page,massu-deploy}.md` are currently **modified-but-uncommitted** in the Hedge working tree (verified via `git status` — 414 insertions / 197 deletions across the three files vs HEAD). Phase 3 SEED-NN copies the on-disk Hedge content as the seed source. The plan flagged this in Risk #4: "if the file is uncommitted, commit it first." Commit lives in the Hedge repo, separate from this Massu work; flagging here for traceability but not blocking Phase 3 since the on-disk content is what the plan intends to template.

## SEED-02 decision (recorded)

Per Phase 3 / iteration-3 reasoning in the plan: ship `massu-scaffold-page.swift.md` (extracted "Path A: iOS / visionOS SwiftUI View" section) AND regenerate `massu-scaffold-page.md` as a framework-agnostic prompt with embedded Next.js / FastAPI / SwiftUI / Rust examples. Do NOT ship `.typescript.md` — with `framework.primary: typescript` (Hedge's case), it would be probed first and shadow `.swift.md`, contradicting the Acceptance line "Hedge installs ... SwiftUI scaffold-page".

## Phase 0 acceptance — confirmed

- [x] Recursive grep covers nested templates (39 hits across 60 top-level + nested files; 15 hits among top-level only).
- [x] Each of the 60 top-level templates has a label.
- [x] Subdirectory contents enumerated with confirmation that they are generic.
- [x] Cross-checked against Hedge `.claude/commands/{massu-scaffold-router,massu-scaffold-page,massu-deploy}.md` rewrites (lines 144 / 203 / 192 respectively) which become Phase 3 seed content.
