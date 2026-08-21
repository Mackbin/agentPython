# Agent 学习资料库

> 基于 Autobots 与 agplateform 项目整理，面向前端工程师理解 AI 应用、Agent Runtime 和全栈交付。

这个仓库不是两个项目的源码，而是围绕项目源码整理的学习与面试资料。阅读时请先建立项目全貌，再按需要深入专题，不要从头顺序通读所有文件。

## 从这里开始

| 目标 | 入口 | 用时 |
| --- | --- | --- |
| 我第一次看这个仓库 | [项目总览](docs/autobots/README.md) | 5 分钟 |
| 我要快速理解架构和 Agent 逻辑 | [项目快速拆解](docs/autobots/Agent项目快速拆解与面试知识卡.md) | 15 分钟 |
| 我要准备面试 | [全栈 AI 应用面试作战手册](docs/autobots/Autobots-全栈AI应用面试作战手册.md) | 1-3 天 |
| 我要查后端、RAG 和数据一致性 | [全栈面试技术文档](docs/autobots/autobots-全栈面试技术文档.md) | 按问题查 |
| 我要理解 SSE 后端实现 | [SSE 源码精讲](sse/README.md) | 半天 |
| 我要按计划补 Agent 基础 | [前端转 Agent 学习指南](guide/README.md) | 按章节 |
| 我要执行长期学习计划 | [90 天学习路径](guide/learning-path.md) | 12 周 |

## 推荐阅读顺序

```text
项目总览
  -> 项目快速拆解
  -> Agent 基础概念（LLM / Tool / Memory / RAG）
  -> SSE 源码精讲
  -> 全栈面试作战手册
  -> 全栈技术文档（遇到具体问题再查）
  -> 90 天学习路径
```

## 文档分工

### 项目与面试

- [项目总览](docs/autobots/README.md)：Autobots、agplateform、WebGPT 的关系、模块边界和代码入口。
- [项目快速拆解](docs/autobots/Agent项目快速拆解与面试知识卡.md)：只保留主线，适合第一次理解和面试前复习。
- [面试作战手册](docs/autobots/Autobots-全栈AI应用面试作战手册.md)：完整的面试准备材料，包含事实边界、调用链、STAR、题库和系统设计。
- [全栈技术文档](docs/autobots/autobots-全栈面试技术文档.md)：偏工程参考，深入 Java、Python、RAG、数据库、安全、稳定性和排障。
- [Nacos 配置迁移方案](docs/autobots/nacos-config-migration.md)：独立的配置中心设计方案，不属于 Agent 学习主线。

### Agent 专题

- [前端转 Agent 学习指南](guide/README.md)：从前端视角解释 Agent、SSE、Runtime、MCP、RAG 和多 Agent。
- [SSE 源码精讲](sse/README.md)：围绕 agplateform 的 `sse.py`，分析事件、队列、背压、心跳和取消。
- [90 天学习路径](guide/learning-path.md)：把概念阅读转成每周练习和验证任务。
- [名词速查](guide/glossary.md)：只用于查术语，不作为主线阅读。

## 两个项目先这样理解

```text
agplateform = Agent 平台控制面与运行时
Autobots    = 基于平台能力构建的企业智能客服业务
WebGPT      = Autobots 依赖的网页知识抓取执行面
```

Autobots 的主问答链路偏 RAG：`知识检索 -> Prompt -> 模型生成 -> 流式返回`。
agplateform 的 Runtime 才包含更典型的 Agent Loop：`模型判断 -> 调用工具 -> 观察结果 -> 继续判断`。

## 内容边界

- 文档中的“源码事实”来自指定时间的项目源码快照，版本变化后需要重新核对。
- `主导 / 参与 / AI 辅助 / 掌握 / 建议` 代表不同的个人贡献边界，面试时不要混用。
- 没有可靠统计口径时，不要把 QPS、准确率、成本或性能提升写成确定事实。
- 本仓库只提交学习资料，不应提交公司凭据、内网密码、客户数据或可复用 Token。

## 离线资料

- [前端转 Agent 学习指南 PDF](assets/前端转Agent学习指南.pdf)
- [SSE 流式协议源码精讲 PDF](assets/SSE流式协议源码精讲.pdf)
