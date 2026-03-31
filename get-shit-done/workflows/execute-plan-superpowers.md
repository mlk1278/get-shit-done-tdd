<purpose>
Bridge between GSD's plan execution and superpowers' subagent-driven development.
Replaces the standard gsd-executor dispatch with per-task implementer subagents,
two-stage review (spec compliance + code quality), TDD enforcement, and
verification-before-completion discipline.

Activated when `workflow.superpowers_execution: true` in .planning/config.json.
Falls back to standard gsd-executor when disabled.
</purpose>

<core_principle>
GSD owns the lifecycle (plans, state, commits, summaries). Superpowers owns the
execution quality (per-task agents, review loops, TDD discipline, verification gates).
This workflow translates between the two — reading GSD's PLAN.md format, dispatching
superpowers-style agents, and writing GSD's expected artifacts.
</core_principle>

<required_reading>
Read STATE.md before any operation to load project context.
Read config.json for planning behavior settings.

@~/.claude/get-shit-done/references/git-integration.md
@~/.claude/get-shit-done/references/tdd.md
</required_reading>

<available_agent_types>
Valid GSD subagent types (use exact names):
- gsd-executor — Single-task implementer (scoped to one task from the plan)
- gsd-verifier — Verifies phase completion
- superpowers:code-reviewer — Code quality review agent

For spec and quality reviews, use `general-purpose` agents with the review prompts below.
</available_agent_types>

<process>

<step name="init_context" priority="first">
Same initialization as standard execute-plan.md:

