# 第13章：MCP（Model Context Protocol）

## ① 核心知识点

- MCP 是基于 JSON-RPC 2.0 的开放协议，用统一生命周期和能力协商连接 LLM 应用与外部数据/工具（本文按 2025-11-25 规范）。
- **Host** 是承载模型和用户交互的应用，负责会话、权限、用户同意和多个连接的编排；**Client** 是 Host 内部与单个 Server 保持会话的连接器；**Server** 提供工具和上下文能力。
- Server 能力：`tools`（模型可调用函数）、`resources`（可读取上下文/数据）、`prompts`（可复用提示模板）；Client 可声明 sampling、roots、elicitation 等能力。
- 传输：本地 `stdio` 适合受控进程；远程首选 Streamable HTTP（POST JSON-RPC，可返回流），连接先 `initialize`，再能力协商。
- MCP 规定协议消息，不替应用完成业务授权；HTTP 授权按规范使用 OAuth 2.1/Protected Resource Metadata，stdio 凭据由本地环境安全注入。

## ② 原理

Host 创建 Client，Client 与 Server 完成 `initialize`/`initialized` 握手，交换协议版本和 capabilities。随后 Client 通过 `tools/list`、`resources/list`、`prompts/list` 发现能力，按需调用 `tools/call`、`resources/read`、`prompts/get`。Tools 是模型控制的动作，但 Host 应在执行前展示并取得用户同意；Resources 通常由应用决定何时读取，Prompts 通常由用户选择模板。Server 不能假设自己能看到 Host 的完整系统提示词；sampling 是可选的反向请求，必须经 Host 同意。请求携带 JSON-RPC id，错误要返回标准错误并支持取消/进度通知。

## ③ 企业架构

```text
Host（Chat/IDE/Agent Runtime）
  ├─ MCP Client A ── stdio ── 本地文件/脚本 Server
  ├─ MCP Client B ── HTTPS Streamable HTTP ── 企业 MCP Gateway
  │                                      ├─ OAuth/租户策略
  │                                      ├─ 工具注册、审计、限流
  │                                      └─ ERP/CRM/知识库 Server
  └─ Consent UI / Policy Engine / Trace
```

MCP Gateway 对 Server 做目录、签名和版本管理；每个 Client 会话绑定租户、用户和最小权限 token。Server 进程隔离并限制文件系统 roots、网络出口和资源 URI；工具副作用走审批/幂等服务。Host 记录 `server_id、tool_name、request_id、consent、latency`，敏感结果脱敏后再进模型上下文。

## ④ 面试标准答案

> MCP 是连接 Host、Client、Server 的协议层，不是 Agent 编排框架。Host 管用户和安全，Client 为每个 Server 维护 JSON-RPC 会话，Server 暴露 tools、resources、prompts。能力通过初始化协商，传输可用 stdio 或 Streamable HTTP。MCP 本身不替业务授权：远程 HTTP 按规范接 OAuth 和资源元数据，本地凭据由宿主注入；工具调用仍需用户同意、服务端鉴权、参数校验和审计。

## ⑤ 高频追问

1. **Host 和 Client 为什么分开？** Host 统一用户体验和策略，Client 隔离每个 Server 的会话、能力和故障，便于多 Server 组合。
2. **Resource 与 Tool 区别？** Resource 是可读取的上下文数据，Tool 是可执行函数；前者通常无副作用，后者可能改变外部状态。
3. **Prompt 是系统提示词吗？** 不是。MCP Prompt 是 Server 提供的可发现模板，Host 可展示/修改，最终系统规则仍由 Host 控制。
4. **MCP 授权是否强制？** 协议要求实现安全同意；HTTP 授权规范定义 OAuth 流程，stdio 等本地场景由宿主自行实施凭据与沙箱策略。

## ⑥ 场景题

**场景：IDE 助手接入代码仓库与工单系统。** Host 为每个项目建立 Client 会话；文件 Server 仅暴露用户批准的 roots，工单 Server 通过 Streamable HTTP+OAuth 访问。只读搜索工具可自动调用，创建工单工具弹出确认并要求幂等键。返回的资源 URI 和版本写入引用，跨 Server 的数据不会默认互相可见。

## ⑦ 生产事故题

**事故：恶意 Server 的工具描述诱导模型上传源码。** 处置：立即撤销 Server 注册和 token，阻断网络出口并审计调用；恢复前对工具描述和返回内容做签名/人工审核，默认关闭高风险工具。Host 将工具注释视为不可信，调用前显示参数和目标，Gateway 做 DLP 与域名白名单。验证：恶意描述测试集 100% 被拦截，未经确认的写工具调用为零。

## ⑧ 常见错误回答

- “MCP Server 就是插件，拿到连接即可访问所有数据”——权限、roots 和租户范围必须显式绑定。
- “工具描述可信，模型会按说明执行”——描述和返回内容都可能被投毒，需策略和人工确认。
- “SSE 是唯一 MCP 传输”——当前规范推荐 Streamable HTTP；stdio 仍适合本地进程。
- “MCP 自带完整业务授权”——协议提供 HTTP 授权规范，资源级权限仍由应用和 Server 强制。

## ⑨ Java/Python 实现思路

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

params = StdioServerParameters(command="python", args=["server.py"], env=SAFE_ENV)
async with stdio_client(params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
        if not policy.allow_tool(user, "ticket.create"):
            raise PermissionError("tool denied")
        result = await session.call_tool("ticket.create", {"title": title})
```

Java 可使用官方 MCP Java SDK，`HttpClientStreamableHttpTransport` 连接远程 Server；在调用前接入 OAuth token、JSON Schema 校验、审批和超时。Server 端不要直接执行 Shell，必须沙箱、白名单和资源级鉴权。

## ⑩ 权衡与最佳实践

- MCP 降低集成重复开发，但动态发现扩大供应链和提示注入风险；企业应使用私有注册表、签名和版本锁定。
- stdio 部署简单、隔离好但难跨主机；Streamable HTTP 便于集中治理，需 TLS、OAuth、会话和限流。
- 自动批准只读工具可提升体验；写操作、外发数据和高权限资源坚持显式同意与审计。
- 采用协议版本和 capability 白名单，未知字段/方法默认拒绝；升级先在兼容性测试环境验证。

**核验依据**

- [MCP 规范 2025-11-25 基础协议总览](https://modelcontextprotocol.io/specification/2025-11-25/basic)
- [MCP Server Tools 规范](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [MCP 传输规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- [MCP 授权规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)
