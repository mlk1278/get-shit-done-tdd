---
name: Native Test Verification (Consolidated)
overview: >
  Make the planner read TESTING.md for all code-producing phases, embed real test commands
  into plans with worktree/post-merge annotations, add a blocking test gate before SUMMARY
  creation, and extend the phase regression gate to run post-merge tests. 4 files, 6 edits.
---

# Native Test Verification — Consolidated Implementation Plan

## Background

### The Problem

An executor built 4 plans of frontend calendar features (week view, day view, sidebar,
drag-to-schedule, conflict dialogs) and ran zero frontend tests. The executor's explanation:
"the plans didn't include test tasks."

### Root Cause

The planner's `load_codebase_context` step has a keyword routing table that only loads
TESTING.md when the phase name contains "testing" or "tests":

```
| testing, tests | TESTING.md, CONVENTIONS.md |
```

For a frontend phase, keywords match "UI, frontend, components" which loads
`CONVENTIONS.md, STRUCTURE.md` — no TESTING.md. The planner never learns what test runners
exist, what commands to use, or where test files go. It writes `<verify>` blocks with
build-only checks or vague criteria. The executor faithfully follows — and ships zero tests.

There is also no aggregate test gate before SUMMARY creation. Individual task verification
failures can be skipped, and no step collects and re-runs all plan tests as a final check.

### Design Principles

- Fix lives in the **planning phase** — planner embeds test commands, executor follows them
- **No new agents, commands, or processes** — changes use existing GSD patterns
- **Minimal orchestrator impact** — all new logic runs in sub-agents except a small
  extension to the existing `regression_gate` (which already runs tests)
- **Planner discretion preserved** — every code task needs a runnable `<automated>` command,
  but it can be a build/typecheck for tasks with no testable behavior
- **Codebase agnostic** — driven entirely by TESTING.md content, which is project-specific

---

## Files to Modify

| # | File | Role | Runs in |
|---|------|------|---------|
| 1 | `.claude/agents/gsd-planner.md` | Planner agent definition | Planner sub-agent |
| 2 | `.claude/get-shit-done/templates/codebase/testing.md` | TESTING.md template | Mapper sub-agent |
| 3 | `.claude/get-shit-done/workflows/execute-plan.md` | Plan execution workflow | Executor sub-agent |
| 4 | `.claude/get-shit-done/workflows/execute-phase.md` | Phase orchestration workflow | Orchestrator (minimal) |

### Files NOT changed

| File | Reason |
|------|--------|
| `gsd-executor.md` | Already runs `<verify>` per-task. The aggregate gate belongs in execute-plan.md. |
| `references/dev-test-loop.md` | Does not exist. Guidance lives in planner where it's consumed. |

---

## Implementation

### File 1: `.claude/agents/gsd-planner.md` — 4 edits

#### Edit A — Load TESTING.md for all code-producing phases

**Location:** `load_codebase_context` step, keyword routing table (~line 1041)

**What:** Add TESTING.md to every row except `setup, config`.

**Replace the table with:**

```markdown
**Rule:** Always load TESTING.md for code-producing phases. Skip only for pure config/docs phases.

| Phase Keywords | Load These |
|----------------|------------|
| UI, frontend, components | CONVENTIONS.md, STRUCTURE.md, TESTING.md |
| API, backend, endpoints | ARCHITECTURE.md, CONVENTIONS.md, TESTING.md |
| database, schema, models | ARCHITECTURE.md, STACK.md, TESTING.md |
| testing, tests | TESTING.md, CONVENTIONS.md |
| integration, external API | INTEGRATIONS.md, STACK.md, TESTING.md |
| refactor, cleanup | CONCERNS.md, ARCHITECTURE.md, TESTING.md |
| setup, config | STACK.md, STRUCTURE.md |
| (default) | STACK.md, ARCHITECTURE.md, TESTING.md |
```

**Why:** This is the root fix. Without TESTING.md, the planner cannot write real test commands.

---

#### Edit B — Strengthen Nyquist Rule + env annotation

**Location:** `<verify>` / Nyquist Rule section (~line 169)

**Replace the Nyquist paragraph with:**

```markdown
**Nyquist Rule:** Every `<verify>` for a code-producing task MUST include a runnable `<automated>` command. Consult TESTING.md for the project's runners and commands.

- **Test exists:** Reference it. `<automated>npx vitest run src/feature.test.ts</automated>`
- **No test exists, task has testable behavior:** Task MUST create the test. Add test file to `<files>`, include test creation in `<action>`, run it in `<automated>`.
- **No testable behavior** (config, styling, glue code): Use a build/typecheck command. `<automated>npm run build</automated>`
- **`<automated>MISSING</automated>`:** Only valid when a separate Wave 0 task creates the test infrastructure itself.

**Execution environment:** Annotate commands that need all parallel plans merged:
- `env="worktree"` (default, omit) — unit, component, single-feature tests. Runs during plan execution.
- `env="post-merge"` — E2E, cross-feature tests needing all worktrees merged. Runs before phase verification.

```xml
<verify>
  <automated>npx vitest run src/calendar/week-view.test.ts</automated>
  <automated env="post-merge">npx playwright test tests/e2e/calendar.spec.ts</automated>
</verify>
```
```

**Why:** Gives the planner clear rules for what level of verification each task needs, and how to annotate worktree vs post-merge scope.

---

#### Edit C — Frontend verification rule

**Location:** After TDD exceptions (~line 335)

**Add:**

