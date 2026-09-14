# Agent Task Draft: Event Replay Reconciliation

> This is a design draft only. It is not yet the final `tasks/event-replay-reconciliation/instruction.md`.

## Objective

在容器中实现一个命令行事件日志恢复器，从 `/app/input/events.jsonl` 和 `/app/input/schema.json` 重建订单 aggregate 的可信状态，并生成 `/app/recovered.json`、`/app/audit.json` 和 `/app/README.md`。

## Required behavior to specify before implementation

- `event_id` 是全局身份；完全相同的重复只应用一次并记录 `duplicate`。
- 相同 `event_id` 的规范化内容不同属于篡改冲突，必须隔离并记录 `tampered_duplicate`。
- 因果前置必须已经成功应用；无法证明前置时记录 `missing_predecessor`，不能猜测。
- 无因果关系的并列事件使用公开的稳定排序规则。
- 未知 schema、未知事件类型、非法状态转换和金额/货币冲突必须进入 quarantine。
- 输出必须对输入行顺序、重复注入和重复执行保持确定性。

## Design warning

最终 instruction 必须写出所有会影响 verifier 判断的规则，包括状态转换、字段类型、金额单位、排序、冲突传播和错误代码。不能在 verifier 中悄悄加入 instruction 没有说明的业务规则，否则会违反 TB3 的 well-specified 要求。

## Agent-facing constraints

- 使用绝对路径；
- 不读取 `/tests` 或 `/solution`；
- 不修改输入和 verifier 文件；
- 不运行时下载依赖；
- 以 task.toml 中的整数 timeout 生成 canonical suffix。
