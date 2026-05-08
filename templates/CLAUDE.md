# Project: <project-name>

<Replace this section with a brief description of the project architecture.>

## Tech Stack

- Language: <language>
- Framework: <framework>
- Test runner: <test runner>

---

<!-- autoresearch:start — managed by /autoresearch, do not edit manually -->
## [autoresearch] Core Invariants
<!-- 始终加载。项目级架构约束，Reviewer 每次必查。
     由 /autoresearch:plan 初始化，人工可编辑，Agent 仅可追加。 -->
- (example) controllers/ may only call services/ — no business logic in controllers
- (example) models/ must not import from services/ or controllers/
- (example) all external I/O (DB, HTTP, filesystem) must go through the repository layer
- (example) no circular imports between modules

## [autoresearch] Known Pitfalls
<!-- 按 scope 分区，按需加载。
     UMA 只加载当前 task.scope 对应的 subsection。
     每个 scope 分区独立 300 token 上限，超限后整区压缩。
     格式：[{scope}] < concise lesson, ensure LLM easily understands > -->

### Common
- [common] <lesson>

### Backend
- [backend] <lesson>

### Frontend
- [frontend] <lesson>

## [autoresearch] Verified Patterns
<!-- 按 scope 分区，按需加载。
     UMA 只加载当前 task.scope 对应的 subsection。
     每个 scope 分区独立 300 token 上限，超限后整区压缩。
     格式：[{scope}] < concise lesson, ensure LLM easily understands > -->

### Common
- [common] <lesson>

### Backend
- [backend] <lesson>

### Frontend
- [frontend] <lesson>

## [autoresearch] Quality Standards
<!-- 按 scope 分区，按需加载。
     UMA 只加载当前 task.scope 对应的 subsection。
     每个 scope 分区独立 300 token 上限，超限后整区压缩。
     格式：[{scope}] < concise lesson, ensure LLM easily understands > -->

### Common
- [common] <lesson>

### Backend
- [backend] <lesson>

### Frontend
- [frontend] <lesson>

## [autoresearch] Project Architecture
<!-- 由 /autoresearch:plan 初始化时扫描生成。始终加载。 -->
<Populated by /autoresearch:plan on first run via codebase scan.>
<!-- autoresearch:end -->
