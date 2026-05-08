# Troubleshooting

## 流程卡住 / 无响应

**症状**：`/autoresearch` 运行后长时间没有输出。

**处理**：中断后检查 `state.json` 的 `current_phase`，再次运行 `/autoresearch` 即可从断点恢复。如果 `state.json` 损坏，将 `current_phase` 手动改回 `PHASE_1_SELECT` 并将 `iteration` 减 1。

---

## Guard 反复触发回滚

**症状**：每次迭代都因为 Guard 失败被 revert。

**原因**：Coder 引入了破坏基础测试的改动，或者 Guard 命令本身不稳定（有随机失败）。

**处理**：
1. 手动运行 Guard 命令确认是否稳定：`pytest -x -q`
2. 查看 `gate_results.json` 中 `guard` 的 `output_tail` 找到具体失败的测试
3. 如果是 Guard 命令本身有 flaky test，在 `config.yaml` 中改用更稳定的命令
4. 如果是 Coder 的改动破坏了 Guard，在 CLAUDE.md 的 Known Pitfalls 中注明，下次 Coder 会优先规避

---

## Fixer 3 次后仍失败

**症状**：门禁失败，Fixer 调用了 3 次，系统 revert 并重新设计。

**原因**：设计本身有问题，或门禁规则过于严格（如 mutation 测试在遗留代码上难以通过）。

**处理**：
1. 检查 `iterations/NNN/design.md`，看设计是否有根本性问题
2. 如果是可选门禁（`optional: true`）频繁阻断，考虑先注释掉，待代码基础稳定后再启用
3. 如果是 `mutmut` 导致，可先将其设为 `optional: true`

---

## AC 测试全通过但功能有问题

**症状**：所有门禁通过，Reviewer 也 APPROVED，但实际运行 feature 有 bug。

**原因**：AC 定义不够精确，没有覆盖到出问题的场景。

**处理**：
1. 在 `prd.json` 中为该 USR 补充缺失的 AC 条目
2. 将 `done` 改回 `false`
3. 重新运行 `/autoresearch`，Planner 会为新 AC 补写测试

---

## CLAUDE.md [autoresearch] 区块越来越大

**症状**：CLAUDE.md 文件很大，影响 context 效率。

**原因**：压缩子 Agent 没有被触发，或压缩效果不够好。

**处理**：手动运行 `/autoresearch:plan` 后选择"压缩经验"，或直接编辑 CLAUDE.md 的 `[autoresearch]` 区块，删除已不再相关的条目（项目代码改变后旧坑已不存在）。

---

## GitHub Issues 同步后 AC 为空

**症状**：Issue 被同步到 `prd.json` 但 `acceptance_criteria` 是空数组。

**原因**：Issue body 没有可解析为 AC 的内容。

**处理**：运行 `/autoresearch:plan`，选择对应的 USR，向导会引导你手动填写 AC。

---

## PR 创建失败（没有 remote）

**症状**：`/autoresearch:ship` 报错 `git push` 失败。

**原因**：本地仓库没有配置 GitHub remote。

**处理**：
```bash
gh repo create <name> --private --source=. --remote=origin
```
然后重新运行 `/autoresearch`，系统会从 `PHASE_7_SHIP` 恢复。
