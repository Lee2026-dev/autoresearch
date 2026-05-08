# config.yaml 字段参考

## task_source

```yaml
task_source:
  type: local          # "local" 或 "github"
  file: .autoresearch/prd.json   # type=local 时有效

  # type=github 时使用：
  repo: owner/repo
  label: autoresearch  # 只同步带此 label 的 Issue
  sync_on_start: true  # 每次 /autoresearch 启动时从 GitHub 同步一次
```

## quality_gates

每个 gate 是一条 shell 命令，以 exit code 0/非0 判断通过/失败。

```yaml
quality_gates:
  - name: <gate名称>        # 用于日志显示
    command: "<shell命令>"  # 运行在项目根目录
    optional: false         # true = 失败只记录警告，不阻断流程
```

**执行顺序**：按数组顺序依次执行，遇到第一个非 optional 的失败即停止。

## guard

```yaml
guard:
  command: "<shell命令>"
```

Guard 在每次 git commit 之后单独运行。失败触发立即 `git revert`，不进入 Fixer 流程。Guard 应选择最快速、最稳定的基础测试命令（如 `pytest -x -q`）。

## ac_test_dir

```yaml
ac_test_dir: "tests/acceptance"
```

AC 测试文件的存放目录。Planner 在此目录下创建 `test_<USR-ID>.<ext>` 文件。质量门禁中的 AC gate 只运行此目录下的文件。

---

## prd.json 字段参考

```jsonc
[
  {
    "id": "USR-001",              // 唯一 ID，格式 USR-{N}，由 :plan 自动分配
    "title": "...",               // 简洁的祈使句标题
    "done": false,                // true 表示任务已完成（PR 已创建）
    "source": "local",            // "local" 或 "github:<issue-number>"
    "priority": 1,                // 数字越小优先级越高
    "acceptance_criteria": [      // 必填，≤5 条，每条是可测试的可观测结果
      "...",
      "..."
    ]
  }
]
```

---

## state.json 字段参考

由 `/autoresearch` 自动维护，不要手动修改。

```jsonc
{
  "active_task_id": "USR-001",
  "current_phase": "PHASE_4_GATES",  // 当前执行到的阶段
  "iteration": 2,                    // 当前迭代序号（从 1 开始）
  "fixer_attempts": 1,               // 本迭代已调用 Fixer 的次数
  "last_commit": "a1b2c3d",          // 最后一次 commit 的 hash
  "started_at": "2026-05-07T10:23Z"  // 当前任务开始时间
}
```

**Phase 枚举值：**
- `PHASE_1_SELECT` — 选取任务
- `PHASE_2_PLAN` — 设计 + AC 测试生成
- `PHASE_3_CODE` — 业务代码实现
- `PHASE_4_GATES` — 质量门禁
- `PHASE_4_FIX` — 门禁失败修复中
- `PHASE_5_REVIEW` — 代码审查
- `PHASE_6_MEMORY` — 经验沉淀
- `PHASE_7_SHIP` — 创建 PR
