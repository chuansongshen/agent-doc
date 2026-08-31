# 企业级 Agent 开发知识库

一份系统化的 **企业级 Agent 开发** 学习资料库，涵盖从大模型接入、RAG、工具调用到 Agent 架构、生产排障与系统设计面试的全链路知识点。

## 📚 目录结构

```
知识库/
├── README.md                                   # 本文件（总览与索引）
├── Agent从入门到精通（附实战）.md               # 入门长文，从概念到实战演练
└── Agent开发/
    └── 知识点/
        ├── 第一优先级目录.md                    # 第一优先级学习顺序（面试核心）
        ├── 第01章~第32章 *.md                   # 各章节知识点
        ├── 企业级Agent与LLM应用基础_第1章与第3章.md  # 合并版文档
        └── 进阶/
            ├── 进阶目录.md                      # 第二优先级学习顺序（进阶）
            └── 第04章~第28章 *.md               # 进阶章节
```

## 📖 快速入口

| 入口 | 内容 | 适用人群 |
|------|------|---------|
| [`Agent从入门到精通`](./Agent从入门到精通（附实战：论文整理Agent搭建）.md) | 万字长文，从"大模型与 Agent 区别"讲起，附论文整理 Agent 实战 | 初学者、想建立整体认知 |
| [`第一优先级目录`](./Agent开发/知识点/第一优先级目录.md) | 21 章核心知识点（面试高频） | 面试准备、核心能力建设 |
| [`进阶目录`](./Agent开发/知识点/进阶/进阶目录.md) | 10 章进阶知识点（Model Gateway、MCP、Multi-Agent 等） | 进阶学习、架构设计 |

## 🗂️ 知识点目录

### 第一优先级（面试核心）

1. [第 1 章：企业级 Agent 到底是什么](./Agent开发/知识点/第01章-企业级Agent到底是什么.md)
2. [第 3 章：Prompt 与 Context 工程](./Agent开发/知识点/第03章-Prompt与Context工程.md)
3. [第 5 章：企业知识库数据处理](./Agent开发/知识点/第05章-企业知识库数据处理.md)
4. [第 6 章：Chunk、Embedding 与向量数据库](./Agent开发/知识点/第06章-Chunk-Embedding与向量数据库.md)
5. [第 7 章：检索系统](./Agent开发/知识点/第07章-检索系统.md)
6. [第 8 章：RAG 数据更新、版本与权限](./Agent开发/知识点/第08章-RAG数据更新版本与权限.md)
7. [第 10 章：企业级 RAG 评测](./Agent开发/知识点/第10章-企业级RAG评测.md)
8. [第 11 章：Function Calling / Tool Calling](./Agent开发/知识点/第11章-Function与ToolCalling.md)
9. [第 12 章：企业级工具执行可靠性](./Agent开发/知识点/第12章-企业级工具执行可靠性.md)
10. [第 15 章：Agent Planning 与执行模式](./Agent开发/知识点/第15章-AgentPlanning与执行模式.md)
11. [第 16 章：Agent Loop、终止与长任务](./Agent开发/知识点/第16章-AgentLoop终止与长任务.md)
12. [第 17 章：State、Checkpoint 与 Durable Execution](./Agent开发/知识点/第17章-StateCheckpoint与DurableExecution.md)
13. [第 19 章：Human-in-the-Loop（HITL）](./Agent开发/知识点/第19章-Human-in-the-Loop.md)
14. [第 21 章：Agent 系统整体架构](./Agent开发/知识点/第21章-Agent系统整体架构.md)
15. [第 22 章：流式输出与长任务](./Agent开发/知识点/第22章-流式输出与长任务.md)
16. [第 23 章：高并发、高可用与弹性](./Agent开发/知识点/第23章-高并发高可用与弹性.md)
17. [第 27 章：Agent Security](./Agent开发/知识点/第27章-Agent安全.md)
18. [第 29 章：Agent Evaluation](./Agent开发/知识点/第29章-Agent评测.md)
19. [第 30 章：Observability 与线上排障](./Agent开发/知识点/第30章-可观测性与线上排障.md)
20. [第 31 章：企业级 Agent 系统设计题](./Agent开发/知识点/第31章-企业级Agent系统设计题.md)
21. [第 32 章：真实生产事故题](./Agent开发/知识点/第32章-真实生产事故题.md)

### 第二优先级（进阶）

1. [第 4 章：大模型接入与 Model Gateway](./Agent开发/知识点/进阶/第04章-大模型接入与ModelGateway.md)
2. [第 9 章：高级 RAG](./Agent开发/知识点/进阶/第09章-高级RAG.md)
3. [第 13 章：MCP](./Agent开发/知识点/进阶/第13章-MCP.md)
4. [第 14 章：Agent Skill](./Agent开发/知识点/进阶/第14章-AgentSkill.md)
5. [第 18 章：Memory 与 Context 管理](./Agent开发/知识点/进阶/第18章-Memory与Context管理.md)
6. [第 20 章：Multi-Agent 与 A2A](./Agent开发/知识点/进阶/第20章-MultiAgent与A2A.md)
7. [第 24 章：缓存、Token 与成本治理](./Agent开发/知识点/进阶/第24章-缓存Token与成本治理.md)
8. [第 25 章：部署与发布](./Agent开发/知识点/进阶/第25章-部署与发布.md)
9. [第 26 章：推理性能优化](./Agent开发/知识点/进阶/第26章-推理性能优化.md)
10. [第 28 章：Text2SQL / Code / Browser Agent 安全](./Agent开发/知识点/进阶/第28章-Text2SQL_Code_Browser安全.md)

## 📐 文档结构说明

每个知识点章节均采用统一的 **10 小节** 结构：

1. 核心知识点
2. 原理
3. 企业架构
4. 面试标准答案
5. 高频追问
6. 场景题
7. 生产事故题
8. 常见错误回答
9. Java / Python 实现思路
10. 权衡与最佳实践

> 说明：文档中的"核验依据"优先采用官方文档、开放标准和原始论文；涉及业务规则、权限和高风险操作时，以企业自身制度与确定性校验为准。