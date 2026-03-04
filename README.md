## Agent 开发学习总结

这是一个**面向实战的 Agent / LLM 开发学习笔记仓库**，记录了从基础概念到可上线 Agent 的完整知识路径。内容以工程视角组织，适合已经在用大模型或准备在业务中落地 Agent 的同学系统梳理认知。

> 推荐从 `01_LLM_RAG_Agent_Study_Guide.md` 开始，它是一份总纲/导航，其余篇章是各个关键模块的详细展开。

---

## 文档结构总览

- **01_LLM_RAG_Agent_Study_Guide.md**  
  整体路线图：从「理解 LLM」到「可上线 Agent」的分阶段学习计划，串联 02–09 作为子章节。

- **02_Tokenizer_BPE_Detailed.md**  
  Token / Tokenizer / BPE 细节：文本如何被拆成 token、为什么要用子词、这些设计如何影响长度上限与费用。

- **03_Embedding_Detailed.md**  
  Embedding 详解：向量空间直觉、相似度度量，以及在 RAG 中如何用向量表达文档与查询。

- **04_Self_Attention_Detailed.md**  
  Self-Attention 机制：Q/K/V、注意力矩阵、mask、缩放因子等细节，帮助你真正搞懂「注意力在算什么」。

- **05_Transformer_Engineering_View.md**  
  Transformer 工程视角：从算子和数据流角度拆解一轮前向计算，结合算力/显存/吞吐做工程取舍。

- **06_Temperature_TopP_Detailed.md**  
  采样与解码策略：Temperature、Top‑P、Top‑K 等参数如何共同决定生成文本的随机性与稳定性。

- **07_Chat_API_Internal_Detailed.md**  
  Chat API 内部：消息结构、特殊 token、system / user / assistant 角色、Function Calling / JSON 输出等。

- **08_One_Question_Input_To_Output_Flow.md**  
  「一个问题」从输入到最终回答的完整调用链，包含工具调用（tool_calls）、多轮决策和错误处理。

- **09_How_To_Understand_RAG.md**  
  RAG 全景：从 chunk 切分、向量检索、重排（re-ranking），到如何评估「检索质量 vs 生成质量」。

---

## 如何使用本仓库

- **按阶段系统学习**  
  从 `01` 开始，按照文中推荐顺序阅读 02–09，并结合自己的项目做小实验（如最小 RAG Demo、带工具的最小 Agent）。

- **按主题查表/复习**  
  当你在做工程实践时遇到问题（例如「RAG 检索质量太差」「模型输出 JSON 不稳定」），可以直接跳到对应篇章查阅细节。

- **配合代码与 SDK**  
  建议在阅读过程中同时打开你常用的大模型 SDK（如 OpenAI / DeepSeek / Anthropic 等），边看文档边写小脚本，形成「概念—实践」闭环。

---

## 适用人群

- 已经在使用大模型 API，希望**从「能用」升级到「理解原理与工程 trade-off」**的开发者；
- 准备在业务中落地 RAG / Agent 系统，需要一条**从零到可上线**的学习路线；
- 想把零散的博客/文章知识整理成一套**结构化认知**的同学。

后续如果有新的专题（如多 Agent 编排、LangGraph 实战等），会继续以同样的风格补充到本仓库中。

# AgentLearning