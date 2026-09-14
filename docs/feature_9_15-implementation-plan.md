# `feature_9_15` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `main` 基线上实现并验证一个符合当前 Terminal-Bench CI 的事件日志恢复任务。

**Architecture:** 任务环境只提供 Python 运行时、公开输入和 schema；agent 生成 `/app` 下的恢复器和三个 artifact；独立 verifier 镜像携带隐藏 fixture、参考语义和性质测试。solution 只用于 oracle，不进入 agent 镜像。

**Tech Stack:** Harbor task format、Docker、Python、pytest、pytest-json-ctrf、JSONL。

## Global Constraints

- 所有路径必须相对于 `D:\workhome\bench\terminal-bench` 的仓库结构，并使用 `tasks/event-replay-reconciliation/` 作为唯一任务目录。
- 从 `main` 创建的 `feature_9_15` 分支中工作。
- 使用仓库当前 task.toml schema 和检查脚本，不照搬旧版本格式。
- verifier 使用 `environment_mode = "separate"`。
- 测试依赖在 `tests/Dockerfile` 构建阶段安装，不在 `tests/test.sh` 运行时联网安装。
- 任务 instruction 必须使用绝对路径并以 task.toml 中的 agent timeout 对应的 canonical suffix 结尾。
- 不把 `solution/` 或 `tests/` 复制进 environment image。

### Task 1: Freeze task proposal

**Files:**
- Read: `CONTRIBUTING.md`
- Read: `docs/prompts/task-proposal.md`
- Read: `docs/prompts/task-implementation.toml`
- Read: `docs/TASK_REVIEW_AUTOMATION.md`
- Read: `docs/REVIEWING.md`
- Create: `docs/feature_9_15-design.md`
- Create: `docs/feature_9_15-agent-task.md`

- [ ] Confirm the task is realistic, outcome-verified, independently checkable, and not merely a trick for current models.
- [ ] Confirm every required artifact and every verifier-visible behavior appears in the agent task draft.
- [ ] Confirm the task has a legitimate reference solution that an expert can implement within the configured timeout.
- [ ] Review the documents for placeholders, contradictory rules, and unspecified conflict precedence.

### Task 2: Initialize the task directory

**Files:**
- Create: `tasks/event-replay-reconciliation/task.toml`
- Create: `tasks/event-replay-reconciliation/README.md`
- Create: `tasks/event-replay-reconciliation/instruction.md`
- Create: `tasks/event-replay-reconciliation/environment/Dockerfile`
- Create: `tasks/event-replay-reconciliation/environment/data/`

- [ ] Prefer `harbor tasks init event-replay-reconciliation --include-canary-strings --metadata-template docs/task-template.toml -p tasks/` when Harbor is available.
- [ ] Fill current repository metadata, category, subcategory, tags, expert estimate, artifact paths, timeouts, and separate verifier settings.
- [ ] Keep README’s four required reviewer sections concise and written as human task rationale.
- [ ] Run `check-canary`, `check-task-fields`, `check-task-package-name`, `check-task-slug`, and `check-instruction-suffix` before adding implementation logic.

### Task 3: Implement the environment and reference solution

**Files:**
- Modify: `tasks/event-replay-reconciliation/environment/Dockerfile`
- Create: `tasks/event-replay-reconciliation/environment/data/events.jsonl`
- Create: `tasks/event-replay-reconciliation/environment/data/schema.json`
- Create: `tasks/event-replay-reconciliation/solution/solve.sh`
- Create: `tasks/event-replay-reconciliation/solution/replay_reference.py`

- [ ] Make the environment image self-contained and copy only public input data.
- [ ] Implement canonical event identity, duplicate handling, causal readiness, schema validation, legal state transitions, quarantine propagation, and stable serialization.
- [ ] Ensure the oracle generates every declared artifact and does not hardcode only one expected output.
- [ ] Run the reference solution in a clean environment and inspect the generated JSON manually.

### Task 4: Implement and harden the separate verifier

**Files:**
- Create: `tasks/event-replay-reconciliation/tests/Dockerfile`
- Create: `tasks/event-replay-reconciliation/tests/test.sh`
- Create: `tasks/event-replay-reconciliation/tests/test_outputs.py`
- Create: `tasks/event-replay-reconciliation/tests/data/`

- [ ] Bake pytest, pytest-json-ctrf, reference fixtures, and verifier-only data into the verifier image.
- [ ] Make test.sh emit Harbor reward files and return the expected status for pass/fail.
- [ ] Test normal replay, duplicate delivery, tampered duplicate, missing predecessor, unknown schema, invalid transition, and amount/currency errors.
- [ ] Test order permutation and duplicate injection invariance.
- [ ] Test stable key ordering and unchanged input files.
- [ ] Review every test against instruction.md so hidden behavior is not based on an undocumented rule.

### Task 5: Run local checks and fix defects

**Files:**
- Modify any task files required by check output.
- Create: `results/feature_9_15-validation.md`

- [ ] Run `for check in scripts/checks/check-*.sh; do bash "$check" tasks/event-replay-reconciliation; done` in a Linux-compatible shell.
- [ ] Run `docker build tasks/event-replay-reconciliation/environment` and build the verifier image.
- [ ] Run `harbor run -p tasks/event-replay-reconciliation --agent oracle` and require reward `1.0`.
- [ ] Run `harbor run -p tasks/event-replay-reconciliation --agent nop` and require reward below `1.0`.
- [ ] Run the implementation rubric review and resolve all critical findings.
- [ ] Record command, commit, environment, output, and any known limitation in the validation document.

### Task 6: Measure difficulty and anti-cheat resistance

**Files:**
- Create: `results/feature_9_15-trials.md`
- Create: `results/feature_9_15-failure-analysis.md`

- [ ] Run three Claude standard trials using the repository’s configured `anthropic/claude-opus-5` and `reasoning_effort=max`.
- [ ] Run three Codex standard trials using the repository’s configured `openai/gpt-5.6-sol` and `reasoning_effort=xhigh`.
- [ ] Treat only completed verifier failures as model failures; rerun infrastructure failures.
- [ ] Run one adversarial trial per agent using the current `/cheat` workflow and require reward `0.0` for each.
- [ ] Run `harbor analyze` for all completed jobs and classify failures by intended difficulty crux.
- [ ] If agents pass reliably, redesign the task difficulty for a real domain reason; do not add arbitrary volume or hidden rules.
- [ ] If adversarial trials get nonzero reward, harden the verifier and repeat the trials.

### Task 7: Final repository review

**Files:**
- Modify: `results/feature_9_15-validation.md`
- Modify: `results/feature_9_15-trials.md`
- Modify: `results/feature_9_15-failure-analysis.md`

- [ ] Confirm all required evidence is present and no infrastructure error is mislabeled as model failure.
- [ ] Run `git diff --check`, inspect `git status`, and verify no unrelated files changed.
- [ ] Commit task implementation and evidence on `feature_9_15`.
- [ ] Push the branch to the user-controlled GitHub fork and provide the repository URL only after the evidence is complete.
