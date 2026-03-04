# Agent 开发学习路线图

这是一条从 0 到「能写出可上线 Agent」的学习/实践路线，用来统筹你后面那几篇详细笔记（02–09）。你可以按阶段顺序学习，也可以按需跳到对应阶段查表。

------------------------------------------------------------------------

## 阶段 0：打底——理解你在驱动什么引擎

**阶段目标：**知道 LLM 本质是什么、输入输出长什么样、算力/成本受什么影响。

- **需要掌握的概念**
  - LLM 本质：\(P(\text{next token} \mid \text{context})\)，是概率语言模型，不是数据库/推理机。
  - Token / Tokenizer / BPE：文本→token→id，为什么要用子词（见 `02_Tokenizer_BPE_Detailed.md`）。
  - Embedding：id→向量，句向量在 RAG 里的作用（见 `03_Embedding_Detailed.md`）。
  - Self-Attention 与 Transformer 数据流：从 token 向量到「下一个 token 概率」（见 `04_Self_Attention_Detailed.md` + `05_Transformer_Engineering_View.md`）。
  - Temperature / Top‑P 等采样参数：如何控制输出随机性（见 `06_Temperature_TopP_Detailed.md`）。

- **推荐阅读顺序**
  - 02 → 03 → 04 → 05 → 06

- **建议练习**
  - 画出一张「文本 → token → embedding → Transformer → 概率 → 下一个 token」的数据流图。
  - 用任意一个 SDK/工具，大致估算几句不同长度中文/英文的 token 数，感受「字数 vs token」的区别。

------------------------------------------------------------------------

## 阶段 1：从「一次调用」到「可控调用」

**阶段目标：**能稳定地用 Chat API 调 LLM，控制角色、格式和随机性。

- **需要掌握的概念**
  - Chat API 消息结构：`system` / `user` / `assistant` 各自的职责。
  - Chat API 如何封装到底层 prompt：特殊 token（如 `<|system|>` 等）的作用（见 `07_Chat_API_Internal_Detailed.md`）。
  - Function Calling / JSON 输出：如何通过 schema 约束模型输出结构。
  - 采样参数（Temperature / Top‑P）的组合对生成文本风格和稳定性的影响（结合 `06_Temperature_TopP_Detailed.md`）。

- **推荐阅读顺序**
  - 07（理解 Chat API）+ 06（回顾采样控制）

- **建议练习**
  - 用官方 SDK 写一个小脚本：
    - 输入一段 user 文本 → 打印 assistant 回复。
    - 改变 system prompt 和温度，观察风格与随机性的变化。
  - 写一个「强约束 JSON 输出」的例子：让模型返回固定字段的 JSON，遇到不合法 JSON 时做简单重试。

------------------------------------------------------------------------

## 阶段 2：RAG——给模型外挂“知识库大脑”

**阶段目标：**能够基于你自己的文档/代码做问答，而不是只依赖模型参数里的旧知识。

- **需要掌握的概念**
  - RAG 流程：文档 → chunk → Embedding → 向量库 → 检索 → 拼进 prompt → 生成（见 `09_How_To_Understand_RAG.md`）。
  - Embedding 模型选型与语义相似度：向量空间、余弦/内积、同一模型用于文档与查询（见 `03_Embedding_Detailed.md`）。
  - 检索细化：
    - chunk 大小与 overlap；
    - Top‑K 的选择；
    - Hybrid 检索（BM25 + 向量）；
    - Re-ranking（先粗后精）；
    - 如何评价检索质量 vs 生成质量（见 `09_How_To_Understand_RAG.md` 第 7 节）。

- **推荐阅读顺序**
  - 03（Embedding 基础）→ 09（RAG 流程 + 检索细化）

- **建议练习**
  - 选一部分你自己的文档（比如工作笔记），做一个小 RAG Demo：
    - 离线：切 chunk → 做 embedding → 写入一个简单向量库（哪怕是内存里的 list + faiss）。
    - 在线：输入问题 → 编码 → Top‑K 检索 → 把片段拼进 prompt → 调用 LLM 回答。
  - 尝试不同 K（如 3、5、8），查看哪些设置下「正确片段」更常出现在检索结果中。

