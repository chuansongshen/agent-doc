# 第08章：RAG 数据更新、版本与权限

## ① 核心知识点

- 更新分为新增、修改、删除和权限变更；四类事件都必须驱动索引和缓存失效。
- 文档、chunk、embedding、索引和 Prompt 需可追踪版本；发布使用 immutable snapshot + alias 原子切换。
- 权限以用户/组/租户标签写入元数据，并在检索层强制过滤；前端隐藏不是安全边界。
- 删除要满足“逻辑撤销立即生效、物理清理按合规期限完成”，包括向量、缓存、日志和备份。
- 一致性选择：强一致适合合规问答，最终一致适合低风险知识；对延迟设明确 SLO。

## ② 原理

采用事件驱动 CDC：`changed` 事件携带 `doc_id、version、content_hash、acl_version`。处理流水线生成新派生数据后写入新 collection，校验完成再切换 alias；旧版本保留回滚窗口。查询携带 `tenant/user/groups`，过滤表达式在向量库执行，并对返回结果做二次授权检查。权限变更优先级高于内容更新，缓存键必须含权限版本；撤销事件到达前可用短 TTL 降低暴露窗口。

## ③ 企业架构

```text
源系统/ACL → CDC事件总线 → 版本化Ingestion → 新索引(collection_vN)
       └──────────────→ 权限目录/策略引擎 ───────┘
查询 → 身份认证 → 检索过滤(tenant, groups, acl_version) → 引用校验 → LLM
                              ↑
                        alias/snapshot 管理
```

事件处理需幂等、可重放、有序性保护（按 `doc_id` 分区）和死信队列。设置“最大陈旧时间”指标，超时自动告警或拒答。备份加密，审计记录谁在何时以哪个版本访问了哪条证据。

## ④ 面试标准答案

> 我们把 RAG 当作有生命周期的数据产品：源系统通过 CDC 发出增删改和 ACL 事件，流水线生成带版本的 chunk/embedding，质量通过后原子切换索引别名。查询时按租户、用户组和权限版本强制过滤，删除先逻辑撤销再物理清理，缓存和备份同步失效。根据业务风险选择强一致或有界最终一致，并监控陈旧度、越权拦截和回滚耗时。

## ⑤ 高频追问

1. **如何做到秒级更新？** 事件驱动增量处理、批量 embedding 和小集合实时索引；重建大索引走异步别名切换。
2. **删除向量需要多久？** 逻辑删除和 ACL 撤销应在 SLO 内生效，物理 compaction 按存储策略完成并有证明。
3. **权限放 metadata 会泄露吗？** 仅存不可逆 ID/标签；检索前后都做授权，禁止把原文或敏感字段放公开日志。
4. **旧版本何时保留？** 依合规和回滚窗口决定，设置 TTL 与法律保留例外。

## ⑥ 场景题

**场景：财务制度库。** 制度发布产生 `effective_at`，检索默认取当前生效版本，审计查询可指定历史日期。权限目录变更时先写 ACL 版本并广播撤销事件，缓存键立即失效；索引重建完成后切换 alias。回答显示制度编号、生效日期和版本，冲突时提示人工确认。

## ⑦ 生产事故题

**事故：离职员工仍能检索机密。** 根因为 ACL 事件未消费且缓存未带权限版本。立即禁用账号并撤销相关 token，切换检索到实时策略校验；清理受影响缓存并审计访问记录。修复事件重放、缓存键和“二次授权”中间件，增加离职演练。验证：撤销后 1 分钟内任何接口均返回 403，抽查日志无新证据泄露。

## ⑧ 常见错误回答

- “更新文件后下次查询自然会读到”——没有增量索引和别名切换就会陈旧。
- “只在数据库删记录即可”——向量、缓存、备份和日志仍可能保留副本。
- “ACL 过滤由前端完成”——可被绕过，必须服务端强制。
- “最终一致永远够用”——合规/离职场景需有界陈旧甚至强一致。

## ⑨ Java/Python 实现思路

```python
def on_event(e):
    key = f"{e.doc_id}:{e.version}:{e.acl_version}"
    if dedupe_store.exists(key): return
    if e.type == "deleted":
        vector.delete(expr=f"doc_id == '{e.doc_id}'")
        cache.invalidate_doc(e.doc_id); policy.revoke(e.doc_id)
    else:
        upsert_new_version(e)  # 写 collection_vN，不直接覆盖线上别名
    dedupe_store.put(key)

def search(user, q):
    f = acl_filter(user.tenant, user.groups, policy.version(user))
    hits = vector.search(embed(q), filter=f, k=20)
    return [h for h in hits if policy.allow(user, h.acl)]
```

Java 使用 Kafka 分区保证同文档有序，数据库唯一键保证幂等；策略引擎超时默认拒绝（fail closed），并把版本号传入 Trace。

## ⑩ 权衡与最佳实践

- 原子重建耗存储但回滚简单；在线原地更新省空间却难保证一致，关键库优先 snapshot+alias。
- 强一致增加延迟和成本；将权限撤销强一致、内容更新有界最终一致，按风险分层。
- 物理删除不可逆，先保留加密备份和审计证据，满足法规后再执行并记录证明。
- 每月演练事件重放、索引回滚、权限撤销和灾备恢复，验证实际陈旧度而非只看代码。

**核验依据**

- [PostgreSQL：行级安全策略](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Milvus：多租户与访问控制概览](https://milvus.io/docs/multi_tenancy.md)
- [NIST SP 800-53：安全与隐私控制](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [OWASP：LLM 应用访问控制与数据泄露风险](https://genai.owasp.org/llm-top-10/)
