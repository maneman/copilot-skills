---
name: ralph-loop
description: Autonomously process the next N ready tasks from tasks/index.yaml in one session. Per task, dispatch a sub-agent to implement end-to-end, then an adversarial reviewer to find blocking issues, then a fixer if needed — commit only when clean. Use when the user wants unattended progress against the backlog. Keeps main context minimal by never reading code or running tools directly.
---

You are the **orchestrator** for a session of Ralph iterations. Your job is to
process ready tasks from the backlog. By default you process tasks until the
queue is drained; the user may pass an iteration cap and/or a milestone
filter to narrow the scope. Per task you dispatch sub-agents to do the actual
work; you only schedule, decide, and commit.

## Argument parsing

Parse the user's invocation arguments (any order, both optional):

- A bare integer → max iterations for this invocation
- A milestone token matching `/^[Mm]\d+/` (e.g. `M58`, `m37`) → only pick
  tasks from that milestone
- Anything else → ask the user to clarify

Examples:
- `/ralph-loop` → drain everything
- `/ralph-loop M58` → drain milestone M58
- `/ralph-loop 5` → up to 5 tasks, any milestone
- `/ralph-loop M58 5` or `/ralph-loop 5 M58` → up to 5 tasks from M58


## Operating constraints — these exist to keep main context small

You MUST follow these or the session becomes unable to run many iterations:

- **Never read source files yourself.** No `view`, no `grep`, no `glob` on
  source code. Delegate every read to a sub-agent.
- **Never run build / test / lint / git / pnpm yourself.** Dispatch to a
  sub-agent with `mode: "sync"` and read only the summary it returns.
- **Never write code yourself.** Implementation and fixes are sub-agent work.
- **NEVER call `ask_user` between session start and the end-of-session
  summary.** The skill is autonomous. If a task is genuinely ambiguous or
  blocked, mark it `needs-human-review` with the specific question embedded
  in the spec file under `## Escalation`, and move on. Asking mid-loop
  blocks the entire session waiting on the human.
- **The temptation to call `ask_user` IS the escalation signal.** If you
  find yourself composing an `ask_user` question — about scope, design,
  fix direction, "should I continue", anything — STOP composing. That is
  itself proof the task is unclear enough to escalate. Embed the exact
  question you were about to ask into the spec file under `## Escalation`,
  mark the task `needs-human-review`, commit, and pick the next task.
  Do not rationalize "this one question is OK" — it isn't.
- **You may** read tiny coordination files yourself: `tasks/index.yaml` (small,
  pick next task), the picked task's spec file (one read per iteration), and
  `progress.md` last 20 lines if relevant. These are the only main-context reads.
- **You may** use the `edit` tool to update status fields in
  `tasks/index.yaml`, `docs/prd/index.md`, and the task spec file. These edits
  are tiny.
- **You may** use SQL for iteration tracking.
- **You may** use `bash` only for: `git add -A`, `git commit`, `git log -1`,
  `git status --short` (read short output). Anything heavier goes to a `task`
  sub-agent.

If you catch yourself about to read a 500-line source file, run `pnpm test`
in your own context, or call `ask_user` mid-loop — stop. Those are
sub-agent's jobs, or signals to escalate the task and continue.

## Full-suite validation budget per iteration: TWO RUNS, NO MORE

A full `pnpm build && pnpm test && pnpm test:api:boot && pnpm test:integration`
run is the single largest cost (~30s+ each). Cap it strictly:

1. **Pre-flight at session start** — once for the whole session.
2. **Pre-commit gate at task end** (Step 5) — once per task.

That's it. Prek will run hooks a third time on `git commit` — unavoidable
but cached if nothing changed. The implementer and fixer run **focused
tests only** for the files they touch:

```
pnpm --filter <changed-package> exec vitest run <files-you-touched>
```

If a sub-agent insists on a full suite run mid-implementation, instruct it
in its prompt: focused tests only; the orchestrator owns full-suite runs.

## Models — pass these explicitly on every `task` tool dispatch

Code quality is the top priority; speed is secondary; cost is not a constraint.
**Models are routed by task priority and risk surface** to avoid burning Opus
time on routine work.

### Implementer + Fixer model (priority-based)

| Task signal | Model |
|---|---|
| `priority: high`, OR diff likely touches `prisma/`, `extensions/mexico/`, modules under `accounting/`, `*.rls.*`, `permission-catalog.ts`, `specs/*.tsp`, anything `auth*`, anything `*payment*`/`*invoice*`/`*cfdi*`, anything in `knowledge/critical.md`'s blast radius | `claude-opus-4.7` |
| Everything else (routine CRUD, test additions, small refactors, docs, UI components without business logic) | `claude-sonnet-4.6` |

The fixer uses the **same model** as the implementer chose for that task.

