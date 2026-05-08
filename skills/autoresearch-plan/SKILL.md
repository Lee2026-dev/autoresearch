---
name: autoresearch-plan
description: Use when initializing a project for autoresearch for the first time, or when adding new tasks with acceptance criteria to an existing project's prd.json.
---

# autoresearch:plan

Setup wizard: turn a goal or GitHub Issue into properly-structured `prd.json` entries with scope, Acceptance Criteria, and gate configuration.

Run this **before** `/autoresearch`. Do not run the main loop until this step is complete.

---

## Procedure

### STEP 1 — Read existing state

Check whether `.autoresearch/config.yaml` and `.autoresearch/prd.json` already exist. If yes, you are adding tasks. If no, this is fresh setup — also check for language markers: `package.json`, `pyproject.toml` / `setup.py`, `go.mod`.

### STEP 2 — Collect the requirement

Ask: *"What do you want to build or fix? Paste a GitHub Issue URL, describe a feature, or give a bug report."*

If a GitHub Issue URL is given, run `gh issue view <number>` to fetch title and body.

### STEP 3 — Assess scope and granularity

**Determine which part(s) of the system are involved:**

Ask (if not obvious from the requirement):
> "Does this task involve backend only, frontend only, or both?"

**If backend only → one USR with `"scope": "backend"`.**

**If frontend only → one USR with `"scope": "frontend"`.**

**If both frontend AND backend:**

Split into two USRs automatically. Present the proposed split:
> "This feature needs both backend and frontend work. I'll create two tasks:
> - USR-00X (backend): [backend part — API, data model, business logic]
> - USR-00Y (frontend): [frontend part — UI component, form, page]
> Does this split look right?"

**Wait for confirmation before writing anything.**

If the user adjusts the split, revise and re-confirm.

**Granularity check** — each USR must satisfy ALL of these:
- ≤ 5 Acceptance Criteria
- Estimated file changes ≤ 5 files
- Independently deliverable (no hard dependency on an incomplete USR)
- One coherent PR

If a single-scope task is still too large, decompose further (same confirmation flow).

### STEP 4 — Define Acceptance Criteria

For each USR, propose 3–5 AC. Frame each as a **testable, observable outcome**:
- ✅ "POST /register with duplicate email returns HTTP 409"
- ❌ "Handle errors properly"

Show proposed AC per USR and wait for confirmation before proceeding.

### STEP 5 — Generate config.yaml (first-time only)

If `.autoresearch/config.yaml` does not exist:

1. Detect language from project files:
   - `pyproject.toml` / `setup.py` → `config-python.yaml`
   - `package.json` → `config-typescript.yaml`
   - `go.mod` → `config-go.yaml`
   - Mixed (e.g., `pyproject.toml` + `package.json`) → ask which template to start from, or use `config-python.yaml` and offer to add a `frontend` scope block

2. Copy matching template to `.autoresearch/config.yaml`.

3. For monorepos, ask:
   > "I see both backend and frontend code. Should I configure separate `backend` and `frontend` scopes? For now I can enable just the backend scope and comment out the frontend one."

4. Ask if any gate commands need adjustment before writing.

If config already exists, skip — unless the user requests a scope to be added.

### STEP 6 — Write prd.json

Assign `USR-{N}` IDs incrementing from the highest existing (or start at USR-001). Write to `.autoresearch/prd.json`:

```json
{
  "id": "USR-001",
  "title": "<imperative title>",
  "scope": "backend",
  "done": false,
  "source": "local",
  "priority": 1,
  "acceptance_criteria": ["<AC 1>", "<AC 2>"]
}
```

For a frontend+backend split, write two entries with sequential IDs and matching `scope` values.

### STEP 7 — Initialize CLAUDE.md / AGENT.md

If `CLAUDE.md` does not exist at project root, copy `templates/CLAUDE.md`. If it exists but lacks the `<!-- autoresearch:start -->` block, append it. Mirror content to `AGENT.md`.

### STEP 8 — Create task directories

For each new USR:
```bash
mkdir -p .autoresearch/tasks/<USR-ID>
mkdir -p .autoresearch/ac-tests
```

Write initial `state.json`:
```json
{
  "active_task_id": "<USR-ID>",
  "current_phase": "PHASE_1_SELECT",
  "iteration": 0,
  "fixer_attempts": 0,
  "last_commit": null,
  "started_at": null
}
```

### STEP 9 — Print summary

```
✅ Setup complete

Tasks added to prd.json:
  USR-001 [backend]: <title> (3 AC)
  USR-002 [frontend]: <title> (2 AC)

Config:  .autoresearch/config.yaml
AC tests: .autoresearch/ac-tests/

Next: run /autoresearch to start working on USR-001.
```

---

## Rules

- Never write to `prd.json` before the user confirms scope, AC, and any decomposition.
- Never overwrite existing `config.yaml` without explicit user instruction.
- IDs must be unique — read existing `prd.json` before assigning.
- If user cancels at any point, do not write partial state.
- A frontend+backend task MUST be split — never assign `scope: both` or leave scope blank.
