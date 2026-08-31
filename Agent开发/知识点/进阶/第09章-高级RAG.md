# 第09章：高级 RAG

## ① 核心知识点

- 高级 RAG 在基础“切块—向量检索—生成”上增加查询变换、结构化过滤、混合召回、重排、压缩、父子文档、多跳和图检索。
- 先判断问题类型：事实查找、跨文档比较、全局总结、时序/数值分析；不同类型使用不同检索路径。
- Query Rewrite/Multi-Query/HyDE 可改善口语和多轮查询，但必须保留原问题、限制调用次数并防语义漂移。
- Parent-Child 或 Small-to-Big 先以小块召回，再扩展到父段落；Contextual Compression 只保留与问题相关的句子。
- GraphRAG 适合跨实体、全局主题问题，索引和维护成本高，不应替代所有向量检索。

## ② 原理

高级 RAG 通常是“规划查询 → 多路召回 → 融合重排 → 上下文压缩 → 生成与引用”。查询分解把复杂问题拆成可检索子问题，执行后聚合证据；RRF 融合不同检索器的排名，避免分数尺度不一致。重排模型用 query-document 交互重新评分，压缩器删除无关句子以节省上下文。全局问题可预先构建实体图和社区摘要，按主题检索摘要而非逐块扫描。每一步记录中间查询和证据，发现改写偏离时回退原查询。

## ③ 企业架构

```text
Query Router(意图/风险)
   ├─ 简单事实 → Hybrid Retriever(BM25+dense) → Reranker
   ├─ 多跳比较 → Query Decomposer → 并行子查询 → Evidence Merger
   ├─ 全局总结 → Graph/Community Retriever → 分层汇总
   └─ 结构化问法 → SQL/Filter Builder → 结果校验
                     ↓
           Context Compressor → LLM → 引用与事实校验
```

检索编排器设置每种路径的预算、超时和最大子查询数；统一证据对象含来源、版本、权限和置信度。SQL/图查询必须参数化并只读白名单。上下文压缩后仍保留原文引用，冲突证据交给规则或人工，而不是让模型自行“平均”。

## ④ 面试标准答案

> 高级 RAG 是按问题类型选择检索策略并增加后处理，而不是无条件堆组件。简单事实走混合检索和重排，复杂问题做查询分解与多跳证据合并，全局主题可用 GraphRAG；父子检索和压缩控制上下文噪声。每条路径都有调用预算、回退和权限过滤，效果用分层评测验证，不能只看最终答案。

## ⑤ 高频追问

1. **什么时候用 HyDE？** 查询描述模糊且语料与问题表述差异大时；生成假设文档有幻觉风险，需与原查询并行并评测。
2. **GraphRAG 一定更准吗？** 对全局和关系问题可能更好，局部事实不一定；构图错误会系统性传播。
3. **如何避免多跳误差累积？** 每跳限定证据类型和深度，校验实体/时间一致性，任何一步失败即可回退。
4. **压缩会不会删掉关键条件？** 用句子级保留规则和关键字段白名单，抽样人工检查后再上线。

## ⑥ 场景题

**场景：供应商合规尽调。** 先识别供应商、法规和时间范围；并行检索法规条款、公司制度和审计记录。RRF 合并后按条款号/生效日期重排，再扩展父段落；冲突时显示各版本证据并要求合规人员确认。涉及“整个供应商群体趋势”的问题切换社区摘要路径，限制最多 20 个社区。

## ⑦ 生产事故题

**事故：上线多查询后延迟和费用翻倍。** Trace 显示每个问题生成 8 个改写，且每条都调用 reranker/LLM。立即将改写数降至 2、对简单意图关闭改写并启用超时回退；随后按意图建立预算和缓存，使用离线集验证 Recall 与 nDCG。验证：p95 延迟、单请求 token 回到 SLO，关键问题 Recall 不低于基线。

## ⑧ 常见错误回答

- “高级 RAG 就是加更多 Agent”——检索质量问题应先用数据和排序解决。
- “HyDE 生成的假文档可以直接当证据”——它仅用于查询向量，答案必须引用真实文档。
- “GraphRAG 能替代向量库”——两者解决的问题不同，通常并行路由。
- “压缩后只保留文本，不需要来源”——失去可审计性和人工核验能力。

## ⑨ Java/Python 实现思路

```python
def advanced_retrieve(question, user):
    intent = classify_intent(question)
    if intent == "global":
        return graph_retrieve(question, user, max_communities=20)
    queries = [question] if intent == "simple" else rewrite(question, limit=2)
    hits = parallel_hybrid_search(queries, user_acl=user.acl, k=50)
    ranked = rerank(question, rrf_merge(hits)[:80])[:12]
    return contextual_compress(question, ranked, keep_citations=True)
```

Java 可用 LangChain4j/LlamaIndex Java 组件实现策略路由；并行任务使用有界线程池，所有子查询带 `parent_trace_id`，超时即取消未完成任务。

## ⑩ 权衡与最佳实践

- 组件越多，潜在召回增益越大但延迟、费用和故障面也越大；从混合检索+重排基线逐步增加。
- 查询改写适合语言噪声，不适合精确编号和 SQL；按意图开关并保留原查询。
- GraphRAG 需稳定实体和关系抽取，先用离线全局问题集证明收益，再投入构图维护。
- 每条证据保留来源和版本，设置“证据不足即拒答”阈值，不能靠更长 Prompt 掩盖检索缺陷。

**核验依据**

- [LangChain：Retrieval 架构](https://docs.langchain.com/oss/python/langchain/retrieval)
- [LangChain：查询变换与多查询](https://www.langchain.com/blog/query-transformations)
- [Milvus：RAG Pipeline 与混合检索](https://milvus.io/docs/rag_pipeline.md)
- [GraphRAG 原始论文：From Local to Global](https://arxiv.org/abs/2404.16130)
