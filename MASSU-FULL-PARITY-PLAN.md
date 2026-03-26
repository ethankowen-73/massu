# MASSU FULL PARITY PLAN — Fix Everything

**Created**: 2026-03-26
**Created by**: Limn Systems Enterprise session (source of truth)
**Reason**: Prior session falsely claimed parity was complete. NPM package v0.5.0 shipped with stale commands, no agents, no patterns, no protocols, no reference files. All consumer projects are stale.
**Authority**: This plan overrides any prior claims of completion.

---

## Scope

Fix FULL feature parity between Limn Systems Enterprise (gold standard) and the entire Massu ecosystem:
**Massu ecosystem** (uses @massu/core npm package):
1. `@massu/core` npm package (the distribution mechanism)
2. `~/massu/` repo (source + docs + website)
3. `~/massu-internal/` repo (internal commands + extended infra)
4. `~/hedge/` project (consumer)
5. `~/justin/` project (consumer)

**Limn direct** (NOT massu — syncs directly from enterprise):
6. `~/limn-mobile/` — iOS/iPad app for Limn Systems. Uses `limn-swift-` prefix. Syncs from enterprise via `/limn-command-sync`, not through massu at all.

---

## Phase 1: Update @massu/core Package Commands (~/massu/)

### 1.1 Update ALL 43 existing commands to match Limn latest

Every command in `packages/core/commands/` must be re-synced from the Limn enterprise source. The adaptation rules are mechanical:
- Replace `limn-` prefix → `massu-` in command names and slash references
- Replace `CR-28` → `CR-9`, `CR-15` → `CR-5`, `CR-35` → `CR-12`
- Replace `limn-systems-enterprise` in paths → project-agnostic
- Keep all toolchain references (npm, tsc, Playwright, Supabase, tRPC) unchanged

**Source**: `~/limn-systems-enterprise/.claude/commands/`
**Target**: `~/massu/packages/core/commands/`

Commands to sync (43 existing — ALL need content update):

| # | Source (Limn) | Target (Massu) | Lines (Limn) |
|---|--------------|----------------|--------------|
| 1 | `_shared-preamble.md` | `_shared-preamble.md` | 116 |
| 2 | `limn-autoresearch.md` | `massu-autoresearch.md` | 258 |
| 3 | `limn-batch.md` | `massu-batch.md` | 359 |
| 4 | `limn-bearings.md` | `massu-bearings.md` | 238 |
| 5 | `limn-ci-fix.md` | `massu-ci-fix.md` | 148 |
| 6 | `limn-commit.md` | `massu-commit.md` | 591 |
| 7 | `limn-create-plan.md` | `massu-create-plan.md` | 650 |
| 8 | `limn-data.md` | `massu-data.md` | 66 |
| 9 | `limn-dead-code.md` | `massu-dead-code.md` | 126 |
| 10 | `limn-debug.md` | `massu-debug.md` | 216 |
| 11 | `limn-gap-enhancement-analyzer.md` | `massu-gap-enhancement-analyzer.md` | 740 |
| 12 | `limn-golden-path.md` | `massu-golden-path.md` | 250 |
| 13 | `limn-guide.md` | `massu-guide.md` | 179 |
| 14 | `limn-hooks.md` | `massu-hooks.md` | 126 |
| 15 | `limn-hotfix.md` | `massu-hotfix.md` | 550 |
| 16 | `limn-incident.md` | `massu-incident.md` | 194 |
| 17 | `limn-loop.md` | `massu-loop.md` | 228 |
| 18 | `limn-loop-playwright.md` | `massu-loop-playwright.md` | 860 |
| 19 | `limn-plan-audit.md` | `massu-plan-audit.md` | 64 |
| 20 | `limn-plan.md` | `massu-plan.md` | 741 |
| 21 | `limn-push.md` | `massu-push.md` | 118 |
| 22 | `limn-recap.md` | `massu-recap.md` | 190 |
| 23 | `limn-scaffold-hook.md` | `massu-scaffold-hook.md` | 94 |
| 24 | `limn-scaffold-page.md` | `massu-scaffold-page.md` | 107 |
| 25 | `limn-scaffold-router.md` | `massu-scaffold-router.md` | 124 |
| 26 | `limn-simplify.md` | `massu-simplify.md` | 315 |
| 27 | `limn-squirrels.md` | `massu-squirrels.md` | 101 |
| 28 | `limn-tdd.md` | `massu-tdd.md` | 217 |
| 29 | `limn-verify.md` | `massu-verify.md` | 1021 |
| 30 | `limn-verify-playwright.md` | `massu-verify-playwright.md` | 610 |

