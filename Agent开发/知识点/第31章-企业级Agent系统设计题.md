# 第31章：企业级 Agent 系统设计题

## ① 核心知识点

- 先澄清目标：用户、任务边界、成功标准、SLO、合规等级、峰值并发、数据驻留和预算。
- 采用“确定性骨架 + 模型决策”：Workflow 负责状态、权限和副作用，Agent 负责分类、规划和自然语言交互。
- 核心组件：接入层、会话/任务服务、Agent Runtime、模型路由、检索与记忆、Tool Gateway、策略引擎、队列/Worker、状态库、评测与可观测性。
- 多 Agent 仅在职责、权限或扩展团队确有隔离收益时使用；Agent 间协议需定义任务、能力、状态和错误。
- 数据分层：短期上下文、可检索知识、长期记忆、审计事件分别存储并设置 ACL/保留期。
- 设计输出必须覆盖容量、故障、降级、回滚、安全、成本和演进路线，而不只是画框图。

## ② 原理

- 请求路径：鉴权→意图识别→检索/规划→工具调用→结果校验→状态提交→响应；每个副作用步骤可重试且幂等。
- 状态机比“无限对话历史”可靠：保存 `task_state、step、version、pending_approval、retry_count`，支持暂停和恢复。
- 模型路由按能力、延迟、价格、区域和配额打分；高风险任务固定经受控模型并要求人工审批。
- 上下文预算：系统指令和安全策略优先，检索按相关性/新鲜度/权限过滤，历史压缩成可验证摘要。
- 跨 Agent 协作使用显式消息和超时，禁止共享可变内存；失败可回退到单 Agent 或人工队列。
- 容量估算：`实例数 ≈ 峰值并发 × 平均处理时长 / 单实例并发上限`，再按模型 TPM、工具 QPS 校验瓶颈。

## ③ 企业架构

```text
Web/IM/API → API Gateway（SSO、租户配额、审计）
           → Conversation/Task Service → Agent Runtime
                                     ├→ Model Router → 多模型/私有模型
                                     ├→ Retrieval/Memory → 向量库+文档库
                                     ├→ Tool Gateway → ERP/CRM/MCP 服务
                                     └→ Policy/Approval → 人工工作台
           → Queue/Worker（长任务）→ 状态库/对象存储
           → OTel/Prometheus/Evals（质量与运维闭环）
```

- 每层定义超时、重试、输入输出 schema 和数据分类；跨层只传必要字段。
- Tool Gateway 统一协议、权限和审计；需要跨组织 Agent 协作时采用 A2A 等标准协议，工具接入遵循 MCP 规范。
- 部署在 K8s 多副本跨区，状态外置；数据库主从/多活方案按一致性要求选择，灾备目标写明 RPO/RTO。
- 发布采用提示/模型/工具独立版本，先离线评测再灰度；异常由 feature flag 一键回滚。
- 成本中心按租户、模型、工具分摊 token、GPU、存储和人工审核成本。

## ④ 面试标准答案

> 我会先问清任务成功标准、SLO、峰值并发、数据和合规边界，再画分层架构。入口做身份和租户配额，Agent Runtime 以状态机编排模型、检索和工具，长任务走队列与 checkpoint；策略引擎负责最小权限和人工审批，工具网关统一 schema、幂等、超时和审计。模型路由支持降级与成本上限，状态和记忆分层存储。上线前做离线评测和压测，线上用 OTel/Prometheus 监控，按版本灰度并准备回滚、灾备和事故演练。

## ⑤ 高频追问

- **为什么不全做成多 Agent？** 协作通信和调试成本高；先单 Agent+Workflow，边界清晰且收益可量化再拆分。
- **记忆放向量库可以吗？** 向量适合检索，不适合权威状态；订单、审批等事实仍由事务库维护。
- **如何保证模型不越权？** 模型只提出意图，策略引擎和工具服务端再次鉴权，默认拒绝。
- **跨区域如何做会话？** 会话状态带版本并存共享存储；按租户路由，故障时以幂等恢复而非复制连接。
- **如何证明设计能扛峰值？** 给出并发/时延/TPM 计算，压测模型、队列、数据库和限额，验证降级曲线。

## ⑥ 场景题

**场景：** 设计“采购寻源 Agent”：读取需求、比价、生成采购单，月末峰值 500 RPS，涉及金额审批。

**解法：** 需求解析和比价可并行，采购单写入走 Workflow；金额和供应商由规则服务校验，超过阈值人工审批；报价检索按租户 ACL 隔离并缓存；峰值通过队列削峰和供应商 API 配额调度；采购写入以业务幂等键防重复，审计保留完整证据。

## ⑦ 生产事故题

**事故：** 新增一个“自动创建工单”子 Agent 后，出现工单重复和跨租户可见。

**排查与修复：** 追踪发现子 Agent 复用了父 Agent 管理员令牌且重试无幂等；记忆查询未带 `tenant_id`。立即禁用子 Agent 写权限、撤销令牌和隔离数据；改用短期能力令牌、租户过滤和唯一键，增加跨租户安全回归与重复调用测试，灰度后恢复。

## ⑧ 常见错误回答

- “模型越大效果越好。”——未结合任务、成本、时延和合规。
- “所有状态都放 Redis/向量库。”——缺少事务一致性和审计依据。
- “多 Agent 越多越智能。”——通信和故障面呈乘法增长。
- “只要接入 MCP/A2A 就安全可互操作。”——协议不替代身份、策略和业务校验。
- “上线后再补监控和评测。”——没有基线就无法证明质量和回归。

## ⑨ Java/Python 实现思路

Python（最小状态机）：

```python
def run(task):
    state = load_state(task.id)
    while state.step not in {"done", "failed"}:
        action = planner(state)                 # 模型仅给出意图
        checked = policy_check(state.user, action)
        result = execute_idempotent(checked, key=f"{task.id}:{state.step}")
        state = save_checkpoint(next_state(state, result))
    return state
```

Java：Spring Boot 编排 API，Temporal/Stateful Workflow 管理长任务，Kafka 做事件总线，Spring Security/OPA 做策略，Micrometer+OpenTelemetry 统一指标与 trace。

## ⑩ 权衡与最佳实践

- 同步响应体验好但受超时限制；超过数秒或需人工审批的任务统一异步化。
- 私有模型降低数据外泄和供应商依赖，运维成本更高；按数据分类实施混合路由。
- 强一致写路径牺牲吞吐；非关键通知可最终一致，关键交易必须事务/幂等。
- 标准协议降低接入成本但版本演进要谨慎；锁定兼容版本并做契约测试。
- 从一个可回滚的最小闭环开始，先证明成功率、成本和人工节省，再扩展能力与 Agent 数量。

**核验依据**

- [Model Context Protocol 规范](https://modelcontextprotocol.io/specification/2025-06-18)
- [A2A Protocol 规范](https://a2a-protocol.org/latest/specification/)
- [Kubernetes Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [OpenTelemetry 语义约定](https://opentelemetry.io/docs/specs/semconv/)
