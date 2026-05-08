# AutoResearch

企业级 AI 自动化研发流水线——以 Skills 为入口，多子 Agent 协作，语言无关质量门禁，全程人工可介入。

---

## 为什么要做这个

现有三个主流 autoresearch 框架均存在无法在生产环境落地的缺陷：

| 框架 | 核心问题 |
|---|---|
| smallnest/autoresearch | Memory 截断失忆、AI 互评分数虚高、Auto Merge 绕过人工审查 |
| Ralph | 质量盲区（假测试可通过）、零可观测性、无工作流闭环 |
| uditgoenka/autoresearch | 无任务层级拆分、Memory 依赖原始 git log |

本项目吸收三者优点，解决全部核心缺陷。

---

## 快速上手

### 1. 安装 Skills

**Claude Code：**
```bash
cp -r skills/autoresearch* ~/.claude/skills/
```

**OpenCode / Codex：** 参考各平台文档，将 `skills/` 下各目录复制到对应 skills 路径。

### 2. 初始化项目

在你的项目根目录运行：
```
/autoresearch:plan
```

### 3. 启动循环

```
/autoresearch
```

自动运行直到所有任务完成。你只需要在最后做 PR Review 和 merge。

---

## Workflow

### 一、初始化（只做一次）

```
开发者
  │
  └─ /autoresearch:plan
       │
       ├─ 输入需求（GitHub Issue URL 或直接描述）
       │
       ├─ 向导判断涉及哪个 scope（带选项的结构化问答）
       │    ├─ 纯后端   → 1 个 USR（scope: backend）
       │    ├─ 纯前端   → 1 个 USR（scope: frontend）
       │    └─ 前后端   → 拆分为 2 个 USR，共享 feature_id，等用户确认
       │
       ├─ 询问依赖关系 → depends_on 字段
       │
       ├─ 逐条定义 Acceptance Criteria（用户确认后写入）
       │
       ├─ 生成 .autoresearch/config.yaml（按语言选模板）
       ├─ 生成 .autoresearch/prd.json（含 scope + AC + feature_id + depends_on）
       ├─ 生成 .autoresearch/tasks/<USR-ID>/prd.md（人类可读需求文档）
       └─ 初始化 CLAUDE.md [autoresearch] 区块
            ├─ 扫描项目目录结构，生成 Project Architecture 摘要
            └─ 推断并写入 Architecture Invariants（用户可编辑确认）
```

### 二、日常开发循环

```
开发者运行 /autoresearch
        │
        ▼
PHASE 0  Scope 预检
  检查 config.yaml 中 scope 是否存在
  检查每条 gate command 是否已安装
  缺失 → 明确报错，不静默跳过
        │
        ▼
读取 prd.json → depends_on 检查 → 取最高优先级未阻塞任务（含 scope + feature_id）
  切换到 feat/<feature_id> branch（已存在则 checkout，否则 checkout -b）
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 2  Planner 子 Agent（autoresearch-coder）      │
│  输入：任务 AC + 项目架构（CLAUDE.md 自动加载）       │
│  输出：design.md                                    │
│        模块桩文件（若被测模块不存在，创建 stub）      │
│        .autoresearch/ac-tests/test_USR_001.py       │
│        （AC 测试骨架，初始全 FAIL: NotImplementedError）│
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 3  Coder 子 Agent（autoresearch-coder）        │
│  输入：design.md + AC 列表                          │
│  输出：实现代码 → git commit                        │
│  目标：使 .autoresearch/ac-tests/test_USR_001.py    │
│        全部 PASS                                    │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 4  质量门禁（纯 shell exit code，不依赖 AI）   │
│                                                     │
│  全局门禁（所有 USR 都跑）                           │
│    + scope 专属门禁（从 config.yaml 按 scope 读取） │
│    + AC 测试（100% 必须通过）                       │
│    + Guard（失败 → 立即 git revert → 重来）         │
│                                                     │
│  失败 → spawn Fixer 子 Agent（最多 3 次）           │
│  3 次仍失 → revert → 回到 PHASE 2 重新设计          │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 5  Reviewer 子 Agent（独立 context）          │
│  输入：git diff + design.md（不含 Coder 对话历史）  │
│  输出：                                             │
│    ① AC 验收矩阵（每条 AC 逐一 ✅ / ❌）            │
│    ② 代码结构审查（假测试 / SOLID / 安全）          │
│    ③ Design Conformance（文件范围是否符合设计）     │
│    ④ Architecture Invariants 检查                  │
│       （对比 CLAUDE.md 中的不变量，违反即 ❌）      │
│                                                     │
│  有 ❌ → 回 Coder 修复 → 重跑门禁（最多 2 轮）      │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 6  经验沉淀                                   │
│  提炼 lessons → 合并到 CLAUDE.md [autoresearch]     │
│  （智能去重压缩，≤500 token，不截断）               │
│  追加到 .autoresearch/tasks/USR-001/progress.md     │
│  （人类可读日志，永远不传给 Agent）                  │
└─────────────────────────────────────────────────────┘
        │
        ▼
检查 feature 内所有 USR 是否全部完成
  还有未完成的 sibling USR → 回 PHASE 1 取下一个任务
  全部完成 → PHASE 7
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ PHASE 7  Ship 子 Agent（feature 级别）              │
│  squash feature branch 所有 commits → 1 个干净 commit│
│  gh pr create（含全部 USR AC checklist + 门禁结果） │
│  硬性禁止 auto merge                                │
└─────────────────────────────────────────────────────┘
        │
        ▼
开发者 Review PR → Merge  ← 唯一需要人工介入的环节
        │
        ▼
prd.json 标记 done: true → 取下一个任务 → 循环
```

