# Agent 基础：从前端走向 Agent

> 基于 autobots & agplateform 项目实战
> 面向:前端工程师转 AI Agent 方向
> 本文负责解释 Agent 概念和技术地图；项目具体链路请先看 [Autobots 项目资料](../docs/autobots/README.md)。

## 本文怎么用

这是一份“概念总览”，不是项目源码手册，也不是 90 天计划。建议按下面顺序阅读：

```text
Agent 核心循环 -> LLM / Tool / Memory / RAG
  -> SSE 流式交互 -> Python/Rust Runtime
  -> MCP -> 多 Agent与编排
```

需要逐行理解 SSE 后端时，转到 [SSE 源码精讲](../sse/README.md)；需要安排练习时，转到 [90 天学习路径](learning-path.md)。

---

## 目录

1. [学习目标与前置知识](#1-学习目标与前置知识)
2. [Agent 是什么 — 用前端能懂的话讲](#2-agent-是什么-用前端能懂的话讲)
3. [你已经会的:从 autobots 前端切入](#3-你已经会的从-autobots-前端切入)
4. [Agent 的四大支柱:LLM / Tool / Memory / RAG](#4-agent-的四大支柱llm--tool--memory--rag)
5. [SSE 流式协议 — 你的起点](#5-sse-流式协议-你的起点)
6. [浏览器内 Agent — Page Agent SDK](#6-浏览器内-agent-page-agent-sdk)
7. [语音 Agent — ASR / TTS / VAD 三件套](#7-语音-agentasr--tts--vad-三件套)
8. [Python Agent Runtime — 从 FastAPI 学起](#8-python-agent-runtime-从-fastapi-学起)
9. [Rust Runtime — 新一代高性能 Agent](#9-rust-runtime-新一代高性能-agent)
10. [MCP 协议 — Agent 的"USB 接口"](#10-mcp-协议-agents-的-usb-接口)
11. [RAG 检索增强 — 让 Agent 有知识](#11-rag-检索增强-让-agent-有知识)
12. [多 Agent 协作 — A2A 与 Swarm](#12-多-agent-协作a2a-与-swarm)
13. [编排层 — Go Orchestrator](#13-编排层go-orchestrator)
14. [90 天学习路径](learning-path.md)
15. [你现在就能接手的 Codex 全栈任务](#15-你现在就能接手的-codex-全栈任务)
16. [关键设计模式速查表](#16-关键设计模式速查表)
17. [名词速查](glossary.md)

---

## 1. 学习目标与前置知识

本文不是行业趋势介绍，而是一张从前端进入 Agent 工程的技术地图。读完后，你应该能：

- 解释一次 Agent 请求从用户输入到最终结果的生命周期。
- 区分普通 LLM Chat、RAG Chat 和带工具循环的 Agent。
- 看懂 SSE、工具调用、Memory、RAG 和 Runtime 的基本边界。
- 从前端状态反推后端事件、任务状态和异常处理。

建议前置知识：TypeScript、HTTP、Promise/async、组件状态管理，以及至少一个 Vue 或 React 项目经验。

你在 Autobots 前端中已有的经验，可以这样映射到 Agent：

| 你做过的 | 在 Agent 领域叫什么 | 重要程度 |
|---|---|---|
| SSE 流式聊天 UI | Streaming Chat | ⭐⭐⭐⭐⭐ |
| 语音通话前端 | Realtime Voice Agent | ⭐⭐⭐⭐⭐ |
| AudioWorklet PCM 采集 | Browser Audio Pipeline | ⭐⭐⭐⭐ |
| Page Agent SDK | In-page Agent | ⭐⭐⭐⭐⭐ |
| React 状态管理 | Agent State Machine | ⭐⭐⭐⭐ |

学习策略：先看第 2-5 章建立 Agent 和流式交互的基础，再按兴趣选择浏览器 Agent、语音、Runtime、MCP、RAG 和多 Agent章节。后端代码由 Codex 辅助生成时，重点放在约束、审查、测试和联调，不把生成代码本身等同于掌握。

---

## 2. Agent 是什么 — 用前端能懂的话讲

### 2.1 一句话定义

> Agent = LLM(大脑) + Tool(手) + Memory(记忆) + Loop(循环)

普通聊天机器人是"一问一答",Agent 是"自己想、自己做、做完反馈、再想"的循环系统。

### 2.2 用前端类比

把 Agent 想象成一个"会自己写代码的实习生":

```
你(用户) → 提需求
    ↓
Agent(LLM) → 想思路(thinking)
    ↓
Agent 调工具(tool_call) → 比如读文件、查数据库、调 API
    ↓
工具返回结果(tool_result)
    ↓
Agent 看结果 → 还没做完?继续调工具
    ↓
做完了 → 输出最终结果(output)
```

这就像你写 React 时:
- `useState` = Agent 的 Memory
- `useEffect` 调 API = Agent 的 Tool call
- 组件循环渲染 = Agent 的 Loop

### 2.3 Agent 的核心循环(必背)

```
User Input
    │
    ▼
┌─────────────────────────────────┐
│  LLM 推理 (Stream 输出)          │
│  ┌─────────────────────────────┐ │
│  │ 输出文本 → 给用户看          │ │
│  │ 输出 tool_use → 要调工具     │ │
│  └─────────────────────────────┘ │
└────────────┬────────────────────┘
             │
   ┌─────────┴─────────┐
   ▼                   ▼
有 tool_use         无 tool_use
   │                   │
   ▼                   ▼
执行工具            循环结束
   │              (final output)
   ▼
tool_result 回灌 LLM
   │
   └──→ 回到 LLM 推理
```

这个循环在 agplateform 的 Rust runtime 里叫 `loop_core`,在 Python runtime 里叫 `AgentRunner`,是所有 Agent 框架的核心。

---

## 3. 你已经会的:从 autobots 前端切入

### 3.1 autobots 前端技术栈回顾

```
autobots-frontend (Vue 3 + TS)
├── app-manager    # 主管理端
├── app-admin      # 企业管理
└── app            # C 端聊天
```

agplateform frontend (React 18 + TS):
- `useSSEChat.ts` — SSE 流式聊天
- `useVoiceSession.ts` — 完整语音会话
- `useAsrSession.ts` — 语音识别
- `useTtsPlayback.ts` — 语音合成
- `pageAgent.tsx` — 浏览器内 Agent
- `silero_vad_v5.onnx` — VAD 模型
- `pcm-capture.worklet.js` — 音频采集

### 3.2 你写过的 SSE 聊天,在 Agent 链路里是什么

你前端写的 `useSSEChat`,本质上是在消费 Agent 的流式输出。Agent 后端会吐出这些事件:

```
event: connected     → 连接建立
event: thinking      → Agent 在思考(显示 loading)
event: tool_call     → Agent 调工具(显示工具卡片)
event: output        → 流式文本(逐字显示)
event: completed     → 完成
event: error         → 出错
```

你前端的 rAF 批处理、错误区分、重试,这些**全都是 Agent 前端的核心技能**。后端只是把这些事件用 SSE 推给你。

### 3.3 你能立即做的事

1. **把 autobots 的 SSE 聊天抽成通用组件**,以后任何 Agent 项目都能复用
2. **学习 Agent 事件协议**(看 agplateform `runtime/agentic_runtime/api/sse.py`),理解前端为什么这么写
3. **用 Codex 给你的 SSE 组件加工具调用 UI**(显示 Agent 调了哪些工具、参数、返回值)

---

## 4. Agent 的四大支柱:LLM / Tool / Memory / RAG

### 4.1 LLM(大脑)

**前端类比**:LLM 就是你调的"AI 接口",但 Agent 场景比普通聊天复杂。

项目里用到的 LLM:

| Provider | 用途 | 文件 |
|---|---|---|
| DeepSeek | autobots 主力 | `autobots-ai/ai/llm_loader.py` |
| 通义千问(qwen-flash) | 云上快模型 | 同上 |
| qwen2.5-72b | 自建推理服务 | 由部署环境配置 |
| Anthropic Claude | agplateform Rust runtime | `rust/crates/ap-runtime/src/model/` |
| OpenRouter | 多模型路由 | Nacos shared-llm.yaml |

**OpenAI 兼容协议**(必懂):
所有 LLM 都用同一套 API:`POST /v1/chat/completions`,换 `base-url` 和 `api-key` 就能切模型。

```python
# 伪代码,所有兼容 OpenAI 的模型都这么调
client = ChatOpenAI(
    model="qwen-flash",
    openai_api_base="https://dashscope.aliyuncs.com/compatible-mode/v1",
    openai_api_key="...",
)
```

**学习要点**:
- `temperature=0` → 确定性输出(业务场景)
- `temperature=0.7` → 创意输出(对话场景)
- `streaming=True` → 流式输出(前端 SSE 消费)

### 4.2 Tool(手)

Agent 调工具用的是 **OpenAI function-calling 协议**。前端要理解这个,因为它决定了你怎么渲染工具 UI。

工具定义(JSON Schema 格式):

```json
{
  "type": "function",
  "function": {
    "name": "create_agent",
    "description": "创建一个子 Agent",
    "parameters": {
      "type": "object",
      "properties": {
        "role": {"type": "string", "description": "角色名"},
        "guidance": {"type": "string", "description": "系统指令"}
      },
      "required": ["role"]
    }
  }
}
```

LLM 会输出:

```json
{"name": "create_agent", "arguments": {"role": "researcher"}}
```

你前端就可以渲染成一张"创建 Agent"的卡片,显示参数。

agplateform 的 6 个内置工具(Claude Code 同款):
- `Bash` — 执行命令
- `FileRead` / `Write` / `Edit` — 文件操作
- `Glob` / `Grep` — 搜索

Swarm 多 Agent 协作工具:
- `self` / `list_agents` / `create` / `send` / `create_group` 等

### 4.3 Memory(记忆)

**前端类比**:Memory 就是 Agent 的 `useState` + `localStorage`。

项目里的三层记忆:
- **会话记忆(working)**:当前对话上下文 → 类比 React state
- **长期记忆(long-term)**:跨会话记住用户偏好 → 类比 localStorage
- **无损重述(lossless_restatement)**:原始记忆存储

autobots 用 `ChatMemoryManager` + `SimpleMemoryManager`。
agplateform 用 LanceDB 做向量记忆检索(`memory.search`)。

### 4.4 RAG(知识)

**前端类比**:RAG 就是"给 Agent 配一个搜索引擎"。

```
用户问:"公司报销流程是什么?"
    ↓
1. 把问题转成向量(embedding)
2. 在知识库向量索引里找最相关的文档片段
3. 把找到的片段塞进 LLM 的 prompt
4. LLM 基于这些片段回答
```

autobots 的 RAG 用 **Milvus 混合检索**(dense + sparse),agplateform 用 **LanceDB**。

---

## 5. SSE 流式协议 — 你的起点

### 5.1 为什么 Agent 必须用 SSE

LLM 推理慢(几秒到几十秒),如果用普通 HTTP,用户要等很久才看到结果。SSE 让你"边生成边显示",体验质的飞跃。

### 5.2 SSE 事件协议(项目实战)

agplateform 的 `api/sse.py` 定义了 Agent 执行的事件类型:

```python
# 事件优先级(队列满时丢低优先级)
PRIORITY = {
    "connected": 1,    # 连接(最低)
    "thinking": 2,      # 思考中
    "tool_call": 3,     # 工具调用
    "output": 4,        # 输出文本(最高)
    "completed": 5,     # 完成
    "error": 5,         # 错误
}
```

SSE 文本格式:

```
event: thinking
data: {"agent_id": "xxx"}

event: tool_call
data: {"tool": "FileRead", "args": {"path": "/tmp/a.py"}}

event: output
data: {"delta": "你好"}

event: output
data: {"delta": "，世界"}

event: completed
data: {}
```

### 5.3 前端消费 SSE(你已经会)

```typescript
// 你在 autobots 写过的伪代码
const eventSource = new EventSource('/api/chat/stream');
eventSource.addEventListener('thinking', (e) => setThinking(true));
eventSource.addEventListener('tool_call', (e) => {
  const tool = JSON.parse(e.data);
  setTools(prev => [...prev, tool]);
});
eventSource.addEventListener('output', (e) => {
  const { delta } = JSON.parse(e.data);
  setMessage(prev => prev + delta);  // 逐字拼接
});
```

### 5.4 进阶:为什么用 rAF 批处理

LLM 一秒能吐几十个 chunk,直接 `setState` 会卡。你用 `requestAnimationFrame` 批量 flush,这是 Agent 前端的**标准优化手段**。

> 📖 **深入学习**:完整源码精讲见 [SSE 流式协议源码精讲](../sse/README.md)

### 5.5 学习任务

- [ ] 读懂 `agplateform/runtime/agentic_runtime/api/sse.py` 的队列实现
- [ ] 把 autobots 的 SSE 组件抽象成通用 `<AgentChat />` 组件
- [ ] 加上工具调用卡片 UI(显示工具名、参数、结果、耗时)

---

## 6. 浏览器内 Agent — Page Agent SDK

### 6.1 什么是 Page Agent

agplateform 有个 `page-agent-sdk.js`,让 Agent 能**直接操作网页 DOM**。这是浏览器端 Agent 的关键能力(类似 Anthropic 的 Computer Use,但更轻量)。

### 6.2 工作流程

```
浏览器加载 page-agent-sdk.js
    ↓
HMAC 握手(拿到 session token)
    ↓
建立 SSE 连接到 Rust runtime
    ↓
Agent 可以:
- 读取页面元素
- 点击按钮
- 填表单
- 截图
    ↓
Agent 操作 → 通过 SSE 反馈给 LLM → LLM 决定下一步
```

### 6.3 前端价值

这是你**最该深挖的方向**。浏览器内 Agent 是 2026 年的爆款方向(Browser Use、Anthropic Computer Use、OpenAI Operator 都是)。

### 6.4 学习任务

- [ ] 读懂 `frontend/src/utils/pageAgent.tsx`
- [ ] 理解 HMAC 握手流程(`rust/crates/ap-runtime/src/page_agent/`)
- [ ] 用 Codex 写一个"自动填表单"的 demo

---

## 7. 语音 Agent — ASR / TTS / VAD 三件套

### 7.1 你做过的,在 Agent 领域的术语

| 你做的 | 术语 | 作用 |
|---|---|---|
| `useAsrSession.ts` | ASR (Automatic Speech Recognition) | 语音转文字 |
| `useTtsPlayback.ts` | TTS (Text-to-Speech) | 文字转语音 |
| `pcm-capture.worklet.js` | AudioWorklet PCM Capture | 浏览器音频采集 |
| `silero_vad_v5.onnx` | VAD (Voice Activity Detection) | 检测说话开始/结束 |

### 7.2 完整语音 Agent 闭环

```
用户说话 → 麦克风采集 PCM
    ↓
VAD 检测到说话 → 开始录音
VAD 检测到停顿 → 结束录音
    ↓
ASR 转文字 → 发给 LLM
    ↓
LLM 流式回复 → TTS 流式播放
    ↓
用户听到回答 → 可以打断(barge-in)
```

### 7.3 Silero VAD(值得学)

项目用 `silero_vad_v5.onnx`,这是 PyTorch 训练的 VAD 模型,导出成 ONNX 在浏览器跑(ONNX Runtime Web)。

**为什么不用 Web Speech API**:
- Web Speech API 浏览器兼容性差
- Silero VAD 是真模型,准确率高,能在本地跑
- 配合自建 ASR 服务,延迟可控

### 7.4 学习任务

- [ ] 理解 AudioWorklet(比 ScriptProcessorNode 现代)
- [ ] 跑通 Silero VAD ONNX 在浏览器的推理
- [ ] 实现一个"打断"逻辑(用户说话时停止 TTS 播放)

---

## 8. Python Agent Runtime — 从 FastAPI 学起

### 8.1 为什么前端要懂后端

你现在用 Codex 一把梭,但**不懂后端就调不好 Codex**。你要能:
- 看懂 Codex 生成的代码对不对
- 出 bug 时知道是前端还是后端问题
- 改小 bug 不用每次都让 Codex 重写

### 8.2 Python Runtime 架构(agplateform/runtime/)

```
agentic_runtime/
├── main.py              # FastAPI 入口
├── api/
│   ├── deps.py          # 依赖注入(从 header 取 tenant_id)
│   └── sse.py           # SSE 流式响应
├── model/
│   ├── base.py          # LLM 适配器抽象
│   └── types.py         # ChatMessage / ChatResponse / StreamChunk
├── mcp/
│   ├── manager.py       # MCP 连接管理
│   └── types.py         # MCP 协议类型
├── skill/sdk.py         # Skill SDK
├── swarm/               # 多 Agent 协作
│   ├── bus.py           # 事件总线
│   ├── store.py         # 持久化
│   └── tools.py         # 协作工具
└── task/store.py        # 异步任务
```

### 8.3 LLM 适配器抽象(设计模式:策略 + 模板方法)

```python
class BaseModelAdapter(ABC):
    @abstractmethod
    async def chat_async(self, messages, tools=None) -> ChatResponse: ...
    @abstractmethod
    async def stream(self, messages, tools=None) -> AsyncIterator[StreamChunk]: ...

    def format_tools(self, tools):
        return [tool.to_openai_format() for tool in tools]  # 默认 OpenAI 格式
```

**前端类比**:这就是你写 React 时的 `interface Props`,定义了"所有 LLM 都得有这些方法"。子类(OpenAI/Anthropic/DeepSeek)各自实现。

### 8.4 依赖注入(FastAPI 风格)

```python
# api/deps.py
def get_tenant_id(x_internal_tenant_id: str = Header(...)) -> str:
    if not x_internal_tenant_id:
        raise HTTPException(401)
    return x_internal_tenant_id
```

**前端类比**:类似 React Context,从全局拿 tenant_id。FastAPI 用 `Header()` 自动从请求头提取。

### 8.5 学习任务

- [ ] 读懂 `model/base.py` 的抽象设计
- [ ] 读懂 `api/sse.py` 的事件队列
- [ ] 用 Codex 生成一个最简 Agent:接 OpenAI,流式返回

---

## 9. Rust Runtime — 新一代高性能 Agent

### 9.1 为什么有 Rust Runtime

Python runtime 性能不够(并发低、内存占用大),agplateform 正在迁移到 Rust。Rust runtime 作为 `ap-gateway` 的库内嵌,不单独起进程。

### 9.2 Rust 你需要懂多少

**不需要会写**,但要能"读"。重点术语:

| Rust 概念 | 前端类比 |
|---|---|
| `struct` | TypeScript interface |
| `enum` | TypeScript union type |
| `trait` | TypeScript interface + 实现 |
| `impl` | class 方法 |
| `async/await` | 一样的 |
| `Result<T, E>` | try/catch 的类型化版本 |
| `Option<T>` | `T \| null` |
| `Box<T>` | 堆分配(类似 `new`) |
| `Arc<T>` | 引用计数(共享所有权) |

### 9.3 核心 crate:ap-runtime

```
rust/crates/ap-runtime/src/
├── agent/loop_core.rs       # Agent 主循环(tool_use 多轮)
├── model/
│   └── openai_compat.rs     # OpenAI 兼容(覆盖 5 个 provider)
├── tool/builtin/
│   ├── bash.rs / file_read.rs / write.rs / edit.rs
│   ├── glob.rs / grep.rs    # 6 内置工具
├── mcp/client.rs            # MCP 客户端
├── skill/                   # SKILL.md 解析 + SkillRunner
├── task/                    # Redis 异步任务
├── page_agent/              # HMAC 会话令牌
└── http/                    # /v1/agents/* 路由
```

### 9.4 OpenAI 兼容适配器(一个适配 5 个 provider)

`OpenAiCompat` 实现 `ModelProvider` trait,一个适配器覆盖:
- OpenAI
- DeepSeek
- DashScope(阿里通义)
- Ollama(本地模型)
- OpenRouter

**学习要点**:这就是"适配器模式",换个 base_url 就能用不同模型。

### 9.5 Anthropic 适配器(手写 SSE)

Rust 生态没有成熟的 Anthropic SDK,所以项目用 `eventsource-stream` 自己解析 SSE。这是工程能力的体现。

### 9.6 学习任务

- [ ] 能读懂 `loop_core.rs` 的主循环逻辑
- [ ] 理解 `trait ModelProvider` 的抽象
- [ ] 不需要会写,出 bug 能定位

---

## 10. MCP 协议 — Agent 的"USB 接口"

### 10.1 什么是 MCP

**MCP (Model Context Protocol)** 是 Anthropic 2024 年提出的开放协议,让 Agent 能标准化地调用外部工具/资源/Prompt。

**前端类比**:MCP 之于 Agent,就像 REST API 之于前端。有了标准协议,Agent 调工具不用每个都写适配。

### 10.2 MCP 三类能力

```
Tools    → 可调用的函数(类似 OpenAI function-calling)
Resources → 可读取的数据(文件、数据库记录)
Prompts  → 可复用的提示词模板
```

### 10.3 MCP 协议(JSON-RPC)

```json
// 客户端 → 服务端
{"method": "initialize", "params": {...}}

// 列出工具
{"method": "tools/list"}

// 调用工具
{"method": "tools/call", "params": {"name": "get_weather", "arguments": {"city": "北京"}}}
```

### 10.4 传输方式

| 传输 | 场景 |
|---|---|
| STDIO | 本地进程(比如 Claude Desktop 调本地 MCP) |
| HTTP | 普通远程调用 |
| SSE | 旧版流式(服务端推送) |
| **Streamable HTTP** | 新版,推荐(HTTP + 可选 SSE) |

agplateform 的 Rust runtime 实现了 Streamable HTTP 客户端。

### 10.5 学习任务

- [ ] 读懂 `runtime/agentic_runtime/mcp/manager.py`
- [ ] 读懂 `rust/crates/ap-runtime/src/mcp/client.rs`
- [ ] 用 Codex 写一个最简 MCP server(提供 1 个工具)

---

## 11. RAG 检索增强 — 让 Agent 有知识

### 11.1 RAG 流程

```
文档入库:
  PDF/Word → 切片(chunk) → embedding → 存向量库

查询:
  用户问题 → embedding → 向量库找相似 → 拼 prompt → LLM 答
```

### 11.2 混合检索(autobots 的核心)

autobots 用 **Milvus 2.6 Hybrid Search**,三路融合:

```python
class HybridMilvusRetriever:
    alpha: float = 0.6   # 稠密权重
    beta: float = 0.4     # 稀疏权重
    rerank_strategy: str = "rrf"  # 或 "weighted_sum"

    def _get_relevant_documents(self, query, expr=None):
        q_dense = self.embeddings.embed_query(query)    # 语义向量
        q_sparse = self.sparse_encoder(query)            # 关键词向量
        # Milvus 同时做 dense ANN + sparse + expr 过滤
        res = self.collection.search(
            data=[q_dense],
            sparse_vectors={"sparse_vector": [q_sparse]},
            rerank={"strategy": "rrf", "weight": [alpha, beta]},
            expr=final_expr,  # 元数据过滤(如 tenant_id)
        )
```

### 11.3 两路检索原理

| 类型 | 擅长 | 例子 |
|---|---|---|
| Dense(稠密向量) | 语义匹配 | "开心" ≈ "高兴" |
| Sparse(稀疏向量) | 关键词匹配 | "百融" 专有名词 |

**RRF (Reciprocal Rank Fusion)** 融合公式:

```
score = Σ 1/(k + rank_i)    # k 通常取 60
```

不需要归一化分数,简单有效。

### 11.4 两阶段检索(粗排 + 精排)

```
Query → HybridMilvusRetriever (粗排, 取 top 50)
      → Reranker (精排, 取 top 5)
            ├─ custom: 外部 API,失败回退本地
            ├─ rankllm: 用 LLM 打分
            └─ llm_extractor: LLM 抽关键句
```

### 11.5 LanceDB vs Milvus

agplateform 用 LanceDB 取代 Milvus:
- **嵌入式**:无需独立服务,像 SQLite
- **多视图索引**:向量 + FTS + metadata 一起查
- **适合 Agent 场景**:轻量、快

### 11.6 学习任务

- [ ] 读懂 `autobots-ai/ai/retrievers/HybridMilvusRetriever.py`
- [ ] 理解 RRF 融合
- [ ] 用 Codex 跑一个最简 RAG(3 个文档切片 + 检索)

---

## 12. 多 Agent 协作 — A2A 与 Swarm

### 12.1 为什么要多 Agent

单个 Agent 能力有限,多个 Agent 分工协作:
- Researcher Agent 负责查资料
- Coder Agent 负责写代码
- Reviewer Agent 负责审查
- 它们之间要通信

### 12.2 A2A (Agent-to-Agent) — Go 编排层

agplateform 的 `orchestrator/`(Go)负责 Agent 间通信。

**Agent Card 概念**:每个 Agent 有一张"名片",描述能力、endpoint,其他 Agent 通过名片找到它。

**Resolver 装饰器链**(设计模式:装饰器):

```
Nacos Resolver (主)
    ↓ 包装
Hybrid Resolver (回退到 Java 后端)
    ↓ 包装
Cached Resolver (Redis 缓存)
```

**三种消息路由**:
- `direct` — HTTP 直连(简单)
- `rocketmq` — 消息队列(异步、解耦)
- `hybrid` — 混合(默认 direct,RocketMQ 不可用降级)

### 12.3 Swarm — Python 多 Agent 协作

agplateform 的 `runtime/agentic_runtime/swarm/` 实现"群聊式"多 Agent。

**双层事件总线**(精妙设计):

```python
class AgentEventBus:
    """进程内 per-agent(asyncio.Queue)
    事件:wakeup / unread / stream / done / error
    队列满丢旧事件(背压)"""

class WorkspaceUIBus:
    """跨进程 per-workspace(Redis pub/sub)
    事件推送到前端 SSE
    fire-and-forget,带任务跟踪
    关闭时 asyncio.wait(timeout=5) 优雅退出"""
```

**内置协作工具**(OpenAI function-calling 格式):
- `self` — 获取自己的身份
- `list_agents` — 列出所有 Agent
- `create` — 创建子 Agent(自动建 P2P 群组)
- `send` / `send_group_message` — 点对点/群消息
- `create_group` / `list_groups` / `get_group_messages`

### 12.4 学习任务

- [ ] 读懂 `orchestrator/internal/a2a/manager.go` 的 Resolver 链
- [ ] 读懂 `swarm/bus.py` 的双层事件总线
- [ ] 理解为什么进程内用 Queue、跨进程用 Redis pub/sub

---

## 13. 编排层 — Go Orchestrator

### 13.1 Orchestrator 职责

```
orchestrator/ (Go)
├── a2a/         # Agent 间通信
├── canary/      # 灰度发布
├── sandbox/     # 沙箱隔离
├── workflow/    # 工作流引擎
└── cache/       # Redis 缓存
```

### 13.2 Canary 灰度发布

为 Agent runtime 做渐进上线:
- **百分比灰度**:10% 流量到新版本
- **租户灰度**:指定租户先用
- **Sticky Session**:同一会话粘到同一版本

### 13.3 Sandbox 沙箱

Agent 执行代码要隔离:
- `docker` — 容器隔离
- `gvisor` — Google gVisor(系统调用级,更安全)
- `none` — 不隔离(开发用)

### 13.4 学习任务(可选,优先级低)

- [ ] 了解 Go 的 goroutine、channel(类比 Promise.all)
- [ ] 读 `canary/types.go` 理解灰度策略
- [ ] 不需要会写 Go

---

## 15. 你现在就能接手的 Codex 全栈任务

### 15.1 低难度(前端为主)

1. **Agent 聊天组件库**:把 autobots SSE 抽成通用组件,支持工具卡片、思考过程、错误重试
2. **Page Agent Demo**:用 page-agent-sdk 做一个"自动填表单"demo
3. **语音 Agent Demo**:整合 ASR + LLM + TTS,做一个语音问答

### 15.2 中难度(前后端结合)

1. **MCP 工具市场前端**:展示可用 MCP 工具,支持一键启用
2. **Agent 编排可视化**:用 React Flow 画 Agent 协作图
3. **RAG 知识库管理**:上传文档、查看切片、测试检索

### 15.3 高难度(全栈挑战)

1. **完整 Agent 应用**:用户提问 → RAG 检索 → LLM 回答 → 工具调用 → 多轮对话
2. **多 Agent 协作场景**:Researcher + Writer + Reviewer 协作写文章
3. **浏览器自动化 Agent**:用 Page Agent SDK 做网页操作自动化

### 15.4 用 Codex 的技巧(针对你)

- **先写接口定义**(TS interface / Python type),让 Codex 按接口实现
- **分阶段生成**:先生成骨架,再填充逻辑,避免一次性生成太多
- **用项目已有代码做参考**:跟 Codex 说"参考 `useSSEChat.ts` 的写法"
- **后端不懂的地方**:让 Codex 先解释,再让它改

---

## 16. 关键设计模式速查表

| 模式 | 项目位置 | 前端类比 |
|---|---|---|
| 工厂方法 | `llm_loader.py` 的 `loadmodel_*` | 工厂函数创建组件 |
| 策略模式 | `BaseModelAdapter` | 不同渲染策略 |
| 模板方法 | `BaseModelAdapter.format_messages` | HOC |
| 装饰器 | Hybrid Resolver(Nacos→Hybrid→Cached) | HOC 嵌套 |
| 观察者 | `WorkspaceUIBus` Redis pub/sub | EventTarget |
| 事件总线 | `AgentEventBus` asyncio.Queue | EventEmitter |
| 权限沙箱 | `SkillPermissions` 最小权限 | RBAC |
| 背压 | SSE 队列溢出丢弃 | throttle/debounce |
| 适配器 | `OpenAiCompat` 一适配多 | adapter pattern |

---

## 结语

你已经做了 autobots 前端,SSE / 语音 / Page Agent 这些 Agent 链路里最难的前端部分你都碰过。

你现在要做的不是"转行",而是:
1. **把前端能力深化**(通用 Agent 组件库)
2. **向后端延伸**(用 Codex 帮你写,你读懂+能改)
3. **建立系统视野**(理解 Agent 四大支柱 + 多 Agent 协作)

90 天后,你会成为"能独立用 Codex 全栈交付 Agent 应用"的工程师,这是 2026 年最稀缺的能力。

> "Agent 时代的前端,不是画 UI,而是设计人与 AI 协作的交互。"

---

*本文档基于 autobots & agplateform 项目源码整理*
*本文按项目资料库结构维护，源码快照和内容状态以文档顶部说明为准。*
