# Transformer 工程级理解

从「数据怎么走、算在哪、调什么」看 Transformer，便于实现、调参和排查问题。

------------------------------------------------------------------------

## 一、工程视角：Transformer 是什么

**一句话**：按层堆叠的、输入输出形状不变的块；每层做「Attention + 前馈」，把「token 序列 + 位置」变成每个位置的上下文向量，最后用 LM Head 映射成词表概率。

**工程上关心**：
- 数据流：每层张量 shape 如何变化
- 算力/显存：谁最吃算力、谁占显存、与序列长度/批大小的关系
- 可调参数：层数、头数、维度、上下文长度、batch

------------------------------------------------------------------------

## 二、整体数据流（一次 forward 在做什么）

```
输入: token_ids [batch, seq_len]
  ↓
Embedding: [batch, seq_len] → [batch, seq_len, hidden_dim]
  ↓
+ 位置编码 (或 RoPE / ALiBi 等)
  ↓
重复 N 层:
  ├─ Multi-Head Attention (含残差 + LayerNorm)
  └─ Feed-Forward (含残差 + LayerNorm)
  ↓
输出仍是 [batch, seq_len, hidden_dim]
  ↓
取最后一个位置 或 做 mean pooling 等 → [batch, hidden_dim]
  ↓
LM Head (线性层): [batch, hidden_dim] → [batch, vocab_size]
  ↓
Softmax → 下一个 token 的概率分布
```

**要点**：中间所有层输入输出都是 `[batch, seq_len, hidden_dim]`，只有最后一步才出现词表维；自回归生成时每次多一个 token，KV 变长，故有 KV Cache 优化。

------------------------------------------------------------------------

## 三、按模块拆解（与代码/实现一一对应）

### 1. Embedding 层

| 工程理解     | 说明 |
| ------------ | ---- |
| 作用         | token ID → 向量，shape 从 `[B, L]` → `[B, L, D]` |
| 参数量       | `vocab_size × hidden_dim`，常占模型很大一块 |
| 实现         | `nn.Embedding(vocab_size, hidden_dim)` |

### 2. 位置编码（Position Encoding）

| 工程理解     | 说明 |
| ------------ | ---- |
| 作用         | 让模型知道「第几个位置」，否则 Attention 对位置不敏感 |
| 形态         | 学习式（如 BERT）、固定式（正弦）、RoPE（LLaMA 等）、ALiBi |
| 工程         | 有的在 embedding 后加，有的在 Attention 内乘（RoPE），影响实现与显存 |

### 3. Transformer Block（单层）

每层 = Attention → 残差 + Norm → FFN → 残差 + Norm：

```
x = x + Attention(LayerNorm(x))   # 子层 1
x = x + FFN(LayerNorm(x))        # 子层 2
```

| 子模块             | 输入/输出 shape   | 工程关注点     |
| ------------------ | ----------------- | -------------- |
| LayerNorm          | [B, L, D] → [B, L, D] | 数值/训练稳定  |
| Multi-Head Attention | [B, L, D] → [B, L, D] | 见下           |
| FFN                | [B, L, D] → [B, L, D] | 常为 D→4D→D，参数量大 |

### 4. Multi-Head Attention（对应 01_Self_Attention）

公式：Q=XWq, K=XWk, V=XWv，Attention = softmax(QK^T / √d_k) V。

| 步骤     | Shape 变化                          | 说明 |
| -------- | ----------------------------------- | ---- |
| 线性变换 | [B,L,D] → Q,K,V 各 [B,L,D]          | 可按头拆成 [B,H,L,D/H] |
| QK^T     | [B,H,L,D_k] × [B,H,D_k,L] → [B,H,L,L] | L×L 最吃显存与算力 |
| Softmax × V | [B,H,L,L] × [B,H,L,D_v] → [B,H,L,D_v] | |
| 多头拼回 | [B,H,L,D_v] → [B,L,D]               | 可再经线性层 |

**结论**：计算与显存都与 **L²** 相关，上下文变长成本快速上升；头数 H 只分掉 D 的维度，不改变 L² 代价。

### 5. Feed-Forward Network（FFN）

| 工程理解     | 说明 |
| ------------ | ---- |
| 形式         | 两线性层 + 中间激活：Linear(D→4D) → 激活 → Linear(4D→D) |
| Shape        | [B, L, D] → [B, L, 4D] → [B, L, D] |
| 作用         | Attention 做完「信息混合」后，逐位置非线性变换；参数量常大于 Attention |

### 6. 最后一层到「下一个 token」

| 工程理解     | 说明 |
| ------------ | ---- |
| 取哪个位置   | 因果 LM 通常只用**最后一个位置**的向量 [B, D] 预测下一个 token |
| LM Head      | 线性层 [B, D] → [B, vocab_size]，有时与 Embedding 共享权重（tie weights） |
| 输出         | Softmax 得 P(next_token \| context)，再采样或 argmax |

------------------------------------------------------------------------

## 四、自回归生成与 KV Cache（工程必知）

推理时逐 token 生成：

```
第 1 步: 输入 [A]        → 得到 P(下一个)
第 2 步: 输入 [A, B]     → 得到 P(下一个)
第 3 步: 输入 [A, B, C]  → ...
```

若每步都重算整段序列的 Attention，会重复计算已生成部分的 K、V。

**KV Cache**：
- 每层每头缓存当前已生成部分的 K、V。
- 新步只算**新 token** 的 Q、K、V，再与缓存 K、V 拼接做 Attention。
- **效果**：每步算量从「与已生成长度平方相关」降为「与长度线性相关」，但需额外显存存 K、V。

工程上：context 长度、batch、层数、头数共同决定显存与延迟。

------------------------------------------------------------------------

## 五、与规模相关的关键参数（调参/选型）

| 参数           | 影响 |
| -------------- | ---- |
| 层数 N         | 越深能力越强，延迟与显存上升 |
| hidden_dim D   | 越宽表达越强，参数量与算力上升 |
| 头数 H         | 通常 D 被 H 整除；头多利于多种关系，L² 代价不变 |
| 词表大小       | 主要影响 Embedding 与 LM Head 参数量 |
| max_seq_len    | 决定 L 上界，直接推高 Attention 的 L² 与 KV Cache 显存 |

------------------------------------------------------------------------

## 六、与现有笔记的对应

- **01_Self_Attention**：Q/K/V、softmax(QK^T/√d_k)V、多头 → 本文的 Multi-Head Attention 与 L×L 显存/算力。
- **04_Chat_API**：角色转成特殊 token 后，经 Embedding 进入上述整条数据流，LM Head 输出再被解码为文本或 tool_calls。

------------------------------------------------------------------------

## 七、工程级理解小结

1. **数据流**：token_ids → Embedding(+位置) → N×(Attention + FFN) → 取最后位置 → LM Head → 概率。
2. **算力/显存热点**：Attention 的 L²、FFN 的 4D²、Embedding/LM Head 的词表维；推理还有 KV Cache。
3. **可调/可选**：层数、维度、头数、max_seq_len、RoPE/ALiBi、是否 tie embedding 与 LM Head。
4. **与 Agent 的关系**：Prompt 经 tokenize 后即为此数据流输入；Agent 开发不直接改这些层，但理解「上下文长度、成本、延迟」来自此处，便于做裁剪与优化。
