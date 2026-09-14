# `feature_9_15_event_replay_reconciliation` Task Design

## 1. Assignment understanding

本分支的目标是为 Terminal-Bench 设计一个全新的、可独立运行的 benchmark task，而不是开发一个普通应用。最终任务必须同时满足：

- 对真实工程或专业工作有价值；
- instruction 能完整定义成功条件；
- verifier 能确定性地区分正确和错误结果；
- oracle 能在受限时间内得到 `1.0`；
- nop 必须得到低于 `1.0`；
- Docker、静态检查、implementation rubric、oracle、nop 全部通过；
- 按当前 CI 默认配置运行标准 agent trials，并证明失败来自模型能力而不是基础设施；
- 运行 adversarial trials 后 reward 必须为 `0.0`，且不能通过修改测试、伪造 artifact 或读取隐藏文件绕过 verifier。

这意味着“任务实现完成”与“任务满足面试验收条件”是两个不同阶段。只有真实运行结果齐全后，才能声称最终完成。

## 2. Repository and branch

开发仓库是：`D:\workhome\bench\terminal-bench`。

基线是仓库当前 `main`，本分支为：

```text
feature_9_15_event_replay_reconciliation
```

不使用外层 `D:\workhome\bench` 中的旧文件作为实现来源；外层内容只能作为之前的思考记录，不能混入本分支。

## 3. Proposed task

### Working title

`event-replay-reconciliation`

### Real-world scenario

在消息系统迁移、局部丢数或多服务故障后，数据工程师需要从各服务导出的事件日志恢复订单聚合状态，并回答哪些状态可信、哪些事件冲突、哪些订单必须人工介入。

### Agent objective

Agent 在容器内实现一个命令行恢复器：读取本地 JSONL 事件和公开 schema，输出最终聚合状态以及可审计的诊断记录。输入包含乱序、重复、缺少前置事件、schema 版本差异和同一事件 ID 的内容冲突。

### Core difficulty

困难不是 JSON 读写，而是多个约束的组合：

1. 用事件 ID 实现幂等去重；
2. 区分完全重复和同 ID 内容篡改；
3. 在部分因果信息下确定可应用顺序；
4. 对未知 schema、非法状态迁移和缺失前置做隔离；
5. 让输入排列、重复注入和重复执行都产生相同结果；
6. 输出足够精确的审计证据，而不是只输出一个看似合理的最终状态。

这些失败模式对应真实数据恢复系统中的完整性风险，且不能靠简单的 last-write-wins 解决。

## 4. Contract boundaries

### Inputs visible to the agent

- `/app/input/events.jsonl`
- `/app/input/schema.json`

输入数据和 schema 会公开基本规则，但隐藏 verifier 会使用未出现在样例中的排列、重复和冲突组合。

### Required artifacts

- `/app/recovered.json`
- `/app/audit.json`
- `/app/README.md`

`recovered.json` 表示每个 aggregate 的最终状态、金额、货币、成功应用的事件 ID 和 quarantine 标志；`audit.json` 表示去重、冲突、缺失前置、非法转换和未知 schema 等诊断。

### Non-goals

- 不要求网络服务、数据库、UI 或通用流处理平台；
- 不将运行时间优化作为主要得分点；
- 不要求 agent 猜测未写入 schema 的业务规则；
- 不把实现过程或使用的编辑器作为评分目标。

## 5. Verification design

Verifier 使用 separate mode，测试镜像拥有真值 fixture 和独立参考语义。agent 镜像不包含 solution 或 tests。

验证分成四层：

1. **Schema and semantics**：检查两个 JSON artifact 的字段、排序、状态转换和审计代码。
2. **Metamorphic invariants**：随机或固定地打乱输入顺序、注入完全重复事件、重复执行，结果必须保持不变。
3. **Integrity cases**：篡改重复、伪造前置、未知 schema 和非法状态转换必须被隔离或报告，不能改变可信状态。
4. **Isolation checks**：测试镜像不把 solution 放入 agent 环境；verifier 只读取声明的 artifacts，并检查输入文件没有被修改。

Verifier 不使用 LLM judge，不在 test.sh 中运行时下载依赖，所有 pytest 和 CTRF 工具在 tests/Dockerfile 中预装。

## 6. Alternatives considered

### Alternative A: event replay and reconciliation — recommended

优点：真实运维/数据工程场景，状态机、因果性、幂等性和审计都可程序化验证；可以构造高质量隐藏 fixture。风险是语义必须写得很明确，否则会变成猜规则。

### Alternative B: binary protocol compatibility implementation

优点：系统方向明确，结果可通过协议样例验证。风险是容易退化为逆向猜格式，困难可能来自样本不足而非有价值的工程能力。

### Alternative C: performance-constrained log indexer

优点：可以测正确性和性能。风险是 agent 可能通过硬编码 fixture 或调参获得结果，且性能阈值受运行环境影响较大。

选择 A，因为它在可验证性、现实价值和困难来源之间最平衡。

## 7. Acceptance gates before implementation is considered complete

设计和代码都不能替代以下证据：

- 当前仓库所有 `scripts/checks/check-*.sh` 通过；
- `harbor check ... -r docs/prompts/task-implementation.toml` 通过；
- Docker environment/verifier image 构建成功；
- oracle reward 为 `1.0`；
- nop reward 小于 `1.0`；
- Claude Code 和 Codex 各完成 3 次标准 trial，六次均为真实 verifier 失败；
- Claude Code 和 Codex 各完成 1 次 adversarial trial，reward 均为 `0.0`；
- `harbor analyze` 对每次完成的 trial 给出失败原因，且没有把基础设施错误算成模型失败；
- 所有命令、版本、reward 和失败分析写入仓库。

## 8. Current design decision

本分支先按上述方向完成任务提案和实施计划。只有文档自审通过后，才进入 `tasks/event-replay-reconciliation/` 的实际实现；实现期间如果 oracle 过于简单、前沿 agent 轻易通过或 cheat 能获得非零 reward，应回到设计阶段修改任务，而不是通过削弱 verifier 或制造无意义复杂度来维持失败率。
