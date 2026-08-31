# 第20章：Multi-Agent 与 A2A

## ① 核心知识点

- **Supervisor**：一个主管 Agent 负责拆分、分派、汇总和终止；子 Agent 不直接互相调用，边界清晰。
- **Peer-to-Peer**：同级 Agent 通过消息协作，适合动态协商；需要明确负责人、超时和冲突解决者。
- **层级协作**：主管→领域主管→执行 Agent，按权限和上下文隔离；限制层级深度和转派次数。
- **Handoff**：把任务控制权和必要上下文交给另一个 Agent，必须携带 `task_id、目标、约束、证据、截止时间`。
- **A2A**：面向独立 Agent 的互操作协议，支持能力发现（Agent Card）、任务生命周期、消息/工件和流式更新。
- **MCP**：面向 LLM 应用与工具/资源/提示的连接协议；MCP 解决“Agent 调工具”，A2A 解决“Agent 找 Agent 协作”。
- **消息契约**：版本化 Schema、关联 ID、幂等键、来源、租户和安全标签；拒绝未知字段和越权指令。
- **可靠性**：每次委派设置 deadline、重试预算、取消语义和最大 fan-out；未知结果先查询，不重复副作用。
- **成本控制**：按任务设置总 token/费用预算，限制并行 Agent 数和上下文复制；汇总尽量使用小模型。
- **终止**：达到目标、预算、超时、连续无进展、循环检测或人工介入即停止并返回状态。

## ② 原理

```text
用户目标 → 协调器选择 Agent → A2A 任务/消息
         → 子 Agent 执行工具(MCP) → 返回状态/工件
         → 协调器校验、汇总或重新分派
```

- A2A 通信只交换任务和工件，不共享对方内部提示、记忆或工具权限。
- handoff 是控制权转移；普通消息是协作，不应隐式改变任务所有者。
- 主管必须对输出做 Schema、证据和权限校验；子 Agent 的自然语言不能直接触发高风险副作用。
- 任务状态用 `SUBMITTED/RUNNING/INPUT_REQUIRED/COMPLETED/FAILED/CANCELED`，状态更新按 `task_id` 幂等。

## ③ 企业架构

- **Coordinator**：全局预算、路由、依赖、汇总和终止。
- **Agent Registry**：Agent Card、能力、版本、SLO、数据驻留、权限和负责人。
- **A2A Gateway**：认证、授权、协议转换、任务查询、流式/回调和审计。
- **各 Agent Runtime**：独立状态、工具网关（可用 MCP）、检查点和限额；默认不共享数据库。
- **共享治理**：Trace 贯穿父子任务；成本按 `root_run_id/agent_id/tool_id` 归因；策略中心统一审批和黑名单。

## ④ 面试标准答案

> 多 Agent 的关键不是数量，而是职责、权限和上下文是否真正隔离。Supervisor 适合统一规划和汇总，Peer 适合动态协商，层级模式适合组织化任务。handoff 要转移明确的任务契约和控制权，不能把全部上下文复制给下游。A2A 是 Agent 间互操作协议，MCP 是应用与工具/资源的连接协议，两者互补。生产上要限制 fan-out、深度、token、费用和转派次数，消息带幂等键和截止时间，主管对结果做证据和权限校验。

## ⑤ 高频追问

**Q：什么时候不该用多 Agent？** 单一模型 + 工作流能稳定完成，或子任务共享大量上下文时，多 Agent 只增加延迟和成本。

**Q：A2A 和 MCP 如何选？** 调用数据库、搜索、文件等能力用 MCP；委派给另一套独立 Agent 用 A2A；可组合使用。

**Q：子 Agent 失败如何恢复？** 由协调器按错误类型重试或换 Agent；先查询原 `task_id`，禁止盲目重新执行副作用。

**Q：如何防止 Agent 死循环？** 记录 handoff 图和状态哈希，限制深度/次数/时间；重复路径或无进展立即暂停并告警。

## ⑥ 场景题

**场景：跨境采购助手**

主管 Agent 先分派合规、价格和物流三个只读 Agent 并行查询；价格 Agent 通过 MCP 调供应商 API；需要外部专业 Agent 时用 A2A 委派并传递报价 Schema。主管汇总证据，超过采购额度进入人工审批，最终下单只由一个受控 Agent 执行。

## ⑦ 生产事故题

**事故：主管与子 Agent 互相 handoff，调用量暴涨且任务未结束。**

- 根因：无任务所有者、最大深度和循环检测，双方都认为对方负责。
- 处置：按 `root_run_id` 熔断转派，暂停非关键子任务，保留已完成工件。
- 修复：每个任务只有一个 owner；handoff 必须指定下一 owner 和返回条件；限制深度/次数/费用并对重复边告警。

## ⑧ 常见错误回答

- “多 Agent 就是多次调用模型。”——还需要角色契约、状态、路由、权限和汇总。
- “A2A 可以直接调用对方数据库。”——A2A 只传任务/工件，工具访问仍由对方控制。
- “MCP 和 A2A 是同一个协议。”——前者连接应用与工具/资源，后者连接独立 Agent。
- “让每个 Agent 共享全部上下文最准确。”——增加泄露、成本和提示注入风险，应按需传递。

## ⑨ Java/Python 实现思路

```python
def delegate(task, agent):
    envelope = {
        "task_id": task.id, "root_run_id": task.root_id,
        "goal": task.goal, "constraints": task.constraints,
        "deadline": task.deadline, "idempotency_key": f"{task.id}:{agent.id}"
    }
    return a2a_gateway.send(agent.card.url, envelope)

def supervise(task, results):
    if task.cost >= task.budget or has_cycle(task.trace):
        return stop(task, "budget_or_cycle")
    return validate_and_merge(results, required_schema=task.output_schema)
```

Java 可用 `CompletableFuture.allOf` 做受限并行，配合信号量控制 fan-out；A2A/MCP SDK 只负责协议，鉴权、幂等、审计仍由企业网关统一处理。

## ⑩ 权衡与最佳实践

- 先模块化单 Agent，再按权限/上下文/发布边界拆分；避免为“看起来智能”而多 Agent 化。
- Supervisor 默认负责终止和预算；Peer 协作必须有 owner、超时和仲裁者。
- A2A 传最小任务契约和工件引用，MCP 工具按最小权限暴露；不共享内部记忆和密钥。
- 子任务结果带证据、版本和置信等级；高风险动作由统一工具网关和 HITL 控制。
- 监控 fan-out、handoff 深度、重复路径、子任务成功率、总 token 和单位业务成本。

**核验依据**

- [A2A Protocol 官方规范](https://a2a-protocol.org/v1.0.0/)
- [A2A Protocol 规范（任务与消息模型）](https://a2a-protocol.org/v0.3.0/specification/)
- [Model Context Protocol 官方规范](https://modelcontextprotocol.io/specification/2025-03-26)
- [LangGraph Multi-Agent/自定义工作流文档](https://docs.langchain.com/oss/python/langchain/multi-agent/custom-workflow)