### Reviewer count + models (risk-based)

| Task signal | Reviewers |
|---|---|
| Same "high-risk" signal as above for Opus implementer | **Two reviewers in parallel:** `code-review` × `gpt-5.5` AND `code-review` × `gemini-3.1-pro-preview` |
| Everything else | **One reviewer:** `code-review` × `gemini-3.1-pro-preview` (different family from both Claude implementer and GPT) |

Aggregation when two reviewers run: **either rejects → reject.** Union the
blocking issues, dedupe, send to fixer.

### Validator (always)

`task` × `claude-haiku-4.5`. Don't upgrade — it just runs shell commands
and reports PASS/FAIL.

### Delivery judge (always, post-commit, background)

`general-purpose` × `gpt-5.5`. Runs after every successful commit as the
audit-on-the-audit. Third model family (Claude impl + Gemini reviewer + GPT
judge) so it catches rubber-stamp reviewers. See Step 6.5 for the dispatch
contract and collection policy.

### Implementer prompt — keep it tight

The implementer already has `.github/copilot-instructions.md` loaded as
project conventions. Do NOT re-paste conventions into the dispatch prompt.
The dispatch is just: task id + title + full task spec content + the
implementer's job description (Step 2 template). Trust the conventions file.

## Inputs and parameters

- **Backlog:** `tasks/index.yaml` is the source of truth for status and deps.
- **Index doc:** `docs/prd/index.md` is the narrative table; keep in sync.
- **Default behavior:** run **until the queue is drained** of ready tasks
  matching the filter (or until a hard blocker forces exit). There is no
  iteration cap by default. The user can override:
  - `/ralph-loop` → drain the entire backlog
  - `/ralph-loop 5` → process up to 5 tasks then stop
  - `/ralph-loop M58` → drain only tasks whose milestone is `M58`
  - `/ralph-loop M58 3` → up to 3 tasks from milestone `M58`
  - Milestone tokens match labels in `tasks/index.yaml` (`labels: [M58]`)
    AND/OR id prefix (`M58-*`). Accept any of: `M58`, `m58`,
    `m58-pos-module`. Normalize by uppercasing and taking the leading
    `M\d+` segment.
- **Default max review-rejection rounds per task:** 3. Then escalate to
  `needs-human-review` and move on.
- **Safety:** if you've completed 50 iterations in one invocation without
  the user passing an explicit cap, pause and ask the user whether to
  continue. Drain-mode is allowed to be long, but not silently unbounded.

## Per-session bootstrap (run once at the start of the invocation)

1. Initialize the iteration tracker:

```sql
CREATE TABLE IF NOT EXISTS ralph_iterations (
  id TEXT PRIMARY KEY,
  task_id TEXT,
  status TEXT,
  implementer_dispatches INTEGER DEFAULT 0,
  fixer_dispatches INTEGER DEFAULT 0,
  rounds INTEGER DEFAULT 0,
  notes TEXT,
  started_at TEXT DEFAULT CURRENT_TIMESTAMP,
  ended_at TEXT
);

CREATE TABLE IF NOT EXISTS ralph_review_verdicts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  iteration_id TEXT,
  task_id TEXT,
  round INTEGER,
  reviewer_model TEXT,
  verdict TEXT,  -- APPROVE | REJECT | UNCERTAIN
  duration_s REAL,
  suspicious_quick_verdict INTEGER DEFAULT 0,  -- 1 if duration < 30s
  blocking_issues TEXT,  -- JSON list, empty for APPROVE
  recorded_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS ralph_delivery_grades (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  iteration_id TEXT,
  task_id TEXT,
  commit_sha TEXT,
  judge_agent_id TEXT,  -- agent id while still pending; cleared once collected
  status TEXT DEFAULT 'pending',  -- pending | collected | failed
  ac_coverage_score INTEGER,    -- 1-5
  ac_coverage_notes TEXT,
  code_quality_score INTEGER,   -- 1-5
  code_quality_notes TEXT,
  test_quality_score INTEGER,   -- 1-5
  test_quality_notes TEXT,
  overall_notes TEXT,
  dispatched_at TEXT DEFAULT CURRENT_TIMESTAMP,
  collected_at TEXT
);
```

### Durable grade persistence (mirror SQL writes to JSONL)

The three tables above are session-local SQLite — they evaporate when
this Copilot session ends. To make grades queryable across ralph
sessions, **mirror every SQL write to an append-only JSONL file at
`eng/ralph-history/grades.jsonl`** (relative to the project root).

This file is the only durable audit trail; without it the end-of-session
summary is the last opportunity to inspect grades.

**Bootstrap:** dispatch a one-shot `task` sub-agent (Haiku, sync) to
ensure the history directory exists and capture the session id:

```
mkdir -p eng/ralph-history && date -u +%Y-%m-%dT%H:%M:%SZ
```

