---
name: autoresearch
description: Use when executing an autonomous development loop — implementing tasks from prd.json with multi-agent coordination, quality gates, and PR creation. Requires autoresearch-plan to have been run first.
---

# autoresearch

Enterprise autonomous research loop. You are the **orchestrator only** — you do not write code. You coordinate sub-agents, run quality gates, manage state, and accumulate lessons in CLAUDE.md.

---

## Pre-flight

Verify before starting:
1. `.autoresearch/config.yaml` exists and has at least one scope defined — else tell user to run `/autoresearch:plan` and stop.
2. `.autoresearch/prd.json` has at least one `"done": false` entry — else stop.
3. `git status` succeeds — else stop.

---

## PHASE 0 — Scope Pre-check

Before entering the main loop, validate that tooling is available for every scope referenced by pending tasks.

For each scope referenced in `prd.json` (among `done: false` entries):

```
scope_config = config.yaml.scopes[scope_name]

If scope does not exist in config.yaml:
  Stop. Print: "[autoresearch] Scope '<X>' used in prd.json but not defined in config.yaml.
  Run /autoresearch:plan to add it."

If scope.quality_gates is empty or missing:
  Log warning: "[autoresearch] Warning: no quality gates configured for scope '<X>' — gates will be skipped."
  Guard still runs if defined.

If scope.guard is missing:
  Log warning: "[autoresearch] Warning: no guard configured for scope '<X>'."

For each gate command in scope.quality_gates:
  tool = first word of command (e.g., "ruff" from "ruff check .")
  Run: command -v <tool>
  If not found:
    If gate has optional: true → log warning and skip gate
    Else → Stop. Print: "[autoresearch] Gate '<gate-name>' requires '<tool>' which is not installed.
    Install it or update .autoresearch/config.yaml."
```

---

## Scope Resolution

Every USR has a `"scope"` field (e.g., `"backend"` or `"frontend"`). Before each phase, resolve the active scope's configuration from `config.yaml`:

```
active_scope = prd_entry.scope
scope_config = config.yaml.scopes[active_scope]

gate_commands  = scope_config.quality_gates
guard_command  = scope_config.guard.command
gate_root      = scope_config.root          # cd here before running gates
ac_test_file   = config.yaml.ac_test_dir + "/test_" + USR_ID + ".<ext>"
```

The file extension for AC tests is inferred from the scope's language (`.py` for Python scopes, `.test.ts` for TypeScript scopes, `_test.go` for Go scopes).

---

## Memory Loading

When loading `[autoresearch]` block from `CLAUDE.md` for a task:

1. Determine `active_scope` from the current USR's `scope` field.
2. Build the prompt block:
   - `[autoresearch] Core Invariants` — always include
   - `[autoresearch] Project Architecture` — always include
   - `[autoresearch] Known Pitfalls / <active_scope>` — include if exists, else fall back to `### Common`
   - `[autoresearch] Verified Patterns / <active_scope>` — include if exists, else fall back to `### Common`
   - `[autoresearch] Quality Standards / <active_scope>` — include if exists, else fall back to `### Common`
3. If a scope-specific section does not exist in `CLAUDE.md`:
   - Fall back to the corresponding `### Common` section.
   - Print: `[autoresearch] Note: no <section>/<scope> section found; falling back to Common. Consider adding scope-specific lessons.`

---

Repeat until all tasks have `"done": true`.

---

### PHASE 1 — Task Selection

Read `prd.json`. Filter to `"done": false` tasks.

**depends_on check**: For each candidate task, check its `depends_on` list. Skip any task where a listed dependency has `"done": false`. If all remaining tasks are blocked by unmet dependencies, stop and print:
```
[autoresearch] Blocked: all pending tasks have unmet dependencies.
  USR-002 waiting for: USR-001
Run /autoresearch again after merging the blocking PR.
```

Select the lowest-`priority` unblocked task.

Read `.autoresearch/tasks/<USR-ID>/state.json`. If `current_phase` is not `PHASE_1_SELECT`, a prior run was interrupted — **resume from that phase**.

**Feature branch creation**: Resolve the feature branch name from `prd.json`:
```bash
feature_id="<prd_entry.feature_id>"   # e.g. "user-auth" or "USR-003"
branch_name="feat/${feature_id}"       # e.g. "feat/user-auth"

current=$(git branch --show-current)
if [ "$current" != "$branch_name" ]; then
  # checkout existing branch (subsequent USR) or create new one (first USR)
  git checkout "$branch_name" 2>/dev/null || git checkout -b "$branch_name"
fi
```

