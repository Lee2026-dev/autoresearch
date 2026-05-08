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

### STEP 3 — Assess scope, granularity, and dependencies

**Scope inference (do this first):** Before asking the user, scan the requirement text for scope signals:

- **Backend signals**: API, endpoint, route, database, schema, migration, model, service, ORM, SQL, query, JWT, auth logic, 接口, 数据库, 模型, 路由, 服务
- **Frontend signals**: UI, button, form, page, component, style, CSS, modal, view, React, Vue, 页面, 组件, 表单, 按钮, 样式

Classify the inferred scope:
- Hits backend signals only → inferred `backend`
- Hits frontend signals only → inferred `frontend`
- Hits both → inferred `both`
- Hits neither → `ambiguous`

**If confident (non-ambiguous inference):** Present the inference and ask for confirmation instead of an open choice:

> "Based on your description, this looks like a **backend-only** task (mentions API/database). Confirm?
> a) Yes, backend only
> b) No — it's frontend only
> c) No — it involves both backend and frontend"

**If ambiguous:** Fall back to the open three-option question:

```
Q: Does this task involve:
a) Backend only (API, data model, business logic)
b) Frontend only (UI component, form, page)
c) Both backend and frontend
```

**Determine which part(s) of the system are involved:**

**If backend only → one USR with `"scope": "backend"`.**

**If frontend only → one USR with `"scope": "frontend"`.**

**If both frontend AND backend:**

Assign a shared `feature_id` (short lowercase slug of the feature name, e.g., `"user-auth"`). Split into two USRs. Present the proposed split:

> "This feature needs both backend and frontend work. I'll create two tasks under feature `user-auth`:
> - USR-00X (backend): [backend part — API, data model, business logic]
> - USR-00Y (frontend): [frontend part — UI component, form, page]
> Both will share branch `feat/user-auth` and result in one PR.
> Does this split look right?"

**Wait for confirmation before writing anything.**

If the user adjusts the split, revise and re-confirm.

**Standalone task**: `feature_id` defaults to the USR's own ID (e.g., `"USR-003"`), giving it its own branch and PR.

**Granularity check** — each USR must satisfy ALL of these:
- ≤ 5 Acceptance Criteria
- Estimated file changes ≤ 5 files
- Independently deliverable (no hard dependency on an incomplete USR from a different feature)
- One coherent PR (together with its feature siblings)

If a single-scope task is still too large, decompose further (same confirmation flow).

**Dependencies**: For each USR, ask if it logically depends on another USR being merged first:

```
Q: Does this task depend on another task being completed first?
a) No dependencies
b) Depends on USR-00X (specify)
c) Multiple dependencies (specify)
```

Set `"depends_on"` accordingly.

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

If config already exists: scan the scopes used by USRs being created in this session. For any scope that is **not defined** in the existing `config.yaml`, immediately alert the user before writing `prd.json`:

> "The tasks you've described require a `frontend` scope, but `.autoresearch/config.yaml` only defines `backend`. Should I add a `frontend` scope block now? (Otherwise the main loop will stop with an error when it reaches that task.)"
>
> a) Yes, add the missing scope now
> b) I'll add it manually later

If the user says yes, append the appropriate scope block from the matching template. Do not overwrite existing entries.

### STEP 6 — Write prd.json and prd.md

Assign `USR-{N}` IDs incrementing from the highest existing (or start at USR-001). Write to `.autoresearch/prd.json`:

```json
{
  "id": "USR-001",
  "title": "<imperative title>",
  "scope": "backend",
  "feature_id": "<feature_id>",
  "done": false,
  "status": "pending",
  "source": "local",
  "priority": 1,
  "depends_on": [],
  "acceptance_criteria": ["<AC 1>", "<AC 2>"]
}
```

- `feature_id`: shared slug for split features (e.g., `"user-auth"`); equals the USR's own ID for standalone tasks
- `depends_on`: list of USR IDs that must be `done: true` before this task starts
- `status`: always `"pending"` on creation

