# 90 天学习路径(可执行手册)

> 每一项都包含:**读什么 / 怎么练 / 验证标准 / Codex 怎么用**

---

## W1:读懂 autobots SSE 聊天实现

**为什么先学这个**:你做过 autobots 前端,从你写的代码切入,建立"我懂一半"的信心。

### 读什么(按顺序)

1. `autobots-frontend/app-manager/src/api/chat.ts` — 你写过的 SSE 调用
2. `autobots-frontend/app-manager/src/api/chatHistory.ts` — 历史消息
3. agplateform `frontend/src/hooks/useSSEChat.ts` — 对照实现(React 版)
4. agplateform `runtime/agentic_runtime/api/sse.py` — 后端怎么发事件

### 怎么练(3 步)

1. 用纸笔把 autobots SSE 的事件流画出来:`连接 → thinking → output → completed`
2. 对比 agplateform 的 `useSSEChat.ts`,找出两者处理方式的差异(比如重试、错误区分)
3. 读 `api/sse.py`,理解前端收到的每个事件后端是怎么发的

### 验证标准

- 能口述"SSE 从用户发消息到看到回答,中间发生了什么"
- 能说出 `thinking` 和 `output` 事件的区别
- 能解释为什么用 SSE 不用 WebSocket(Agent 是单向流式输出,SSE 更简单)

### Codex 怎么用

- 让 Codex 解释 `api/sse.py` 里"事件优先级队列"那段代码
- 让 Codex 给你写一段"为什么 SSE 队列满了要丢事件"的注释

**时间投入**:7-10 小时(每天 1 小时)

---

## W2:抽象通用 `<AgentChat />` 组件

**为什么学这个**:把已有能力沉淀成可复用组件,是前端转 Agent 的第一步产出。

### 读什么

1. 复读 W1 的文件,这次关注"可抽象点"
2. 看 agplateform `frontend/src/hooks/useSSEChat.ts` 的公开 API 设计(参数、返回值)
3. 参考 shadcn/ui 或 Ant Design 的组件 API 设计风格

### 怎么练(4 步)

1. 用纸设计 `<AgentChat />` 的 Props:

```tsx
<AgentChat
  agentId="xxx"
  sessionId="xxx"
  streamUrl="/v1/agents/xxx/execute"
  onToolCall={(tool) => ...}
  onMessage={(msg) => ...}
/>
```

2. 让 Codex 基于 autobots 现有代码生成骨架
3. 你手动调整 API,确保支持:流式文本、工具调用、思考状态、错误重试
4. 写一个 Storybook 或 demo 页面展示三种状态(loading / streaming / error)

### 验证标准

- 组件能接任意 OpenAI 兼容的 Agent 后端
- 在 autobots 里替换原有聊天 UI,功能不退化
- 组件代码 < 300 行,API 清晰

### Codex 怎么用

- "参考 autobots-frontend 的 chat.ts,帮我生成一个 React AgentChat 组件,Props 是 ..."
- 让 Codex 写测试用例(mock SSE 事件)

**时间投入**:10-14 小时

---

## W3:学 OpenAI function-calling 协议

**为什么学这个**:工具调用是 Agent 的核心,不懂协议就看不懂 Agent 在干什么。

### 读什么

1. agplateform `runtime/agentic_runtime/swarm/tools.py` — 真实工具定义
2. agplateform `runtime/agentic_runtime/model/base.py` 的 `format_tools` 方法
3. OpenAI 官方文档 function-calling(用 WebSearch 找最新版)
4. Anthropic tool_use 文档(对照看差异)

### 怎么练(4 步)

