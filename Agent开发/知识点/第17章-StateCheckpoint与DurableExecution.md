# 第17章：State、Checkpoint 与 Durable Execution

## ① 核心知识点

- **State**：当前任务的结构化事实，如步骤状态、输入、工具结果引用、审批状态、预算和版本。
- **Checkpoint**：在安全边界保存可恢复快照；至少在副作用完成后、人工暂停前、阶段切换时写入。
- **Durable Execution**：进程或机器故障后，从持久化历史继续，而不是从头猜测执行。
- **事件与快照**：事件便于审计和重放，快照便于快速恢复；大结果存外部对象存储，仅保存引用和校验和。
- **版本兼容**：状态带 `schema_version`、工作流版本和迁移策略；禁止直接用新代码解释旧状态而不迁移。
- **一致性**：状态更新与幂等记录要原子或可检测；并发恢复使用租约/版本号防止双写。
- **确定性**：工作流编排代码应可重放；随机数、当前时间、网络调用放在可记录的 Activity/Task 中。
- **保留与隐私**：按租户和法规设置 TTL、加密、访问审计和删除流程。

## ② 原理

```text
事件 e1 → 快照 S1 → 事件 e2 → 快照 S2
故障恢复：加载最近快照 + 重放后续事件 → 继续未完成步骤
```

恢复时，对 `RUNNING` 步骤先查询幂等状态；无法确认则标记 `UNKNOWN` 并人工/对账处理。只有确定未执行才允许重试。

## ③ 企业架构

- **运行时**：工作流定义、任务队列、重试和信号；LLM 调用作为可重放的 Activity。
- **状态库**：PostgreSQL 等保存检查点元数据、事件和租约；对象存储保存大文件。
- **恢复服务**：按 `run_id` 加载快照，执行 schema 迁移，恢复队列任务。
- **加密与隔离**：租户密钥、行级权限、敏感字段脱敏；备份同样加密。
- **运维**：检查点延迟、恢复耗时、孤儿运行、历史大小和迁移失败率。

## ④ 面试标准答案

> State 是业务执行事实，Checkpoint 是可恢复快照，Durable Execution 是把每个步骤的结果持久化，使进程崩溃后可继续。设计上要让工作流编排保持确定性，把 API/LLM/随机操作封装为可重试的 Activity；副作用步骤使用幂等键，恢复时先查询未知状态。状态带版本并可迁移，敏感数据按租户加密和 TTL 清理。

## ⑤ 高频追问

**Q：每一行代码都要 checkpoint 吗？** 不需要；在阶段、任务完成和可能暂停的边界保存即可，过密会增加成本。

**Q：快照和事件如何取舍？** 快照加速恢复，事件保留审计；通常组合使用并定期压缩历史。

**Q：恢复时再次调用 LLM 会不会不同？** 会，所以将响应作为 Activity 结果记录；恢复优先复用已记录结果，不重复请求。

## ⑥ 场景题

**场景：贷款审批 Agent**

材料收集、征信查询、规则评估和人工审批分别 checkpoint。征信报告存对象存储引用；审批等待期间任务可暂停数天。恢复后从审批状态继续，不能重新拉取并产生不同评分。

## ⑦ 生产事故题

**事故：发布新版本后旧任务恢复失败，数千单卡住。**

- 根因：状态没有版本和迁移；代码重排了步骤/字段。
- 处置：冻结新发布，按旧工作流版本恢复；对无法迁移的任务导出状态并人工处理。
- 修复：状态 schema 版本化、向后兼容读取、迁移脚本和回放测试；工作流版本不可修改已运行历史。

## ⑧ 常见错误回答

- “把全部聊天记录保存起来就是 State。”——State 应是可校验的业务事实，不是无界文本。
- “重启后重新调用所有工具即可。”——会重复副作用且结果可能变化。
- “数据库事务能覆盖外部 API。”——跨系统需幂等、Outbox/补偿和状态查询。
- “Durable 就不需要监控。”——仍可能业务卡住、配额耗尽或人工等待超时。

## ⑨ Java/Python 实现思路

```python
def checkpoint(state, event):
    with db.transaction():
        db.append_event(state.run_id, event)
        db.save_snapshot(state.run_id, state.version + 1, state.to_json())

def recover(run_id):
    state = db.load_snapshot(run_id)
    for event in db.events_after(run_id, state.version):
        state = reduce(state, event)
    return migrate_if_needed(state)
```

Java 可使用 Temporal Java SDK 的 Workflow/Activity，或自建 `event + snapshot` 表；序列化仅使用可跨版本的 JSON/Protobuf，拒绝把不可序列化对象放入状态。

## ⑩ 权衡与最佳实践

- 状态最小化：保存事实和引用，不保存可重新计算的大段文本。
- 外部副作用先写幂等记录，再调用；结果回写与状态迁移可重试。
- 工作流代码保持确定性，时间/随机/网络放入 Activity，并记录返回值。
- 设定历史大小、快照频率、TTL 和加密策略；定期演练恢复与数据删除。
- 迁移优先兼容，无法安全迁移时暂停并人工接管，不隐式丢单。

**核验依据**

- [LangGraph Persistence 官方文档](https://langchain-ai.github.io/langgraph/concepts/time-travel/?h=time+travel)
- [LangGraph Interrupts 与 Checkpoint](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/breakpoints/)
- [Temporal 平台 Durable Execution](https://docs.temporal.io/)
- [AWS Durable Execution 幂等最佳实践](https://docs.aws.amazon.com/durable-execution/patterns/best-practices/idempotency/)