For a frontend+backend split, write two entries with sequential IDs, same `feature_id`, and matching `scope` values.

**Generate human-readable PRD document** for each USR at `.autoresearch/tasks/<USR-ID>/prd.md`:

```markdown
# PRD: <USR-ID> — <title>

## User Story
As a <role>, I want to <action> so that <benefit>.

## Acceptance Criteria
1. <AC 1>
2. <AC 2>
3. <AC 3>

## Scope: <scope>
## Feature: <feature_id>
## Source: <source>
## Dependencies: <depends_on list, or "none">

## Non-Goals
- <things explicitly out of scope for this USR>

## Technical Notes
<Brief initial thoughts on implementation approach — Planner will refine in design.md>
```

### STEP 7 — Initialize CLAUDE.md / AGENT.md

If `CLAUDE.md` does not exist at project root, copy `templates/CLAUDE.md`. If it exists but lacks the `<!-- autoresearch:start -->` block, append it. Mirror content to `AGENT.md`.

**Codebase scan (first-time setup only)** — if the `### Project Architecture` section inside the `[autoresearch]` block is still the placeholder text:

1. Read: project root directory listing, `README.md` (if present), main source directories (e.g., `src/`, `backend/`, `frontend/`)
2. Write a 5–10 line **Project Architecture** summary into CLAUDE.md describing: main modules, key entry points, and directory structure.
3. Infer 3–5 **Architecture Invariants** from the directory structure and README (e.g., if `controllers/` + `services/` + `models/` are present, write corresponding layering rules).

Present the inferred invariants to the user:
```
I've inferred these Architecture Invariants from your codebase:
1. controllers/ must not contain business logic (only call services/)
2. models/ must not import from services/ or controllers/
3. all DB access must go through repositories/

Edit .autoresearch/tasks/ ... or update CLAUDE.md directly to adjust these.
```

These are written to CLAUDE.md's `### Architecture Invariants` section. The user can edit freely — agents will respect whatever is written there.

### STEP 8 — Create task directories

For each new USR:
```bash
mkdir -p .autoresearch/tasks/<USR-ID>
mkdir -p .autoresearch/ac-tests
mkdir -p .autoresearch/logs
mkdir -p .autoresearch/lessons
```

Create the lessons archive file:
```bash
echo "# Archived Lessons\n\n> Auto-generated by /autoresearch when scope sections exceed 300 tokens.\n" > .autoresearch/lessons/archive.md
```

Write initial `state.json`:
```json
{
  "active_task_id": "<USR-ID>",
  "feature_id": "<feature_id>",
  "branch_name": "feat/<feature_id>",
  "current_phase": "PHASE_1_SELECT",
  "iteration": 0,
  "fixer_attempts": 0,
  "review_loop_count": 0,
  "last_review_score": null,
  "last_commit": null,
  "started_at": null
}
```

### STEP 9 — Print summary

```
✅ Setup complete

Tasks added to prd.json:
  USR-001 [backend, feat/user-auth]: <title> (3 AC)
  USR-002 [frontend, feat/user-auth]: <title> (2 AC)  ← depends on USR-001
  USR-003 [backend, feat/usr-003]:   <title> (2 AC)   ← standalone

Config:   .autoresearch/config.yaml
AC tests: .autoresearch/ac-tests/
PRD docs: .autoresearch/tasks/<USR-ID>/prd.md

USR-001 and USR-002 share branch feat/user-auth → one combined PR.
USR-003 gets its own branch feat/usr-003 → separate PR.

Next: run /autoresearch to start working on USR-001.
```

---

## Rules

- Never write to `prd.json` before the user confirms scope, AC, and any decomposition.
- Never overwrite existing `config.yaml` without explicit user instruction.
- IDs must be unique — read existing `prd.json` before assigning.
- If user cancels at any point, do not write partial state.
- A frontend+backend task MUST be split — never assign `scope: both` or leave scope blank.
