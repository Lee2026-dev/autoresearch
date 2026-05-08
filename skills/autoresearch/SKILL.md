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

## Main Loop

Repeat until all tasks have `"done": true`.

---

### PHASE 1 — Task Selection

Read `prd.json`. Select lowest-`priority` `"done": false` task.

Read `.autoresearch/tasks/<USR-ID>/state.json`. If `current_phase` is not `PHASE_1_SELECT`, a prior run was interrupted — **resume from that phase**.

Update `state.json`:
```json
{
  "active_task_id": "<USR-ID>",
  "current_phase": "PHASE_2_PLAN",
  "iteration": <N+1>,
  "fixer_attempts": 0,
  "last_commit": null,
  "started_at": "<ISO timestamp>"
}
```

Print: `[autoresearch] Starting <USR-ID> [<scope>]: <title> (iteration N)`

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

Spawn sub-agent with **autoresearch-reviewer** skill. Provide:
- `git diff HEAD~1`
- `design.md` from current iteration
- AC list and scope from `prd.json`

Sub-agent writes `.autoresearch/tasks/<USR-ID>/iterations/<NNN>/review.md`.

If any AC is `❌`:
- Spawn **autoresearch-coder** with review notes
- Re-run PHASE 4 after fix
- Maximum 2 review→fix cycles

Update `state.json`: `current_phase → PHASE_6_MEMORY`

---

### PHASE 6 — Memory & Experience Sedimentation

Extract lessons: what failed and why, patterns that worked, scope-specific pitfalls.

Update `<!-- autoresearch:start -->` block in `CLAUDE.md` (and `AGENT.md`):
- Add non-duplicate items under appropriate sub-sections
- If block exceeds 500 tokens: spawn compression sub-agent with instruction *"Condense [autoresearch] lessons to under 400 tokens. Preserve all distinct actionable insights. Deduplicate, do not truncate."*

Append to `.autoresearch/tasks/<USR-ID>/progress.md` (human log — never pass to agents):
```
[<timestamp>] Iteration N [<scope>]: <summary>
```

Append row to `.autoresearch/logs/<USR-ID>.tsv` and `global.tsv`:
```
<iter>  <commit>  <USR-ID>  <scope>  <gates_passed>/<total>  <review_status>  <desc>
```

Update `state.json`: `current_phase → PHASE_7_SHIP`

---

### PHASE 7 — Ship

Spawn sub-agent with **autoresearch-ship** skill. Provide task title, scope, AC, `gate_results.json` summary, `review.md`, USR-ID, and iteration number.

After PR URL confirmed:
- Set `"done": true` in `prd.json`
- Reset `state.json` to `PHASE_1_SELECT`

Print: `[autoresearch] <USR-ID> [<scope>] complete. PR: <url>`

---

### Loop End

```
[autoresearch] All tasks complete.
  USR-001 [backend]:  2 iterations — PR #23
  USR-002 [frontend]: 1 iteration  — PR #24
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