Plus these massu-only commands that already exist but may need updating against their Limn equivalents:
| # | Massu Command | Limn Equivalent | Action |
|---|--------------|-----------------|--------|
| 31 | `massu-changelog.md` | N/A | Keep as-is (massu-specific) |
| 32 | `massu-cleanup.md` | N/A | Keep as-is (massu-specific) |
| 33 | `massu-deploy.md` | N/A | Keep as-is (massu-specific) |
| 34 | `massu-deps.md` | N/A | Keep as-is (massu-specific) |
| 35 | `massu-doc-gen.md` | N/A | Keep as-is (massu-specific) |
| 36 | `massu-docs.md` | `limn-docs.md` | Update from Limn |
| 37 | `massu-estimate.md` | N/A | Keep as-is (massu-specific) |
| 38 | `massu-new-feature.md` | N/A (archived in Limn) | Keep as-is |
| 39 | `massu-parity.md` | N/A | Keep as-is (massu-specific) |
| 40 | `massu-push-light.md` | N/A | Keep as-is (massu-specific) |
| 41 | `massu-refactor.md` | N/A (archived in Limn) | Keep as-is |
| 42 | `massu-release.md` | N/A | Keep as-is (massu-specific) |
| 43 | `massu-review.md` | N/A | Keep as-is (massu-specific) |
| 44 | `massu-status.md` | N/A | Keep as-is (massu-specific) |
| 45 | `massu-test.md` | `limn-test.md` | Update from Limn |

### 1.2 Add MISSING commands (not in massu core today)

These Limn commands have no massu equivalent and should be added:

| # | Source (Limn) | New Target (Massu) | Lines | Purpose |
|---|--------------|-------------------|-------|---------|
| 1 | `limn-checkpoint.md` | `massu-checkpoint.md` | 588 | Phase boundary audit |
| 2 | `limn-command-health.md` | `massu-command-health.md` | 132 | Command quality dashboard |
| 3 | `limn-command-improve.md` | `massu-command-improve.md` | 232 | Score-driven optimization |
| 4 | `limn-infra-audit.md` | `massu-infra-audit.md` | 187 | .claude/ health check |
| 5 | `limn-learning-audit.md` | `massu-learning-audit.md` | 211 | Pattern effectiveness |
| 6 | `limn-production-verify.md` | `massu-production-verify.md` | 433 | Live production check |
| 7 | `limn-full-audit.md` | `massu-full-audit.md` | 61 | Audit dispatcher |
| 8 | `limn-security.md` | `massu-security.md` | 619 | Security audit |
| 9 | `limn-book-edit.md` | SKIP | — | Enterprise-specific |
| 10 | `limn-article-review.md` | SKIP | — | Enterprise-specific |
| 11 | `limn-rollback.md` | `massu-rollback.md` | 613 | Safe rollback |

### 1.3 Add ALL reference sub-directories

These folder-based commands need their reference files. Create adapted versions of each:

