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

## 核心设计

```
用户触发 /autoresearch
        │
        ▼
主 Skill（编排者，不写代码）
  ├─ PHASE 1   选取最高优先级任务（prd.json 或 GitHub Issues）
  ├─ PHASE 2   spawn Planner → design.md + AC 测试骨架（TDD）
  ├─- PHASE 3   spawn Coder   → 实现代码 → git commit
  ├─ PHASE 4   质量门禁（纯 shell exit code，语言无关）
  │            ├─ 基础门禁（config.yaml 定义）
  │            ├─ AC 专项测试（100% 必须通过）
  │            └─ Guard（失败立即 git revert）
  ├─ PHASE 5   spawn Reviewer → AC 验收矩阵（独立 context）
  ├─ PHASE 6   经验沉淀 → 合并到 CLAUDE.md（≤500 token，不截断）
  └─ PHASE 7   spawn Ship → gh pr create，等待人工 merge
```

**关键设计决策：**

- **经验落到 CLAUDE.md / AGENT.md**：由 agent 运行时自动加载，不作为 system prompt 传入，token 消耗恒定
- **progress.md 仅供人类阅读**：不传给任何 agent
- **AC 驱动验收**：Planner 先写测试骨架（初始全 FAIL），Coder 使其全通过，feature 完成的定义是 AC 测试 100% PASS + Reviewer 逐条确认
- **无 Auto Merge**：ship skill 硬性禁止任何 merge 操作

---

## 快速上手

### 1. 安装 Skills

**Claude Code：**
```bash
cp -r skills/autoresearch* ~/.claude/skills/
```

**OpenCode / Codex：**
参考各平台文档，将 `skills/` 下的目录复制到对应 skills 路径。

### 2. 初始化项目

在你的项目根目录运行：

```
/autoresearch:plan
```

向导会引导你：
- 输入需求（GitHub Issue URL 或直接描述）
- 判断粒度，必要时拆分为多个子任务（等待你确认后写入）
- 逐条定义 Acceptance Criteria
- 生成 `.autoresearch/config.yaml` 和 `.autoresearch/prd.json`
- 初始化 `CLAUDE.md` 的 `[autoresearch]` 经验区块

### 3. 启动循环

```
/autoresearch
```

自动运行直到所有任务完成。你只需要在最后做 PR Review 和 merge。

---

## 任务粒度

一个 `prd.json` 条目（USR）= 一次 autoresearch 运行可交付的最小单元：

| 约束 | 标准 |
|---|---|
| Acceptance Criteria | ≤ 5 条 |
| 预计改动文件 | ≤ 5 个 |
| 可独立交付 | 不依赖未完成的其他 USR |
| PR 粒度 | 一个 USR 对应一个 PR |

GitHub Issue 与 prd.json 的映射：
- 小 Issue → 1:1 映射为一个 USR
- 大 Issue → `/autoresearch:plan` 向导拆分为多个 USR（确认后写入）

---

## 目录结构

```
skills/
  autoresearch/SKILL.md          ← 主编排循环（用户入口）
  autoresearch-plan/SKILL.md     ← 初始化向导（用户入口）
  autoresearch-coder/SKILL.md    ← 编码子 Agent（由主 skill spawn）
  autoresearch-reviewer/SKILL.md ← 审查子 Agent（由主 skill spawn）
  autoresearch-fix/SKILL.md      ← 修复子 Agent（由主 skill spawn）
  autoresearch-ship/SKILL.md     ← PR 创建子 Agent（由主 skill spawn）

templates/
  config-python.yaml             ← Python 质量门禁模板
  config-typescript.yaml         ← TypeScript 质量门禁模板
  config-go.yaml                 ← Go 质量门禁模板
  prd.json                       ← prd.json 示例
  CLAUDE.md                      ← 含 [autoresearch] 区块的 CLAUDE.md 模板

docs/
  config-reference.md            ← config.yaml / prd.json / state.json 字段说明
  troubleshooting.md             ← 常见问题与处理方法
```

**运行时生成（在你的项目里）：**

```
你的项目/
  CLAUDE.md                          ← 经验库（autoresearch 自动维护）
  AGENT.md                           ← OpenCode/Codex 版本
  tests/acceptance/test_USR_001.py   ← Planner 生成的 AC 测试
  .autoresearch/
    config.yaml                      ← 质量门禁配置
    prd.json                         ← 任务清单
    tasks/USR-001/
      state.json                     ← 运行时状态（支持 crash recovery）
      progress.md                    ← 人类可读迭代日志
      iterations/001/
        design.md                    ← Planner 的设计决策
        review.md                    ← Reviewer 的 AC 验收矩阵
        gate_results.json            ← 门禁结果快照
    logs/
      global.tsv                     ← 全局性能追踪
      USR-001.tsv                    ← 单任务日志
```

---

## 质量门禁（语言无关）

门禁命令在 `config.yaml` 里声明，替换命令即可支持任意语言：

```yaml
quality_gates:
  - name: lint      command: "ruff check ."
  - name: typecheck command: "mypy --strict ."
  - name: test      command: "pytest --cov --cov-fail-under=70"
  - name: security  command: "bandit -r ."
  - name: mutation  command: "mutmut run"
    optional: true  # 失败只记录警告，不阻断

guard:
  command: "pytest -x -q"   # 失败立即 git revert
```

| 失败类型 | 行为 |
|---|---|
| AC 测试 FAIL | spawn Fixer（最多 3 次），仍失 → git revert |
| 基础门禁 FAIL（失败数递减） | 允许继续，记录趋势 |
| Guard FAIL | 立即 git revert，无例外 |
| 变异测试 FAIL | 标记警告，不阻断 |

---

## 验收机制（四层防线）

```
层 1  AC 测试全部 PASS（Planner 生成，逐条对应，非覆盖率）
层 2  变异测试通过（验证测试本身不是造假）
层 3  Reviewer AC 验收矩阵全部 ✅（独立 Agent，不共享 Coder context）
层 4  人工 PR Review（最终防线，不可绕过）
```

---

## Crash Recovery

任何时刻中断后，重新运行 `/autoresearch` 即可恢复——系统读取 `state.json` 的 `current_phase` 字段从断点续跑。

---

## 文档

- [config.yaml / prd.json / state.json 字段参考](docs/config-reference.md)
- [故障排查](docs/troubleshooting.md)
