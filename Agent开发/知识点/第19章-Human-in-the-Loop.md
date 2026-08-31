# 第19章：Human-in-the-Loop（HITL）

## ① 核心知识点

- **定位**：在高风险、不确定或不可逆动作前暂停，让具备权限的人审批、修改、补充或接管。
- **触发条件**：金额/权限/数据敏感度阈值、模型置信度低、策略冲突、连续失败、用户明确要求。
- **审批对象**：展示目标、工具名、完整参数、影响范围、证据、风险、过期时间和可选动作（批准/修改/拒绝）。
- **身份与权限**：审批人必须认证、具备相应角色；禁止发起人审批自己产生的高风险请求。
- **状态机**：`PENDING_APPROVAL → APPROVED/REJECTED/EXPIRED/CANCELED`，审批决定不可静默覆盖。
- **恢复**：暂停时保存 checkpoint 和 `approval_id`；恢复使用原 run_id，不能重新生成计划绕过审批。
- **超时**：审批请求设置 SLA；到期默认拒绝或转人工队列，不默认放行。
- **重试**：通知投递失败可按指数退避重试，始终复用同一 `approval_id`；审批决定和实际副作用不可因通知重试重复执行。
- **审计**：记录谁在何时基于何证据做了什么决定；敏感字段按权限展示和脱敏。

## ② 原理

```text
Agent 生成动作 → 策略评估 → 需要人工？
  否：执行并记录
  是：持久化状态/审批单 → 等待
      → 批准/修改/拒绝/超时 → 校验决定 → 继续或终止
```

人工输入是不可信外部输入，恢复时需再次做 Schema、权限和业务状态校验。若工具参数被修改，必须重新计算风险和幂等键。

## ③ 企业架构

- **Policy Engine**：规则、阈值、职责分离和审批路由，策略版本化。
- **Approval Service**：生成审批单、通知、SLA、升级和防重复提交。
- **Agent Runtime**：在 interrupt 点写 checkpoint，监听审批事件后恢复。
- **审计与证据库**：保存输入快照、工具参数、模型版本、审批意见和执行结果。
- **安全控制**：短期签名链接、重放保护、租户隔离、最小字段展示。

## ④ 面试标准答案

> HITL 不是让人“随时盯着模型”，而是在可定义的风险边界设置可恢复的审批闸门。系统先持久化待审批动作和证据，再由有权限的人批准、修改或拒绝；恢复时复用同一 run_id 并重新校验参数和权限。审批有超时和默认策略，所有决定可审计，高风险默认拒绝而不是默认放行。

## ⑤ 高频追问

**Q：人工修改参数后怎么办？** 重新做 Schema、权限、风险和幂等键校验，记录原值与修改差异。

**Q：审批人不在线？** 按 SLA 升级或转人工队列；到期拒绝/取消，不能无限占用资源。

**Q：如何避免审批疲劳？** 低风险自动放行，高风险聚合摘要、证据和差异；策略持续用误报/漏报数据调优。

## ⑥ 场景题

**场景：Agent 代发对外公函**

草稿生成后先做敏感词和收件人校验；涉及外部客户或法律承诺时必须由法务角色审批。审批人可编辑正文但不能更换收件人权限范围；发送工具使用邮件幂等键，超时先查发送状态。

## ⑦ 生产事故题

**事故：审批链接被转发，未授权人员批准高额付款。**

- 根因：链接本身代表权限，没有二次身份认证和审批人绑定。
- 处置：立即撤销未执行审批、冻结付款、审计访问日志并通知安全团队。
- 修复：登录态 + MFA + 角色校验；审批令牌短时一次性、绑定 `approval_id` 与用户；执行前再次鉴权。

## ⑧ 常见错误回答

- “加一个确认按钮就算 HITL。”——没有身份、权限、证据和审计不满足企业要求。
- “人工批准后直接执行，不用再校验。”——状态可能变化，必须执行前复核。
- “审批超时默认通过避免阻塞。”——高风险场景应默认拒绝或升级。
- “人工输入可以直接拼到 Prompt。”——需结构化、校验并防提示注入。

## ⑨ Java/Python 实现思路

```python
def approval_gate(action, actor):
    if not policy_requires_approval(action, actor):
        return action
    approval_id = approvals.create(action, actor, ttl=900)
    checkpoint({"status": "PENDING_APPROVAL", "approval_id": approval_id})
    decision = wait_for_approval(approval_id)
    if decision.status != "APPROVED":
        raise RuntimeError(f"审批未通过: {decision.status}")
    validate_again(action, decision.edited_args, actor)
    return action.with_args(decision.edited_args)
```

Java 可用事件总线传递 `ApprovalDecided`，消费者按 `approval_id` 做幂等；等待不占用线程，恢复由持久化运行时触发。

## ⑩ 权衡与最佳实践

- 审批点少而关键，避免把低风险任务全部人工化。
- 默认拒绝、最小权限、职责分离；执行前再次检查资源状态。
- 展示“将要发生什么”及证据，不展示无法验证的模型置信度数字。
- 设定审批 SLA、升级和取消；审批决定不可修改，只能追加更正记录。
- 用真实审批日志评估通过率、误拒率、平均等待和事故数。

**核验依据**

- [LangGraph Interrupts（HITL）官方文档](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/breakpoints/)
- [LangGraph 工具调用人工审核](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/review-tool-calls/)
- [OWASP AISVS：AI 安全验证标准](https://owasp.org/www-project-artificial-intelligence-security-verification-standard-aisvs-docs/)