**`massu-golden-path/references/`** (15 files from Limn):
```
phase-0-requirements.md
phase-1-plan-creation.md
phase-2-implementation.md
phase-2.5-gap-analyzer.md
phase-3-simplify.md
phase-4-commit.md
phase-5-push.md
phase-5.5-production-verify.md
phase-6-completion.md
approval-points.md
competitive-mode.md
error-handling.md
sprint-contract-protocol.md
qa-evaluator-spec.md
vr-visual-calibration.md
```

**`massu-loop/references/`** (7 files):
```
plan-extraction.md
iteration-structure.md
guardrails.md
checkpoint-audit.md
vr-plan-spec.md
auto-learning.md
loop-controller.md
```

**`massu-debug/references/`** (5 files):
```
investigation-phases.md
common-shortcuts.md
codegraph-tracing.md
auto-learning.md
report-format.md
```

**`massu-autoresearch/references/`** (3 files):
```
scoring-protocol.md
eval-runner.md
safety-rails.md
```

**`massu-data/references/`** (2 files):
```
common-queries.md
table-guide.md
```

**`_shared-references/`** (5 files):
```
verification-table.md
auto-learning-protocol.md
security-pre-screen.md
blast-radius-protocol.md
test-first-protocol.md
```

### 1.4 Add Agent Definitions to Package

Create `packages/core/agents/` directory with adapted versions of all 11 Limn agents:

| # | Source (Limn ~/.claude/agents/) | Target (packages/core/agents/) | Lines |
|---|-------------------------------|-------------------------------|-------|
| 1 | `limn-architecture-reviewer.md` | `massu-architecture-reviewer.md` | 104 |
| 2 | `limn-blast-radius-analyzer.md` | `massu-blast-radius-analyzer.md` | 84 |
| 3 | `limn-competitive-scorer.md` | `massu-competitive-scorer.md` | 126 |
| 4 | `limn-help-sync.md` | `massu-help-sync.md` | 73 |
| 5 | `limn-migration-writer.md` | `massu-migration-writer.md` | 94 |
| 6 | `limn-output-scorer.md` | `massu-output-scorer.md` | 87 |
| 7 | `limn-pattern-reviewer.md` | `massu-pattern-reviewer.md` | 84 |
| 8 | `limn-plan-auditor.md` | `massu-plan-auditor.md` | 170 |
| 9 | `limn-schema-sync-verifier.md` | `massu-schema-sync-verifier.md` | 70 |
| 10 | `limn-security-reviewer.md` | `massu-security-reviewer.md` | 98 |
| 11 | `limn-ux-reviewer.md` | `massu-ux-reviewer.md` | 106 |

### 1.5 Add Patterns to Package

Create `packages/core/patterns/` with generic (non-enterprise-specific) patterns:

| # | Source Pattern | Applicable? | Action |
|---|--------------|-------------|--------|
| 1 | `build-patterns.md` | Yes | Adapt (remove Limn-specific paths) |
| 2 | `database-patterns.md` | Yes | Adapt (generic Supabase/Prisma) |
| 3 | `security-patterns.md` | Yes | Adapt |
| 4 | `testing-patterns.md` | Yes | Adapt |
| 5 | `mcp-integration.md` | Yes | Adapt |
| 6 | `schema-sync-patterns.md` | Yes | Adapt |
| 7 | `tool-routing.md` | Yes | Copy (already generic) |
| 8 | `integration-testing-checklist.md` | Yes | Adapt |
| 9 | `auth-patterns.md` | Partial | Adapt (remove Limn portal specifics) |
| 10 | `ui-patterns.md` | Partial | Adapt (remove Limn design tokens) |
| 11 | `form-patterns.md` | No | Enterprise-specific components |
| 12 | `crm-field-mappings.md` | No | Enterprise-specific |
| 13 | `email-patterns.md` | No | Enterprise-specific |
| 14 | `display-patterns.md` | No | Enterprise-specific |
| 15 | `component-patterns.md` | Partial | Adapt |
| 16 | `ai-knowledge-patterns.md` | Yes | Adapt |
| 17 | `rbac-patterns.md` | Yes | Adapt |
| 18 | `access-control-patterns.md` | Yes | Adapt |
| 19 | `realtime-patterns.md` | Yes | Adapt |