### 三、Feature Branch 与 PR 模型

一个 **feature** 可由多个 USR 组成（例如前后端拆分）。同一 feature 的所有 USR **共享一个 branch 和一个 PR**：

```
需求："用户注册功能"（前后端都涉及）
         ↓ /autoresearch:plan 拆分
USR-001  scope: backend   feature_id: user-auth   "POST /register 接口"
USR-002  scope: frontend  feature_id: user-auth   "注册表单页面"

         ↓ 执行顺序
USR-001 [backend] 在 feat/user-auth branch 上完成
USR-002 [frontend] 在同一 feat/user-auth branch 上继续
         ↓
所有 USR 完成 → squash → gh pr create → 人工 Review → Merge
```

独立需求的 USR（无前后端拆分）使用独立 branch + 独立 PR：

```
USR-003  scope: backend   feature_id: USR-003   "数据导出功能"
  └─ feat/usr-003 branch → 独立 PR
```

### 四、中断恢复

任何阶段中断后，重新运行 `/autoresearch`——读取 `.autoresearch/tasks/<USR-ID>/state.json` 的 `current_phase` 字段从断点续跑，不会从头开始。

---

## 多语言 / Monorepo 支持

通过 `config.yaml` 的 `scopes` 字段支持多语言，每个 scope 独立声明门禁命令：

```yaml
# .autoresearch/config.yaml

ac_test_dir: ".autoresearch/ac-tests"   # 所有 AC 测试集中存放，不占用项目目录

scopes:
  backend:
    root: "backend/"        # 门禁命令在此目录下执行
    quality_gates:
      - name: lint          command: "ruff check ."
      - name: typecheck     command: "mypy --strict ."
      - name: test          command: "pytest --cov --cov-fail-under=70"
      - name: security      command: "bandit -r . -ll"
    guard:
      command: "pytest -x -q"

  frontend:
    root: "frontend/"
    quality_gates:
      - name: lint          command: "eslint src/ --max-warnings 0"
      - name: typecheck     command: "tsc --noEmit"
      - name: test          command: "jest --coverage --ci"
    guard:
      command: "jest --ci --passWithNoTests"
```

**PHASE 0 会在循环开始前预检每个 scope**：scope 不存在或 gate 命令未安装时立即报错，不会在运行到一半时才失败。

提供的语言模板：
- `templates/config-python.yaml` — ruff / mypy / pytest / bandit
- `templates/config-typescript.yaml` — eslint / tsc / jest / npm audit
- `templates/config-go.yaml` — golangci-lint / go test / gosec

---

## 架构守护

### Architecture Invariants

`/autoresearch:plan` 初始化时扫描项目目录，推断并写入目标项目 `CLAUDE.md` 的 `### Architecture Invariants` 区块：

```markdown
### Architecture Invariants
- controllers/ 只允许调用 services/，不得包含业务逻辑
- models/ 不得 import services/ 或 controllers/
- 所有外部 IO 必须经过 repository 层
- 禁止跨模块循环 import
```

**Reviewer 在每次 review 中逐条核查这些约束**：违反任意一条 → `review.md` 写入 ❌ → 阻断 PR，直到 Coder 修复。

**维护规则**：
- 人工可随时编辑 `CLAUDE.md` 中的不变量
- Agent 只可追加，不得删除
- 不变量越明确，架构腐化被发现得越早

---

## 目录结构

