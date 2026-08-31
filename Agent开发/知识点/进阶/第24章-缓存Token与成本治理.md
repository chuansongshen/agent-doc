# 第24章：缓存、Token 与成本治理

## ① 核心知识点

- **Prompt 缓存**：复用稳定前缀（系统指令、工具定义、示例）；动态用户内容放后面。命中依赖精确前缀和缓存生命周期。
- **KV Cache**：推理服务复用已计算的键值状态，降低长上下文首 token 延迟；通常由模型服务层管理。
- **响应缓存**：对确定性、低时效查询缓存最终答案；必须绑定模型、提示、数据版本和权限。
- **语义缓存**：按嵌入相似度复用近似问题答案；只适合容错场景，金融/权限敏感请求默认禁用。
- **RAG 缓存**：缓存查询改写、Embedding、检索结果或文档片段；键必须包含索引版本、过滤条件和租户。
- **工具缓存**：只读 API 可短 TTL 缓存；写操作和用户特定结果禁止共享缓存，副作用不能靠缓存“模拟成功”。
- **失效与污染**：版本、TTL、事件失效、权限变化都能使缓存失效；检测提示注入、恶意文档和跨租户污染。
- **Token Budget**：按请求/任务/租户设置输入、输出、推理、工具和总费用预算，超限降级或转人工。
- **模型路由**：按质量、延迟、数据驻留和成本选择模型；小模型处理分类/改写，大模型处理复杂决策。
- **成本归因**：每次调用带 `tenant、user、agent、run、tool、model、cache_hit` 标签，区分输入、缓存输入、输出和失败重试成本。
- **边界规则**：缓存读写使用幂等键；缓存/模型调用超时回源或降级，不把未知结果写入缓存；超预算需降级或人工审批；缓存层恢复后以版本校验，禁止旧值回灌。

## ② 原理

```text
请求 → 规范化/权限 → 多级缓存查询
  命中：校验版本/ACL/新鲜度后返回
  未命中：预算检查 → 路由模型 → 记录用量 → 写入可缓存层
```

- **缓存键**：`hash(model_snapshot, prompt_version, tool_schema, data_version, tenant_scope, normalized_input)`。
- **失效顺序**：权限变化/数据撤回立即失效；知识库发布按版本失效；普通数据按 TTL 或事件失效。
- **防污染**：缓存值保存来源和生成时间；检索/工具结果变更或被撤回时不能继续命中。
- **预算算法**：预估输入 + 预留输出 + 可能重试成本；剩余预算不足时缩短上下文、切换模型或停止。

## ③ 企业架构

- **Cache Gateway**：统一键规范、TTL、ACL、加密、命中指标和失效事件。
- **分层缓存**：进程 LRU（极短）、Redis（共享响应/检索）、模型侧 Prompt/KV Cache、对象存储（大结果）。
- **Token Meter**：解析 API usage，记录输入/缓存输入/输出/推理 token，按标签汇总到成本平台。
- **Budget/Router**：预算、配额、限流和模型路由策略；按任务重要性设置降级路径。
- **可靠性**：缓存命中校验超时即回源；写入失败不影响主流程；恢复时按版本和 ACL 重建，避免污染。
- **数据治理**：缓存内容脱敏、租户隔离、删除传播、密钥轮换和访问审计。

## ④ 面试标准答案

> 成本治理不是简单把结果放 Redis，而是从 token 产生、缓存命中、模型路由到业务归因的闭环。Prompt/KV 缓存适合稳定前缀和推理复用；响应、语义、RAG、工具缓存必须绑定模型/提示/数据版本、租户和权限，并有 TTL 与失效机制。每次请求预留输入、输出和重试预算，按质量、延迟和成本路由模型。所有调用记录 token、缓存命中、租户和 run 标签，按单位业务成本评估，命中错误或数据撤回时可快速失效。

## ⑤ 高频追问

**Q：Prompt 缓存为什么命中率低？** 动态内容放在前缀、工具顺序不稳定、模型版本/键变化或缓存已过期；应固定前缀并规范化目录。

**Q：语义缓存能否用于所有问答？** 不能；近似答案可能不满足时效、权限和精确性要求，高风险请求应绕过。

**Q：KV Cache 和 Prompt Cache 是一回事吗？** 目标都为复用计算，但层次不同：KV 是推理运行态，Prompt Cache 是 API/服务层复用输入前缀。

**Q：如何防止重试把成本打爆？** 统一重试预算、按错误分类、指数退避并计入总 token/费用；429 时遵循服务端节流。

## ⑥ 场景题

**场景：企业知识问答成本过高**

固定系统指令和工具定义置于前缀以利用 Prompt Cache；Embedding 和文档片段按索引版本缓存；简单 FAQ 用响应缓存，时效数据绕过。入口先估算 token，超过预算压缩上下文或路由小模型；仪表盘按部门/应用显示每次有效回答成本。

## ⑦ 生产事故题

**事故：知识库发布后，缓存仍返回旧的薪资政策。**

- 根因：响应缓存键没有知识库版本，发布事件也未触发失效。
- 处置：立即按业务域清空旧缓存，公告期间强制回源并人工核对答案。
- 修复：键加入 `index_version/policy_version`；发布、撤回和权限变更发送失效事件；缓存结果附来源版本和过期时间。

## ⑧ 常见错误回答

- “缓存命中就一定省钱。”——写入成本、失效和错误命中可能抵消收益。
- “把所有请求做语义缓存。”——近似答案会带来事实、权限和合规风险。
- “只看输出 token 控制成本。”——输入、缓存读写、推理和工具调用也计费。
- “模型换个别名不影响缓存。”——快照、提示、工具 Schema 变化都可能使结果失效。

## ⑨ Java/Python 实现思路

```python
def cached_answer(req):
    key = stable_hash(req.model, req.prompt_version, req.tool_schema,
                      req.data_version, req.tenant, normalize(req.input))
    hit = cache.get(key)
    if hit and acl_ok(hit, req.actor) and not expired(hit):
        meter.record(req, cache_hit=True, tokens=0)
        return hit.value
    budget.ensure(req.estimated_input + req.max_output)
    resp = model.call(req)
    meter.record(req, cache_hit=False, usage=resp.usage)
    if resp.cacheable and resp.confidence >= .9:
        cache.put(key, resp.value, ttl=300)
    return resp.value
```

Java 可使用 Caffeine/Redis 做应用层缓存，`CacheKey` 必须包含版本和租户；成本记录通过拦截器统一采集，避免业务代码漏记或重复记账。

## ⑩ 权衡与最佳实践

- 先优化提示前缀、上下文大小和模型路由，再引入复杂语义缓存；命中率不是唯一目标。
- 缓存默认短 TTL、按租户隔离、结果带来源和版本；高风险/强时效请求默认回源。
- Token Budget 必须由运行时硬执行，重试、工具和并行 Agent 共用总预算。
- 以“每个成功任务成本、缓存命中率、错误命中率、P95 延迟、质量分”评估收益。
- 预算告警和失效开关可在分钟级生效；成本异常先降级模型/关闭非关键缓存写入，再定位根因。

**核验依据**

- [OpenAI Prompt Caching 官方说明](https://openai.com/index/api-prompt-caching/)
- [OpenAI Responses API：用量与缓存字段](https://developers.openai.com/api/reference/cli/resources/responses/methods/create)
- [OpenAI Token 计数与定价说明](https://help.openai.com/en/articles/4936856)
- [CacheBlend：RAG KV Cache 复用论文](https://arxiv.org/abs/2405.16444)
