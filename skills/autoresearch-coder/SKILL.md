---
name: autoresearch-coder
description: Use when the autoresearch orchestrator needs to write AC test stubs, implement business logic for a USR task, or apply a targeted fix after a quality gate failure.
---

# autoresearch-coder

Focused implementation agent. Receive a precise specification, produce correct code. Do not explore unrelated areas or refactor beyond what the task requires.

CLAUDE.md is auto-loaded — read it for project conventions, known pitfalls, and verified patterns before touching any file.

---

## Sub-task A: Write AC Test Stubs

Triggered by instruction: *"Write design.md and AC test stubs."*

1. Read the task title, AC list, and project architecture summary provided.
2. Write `design.md` to `.autoresearch/tasks/<USR-ID>/iterations/<NNN>/design.md`:
   - Which files will change and why
   - Data flow for this feature
   - Key design decisions
3. Write the AC test file at the path provided by the orchestrator (`<ac_test_file>`, inside `.autoresearch/ac-tests/`) — one test function per AC:
   - Name clearly describes the AC (e.g., `test_duplicate_email_returns_409`)
   - Contains a real assertion — never `assert True` or `pass`
   - Imports follow existing test suite conventions
   - **All tests fail at this point** — this is correct and expected
4. Do not write any business logic.
5. Commit:
   ```
   git add <design.md> <test file>
   git commit -m "test(<USR-ID>): add AC test stubs — <brief description>"
   ```

---

## Sub-task B: Implement Business Logic

Triggered by instruction: *"Implement. The AC tests already exist — make them pass."*

1. Read `design.md`. Understand exactly which files change and why.
2. Read `<ac_test_dir>/test_<USR-ID>.<ext>`. These are your acceptance criteria — make all of them pass.
3. Implement the **minimum code** to pass every AC test. Apply YAGNI.
4. Follow conventions in CLAUDE.md and surrounding code.

**Constraints:**
- Do NOT modify AC test files to make tests pass. The tests define correctness.
- Do not delete or weaken existing tests outside `<ac_test_dir>`.
- Respect complexity limits (cyclomatic complexity ≤ 15 per function).
- Do not add `# noqa`, `# type: ignore`, or equivalent suppression comments.

5. Verify AC tests pass locally before committing.
6. Stage precisely — do not use `git add -A` or `git add .`.
7. Commit:
   ```
   git commit -m "feat(<USR-ID>): <imperative description>"
   ```

---

## Sub-task C: Apply Fix

Triggered by instruction: *"Fix gate failure."*

1. Read the full gate error output provided.
2. Identify root cause and classify:
   - **Style/format** → run auto-fix command (`ruff format`, `eslint --fix`, etc.)
   - **Type error** → add/correct annotations; never use `# type: ignore`
   - **Test failure** → fix the implementation, not the test
   - **Security** → fix the vulnerability; never use `# nosec` without comment
   - **Complexity** → extract into smaller named functions
3. Apply minimal fix. Do not refactor unrelated code.
4. Do NOT re-run the gate suite — the orchestrator handles re-verification.
5. Commit:
   ```
   git commit -m "fix(<USR-ID>): <what was fixed> (gate: <name>)"
   ```

If you cannot determine the fix, write `fix-blocker.md` in the iteration directory describing what you tried, then stop.

---

## Hard Rules

- Never modify `.autoresearch/ac-tests/test_<USR-ID>.*` except in Sub-task A.
- Never use `git add -A` or `git add .`.
- Never add suppression annotations to pass gates.
- One commit per sub-task. Do not squash or amend.
- If the design is impossible or contradictory, write `design-blocker.md` and stop. Do not guess.
