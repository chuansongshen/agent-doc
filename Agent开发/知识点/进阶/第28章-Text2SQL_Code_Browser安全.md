# 第28章：Text2SQL / Code / Browser Agent 安全

## ① 核心知识点

- Text2SQL：模型只生成候选查询；服务端解析 AST，执行表/列/函数白名单、租户条件、只读事务、参数化、超时、行数和成本上限。
- Text2SQL 禁止 DDL/DML、`COPY`/文件函数、跨租户表和多语句；使用 `EXPLAIN` 预估成本，超阈值转人工或要求缩小范围。
- Coding Agent：代码和命令在一次性沙箱中运行；非 root、最小目录挂载、默认断网/域名白名单、CPU/内存/时间/进程上限，凭证不进环境。
- 命令控制：解析为结构化动作，允许列表优先；禁止 shell 拼接、危险递归删除、提权和未经审批的安装/发布。
- Browser Agent：域名/协议白名单、独立 BrowserContext、Cookie/会话隔离、下载隔离与扫描；支付、转账、删除、提交表单必须人工确认。
- 三类 Agent 都要记录计划、实际动作、权限判断和结果；高风险动作支持 kill switch、撤销令牌和回放。

## ② 原理

- **Text2SQL 信任边界：** 自然语言和数据库元数据可能被提示注入；SQL 文本不是权限证明，最终权限由数据库角色和策略强制。
- AST 校验比正则可靠：识别语句类型、表/列引用、函数、子查询和注释；再用参数绑定处理值，禁止模型直接拼接标识符。
- 只读角色降低破坏面，但仍可能拖垮数据库；`statement_timeout`、`lock_timeout`、资源组和并发配额同样必要。
- Coding Agent 将“代码执行”视为不可信程序：沙箱提供文件、网络、进程、系统调用和资源隔离；工作区使用临时副本，结果以补丁返回。
- Browser Agent 将网页文本、DOM、下载文件都视为不可信输入；导航和动作由策略层决定，模型不能修改白名单或绕过审批。
- 防护采用纵深：身份/权限→策略→沙箱/浏览器隔离→输入输出校验→人工确认→审计与评测。

## ③ 企业架构

```text
用户 → Agent API/身份层 → 风险策略引擎
      ├─ Text2SQL Gateway → AST 校验 → 只读 DB 角色/资源组 → 脱敏结果
      ├─ Code Runner → 临时沙箱（文件/网络/资源）→ 补丁+测试报告
      └─ Browser Runner → 域名代理/独立 Context → 下载扫描/审批 → 页面结果
      └─ 审计、OTel trace、安全评测与全局 kill switch
```

- 策略按租户、数据分类和动作风险分级；高风险动作返回“待审批计划”，审批通过才签发一次性能力令牌。
- Text2SQL 元数据服务只暴露授权表/列描述；数据库启用行级安全（RLS）和只读副本，结果按字段脱敏。
- Code Runner 使用临时容器/微 VM，挂载最小工作区；网络默认拒绝，依赖通过内部镜像代理缓存。
- Browser Runner 代理强制 HTTPS 和域名白名单，下载文件落入隔离目录并做类型/恶意内容扫描，禁止自动上传外部站点。
- 所有执行关联 `trace_id、tenant_id、policy_version、tool_version`，保留输入摘要和动作证据，敏感内容脱敏。

## ④ 面试标准答案

> Text2SQL 不是把自然语言直接拼成 SQL，而是模型提出候选，服务端用 AST 做语句、表列和函数白名单校验，参数化执行，绑定只读角色、租户条件、超时、行数和成本上限。Coding Agent 放入临时非 root 沙箱，命令结构化、网络默认拒绝、凭证隔离，高风险发布需审批。Browser Agent 使用域名白名单和独立会话，下载隔离扫描，支付/删除/提交必须人工确认。三者统一由策略引擎、审计、评测和 kill switch 控制。

## ⑤ 高频追问

- **只读账号还安全吗？** 不能写不等于不能泄露或拖垮数据库，仍需 RLS、脱敏、超时、成本和结果大小限制。
- **为什么不用正则过滤 `DROP`？** 注释、编码、嵌套和方言会绕过正则，应使用成熟解析器和 AST。
- **Coding Agent 能否直接执行 `npm install`？** 仅在沙箱和内部镜像中按包白名单执行，限制脚本、网络和磁盘。
- **Browser Agent 为什么要独立 Context？** 避免复用用户 Cookie、localStorage 和登录态，降低跨站和跨任务泄露。
- **能否自动点击“提交订单”？** 只读浏览可自动；支付、转账、删除、最终提交需展示摘要并人工确认。

