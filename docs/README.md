# AutoResearch Enterprise

企业级 AI 自动化研发流水线。吸收三个主流框架的优点，解决其全部核心缺陷。

## 核心设计原则

| 原问题 | 解决方案 |
|--------|---------|
| Memory 截断导致失忆 | 经验写入 CLAUDE.md（自动加载，智能合并） |
| AI 互评分数虚高 | 全部改为 shell exit code，无主观打分 |
| Auto Merge 绕过人工审查 | ship skill 只创建 PR，硬性禁止 merge |
| 质量盲区（假测试） | 变异测试验证测试真实性 |
| 零可观测性 | TSV 日志 + 每轮 design/review/gate 产物 |
| 缺乏任务层级 | :plan 向导负责粒度拆分，AC 驱动验收 |

---

## 快速上手（3 步）

### 第 1 步：安装 Skills

将 `skills/` 目录下的文件安装到你的 agent 运行时：

**Claude Code：**
```bash
# 复制到 Claude Code 全局 skills 目录
cp skills/autoresearch*.md ~/.claude/plugins/local/skills/
```

**OpenCode / Codex：**
参考各平台的 skills 安装文档，将 `skills/` 下的 `.md` 文件放入对应的 skills 目录。

### 第 2 步：初始化项目

在你的项目根目录运行：
```
/autoresearch:plan
```

向导会引导你：
- 输入需求（GitHub Issue URL 或直接描述）
- 拆分任务粒度（如果需要）
- 定义 Acceptance Criteria
- 生成 `.autoresearch/config.yaml` 和 `.autoresearch/prd.json`
- 初始化 `CLAUDE.md`

### 第 3 步：启动循环

```
/autoresearch
```

系统自动运行：设计 → 编码 → 质量门禁 → Review → 经验沉淀 → 创建 PR。

你只需要在 PR 创建后做最终的人工 Code Review 和合并。

---

## 目录结构

```
你的项目/
├── CLAUDE.md                  ← autoresearch 自动维护的经验库
├── AGENT.md                   ← OpenCode/Codex 版本（内容相同）
├── tests/acceptance/          ← AC 测试（由 :plan 生成，:coder 实现）
└── .autoresearch/
    ├── config.yaml            ← 质量门禁配置
    ├── prd.json               ← 任务清单
    └── tasks/
        └── USR-001/
            ├── state.json     ← 运行时状态（支持 crash recovery）
            ├── progress.md    ← 人类可读迭代日志
            └── iterations/001/
                ├── design.md
                ├── review.md
                └── gate_results.json
```

---

## Skills 说明

| Skill | 用途 |
|---|---|
| `/autoresearch:plan` | 初始化项目，添加任务，定义 AC |
| `/autoresearch` | 主循环，自动运行直到所有任务完成 |
| `/autoresearch:coder` | 子 Agent：写 AC 测试 + 实现代码（由主 skill 调用）|
| `/autoresearch:reviewer` | 子 Agent：独立 review（由主 skill 调用）|
| `/autoresearch:fix` | 子 Agent：修复门禁错误（由主 skill 调用）|
| `/autoresearch:ship` | 子 Agent：创建 PR（由主 skill 调用）|

用户直接使用的只有 `:plan` 和主 skill。其余均由主 skill 自动 spawn。

---

## 语言支持

通过 `config.yaml` 的 `quality_gates` 字段替换命令即可支持任意语言。

提供的模板：
- `templates/config-python.yaml` — ruff / mypy / pytest / bandit / mutmut
- `templates/config-typescript.yaml` — eslint / tsc / jest / npm audit / stryker
- `templates/config-go.yaml` — golangci-lint / go test / gosec

---

## 验收机制（四层防线）

一个 feature 算"完成"需要：
1. **AC 测试全部 PASS** — 每条 AC 对应一个测试，100% 通过
2. **变异测试通过** — 验证测试本身不是造假
3. **Reviewer AC 验收矩阵全 ✅** — 独立 Agent 逐条确认
4. **人工 PR Review** — 最终防线，不可绕过

---

## Crash Recovery

任何时刻中断后，重新运行 `/autoresearch` 即可从中断处恢复。系统读取 `.autoresearch/tasks/<USR-ID>/state.json` 的 `current_phase` 字段决定从哪里续跑。

---

## 常见问题

详见 `docs/troubleshooting.md`。
