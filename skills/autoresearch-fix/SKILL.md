---
name: autoresearch-fix
description: Use when the autoresearch orchestrator needs to repair a specific quality gate failure — lint, typecheck, test, security, or complexity violation.
---

# autoresearch-fix

Precision repair agent. Fix exactly the gate failure provided — nothing more.

CLAUDE.md is auto-loaded — check Known Pitfalls before diagnosing.

---

## Context Received

- Full output of the failing gate(s) — not truncated
- List of files changed in the current commit (`git diff HEAD~1 --name-only`)
- Gate name and command that failed

---

## Procedure

### STEP 1 — Diagnose

Read the full gate output. Identify the exact error, file, and line number. Check Known Pitfalls in CLAUDE.md — if a matching pitfall has a documented solution, apply it first.

### STEP 2 — Classify and Fix

| Gate type | Fix approach |
|---|---|
| **Format / lint** | Run auto-fix if available (`ruff format .`, `eslint --fix`). Review changes before committing. |
| **Type error** | Add/correct type annotations. Never use `# type: ignore` or `as Any`. |
| **Test failure** | Fix the implementation, not the test. Never modify `.autoresearch/ac-tests/test_<USR-ID>.*`. |
| **Security** | Fix the specific vulnerability. Never use `# nosec` without an explanatory comment. |
| **Complexity** | Extract the complex function into smaller named functions. |
| **Coverage** | Add tests for uncovered paths in the regular test suite (not `.autoresearch/ac-tests`). |

### STEP 3 — Verify Locally

Run the specific failing gate to confirm it now passes:
```bash
<gate command>
echo "Exit: $?"
```

If it still fails, diagnose again. Do not commit a fix that does not resolve the gate.

### STEP 4 — Commit

Stage only modified files:
```bash
git add <files>
git commit -m "fix(<USR-ID>): <what was fixed> (gate: <gate name>)"
```

Do NOT re-run the full gate suite — the orchestrator handles re-verification.

---

## If Blocked

If you cannot determine the fix after reading the full error output and CLAUDE.md, write `fix-blocker.md` in `.autoresearch/tasks/<USR-ID>/iterations/<NNN>/` with:
- What you tried
- What the error says
- What you need to resolve it

Then stop. Do not guess or make random changes.

---

## Hard Rules

- Fix one gate failure per invocation.
- Never use suppression annotations (`# noqa`, `# type: ignore`, `# nosec`, `eslint-disable`) to pass a gate.
- Never modify AC test files.
- Never run the full gate suite yourself.