### 1.6 Add Protocols to Package

Create `packages/core/protocols/`:

| # | Protocol | Action |
|---|----------|--------|
| 1 | `plan-implementation.md` | Adapt (CR refs) |
| 2 | `recovery.md` | Adapt |
| 3 | `verification.md` | Adapt |

### 1.7 Add Reference Files to Package

Create `packages/core/reference/`:

| # | Reference | Action |
|---|-----------|--------|
| 1 | `vr-verification-reference.md` | Adapt |
| 2 | `patterns-quickref.md` | Adapt |
| 3 | `lessons-learned.md` | Adapt |
| 4 | `subagents-reference.md` | Adapt |
| 5 | `command-taxonomy.md` | Adapt |
| 6 | `hook-execution-order.md` | Adapt |
| 7 | `standards.md` | Copy |

### 1.8 Update package.json

```json
"files": [
  "src/**/*",
  "!src/__tests__/**",
  "dist/**/*",
  "commands/**/*",
  "agents/**/*",
  "patterns/**/*",
  "protocols/**/*",
  "reference/**/*",
  "LICENSE"
]
```

### 1.9 Update install-commands to install ALL infrastructure

Modify `packages/core/src/commands/install-commands.ts`:

Current behavior: Only copies `commands/*.md` to `.claude/commands/`

New behavior:
1. Copy `commands/**/*.md` to `.claude/commands/` (including sub-directories!)
2. Copy `agents/*.md` to project-local `.claude/agents/` (or `~/.claude/agents/` with user consent)
3. Copy `patterns/*.md` to `.claude/patterns/`
4. Copy `protocols/*.md` to `.claude/protocols/`
5. Copy `reference/*.md` to `.claude/reference/`

Update `massu init` to call the expanded installer.

### 1.10 Sync massu repo's own .claude/ with packages/core

The massu repo has its own `.claude/commands/` with OLDER versions (234 lines for golden-path vs 973 in packages/core). These must be synced:
- Delete `.claude/commands/*.md`
- Run `massu install-commands` against the updated packages/core

### 1.11 Bump Version and Publish

- Version: `0.6.0` (significant infrastructure addition)
- Run `npm run build` to compile hooks
- Run `npm publish`
- Verify published package includes all new directories

---

## Phase 2: Update Consumer Projects

### 2.1 Hedge (~/hedge/)

```bash
cd ~/hedge
npm update @massu/core
npx massu install-commands
```

Verify:
- [ ] Commands count matches core (should be ~55+ core + 7 hedge-specific)
- [ ] Reference sub-directories exist for golden-path, loop, debug, autoresearch, data
- [ ] _shared-references/ exists with 5 files
- [ ] Agents installed (10 massu agents)
- [ ] Patterns installed
- [ ] Protocols installed

### 2.2 Justin (~/justin/)

Currently uses absolute path to massu-internal hooks. Should switch to npm:

```bash
cd ~/justin
npm install @massu/core@latest  # or update existing
npx massu install-commands
```

Update `settings.local.json` to use `node_modules/@massu/core/dist/hooks/` instead of absolute path.

Verify same checklist as Hedge.

### 2.3 Massu-Internal (~/massu-internal/)

This repo has additional `massu-internal-*` commands (33 of them). Need to:
1. Update all 43 core commands from updated packages/core
2. Keep all 33 `massu-internal-*` commands unchanged
3. Update reference sub-directories
4. Update agents (verify all 10 are current)
5. Add any missing patterns/protocols/reference files

### 2.4 Limn-Mobile (~/limn-mobile/) — NOT MASSU, SEPARATE ECOSYSTEM

**limn-mobile is the iOS/iPad app for Limn Systems. It does NOT use massu.** It syncs directly from limn-systems-enterprise using `limn-swift-` prefix adaptations.

