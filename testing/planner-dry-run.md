# GSD Planner Dry Run — TeamBoard Phase 3

**Phase goal:** "Working task board with drag-and-drop columns, task cards, and real-time status updates."

**Date:** 2026-03-29

---

## Planning Reasoning

### Step 1: Goal-Backward Analysis

**Goal (outcome-shaped):** A user can manage tasks on a visual board — dragging cards between columns, seeing status update live without a page reload.

**Observable Truths (what must be TRUE from the user's perspective):**
1. User can see columns (e.g., Todo, In Progress, Done) with task cards in each
2. User can drag a card from one column and drop it onto another column
3. After dropping, the card appears in the new column immediately (optimistic update)
4. The status change persists — refreshing the page shows the card in its new column
5. If another browser tab moves a card, this tab updates within a few seconds (real-time)
6. Columns are readable: card title, assignee, and priority are visible at a glance

**Required Artifacts (what must EXIST for each truth):**
- Truth 1: `BoardColumn` component, `TaskCard` component, `BoardPage` layout, types (`Task`, `Column`, `BoardState`)
- Truth 2: Drag-and-drop integration — `dnd-kit` library wired to card and column components
- Truth 3: Optimistic state update in `useBoardState` hook, `PATCH /api/tasks/[id]` endpoint
- Truth 4: Persistent `PATCH` handler writing to database; board initial load fetches from DB
- Truth 5: WebSocket or SSE subscription — server broadcasts task-moved events; board hook subscribes
- Truth 6: `TaskCard` renders title, assignee avatar/name, priority badge

**Key Links (where this most likely breaks):**
- `TaskCard` `onDragEnd` → `useBoardState.moveTask()` → `PATCH /api/tasks/[id]` → DB write
- SSE endpoint broadcasts → `useBoardRealtimeSync` subscribes → board state merges incoming event
- Board initial render → `GET /api/board` → returns columns with nested tasks
- `PATCH /api/tasks/[id]` validation → rejects invalid column IDs or missing task IDs

---

### Step 2: TDD Detection Analysis

Going feature by feature:

| Feature | Can write `expect(fn(input)).toBe(output)` before writing fn? | Decision |
|---|---|---|
| `Task` / `Column` TypeScript types | No (pure type file, no logic) | Standard plan, typecheck only |
| `GET /api/board` — fetch columns+tasks | Yes: `GET /api/board` returns `{ columns: Column[] }` with nested tasks | TDD plan |
| `PATCH /api/tasks/[id]` — move task to new column | Yes: valid columnId → 200 + updated task; invalid columnId → 400; missing task → 404 | TDD plan |
| `useBoardState` hook — optimistic moveTask | Yes: calling `moveTask(taskId, newColumnId)` reorders internal state predictably | TDD plan |
| `BoardColumn` + `TaskCard` UI components | No: visual layout, styling, JSX structure. Not `fn(input) → output` | Standard plan, component tests |
| dnd-kit wiring to board | No: glue code connecting existing dnd-kit library to existing components | Standard plan, component/interaction test |
| SSE real-time sync (`/api/board/events`) | Yes: server sends events; client handler merges them into board state | Split: server endpoint is TDD; client hook is TDD |
| `useBoardRealtimeSync` hook | Yes: incoming SSE `task-moved` event → board state reflects new column | TDD plan |
| Board page layout wiring | No: glue code composing components already built in other plans | Standard plan |

**Summary of plan types identified:**
- 4 TDD plans (API GET, API PATCH, useBoardState, useBoardRealtimeSync)
- 3 standard plans (types + scaffolding, UI components, dnd wiring + page assembly)
- Total: 7 plans

That's fine-grained. Let me re-evaluate sizing. Each TDD plan targets ~40% context (one feature, RED→GREEN→REFACTOR). Standard plans are 2-3 tasks each at ~50% context. 7 plans for a feature set of this size (board + D&D + real-time) is appropriate — each plan handles one complete concern at a comfortable context budget.

---

### Step 3: Dependency Graph

```
Task A (Plan 01): Types + scaffolding — no dependencies, creates contracts for everything
Task B (Plan 02): TDD — GET /api/board — depends on Plan 01 (types)
Task C (Plan 03): TDD — PATCH /api/tasks/[id] — depends on Plan 01 (types)
Task D (Plan 04): TDD — useBoardState hook — depends on Plan 01 (types)
Task E (Plan 05): UI components (BoardColumn, TaskCard) — depends on Plan 01 (types)
Task F (Plan 06): TDD — useBoardRealtimeSync — depends on Plan 04 (hook interface)
Task G (Plan 07): dnd-kit wiring + board page assembly — depends on Plans 02,03,04,05,06

Wave analysis:
  Wave 1: Plan 01 (foundation — types, DB schema, scaffolding)
  Wave 2: Plans 02, 03, 04, 05 (independently implement against Wave 1 contracts)
  Wave 3: Plan 06 (depends on Plan 04's hook interface being finalized)
  Wave 4: Plan 07 (wires everything together — needs all Wave 2 + Wave 3 complete)
```

Wave 3 note: `useBoardRealtimeSync` (Plan 06) must be in Wave 3, not Wave 2, because it depends on the `BoardState` shape established by `useBoardState` (Plan 04). The plans do not share files — Plan 04 owns `src/hooks/useBoardState.ts` and Plan 06 owns `src/hooks/useBoardRealtimeSync.ts`, but Plan 06's behavior tests import from Plan 04's output type (`BoardState`). If they were parallel and Plan 04 changes the `BoardState` shape, Plan 06's tests would conflict.

---

### Step 4: Requirement ID Assignment

Since no ROADMAP.md exists (dry run), I will derive logical requirement IDs from the phase goal:

- `BOARD-01` — Columns display with task cards
- `BOARD-02` — Drag-and-drop card movement between columns
- `BOARD-03` — Optimistic status update on drag
- `BOARD-04` — Status persists in database
- `BOARD-05` — Real-time updates across sessions/tabs

---

## Plan Breakdown Summary

| Plan | Type | Wave | Covers | Depends On |
|------|------|------|--------|------------|
| 01 | execute | 1 | Types, DB schema, test infrastructure | — |
| 02 | tdd | 2 | GET /api/board endpoint | 01 |
| 03 | tdd | 2 | PATCH /api/tasks/[id] endpoint | 01 |
| 04 | tdd | 2 | useBoardState hook (optimistic moves) | 01 |
| 05 | execute | 2 | BoardColumn + TaskCard UI components | 01 |
| 06 | tdd | 3 | useBoardRealtimeSync hook | 04 |
| 07 | execute | 4 | dnd-kit wiring + board page assembly | 02,03,04,05,06 |

Wave 2 plans (02, 03, 04, 05) all run in parallel — they each own exclusive files.

---

## Complete Plan Definitions

---

### PLAN 01 — Foundation: Types, Schema, and Test Scaffolding

```markdown
---
phase: 03-task-board
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - src/types/board.ts
  - prisma/schema.prisma
  - src/lib/db.ts
autonomous: true
requirements: [BOARD-01, BOARD-02, BOARD-03, BOARD-04, BOARD-05]

must_haves:
  truths:
    - "TypeScript types for Task, Column, BoardState, TaskMovedEvent are defined and exported"
    - "Prisma schema has Task and Column models with correct relations"
    - "DB client singleton is importable from src/lib/db.ts"
  artifacts:
    - path: "src/types/board.ts"
      provides: "Shared TypeScript contracts for all board features"
      exports: ["Task", "Column", "BoardState", "TaskMovedEvent", "MoveTaskPayload"]
    - path: "prisma/schema.prisma"
      provides: "Task and Column data models"
      contains: "model Task, model Column"
    - path: "src/lib/db.ts"
      provides: "Prisma client singleton"
      exports: ["db"]
  key_links:
    - from: "src/types/board.ts"
      to: "all other plans"
      via: "import { Task, Column, BoardState } from '@/types/board'"
      pattern: "from '@/types/board'"
    - from: "src/lib/db.ts"
      to: "API routes"
      via: "import { db } from '@/lib/db'"
      pattern: "from '@/lib/db'"
---

<objective>
Establish the shared type contracts and database schema that all other Phase 3 plans build against.

Purpose: Interface-first ordering — defining contracts before any implementation prevents the scavenger hunt anti-pattern where executors must explore the codebase to understand data shapes.
Output: Exported TypeScript types, Prisma models, and a working DB client. No application logic.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Define board TypeScript types</name>
  <files>src/types/board.ts</files>
  <action>
    Create and export all shared types. No implementation code — only type declarations:

    - `ColumnId = string` (branded type alias for clarity)
    - `TaskId = string`
    - `Priority = 'low' | 'medium' | 'high'`
    - `interface Task { id: TaskId; title: string; description?: string; columnId: ColumnId; priority: Priority; assignee?: string; order: number; createdAt: string; updatedAt: string; }`
    - `interface Column { id: ColumnId; title: string; order: number; tasks: Task[]; }`
    - `interface BoardState { columns: Column[]; }`
    - `interface MoveTaskPayload { taskId: TaskId; targetColumnId: ColumnId; targetOrder: number; }`
    - `interface TaskMovedEvent { type: 'task-moved'; taskId: TaskId; fromColumnId: ColumnId; toColumnId: ColumnId; order: number; timestamp: number; }`

    Export all from a single file. No imports from external packages needed.
  </action>
  <verify>
    <automated>npx tsc --noEmit</automated>
  </verify>
  <done>
    src/types/board.ts exists and exports all 8 type definitions. TypeScript reports no errors.
    All other plans can import types from '@/types/board' without codebase exploration.
  </done>
</task>

<task type="auto">
  <name>Task 2: Prisma schema — Column and Task models, DB client singleton</name>
  <files>prisma/schema.prisma, src/lib/db.ts</files>
  <action>
    Add to prisma/schema.prisma:

    ```prisma
    model Column {
      id    String @id @default(cuid())
      title String
      order Int    @default(0)
      tasks Task[]
    }

    model Task {
      id          String   @id @default(cuid())
      title       String
      description String?
      columnId    String
      column      Column   @relation(fields: [columnId], references: [id], onDelete: Cascade)
      priority    String   @default("medium")  // "low" | "medium" | "high"
      assignee    String?
      order       Int      @default(0)
      createdAt   DateTime @default(now())
      updatedAt   DateTime @updatedAt
    }
    ```

    Run `npx prisma db push` to apply schema.
    Run `npx prisma generate` to regenerate client.

    Create src/lib/db.ts:
    ```typescript
    import { PrismaClient } from '@prisma/client'

    const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

    export const db = globalForPrisma.prisma ?? new PrismaClient()

    if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
    ```

    This is the standard Next.js singleton pattern — prevents multiple PrismaClient instances in dev.
  </action>
  <verify>
    <automated>npx tsc --noEmit</automated>
  </verify>
  <done>
    prisma/schema.prisma includes Column and Task models with correct relation.
    src/lib/db.ts exports `db` as a PrismaClient singleton.
    `npx prisma db push` ran successfully (schema applied).
    TypeScript reports no errors.
  </done>
</task>

</tasks>

<verification>
- `src/types/board.ts` exports all required types
- `prisma/schema.prisma` contains Column and Task models
- `src/lib/db.ts` exports `db` singleton
- `npx tsc --noEmit` passes with no errors
</verification>

<success_criteria>
All downstream Wave 2 plans can import types and db without any codebase exploration.
Schema is applied to database. No application logic ships in this plan.
</success_criteria>

<output>
After completion, create `.planning/phases/03-task-board/03-01-SUMMARY.md`
</output>
```

**Testing decision for Plan 01:**
- Task 1 (type file): Pure type declarations — no runtime behavior. TDD not applicable (`expect(fn(input)).toBe(output)` has no `fn`). Typecheck (`tsc --noEmit`) is the correct and sufficient verification. No test file included.
- Task 2 (schema + db client): Config/schema change + boilerplate singleton pattern — no business logic. Typecheck is sufficient. A test that checks "db is a PrismaClient" would test the framework, not our code. No test file included.

---

### PLAN 02 — TDD: GET /api/board endpoint

```markdown
---
phase: 03-task-board
plan: 02
type: tdd
wave: 2
depends_on: [01]
files_modified:
  - src/app/api/board/route.ts
  - src/app/api/board/route.test.ts
autonomous: true
requirements: [BOARD-01]

must_haves:
  truths:
    - "GET /api/board returns columns ordered by Column.order, each with tasks ordered by Task.order"
    - "Response shape matches BoardState type"
    - "Empty board returns { columns: [] }"
  artifacts:
    - path: "src/app/api/board/route.ts"
      provides: "GET endpoint returning full board state"
      exports: ["GET"]
    - path: "src/app/api/board/route.test.ts"
      provides: "Vitest tests proving endpoint behavior"
  key_links:
    - from: "src/app/api/board/route.ts"
      to: "prisma.column.findMany"
      via: "db query with tasks include"
      pattern: "db\\.column\\.findMany"
---

<objective>
TDD implementation of the GET /api/board endpoint that returns the full board state.

Purpose: TDD ensures the response contract is designed before the query, preventing the common mistake of returning raw Prisma output instead of the typed BoardState shape.
Output: Working GET endpoint with full test coverage of happy path and edge cases.
</objective>

<feature>
  <name>GET /api/board — fetch full board state</name>
  <files>src/app/api/board/route.ts, src/app/api/board/route.test.ts</files>
  <behavior>
    Test 1: GET /api/board with seeded columns and tasks returns 200 with body matching:
      { columns: [ { id, title, order, tasks: [ { id, title, columnId, priority, order, ... } ] } ] }
      Columns ordered by `order` ASC. Tasks within each column ordered by `order` ASC.

    Test 2: GET /api/board with no data returns 200 with body { columns: [] }

    Test 3: Response shape — every column has `id`, `title`, `order`, `tasks`.
      Every task has `id`, `title`, `columnId`, `priority`, `order`, `createdAt`, `updatedAt`.
      (Shape test: verifies we don't accidentally leak DB internals or omit required fields.)

    Cases:
      seeded DB with 2 columns, 3 tasks → response.columns.length === 2
      column[0].tasks ordered by Task.order → tasks[0].order < tasks[1].order
      empty DB → { columns: [] }
  </behavior>
  <implementation>
    In src/app/api/board/route.ts, export async GET():
    - Query: db.column.findMany({ orderBy: { order: 'asc' }, include: { tasks: { orderBy: { order: 'asc' } } } })
    - Map Prisma result to BoardState shape (convert Date fields to ISO strings)
    - Return NextResponse.json({ columns }) with status 200
    - Wrap in try/catch; on error return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  </implementation>
</feature>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/phases/03-task-board/03-01-SUMMARY.md

<interfaces>
From src/types/board.ts:
```typescript
export type ColumnId = string
export type TaskId = string
export type Priority = 'low' | 'medium' | 'high'
export interface Task { id: TaskId; title: string; description?: string; columnId: ColumnId; priority: Priority; assignee?: string; order: number; createdAt: string; updatedAt: string; }
export interface Column { id: ColumnId; title: string; order: number; tasks: Task[]; }
export interface BoardState { columns: Column[]; }
```
</interfaces>
</context>

<verification>
vitest run src/app/api/board/route.test.ts
</verification>

<success_criteria>
- Failing test written (RED) and committed: `test(03-02): add failing test for GET /api/board`
- Implementation passes all tests (GREEN): `feat(03-02): implement GET /api/board`
- Refactor committed if cleanup needed: `refactor(03-02): clean up board query`
- `vitest run src/app/api/board/route.test.ts` passes with all 3 tests green
</success_criteria>

<output>
After completion, create `.planning/phases/03-task-board/03-02-SUMMARY.md` with RED/GREEN/REFACTOR notes.
</output>
```

**Testing decision for Plan 02:**
- This is a TDD plan. Test file `src/app/api/board/route.test.ts` is listed in `<files>` and is written FIRST (RED phase) before implementation.
- Test runner: `vitest run src/app/api/board/route.test.ts` (from TESTING.md: "Unit tests: vitest run [file]")
- Why TDD: The endpoint has a defined request/response contract. We can write `expect(response.body.columns).toHaveLength(2)` before writing any implementation. This is the classic API endpoint TDD case.
- No `env="post-merge"` needed here — this is a unit/integration test of a single route file, runnable in worktree.

---

### PLAN 03 — TDD: PATCH /api/tasks/[id] endpoint

```markdown
---
phase: 03-task-board
plan: 03
type: tdd
wave: 2
depends_on: [01]
files_modified:
  - src/app/api/tasks/[id]/route.ts
  - src/app/api/tasks/[id]/route.test.ts
autonomous: true
requirements: [BOARD-03, BOARD-04]

must_haves:
  truths:
    - "PATCH /api/tasks/[id] with valid columnId updates task's columnId in DB and returns updated task"
    - "PATCH with invalid columnId returns 400 with { error: string }"
    - "PATCH for non-existent taskId returns 404"
  artifacts:
    - path: "src/app/api/tasks/[id]/route.ts"
      provides: "PATCH endpoint for moving a task to a new column"
      exports: ["PATCH"]
    - path: "src/app/api/tasks/[id]/route.test.ts"
      provides: "Vitest tests for the PATCH endpoint"
  key_links:
    - from: "src/app/api/tasks/[id]/route.ts"
      to: "prisma.task.update"
      via: "db.task.update({ where: { id }, data: { columnId, order } })"
      pattern: "db\\.task\\.update"
---

<objective>
TDD implementation of the PATCH /api/tasks/[id] endpoint that persists a task's column change.

Purpose: TDD enforces the validation contract (400/404 responses) before implementation, preventing the common pattern of shipping a PATCH endpoint that silently accepts invalid data.
Output: Validated PATCH endpoint with error cases tested.
</objective>

<feature>
  <name>PATCH /api/tasks/[id] — move task to new column</name>
  <files>src/app/api/tasks/[id]/route.ts, src/app/api/tasks/[id]/route.test.ts</files>
  <behavior>
    Test 1 (happy path): PATCH /api/tasks/task-1 with body { columnId: "col-2", order: 0 }
      → 200, response body includes { id: "task-1", columnId: "col-2", order: 0 }
      → DB record updated (verify with follow-up query)

    Test 2 (invalid columnId): PATCH /api/tasks/task-1 with body { columnId: "" }
      → 400, response body { error: "columnId is required" }

    Test 3 (missing task): PATCH /api/tasks/nonexistent-id with valid body
      → 404, response body { error: "Task not found" }

    Test 4 (missing body fields): PATCH /api/tasks/task-1 with body {}
      → 400, response body { error: "columnId is required" }

    Cases:
      valid payload → 200 + { id, columnId, order, updatedAt }
      empty columnId → 400 + { error: "columnId is required" }
      missing taskId in DB → 404 + { error: "Task not found" }
      body missing columnId key → 400 + { error: "columnId is required" }
  </behavior>
  <implementation>
    In src/app/api/tasks/[id]/route.ts, export async PATCH(request, { params }):
    - Parse body, validate columnId is present and non-empty string → 400 if invalid
    - db.task.findUnique({ where: { id: params.id } }) → 404 if null
    - db.task.update({ where: { id: params.id }, data: { columnId: body.columnId, order: body.targetOrder ?? 0, updatedAt: new Date() } })
    - Return NextResponse.json(updatedTask) with status 200
    - Catch Prisma P2025 (not found) → 404
    - Catch other errors → 500
  </implementation>
</feature>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/phases/03-task-board/03-01-SUMMARY.md

<interfaces>
From src/types/board.ts:
```typescript
export interface MoveTaskPayload { taskId: TaskId; targetColumnId: ColumnId; targetOrder: number; }
export interface Task { id: TaskId; title: string; columnId: ColumnId; priority: Priority; order: number; createdAt: string; updatedAt: string; }
```
</interfaces>
</context>

<verification>
vitest run src/app/api/tasks/[id]/route.test.ts
</verification>

<success_criteria>
- Failing tests written and committed: `test(03-03): add failing tests for PATCH /api/tasks/[id]`
- Implementation makes all 4 tests pass: `feat(03-03): implement PATCH /api/tasks/[id]`
- Refactor if needed: `refactor(03-03): extract validation helper`
- `vitest run src/app/api/tasks/[id]/route.test.ts` passes with 4 tests green
</success_criteria>

<output>
After completion, create `.planning/phases/03-task-board/03-03-SUMMARY.md`
</output>
```

**Testing decision for Plan 03:**
- TDD plan. Test file written first (RED). Four behavior cases defined before any implementation.
- Runner: `vitest run src/app/api/tasks/[id]/route.test.ts`
- Why TDD: This endpoint has explicit input/output contracts with branching validation logic. Writing `expect(response.status).toBe(400)` before implementing the validation is the TDD sweet spot. Without TDD, the 404 and validation cases are the ones developers skip.
- Why not `env="post-merge"`: This is a route unit test — it mocks the Prisma client and tests the route handler in isolation. No need to wait for all plans to merge.

---

### PLAN 04 — TDD: useBoardState hook (optimistic moves)

```markdown
---
phase: 03-task-board
plan: 04
type: tdd
wave: 2
depends_on: [01]
files_modified:
  - src/hooks/useBoardState.ts
  - src/hooks/useBoardState.test.ts
autonomous: true
requirements: [BOARD-01, BOARD-03]

must_haves:
  truths:
    - "useBoardState initializes with a BoardState and exposes columns"
    - "moveTask(taskId, targetColumnId, targetOrder) moves the task optimistically in local state"
    - "moveTask triggers the PATCH API call with correct payload"
    - "On API failure, state rolls back to pre-move position"
  artifacts:
    - path: "src/hooks/useBoardState.ts"
      provides: "React hook managing board state with optimistic updates"
      exports: ["useBoardState", "UseBoardStateReturn"]
    - path: "src/hooks/useBoardState.test.ts"
      provides: "Vitest tests for hook behavior"
  key_links:
    - from: "src/hooks/useBoardState.ts"
      to: "PATCH /api/tasks/[id]"
      via: "fetch call in moveTask"
      pattern: "fetch.*api/tasks"
---

<objective>
TDD implementation of the useBoardState hook that manages optimistic drag-and-drop state.

Purpose: The optimistic update + rollback logic is pure state machine behavior — ideal TDD territory. Tests written first ensure the rollback case is not forgotten, which is the most common real-time UI bug.
Output: A tested React hook with optimistic update, API call, and rollback behavior.
</objective>

<feature>
  <name>useBoardState — board state with optimistic task movement</name>
  <files>src/hooks/useBoardState.ts, src/hooks/useBoardState.test.ts</files>
  <behavior>
    Test 1 (initialization): Given initial BoardState with 2 columns and 3 tasks,
      hook returns { columns } where columns matches initial data.

    Test 2 (optimistic move): Given task "t-1" in column "col-1",
      after moveTask("t-1", "col-2", 0):
      → columns["col-1"].tasks does NOT contain "t-1"
      → columns["col-2"].tasks DOES contain "t-1"
      → state updates BEFORE the API response (optimistic)

    Test 3 (API call): After moveTask("t-1", "col-2", 0),
      → fetch was called once with URL matching /api/tasks/t-1, method PATCH,
        body containing { columnId: "col-2", targetOrder: 0 }

    Test 4 (rollback on failure): Given mock fetch returning 500,
      after moveTask("t-1", "col-2", 0):
      → state eventually reverts: "t-1" is back in "col-1"

    Cases:
      initial state → columns reflects BoardState input
      moveTask(valid) → immediate local state change (optimistic)
      moveTask(valid) → fetch PATCH called with correct payload
      moveTask + API 500 → state rolled back to pre-move
  </behavior>
  <implementation>
    Hook interface:
    ```typescript
    export interface UseBoardStateReturn {
      columns: Column[];
      moveTask: (taskId: TaskId, targetColumnId: ColumnId, targetOrder: number) => Promise<void>;
    }
    export function useBoardState(initial: BoardState): UseBoardStateReturn
    ```

    Implementation:
    - useState(initial.columns) for local columns state
    - moveTask: (1) capture pre-move state snapshot, (2) apply optimistic update to state,
      (3) call PATCH /api/tasks/{taskId} with { columnId: targetColumnId, targetOrder },
      (4) on error: restore state snapshot, optionally re-throw or toast
    - Export `useBoardState` and `UseBoardStateReturn` type
  </implementation>
</feature>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/phases/03-task-board/03-01-SUMMARY.md

<interfaces>
From src/types/board.ts:
```typescript
export interface BoardState { columns: Column[]; }
export interface Column { id: ColumnId; title: string; order: number; tasks: Task[]; }
export interface Task { id: TaskId; title: string; columnId: ColumnId; priority: Priority; order: number; createdAt: string; updatedAt: string; }
```
</interfaces>
</context>

<verification>
vitest run src/hooks/useBoardState.test.ts
</verification>

<success_criteria>
- Failing tests committed: `test(03-04): add failing tests for useBoardState`
- Implementation passes: `feat(03-04): implement useBoardState with optimistic updates`
- Refactor if needed: `refactor(03-04): extract rollback logic`
- All 4 test cases pass. Rollback case is explicitly tested.
</success_criteria>

<output>
After completion, create `.planning/phases/03-task-board/03-04-SUMMARY.md`
</output>
```

**Testing decision for Plan 04:**
- TDD plan. The rollback-on-failure behavior is the critical case — without a test written first, implementations always skip it.
- Runner: `vitest run src/hooks/useBoardState.test.ts`
- Why TDD: This is a state machine. `moveTask(taskId, targetColumnId, order)` → predictable state transformation. Classic `fn(input) → output` structure even with React state, using `renderHook` from `@testing-library/react`.
- The test for "optimistic update" must assert the state changed BEFORE `await`ing the fetch mock — this is behavior that only TDD reliably captures.

---

### PLAN 05 — UI Components: BoardColumn and TaskCard

```markdown
---
phase: 03-task-board
plan: 05
type: execute
wave: 2
depends_on: [01]
files_modified:
  - src/components/board/TaskCard.tsx
  - src/components/board/TaskCard.test.tsx
  - src/components/board/BoardColumn.tsx
  - src/components/board/BoardColumn.test.tsx
autonomous: true
requirements: [BOARD-01, BOARD-02]

must_haves:
  truths:
    - "TaskCard renders task title, priority badge, and assignee"
    - "BoardColumn renders its title and all task cards in order"
    - "TaskCard accepts drag event handlers as props (wired in Plan 07)"
  artifacts:
    - path: "src/components/board/TaskCard.tsx"
      provides: "Individual task card UI component"
      exports: ["TaskCard", "TaskCardProps"]
    - path: "src/components/board/BoardColumn.tsx"
      provides: "Column component rendering header + task list"
      exports: ["BoardColumn", "BoardColumnProps"]
  key_links:
    - from: "src/components/board/BoardColumn.tsx"
      to: "src/components/board/TaskCard.tsx"
      via: "maps column.tasks → <TaskCard>"
      pattern: "column\\.tasks\\.map"
---

<objective>
Build the visual BoardColumn and TaskCard components as pure rendering components — no drag logic yet.

Purpose: Separating visual components from drag-and-drop wiring means each can be tested independently. Plan 07 wires dnd-kit onto these stable interfaces.
Output: Rendered, tested components ready for dnd-kit integration.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/phases/03-task-board/03-01-SUMMARY.md

<interfaces>
From src/types/board.ts:
```typescript
export type Priority = 'low' | 'medium' | 'high'
export interface Task { id: TaskId; title: string; description?: string; columnId: ColumnId; priority: Priority; assignee?: string; order: number; }
export interface Column { id: ColumnId; title: string; order: number; tasks: Task[]; }
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: TaskCard component</name>
  <files>src/components/board/TaskCard.tsx, src/components/board/TaskCard.test.tsx</files>
  <behavior>
    - Test 1: renders task.title in a visible element
    - Test 2: renders priority badge with text matching task.priority ("low", "medium", or "high")
    - Test 3: renders assignee name when task.assignee is set
    - Test 4: does NOT render assignee element when task.assignee is undefined
    - Test 5: accepts and passes through onDragStart and onDragEnd props without error
  </behavior>
  <action>
    Create TaskCardProps interface:
    ```typescript
    export interface TaskCardProps {
      task: Task;
      onDragStart?: (taskId: TaskId) => void;
      onDragEnd?: () => void;
    }
    ```

    Implement TaskCard:
    - Container div with Tailwind: `bg-white rounded-lg shadow-sm border border-gray-200 p-3 cursor-grab active:cursor-grabbing hover:shadow-md transition-shadow`
    - Title: `<p className="text-sm font-medium text-gray-900">{task.title}</p>`
    - Priority badge: `<span className={cn("text-xs px-1.5 py-0.5 rounded-full font-medium", priorityColors[task.priority])}>{task.priority}</span>`
      where priorityColors = { low: 'bg-blue-100 text-blue-700', medium: 'bg-yellow-100 text-yellow-700', high: 'bg-red-100 text-red-700' }
    - Assignee (conditional): `{task.assignee && <span className="text-xs text-gray-500">{task.assignee}</span>}`
    - Wire onDragStart/onDragEnd to draggable div props (placeholder — dnd-kit will replace in Plan 07)

    Test file uses @testing-library/react `render` and `screen` queries.
  </action>
  <verify>
    <automated>vitest run src/components/board/TaskCard.test.tsx</automated>
  </verify>
  <done>
    TaskCard renders title, priority badge, and conditional assignee.
    All 5 component tests pass. Props interface exported.
  </done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: BoardColumn component</name>
  <files>src/components/board/BoardColumn.tsx, src/components/board/BoardColumn.test.tsx</files>
  <behavior>
    - Test 1: renders column.title in a heading element
    - Test 2: renders one TaskCard for each task in column.tasks
    - Test 3: renders task count badge ("3 tasks" for a column with 3 tasks)
    - Test 4: renders empty state message ("No tasks") when column.tasks is empty
    - Test 5: accepts onTaskDrop prop (function) without error — wired in Plan 07
  </behavior>
  <action>
    Create BoardColumnProps interface:
    ```typescript
    export interface BoardColumnProps {
      column: Column;
      onTaskDrop?: (taskId: TaskId, targetColumnId: ColumnId, targetOrder: number) => void;
    }
    ```

    Implement BoardColumn:
    - Outer container: `<div className="flex flex-col w-72 bg-gray-50 rounded-xl border border-gray-200 p-3 min-h-[200px]">`
    - Header: `<div className="flex items-center justify-between mb-3">` with `<h2 className="font-semibold text-gray-700">{column.title}</h2>` and task count badge
    - Task list: `<div className="flex flex-col gap-2">` mapping column.tasks to TaskCard components
    - Empty state: `{column.tasks.length === 0 && <p className="text-sm text-gray-400 text-center py-4">No tasks</p>}`

    Note: onTaskDrop is accepted as a prop and passed through to TaskCard for Plan 07 to wire up.

    Test file uses @testing-library/react.
  </action>
  <verify>
    <automated>vitest run src/components/board/BoardColumn.test.tsx</automated>
  </verify>
  <done>
    BoardColumn renders column title, task count, all TaskCards, and empty state.
    All 5 component tests pass.
  </done>
</task>

</tasks>

<verification>
vitest run src/components/board/TaskCard.test.tsx
vitest run src/components/board/BoardColumn.test.tsx
</verification>

<success_criteria>
- TaskCard renders title, priority, assignee with correct Tailwind classes
- BoardColumn renders title, task count, TaskCards
- 10 component tests total pass (5 per component)
- Props interfaces exported and ready for Plan 07 wiring
</success_criteria>

<output>
After completion, create `.planning/phases/03-task-board/03-05-SUMMARY.md`
</output>
```

**Testing decision for Plan 05:**
- Standard plan (not TDD plan) because the work is UI rendering — not `fn(input) → output` logic.
- HOWEVER: the Nyquist Rule still applies. UI components that render data and accept event props MUST have component tests. Each task uses `tdd="true"` at task level to enforce test-first behavior within the standard plan structure.
- Test files ARE included in `<files>` for both tasks.
- Runner: `vitest run [file]` using `@testing-library/react` (from TESTING.md: "Component tests: vitest run [file] (uses @testing-library/react)")
- Why not a full TDD plan: Visual component rendering is not `fn(input).toBe(output)`. The tests verify "renders X" but the implementation is layout/styling work, not algorithm design.
- Why `tdd="true"` at task level: The behavior block makes test expectations explicit before implementation, giving the executor a clear RED step even without a dedicated TDD plan.

---

### PLAN 06 — TDD: useBoardRealtimeSync hook

```markdown
---
phase: 03-task-board
plan: 06
type: tdd
wave: 3
depends_on: [04]
files_modified:
  - src/app/api/board/events/route.ts
  - src/app/api/board/events/route.test.ts
  - src/hooks/useBoardRealtimeSync.ts
  - src/hooks/useBoardRealtimeSync.test.ts
autonomous: true
requirements: [BOARD-05]

must_haves:
  truths:
    - "GET /api/board/events streams SSE with Content-Type: text/event-stream"
    - "When a task moves via PATCH, an SSE event is broadcast to all connected clients"
    - "useBoardRealtimeSync subscribes to SSE and calls moveTask when task-moved event arrives"
    - "useBoardRealtimeSync cleans up EventSource on unmount"
  artifacts:
    - path: "src/app/api/board/events/route.ts"
      provides: "SSE endpoint broadcasting task-moved events"
      exports: ["GET"]
    - path: "src/hooks/useBoardRealtimeSync.ts"
      provides: "Hook subscribing to SSE and syncing board state"
      exports: ["useBoardRealtimeSync"]
  key_links:
    - from: "src/app/api/board/events/route.ts"
      to: "EventEmitter or global broadcast mechanism"
      via: "writes to SSE stream when PATCH /api/tasks/[id] is called"
      pattern: "text/event-stream"
    - from: "src/hooks/useBoardRealtimeSync.ts"
      to: "/api/board/events"
      via: "new EventSource('/api/board/events')"
      pattern: "new EventSource"
---

<objective>
TDD implementation of the SSE endpoint and client subscription hook for real-time board sync.

Purpose: The SSE event contract (what data is sent, what the client does with it) must be defined before implementation. TDD catches the common bug where the client hook doesn't handle the "already in correct state" case, causing infinite re-renders.
Output: Server SSE endpoint + client subscription hook, both with behavior tests.
</objective>

<feature>
  <name>SSE real-time board sync — server endpoint and client hook</name>
  <files>
    src/app/api/board/events/route.ts,
    src/app/api/board/events/route.test.ts,
    src/hooks/useBoardRealtimeSync.ts,
    src/hooks/useBoardRealtimeSync.test.ts
  </files>
  <behavior>
    SSE Endpoint Tests (src/app/api/board/events/route.test.ts):

    Test 1: GET /api/board/events returns response with Content-Type: text/event-stream
    Test 2: When boardEventEmitter.emit('task-moved', event) is called,
      the SSE stream sends a message in format: `data: {"type":"task-moved",...}\n\n`
    Test 3: Response includes headers Cache-Control: no-cache and Connection: keep-alive

    useBoardRealtimeSync Hook Tests (src/hooks/useBoardRealtimeSync.test.ts):

    Test 4: Hook creates an EventSource pointing to '/api/board/events' on mount
    Test 5: When EventSource receives a 'task-moved' message,
      hook calls the provided moveTask callback with (taskId, toColumnId, order)
    Test 6: Hook closes EventSource on unmount (EventSource.close() called)
    Test 7: Hook does NOT call moveTask if event.taskId is already in event.toColumnId
      (prevents infinite loop from own mutations being echoed back)

    Cases:
      GET /api/board/events → 200 + text/event-stream headers
      emit task-moved event → SSE data written with correct JSON shape
      mount hook → EventSource created at correct URL
      receive task-moved message → moveTask(taskId, toColumnId, order) called
      unmount hook → EventSource.close() called
      receive event where task already in correct column → moveTask NOT called
  </behavior>
  <implementation>
    Server (src/app/api/board/events/route.ts):
    - Create a module-level `boardEventEmitter = new EventEmitter()` (or import global singleton)
    - Export GET() returning new Response(readable, { headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', 'Connection': 'keep-alive' } })
    - ReadableStream controller writes SSE data when boardEventEmitter emits 'task-moved'
    - Export `boardEventEmitter` so PATCH /api/tasks/[id] can call boardEventEmitter.emit()

    Hook (src/hooks/useBoardRealtimeSync.ts):
    ```typescript
    export function useBoardRealtimeSync(
      columns: Column[],
      moveTask: (taskId: TaskId, toColumnId: ColumnId, order: number) => void
    ): void
    ```
    - useEffect: create EventSource('/api/board/events')
    - es.onmessage: parse event.data as TaskMovedEvent
    - Guard: find current column of taskId in columns; if already equals toColumnId, skip
    - Otherwise: call moveTask(taskId, toColumnId, order)
    - Return cleanup: es.close()

    Update src/app/api/tasks/[id]/route.ts (PATCH handler) to import and call
    `boardEventEmitter.emit('task-moved', event)` after successful DB write.
    NOTE: This modifies a file owned by Plan 03 — Plan 06 runs in Wave 3 AFTER Plan 03
    is complete, so there is no parallel conflict.
  </implementation>
</feature>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/phases/03-task-board/03-04-SUMMARY.md

<interfaces>
From src/types/board.ts:
```typescript
export interface TaskMovedEvent { type: 'task-moved'; taskId: TaskId; fromColumnId: ColumnId; toColumnId: ColumnId; order: number; timestamp: number; }
```

From src/hooks/useBoardState.ts (Plan 04 output):
```typescript
export interface UseBoardStateReturn {
  columns: Column[];
  moveTask: (taskId: TaskId, targetColumnId: ColumnId, targetOrder: number) => Promise<void>;
}
```
</interfaces>
</context>

<verification>
vitest run src/app/api/board/events/route.test.ts
vitest run src/hooks/useBoardRealtimeSync.test.ts
</verification>

<success_criteria>
- Failing tests committed: `test(03-06): add failing tests for SSE endpoint and sync hook`
- Implementation passes: `feat(03-06): implement SSE board events and useBoardRealtimeSync`
- Refactor if needed: `refactor(03-06): extract SSE helpers`
- All 7 tests pass. Especially: Test 7 (no-op when already in correct column) must pass.
</success_criteria>

<output>
After completion, create `.planning/phases/03-task-board/03-06-SUMMARY.md`
</output>
```

**Testing decision for Plan 06:**
- TDD plan. Two sets of tests: one for the SSE server endpoint, one for the client hook.
- Runner: `vitest run [file]` for both files.
- Why TDD: The SSE endpoint has a clear contract (Content-Type header, data format). The hook has a clear behavioral contract (EventSource created, callback called, cleanup on unmount). Most importantly, Test 7 (guard against echoing own mutations) is the bug that always gets missed without TDD.
- The "already in correct column" guard test is the key TDD benefit here — without writing the test first, this case is almost never caught in manual testing.
- Wave 3 because it imports from Plan 04's output types and modifies Plan 03's file after Plan 03 completes.

---

### PLAN 07 — Wiring: dnd-kit integration and board page assembly

```markdown
---
phase: 03-task-board
plan: 07
type: execute
wave: 4
depends_on: [02, 03, 04, 05, 06]
files_modified:
  - src/components/board/DraggableTaskCard.tsx
  - src/components/board/DroppableBoardColumn.tsx
  - src/app/board/page.tsx
  - src/app/board/page.test.tsx
autonomous: false
requirements: [BOARD-02, BOARD-03]

must_haves:
  truths:
    - "Board page loads and displays columns with task cards"
    - "User can drag a task card and drop it onto a different column"
    - "Dropped card moves to new column immediately (optimistic update visible)"
    - "Full drag-and-drop flow works end-to-end in browser"
  artifacts:
    - path: "src/components/board/DraggableTaskCard.tsx"
      provides: "TaskCard wrapped with dnd-kit useDraggable"
      exports: ["DraggableTaskCard"]
    - path: "src/components/board/DroppableBoardColumn.tsx"
      provides: "BoardColumn wrapped with dnd-kit useDroppable"
      exports: ["DroppableBoardColumn"]
    - path: "src/app/board/page.tsx"
      provides: "Full board page composing DndContext + columns + cards + realtime sync"
      exports: ["default (page component)"]
  key_links:
    - from: "src/app/board/page.tsx"
      to: "useBoardState"
      via: "const { columns, moveTask } = useBoardState(initialData)"
      pattern: "useBoardState"
    - from: "src/app/board/page.tsx"
      to: "useBoardRealtimeSync"
      via: "useBoardRealtimeSync(columns, moveTask)"
      pattern: "useBoardRealtimeSync"
    - from: "DndContext onDragEnd"
      to: "moveTask"
      via: "moveTask(active.id, over.id, targetOrder)"
      pattern: "onDragEnd"
---

<objective>
Wire dnd-kit drag-and-drop onto the existing components and assemble the full board page.

Purpose: All logic (state, API, realtime) is pre-tested. This plan is glue code connecting tested pieces. The checkpoint confirms the drag interaction works visually.
Output: A working board page with drag-and-drop and real-time sync.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/STATE.md
@.planning/phases/03-task-board/03-02-SUMMARY.md
@.planning/phases/03-task-board/03-04-SUMMARY.md
@.planning/phases/03-task-board/03-05-SUMMARY.md
@.planning/phases/03-task-board/03-06-SUMMARY.md

<interfaces>
From src/hooks/useBoardState.ts:
```typescript
export function useBoardState(initial: BoardState): UseBoardStateReturn
export interface UseBoardStateReturn { columns: Column[]; moveTask: (taskId, targetColumnId, targetOrder) => Promise<void>; }
```

From src/hooks/useBoardRealtimeSync.ts:
```typescript
export function useBoardRealtimeSync(columns: Column[], moveTask: Function): void
```

From src/components/board/TaskCard.tsx:
```typescript
export interface TaskCardProps { task: Task; onDragStart?: (taskId) => void; onDragEnd?: () => void; }
```

From src/components/board/BoardColumn.tsx:
```typescript
export interface BoardColumnProps { column: Column; onTaskDrop?: (taskId, targetColumnId, targetOrder) => void; }
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: dnd-kit draggable/droppable wrappers</name>
  <files>
    src/components/board/DraggableTaskCard.tsx,
    src/components/board/DroppableBoardColumn.tsx
  </files>
  <action>
    Install dnd-kit if not present: `npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities`

    Create DraggableTaskCard.tsx:
    - Import `useDraggable` from '@dnd-kit/core'
    - Wrap TaskCard: `const { attributes, listeners, setNodeRef, transform, isDragging } = useDraggable({ id: task.id })`
    - Apply `style={{ transform: CSS.Translate.toString(transform), opacity: isDragging ? 0.5 : 1 }}` to wrapper div
    - Forward all TaskCardProps, pass `setNodeRef` and `{...listeners, ...attributes}` to drag handle div
    - Export DraggableTaskCard

    Create DroppableBoardColumn.tsx:
    - Import `useDroppable` from '@dnd-kit/core'
    - Wrap BoardColumn: `const { isOver, setNodeRef } = useDroppable({ id: column.id })`
    - Apply `ref={setNodeRef}` and `className={cn(..., isOver && 'bg-blue-50 border-blue-300')}` for drop highlight
    - Forward all BoardColumnProps
    - Export DroppableBoardColumn

    Neither component has separate test files — they are glue wrappers around already-tested
    components using a well-tested library. The integration test in Task 3 covers them.
  </action>
  <verify>
    <automated>npx tsc --noEmit</automated>
  </verify>
  <done>
    DraggableTaskCard.tsx and DroppableBoardColumn.tsx compile without TypeScript errors.
    dnd-kit package is in package.json.
  </done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Board page assembly</name>
  <files>src/app/board/page.tsx, src/app/board/page.test.tsx</files>
  <behavior>
    - Test 1: Board page renders column titles from fetched data
    - Test 2: Board page renders task cards within their correct columns
    - Test 3: Board page renders DndContext wrapping the columns
    - Test 4: useBoardState is called with the initial server-fetched board data
    - Test 5: useBoardRealtimeSync is called with current columns and moveTask
  </behavior>
  <action>
    Create src/app/board/page.tsx as a Next.js page component:

    ```typescript
    // Server component fetches initial data
    export default async function BoardPage() {
      const res = await fetch('/api/board', { cache: 'no-store' })
      const initial: BoardState = await res.json()
      return <BoardClient initial={initial} />
    }
    ```

    Create src/app/board/BoardClient.tsx (client component, "use client"):
    - Import useBoardState, useBoardRealtimeSync, DndContext, DragEndEvent from @dnd-kit/core
    - Import DroppableBoardColumn, DraggableTaskCard
    - const { columns, moveTask } = useBoardState(initial)
    - useBoardRealtimeSync(columns, moveTask)
    - onDragEnd handler: extract active.id (taskId), over.id (targetColumnId),
      compute targetOrder as over column's tasks.length,
      call moveTask(taskId, targetColumnId, targetOrder)
    - Render: `<DndContext onDragEnd={handleDragEnd}>` wrapping a flex-row div of DroppableBoardColumn components, each with DraggableTaskCard children

    page.test.tsx: mock useBoardState, useBoardRealtimeSync, @dnd-kit/core, then render and assert.
  </action>
  <verify>
    <automated>vitest run src/app/board/page.test.tsx</automated>
    <automated env="post-merge">npx playwright test tests/board.spec.ts</automated>
  </verify>
  <done>
    Board page renders columns and cards. All 5 component tests pass.
    Page correctly wires useBoardState, useBoardRealtimeSync, and DndContext.
  </done>
</task>

<task type="checkpoint:human-verify" gate="blocking">
  <what-built>
    Full board page with drag-and-drop and real-time sync.
    All plans from Wave 1-4 are complete and integrated.
    Board page is accessible at http://localhost:3000/board
  </what-built>
  <how-to-verify>
    1. Run `npm run dev`, open http://localhost:3000/board
    2. Verify columns display with task cards showing title, priority badge, and assignee
    3. Drag a task card to a different column — it should move immediately (optimistic)
    4. Refresh the page — task should remain in the new column (persisted)
    5. Open a second browser tab at the same URL
    6. Move a card in Tab 1 — within 2-3 seconds, Tab 2 should show the card in the new column (real-time)
    7. Trigger a network error (DevTools → Network → Offline), drag a card, re-enable network — card should snap back to its original column (rollback)
  </how-to-verify>
  <resume-signal>Type "approved" if all 7 steps pass, or describe which step failed</resume-signal>
</task>

</tasks>

<verification>
vitest run src/app/board/page.test.tsx
npx playwright test tests/board.spec.ts  [post-merge]
Manual checkpoint: drag-and-drop, persist, real-time sync, rollback
</verification>

<success_criteria>
- Board page renders at /board with columns and tasks
- Drag-and-drop moves cards between columns with optimistic updates
- Page refresh preserves card positions (DB persisted)
- Second tab receives real-time updates within ~3 seconds
- Rollback on network failure reverts card to original column
- Human checkpoint approved
</success_criteria>

<output>
After completion, create `.planning/phases/03-task-board/03-07-SUMMARY.md`
</output>
```

**Testing decision for Plan 07:**
- Standard plan (not TDD) because this is glue code connecting existing tested components and libraries.
- Task 1 (dnd-kit wrappers): Pure connector code — wraps a well-tested library around already-tested components. The wrappers have no business logic. Typecheck (`tsc --noEmit`) is sufficient for the wrappers themselves. They ARE covered by the E2E test in Task 2's post-merge command.
- Task 2 (board page): `tdd="true"` at task level because it wires several hooks and we want to test the wiring contract (correct hooks called, DndContext present). Test file in `<files>`. Component test verifies wiring logic without requiring a browser.
- Task 2 also includes `env="post-merge"` E2E command — the drag-and-drop interaction and real-time sync REQUIRE a real browser. This is exactly what `env="post-merge"` is for: behavior that cannot be verified in unit/component tests because it requires all plans merged, a running server, and a real browser.
- Task 3 is a `checkpoint:human-verify` — the drag feel, animation smoothness, and cross-tab sync timing require human eyes. Automated tests can verify behavior but not usability.

---

## Phase Integration Tests

These are the `env="post-merge"` tests that run after all 7 plans are merged into the main branch.

### File: `tests/board.spec.ts` (Playwright)

| Test Name | Flow Covered | Runner | Why E2E |
|---|---|---|---|
| "displays columns and task cards on initial load" | GET /api/board → board renders | Playwright | Requires real HTTP server + DB |
| "moves task card to new column via drag-and-drop" | drag task → optimistic update → PATCH → DB write | Playwright | Drag-and-drop requires a real browser; programmatic dispatchEvent cannot simulate dnd-kit's pointer events |
| "persists card position across page reload" | drag → refresh → board re-loads from DB | Playwright | Tests the full DB persistence loop end-to-end |
| "syncs card move to second browser tab in real time" | drag in tab 1 → SSE event → board updates in tab 2 | Playwright | SSE real-time requires two browser contexts running simultaneously |
| "rolls back card on API failure" | drag → mock 500 → card reverts | Playwright | Rollback visibility requires actual render cycle; brittle to mock at unit level |

**Runner:** `npx playwright test tests/board.spec.ts`

**Env annotation:** All 5 tests are `env="post-merge"` — they appear in Plan 07, Task 2's `<automated env="post-merge">` command. They need all parallel plans merged (DndContext from Plan 07, useBoardState from Plan 04, SSE from Plan 06) and a running Next.js dev server with a seeded database.

**Playwright annotations used:**
- Tests 1-3: `test()` (single browser context)
- Test 4: `test()` with `const context2 = await browser.newContext()` and `const page2 = await context2.newPage()` — two tabs
- Test 5: `test()` with `page.route('/api/tasks/*', route => route.fulfill({ status: 500 }))` to simulate failure

---

## Testing Coverage Summary

| Plan | Test File(s) | Runner Command | Type | Rationale |
|------|-------------|----------------|------|-----------|
| 01 — Types + Schema | none | `npx tsc --noEmit` | typecheck | Pure type declarations and config — no runtime logic to test |
| 02 — GET /api/board | `src/app/api/board/route.test.ts` | `vitest run src/app/api/board/route.test.ts` | TDD unit | Clear request/response contract; 3 behavior tests written before implementation |
| 03 — PATCH /api/tasks/[id] | `src/app/api/tasks/[id]/route.test.ts` | `vitest run src/app/api/tasks/[id]/route.test.ts` | TDD unit | 4 cases including validation errors; test-first prevents skipping error paths |
| 04 — useBoardState | `src/hooks/useBoardState.test.ts` | `vitest run src/hooks/useBoardState.test.ts` | TDD unit | State machine with rollback; rollback test is the one always skipped without TDD |
| 05 — UI Components | `src/components/board/TaskCard.test.tsx`, `src/components/board/BoardColumn.test.tsx` | `vitest run src/components/board/TaskCard.test.tsx` + `vitest run src/components/board/BoardColumn.test.tsx` | Component (task-level tdd) | Visual components need render tests; not TDD plans but `tdd="true"` tasks enforce test-first |
| 06 — SSE + Sync Hook | `src/app/api/board/events/route.test.ts`, `src/hooks/useBoardRealtimeSync.test.ts` | `vitest run [file]` (both) | TDD unit | SSE contract + no-op guard are the hardest cases to discover without test-first |
| 07 — Wiring + Page | `src/app/board/page.test.tsx` | `vitest run src/app/board/page.test.tsx` | Component (task-level tdd) | Wiring test confirms correct hooks called; glue wrappers covered by E2E |
| Phase integration | `tests/board.spec.ts` | `npx playwright test tests/board.spec.ts` | E2E (post-merge) | Drag-and-drop, persistence, cross-tab real-time, rollback require real browser |

**Total test files:** 8
**Total plans:** 7 (4 TDD, 3 standard with task-level tdd)
**Automated verification coverage:** Every code-producing task has an `<automated>` command. The only tasks using typecheck-only verification are the type file and schema tasks in Plan 01, which contain no runtime logic.
