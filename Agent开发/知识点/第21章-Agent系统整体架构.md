# 第21章：Agent 系统整体架构

## ① 核心知识点

- **分层**：交互层、Agent Runtime、模型层、工具层、状态/记忆层、治理与观测层。
- **控制面与数据面**：控制面管理提示/工具/策略/模型版本；数据面执行用户任务并隔离租户。
- **运行时职责**：循环、计划、状态、超时、重试、checkpoint、审批、取消和成本预算；模型只做建议。
- **工具治理**：注册、Schema、权限、版本、SLO、幂等、审计和弃用。
- **状态与记忆**：短期线程状态用于恢复；长期记忆按用户/租户隔离并可删除；敏感数据不默认进入模型上下文。
- **安全**：零信任工具访问、输入/输出过滤、提示注入防护、密钥托管、沙箱和数据最小化。
- **可靠性**：队列、限流、熔断、异步长任务、重试预算、死信和对账。
- **质量闭环**：离线评测、线上 Trace、人工反馈、回放回归、灰度和一键回滚。

## ② 原理

```text
请求 → 身份/租户 → 任务编排 → 模型推理
                         ↓
                 策略/工具网关
                 ↙      ↓      ↘
              工具执行  状态库  HITL
                         ↓
                   结果/审计/指标
```

关键原则是“模型不拥有权限、运行时拥有控制权”：每次工具调用都经过策略、Schema、幂等和审计；每次状态变化都可追踪和恢复。

## ③ 企业架构

- **接入层**：API Gateway、SSO、租户配额、请求签名和敏感信息初筛。
- **Agent Runtime**：任务状态机、规划/循环、上下文组装、模型路由、预算和取消。
- **Model Gateway**：供应商适配、超时/重试、成本统计、内容安全和模型降级。
- **Tool Gateway**：工具注册、鉴权、Schema、限流、幂等、沙箱、审计。
- **State/Memory**：检查点、事件、向量/文档检索、TTL、加密和删除。
- **Governance**：策略即代码、评测平台、Trace、告警、版本发布、审计报表。

## ④ 面试标准答案

> 企业级 Agent 采用分层和控制面/数据面分离。Runtime 负责状态机、循环、预算、超时、重试、恢复和审批；Model Gateway 负责模型适配与成本；Tool Gateway 负责权限、Schema、幂等和审计；状态层保存检查点和记忆；治理层提供评测、Trace、灰度和回滚。高风险动作固定工作流并接入 HITL，模型只在受控边界内决策。

## ⑤ 高频追问

**Q：为什么不让模型直接访问内部服务？** 无法可靠执行权限、审计、限流和幂等；必须经工具网关。

**Q：单 Agent 还是多 Agent？** 先单 Agent + 工作流；只有职责、权限或上下文确实隔离时再拆多 Agent。

**Q：如何支持多模型？** 通过 Model Gateway 抽象统一协议和指标，按任务质量、延迟、成本和数据驻留路由。

**Q：怎么定义上线门槛？** 核心任务成功率、危险动作拦截率、P95 延迟、成本上限、恢复演练和审计完整性同时达标。

## ⑥ 场景题

**场景：企业 IT 服务台 Agent**

FAQ/RAG 处理只读问答；密码重置走固定工作流并校验身份；生产变更需变更单和审批；工具调用统一过网关。长时间排障返回任务号，状态和日志按租户隔离，最终由规则检查是否真正恢复。

## ⑦ 生产事故题

**事故：知识库误配置导致 Agent 将 A 租户数据返回给 B 租户。**

- 根因：检索索引和缓存键未包含租户，运行时也未做结果归属校验。
- 处置：关闭跨租户检索、清理缓存、审计访问、通知受影响客户。
- 修复：所有索引/缓存/状态键强制带租户；结果返回前做 ACL 过滤；增加跨租户回归测试和数据泄露告警。

## ⑧ 常见错误回答

- “把所有能力放进一个超级 Agent。”——权限、上下文和故障面不可控。
- “有 RAG 就安全。”——检索结果仍可能越权或含恶意指令。
- “模型厂商负责全部监控。”——业务状态、工具副作用和租户指标仍需自建。
- “上线后看用户满意度就够了。”——还需任务成功、风险拦截、成本、延迟和恢复指标。

## ⑨ Java/Python 实现思路

```python
def handle(req):
    ctx = auth_and_tenant(req)
    run = runtime.start(ctx, budget=Budget(steps=20, cost=1.0))
    while run.active:
        decision = model_gateway.decide(run.context)
        policy.check(decision, ctx)
        result = tool_gateway.execute(decision, ctx, deadline=run.deadline)
        run = runtime.apply(run, result)  # 持久化 checkpoint
    return runtime.render(run)
```

Java 可按模块拆分 Spring Boot 服务；先用单仓库共享契约和 Trace，只有独立扩缩容、权限或发布节奏明确时再拆服务。

## ⑩ 权衡与最佳实践

- 先做“工作流骨架 + 少量 Agent 决策”，避免一开始多 Agent 化。
- 控制面配置版本化、可审计、可回滚；数据面无权修改策略。
- 所有外部调用有 deadline、幂等键、重试分类和审计；长任务异步并可恢复。
- 默认租户隔离和最小权限；敏感数据按需检索、脱敏和 TTL 清理。
- 建立从离线评测到线上 Trace、人工反馈、灰度发布的闭环；事故演练先于规模扩张。

**核验依据**

- [OpenAI API 快速开始与 Tools](https://platform.openai.com/docs/quickstart/make-your-first-api-request)
- [LangGraph Workflows 与 Agents 官方文档](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
- [Temporal 平台 Durable Execution](https://docs.temporal.io/)
- [OWASP AISVS：AI 安全验证标准](https://owasp.org/www-project-artificial-intelligence-security-verification-standard-aisvs-docs/)
