# 第01章：企业级 Agent 到底是什么

## ① 核心知识点

- Agent 是“模型决策 + 工具执行 + 状态反馈”的闭环系统；模型只负责概率性判断，业务代码负责权限、事务、重试和终止。
- 与 LLM/ChatBot/RAG/Workflow 的边界：LLM 单轮生成；ChatBot 维护对话；RAG 固定“检索后生成”；Workflow 路径预先定义；Agent 可依据状态动态选择下一步。
- 生产必备：目标与完成标准、上下文、持久状态、工具契约、运行时、护栏、追踪与评测。
- Function Calling 只是结构化动作接口，不等于完整 Agent。
- 高风险动作（付款、删库、发信）必须人工确认或规则审批，不能只靠提示词。

## ② 原理

典型循环为 `Observe → Decide/Plan → Act → Observe`：读取任务和状态，选择工具并生成参数，执行后把结果写回状态，直到完成、超时、达到步数上限或转人工。上下文应按优先级组装（系统规则>权限>任务>检索>工具结果），状态保存事实而非完整聊天记录。每个工具使用 JSON Schema 校验输入，输出带 `status/error_code/request_id`，便于重试和审计。终止条件、幂等键和超时由运行时强制执行，避免模型陷入循环。

## ③ 企业架构

```text
API/IM → 会话与身份层 → Agent Runtime(状态机/队列/超时)
                         ├─ LLM Gateway(路由、限流、缓存)
                         ├─ Tool Gateway(白名单、Schema、审计)
                         ├─ RAG/业务服务/数据库
                         ├─ Guardrail(输入、输出、风险审批)
                         └─ Trace、指标、评测与反馈库
```

状态存储使用带版本的数据库，长任务用队列和 checkpoint 支持恢复；工具服务与 Agent 解耦，通过 mTLS/OAuth 和最小权限访问。将固定步骤编排成 Workflow，仅把需要判断的节点交给 Agent，可控性最高。

## ④ 面试标准答案

> 企业级 Agent 不是“模型加几个函数”，而是围绕目标运行的可恢复闭环。模型决定下一步，运行时持久化状态、执行工具、控制超时和重试，护栏负责权限与风险，观测和评测保证可运营。路径确定的流程用 Workflow，路径不确定且需要多步工具协作的局部才使用 Agent；高风险操作必须审批和幂等。

## ⑤ 高频追问

1. **如何防止死循环？** 运行时设置最大步数/总时长，检测重复动作，失败后退避重试，仍失败转人工。
2. **记忆放哪里？** 短期状态放 checkpoint；长期偏好经用户同意后存结构化表，敏感信息加密并设 TTL，不把全部历史塞进 Prompt。
3. **多 Agent 是否更好？** 只有角色边界和收益可量化时拆分；默认单 Agent+工具，减少通信和调试成本。
4. **如何证明安全？** 工具白名单、参数与权限校验、审批日志、越权/提示注入测试和线上审计。

## ⑥ 场景题

**场景：客服退款。** Agent 读取订单和用户身份，调用规则服务计算可退金额；金额≤5000 元自动退款，超过则创建审批单。每次写操作带 `request_id` 幂等键，退款成功后再通知用户；支付超时只重试查询，不重试扣款。任何权限不足、证据缺失或风险命中都转人工。

## ⑦ 生产事故题

**事故：重复退款。** 原因是模型在工具超时后再次发起扣款且工具非幂等。处置：立即关闭写工具开关、按订单冻结重复请求；核对支付流水并人工冲正；补上服务端幂等键、唯一约束和“查询确认→单次提交”状态机；新增超时/重试演练和重复调用告警。验证标准：同一 `order_id+operation` 并发 100 次只产生一笔支付。

## ⑧ 常见错误回答

- “模型自己会记住上下文”——错，必须由应用持久化并控制上下文窗口。
- “加一个更强模型就能解决越权”——错，权限和审批必须在工具/服务端执行。
- “Function Calling 就是 Agent”——错，它只描述动作意图，没有循环、状态和恢复。
- “所有流程都应该 Agent 化”——错，确定性流程优先 Workflow，降低延迟与风险。

## ⑨ Java/Python 实现思路

```python
MAX_STEPS = 8
state = load_checkpoint(task_id)
for step in range(MAX_STEPS):
    decision = llm_json(system_rules, state, tools_schema)
    if decision["type"] == "final":
        save_checkpoint(task_id, state); return decision["answer"]
    tool = allowlist[decision["tool"]]
    args = validate_schema(tool.schema, decision["args"])
    result = tool.call(args, idempotency_key=f"{task_id}:{step}")
    state = update_state(state, result); save_checkpoint(task_id, state)
raise RuntimeError("step limit; handoff required")
```

Java 中用 `sealed interface Event` 定义状态事件，`ExecutorService`/消息队列承载异步任务，Jackson 校验工具参数，Resilience4j 做超时和退避；不要在 Controller 里写循环。日志至少包含 `trace_id、task_id、prompt_version、tool、latency、error_code`，敏感字段脱敏。

## ⑩ 权衡与最佳实践

- Agent 自主性越高，覆盖未知路径越好，但成本、延迟和不可预测性上升；先用 Workflow 骨架，逐步放开决策节点。
- 单 Agent 易观测，多 Agent 易分工；以端到端成功率和运维成本做拆分依据。
- 先定义完成/失败/转人工三态，再设计 Prompt；所有外部副作用都要幂等、可回滚、可审计。
- 用离线回放集和线上抽样持续评测，不以“看起来回答不错”作为上线标准。

**核验依据**

- [OpenAI：构建 AI Agent 实践指南](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
- [OpenAI Agents SDK：Agents](https://openai.github.io/openai-agents-js/guides/agents/)
- [NIST AI 风险管理框架](https://www.nist.gov/itl/ai-risk-management-framework)
- [OWASP LLM 应用十大风险](https://genai.owasp.org/llm-top-10/)
