# Plan AD: CLAUDE.md Auto-Generation in massu init

**Date:** 2026-04-13 | **Author:** Claude | **Status:** DRAFT — Awaiting Approval
**Repo:** `/Users/ekoultra/massu` (NOT hedge — this is a massu package change)
**Incident:** `docs/incidents/2026-04-13-missing-claude-md.md` — hedge ran 6+ weeks without CLAUDE.md

---

## Goal

When a user runs `npx massu init`, massu should automatically generate a well-structured `CLAUDE.md` file tailored to the detected project type. This ensures every massu-powered project starts with proper Claude Code project instructions from day one — preventing the incident where hedge ran for 6+ weeks without one.

**After Plan AD:**
- `npx massu init` generates CLAUDE.md as its first step
- CLAUDE.md is tailored to detected framework (Next.js, FastAPI, React, vanilla TS, etc.)
- CLAUDE.md includes massu-specific sections (workflow commands, memory system, patterns)
- `npx massu doctor` verifies CLAUDE.md exists
- Existing CLAUDE.md files are preserved (never overwritten)

---

## Phase A: CLAUDE.md Generator (5 items)

### P-A01: Create `generateClaudeMd()` function
- **File:** Modify — `packages/core/src/commands/init.ts`
- **Purpose:** New exported function that generates CLAUDE.md based on detected framework
- **Design:**
  ```typescript
  export function generateClaudeMd(
    projectRoot: string,
    framework: FrameworkDetection,
    python: PythonDetection
  ): { created: boolean; skipped: boolean } {
    const claudeMdPath = resolve(projectRoot, 'CLAUDE.md');
    
    // NEVER overwrite existing CLAUDE.md
    if (existsSync(claudeMdPath)) {
      return { created: false, skipped: true };
    }
    
    const projectName = basename(projectRoot);
    const content = buildClaudeMdContent(projectName, framework, python);
    writeFileSync(claudeMdPath, content, 'utf-8');
    return { created: true, skipped: false };
  }
  ```
- **Key:** Preserves existing files — idempotent and safe
- **Verification:** VR-GREP `generateClaudeMd` in init.ts exists; VR-FILE CLAUDE.md created after init

### P-A02: Build framework-aware content generator
- **File:** New — `packages/core/src/claude-md-templates.ts`
- **Purpose:** Template functions that produce CLAUDE.md sections based on detected stack
- **Sections generated:**
  1. **Project Overview** — name, detected tech stack summary
  2. **Tech Stack** — table of detected frameworks, languages, tools
  3. **Directory Structure** — scanned from actual filesystem (top 2 levels)
  4. **Coding Conventions** — framework-specific defaults (e.g., Next.js app router patterns, FastAPI async conventions)
  5. **Testing** — detected test framework and location
  6. **Massu Workflow** — standard `/massu-create-plan → /massu-plan → /massu-golden-path` reference
  7. **Memory System** — how massu memory works, what gets stored
  8. **Critical Rules** — starter set (expand as project matures)
- **Framework-specific templates:**
  | Framework | Extra Sections |
  |-----------|---------------|
  | Next.js | App Router conventions, server/client components, API routes |
  | FastAPI | Async patterns, Pydantic models, router organization |
  | SvelteKit | Load functions, form actions, server routes |
  | React (vanilla) | Component patterns, state management |
  | Python (generic) | venv management, type hints, testing with pytest |
  | TypeScript (generic) | ESM imports, strict mode, build tooling |
- **Verification:** VR-FILE `claude-md-templates.ts` exists; unit test generates content for each framework type

