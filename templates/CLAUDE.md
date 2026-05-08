# Project: <project-name>

<Replace this section with a brief description of the project architecture.>

## Tech Stack

- Language: <language>
- Framework: <framework>
- Test runner: <test runner>

---

<!-- autoresearch:start — managed by /autoresearch, do not edit manually -->
## [autoresearch] Lessons

### Architecture Invariants
<!-- Project-level architectural constraints. Initialized by /autoresearch:plan on first run.
     Humans maintain this list. Agents MAY append but MUST NOT delete entries.
     Reviewer enforces every invariant on each diff — a violation blocks the PR. -->
- (example) controllers/ may only call services/ — no business logic in controllers
- (example) models/ must not import from services/ or controllers/
- (example) all external I/O (DB, HTTP, filesystem) must go through the repository layer
- (example) no circular imports between modules

### Project Architecture
<Populated by /autoresearch:plan on first run via codebase scan.>

### Known Pitfalls
<Populated by /autoresearch as pitfalls are discovered.>

### Quality Standards
<Project-specific quality requirements beyond the base gates.>

### Verified Patterns
<Coding patterns confirmed to work well in this codebase.>
<!-- autoresearch:end -->
