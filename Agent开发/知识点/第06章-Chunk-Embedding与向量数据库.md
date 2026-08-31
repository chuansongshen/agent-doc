# 第06章：Chunk、Embedding 与向量数据库

## ① 核心知识点

- Chunk 是检索和引用的最小知识单元；过大降低召回精度，过小丢失语义，通常按结构切分并保留 10%~20% 重叠，再用评测调参。
- Embedding 将文本映射为向量；相似度常用余弦或内积，查询和文档必须使用同一模型、同一预处理和维度。
- 向量库同时存向量、文本引用和元数据；生产需索引、过滤、备份、租户隔离和监控。
- ANN（HNSW/IVF/DiskANN）以少量召回换取速度；小数据可 FLAT 精确搜索。
- 仅向量检索对专有名词和编号不稳，应与 BM25/关键词做混合检索并重排。

## ② 原理

给定查询向量 `q` 和文档向量 `v`，余弦相似度为 `q·v/(||q|| ||v||)`；归一化后内积等价。Chunk 元数据包含 `doc_id、version、section、page、acl、timestamp`，返回结果必须能回源。HNSW 通过分层图近邻搜索，`ef_search` 越大召回高但延迟高；IVF 先聚类再搜部分桶，`nprobe` 控制精度。Embedding 模型更新会改变向量空间，必须新建 collection/索引并离线对比，不能混写。

## ③ 企业架构

```text
Chunker → Embedding Worker(GPU/批量) → Vector DB
   └→ 文本/元数据对象存储             ├─ dense + sparse 字段
                                        ├─ ACL/租户过滤
                                        └─ 备份、压测、监控
```

以 `embedding_model、dimension、chunker_version` 作为索引配置；通过别名指向当前版本，重建完成后原子切换。Milvus 支持 HNSW、IVF、DiskANN、BM25 与混合检索；PostgreSQL+pgvector 适合已有关系数据且规模中等。大规模场景按租户或业务分片，避免单集合热点。

## ④ 面试标准答案

> Chunk 决定语义边界，Embedding 决定相似度空间，向量数据库负责高效近邻搜索和元数据过滤。我们按标题/段落/表格结构切分，保留引用信息并用离线集调 chunk 大小、重叠和 topK；向量版本与模型绑定，采用 HNSW/IVF 等 ANN，在专有名词场景增加 BM25 混合和 rerank。上线前验证召回率、延迟、成本及租户隔离。

## ⑤ 高频追问

1. **固定 512 token 可以吗？** 只能作为基线；条款、表格等结构化内容应结构感知切分。
2. **余弦和内积怎么选？** 向量已归一化时等价；未归一化且需考虑长度时按模型文档选择并保持一致。
3. **topK 越大越好？** 过大增加噪声和生成成本；先高召回取候选，再用 reranker 截断。
4. **向量库能替代数据库吗？** 不能；原文、事务和复杂关系仍由对象存储/关系库负责，向量库保存索引与必要元数据。

## ⑥ 场景题

**场景：研发故障手册。** 标题和步骤按层级切块，代码块不跨 chunk；每块带产品、版本、语言和 ACL。查询先 dense top50 + BM25 top50，RRF 合并后 rerank top8；答案引用具体章节。新 embedding 模型先在 1000 条故障问答上比较 Recall@20、nDCG 和 p95 延迟，再决定切换。

## ⑦ 生产事故题

**事故：模型升级后召回率骤降。** 发现查询向量仍用旧模型而文档已部分重嵌入，导致空间不一致。立即将别名切回旧 collection；补齐模型/维度校验和批次状态，完成全量重嵌入后再灰度。验证：离线 Recall@20 不低于基线 2 个百分点，p95 延迟和成本在预算内。

## ⑧ 常见错误回答

- “chunk 越小越精准”——会丢失上下文和指代关系。
- “向量相似就代表事实正确”——相似度只用于候选，仍需来源、权限和答案校验。
- “换 embedding 无需重建索引”——向量空间变化，必须隔离版本。
- “只看平均延迟”——尾延迟和过滤后召回更影响用户体验，应看 p95/p99 与分租户指标。

## ⑨ Java/Python 实现思路

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("BAAI/bge-m3")
chunks = split_by_heading(text, max_tokens=400, overlap=60)
vectors = model.encode(chunks, normalize_embeddings=True, batch_size=64)
collection.insert([{"id": c.id, "vec": v.tolist(), "text": c.text,
                    "model": "bge-m3", "acl": c.acl} for c, v in zip(chunks, vectors)])
hits = collection.search(query_vec, limit=50,
                         filter="tenant_id == 't1' && acl in ['hr']")
```

Java 可用 Milvus Java SDK 或 pgvector JDBC；统一在网关校验模型名、维度、距离度量，批量写入并记录失败偏移，避免逐条网络调用。

## ⑩ 权衡与最佳实践

- 结构切分质量高但实现复杂；先按标题/段落做基线，再对表格、代码、FAQ 增加专用规则。
- HNSW 召回和延迟易调，内存占用较高；IVF/DiskANN 更省资源但需训练/参数调优。
- dense+BM25+rerank 通常更稳，代价是额外 CPU/模型延迟；仅在离线评测证明确有收益时启用。
- 向量、文本、ACL 三者必须同版本发布；备份恢复演练要覆盖索引重建。

**核验依据**

- [Milvus：系统与索引概览](https://milvus.io/docs/overview.md)
- [Milvus：集合与相似度度量](https://milvus.io/docs/manage-collections.md)
- [pgvector 官方仓库](https://github.com/pgvector/pgvector)
- [OpenAI Embeddings 常见问题](https://help.openai.com/en/articles/6824809-embeddings-frequently-asked-questions)