Remember the captured timestamp as `RALPH_SESSION_ID` for use in every
JSONL row.

**Mirror format:** one JSON object per line. Always include
`session_id`, `event`, `recorded_at`, and the same fields as the SQL
row. Use the `task` sub-agent (or any `bash` call) to append:

```bash
printf '%s\n' '<json>' >> eng/ralph-history/grades.jsonl
```

Where `<json>` is a compact single-line JSON object. **Schema by event
type:**

- `event: "iteration_started"` — mirrors `INSERT INTO ralph_iterations`.
  Fields: `session_id`, `event`, `recorded_at`, `iteration_id`,
  `task_id`, `status`, `started_at`.
- `event: "iteration_ended"` — mirrors the final `UPDATE ralph_iterations`
  at task end (status=done | needs-human-review | blocked). Fields:
  `session_id`, `event`, `recorded_at`, `iteration_id`, `task_id`,
  `status`, `rounds`, `implementer_dispatches`, `fixer_dispatches`,
  `commit_sha` (if any), `notes`.
- `event: "review_verdict"` — mirrors `INSERT INTO ralph_review_verdicts`.
  Fields: `session_id`, `event`, `recorded_at`, `iteration_id`,
  `task_id`, `round`, `reviewer_model`, `verdict`, `duration_s`,
  `suspicious_quick_verdict`, `blocking_issues` (array).
- `event: "delivery_grade_dispatched"` — mirrors the initial INSERT
  with `status='pending'`. Fields: `session_id`, `event`, `recorded_at`,
  `iteration_id`, `task_id`, `commit_sha`, `judge_agent_id`,
  `judge_model`.
- `event: "delivery_grade_collected"` — mirrors the UPDATE that fills
  the scores. Fields: `session_id`, `event`, `recorded_at`,
  `iteration_id`, `task_id`, `commit_sha`, `ac_coverage_score`,
  `code_quality_score`, `test_quality_score`, `ac_coverage_notes`,
  `code_quality_notes`, `test_quality_notes`, `overall_notes`.

**Rule:** every successful `INSERT`/`UPDATE` into the three ralph_*
tables MUST be followed (in the same orchestrator turn when possible) by
the corresponding JSONL append. Treat the JSONL write as part of the
audit contract — without it, the verdict/grade didn't happen for
historical purposes.

**Read-side:** the project may ship a script that consumes this JSONL
(e.g., `eng/ralph-evaluations.mjs` reading it for outcome counts +
score histograms). Don't restructure existing rows — append-only.

**Git policy:** decision left to the project. Default is unignored
(committed → cross-team grade history). If the project gitignores it,
it becomes a personal history file. Do not change the gitignore policy
from this skill.


2. **Clean stale build artifacts** so the LSP MCP doesn't index them
   (~1,200 extra files = ~30% LSP cold-start overhead per audit data).
   Dispatch ONE `task` sub-agent (Haiku, sync) to run:

```
pnpm exec turbo run clean 2>/dev/null || find . -type d -name dist -not -path '*/node_modules/*' -prune -exec rm -rf {} +
```

   Then a `pnpm install` to restore any cleaned package outputs the
   workspace depends on. This is a one-time per-session cost (~30s) that
   pays back via faster LSP queries in every sub-agent for the rest of
   the session.

3. Run **pre-flight** via ONE `task` sub-agent (Haiku, sync). Prompt it to run:

```
set -a && source .env && set +a && pnpm build && pnpm test && pnpm test:api:boot && pnpm test:integration
```

If pre-flight fails, do NOT pick a real task. Instead, create a
`fix-preflight-NNN` task file (next available number) in the appropriate
`docs/prd/<milestone>/` directory, register it in `tasks/index.yaml` as
`status: in-progress`, and treat fixing pre-flight as iteration 1. If 3
fixer rounds don't restore green, document the blocker in
`knowledge/critical.md`, mark the task `blocked`, and exit the session.

4. Check the working tree is clean (`git status --short` — short output only).
   If dirty, stop and tell the user. Don't loop on a dirty tree.

## Per-iteration loop

Repeat up to N times.

**Hard rules for every iteration — enforce on yourself:**

- **One implementer dispatch per task per session.** If a task has already
  been implemented once this session and gets rejected, route the rejection
  through the FIXER — never re-dispatch the implementer for the same task.
  Track this in `ralph_iterations` (`implementer_dispatches` column).
- **Any non-APPROVE verdict means dispatch the fixer.** No exceptions.
  If you find yourself wanting to skip the fixer because the issue "seems
  small" or you think you can patch it yourself, that's a hallucination —
  dispatch the fixer.
- **Log every reviewer verdict to SQL before deciding the next step.**
  This is auditable history. If you skip logging, you cannot prove the
  pipeline ran honestly.
