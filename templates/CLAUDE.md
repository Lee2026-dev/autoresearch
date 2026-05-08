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

## [autoresearch] Scoring Criteria
<!-- Reviewer 综合评分依据。80 分及格。 -->

### Weights
- AC 验收: 50%
- 代码质量: 20%
- 架构约束: 20%
- 设计一致性: 10%

### Passing Threshold
- 及格线: 80/100
- < 80: FAIL → 打回 Coder 修改 → 重新 Review

### Max Review Loops
- 默认: 10 次
- 可通过环境变量 `AUTORESEARCH_MAX_REVIEW_LOOPS` 覆盖

### Rules per Dimension

#### AC 验收 (50 分)
- 通过率 = (通过的 AC 数 / 总 AC 数) × 50

#### 代码质量 (20 分)
- 函数 ≤ 40 行: 10 分
- 循环复杂度 ≤ 15: 5 分
- 无明显重复代码: 5 分

#### 架构约束 (20 分)
- Core Invariants 遵守: 每条 3 分，最多 10 分
- Quality Gate Rules 遵守: 每条 2 分，最多 10 分

#### 设计一致性 (10 分)
- 实现符合 design.md: 5 分
- 无意外变更: 5 分

## [autoresearch] Project Architecture
<!-- 由 /autoresearch:plan 初始化时扫描生成。始终加载。 -->
<Populated by /autoresearch:plan on first run via codebase scan.>
<!-- autoresearch:end -->