```markdown
## Frontend/UI Verification

When TESTING.md documents component/unit infrastructure: UI tasks MUST include test files in `<files>` with `<automated>` commands.

When TESTING.md documents E2E infrastructure: Plans changing user-visible behavior MUST include at least one `<automated env="post-merge">` E2E command.

When TESTING.md shows no test infrastructure for a surface: Add a Wave 0 task to scaffold it. Never ship a plan with zero automated verification for new interactive features.
```

**Why:** Explicit frontend rule prevents the calendar problem — no more UI plans with zero tests.

---

#### Edit D — Fix stale task example

**Location:** Task template example (~line 447)

**Replace the flat `<verify>[Command or check]</verify>` example with:**

```xml
<task type="auto">
  <name>Task 1: [Action-oriented name]</name>
  <files>path/to/file.ext, path/to/file.test.ext</files>
  <action>[Specific implementation]</action>
  <verify>
    <automated>npm test -- path/to/file.test.ext</automated>
  </verify>
  <done>[Acceptance criteria]</done>
</task>
```

**Why:** Examples must match the Nyquist Rule. A flat `<verify>` example teaches the planner to skip `<automated>`.

---

### File 2: `.claude/get-shit-done/templates/codebase/testing.md` — 1 edit

**Location:** After the existing "Test Types" section

**Append:**

```markdown
## Verification Commands by Scope

### Worktree-safe (fast feedback)
Commands that work in an isolated branch:
- Unit tests: `[exact command]`
- Component tests: `[exact command]`

### Post-merge (full integration)
Commands requiring all parallel work merged:
- E2E tests: `[exact command]`
- Cross-feature integration: `[exact command]`

## Coverage Gaps

Surfaces with no automated test coverage. Planners use this to create Wave 0 scaffolding tasks.

- [ ] [surface] — [what's missing]
```

**Why:** The planner can only write correct `env=` annotations if TESTING.md distinguishes which commands are worktree-safe. The mapper is the agent that reads package.json and test configs — it should capture this. Coverage Gaps lets the planner know when Wave 0 scaffolding is needed.

---

### File 3: `.claude/get-shit-done/workflows/execute-plan.md` — 1 edit

**Location:** Insert new step before `generate_user_setup` (~line 376)

**Add:**

```xml
<step name="test_passage_gate">
Collect every `<automated>` command from the plan's task `<verify>` blocks. Exclude `env="post-merge"` commands (those run at phase level after merge).

Run each remaining command.

**All pass:** Continue to generate_user_setup.

**Any fail:**
1. Identify root cause.
2. Fix the implementation, not the test.
3. Re-run. Budget: 3 fix cycles per failure.
4. Still failing after 3 cycles: STOP. Return checkpoint:

**Plan:** {phase}-{plan}
**Failing:** {test commands and output}
**Tried:** {what was attempted}

Do NOT create SUMMARY.md with failing tests. Tests pass or the plan is blocked.
</step>
```

**Why:** This is the hard gate. No SUMMARY without passing tests. Runs in the executor sub-agent — zero orchestrator impact.

---

### File 4: `.claude/get-shit-done/workflows/execute-phase.md` — 1 edit

**Location:** Extend existing `regression_gate` step (~line 522), append after current regression test logic

**Add:**

```markdown
**Post-merge verification:** After regression tests, collect `env="post-merge"` commands from this phase's PLAN files:

```bash
grep -h 'env="post-merge"' ${PHASE_DIR}/*-PLAN.md 2>/dev/null | \
  sed 's/.*>\(.*\)<.*/\1/' | sort -u
```

Run each unique command. Skip if none found.

**All pass:** Proceed to verify_phase_goal.

**Any fail:** Spawn `gsd-executor` to fix post-merge failures. Re-run once. If still failing: present to user with Fix or Abort options. Do NOT offer "Continue to verification anyway."
```

**Also:** Tighten the existing regression failure path — remove the "Continue to verification anyway" option from the current regression gate.

**Why:** This is the only orchestrator-level addition. It follows the existing `regression_gate` pattern (which already runs tests in the orchestrator). Fix work is delegated to a spawned executor sub-agent.

---

## Discretion Model

| Situation | What planner writes |
|-----------|-------------------|
| New interactive feature (calendar drag, auth flow) | Real tests — unit + E2E |
| Existing logic modification | Run existing tests covering that code |
| Config, styling, glue code, no behavioral change | Build/typecheck command |
| No test infra exists for the surface | Wave 0 scaffolding task |
| Pure docs/config phase | TESTING.md not loaded, no gate applies |

The planner has discretion on *what kind* of `<automated>` command to use. It does NOT have discretion to omit `<automated>` entirely for code-producing tasks.

---

## How This Catches the Calendar Problem

1. Planner loads TESTING.md for "UI, frontend, components" phase -> sees Vitest + Playwright
2. Frontend rule forces `<automated>` commands for component tests + `env="post-merge"` E2E
3. If no test infrastructure exists, Wave 0 scaffolding task is required
4. `test_passage_gate` blocks SUMMARY if per-worktree tests fail
5. `regression_gate` runs Playwright E2E after worktrees merge, blocks phase if it fails
6. No escape hatch — tests pass or work is blocked

---

## Provenance

| Element | Source |
|---------|--------|
| Root cause identification, 3-file skeleton, `env=` attribute design | Opus 4.6 |
| TESTING.md scope section, Coverage Gaps, frontend rule | GPT 5.4 Extra High |
| `env="post-merge"` naming | Opus + Gemini (convergent) |
| Extending `regression_gate` vs new step | GPT 5.4 Extra High |
| Executor sub-agent spawn for post-merge fixes | Opus 4.6 |
| Discretion model (build check for non-testable tasks) | Consolidated synthesis |