### P-A03: Directory structure scanner
- **File:** `packages/core/src/claude-md-templates.ts` (same file as P-A02)
- **Purpose:** Scan the project's actual directory tree (2 levels deep) and format as a markdown tree for the CLAUDE.md
- **Design:**
  ```typescript
  function scanDirectoryStructure(projectRoot: string, maxDepth: number = 2): string {
    // Walk filesystem, skip node_modules/.git/.venv/__pycache__/dist/build
    // Return formatted markdown tree like:
    // ```
    // project/
    // ├── src/
    // │   ├── app/
    // │   ├── components/
    // │   └── lib/
    // ├── tests/
    // └── package.json
    // ```
  }
  ```
- **Exclusions:** `node_modules`, `.git`, `.venv`, `venv`, `__pycache__`, `dist`, `build`, `.next`, `.nuxt`, `.svelte-kit`, `coverage`
- **Verification:** VR-TEST — unit test scans a fixture directory and produces expected tree

### P-A04: Wire into init flow as Step 0
- **File:** Modify — `packages/core/src/commands/init.ts`
- **Change:** Add CLAUDE.md generation as the FIRST step in `runInit()`, before config generation
- **Location:** After framework detection (line ~543), before config creation (line ~556)
- **Code:**
  ```typescript
  // Step 1.5: Generate CLAUDE.md (MUST be first — incident 2026-04-13)
  const claudeMdResult = generateClaudeMd(projectRoot, framework, python);
  if (claudeMdResult.created) {
    console.log('  Created CLAUDE.md (project instructions for Claude Code)');
  } else {
    console.log('  CLAUDE.md already exists (preserved)');
  }
  ```
- **Verification:** VR-GREP `generateClaudeMd` called in `runInit()`

### P-A05: Update InitResult type
- **File:** Modify — `packages/core/src/commands/init.ts`
- **Change:** Add `claudeMdCreated: boolean` and `claudeMdSkipped: boolean` to `InitResult` interface (line ~39)
- **Verification:** VR-GREP `claudeMdCreated` in InitResult

---

## Phase B: Doctor Check (2 items)

### P-B01: Add CLAUDE.md check to doctor
- **File:** Modify — `packages/core/src/commands/doctor.ts`
- **Purpose:** `massu doctor` should verify CLAUDE.md exists and warn if missing
- **Design:**
  ```typescript
  function checkClaudeMd(projectRoot: string): CheckResult {
    const claudeMdPath = resolve(projectRoot, 'CLAUDE.md');
    if (!existsSync(claudeMdPath)) {
      return {
        name: 'CLAUDE.md',
        status: 'warn',
        detail: 'CLAUDE.md not found. Run: npx massu init (or create manually)',
      };
    }
    // Check it's not empty
    const content = readFileSync(claudeMdPath, 'utf-8');
    if (content.trim().length < 50) {
      return {
        name: 'CLAUDE.md',
        status: 'warn',
        detail: 'CLAUDE.md exists but appears empty or minimal',
      };
    }
    return {
      name: 'CLAUDE.md',
      status: 'pass',
      detail: 'CLAUDE.md found and has content',
    };
  }
  ```
- **Wire:** Add to the `checks` array in `runDoctor()` alongside existing checks
- **Verification:** VR-GREP `checkClaudeMd` in doctor.ts

### P-B02: Update doctor header comment
- **File:** Modify — `packages/core/src/commands/doctor.ts`
- **Change:** Add item 11 to the header comment list (line ~10): `11. CLAUDE.md exists with content`
- **Verification:** VR-GREP `CLAUDE.md` in doctor.ts header comment

---

## Phase C: Tests (3 items)

### P-C01: Unit tests for generateClaudeMd
- **File:** New — `packages/core/src/__tests__/claude-md-generation.test.ts`
- **Tests:**
  - Creates CLAUDE.md in empty directory
  - Skips if CLAUDE.md already exists (preserves existing)
  - Content includes project name
  - Content includes detected framework (Next.js → mentions app router)
  - Content includes massu workflow section
  - Content includes memory system section
  - Python project gets FastAPI-specific conventions
  - Directory structure section reflects actual filesystem
- **Verification:** VR-TEST `npm test -- claude-md-generation`

### P-C02: Unit tests for directory scanner
- **File:** Same as P-C01
- **Tests:**
  - Scans fixture directory correctly
  - Excludes node_modules, .git, __pycache__
  - Respects maxDepth parameter
  - Handles empty directories
  - Handles permission errors gracefully
- **Verification:** VR-TEST

### P-C03: Integration test for init flow
- **File:** Existing — `packages/core/src/__tests__/init.test.ts` (or create if missing)
- **Tests:**
  - Full `runInit()` creates CLAUDE.md alongside massu.config.yaml
  - CLAUDE.md content matches detected framework
  - Re-running init preserves existing CLAUDE.md
- **Verification:** VR-TEST

---

## Phase D: Massu's Own CLAUDE.md (1 item)

### P-D01: Create CLAUDE.md for the massu project itself
- **File:** New — `/Users/ekoultra/massu/CLAUDE.md`
- **Purpose:** The massu project itself doesn't have a CLAUDE.md (same incident!)
- **Content:** Massu-specific: TypeScript monorepo, ESM, esbuild hooks, better-sqlite3, MCP server protocol, tool registration patterns (3-function: getDefs/isTool/handleCall), vitest testing, massu.config.yaml schema
- **Verification:** VR-FILE `/Users/ekoultra/massu/CLAUDE.md` exists

---

## Execution Strategy

This is a small, focused plan — single golden path run.

### Run 1: All Phases (A + B + C + D)
**Items:** P-A01 through P-D01 (11 items)
**Risk:** LOW — additive feature, no existing code modified destructively
**Prompt:**
```
Implement Plan AD (Single Run): CLAUDE.md Auto-Generation in massu init. Plan: docs/plans/2026-04-13-plan-AD-massu-claude-md-generation.md. Create claude-md-templates.ts with framework-aware content generation and directory scanner. Add generateClaudeMd() to init.ts as Step 0 (before config generation). Add CLAUDE.md check to doctor.ts. Write tests for generation, scanning, and init integration. Create CLAUDE.md for the massu project itself. All 11 items.
```

---

## Dependency Graph

```
P-A02 (templates) ──> P-A01 (generator fn) ──> P-A04 (wire into init)
P-A03 (dir scanner) ─┘                         P-A05 (type update)
                                                     │
P-B01 (doctor check) ──> P-B02 (header)              │
                                                     │
P-C01 (gen tests) ──────────────────────────────> depends on A01, A02
P-C02 (scanner tests) ─────────────────────────> depends on A03
P-C03 (integration tests) ─────────────────────> depends on A01-A05

P-D01 (massu CLAUDE.md) ──> no deps (standalone)
```

---

## Item Summary

| Phase | Items | Description |
|-------|-------|-------------|
| A: Generator | 5 | generateClaudeMd, templates, scanner, init wiring |
| B: Doctor | 2 | CLAUDE.md existence check in health report |
| C: Tests | 3 | Unit + integration tests |
| D: Self-heal | 1 | CLAUDE.md for massu itself |
| **Total** | **11** | |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Template content too generic | Medium | Low | Real directory scanning + framework-specific sections make it useful from day one |
| Overwrites user's CLAUDE.md | N/A | N/A | Explicit existsSync guard — never overwrites |
| Init performance slower | Low | Low | Single readdir + file write — negligible |
| Template maintenance burden | Low | Medium | Templates are simple string builders, not complex logic |

---

## Post-Build Reflection

1. **"Now that I've built this, what would I have done differently?"**
   - To be answered by implementing agent after verification

2. **"What should be refactored before moving on?"**
   - To be answered by implementing agent after verification

---

*Plan generated from massu codebase investigation on 2026-04-13. All file paths verified against `/Users/ekoultra/massu`.*
