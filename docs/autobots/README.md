# Autobots 项目资料

这里集中放 Autobots、agplateform 和 WebGPT 相关的项目分析。三者不是同一个仓库，但共同组成了“企业客服业务 + Agent 平台 + 网页知识执行”的完整链路。

## 先看什么

```text
第一次看项目：项目快速拆解
准备面试：面试作战手册
查实现细节：全栈面试技术文档
查配置设计：Nacos 配置迁移方案
```

## 系统关系

| 系统 | 架构角色 | 负责什么 |
| --- | --- | --- |
| Autobots | 业务应用层 | 用户、智能客服、知识库、RAG、会话、线索和运营 |
| agplateform | 平台控制面 / Agent Runtime | Agent、模型、工具、租户、配额、执行和多 Agent 编排 |
| WebGPT | 网页抓取执行面 | URL 校验、任务调度、网页抓取、正文清洗和文档推送 |

## 核心链路

### 知识摄入

```text
前端导入文件/URL
 -> Gateway 鉴权
 -> Java external 编排
 -> WebGPT 异步抓取
 -> 文档解析和切块
 -> Embedding + Milvus
 -> task_id 关联知识范围
 -> 知识生效
```

### 在线问答

```text
前端问题
 -> Gateway / Python QA Route
 -> 服务端计算可访问知识
 -> 向量检索
 -> Prompt + LLM
 -> HTTP 流式返回
 -> 会话、线索和运营后置任务
```

### Agent 执行

```text
用户目标
 -> Runtime 调用模型
 -> 模型选择 Tool / MCP
 -> 执行并返回 tool_result
 -> 模型继续判断
 -> 完成或失败
```

## 文件怎么分工

| 文件 | 用途 | 不适合做什么 |
| --- | --- | --- |
| [项目快速拆解](Agent项目快速拆解与面试知识卡.md) | 15 分钟建立全局认知 | 不适合查所有实现细节 |
| [全栈 AI 应用面试作战手册](Autobots-全栈AI应用面试作战手册.md) | 面试主资料，按准备顺序组织 | 不适合当源码 API 手册 |
| [全栈面试技术文档](autobots-全栈面试技术文档.md) | 后端、RAG、数据、安全、排障参考 | 不建议线性通读 |
| [Nacos 配置迁移方案](nacos-config-migration.md) | 配置中心专项设计 | 与 Agent 基础学习无关 |

## 常用代码入口

| 能力 | 入口 |
| --- | --- |
| 前端聊天 | `autobots-frontend/app/src/components/Chat/` |
| 管理端知识库 | `autobots-frontend/app-manager/src/views/Knowledge/` |
| URL 导入 | `app-manager/src/views/Knowledge/Content/Add/UrlUploadModal.vue` |
| Java 知识适配 | `autobots-external/.../KnowledgeController.java`、`KnowledgeServiceImpl.java` |
| Python 问答 | `autobots-ai/server/qa_route.py`、`services/qa_service.py` |
| 文档摄入 | `autobots-ai/server/document_route.py`、`ingest_route.py` |
| RAG 能力 | `autobots-ai/ai/` 下 Retriever、Reranker、LLM、Memory |
| Agent Runtime | `agplateform/runtime/agentic_runtime/`、`agplateform/rust/crates/ap-runtime/` |
| SSE 后端 | `agplateform/runtime/agentic_runtime/api/sse.py` |

## 资料边界

AP 和 WebGPT 是关联项目，不等同于 Autobots 个人代码产出。面试时应区分：自己主导的前端和构建工作、参与的跨服务联调、通过源码掌握的系统设计，以及尚未落地的演进方案。