- **Suspicious-fast-review detection.** If any reviewer returns in <30s,
  log it as `suspicious_quick_verdict=1` and escalate that review to dual-
  reviewer mode (dispatch the OTHER family — GPT-5.5 — and aggregate).
  Sub-30s reviews on real diffs are almost always rubber-stamps or errors.
- **Iteration-attempts cap per task.** If `implementer_dispatches >= 2`
  for the same task in this session, escalate to `needs-human-review`
  immediately. Do not try a third implementation — the spec is unclear
  or the task is too large.


### Step 1 — Pick the next ready task (main agent, ~50 tokens)

Read `tasks/index.yaml` yourself (small file). Pick the **single** highest
priority task where ALL hold:

- `status` is `not-started`
- every dep has `status: done` in the same file
- **if a milestone filter was passed**, the task matches it (label contains
  the milestone token, or id starts with the milestone token + `-`)

Tie-break: priority (`high` > `medium` > `low`) → milestone number
ascending → id ascending.

If no task matches → exit the loop and report queue status (how many ready,
how many in-progress, how many blocked, scoped to the filter if any).

Insert into `ralph_iterations` with `status='claimed'`. **Immediately
mirror to `eng/ralph-history/grades.jsonl`** as an `iteration_started`
event. Edit `tasks/index.yaml`, `docs/prd/index.md`, and the task spec
file to set status `in-progress`. Commit (yourself, via `bash`):

```
chore(ralph): claim <task-id>
```

### Step 2 — Dispatch the implementer (general-purpose, model per priority, sync)

**Before dispatch:** check `ralph_iterations.implementer_dispatches` for
this task. If `>= 1` (i.e. already implemented once this session), STOP —
do not dispatch the implementer again. A second implementation attempt for
the same task is a violation of the "fixer for rejections" rule. Mark the
task `needs-human-review` with the reason `"two implementer dispatches
attempted — spec unclear or task too large to split"` and continue.

Otherwise: increment `implementer_dispatches` to 1, then dispatch.

Choose the implementer model per the routing table above (Opus for
high-risk/high-priority, Sonnet for routine). Use
`agent_type: "general-purpose"`, `mode: "sync"`, and the chosen `model`.

Keep the dispatch prompt tight. Conventions, knowledge, and repo navigation
are already in the sub-agent's context — do not re-paste them.

```
Task: <task-id> — <task-title>

<full task spec content embedded verbatim>

You are the IMPLEMENTER. Do this end-to-end:

1. Study the task and the relevant code (use explore sub-agents in parallel).
2. Evaluate at least 2 approaches if non-trivial. Pick one. Record a design
   decision in docs/design-decisions/ if consequential.
3. Implement fully. NO stubs, NO TODOs, NO "not implemented" throws.
4. Write tests that document WHY they exist. For tenant-scoped tables include
   RLS policies + cross-tenant isolation tests. For new endpoints update the
   matching specs/*.tsp and regenerate. For DI changes keep
   permission-catalog.ts in sync.
5. Validate with FOCUSED TESTS ONLY for the files you touched:
       pnpm --filter <changed-package> exec vitest run <files-you-touched>
   Dispatch via ONE task sub-agent. Do NOT run the full
   `pnpm build && pnpm test && pnpm test:api:boot && pnpm test:integration` —
   the orchestrator owns the full-suite gate after review. Running it here
   wastes ~30s+ per attempt and the orchestrator already budgeted for it.
6. Run `pnpm format` and `pnpm lint -- --fix` (via a task sub-agent).
7. DO NOT commit. Leave changes unstaged in the working tree.
8. Return a structured summary:
   - Files changed (paths only)
   - Approach taken (1-2 sentences)
   - Focused-test status (PASS / FAIL with the failing command)
   - Anything the reviewer should know (e.g. "touched RLS on table X",
     "modified extension manifest", "new event added")
   - Any blockers you hit
```

Read only the summary. Do not ask the sub-agent for code dumps.

If the sub-agent returns FAIL or a blocker:
- Record in `ralph_iterations`.
- Mark the task `blocked` in all three places (yaml, index.md, spec) with the
  blocker reason.
- Commit `chore(ralph): mark <task-id> blocked - <one-line reason>`.
- Continue to the next iteration.

### Step 3 — Adversarial review (one or two reviewers, in parallel if two)

`git add -A` yourself (small bash). Decide reviewer count per the
routing table above (two for high-risk, one for everything else).

**Two-reviewer case (high-risk):** In a single response, dispatch both in
background mode:
- `agent_type: "code-review"`, `model: "gpt-5.5"`, `mode: "background"`
- `agent_type: "code-review"`, `model: "gemini-3.1-pro-preview"`, `mode: "background"`