```bash
INIT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" init execute-phase "${PHASE}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Extract: `executor_model`, `commit_docs`, `sub_repos`, `phase_dir`, `phase_number`, `plans`, `summaries`, `incomplete_plans`, `state_path`, `config_path`.

**Resolve reviewer models:**
```bash
SPEC_REVIEWER_MODEL=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" resolve-model gsd-spec-reviewer 2>/dev/null || echo "sonnet")
QUALITY_REVIEWER_MODEL=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" resolve-model gsd-quality-reviewer 2>/dev/null || echo "sonnet")
```

These use GSD's deterministic model resolution (profile → per-agent override → alias map).
If `resolve-model` is not available as a CLI command, extract from the init JSON or resolve
using the same `config-get model_profile` + MODEL_PROFILES lookup that `resolveModelInternal` uses.
</step>

<step name="identify_plan">
Same as standard execute-plan.md. Find first PLAN without matching SUMMARY.
</step>

<step name="record_start_time">
```bash
PLAN_START_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
PLAN_START_EPOCH=$(date +%s)
```
</step>

<step name="parse_plan_and_extract_tasks">
Read the PLAN.md file. Parse:
- **Frontmatter:** phase, plan, type, wave, depends_on, requirements, must_haves, files_modified, autonomous
- **Tasks:** Extract each `<task>` element with all attributes and child elements

**Translation: GSD XML tasks → Superpowers dispatch format**

For each `<task>` in the plan, build a dispatch context block:

```
TASK_N_CONTEXT = {
  task_number: N,
  name: from <name>,
  type: from type attribute (auto, checkpoint:*, tdd plan),
  tdd: from tdd attribute (true/false) OR plan type=="tdd",
  files: from <files>,
  read_first: from <read_first> (if present),
  action: from <action>,
  verify: from <verify> (including <automated> sub-elements),
  done: from <done>,
  acceptance_criteria: from <acceptance_criteria> (if present),
  behavior: from <behavior> (if tdd="true"),
  implementation: from <implementation> (if present),
}
```

Store all extracted tasks for sequential dispatch. Track:
- `TOTAL_TASKS` = count
- `COMPLETED_TASKS` = [] (task number, commit hash pairs)
- `DEVIATIONS` = [] (rule number, description pairs)
</step>

<step name="execute_tasks_with_review_loop">
For each task (sequentially — never parallel within a plan):

**A. Skip checkpoints (handle normally)**
If task type starts with `checkpoint:` → handle via standard checkpoint_protocol from execute-plan.md. Do NOT dispatch as implementer.

**B. Dispatch implementer subagent**

Translate the GSD task into a superpowers implementer prompt:

```
Task(
  description="Implement Task {N}: {task_name}",
  model="{executor_model}",
  isolation="worktree",
  prompt="
    You are implementing a single task from a GSD phase plan.

    ## Task Description

    **Goal:** {task_name}

    **Files:** {task_files}

    {IF read_first exists:}
    **Read First (MANDATORY):** {read_first_files}
    You MUST read every file listed above BEFORE making any edits.
    {ENDIF}

    **Action:** {task_action}

    **Acceptance Criteria:**
    {task_acceptance_criteria OR task_done}

    **Verify:** {task_verify}

    {IF tdd is true:}
    ## TDD Required — RED-GREEN-REFACTOR

    This task MUST follow test-driven development. The Iron Law applies:
    NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.

    **Behavior to test:**
    {task_behavior}

    **Cycle:**
    1. RED: Write failing test describing the behavior above. Run test. It MUST fail.
       If it passes, you're testing existing behavior — fix the test.
       Commit: test({phase}-{plan}): add failing test for {feature}

    2. GREEN: Write minimal code to make the test pass. No extras.
       Run test. It MUST pass. All other tests must still pass.
       Commit: feat({phase}-{plan}): implement {feature}

    3. REFACTOR (if needed): Clean up. Tests MUST still pass.
       Only commit if changes made: refactor({phase}-{plan}): clean up {feature}

    {ELSE:}
    ## Implementation

    Follow the action description. Write tests alongside implementation.
    Commit: {type}({phase}-{plan}): {task_name}
    {ENDIF}

    ## Context

    **Phase:** {phase_number}-{phase_name}
    **Plan:** {plan_number} of {total_plans}
    **Project:** Read ./CLAUDE.md for project conventions.

    <files_to_read>
    - {plan_file_path} (Plan — read for full context)
    - .planning/PROJECT.md (Project context)
    - .planning/STATE.md (Current state)
    - ./CLAUDE.md (Project instructions, if exists)
    </files_to_read>

    ## Commit Protocol

    Stage files individually (NEVER git add . or git add -A).
    Format: {type}({phase}-{plan}): {description}

    {IF parallel_execution:}
    Use --no-verify on all git commits (parallel agent — orchestrator validates hooks).
    {ENDIF}

    ## Verification Before Completion

    Before claiming ANY task is done:
    1. IDENTIFY what command proves your claim
    2. RUN the command (fresh, complete)
    3. READ the full output
    4. VERIFY it confirms your claim
    5. ONLY THEN report completion

    If you haven't run the verification command, you CANNOT claim it passes.
    No 'should work', no 'looks correct'. Evidence only.

    ## Deviation Rules

    While executing, if you discover unplanned work:
    - Rule 1 (Bug): Auto-fix broken behavior, track as [Rule 1 - Bug]
    - Rule 2 (Missing Critical): Auto-add missing essentials, track as [Rule 2 - Missing Critical]
    - Rule 3 (Blocking): Auto-fix blockers, track as [Rule 3 - Blocking]
    - Rule 4 (Architectural): STOP and report — do not proceed

    ## Self-Review Before Reporting

    Before reporting back, review your work:
    - Did I implement everything in the spec? Miss anything?
    - Are names clear? Code clean? Tests real (not mocked)?
    - Did I avoid overbuilding? Follow existing patterns?
    - If TDD: did I watch each test fail before implementing?

    ## Report Format

    When done, report:
    - **Status:** DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
    - **What you implemented**
    - **Files changed:** [list actual files]
    - **Acceptance criteria status:**
      - [criterion]: PASS/FAIL
    - **Verify command output:** [paste actual output]
    - **Commit hash(es):** [from git rev-parse --short HEAD]
    - **Deviations:** [Rule N - Type] description (if any)
    - **Concerns:** (if DONE_WITH_CONCERNS)
  "
)
```

**C. Handle implementer response**

| Status | Action |
|--------|--------|
| **DONE** | Proceed to spec review |
| **DONE_WITH_CONCERNS** | Read concerns. If correctness/scope issue → address before review. If observation → note and proceed to review. |
| **NEEDS_CONTEXT** | Provide missing context. Re-dispatch implementer with additional info. |
| **BLOCKED** | Assess: context problem → provide + re-dispatch. Model capability → re-dispatch with more capable model. Task too large → break down. Plan wrong → escalate to user (Rule 4). |

**D. Dispatch spec compliance reviewer**

```
Task(
  description="Review spec compliance for Task {N}",
  model="{SPEC_REVIEWER_MODEL}",
  prompt="
    You are reviewing whether an implementation matches its GSD plan specification.

    ## What Was Requested (from PLAN.md)

    **Task:** {task_name}
    **Action:** {task_action}
    **Files:** {task_files}
    **Acceptance Criteria:** {task_acceptance_criteria OR task_done}
    **Verify:** {task_verify}

    ## What Implementer Claims They Built

    {implementer_report}

    ## CRITICAL: Do Not Trust the Report

    The implementer may have been optimistic. Verify everything independently.

    **DO NOT** take their word. **DO** read the actual code.

    Compare actual implementation to the plan's <action> and <done> criteria
    line by line. Check for:

    - **Missing requirements:** Things in <action> or <acceptance_criteria> not implemented
    - **Extra/unneeded work:** Features built that weren't in the plan
    - **Misunderstandings:** Right feature, wrong approach

    {IF tdd:}
    ## TDD Verification

    Also verify:
    - Failing test was committed BEFORE implementation
    - Test actually tests the behavior described in <behavior>
    - Implementation is minimal (not over-engineered)
    - Check git log: test commit should precede feat commit
    {ENDIF}

    ## Report

    - ✅ Spec compliant (everything matches after code inspection)
    - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
  "
)
```

**If spec issues found (❌):** Send issues back to implementer (same or fresh subagent). Re-dispatch spec reviewer after fixes. Repeat until ✅. Max 3 iterations — if still failing after 3, escalate to user.

**E. Dispatch code quality reviewer**

Only after spec compliance passes ✅.

```
Task(
  description="Review code quality for Task {N}",
  model="{QUALITY_REVIEWER_MODEL}",
  prompt="
    Review the implementation of Task {N}: {task_name}

    **What was implemented:** {implementer_report}
    **Plan requirements:** {task_action} / {task_done}
    **Base SHA:** {commit_before_task}
    **Head SHA:** {current_commit}

    Review for:
    - Code quality: separation of concerns, error handling, type safety, DRY, edge cases
    - Architecture: sound design, follows existing patterns
    - Testing: tests verify behavior (not mocks), edge cases covered, all passing
    - File organization: each file has one clear responsibility
    - No scope creep beyond plan requirements

    Report:
    - **Strengths:** What's well done
    - **Issues:**
      - Critical (must fix): bugs, security, data loss
      - Important (should fix): architecture problems, test gaps
      - Minor (nice to have): style, optimization
    - **Assessment:** Approved | Needs fixes
  "
)
```

**If Critical or Important issues found:** Send to implementer for fixes. Re-dispatch quality reviewer. Repeat until approved. Max 3 iterations.

**F. Record task completion**

After both reviews pass:

```bash
# Track completed task
COMPLETED_TASKS+=("Task ${N}: ${TASK_COMMIT_HASH}")
TASK_COUNT=$((TASK_COUNT + 1))
```

Log: `✓ Task {N}: {task_name} — implemented, spec-verified, quality-approved`

**G. Proceed to next task**
</step>

<step name="test_passage_gate">
Same as standard execute-plan.md.

Collect every `<automated>` command from the plan's task `<verify>` blocks.
Run each command. All must pass before creating SUMMARY.md.

Failing after 3 fix cycles → STOP and return checkpoint.
</step>

<step name="create_summary">
Same as standard execute-plan.md. Create SUMMARY.md with GSD's expected format.

**Additional section for superpowers bridge:**

```markdown
## Review History