Update `state.json`:
```json
{
  "active_task_id": "<USR-ID>",
  "feature_id": "<feature_id>",
  "branch_name": "<branch_name>",
  "current_phase": "PHASE_2_PLAN",
  "iteration": <N+1>,
  "fixer_attempts": 0,
  "last_commit": null,
  "started_at": "<ISO timestamp>"
}
```

Print: `[autoresearch] Starting <USR-ID> [<scope>]: <title> (iteration N, branch: <branch_name>)`

---

### PHASE 2 — Design & AC Test Generation

Spawn sub-agent with **autoresearch-coder** skill. Provide:
- Task title, scope, and AC from `prd.json`
- `[autoresearch]` block from `CLAUDE.md`
- `ac_test_file` path (resolved above)
- Instruction: *"Write design.md and AC test stubs. Do not implement business logic yet."*

Sub-agent must produce:
1. `.autoresearch/tasks/<USR-ID>/iterations/<NNN>/design.md`
2. `<ac_test_file>` — one failing test per AC

Verify both files exist. Retry once on failure, then stop with error.

Update `state.json`: `current_phase → PHASE_3_CODE`

---

### PHASE 3 — Implementation

Spawn sub-agent with **autoresearch-coder** skill. Provide:
- `design.md` from current iteration
- Task title, scope, AC
- `ac_test_file` path
- Instruction: *"Implement. The AC tests already exist — make them pass."*

After sub-agent commits, capture: `git log -1 --format="%H"`

Update `state.json`: `current_phase → PHASE_4_GATES`, `last_commit → <hash>`

---

### PHASE 4 — Quality Gates

Run global gates (if any) first, then scope-specific gates. For each gate, `cd` to `scope_config.root` before executing.

Save results to `.autoresearch/tasks/<USR-ID>/iterations/<NNN>/gate_results.json`.

**AC gate**: run only `<ac_test_file>` — all tests must pass (100%).

**Guard**: run `scope_config.guard.command` from `scope_config.root`. If guard FAILS:
- `git revert HEAD --no-edit`
- Reset state to `PHASE_1_SELECT`
- Log `rollback` to TSV
- Print: `[autoresearch] Guard failed — reverted. Restarting task.`

**Trend tracking**: compare failure count to previous iteration's `gate_results.json`. Log `improving` / `regression`.

If all gates pass → PHASE 5.

If gates fail (guard passed) and `fixer_attempts < 3`:
- Set `current_phase → PHASE_4_FIX`, increment `fixer_attempts`
- Spawn **autoresearch-fix** with full gate output (no truncation)
- After fix commit, re-run PHASE 4

If `fixer_attempts >= 3`: revert, reset to `PHASE_2_PLAN`, increment iteration.

---

### PHASE 5 — Code Review

**Get max review loops from environment:**
```
max_review_loops = ${AUTORESEARCH_MAX_REVIEW_LOOPS:-10}
```

**Initialize review loop counter if not exists:**
```
If state.json does not have "review_loop_count":
  Add "review_loop_count": 0
  Add "last_review_score": null
```

**Spawn sub-agent with** **autoresearch-reviewer** skill. Provide:
- `git diff HEAD~1`
- `design.md` from current iteration
- AC list and scope from `prd.json`

Sub-agent writes `.autoresearch/tasks/<USR-ID>/iterations/<NNN>/review.md`.

**Extract score from review.md:**
- Read the `## 综合评分: X/100` line
- Extract the numeric score

**If score ≥ 80 (PASS):**
- Print: `[autoresearch] Review score: {score}/100 - PASS`
- Update `state.json`: `current_phase → PHASE_6_MEMORY`, `last_review_score → {score}`
- Continue to PHASE 6

**If score < 80 (FAIL):**
- Increment `review_loop_count` in state.json
- Print: `[autoresearch] Review score: {score}/100 - FAIL. Review loop {N}/{max_review_loops}`

- If `review_loop_count < max_review_loops`:
  - Read the full `review.md` (especially "Issues Found" section)
  - Spawn **autoresearch-coder** with instruction:
    ```
    Fix review issues. Score was {score}/100 (failed). Issues:
    - "Issues Found" list with file:line references
    - Architecture invariant violations
    - AC matrix with ❌ items
    - Any quality gate rule violations
    ```
  - After fix commit: **re-run PHASE 4** (quality gates)
  - Then **re-run PHASE 5** (review) → re-evaluate score

