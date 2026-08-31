# 第16章：Agent Loop、终止与长任务

## ① 核心知识点

- **Agent Loop**：读取状态→调用模型→执行工具→写入观察→判断是否继续，直到完成、失败、取消或转人工。
- **终止条件**：业务完成、不可恢复错误、用户取消、预算耗尽、截止时间到、连续空转或风险升级。
- **硬上限**：最大步数、模型调用数、工具调用数、token、费用和墙钟时间；任何一个超限都要可解释地停止。
- **进展检测**：比较状态哈希、目标指标和工具结果；连续若干轮无状态变化即判定循环。
- **长任务**：拆为可恢复阶段，用持久状态和检查点；前台返回 `run_id`，后台异步执行。
- **取消与暂停**：取消信号可传播到队列和下游；不可中断步骤完成后再停，并标记状态。
- **背压**：并发、队列长度和租户配额受控，避免 Agent 自我扩张。
- **最终答复**：必须引用真实工具结果和状态，不得把“计划完成”说成“业务完成”。

## ② 原理

```text
while budget_ok:
    decision = model(state)
    if decision.finish: return SUCCESS
    if not allowed(decision): return BLOCKED
    observation = execute(decision.tool, deadline)
    state = reduce(state, observation)
    if no_progress(state): return NEED_HUMAN
return TIMEOUT_OR_BUDGET
```

循环控制由运行时负责，模型只能建议下一步。每轮先检查预算，再执行工具；工具结果写入检查点后才允许下一轮，避免进程崩溃丢失进度。

## ③ 企业架构

- **Run Manager**：创建 `run_id`、租户配额、状态机、取消和恢复接口。
- **Loop Worker**：短循环同步；长任务由队列调度，心跳续租。
- **Checkpoint Store**：保存每轮输入摘要、决策、工具结果引用和预算余额。
- **Watchdog**：检测超时、无进展、重复调用和孤儿任务，触发暂停/转人工。
- **通知层**：阶段性进度、审批请求、失败告警；避免每轮都打扰用户。

## ④ 面试标准答案

> Agent Loop 不是“while true 调模型”，而是带预算和状态机的受控执行。运行时在每轮检查截止时间、步数、成本和风险；工具结果持久化后再继续。终止包括成功、不可重试失败、取消、超时、预算耗尽和无进展，达到上限要返回可恢复的 run_id。长任务采用异步队列、心跳、检查点和幂等步骤，进程重启后从最近检查点恢复。

## ⑤ 高频追问

**Q：如何判断模型陷入循环？** 监控状态哈希、工具参数重复、目标指标无变化；连续 N 轮无进展就暂停并转人工。

**Q：达到最大步数怎么办？** 保存状态和原因，给用户“未完成”及下一步选项，不伪造成功。

**Q：长任务如何让用户看到进度？** 返回 `run_id`，提供阶段事件和查询接口；最终结果由持久状态生成。

## ⑥ 场景题

**场景：月度财务对账**

按账期拆分批次；每批完成后写检查点和对账差异。批次失败只重跑该批；连续三批无新差异则暂停检查规则。前台立即返回任务号，完成后通知并提供差异报告。

## ⑦ 生产事故题

**事故：Worker 重启后从头执行，重复发送 2 万封邮件。**

- 根因：循环状态只在内存，副作用步骤没有持久幂等键。
- 处置：暂停任务、按发送供应商回执去重、补发失败项。
- 修复：每步完成后 checkpoint；邮件键 `campaign:user` 唯一；恢复时先加载状态并查询未知发送结果。

## ⑧ 常见错误回答

- “给模型一个停止词就能终止。”——模型可能忽略，必须由运行时硬控制。
- “长任务一直挂在 HTTP 请求里。”——易受超时和重启影响，应异步化。
- “无进展只看文本是否变化。”——文本变化不等于业务状态变化，应看结构化状态和目标指标。
- “取消就是杀进程。”——可能留下半完成副作用，需协作取消和状态标记。

## ⑨ Java/Python 实现思路

```python
def run_agent(state):
    for _ in range(state.max_steps):
        if deadline_exceeded(state) or state.cancelled:
            return checkpoint_and_stop(state, "stopped")
        decision = decide(state)
        if decision.done:
            return finish(state)
        obs = execute_idempotent(decision, state.run_id)
        new_state = reduce(state, obs)
        if no_progress(state, new_state):
            return checkpoint_and_stop(new_state, "no_progress")
        state = checkpoint(new_state)
    return checkpoint_and_stop(state, "budget_exhausted")
```

Java 侧用 `ScheduledExecutorService` 做 watchdog、`CompletableFuture` 传播取消；生产长任务交给 Temporal、Durable Functions 等持久运行时。

## ⑩ 权衡与最佳实践

- 循环上限宁可保守，超限转异步/人工；不要让模型自行放宽预算。
- 进度事件按阶段聚合，减少日志和通知噪音。
- 每个副作用步骤有幂等键、超时、取消语义和补偿路径。
- 通过故障注入测试进程崩溃、网络分区、重复消息和恢复一致性。
- 将“完成”定义为可验证业务条件，而不是模型输出一句完成。

**核验依据**

- [Temporal 平台：崩溃恢复与持久执行](https://docs.temporal.io/)
- [Temporal Durable Execution 技术指南](https://assets.temporal.io/durable-execution.pdf)
- [LangGraph Workflows 与 Agents 官方文档](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
