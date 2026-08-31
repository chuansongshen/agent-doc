# 第18章：Memory 与 Context 管理

## ① 核心知识点

- **短期记忆**：当前线程的消息、工具结果和阶段状态；服务于连续对话与恢复，按线程隔离。
- **长期记忆**：跨会话保留的用户偏好、事实和历史事件；必须有来源、时间、租户和删除策略。
- **情景记忆**：发生过的事件及结果（如“上次报销被退回”），适合按时间和任务检索。
- **语义记忆**：稳定事实、定义和关系（如部门编码），适合向量/关键词/知识图谱混合检索。
- **工作记忆**：当前任务的目标、约束、未完成步骤和临时变量；优先放入模型上下文。
- **记忆抽取**：从对话/工具结果提取候选事实，经过置信度、敏感性、用户同意和去重后写入。
- **检索与注入**：按权限过滤，再按相关性、时效性、来源可信度排序；只注入完成当前任务所需的最小内容。
- **冲突与衰减**：新事实不能静默覆盖旧事实；记录版本和证据，按 TTL/时间衰减，支持人工更正。
- **上下文压缩**：滑动窗口、递归摘要、结构化摘要和重要性裁剪；保留目标、约束、决策、待办和证据。
- **边界规则**：未知或低置信记忆不得当作事实；跨租户、敏感数据、审批决定不能写入共享记忆。写入使用 `memory_id+version` 幂等；检索/压缩超时则回退当前状态；敏感记忆需审批。

## ② 原理

```text
输入/工具结果 → 候选记忆抽取 → 过滤/脱敏/去重 → 持久化
当前任务 → 权限检索 → 排序/压缩 → Context 组装 → 模型
```

- 抽取与回答解耦：抽取失败不阻塞主任务，写入异步队列并可重试。
- 检索结果带 `memory_id、source、version、valid_from、valid_to、confidence`，模型只能引用可见证据。
- 冲突采用“有效期 + 来源优先级 + 人工确认”；无法判定时同时呈现冲突并询问用户。
- 压缩摘要必须可追溯到原消息；摘要失真或版本变更时重新生成。

## ③ 企业架构

- **Memory Service**：抽取、规范化、去重、版本、TTL、删除和审计。
- **多存储**：线程状态存关系库/Checkpoint；语义记忆存向量库；高价值关系存图数据库；原文存对象存储。
- **Context Builder**：合并系统指令、工作记忆、检索片段和工具结果，执行预算与脱敏。
- **可靠性**：写入采用幂等 upsert；检索超时回退短期状态；服务恢复后按事件补写，不能重复生成记忆。
- **策略层**：按租户、用户、数据域和用途做 ACL；禁止使用未授权跨域记忆。
- **质量闭环**：记忆命中率、过期命中率、冲突率、删除完成率、摘要覆盖率和上下文 token 占比。

## ④ 面试标准答案

> Memory 是跨步骤或跨会话管理的事实，Context 是本轮真正提供给模型的输入。生产系统通常把短期线程状态、工作记忆、长期语义/情景记忆分层存储；写入前做敏感性、同意、去重和版本控制，读取时做权限过滤和相关性排序。上下文超限时优先保留目标、约束、未完成步骤和证据，摘要必须可追溯。冲突不静默覆盖，过期记忆按 TTL 或时间衰减，用户可查看和删除。

## ⑤ 高频追问

**Q：向量库就是长期记忆吗？** 不是；它只是检索索引，还需要来源、权限、版本、TTL 和删除机制。

**Q：如何防止错误记忆长期污染？** 低置信候选不直接写入；记录证据和有效期，重要事实要求用户确认，定期评估和清理。

**Q：摘要会丢失什么？** 可能丢失数字、否定、时序和待办；用结构化字段保存约束/决策/证据，摘要仅作补充。

**Q：检索很多内容是否更好？** 过多会稀释注意力和增加成本；按任务预算取最小充分集合，必要时分批查询。

## ⑥ 场景题

**场景：差旅助手跨月记住用户偏好**

短期记忆保留本次行程；长期记忆只保存“偏好靠过道、预算上限”等用户明确确认的事实；情景记忆保存上次改签结果。检索按用户和租户 ACL 过滤，偏好变更写新版本并设置有效期，不把一次临时要求永久化。

## ⑦ 生产事故题

**事故：A 租户员工查询到 B 租户的客户偏好。**

- 根因：向量检索和缓存键未包含租户，Context Builder 未做最终 ACL。
- 处置：关闭跨租户检索、清理缓存、审计访问并通知安全团队。
- 修复：所有记忆记录强制 `tenant_id`；检索、缓存、日志三处校验归属；增加跨租户回归和删除演练。

## ⑧ 常见错误回答

- “把全部历史消息塞进 Prompt 就有记忆。”——会超限、泄露且无法治理。
- “向量相似度最高就是真实答案。”——相似不代表时效、权限和可信度。
- “新记忆直接覆盖旧记忆。”——应保留版本和证据，处理冲突与有效期。
- “摘要越短越好。”——短但丢失约束会导致错误执行，应以任务完成率衡量。

## ⑨ Java/Python 实现思路

```python
def build_context(run, query, budget):
    working = run.state["goal_constraints_todos"]
    memories = memory.search(query, tenant=run.tenant, acl=run.actor)
    memories = rank_and_dedupe(memories, now=time.time())
    selected = fit_budget(working, memories, budget)
    return redact(selected, run.actor)

def save_candidate(candidate, evidence, user_id):
    if candidate.confidence < .8 or contains_sensitive(candidate):
        return "PENDING_REVIEW"
    return memory.upsert_version(candidate, evidence=evidence, owner=user_id, ttl_days=180)
```

Java 可用 `MemoryRecord`（含版本、来源、ACL、有效期）配合 PostgreSQL + 向量扩展；Context Builder 统一做 token 预算和脱敏，不在各 Agent 中重复实现。

## ⑩ 权衡与最佳实践

- 先做线程状态和结构化工作记忆，再引入长期记忆；不为“可能有用”永久保存数据。
- 记忆写入默认保守、读取默认最小；提供查看、纠正和删除入口。
- 采用混合检索和时间/来源加权；结果必须带证据和版本，过期自动降权或剔除。
- 压缩保留目标、约束、决策、待办、数字和引用；摘要失败时回退到原文片段。
- 监控命中质量、冲突和泄露风险；记忆服务故障时降级为当前会话，不阻塞核心业务。

**核验依据**

- [LangGraph Persistence 官方文档](https://langchain-ai.github.io/langgraph/concepts/time-travel/?h=time+travel)
- [LangGraph Interrupts 与持久化恢复](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/breakpoints/)
- [MemGPT：Towards LLMs as Operating Systems 论文](https://arxiv.org/abs/2310.08560)
- [OpenAI Responses API：上下文管理与用量字段](https://developers.openai.com/api/reference/cli/resources/responses/methods/create)
