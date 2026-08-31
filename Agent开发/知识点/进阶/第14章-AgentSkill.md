# 第14章：Agent Skill

## ① 核心知识点

- Skill 是可复用的“做事方法包”：通常为目录内的 `SKILL.md` 指令，加可选脚本、参考资料和资产；重点是过程知识和约束。
- Tool 是一次能力调用（函数/API），Workflow 是开发者预先编排的步骤，Skill 则指导 Agent 在合适时机组合工具和步骤；Skill 本身不天然拥有新权限。
- `SKILL.md` 至少包含 YAML frontmatter 的 `name`、`description`，正文写触发条件、输入、流程、输出和检查；渐进披露只在需要时加载详细资源。
- 触发依据是任务与 description 匹配或用户显式调用；名称、描述和指令都应明确边界，避免“万能技能”。
- 版本用 SemVer/Git tag 管理，声明兼容的工具、模型和资源版本；升级可回滚，破坏性变更提高主版本。

## ② 原理

Agent 先扫描技能元数据，匹配后读取完整 `SKILL.md`，再按指令引用脚本/参考文件，最后调用已有工具完成任务。Skill 通过上下文注入改变决策流程，不等于代码执行权限；脚本仍在宿主沙箱和审批策略下运行。多个 Skill 组合时按优先级叠加：安全/系统规则 > 领域 Skill > 任务参数；冲突必须停止并请求澄清。进阶技能可把一个 Skill 的输出作为下一个的结构化输入，但要限制循环深度和资源预算。

## ③ 企业架构

```text
Skill Registry（签名、版本、负责人、SBOM）
          ↓ 元数据索引
Host/Agent Runtime → Trigger Matcher → Context Loader
          ↓                    ├─ SKILL.md 指令
          ↓                    ├─ references/按需加载
          ↓                    └─ scripts/沙箱执行
          → Tool Gateway（权限、审批、审计）→ 企业系统
```

Registry 做来源审核、恶意指令扫描、依赖锁定和撤销；Runtime 记录 `skill_id/version`。每个 Skill 声明输入输出 Schema、所需工具和数据范围，默认只读；脚本使用临时目录、网络白名单和 CPU/时间配额。组合 Skill 通过显式 DAG 或事件传递，禁止隐式共享 secrets。

## ④ 面试标准答案

> Skill 是可版本化的流程与领域知识包，核心文件是带元数据的 `SKILL.md`，可附脚本和参考资料，按任务匹配后渐进加载。Tool 提供单次 API 能力，Workflow 固定执行路径，Skill 负责指导 Agent 何时、如何组合它们。Skill 不自动获得权限，脚本和工具仍受宿主沙箱、审批、最小权限和审计约束。企业应做签名注册、SemVer 兼容、灰度和回滚。

## ⑤ 高频追问

1. **Skill 和 Prompt 模板有什么不同？** Skill 包含触发元数据、流程、资源和验证，能跨任务复用；模板通常只是一段输入文本。
2. **Skill 能调用不存在的工具吗？** 不能。它只能指导宿主使用已授权工具；缺少依赖应明确失败而不是模拟结果。
3. **如何避免 Skill 过长？** 元数据保持短，正文给核心流程，详细规则拆到 references 按需读取，脚本处理确定性逻辑。
4. **组合冲突怎么办？** 预先声明优先级和输出 Schema；发生权限或安全冲突时以宿主策略为准并停止执行。

## ⑥ 场景题

**场景：财务月结 Skill。** description 仅匹配“月结/对账”任务；流程依次读取账单、执行校验脚本、生成差异报告、提交人工审批。Skill 声明需要 `ledger.read` 和 `report.write`，不包含付款权限；脚本版本锁定并在沙箱运行。升级字段时发布 v2，旧任务继续使用 v1，报告中记录 Skill 与工具版本。

## ⑦ 生产事故题

**事故：第三方 Skill 脚本窃取环境变量。** 立即撤销 Skill 版本和注册表签名，隔离运行节点并轮换可能泄露的密钥；审计脚本调用和外发流量。修复为无默认 secrets、临时凭据、静态扫描、SBOM、网络白名单和人工审批；灰度前运行恶意样本集。验证：脚本无法读取未声明变量，外联域名仅允许白名单，撤销版本在 5 分钟内全局生效。

## ⑧ 常见错误回答

- “Skill 就是更长的 Prompt”——忽略了版本、资源、脚本、触发和治理。
- “安装 Skill 等于授予它所有工具权限”——权限由宿主/网关决定，默认拒绝。
- “把所有参考资料一次性注入最可靠”——会浪费上下文并增加冲突，应渐进加载。
- “改 Skill 内容不需要发版本”——不可复现、难回滚，必须锁定版本和依赖。

## ⑨ Java/Python 实现思路

```python
from pathlib import Path
import yaml

def load_skill(path, task):
    text = Path(path, "SKILL.md").read_text()
    meta, body = parse_frontmatter(text)
    version = meta.get("version", "0.0.0")
    if not matches(meta["description"], task):
        return None
    if not registry.verify_signature(path, meta["name"], version):
        raise PermissionError("untrusted skill")
    return {"meta": {**meta, "version": version}, "instructions": body,
            "refs": lambda p: safe_read(path, "references/" + p)}
```

Java 可用 SnakeYAML 解析 frontmatter、`ServiceLoader`/数据库索引技能元数据；脚本交给容器或进程沙箱，执行前由 Tool Gateway 校验声明的工具与权限。日志记录 `skill_id/version/trigger/result`，不记录秘密和完整用户数据。

## ⑩ 权衡与最佳实践

- Skill 提升复用和一致性，但会增加上下文与供应链风险；采用短元数据、渐进加载和私有注册表。
- 固定 Workflow 可预测性高；Skill 适合跨流程复用判断和操作规范，关键事务仍由 Workflow/代码锁定。
- 自动触发方便但可能误触；高风险 Skill 要求显式调用、审批和可见的执行计划。
- 版本、依赖、权限和评测结果一起发布；保留上一稳定版，出现回归可按任务继续运行旧版本。

**核验依据**

- [Agent Skills 开放规范与目录结构](https://github.com/agentskills/agentskills)
- [OpenAI Academy：Skills 与 SKILL.md](https://openai.com/academy/skills/)
- [Microsoft Agent Framework：Agent Skills](https://learn.microsoft.com/en-us/agent-framework/agents/skills)
- [OpenAI Skills 官方仓库](https://github.com/openai/skills)
