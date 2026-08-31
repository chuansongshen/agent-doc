# 第04章：大模型接入与 Model Gateway

## ① 核心知识点

- Model Gateway 是统一的模型访问层：屏蔽供应商协议差异，集中做路由、鉴权、限流、重试、审计、成本和可观测性。
- 统一请求/响应协议要覆盖文本、多模态、流式、工具调用、结构化输出、错误码和取消；不能只封装一个 `chat()` 方法。
- 路由依据包括任务能力、上下文长度、数据驻留、租户配额、实时延迟和价格；路由表必须版本化。
- 故障转移按错误类型处理：超时/限流可退避重试，参数错误/内容策略拒绝不应盲重试；跨模型重试要防止重复副作用。
- Gateway 不替代业务权限和安全策略；密钥在网关托管，租户隔离、脱敏和审批仍由上游/工具层负责。

## ② 原理

请求进入后依次执行：身份校验 → 能力与策略匹配 → 配额/预算检查 → 提供商适配 → 超时与重试 → 结果归一化 → 指标与账单。路由可用规则（任务类型、模型能力）加实时信号（p95、错误率、剩余配额）打分。重试采用指数退避加随机抖动，并设置总截止时间；流式响应要透传事件序号和终止原因。对非幂等工具调用，网关只重试模型请求，不自动重放业务副作用。记录供应商请求 ID，便于追责和故障定位。

## ③ 企业架构

```text
业务服务/Agent Runtime
        → Gateway API（统一 Schema、鉴权、预算）
        → 路由与策略引擎（能力、租户、地域、灰度）
        → Provider Adapter（OpenAI/其他厂商/自托管）
        → 供应商 API 或内部推理集群
        ↘ 日志、Trace、成本计量、配额与告警
```

Adapter 将供应商差异映射为内部 `request_id、finish_reason、tool_calls、usage`；保留原始响应哈希用于审计。配置中心管理模型别名、上下文上限、价格、区域和降级链，变更走审批和灰度。网关无状态部署，限流计数放 Redis/服务商配额系统；密钥使用 KMS，禁止写入日志。

## ④ 面试标准答案

> Model Gateway 是企业模型访问的统一控制面。它把多供应商协议归一化，并集中实现模型路由、限流、超时重试、流式和工具调用适配、成本计量及审计。路由按能力、延迟、价格、数据合规和租户策略决定，错误按可重试性分类；副作用请求用幂等键避免重复。业务权限不放在网关里，仍由工具服务端强制执行。

## ⑤ 高频追问

1. **为什么不让业务直连模型？** 会造成密钥散落、协议重复、配额失控和供应商锁定；网关统一治理，但保留直连通道用于故障演练。
2. **如何做多模型降级？** 先按任务能力筛选兼容模型，再按实时健康度和预算选择；切换时携带同一 trace 和重试预算。
3. **流式如何统一？** 内部定义事件协议（delta/tool/progress/completed/error），Adapter 转换供应商事件并保证顺序与心跳。
4. **如何控制成本？** 预估输入/输出 token，按租户预算拦截；利用提示缓存和小模型路由，但以评测确认质量不回退。

## ⑥ 场景题

**场景：集团客服同时接入三家模型。** 意图分类和摘要使用低价小模型，退款审批判断使用高能力模型；含个人数据的请求只路由到合规区域。每个租户有 TPM/日预算，超限返回可解释的降级提示。Gateway 将三家工具调用格式统一为内部 Schema，失败时只重试无副作用的推理请求，最终响应附带模型与版本。

## ⑦ 生产事故题

**事故：供应商短时 429 导致全站请求堆积。** 先在网关启用按租户和模型的令牌桶限流，熔断故障提供商并切换兼容模型；队列任务暂停非关键请求，前端显示排队状态。复盘发现重试无抖动且 `max_tokens` 过大，修复为指数退避、总截止时间和动态输出上限，增加供应商错误率、队列长度和降级率告警。验证：故障演练 10 分钟内 p95 受控、无请求风暴、关键流程成功率达标。

## ⑧ 常见错误回答

- “网关只做转发，越薄越好”——没有限流、审计和适配就无法治理多模型。
- “所有错误都重试三次”——参数错误、策略拒绝和上下文超限重试只会放大故障。
- “切换模型不会影响结果”——能力、上下文和工具协议可能不同，必须有兼容性测试。
- “把权限判断放网关即可”——业务资源状态和最终授权必须在工具服务端再次校验。

## ⑨ Java/Python 实现思路

```python
def complete(req):
    policy = policy_engine.check(req.tenant, req.task, req.data_region)
    candidates = route_table.compatible(req.capabilities, policy)
    for model in health_rank(candidates):
        try:
            return adapter(model).call(req, timeout=deadline(req))
        except (RateLimitError, TimeoutError) as e:
            metrics.inc("model_retry", model=model, reason=type(e).__name__)
            backoff_with_jitter()
    raise ServiceUnavailable("no compatible model")
```

Java 可用 WebClient/OkHttp 实现异步流式，Resilience4j 做 TimeLimiter、Retry、CircuitBreaker；用 Jackson 将供应商响应映射到内部 sealed DTO。日志只记录 token、延迟、状态和请求 ID，正文与密钥脱敏。

## ⑩ 权衡与最佳实践

- 多供应商提升可用性和议价能力，但增加协议适配与评测成本；先统一核心能力，少量扩展按业务价值推进。
- 复杂动态路由可能“聪明反被聪明误”；用可解释规则做基线，再引入健康度和成本优化。
- 重试提高成功率却会放大费用和延迟；按错误分类、总预算和幂等策略限制。
- 配置、模型快照、价格和能力矩阵必须版本化，灰度失败可一键回滚。

**核验依据**

- [OpenAI：模型选择与能力](https://developers.openai.com/api/docs/models)
- [OpenAI Responses API：创建响应](https://developers.openai.com/api/reference/cli/resources/responses/methods/create)
- [OpenAI：API 限流与指数退避建议](https://help.openai.com/en/articles/6891753-api-rate-limit-advice)
- [OpenAI：构建 Agent 的模型选择指南](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