1. 抄一个 `swarm/tools.py` 里的工具定义(比如 `create_agent`),手写一遍
2. 用 curl 调 OpenAI API,看 LLM 返回的 tool_call 是什么格式:

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -d '{"model":"gpt-4o","messages":[...],"tools":[...]}'
```

3. 让 Codex 写一个 mock LLM,返回固定的 tool_call,你前端渲染
4. 实现工具卡片 UI:显示工具名、参数(JSON)、返回值、耗时、状态(pending/success/error)

### 验证标准

- 能手写一个工具定义(不查文档)
- 能解释 tool_call 和 tool_result 的区别
- 工具卡片能正确渲染 3 种状态

### Codex 怎么用

- "帮我写一个 mock server,返回固定的 tool_call 事件"
- "这个工具定义的 JSON Schema 帮我检查对不对"

**时间投入**:10 小时

---

## W4:学 Page Agent SDK

**为什么学这个**:浏览器内 Agent 是 2026 最火方向,你做过前端,这是你的护城河。

### 读什么

1. agplateform `frontend/src/utils/pageAgent.tsx`
2. agplateform `frontend/public/page-agent-sdk.js`(可能是压缩版,先扫一遍)
3. agplateform `frontend/public/page-agent-test.html`(测试页)
4. agplateform `rust/crates/ap-runtime/src/page_agent/`(HMAC 握手,只读逻辑)

### 怎么练(4 步)

1. 跑通 `page-agent-test.html`,看 Agent 能操作页面
2. 抓包看握手流程:浏览器 → 后端,拿到 token 的请求长什么样
3. 让 Codex 写一个最简 demo:页面有个表单,Agent 自动填入数据并提交
4. 加一个"Agent 操作可视化"效果:Agent 点击的元素高亮闪烁

### 验证标准

- 能说清楚 HMAC 握手为什么安全(防 token 被偷)
- demo 跑通:Agent 能读 DOM、点击、填表
- 能解释 Page Agent 和 Browser Use(开源项目)的异同

### Codex 怎么用

- "帮我在 page-agent-test.html 基础上加一个自动填表单的功能"
- "HMAC 握手那段代码帮我画个时序图"

**时间投入**:12-15 小时

---

## W5:学 Python FastAPI + 异步

**为什么学这个**:你要能改后端,Python 是 agplateform runtime 的主语言。

### 读什么

1. agplateform `runtime/agentic_runtime/main.py` — FastAPI 应用入口
2. agplateform `runtime/agentic_runtime/cli.py` — 启动流程
3. agplateform `runtime/agentic_runtime/api/deps.py` — 依赖注入
4. FastAPI 官方教程(中文)的"异步"和"依赖注入"两节

### 怎么练(4 步)

1. 在本地起一个最简 FastAPI:

```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/hello")
async def hello():
    return {"msg": "hi"}
```

2. 加一个 SSE 流式接口:

```python
from fastapi.responses import StreamingResponse
@app.get("/stream")
async def stream():
    async def gen():
        for i in range(5):
            yield f"data: chunk {i}\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```

3. 接 OpenAI,实现"用户提问 → LLM 流式回答"的最简 Agent
4. 用你 W2 写的 `<AgentChat />` 连接这个后端,端到端跑通

### 验证标准

- 本地能 `uvicorn main:app` 起服务
- 能用 `async/await` 写异步函数
- 前端能消费到你后端的 SSE 流

### Codex 怎么用

- "帮我写一个 FastAPI 最简 SSE 接口,接 OpenAI streaming"
- 出错时让 Codex 解释报错原因(异步相关的坑很多)

**时间投入**:12-15 小时

---

## W6:学 LLM 适配器抽象

**为什么学这个**:理解抽象层,以后加任何模型都不怕。

### 读什么

1. agplateform `runtime/agentic_runtime/model/base.py` — 抽象基类
2. agplateform `runtime/agentic_runtime/model/types.py` — 数据类型
3. autobots `autobots-ai/ai/llm_loader.py` — 工厂函数风格对比
4. 复习 Python `abc` 模块和"策略模式"

### 怎么练(4 步)

1. 在纸上画 UML:`BaseModelAdapter` → 子类(OpenAI/DeepSeek/Anthropic)
2. 让 Codex 基于你 W5 的代码,重构出 `BaseModelAdapter`
3. 实现一个 `DeepSeekAdapter` 子类(参考 autobots 的 `loadmodel_deepseek`)
4. 加一个 `config.yaml`,运行时切换不同 provider

### 验证标准

- 能说清楚"策略模式"在这个抽象里怎么用
- 切换 provider 不用改业务代码
- 能解释 `format_tools` 为什么是模板方法(子类可重写但不强制)

### Codex 怎么用

- "把我的 FastAPI 代码重构成 BaseModelAdapter 模式"
- "帮我写 DeepSeekAdapter,参考 autobots-ai/ai/llm_loader.py"

**时间投入**:10-12 小时

---

## W7:学 MCP 协议

**为什么学这个**:MCP 是 Agent 调工具的标准协议,2026 年会爆发。

### 读什么

