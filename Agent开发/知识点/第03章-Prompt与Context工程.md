# 第03章：Prompt Engineering 与 Context Engineering

## ① 核心知识点

- Prompt Engineering 设计指令、示例和输出格式；Context Engineering 设计运行时向模型提供哪些信息、以何顺序、何时更新。
- Prompt 分层：系统规则、开发者约束、任务输入、工具/检索结果、输出 Schema。高优先级规则与不可信内容要明确分隔。
- 结构化输出（JSON Schema/函数参数）比自然语言解析可靠；示例应覆盖正常、边界和拒答。
- 上下文管理包括裁剪、摘要、检索、记忆、工具结果压缩和 token 预算；不是“放得越多越好”。
- Prompt 必须版本化、可回滚、可 A/B，和模型、工具 Schema 一起记录。

## ② 原理

模型根据序列概率生成文本，指令清晰度、相关上下文和输出约束共同决定结果。先定义任务成功指标，再通过少量高质量示例约束格式。上下文组装采用“最小充分原则”：身份与权限先注入，检索片段带来源和时间，用户提供的文本视为数据而非指令。对长对话可按重要性保留最近交互、任务状态和关键事实，旧消息摘要并保留原文引用。每次调用记录 `prompt_version、context_ids、token_usage`，才能复现问题。

## ③ 企业架构

```text
模板仓库(Git/DB) → Prompt Compiler(变量校验、版本) → Context Builder
       ↑                                      ├─ 会话状态/记忆
评测与审批 ← 灰度路由 ← LLM Gateway ←───────┼─ RAG/工具结果
                                             └─ 安全过滤/脱敏
```

Context Builder 负责 token 预算、去重、排序和引用标记；Gateway 负责模型路由、超时、重试与成本；内容安全层检测提示注入、恶意链接和敏感信息。生产禁止把密钥、完整个人信息或未授权文档直接放入上下文。

## ④ 面试标准答案

> Prompt Engineering 关注“如何写指令”，Context Engineering 关注“系统在当前步骤给模型哪些事实”。企业落地需要模板版本、结构化输出、上下文预算和注入防护，并通过离线数据集和线上指标验证。提示词只能影响模型行为，不能替代服务端权限、数据校验和事务控制。

## ⑤ 高频追问

1. **Zero-shot 还是 Few-shot？** 先 zero-shot；格式或边界不稳定时加入 2~5 个代表性示例，并用评测确认收益。
2. **如何控制上下文超长？** 预估 token，按优先级截断；对历史做摘要，对文档做检索和压缩，超限时明确降级或转人工。
3. **如何防提示注入？** 不可信内容加分隔和来源标签；工具执行前做权限与参数校验，禁止模型自行改变系统规则。
4. **温度怎么选？** 分类/工具参数取低温，创意文案可高温；以任务评测而非经验值决定。

## ⑥ 场景题

**场景：合同审查。** 系统规则要求仅依据已授权合同片段，输出风险项 JSON（条款、证据、严重度、建议）。Context Builder 先放用户角色和审查范围，再放按条款检索的片段，每段带 `doc_id/version/page`。缺少证据时输出“无法判断”，不允许猜测；高风险结论进入人工复核队列。

## ⑦ 生产事故题

**事故：更新系统提示词后拒答率暴增。** 通过 `prompt_version` 对比发现新模板把工具说明放在用户文本后且示例要求过严。回滚到上一版本，保留兼容字段；用 200 条回放集检查任务成功率、拒答率、工具参数错误率；审批后分 5%→25%→100% 灰度，并设置自动回滚阈值。

## ⑧ 常见错误回答

- “把所有历史和文档都塞进去最准确”——会造成噪声、超窗和成本上升。
- “系统提示词能阻止一切攻击”——提示注入可绕过文本规则，必须在工具和数据层防护。
- “温度越低越正确”——低温只降低随机性，事实正确取决于上下文和验证。
- “换模型即可解决格式错误”——先用 Schema、重试和输出校验，模型升级需有评测证据。

## ⑨ Java/Python 实现思路

```python
def build_context(state, docs, budget=6000):
    blocks = [f"<policy>{state.policy}</policy>", f"<task>{state.task}</task>"]
    for d in rank_by_priority(docs):
        block = f"<doc id='{d.id}' ver='{d.ver}'>{d.text}</doc>"
        if token_count("\n".join(blocks+[block])) > budget: break
        blocks.append(block)
    return "\n".join(blocks)

response = client.responses.create(
    model=model, input=build_context(state, docs),
    text={"format": {"type": "json_schema", "name": "review", "schema": SCHEMA}}
)
```

Java 使用模板引擎（如 Mustache）和 Jackson `JsonSchema` 校验；上下文构建函数纯化，便于单测 token 预算、敏感字段脱敏和文档去重。

## ⑩ 权衡与最佳实践

- 更详细 Prompt 不一定更好：优先清晰目标、约束、示例和失败处理；每增一段都用评测验证。
- 摘要节省 token 但可能丢失细节；关键事实保留原文引用和可追溯 ID。
- 统一模板便于治理，团队可在局部任务使用专用模板；禁止运行时随意拼接未审查指令。
- 记录输入哈希和版本，避免日志泄露正文；敏感数据使用最小化、加密和保留期限策略。

**核验依据**

- [OpenAI：Prompt Engineering 最佳实践](https://help.openai.com/en/articles/6654000-best-prompt-gineering-with-the-openai-api)
- [OpenAI Agents SDK：Agents](https://openai.github.io/openai-agents-js/guides/agents/)
- [OWASP：LLM 应用提示注入风险](https://genai.owasp.org/llm-top-10/)
- [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)