------------------------------------------------------------------------

## 阶段 3：最小 Agent——工具 + 循环

**阶段目标：**从「一次调用」升级为「会调用工具、会多步推理」的 Agent。

- **需要掌握的概念**
  - 普通 LLM 调用：问 → 答（单轮）。
  - Agent 调用：Thought → Action → Observation → Thought…（循环决策）。
  - 带工具的完整调用链：用户提问 → Chat API 输出 `tool_calls` → 后端执行工具 → 结果回填 → 再次调用模型 → 最终答复（见 `08_One_Question_Input_To_Output_Flow.md`）。
  - 工具定义与执行：工具 schema（name/description/parameters）由你/编排层定义，LLM 只负责根据 schema 生成 tool_call，真正的调用在模型外。
  - Agent 工程核心：任务拆解、工具调用与异常处理（死循环、超时、JSON 解析失败等）。

- **推荐阅读顺序**
  - 08（完整调用链示例）→ 回看 07（Function Calling 部分）→ 结合 01 当前这篇路线图整体理解。

- **建议练习**
  - 自己用 SDK 写一个「最小工具 Agent」：
    - 定义一个 `get_weather` 或 `calculator` 工具的 schema；
    - 第一次调用 Chat API，让模型决定是否调用工具以及参数；
    - 在本地真的执行工具（可以是假函数返回固定数据）；
    - 把工具结果作为 `tool` 消息塞回 messages，再调一次模型生成最终自然语言回复；
    - 加上最大步数/超时和简单错误处理，避免死循环。

------------------------------------------------------------------------

## 阶段 4：工程化 Agent——从 Demo 到可上线

**阶段目标：**让 Agent 不只是能跑一两个例子，而是可以维护、可以监控、可以集成到业务里。

- **需要掌握的概念**
  - 工程分层结构：应用层 → Agent 调度层 → RAG 层 → LLM 接口层（原文第 5 部分）。
  - 日志与可观测：
    - 每轮 prompt、tool_call、工具输入输出、usage（token 数、时延等）的记录。
  - 稳定性与保护：
    - JSON/结构化输出失败的重试策略；
    - 工具报错、网络异常、超时的处理；
    - 限制最大步数、防止 Agent 死循环。
  - 框架选型与使用（按需）：
    - 当你已经理解上面这些概念后，再去看 LangChain / LangGraph / LlamaIndex 等框架，只需要学会「如何用它们实现你已经理解的调用链」，而不是被各种术语带着走。

- **推荐阅读顺序**
  - 当前这篇路线图（01）作为总纲 → 结合 08 看调用链 → 结合自己的项目/工具实践。

- **建议练习**
  - 在「最小 Agent」基础上增加：
    - 简单日志记录（例如把每轮 messages + tool 调用写入一个本地 log 文件）；
    - 引入一个真实 API（例如公开的天气或汇率接口）；
    - 为工具/模型调用增加重试和兜底回复逻辑。

------------------------------------------------------------------------

## 路线图与各文档的对应关系

- **01（本篇）**：Agent 开发整体路线图与阶段目标。
- **02**：Token 与 Tokenizer（文本→id）的底层认知。
- **03**：Embedding（id/文本→向量），为 RAG 和模型内部表示打基础。
- **04**：Self-Attention 机制细节。
- **05**：Transformer 工程级数据流与算力分析。
- **06**：Temperature / Top‑P 等采样控制。
- **07**：Chat API 内部结构与 Function Calling。
- **08**：一个问题从输入到输出的完整调用链（带工具的 Agent）。
- **09**：RAG 的目标、流程与检索策略细化。

按这个路线走完，你会从「理解 LLM 怎么算」走到「会写能调工具、接知识库、可上线的 Agent」。之后再看更复杂的多 Agent 编排、LangGraph 等框架，就有清晰坐标了。