1. agplateform `runtime/agentic_runtime/mcp/types.py` — 协议类型
2. agplateform `runtime/agentic_runtime/mcp/manager.py` — 连接管理
3. agplateform `rust/crates/ap-runtime/src/mcp/client.rs` — Rust 实现(对照)
4. MCP 官方规范(modelcontextprotocol.io)

### 怎么练(5 步)

1. 用纸画出 MCP 三次握手:`initialize → tools/list → tools/call`
2. 让 Codex 帮你写一个最简 MCP server(Python),提供 1 个工具 `get_time`
3. 用 curl 模拟客户端调你的 server
4. 把这个 MCP server 接到你的 Agent 上,Agent 能调 `get_time`
5. 加一个 Streamable HTTP 传输(不用 STDIO)

### 验证标准

- 能说清楚 MCP 和 OpenAI function-calling 的关系(MCP 是更上层的协议)
- 你的 MCP server 能被标准 MCP client 调用
- 能解释为什么有 STDIO / HTTP / SSE / Streamable HTTP 四种传输

### Codex 怎么用

- "帮我写一个最简 MCP server,Python,提供 get_time 工具,用 Streamable HTTP"
- "我的 server 报错了,帮我看日志"

**时间投入**:12-15 小时

---

## W8:学 RAG 基础

**为什么学这个**:让 Agent 有知识,是 80% 企业场景的需求。

### 读什么

1. autobots `autobots-ai/ai/retrievers/HybridMilvusRetriever.py` — 混合检索
2. autobots `autobots-ai/ai/embedding_loader.py` — 向量化(注意是注释掉的代码,说明在迁移)
3. autobots `autobots-ai/ai/retrievers/RerankRetriever.py` — 两阶段检索
4. autobots `autobots-ai/ai/rerankers/custom_reranker.py` — 精排

### 怎么练(5 步)

1. 在纸上画 RAG 完整流程:入库(切片→embedding→存)+ 查询(query→embedding→检索→拼prompt)
2. 让 Codex 帮你用 LanceDB(轻量,不用起 Milvus)实现最简 RAG
3. 准备 3 个 txt 文档,入库
4. 实现"用户提问 → 检索 → LLM 回答"
5. 加一个 Reranker(用 LLM 打分),对比前后效果

### 验证标准

- 能说清楚 dense 和 sparse 检索的区别
- 能解释 RRF 融合公式 `1/(k+rank)`
- 你的 RAG 能回答 3 个文档里的问题

### Codex 怎么用

- "帮我用 LanceDB + OpenAI embedding 写一个最简 RAG,Python"
- "帮我加一个 LLM Reranker,用 DeepSeek 给文档打分"

**时间投入**:15-18 小时

---

## W9:学 Rust runtime(只读)

**为什么学这个**:agplateform 在迁 Rust,你要能看懂,出 bug 能定位。

### 读什么

1. agplateform `rust/Cargo.toml` — workspace 结构
2. agplateform `rust/crates/ap-runtime/src/agent/loop_core.rs` — Agent 主循环
3. agplateform `rust/crates/ap-runtime/src/model/openai_compat.rs` — LLM 适配
4. agplateform `rust/crates/ap-runtime/src/tool/builtin/grep.rs` — 工具实现

### 怎么练(4 步)

1. 学 Rust 基础语法(只学 4 个概念):`struct` / `enum` / `trait` / `Result<T,E>`
2. 对照 `loop_core.rs`,画出 Agent 主循环流程图
3. 读 `openai_compat.rs`,理解一个适配器怎么覆盖 5 个 provider
4. 读 `grep.rs`,理解工具怎么定义和执行

### 验证标准

- 能看懂 Rust 代码结构(不需要会写)
- 能指出"这是 trait 实现"、"这是 async 函数"、"这是 Result 返回"
- 能用 Rust 术语解释 loop_core 在做什么

### Codex 怎么用

- "这段 Rust 代码帮我逐行解释"
- "loop_core.rs 的 tool_use 循环画个流程图"

**时间投入**:12-15 小时

---

## W10:学多 Agent 协作(Swarm)

**为什么学这个**:多 Agent 是 Agent 的进阶,也是企业场景的真实需求。

### 读什么

1. agplateform `runtime/agentic_runtime/swarm/bus.py` — 双层事件总线
2. agplateform `runtime/agentic_runtime/swarm/store.py` — 持久化
3. agplateform `runtime/agentic_runtime/swarm/tools.py` — 协作工具
4. 复习 W3 学的 function-calling