## ⑥ 场景题

**场景：** 财务人员问“本季度各部门报销总额”，Text2SQL 需支持自然语言追问。

**解法：** 仅暴露授权视图和字段说明；模型生成 `SELECT` 后解析 AST，注入租户/部门过滤并绑定参数；先 `EXPLAIN`，超时或扫描量过大要求增加筛选；只读事务执行，结果脱敏并记录 SQL 指纹和审计证据。

## ⑦ 生产事故题

**事故：** Coding Agent 运行测试脚本读取了宿主机云凭证并外传。

**处置：** 立即撤销/轮换凭证、断开沙箱网络、冻结任务并保全 trace；确认挂载和环境变量泄露路径。修复为临时凭证、空环境、只读最小工作区、出站域名白名单和 DLP；用同类脚本做逃逸与外传回归。

**事故：** Browser Agent 在仿冒站点自动提交了删除操作。

**处置：** 停止浏览动作、撤销会话并联系业务恢复；将域名信誉、导航链和页面指令纳入审计。上线域名固定白名单、危险动作二次确认和删除 API 幂等/回收站，灰度验证后恢复。

## ⑧ 常见错误回答

- “给模型数据库管理员权限，查询最方便。”——违反最小权限且无法控制数据范围。
- “过滤掉 DROP/DELETE 字符串即可。”——正则不是 SQL 解析器，容易被绕过。
- “容器隔离了，所以代码 Agent 可以访问宿主机目录。”——挂载和内核漏洞仍可能造成逃逸。
- “浏览器登录态复用能提升体验。”——会把 Cookie、支付和个人数据带入不可信任务。
- “只要让模型说‘我确认过’就能自动付款。”——确认必须由可信用户和服务端策略完成。

## ⑨ Java/Python 实现思路

Python（AST 与资源约束骨架）：

```python
def execute_sql(sql, tenant_id):
    ast = parse_sql(sql)  # 使用成熟解析器，禁止正则替代
    if ast.kind != "SELECT" or not tables_allowed(ast) or has_forbidden_fn(ast):
        raise ValueError("query denied")
    sql2, params = bind_tenant_filter(ast, tenant_id)
    with readonly_conn(statement_timeout_ms=3000) as c:
        plan = c.explain(sql2, params)
        if plan.total_cost > 1e6:
            raise ValueError("query too expensive")
        return c.fetch(sql2, params, max_rows=1000)
```

Java：JSqlParser/Apache Calcite 解析 AST，jOOQ 参数化生成 SQL；数据库使用独立只读角色、RLS 和 `SET LOCAL statement_timeout`。Code Runner 通过 gVisor/微 VM 或受限容器执行；Browser 采用 Playwright `BrowserContext`，代理层拦截导航与下载并把危险动作交给审批服务。

## ⑩ 权衡与最佳实践

- AST 白名单覆盖越广，维护成本越高；优先提供授权视图和有限查询模板，复杂分析转人工/离线任务。
- 只读副本降低写风险但仍可能被慢查询拖垮；按租户并发、成本和扫描量限额，必要时异步化。
- 强沙箱隔离效果好但启动慢；低风险 lint 可用轻量容器，高风险执行使用微 VM 并设置短时生命周期。
- 浏览器自动化节省人工却难验证页面意图；所有不可逆动作展示目标、金额、收件人和域名后再确认。
- 安全回归集覆盖注入、AST 绕过、沙箱逃逸、域名欺骗、下载/外传和重复提交；策略与工具版本变化即触发评测。

**核验依据**

- [PostgreSQL GRANT 权限文档](https://www.postgresql.org/docs/current/sql-grant.html)
- [PostgreSQL EXPLAIN 文档](https://www.postgresql.org/docs/current/sql-explain.html)
- [OWASP Secure Coding with AI Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secure_Coding_with_AI_Cheat_Sheet.html)
- [Playwright BrowserContext 隔离](https://playwright.dev/docs/browser-contexts)