- If `review_loop_count ≥ max_review_loops`:
  - Print: `[autoresearch] Review score: {score}/100 - FAIL after {max_review_loops} review loops. Marking as blocker, requires human intervention.`
  - Update `state.json`: `current_phase → PHASE_5_BLOCKED`
  - Mark USR as "blocked" in prd.json
  - Report to user for manual resolution

---

### PHASE 6 — Memory & Experience Sedimentation

Extract lessons: what failed and why, patterns that worked, scope-specific pitfalls.

**Per-section rules** (lessons are written into scope sub-sections, not a flat list):
1. Identify the target scope section for the lesson (e.g. `Known Pitfalls / backend`).
2. Append the new lesson at the **top** of the section (newest first).
3. Count tokens in that section:
   - If ≤ 300 tokens for that scope subsection → done.
   - If > 300 tokens for that scope subsection → trigger section compression.

**Section compression flow**:
1. Read the full scope section content before compression (e.g. `Known Pitfalls / backend`).
2. Identify the oldest lesson(s) at the bottom.
3. Write to `.autoresearch/lessons/archive.md`:
   ```markdown
   ## [<timestamp>] Archival from `<section-name>/<scope>`

   **Reason**: Section exceeded 300 token limit after adding new lesson

   **Before compression**:
   <full section content before change>

   **After compression**:
   <full section content after change>

   **Removed/condensed**:
   - <old lesson 1>
   - <old lesson 2>
   ```
4. Spawn compression sub-agent with instruction:
   *"Condense this scope section to under 300 tokens. Preserve all distinct actionable insights. Deduplicate, do not truncate."*
5. Replace the scope section in `CLAUDE.md` with the compressed result.

Update `<!-- autoresearch:start -->` block in `CLAUDE.md` (and `AGENT.md`):
- Add non-duplicate items under the appropriate **scope sub-section**.
- Maintain per-scope sections; never write everything into a flat list.
- If a scope subsection exceeds 300 tokens, trigger section compression (see flow above).

Append to `.autoresearch/tasks/<USR-ID>/progress.md` (human log — never pass to agents):
```
[<timestamp>] Iteration N [<scope>]: <summary>
```

Append row to `.autoresearch/logs/<USR-ID>.tsv` and `global.tsv`:
```
<iter>  <commit>  <USR-ID>  <scope>  <gates_passed>/<total>  <review_status>  <desc>
```

Ensure `.autoresearch/lessons/archive.md` exists. If not, create it with the header:
```markdown
# Archived Lessons
> Auto-generated by /autoresearch when scope sections exceed 300 tokens.
```

Update `state.json`: `current_phase → PHASE_7_SHIP`

---

### PHASE 7 — Ship (feature-level)

First, mark the current USR as `"done": true` in `prd.json` and update its `status` to `"done"`.

**Check if the entire feature is ready to ship:**

```
feature_id = current_task.feature_id
siblings = all entries in prd.json where feature_id == current feature_id

If any sibling has done: false:
  Print: "[autoresearch] USR-<X> [<scope>] done. Waiting for sibling tasks before creating PR:"
  For each pending sibling: print "  - <USR-ID> [<scope>]: <title> (status: pending/in_progress)"
  Reset state.json to PHASE_1_SELECT and continue the loop.

If all siblings are done:
  Proceed to spawn autoresearch-ship.
```

Spawn sub-agent with **autoresearch-ship** skill. Provide:
- `feature_id` and derived feature title
- `branch_name` from `state.json`
- All sibling USR titles, scopes, and AC lists
- `gate_results.json` summaries for all USRs
- `review.md` files for all USRs

After PR URL confirmed:
- Reset `state.json` to `PHASE_1_SELECT`

Print: `[autoresearch] Feature <feature_id> complete. PR: <url>`

---

### Loop End

```
[autoresearch] All tasks complete.
  feat/user-auth:  USR-001 [backend] + USR-002 [frontend] — PR #23
  feat/usr-003:    USR-003 [backend] — PR #24
CLAUDE.md updated. Review PRs to merge.
```

---

## Hard Rules

- You are orchestrator only. **Never write or edit source code directly.**
- **Never** run `git merge`, `gh pr merge`, or any merge command.
- **Never** modify `.autoresearch/ac-tests/` directly.
- Always read `state.json` before deciding which phase to enter.
- Gate output must capture full tail (50 lines per gate), never truncated further.
- `progress.md` is for humans only. **Never pass it as context to any sub-agent.**
- If a scope referenced in a USR does not exist in `config.yaml`, stop and tell the user to add it via `/autoresearch:plan`.