### 怎么练(5 步)

1. 在纸上画双层事件总线:`AgentEventBus`(进程内 Queue)+ `WorkspaceUIBus`(Redis pub/sub)
2. 让 Codex 跑一个最简 Swarm:2 个 Agent(Researcher + Writer)
3. Researcher 用 `send` 给 Writer 发消息
4. 前端订阅 `WorkspaceUIBus`,看到两个 Agent 的对话
5. 加一个 `create_group`,3 个 Agent 群聊

### 验证标准

- 能解释为什么进程内用 Queue、跨进程用 Redis pub/sub
- 2 个 Agent 能互相发消息
- 前端能看到对话流

### Codex 怎么用

- "帮我基于 agplateform swarm 写一个 2 Agent 协作 demo"
- "为什么用 Redis pub/sub 而不是直接 HTTP 通知?"

**时间投入**:15-18 小时

---

## W11:学 A2A + Orchestrator

**为什么学这个**:理解 Agent 怎么被发现、怎么路由,建立系统视野。

### 读什么

1. agplateform `orchestrator/internal/a2a/manager.go` — A2A Manager
2. agplateform `orchestrator/internal/a2a/resolver/` — Agent 发现
3. agplateform `orchestrator/internal/canary/types.go` — 灰度
4. agplateform `orchestrator/internal/sandbox/types.go` — 沙箱

### 怎么练(3 步,Go 不需要会写)

1. 在纸上画 Resolver 装饰器链:`Nacos → Hybrid → Cached`
2. 理解三种路由(direct / rocketmq / hybrid)的取舍
3. 理解灰度的 4 种策略(百分比 / 租户 / 用户属性 / sticky session)

### 验证标准

- 能说清楚 Agent Card 是什么
- 能解释 Hybrid Resolver 为什么是"装饰器模式"
- 能说清楚灰度发布解决什么问题

### Codex 怎么用

- "这段 Go 代码帮我解释"
- "装饰器模式在 Resolver 链里怎么体现?"

**时间投入**:8-10 小时(只读,不写)

---

## W12:综合项目 — 用 Codex 全栈做一个 Agent 应用

**为什么学这个**:把前 11 周串起来,证明你能独立交付。

### 项目选择(三选一)

#### 项目 A:个人知识助手

- 前端:React + 你 W2 的 `<AgentChat />`
- 后端:FastAPI + 你 W6 的 LLM 适配器
- 知识:你 W8 的 RAG
- 用户上传 PDF → 切片 → 提问 → Agent 回答

#### 项目 B:多 Agent 写作助手

- 基于你 W10 的 Swarm
- 3 个 Agent:Researcher(查资料)+ Writer(写初稿)+ Reviewer(改稿)
- 前端展示协作过程

#### 项目 C:浏览器自动化

- 基于你 W4 的 Page Agent
- 用户说"帮我登录 XX 网站" → Agent 操作

### 怎么做(分 4 天)

1. Day 1-2:让 Codex 生成骨架,你审代码、改 API
2. Day 3:接通前后端,跑通主流程
3. Day 4:打磨 UI、修 bug、写 README

### 验证标准

- 端到端跑通
- 能给同事 demo
- 代码你能讲清楚每段在干什么

### Codex 怎么用

- 分阶段生成,不要一次生成全部
- "先生成后端骨架,前端我之后写"
- 出 bug 先让 Codex 解释,再让它改

**时间投入**:20-25 小时

---

## 每日习惯(贯穿 90 天)

| 习惯 | 时长 | 方法 |
|---|---|---|
| 读源码 | 30 分钟 | 每天专注一个文件,不贪多 |
| 写代码 | 30-60 分钟 | 用 Codex,但每次都要理解它写的是什么 |
| 记笔记 | 10 分钟 | 用 Obsidian 或飞书,记录"今天学了什么" |
| 周复盘 | 30 分钟 | 每周末回顾,调整下周计划 |

---

## 学习节奏建议

- **不要追求完美**:W1 的产出不完美没关系,关键是动起来
- **用 Codex 当老师**:不懂就问,让它解释,不是让它替你学
- **优先读,次要写**:前期多读项目源码,后期再多写
- **建立里程碑**:每完成一周,奖励自己(看个电影、吃顿好的)

---

*本文档基于 autobots & agplateform 项目源码整理*