- Prefix: `limn-swift-` (NOT `massu-`)
- Sync source: `~/limn-systems-enterprise/.claude/commands/` directly
- Sync tool: `/limn-command-sync` run from the enterprise repo
- Toolchain: Swift/Xcode/XCTest (not npm/tsc/Playwright)
- This is the most complex adaptation due to Swift toolchain substitutions

**Execute from a session in `~/limn-systems-enterprise/`** using `/limn-command-sync` targeting limn-mobile. Do NOT run from massu.

---

## Phase 3: Update Documentation (~/massu/docs/)

### 3.1 Update Command Count References

Files that hardcode "43 commands":
- `docs/getting-started/installation.mdx` — update to new count
- `docs/getting-started/quick-start.mdx` — update to new count
- `docs/reference/cli-reference.mdx` — update to new count
- `docs/commands/index.mdx` — update command listing table
- `docs/features/workflow-commands.mdx` — update prose

### 3.2 Add New Command Pages

Create individual `.mdx` pages in `docs/commands/` for each new command:
- `massu-checkpoint.mdx`
- `massu-command-health.mdx`
- `massu-command-improve.mdx`
- `massu-infra-audit.mdx`
- `massu-learning-audit.mdx`
- `massu-production-verify.mdx`
- `massu-full-audit.mdx`
- `massu-security.mdx`
- `massu-rollback.mdx`

### 3.3 Add Documentation for New Infrastructure

New doc pages needed:
- `docs/features/agents.mdx` — what agents are, how they work, list of all 11
- `docs/features/patterns.mdx` — what patterns are, how they're used
- `docs/features/protocols.mdx` — what protocols are (plan-implementation, recovery, verification)
- `docs/reference/agents-reference.mdx` — detailed agent reference
- `docs/reference/patterns-reference.mdx` — detailed pattern reference

### 3.4 Update Hook Count

If any new hooks are added, update the "11 hooks" references in:
- `docs/getting-started/installation.mdx`
- `docs/reference/cli-reference.mdx`

---

## Phase 4: Verification Checklist

After ALL phases complete, verify each project:

### Per-Project Verification

```bash
# In each project directory:

# 1. Command count
ls .claude/commands/*.md | wc -l
# Expected: massu core count + project-specific count

# 2. Reference directories exist
ls .claude/commands/massu-golden-path/references/ | wc -l  # Expected: 15
ls .claude/commands/massu-loop/references/ | wc -l          # Expected: 7
ls .claude/commands/massu-debug/references/ | wc -l         # Expected: 5
ls .claude/commands/massu-autoresearch/references/ | wc -l  # Expected: 3
ls .claude/commands/massu-data/references/ | wc -l          # Expected: 2
ls .claude/commands/_shared-references/ | wc -l             # Expected: 5

# 3. Agents exist
ls .claude/agents/*.md | wc -l  # Expected: 10-11

# 4. Golden-path content check (the canary)
grep -c "Phase 2.5" .claude/commands/massu-golden-path.md    # Expected: >= 1
grep -c "Phase 5.5" .claude/commands/massu-golden-path.md    # Expected: >= 1
grep -c "competitive" .claude/commands/massu-golden-path.md  # Expected: >= 1

# 5. No stale limn- references (massu repos only)
grep -r "limn-" .claude/commands/ --include="*.md" -l
# Expected: 0 files (all should be massu- prefixed)

# 6. Patterns exist
ls .claude/patterns/*.md | wc -l  # Expected: 10+

# 7. Protocols exist
ls .claude/protocols/*.md | wc -l  # Expected: 3
```

### NPM Package Verification

