# 名词速查

Agent 领域的常用术语,按字母排序。

---

## A

| 术语 | 解释 |
|---|---|
| A2A | Agent-to-Agent,Agent 间通信协议 |
| Agent | 能自主调用工具、循环推理的 AI |
| Agent Card | Agent 的"名片"(描述能力) |
| Artifact | Agent 产出物(如生成的文件) |
| ASR | Automatic Speech Recognition,语音转文字 |

---

## B

| 术语 | 解释 |
|---|---|
| barge-in | 语音打断(用户说话时停止 TTS) |
| Browser Use | 浏览器自动化 Agent(开源项目) |

---

## C

| 术语 | 解释 |
|---|---|
| Canary | 灰度发布 |
| Computer Use | Anthropic 的电脑操作 Agent |

---

## D

| 术语 | 解释 |
|---|---|
| Dense Vector | 稠密向量(语义匹配) |

---

## E

| 术语 | 解释 |
|---|---|
| Embedding | 把文本转成向量(用于相似度计算) |

---

## F

| 术语 | 解释 |
|---|---|
| Function Calling | LLM 调用外部函数的协议(OpenAI 提出) |

---

## H

| 术语 | 解释 |
|---|---|
| HMAC | 哈希消息认证码(Page Agent 握手用) |
| HEARTBEAT | 心跳(保活) |

---

## L

| 术语 | 解释 |
|---|---|
| LanceDB | 嵌入式向量数据库(agplateform 用) |
| LLM | 大语言模型(GPT/Claude/通义) |
| Long-term Memory | 长期记忆(跨会话) |
| Loop | Agent 的"思考-调工具-再思考"循环 |
| Lossless Restatement | 无损重述(原始记忆) |

---

## M

| 术语 | 解释 |
|---|---|
| MCP | Model Context Protocol,Agent 调工具的标准协议 |
| Memory | Agent 的记忆系统 |
| Milvus | 分布式向量数据库(autobots 用) |

---

## O

| 术语 | 解释 |
|---|---|
| OpenAI 兼容 | 所有用 `/v1/chat/completions` 接口的 LLM |
| OUTPUT | SSE 事件:流式输出文本 |

---

## P

| 术语 | 解释 |
|---|---|
| Page Agent | 浏览器内 Agent |
| PCM | Pulse Code Modulation,音频原始格式 |

---

## R

| 术语 | 解释 |
|---|---|
| RAG | Retrieval-Augmented Generation,检索增强生成 |
| Reranker | 精排模型(粗排后二次排序) |
| Resolver | Agent 发现器(A2A 里) |
| RRF | Reciprocal Rank Fusion,排名融合算法 |

---

## S

| 术语 | 解释 |
|---|---|
| Sandbox | 沙箱隔离(执行代码用) |
| Silero VAD | 语音活动检测模型(项目用 onnx 版) |
| Sparse Vector | 稀疏向量(关键词匹配) |
| SSE | Server-Sent Events,服务端流式推送 |
| Sticky Session | 会话粘性(灰度发布用) |
| Streamable HTTP | MCP 新版传输(HTTP + 可选 SSE) |
| Streaming | 流式输出(逐字显示) |
| Swarm | 多 Agent 群聊协作 |

---

## T

| 术语 | 解释 |
|---|---|
| Thinking | SSE 事件:Agent 思考中 |
| Tool | Agent 调用的外部函数 |
| Tool Call | LLM 输出"要调工具"的信号 |
| Tool Result | 工具执行结果,回灌给 LLM |
| TTS | Text-to-Speech,文字转语音 |

---

## V

| 术语 | 解释 |
|---|---|
| Vector Store | 存向量的数据库(Milvus / LanceDB) |
| VAD | Voice Activity Detection,语音活动检测 |

---

## W

| 术语 | 解释 |
|---|---|
| Working Memory | 工作记忆(当前对话) |
| Workflow | 工作流(多步骤编排) |

---

## 数字

| 术语 | 解释 |
|---|---|
| 6 内置工具 | Bash / FileRead / Write / Edit / Glob / Grep |
