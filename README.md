# Agent 学习指南

> 基于 autobots & agplateform 项目实战
> 面向:前端工程师转 AI Agent 方向

---

## 📚 目录

本站点包含两份学习资料:

### 1. [前端转 Agent 学习指南](guide/README.md)

完整的 90 天学习路径,从你做过的 autobots 前端切入,逐步深入到后端 Runtime、Rust、MCP、多 Agent 协作。

- Agent 四大支柱:LLM / Tool / Memory / RAG
- SSE 流式协议(你做过的一半)
- 浏览器内 Agent(Page Agent SDK)
- 语音 Agent(ASR / TTS / VAD)
- Python / Rust Runtime
- MCP 协议
- RAG 检索增强
- 多 Agent 协作(A2A / Swarm)
- 90 天分周可执行学习手册

### 2. [SSE 流式协议源码精讲](sse/README.md)

agplateform `runtime/agentic_runtime/api/sse.py` 逐行分析,共 20 章:

- SSEEventType 事件类型枚举
- SSEEvent 事件对象
- SSEStream 流管理器
- 事件优先级设计
- 背压丢弃策略(精妙)
- OUTPUT_DISCARD 撤回机制
- 异步迭代器(最难)
- 与 FastAPI 的集成
- 与前端的对应关系
- 自己实现最简版本

---

## � PDF 离线版

如果你想要 PDF 版本离线看或打印,可以下载:

- [前端转 Agent 学习指南.pdf](assets/前端转Agent学习指南.pdf)(945 KB)
- [SSE 流式协议源码精讲.pdf](assets/SSE流式协议源码精讲.pdf)(675 KB)

> PDF 内容与在线版一致,在线版会持续更新

---

## �🚀 快速开始

| 我想... | 看这里 |
|---|---|
| 建立整体认知 | [前端转 Agent 学习指南](guide/README.md) |
| 深入 SSE 细节 | [SSE 流式协议源码精讲](sse/README.md) |
| 看学习路径 | [90 天学习路径](guide/learning-path.md) |
| 查名词解释 | [名词速查](guide/glossary.md) |

---

## 🎯 你是谁

如果你是:

- **做过 autobots 前端的工程师**:你已经会一半了,从 [第 3 章](guide/README.md#3-你已经会的从-autobots-前端切入) 开始
- **纯前端工程师**:从头开始,先看 [第 2 章 Agent 是什么](guide/README.md#2-agent-是什么-用前端能懂的话讲)
- **后端工程师**:跳到 [第 8 章 Python Runtime](guide/README.md#8-python-agent-runtime-从-fastapi-学起)

---

## 📝 项目背景

### autobots

业务 AI 服务(智能客服 / 线索 / 语音通话):
- Python 3.10+ / FastAPI
- LangChain + Milvus + DeepSeek/通义
- Vue 3 + TypeScript 前端

### agplateform

企业级 Agent 协作平台:
- Java 网关(Spring Cloud Gateway)
- Python Runtime(legacy)+ Rust Runtime(新一代)
- Go Orchestrator(A2A 多智能体编排)
- React 18 + TypeScript 前端

---

*生成时间:2026-08-20*