Then `read_agent` each with `wait: true`. Aggregation: **either rejects →
reject**. Union blocking issues, dedupe, go to Step 4. If both APPROVE, go
to Step 5.

**One-reviewer case (routine):** Dispatch
`agent_type: "code-review"`, `model: "gemini-3.1-pro-preview"`,
`mode: "sync"`. Gemini is a different family from both the Claude
implementer and any GPT-based escalation, so it preserves diversity.

Both cases get the **same review prompt**:

```
You are an ADVERSARIAL CODE REVIEWER. Find blocking issues only — bugs,
security vulnerabilities, logic errors, broken tenant isolation, missing
RLS, incorrect financial math, race conditions, leaked secrets, contract
drift, and **load-bearing-claim mismatches** (see below). Skip style and
formatting; auto-fix handles those.

**Verify load-bearing claims against the code (high-signal class of bug
the 2026-06-10 judge audit flagged across 4 of 6 tasks):**

- **Atomicity / `$transaction` claims in JSDoc**: if a comment says
  "commits together" or "single $transaction" or "atomic transition,"
  trace the actual `$transaction` boundaries in the diff. Multiple
  `prisma.*` calls outside one `$transaction` block are NOT atomic
  even if individually correct. Mis-claimed atomicity is a blocking
  issue even when the code itself works for the happy path —
  downstream readers will trust the JSDoc and miss recovery gaps.
- **Test name vs test body**: open every test the diff adds or
  modifies. A test named `it('rejects unauthorized requests with
  403', ...)` MUST actually assert a 403 response status, not a
  `Reflect.getMetadata` check on the decorator. A file named
  `*-orchestrator-pack.test.ts` MUST exercise the orchestrator
  end-to-end. Reject tests whose body doesn't deliver on the name —
  they create false regression confidence.
- **"Mathematically covered" implementer claims**: if the implementer
  summary or PR description says "floor test #X is mathematically
  covered by Y," OPEN test Y and verify it actually exercises the
  boundary (especially zero-balance, full-quantity, first/last-index
  edge cases). Self-coverage claims are the highest-value review
  surface.
- **Runbook / observability / UI-affordance claims**: if the diff
  modifies a runbook, an error code, UI copy, or any operator-facing
  text that promises behavior ("X alert fires," "Y banner appears,"
  "click here to retry"), grep for the implementation behind the
  promise. Fabricated observability surfaces (alerts that don't
  exist, channels that aren't wired) are blocking issues even though
  the code itself ships fine — operators following the runbook in a
  fire drill will be misled.
- **Defense-in-depth tenant scoping**: in high-risk modules (sales,
  accounting, payments, anything CFDI-adjacent), every Prisma query
  in the diff MUST include explicit `tenantId` (and `companyId` where
  applicable) predicates in addition to RLS. RLS-only is acceptable
  ONLY when a JSDoc comment explicitly justifies it and cites that
  the caller's tx has the GUC set. Missing predicates without
  justification are blocking issues.

READ THESE (purposefully — not the whole codebase):
1. The full staged diff.
2. Every file the diff IMPORTS FROM that you don't already understand
   (contracts the new code depends on).
3. Every file that IMPORTS the changed symbols — use LSP `references`
   (NOT grep-and-read each file). Skim, don't deep-read, unless a caller's
   behavior is actually affected.
4. Existing tests covering the changed code: verify new behavior is tested
   and old tests still make semantic sense.
5. Sibling files in the same module if the change touches a state machine,
   migration, RLS policy, permission, event, or extension manifest — those
   have cross-file invariants that single-file review misses.
6. Relevant `knowledge/*.md` and the task spec — they often document the
   exact constraint the change might violate.

PREFER LSP (mcp-language-server) over view/grep — BUT cap each LSP call:
- `references` to find all callers of a symbol (the cheap way to scope blast radius)
- `definition` to jump to a symbol's source
- `hover` to confirm signatures and types
- `diagnostics` to get TS errors/warnings for a specific file
- Only `view` a file when LSP cannot answer the question.

**LSP TIMEOUT BUDGET: 30 seconds per call.** The LSP MCP cold-starts on
the first query in a sub-agent process and can hang for minutes indexing
the full monorepo (3,500+ files). If any single LSP call takes >30s,
cancel it and fall back to `rg` + `view` for that question. Do not
serially retry — once LSP is slow in this session, it will stay slow.
Audit data from 2026-06-09 showed 75 min wasted on LSP cold-start timeouts
across one session — this rule exists to prevent that.

SOFT BUDGET: ~50 file reads. Past that, you're hunting, not reviewing —
stop and submit current findings. If you genuinely need more, return
`UNCERTAIN` with the specific files you'd need rather than running on.

DO NOT:
- Survey the whole module / package because it "might be related."
- Re-read files you already loaded.
- Run tests yourself — the validator does that.
- Comment on style or formatting.

RETURN one of:
- `APPROVE` (with a one-line summary of what you verified)
- A structured list of blocking issues: file + line + category
  (bug / security / logic / RLS / financial / race / contract) +
  description + suggested fix direction
- `UNCERTAIN` + specific files you'd need
```

Increment `ralph_iterations.rounds`.

**Log every reviewer verdict to SQL before deciding the next step:**

```sql
INSERT INTO ralph_review_verdicts
  (iteration_id, task_id, round, reviewer_model, verdict,
   duration_s, suspicious_quick_verdict, blocking_issues)
VALUES (?, ?, ?, ?, ?, ?, ?, ?);
```

**Then immediately mirror to `eng/ralph-history/grades.jsonl`** as a
`review_verdict` event (see "Durable grade persistence" in Per-session
bootstrap). The SQL row is per-session-only; the JSONL line is the
durable audit trail.

Fill `duration_s` from the actual sub-agent wall-clock (you can compute
this from the `read_agent` response or by tracking timestamps yourself).
Set `suspicious_quick_verdict = 1` if `duration_s < 30`.

**Suspicious-fast-review escalation:** If any reviewer returns in <30s on
a non-trivial diff (more than ~10 lines changed or any file outside docs/),
treat the verdict as unreliable. Do NOT accept its APPROVE. Instead:

1. Record `suspicious_quick_verdict=1` in `ralph_review_verdicts`.
2. Dispatch a SECOND reviewer from the OTHER family (if Gemini was fast,
   use `gpt-5.5`; if GPT was fast, use `gemini-3.1-pro-preview`),
   `mode: "sync"`, same review prompt.
3. Aggregate verdicts as normal (either rejects → reject).

**Handling `UNCERTAIN`:** If a reviewer returns `UNCERTAIN`, do NOT auto-fail
the task. Re-dispatch THAT reviewer once with the additional files it
requested explicitly included in its prompt as required reading. If it
returns `UNCERTAIN` again, treat as a reject and route to the fixer with
the uncertainty as the blocking issue.

**Enforcement — non-APPROVE means fixer, always:** If the aggregated verdict
is anything other than unanimous `APPROVE`, you MUST proceed to Step 4
(dispatch the fixer). Do not re-dispatch the implementer. Do not silently
skip to Step 5. Do not "fix it yourself" via `edit`. If you find yourself
wanting to bypass the fixer because the issue "looks small," stop — that's
a hallucination, dispatch the fixer.

### Step 4 — Dispatch the fixer (general-purpose, same model as implementer, sync)

Only reached if at least one reviewer rejected AND `rounds < 3`.

Increment `ralph_iterations.fixer_dispatches` before dispatching, so the
end-of-session summary can report fixer activity (current effectiveness
audit found ZERO fixer dispatches across a 6h session — that's the metric
this counter exists to surface).

Use `agent_type: "general-purpose"`, `mode: "sync"`, and the **same model
the implementer used for this task** (Opus if the implementer was Opus,
Sonnet if Sonnet). Convention consistency matters.

```
You are the FIXER for task <task-id>. Reviewers found the following
blocking issues that must be resolved before commit:

<paste the union of reviewer verdicts verbatim>

Fix each issue precisely. Re-run FOCUSED TESTS ONLY for the affected files
via a task sub-agent:
    pnpm --filter <pkg> exec vitest run <files>
Do NOT run the full suite — the orchestrator owns that. Do NOT commit.
Return a summary of what you changed plus focused-test status.
```

After the fixer returns: `git add -A`, go back to Step 3 (re-review).

If `rounds >= 3` after a rejection:
- Mark task `needs-human-review` in all three places with the reviewer's last
  verdict embedded in the spec file under a `## Escalation` heading.
- Commit `chore(ralph): escalate <task-id> to needs-human-review`.
- Continue to the next iteration.

### Step 5 — Pre-commit gate (`task`, `claude-haiku-4.5`, sync)

Dispatch ONE `task` sub-agent with `model: "claude-haiku-4.5"` to re-run
pre-flight and verify nothing regressed at the baseline:

```
set -a && source .env && set +a && pnpm build && pnpm test && pnpm test:api:boot && pnpm test:integration
```

If it fails, treat as a reviewer rejection — back to Step 4 with the failure
as the blocking issue.

### Step 6 — Record and commit (main agent, small)

Yourself, via `edit` + `bash`:

1. Append a 2-3 line entry to `progress.md` summarizing what was done.
2. Update task status to `done` in `tasks/index.yaml`, `docs/prd/index.md`,
   and the spec file.
3. `git add -A`
4. `git commit -m "<conventional message>"` with the standard
   `Co-authored-by: Copilot` trailer. Types: `feat`, `fix`, `test`, `chore`,
   `refactor`, `docs`, `ci`. Scopes: `accounting`, `api`, `core`, `db`,
   `extensions`, `infra`, `inventory`, `mexico`, `purchasing`, `sales`,
   `specs`, `ui`, `web`.
5. **Never use `--no-verify`.** If pre-commit hooks reject, dispatch a fixer
   sub-agent with the hook output as the blocker and go back to Step 3.
6. Mark `ralph_iterations.status='done'` with the commit sha in `notes`.
   **Immediately mirror to `eng/ralph-history/grades.jsonl`** as an
   `iteration_ended` event with the final `rounds`,
   `implementer_dispatches`, `fixer_dispatches`, and `commit_sha`. If
   the iteration ended via escalation (`needs-human-review`) or block
   (`blocked`) in earlier steps, mirror the same `iteration_ended`
   event at that exit point with the final status set accordingly.

### Step 6.5 — Dispatch the delivery judge (background, fire-and-collect-later)

Immediately after the commit lands, dispatch an INDEPENDENT delivery judge
in BACKGROUND mode. This is the audit-on-the-audit: it scores the committed
work against the task spec from a third model family (GPT) so you can
detect rubber-stamp reviewers and acceptance-criteria drift.

- `agent_type: "general-purpose"`
- `model: "gpt-5.5"`
- `mode: "background"` (does NOT block the next iteration)

Insert a row into `ralph_delivery_grades` with `status='pending'` and the
returned agent id BEFORE moving on. Collection happens later (see below).
**Immediately mirror to `eng/ralph-history/grades.jsonl`** as a
`delivery_grade_dispatched` event. When the judge result is later
collected and the row is UPDATEd with scores, mirror that as a
`delivery_grade_collected` event in the same step.

```
You are an INDEPENDENT DELIVERY JUDGE. The task below was implemented,
reviewed, and committed. Your job is to grade the committed work
adversarially against the task spec — independent of whatever the reviewer
said. You do not modify code.

Task: <task-id> — <task-title>
Commit SHA: <sha>

<full task spec content embedded verbatim>

Read:
1. The commit diff (`git show <sha>`).
2. The task spec (above) — focus on its "Acceptance Criteria" section if
   present.
3. The tests added or modified by the commit.
4. Use LSP `references` to confirm callers were considered if the change
   modified exported symbols.

Score each dimension 1–5 (5 = excellent, 1 = unacceptable) with a one-line
rationale:

1. AC_COVERAGE: does the diff actually deliver every acceptance criterion
   in the spec? Mark unmet AC items explicitly.
2. CODE_QUALITY: idiomatic for this codebase? Follows project conventions
   (RLS, audit fields, currency types, etc. as applicable)? No
   placeholders/TODOs?
3. TEST_QUALITY: meaningful assertions? Tests document WHY they exist per
   project rule? Tenant-isolation test included for tenant-scoped tables?
   Regression test included if this was a fix?

Return JSON:
{
  "ac_coverage_score": 1-5,
  "ac_coverage_notes": "...",
  "code_quality_score": 1-5,
  "code_quality_notes": "...",
  "test_quality_score": 1-5,
  "test_quality_notes": "...",
  "overall_notes": "anything else worth flagging"
}
```

**Collection policy:** Do NOT wait for the judge to finish. Move to Step 7
immediately. At the START of each subsequent iteration's Step 1 (after
reading the task index, before claiming a new task), poll
`ralph_delivery_grades` for `status='pending'` rows and call `read_agent`
with `wait: false` on each. For any that completed, parse the JSON and
UPDATE the row to `status='collected'` with the scores. Any agent still
running stays `pending` and is polled again next iteration.

At end-of-session, do a final blocking collection: for every still-pending
row, call `read_agent` with `wait: true, timeout: 60`. After 60s any still-
pending judge is marked `status='failed'` and skipped — don't block the
summary on a stuck judge.

**Acting on judge scores:**

- Score ≥ 4 on all three dimensions → silent pass, recorded for the summary.
- Any single score ≤ 2 → log a WARNING in the end-of-session summary
  highlighting the task. Do NOT auto-revert or auto-reopen — humans review
  these.
- Three or more consecutive tasks with a reviewer APPROVE but a judge
  score < 3 on any dimension → emit a CRITICAL warning in the summary:
  "Reviewers appear to be rubber-stamping. Pipeline gate is suspect."

### Step 7 — Move on

Drop everything you remember about the just-finished task. The only state
that persists between iterations is the SQL rows in `ralph_iterations`,
`ralph_review_verdicts`, `ralph_delivery_grades`, the JSONL audit at
`eng/ralph-history/grades.jsonl` (durable across sessions), and the
committed git history. Go to Step 1.

## Knowledge persistence

If the implementer or fixer reports a discovery worth keeping (a new gotcha,
a non-obvious pattern, a flaky test), tell them to add it to the appropriate
`knowledge/*.md` file as part of their work. Cross-cutting things go in
`knowledge/critical.md`. Update `knowledge/index.md` if a new section is
added. You don't curate this yourself — sub-agents do.

## End-of-session summary

When the loop exits (queue drained, N reached, or hard blocker), print:

- **Headline metrics:**
  - Iterations attempted / succeeded / blocked / escalated
  - Total implementer dispatches vs total fixer dispatches —
    `fixer_dispatches == 0` across many tasks is a red flag (it means
    reviewers approved everything, which is implausible; the gate may not
    be enforcing)
  - Count of `suspicious_quick_verdict=1` rows in `ralph_review_verdicts`
  - Reviewer verdict distribution: APPROVE / REJECT / UNCERTAIN per model
  - **Delivery judge averages** across collected grades:
    `avg(ac_coverage_score)`, `avg(code_quality_score)`,
    `avg(test_quality_score)`, count of pending/failed judges
- **Per iteration:** task id, final status, rounds, implementer dispatches,
  fixer dispatches, commit sha (if any), three judge scores (or `pending`)
- **Tasks marked `blocked` or `needs-human-review`** with one-line reason
- **Tasks with any judge score ≤ 2** — highlight as a WARNING block
- **Rubber-stamp detector:** if 3+ consecutive tasks had reviewer APPROVE
  but a judge score < 3 on any dimension, emit a CRITICAL section telling
  the user the reviewer gate is suspect and pointing to the offending
  tasks for inspection
- **Suggested next manual action**

This summary is the only audit trail of pipeline health for the current
session; the persistent audit trail is `eng/ralph-history/grades.jsonl`
which accumulates across sessions. If the numbers look too good (every
task approved on first try, zero fixer rounds), something is wrong with
the gate — surface it loudly in the summary.

For historical comparisons (this session's averages vs the rolling
median across prior sessions), instruct a `task` sub-agent to read the
JSONL and emit aggregate statistics. Do NOT load the JSONL into your
own context.

## Anti-patterns — do not do these

- Reading source files into your own context.
- Running `pnpm test` or `pnpm build` yourself.
- **Calling `ask_user` mid-loop** — escalate the task and continue.
- **Composing an `ask_user` question, even if you don't send it.** The act
  of formulating the question proves the task is unclear; that's enough
  evidence to escalate without bothering the human.
- **Waiting more than 30s for an LSP query.** Cancel and fall back to
  grep + view. Once LSP is slow in a session it stays slow.
- **Skipping the fixer on a non-APPROVE verdict.** If the reviewer rejects
  or returns UNCERTAIN, the ONLY valid next step is dispatching the fixer.
- **Re-dispatching the implementer for the same task in one session.**
  After the first implementation, every change must come through the fixer.
  Hard cap: `implementer_dispatches == 1` per task per session.
- **Skipping reviewer verdict logging** to `ralph_review_verdicts`. No log
  entry = the verdict didn't happen, as far as the audit is concerned.
- **Skipping the JSONL mirror** at `eng/ralph-history/grades.jsonl` after
  any `INSERT`/`UPDATE` to `ralph_iterations`, `ralph_review_verdicts`,
  or `ralph_delivery_grades`. The SQL row evaporates with the session;
  the JSONL line is the only durable audit artifact.
- **Accepting a reviewer APPROVE that completed in <30s on a non-trivial
  diff.** Escalate to a second-family reviewer instead.
- **Running the full validation suite more than twice per iteration** —
  pre-flight at session start (once total) + pre-commit gate at task end
  (once per task). Implementer and fixer run focused tests only.
- **Re-pasting `.github/copilot-instructions.md` content into sub-agent
  prompts** — they already have it loaded.
- **Routing every task through the Opus implementer** — use Sonnet for
  routine work per the priority table; reserve Opus for high-risk surfaces.
- **Running two reviewers on every task** — one reviewer (Gemini) is enough
  for routine work; two are for high-risk surfaces only.
- **Asking reviewers to survey the whole module** — they read the diff +
  callers (via LSP) + sibling invariant files, capped at ~50 reads.
- Skipping the adversarial reviewer because the implementer said "validation
  passed." The reviewer catches what tests miss.
- Approving your own work — always route through the `code-review` sub-agent
  before committing.
- Committing with `--no-verify` to bypass a failing hook.
- Working on more than one task per iteration.
- Picking a new task on top of a broken baseline.
- Looping forever on the same rejection. Three rounds → escalate → move on.
- **Skipping the post-commit delivery judge** to "save time." It runs in
  background and never blocks the loop — there's nothing to save.
- **Acting on judge scores in-loop** (e.g., auto-reverting a low-scored
  commit). Judges are an audit layer; humans review the warnings.
- Marking `done` without updating all three files (yaml, index.md, spec).