```
skills/
  autoresearch/SKILL.md          ← 主编排循环（用户入口）
  autoresearch-plan/SKILL.md     ← 初始化向导（用户入口）
  autoresearch-coder/SKILL.md    ← 编码子 Agent（设计+实现+修复）
  autoresearch-reviewer/SKILL.md ← 审查子 Agent（AC + 架构不变量）
  autoresearch-fix/SKILL.md      ← 修复子 Agent
  autoresearch-ship/SKILL.md     ← PR 创建子 Agent（feature 级别）

templates/
  config-python.yaml             ← Python scope 配置模板
  config-typescript.yaml         ← TypeScript scope 配置模板
  config-go.yaml                 ← Go scope 配置模板
  prd.json                       ← prd.json 示例（含 feature_id / depends_on）
  CLAUDE.md                      ← 含 [autoresearch] 区块的模板（含 Architecture Invariants）
```

**运行时生成（在你的项目里）：**

```
<project-root>/
  CLAUDE.md                          ← 经验库（[autoresearch] 区块自动维护）
                                        含 Architecture Invariants（人工维护）
  AGENT.md                           ← OpenCode/Codex 同步版本
  .autoresearch/
    config.yaml                      ← 质量门禁 + scope 配置
    prd.json                         ← 任务清单（含 feature_id / depends_on）
    ac-tests/
      test_USR_001.py                ← Planner 生成的 AC 测试（后端）
      test_USR_002.test.ts           ← Planner 生成的 AC 测试（前端）
    tasks/
      USR-001/
        prd.md                       ← 人类可读需求文档（/autoresearch:plan 生成）
        state.json                   ← 运行时状态（含 feature_id / branch_name）
        progress.md                  ← 人类可读迭代日志（不传给 Agent）
        iterations/
          001/
            design.md                ← Planner 的设计决策
            review.md                ← Reviewer 的验收矩阵（含架构不变量检查）
            gate_results.json        ← 门禁结果快照
    logs/
      global.tsv                     ← 全局追踪日志
      USR-001.tsv                    ← 单任务日志
```

---

## prd.json 字段参考

```json
{
  "id": "USR-001",
  "title": "注册接口：邮箱+密码",
  "scope": "backend",
  "feature_id": "user-auth",
  "done": false,
  "status": "pending",
  "source": "local",
  "priority": 1,
  "depends_on": [],
  "acceptance_criteria": [
    "POST /register 成功返回 201 和用户 ID",
    "重复邮箱返回 409",
    "密码 < 8 chars 返回 400"
  ]
}
```

| 字段 | 说明 |
|---|---|
| `feature_id` | 同一需求拆分的 USR 共享同一值（如 `"user-auth"`），独立 USR 默认等于自身 ID |
| `depends_on` | 必须先完成的 USR ID 列表，orchestrator 据此跳过被阻塞的任务 |
| `status` | `pending` / `blocked` / `in_progress` / `done` |

---

## 验收机制（五层防线）

```
层 1  AC 测试全部 PASS
      Planner 按 AC 生成测试骨架（模块桩确保 import 可解析，初始失败 NotImplementedError）
      Coder 使其全通过；存放于 .autoresearch/ac-tests/，不侵入项目目录

层 2  变异测试通过（可选）
      验证 AC 测试本身不是造假（mutmut / stryker）

层 3  Reviewer AC 验收矩阵全部 ✅
      独立子 Agent，只看 diff + design.md，不共享 Coder context

层 4  Architecture Invariants 全部 ✅
      Reviewer 对比 CLAUDE.md 中的架构约束，违反即阻断

层 5  人工 PR Review（最终防线，不可绕过）
```

---

## 关键设计决策

| 决策 | 原因 |
|---|---|
| 经验写入 CLAUDE.md，不传 progress.md | Agent 自动加载，token 消耗恒定，不随迭代增长 |
| AC 测试放 `.autoresearch/ac-tests/` | 不占用项目目录，不污染项目测试套件 |
| 门禁全部是 shell 命令 | 客观 exit code，不依赖 AI 判断，无法造假 |
| Reviewer 独立 context | 无法看到 Coder 的思路，保证 review 真正独立 |
| feature_id 而非 USR 级 branch | 前后端拆分的 USR 共享 branch，用户只需 review 一个 PR |
| 模块桩文件（stub） | AC 测试在 PHASE 2 生成时被测模块可能不存在，stub 使 import 可解析 |
| Architecture Invariants 由人维护 | AI 生成初稿，人工确认；约束一旦写定，所有后续 Agent 都受其约束 |
| 禁止 Auto Merge | 保留人类对架构走向的最终把控权 |

---

## 文档

- [config.yaml / prd.json / state.json 字段参考](docs/config-reference.md)
- [故障排查](docs/troubleshooting.md)