| Task | Spec Review | Quality Review | TDD | Iterations |
|------|-------------|----------------|-----|------------|
| 1 | ✅ | ✅ | Yes | 1 |
| 2 | ✅ (2nd pass) | ✅ | No | 2 |
| 3 | ✅ | ✅ | Yes | 1 |
```

This documents the per-task review loop results for traceability.
Include any spec/quality issues that were caught and fixed during reviews
under the "Deviations from Plan" section with tag `[Review - Spec]` or `[Review - Quality]`.
</step>

<step name="update_state_and_roadmap">
Same as standard execute-plan.md. All gsd-tools commands apply identically:

```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state advance-plan
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state update-progress
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state record-metric \
  --phase "${PHASE}" --plan "${PLAN}" --duration "${DURATION}" \
  --tasks "${TASK_COUNT}" --files "${FILE_COUNT}"
```

Extract decisions and update STATE.md. Update ROADMAP. Mark requirements complete.
Commit metadata. All identical to execute-plan.md steps.
</step>

<step name="offer_next">
Same as standard execute-plan.md routing (more plans / phase done / milestone done).
</step>

</process>

<configuration>
## Enabling Superpowers Execution

Add to `.planning/config.json`:

```json
{
  "workflow": {
    "superpowers_execution": true
  }
}
```

Or via CLI:
```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" config-set workflow.superpowers_execution true
```

When enabled, execute-phase.md routes plan execution through this workflow
instead of the standard gsd-executor agent.

When disabled (default), standard gsd-executor handles execution as before.
</configuration>

<review_iteration_limits>
## Review Loop Limits

- **Spec compliance:** Max 3 review iterations per task. If issues persist after 3 rounds,
  escalate to user with specific failing criteria.
- **Code quality:** Max 3 review iterations per task. Critical issues block; minor issues
  can be deferred to a follow-up if the user approves.
- **TDD verification:** If RED phase test passes when it shouldn't (testing existing behavior),
  max 2 attempts to fix the test before escalating.
</review_iteration_limits>

<fallback>
## Fallback to Standard Execution

If superpowers execution encounters systematic failures (e.g., review loop exhausted on
multiple tasks, model availability issues), fall back gracefully:

1. Log: "Superpowers review loop exhausted for Task {N}. Falling back to standard execution."
2. Execute remaining tasks using standard gsd-executor behavior (no per-task review).
3. Note in SUMMARY.md which tasks used superpowers flow vs standard.
4. Phase-level gsd-verifier still runs (catches issues the reviews missed).
</fallback>

<success_criteria>
- All tasks from PLAN.md completed
- Each task passed spec compliance review ✅
- Each task passed code quality review ✅
- TDD tasks followed RED-GREEN-REFACTOR with verified failing tests
- All verifications pass with evidence (not claims)
- SUMMARY.md created with review history
- STATE.md, ROADMAP.md, REQUIREMENTS.md updated via gsd-tools
- All commits follow GSD format: {type}({phase}-{plan}): {description}
</success_criteria>