```bash
# After publish, verify package contents:
npm pack @massu/core --dry-run 2>&1 | head -100
# Verify: commands/, agents/, patterns/, protocols/, reference/ all listed

# Install in temp dir and check:
mkdir /tmp/massu-test && cd /tmp/massu-test
npm init -y && npm install @massu/core@latest
ls node_modules/@massu/core/commands/ | wc -l      # Expected: 55+
ls node_modules/@massu/core/agents/ | wc -l        # Expected: 11
ls node_modules/@massu/core/patterns/ | wc -l      # Expected: 10+
ls node_modules/@massu/core/protocols/ | wc -l     # Expected: 3
ls node_modules/@massu/core/reference/ | wc -l     # Expected: 7
```

---

## Skip List (DO NOT sync these)

Per the enterprise skip list, these commands are enterprise-only and should NOT appear in massu:

| Command | Reason |
|---------|--------|
| `limn-massu-parity` | Enterprise-specific |
| `limn-mobile-research` | Enterprise-specific |
| `format-supabase-migration` | Enterprise-specific template |
| `format-trpc-router` | Enterprise-specific template |
| `limn-db-audit` | Enterprise-specific |
| `limn-db-branch` | Enterprise-specific |
| `limn-scaffold-mcp` | Enterprise-specific |
| `limn-command-sync` | Enterprise-specific (the sync tool itself) |
| `careful` | Enterprise-specific |
| `freeze` | Enterprise-specific |
| `limn-book-edit` | Enterprise-specific |
| `limn-article-review` | Enterprise-specific |
| `limn-api-contract` | Enterprise-specific |
| `limn-docs` | Enterprise-specific (help site sync) |
| `limn-new-pattern` | Enterprise-specific |
| `limn-perf` | Enterprise-specific |
| `limn-type-mismatch-audit` | Enterprise-specific |
| `limn-ui-audit` | Enterprise-specific |

---

## Adaptation Rules Quick Reference

### Prefix Mapping
| Find | Replace |
|------|---------|
| `limn-` (command names) | `massu-` |
| `/limn-` (slash commands) | `/massu-` |
| `CR-28` | `CR-9` |
| `CR-15` | `CR-5` |
| `CR-35` | `CR-12` |
| `limn-systems-enterprise` (paths) | project-agnostic |

### Keep Unchanged
- `npm run build`, `npm test`, `npx tsc --noEmit`
- `pattern-scanner.sh`
- Playwright references
- `src/` paths
- tRPC patterns
- Supabase references
- MCP tool names

---

## Execution Order

**Massu ecosystem** (sequential — npm publish gates consumer updates):
1. **Session in ~/massu/** — Phase 1 (all 11 sub-phases) + Phase 3 (docs) + npm publish
2. **Session in ~/massu-internal/** — Phase 2.3
3. **Session in ~/hedge/** — Phase 2.1 (`npm update @massu/core` + `npx massu install-commands`)
4. **Session in ~/justin/** — Phase 2.2 (`npm update @massu/core` + `npx massu install-commands`)

**Limn direct** (independent — can run in parallel with massu work):
5. **Session in ~/limn-systems-enterprise/** — Run `/limn-command-sync` targeting limn-mobile

**Verification**:
6. **Any session** — Phase 4 verification checklist

**Critical path**: Phase 1 must complete and npm publish must succeed before Phases 2-4 can start for massu consumers. Phase 5 (limn-mobile) is independent and can run anytime.

---

## Definition of Done

- [ ] @massu/core 0.6.0 published with commands + agents + patterns + protocols + reference
- [ ] `massu install-commands` installs ALL infrastructure (not just commands)
- [ ] Golden-path has Phase 2.5, Phase 5.5, competitive mode, 15 reference files
- [ ] All 11 agents distributed
- [ ] All consumer projects updated and verified (Phase 4 checklist passes)
- [ ] Documentation updated with correct counts and new pages
- [ ] Zero `limn-` references in any massu repo
- [ ] massu repo's own .claude/ matches packages/core content

**This plan is the source of truth. Do not claim completion without running the Phase 4 verification checklist and showing output.**
