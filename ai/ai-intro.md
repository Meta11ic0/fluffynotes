# AI 发展、LLM 与基础设施入门指南

> 时效交叉核对于 2026-07-17（模型日期/参数、TGI 归档、vLLM/SGLang 归属、ORT 角色、Open Responses 等）；细节与数字以 [ai-engineering.md §5](ai-engineering.md) 为准。

## 写在前面

本文面向有编程基础、使用过 AI 工具、希望理解其工作原理的读者。不要求机器学习背景。

同目录三份文档按下面的顺序组织：

| 文档 | 职责 | 何时读 |
|------|------|--------|
| **本文** | 认知地图：概念框架、简史、六个关键问题 | 第一站 |
| [ai-engineering.md](ai-engineering.md) | 工程深读：请求链路、LLM 机制、工种边界、技术栈数据 | 读完本文后 |
| [ai-infra-project-plan.md](ai-infra-project-plan.md) | 动手手册：作品集向 Step 0→4（主线是 LLM 服务化，不是纯 Infra 深水区） | 准备动手时 |

主线是理解「模型 + RAG + 工具 + Eval/观测」如何组成可用产品。CUDA 与多卡调度不在入门范围内。

打开 ChatGPT 问问题，看到 AI 一个字一个字回复。本文从这一点出发，回答：

1. AI 怎么发展到今天？为什么近几年才感觉「到处都是」？
2. 大语言模型（LLM）是什么？和传统程序差在哪？
3. 为什么会出现大量「AI 基础设施」相关岗位？
4. DeepSeek 这类事件意味着什么？
5. 像 Cursor 这样的产品，底层在做什么？

**阅读建议：** 第一章建框架（约 30 分钟），第二章看历史（§2.1 简述早期发展，从 §2.2 Transformer 起进入主线），第三章 §3.1–3.6（核心），第四章产业与决策（可当附录）。全文约 1-2 小时。

**按目的选读：**

| 阅读目标 | 建议路径 | 约需 |
|---------|---------|:---:|
| 产品/设计：建决策框架 | §1.1→§1.3→§3.3→§3.6→§4.5；不必读 plan | 快速浏览 |
| DevOps / SRE | §1.5→§3.4→§3.5 → engineering §1.3 + §3.4 → plan Step 2B | 约 1 小时 |
| 有 ML 背景，补 serving | §1.5 + §3.3–3.5 → engineering §2 → plan Step 1–2 | 约 2 小时 |
| 前端/全栈 → 应用工程 | §1.1→§3.5 → engineering §1.6 + §2.4–2.5 → plan Step 2 | 约 1.5 小时 |
| 关心幻觉 / RAG 质量 | §1.3→§3.3→**§3.6** → plan Step 3 | 约 1 小时 |
| 已会 RAG，补底层 | 跳过 §1.3/§3.3 → engineering §2 → plan Step 2 + Step 4 | 约 2 小时 |
| Tech Lead 带队评估 | §1.3→§3.5→**§4.5** → engineering §1；plan 仅作个人作品集参考 | 约 45 分钟 |
| 求职作品集 | 本文 §1+§3 → engineering §1–3 → plan Step 0–2C | 约 3+ 小时 |
| 做可演示产品（非求职） | §1.3 + **§4.5 成本卡** → engineering §5.7 → API 出 demo → 再自托管 | 按产品节奏 |

文中关键缩写都会解释；里程碑尽量给公开来源。薪资与岗位行情以 [ai-engineering.md §4](ai-engineering.md) 为权威出处，stars / 技术栈公开指标以 [ai-engineering.md §5](ai-engineering.md) 为准；本文只保留趋势判断。

---

## 第一章：先建立认知框架

这一章的目标：**读完后能看懂 AI 技术讨论在说什么。**

---

### 1.1  今天用的 AI，底层到底是什么

用 ChatGPT 问问题、用 Claude 分析文档。这些产品通常以 LLM（大语言模型，Large Language Model）为核心；外层可以是对话界面、IDE 集成或工具调用，形态各不相同。Cursor 写代码时底座仍是 LLM，外面多了一层工具编排。

若用一句话概括今天的大模型 AI：它是在海量文本上训练出来的「下一个字预测器」。规模够大、训练够充分、外围工程够完善时，表现上会像能理解语言、调用工具、完成复杂任务的数字助手。

这里有三个层次：

1. 底层本质：做概率预测，给定前面的字，猜下一个字最可能是什么。
2. 表层体验：对语言、代码、知识训练得足够多，用起来像「会交流、会做事」。
3. 工程现实：产品里能用，不只靠模型本身，还靠推理、缓存、调度、观测等外围系统。

Cursor 一类产品的链路大致是：

```
输入自然语言
  → 模型理解上下文（代码文件、历史对话、项目结构）
  → 系统决定调用哪些工具（读文件、改文件、执行命令）
  → 工具返回结果
  → 模型根据结果继续规划或回答
```

产品层常给人「给清晰指令后系统会匹配动作」的感觉；技术层则是模型负责理解与规划，工具执行读写和命令，推理/调度/观测保障整条链路跑通。

---

### 1.2  AI → ML → DL：三层关系一张图

这三个词经常混用，但它们不是同一层级。

```
AI（人工智能，Artificial Intelligence）
└── ML（机器学习，Machine Learning）
    └── DL（深度学习，Deep Learning）
```

#### AI（人工智能），最外层，最宽泛

让机器表现出「像智能」的行为。导航推荐路径、短视频推荐、语音助手、聊天机器人，这些都算 AI。AI 是**总目标**，不是一种具体算法。

#### ML（机器学习），AI 的实现路径之一

核心思想：**不手写每条规则，让机器从数据里学习规律。**

举个例子（垃圾邮件检测）：
- **规则系统**：手写「含『中奖』就是垃圾邮件」
- **ML 系统**：给模型 10 万封带标签邮件，它自己学「哪些模式更像垃圾邮件」

ML 的关键效果：遇到没见过的表达也能判断对，可扩展性远高于手写规则。

#### DL（深度学习），ML 中最成功的一大类

用**多层神经网络**从数据中学习。在 1.1 节读到的大语言模型（LLM），底层就是一个规模极大的深度神经网络，几十到上百层，每层做固定的数学运算，层与层之间传递结果。对图像、语音、文本这类复杂模式识别效果特别好，在大数据和大算力条件下性能提升远超传统 ML 方法。

**一句话关系：** AI 是目标，ML 是路径，DL 是当前最成功的那条路。

---

### 1.3  纠正一个关键误解：AI ≠ if-else + 数据库

这是一个非常常见、也非常合理的初始理解：

> 「AI 是不是就是程序接一个数据库，用户问 A 就去数据库查相关答案，然后组织成自然语言回答？」

这个理解**有一部分对，但不完整**。把它彻底搞清楚，后面的学习才不会走偏。

#### 规则系统：人写规则

```
如果用户问"营业时间"，回答"9:00-18:00"
如果用户问"退货规则"，回答"7 天无理由"
如果用户说"查询订单"，调用订单接口
```

- 规则由人手工编写
- 规则覆盖到哪里，能力就到哪里
- 没写到的场景，系统就不会

#### 学习系统：机器自己学

真正的变化是：模型自己从数据里学出模式和表示，不再是把 if-else 换成数据库查询。

| | 规则系统 | 学习系统 |
|---|---|---|
| **知识在哪** | 人写的规则里 | 模型的参数（数字）里 |
| **遇到没见过的情况** | 通常失效 | 可能泛化应对 |
| **扩展方式** | 加更多规则（越来越难维护） | 加更多数据（模型自己学） |

这里说的「参数」到底长什么样？是**几亿到几万亿个浮点数**（比如 `0.0312`、`-1.847`），按固定顺序排列。这些数字不是人写的，是模型从训练数据里「练」出来的。可以把它们想象成几万亿个微调旋钮，每个旋钮拧到什么刻度，决定了模型看到某种输入时更可能输出什么。训练的过程就是不断拧这些旋钮，让输出越来越接近期望答案。这在 3.1 节会展开。

那回到最初的理解，「AI 就是接数据库查答案」。规则系统对应了「if-else」，而「接数据库」这部分有没有对应的东西？**有，它叫 RAG。**

#### 那 RAG（检索增强生成）是什么

**RAG = Retrieval-Augmented Generation = 检索增强生成。** 一句话：**先查资料，再回答。**

为什么需要 RAG？因为大模型有两个先天局限：

1. **知识截止在训练时**，训练完成后参数固定，不知道之后发生的事
2. **可能「幻觉」**，编造不存在的信息，尤其在需要精确事实的时候

RAG 的做法很直白：用户提问后，先从外部知识库检索相关文档片段，把这些片段连同问题一起喂给模型，让模型**基于资料**组织回答，而不是纯靠记忆。

```
没有 RAG：    用户问 → 模型凭记忆答（可能过时、可能编造）
有 RAG：      用户问 → 检索知识库 → 找到相关文档 → 拼进 Prompt → 模型基于资料答
```

**规则系统和学习系统是两种根本不同的范式**。前者是人写逻辑，后者是从数据中学。**RAG 是叠加在学习系统之上的一个增强技术**，给大模型接一个外部资料库，弥补它记忆有限、知识过时的问题。

用类比来感受它们的关系：

```
规则系统：        像一本写死流程的操作手册
                  没写到的就不会，但写到的绝对按规矩来

学习系统（大模型）：像一个读过很多书的人，凭记忆回答
                  泛化能力强，但知识截止在训练时，可能记错

     └─ 大模型 + RAG：给这个人配了一个随时可以翻的资料库
                      他仍然用自己的语言能力组织回答
                      但遇到需要精确事实的时候，先翻资料再说话
```

RAG 不是大模型的替代品。它的底座仍然是大模型。没有大模型的语言能力，光有资料库什么也做不了。

> **现代 AI 产品的核心公式是 模型 + RAG + 工具 + Eval/观测。** 模型负责理解和生成，RAG 接外部知识，工具调用让模型不只「说」还能「做」，Eval/观测证明效果与延迟/成本可控。规则系统仍适合安全过滤、格式校验等兜底。RAG 能力与局限见 §3.3；评估见 §3.6。

---

### 1.4  一个句子是怎么被模型「理解」的

> **初读提示：** 本节用七步追踪一个真实句子的完整旅程，不讲矩阵运算，只建立直觉。如果只想先建立全局概念，可以跳到 §1.5「核心名词速览」，等读完第三章再回来看这一节。想了解每一层的精确矩阵计算和形状变化，见 [ai-engineering.md §2](ai-engineering.md)。
>
> **范围：** 本节只沿 **Causal Language Model（CLM，因果语言模型）** 讲——也就是 GPT / Llama / Qwen 这类「从左到右、一次预测下一个 token」的生成式底座。名称里的「因果」是对英文 causal 的通行译法，指当前步不能看未来 token（不是日常说的因果关系）；和「自回归（autoregressive）」常指同一类模型。深度学习里还有视觉、强化学习等；Transformer 里也有 BERT 这类双向路线，不在本节追踪范围内。

用「我 喜欢 吃 红烧肉」这句话，从输入到第一个输出字，追踪全过程。

#### 第一步：把文字变成编号

输入的汉字，模型一个都不认识。它只处理数字。所以第一件事是**分词**，用一个叫 Tokenizer 的工具把句子拆成小块（token），每个小块对应词表里的一个整数编号。

```
输入: "我喜欢吃红烧肉"
输出: 4 个整数，比如 [8247, 15302, 901, 43891]   ← 切分与编号均为教学示意，非真实词表
```

一个 token 可能是一个完整的词（「红烧肉」），也可能是一个字甚至半个字，取决于分词器的设计。模型按 token 数量收费，生成速度按每秒 token 数（tokens/s）衡量。看到字往外蹦，底层是一个一个 token 在往外蹦。

#### 第二步：把编号变成有意义的数字串

光有编号不够，模型还需要知道每个 token 的**语义**。于是每个编号被映射成一个高维向量，一长串小数。这个过程叫 Embedding。

关键特性：意思相近的词，向量在数学空间里的距离也近。「猫」和「狗」的向量很近，「猫」和「电脑」的向量很远。这个性质不是人设定的，是模型在海量文本中自己学会的。

#### 第三步：告诉模型「谁在前谁在后」

现在每个 token 都有了一个携带语义的向量。但有一个盲区：模型还不知道顺序。「我 喜欢 吃 红烧肉」和「红烧肉 喜欢 吃 我」用的是同一组词的向量，含义完全不同，但只看向量的话它们长得一模一样。

解决方法是注入**位置信息**。直观理解是：第 1 位的「我」带上「位置 1」的标记，第 4 位的「红烧肉」带上「位置 4」的标记，模型才能区分「谁先出现、谁后出现」。早期做法常把位置向量加到 embedding 上；当代 decoder-only 模型多用 RoPE，在 Attention 里旋转 Q/K（机制见 [ai-engineering.md §2.1](ai-engineering.md)）。

#### 第四步：Attention 计算「当前词该关注谁」

有了位置信息，模型现在知道每个词在句子的哪个位置。以句末的「红烧肉」为例，它可以看到前面的「我」「喜欢」「吃」，其中「吃」与它的关系最直接。

模型需要一种机制来判断「处理当前词时，前面哪些词更重要」。这个机制就叫 **Attention（注意力）**。预测下一个 token 时，当前位置还看不到后面尚未出现的字，因此只许看自己及前面——这种受限的 Attention 叫**因果注意力**（实现上常靠 mask 挡住未来位置）。

- 对「吃」权重大（动作和对象的关系）
- 对「喜欢」也有权重（态度关系）
- 对「我」有权重（动作主体）

这就是 Attention 的核心直觉，**动态聚焦于当前最相关的部分**。

互看之后，每个位置上的向量已经掺进了上下文，但还要再经过一块只作用于「当前位置」的变换，才算走完一层。这就是下一步的 FFN。

#### 第五步：FFN，每个 token 各自再加工

仍以句末的「红烧肉」为例。Attention 已经按权重把「吃」「喜欢」「我」写进了这一位的向量；上下文拢齐了，还要依据这份背景，把当前位置的表达再改写一遍，才能交给后面的层。

这一步叫 **FFN**（Feed-Forward Network，前馈网络）。它不再看别的 token，只用训练学来的参数，对当前位置这一个向量做非线性变换。同一层里每个位置各自做一遍。

为什么拢完还要再变：

- 拢来的主要是加权拼进的上下文；还要依据这份背景，把当前位置的表示继续改写
- Attention 管的是谁的信息该进来；进来之后怎么算，靠的是另一块逐位置计算
- 实际模型里很大一块参数容量落在 FFN 上，正是为这种「各自改写」准备的

一层里可以记成：**Attention 负责互看，FFN 负责各自消化。** 两者做完，才叫 Transformer 的一层。机制见 [ai-engineering.md §2.3](ai-engineering.md)。

#### 第六步：多层堆叠，逐层抽象

单层（Attention + FFN）不够。实际模型会堆几十到上百层（比如 Llama-3-8B 有 32 层），每一层都在前一层的输出上继续「互看 + 各自加工」。用一个稍复杂点的例子来看，堆叠之后同一位置的向量可能携带什么：

> 「小明说他昨天去了那家新开的川菜馆，菜很辣但是他很喜欢」

这个句子经过分词后大约有十几个 token。为了展示逐层编码的过程，追踪**最后一个 token「喜欢」**（或 tokenizer 切出的子词「欢」）：它在句末，经因果注意力能看到前面所有 token，携带的上下文最完整。

每一层产出的是高维向量（一串数字），不是自然语言「结论」。研究者做过**探针实验**：在某层输出的向量上训练简单分类器，看它能否识别某种信息，从而推测该层编码了什么。

下面用探针视角做一个**教学示意**（层号与「表层 / 句法 / 语义 / 语用」的对应是方便记忆的故事，不是固定物理分层；真实模型里信息是分布式的，不同架构差异很大）：

```
浅层：更像在编码局部邻接与浅层模式（「喜欢」附近有「很」「他」）
中层：更容易检出指代、转折等结构线索（「他」可能指向「小明」）
深层：更容易检出语义冲突与整体倾向（「辣」与「喜欢」被「但是」隔开后如何收束）
```

这里需要保留的判断是：**层越深，同一个位置的向量通常携带越多上下文，但不能把第 N 层固定解释为某一种职责。**

**两点补充：**

第一，上面的例子只追踪了「喜欢」这一个位置，但实际上**输入句子的每一个 token 都在同时走这个过程**，「小」「明」「说」「他」「昨」「天」……每个字过完模型配置的全部层（Llama-3-8B 是 32 层）后，都得到了一个编码了丰富信息的向量。只是不同位置的向量携带的上下文量不同，越靠后的位置看到的越多。

第二，编码完之后呢？这就到了第七步。

#### 第七步：从向量到「下一个字」

模型配置的全部层都走完后，每个 token 都得到了一个编码了上下文的最终向量。要预测下一个字，模型取**最后一个 token** 的最终向量（本例是「喜欢」或其子词）。这个向量经最后一轮矩阵乘法投影到词表大小，再经 softmax 变成概率分布：

```
"喜欢"的最终向量 → [lm_head 投影] → ["吃": 0.31, "的": 0.18, "。" : 0.12, "这家": 0.08, ...]
                                                    ↑ 概率为教学示意，非真实模型输出
```

然后按策略选一个：贪心就选概率最高的「吃」；加一点随机性，也可能选到「的」或「这家」。选出来的 token 追加到输入末尾，整个流程再跑一遍，生成下一个。这就是屏幕上一个字一个字往外蹦的原因。

因此 LLM 可以概括成「下一个 token 预测器」：输入一串 token，经几十层编码，把末尾位置的向量投影成词表上的概率分布再选出下一个。规模上去之后，通用能力才会显得突出。

---

### 1.5  核心名词速览

这一节不展开，只快速说明「这个词在系统里起什么作用」。需要深入理解时翻到第三章。

#### 基础概念

| 名词 | 一句话 | 在哪里用 |
|------|--------|---------|
| **LLM** | 大语言模型，在超大规模文本上训练出来的「下一个 token 预测器」 | 所有 AI 聊天/编码产品 |
| **Token** | 模型处理文本的最小单位，不一定是完整的一个字 | 分词、计费、速度计算 |
| **Embedding** | 把词/句映射成数字向量的技术，语义接近的向量也接近 | 检索、相似度计算、RAG |
| **Transformer** | 2017 年提出的以 Attention 为核心的神经网络架构 | 当代大语言模型的主要底座 |
| **Attention** | 让每个 token 在处理时动态关注上下文中其他 token 的机制 | Transformer 核心组件 |
| **FFN** | 每层中对单个 token 各自做非线性变换的前馈模块；与 Attention 合起来构成一层 | Transformer 层内；MoE 等变体也常作用在这块 |

#### 模型生命周期

| 名词 | 一句话 | 谁在做 |
|------|--------|--------|
| **训练（Training）** | 在海量数据上学习参数，形成模型通用能力 | 算法团队 |
| **推理（Inference）** | 训练好的模型上线响应用户请求 | AI Infra / 推理工程师 |
| **微调（Fine-tuning）** | 在预训练模型基础上做任务定制 | 算法 + 工程 |

#### 从「会说」到「会干活」

| 名词 | 一句话 | 解决的问题 |
|------|--------|-----------|
| **RLHF** | 基于人类偏好反馈优化模型输出 | 让模型更「听话」，减少有害回答 |
| **RAG** | 先检索外部知识，再让模型基于检索结果生成答案 | 时效性、私有知识、可追溯性 |
| **Agent** | 模型 + 工具 + 规划执行流程的组合系统 | 不只回答问题，还能调用工具执行任务 |

#### 基础设施常见词

| 名词 | 一句话 | JD 中常见 |
|------|--------|-----------------|
| **推理引擎** | 高效执行推理计算的核心组件（如 ONNX Runtime、TensorRT） | 常见 |
| **推理框架** | 引擎 + KV Cache 调度 + 多卡并行 + 请求队列（如 vLLM、SGLang） | 常见 |
| **KV Cache** | 缓存推理过程中的中间状态，避免重复计算 | 常见，高频 |
| **GPU** | 在 AI 中的核心价值是并行矩阵运算，不是画图 | 常见 |
| **QPS** | 每秒可处理请求数，衡量系统容量 | 常见 |
| **P95/P99** | 95%/99% 请求在该时间内完成，衡量用户真实体验 | 常见 |
| **TTFT / TPOT** | 首 token 延迟 / 后续 token 平均间隔；拆「接口慢」用，口径见 engineering §3.4 | 常见，高频 |
| **SSE** | 服务端用 `text/event-stream` 推事件；Chat Completions 流式常见，前端多用 fetch 解析 | 常见，高频 |
| **messages / tool_call** | 多轮角色消息与工具调用协议字段；最小闭环见 engineering §1.6 | 常见，高频 |
| **Batching** | 把多个请求合并一起算，提升吞吐 | 常见 |
| **FastAPI** | Python 异步 Web 框架，常用于 AI 服务化 API | 常见，高频 |
| **Docker / K8s** | 容器化常见；K8s 编排在平台/部署岗高频 | 常见，高频 |
| **SLM** | 小语言模型（0.5B-7B），适合高频简单任务；窄任务上常更便宜，视部署与调用量而定（见 engineering §5.7） | 增长中 |

> 以上各项的 2026 年公开数据（查询 2026-07-17）：vLLM ~86k stars、SGLang ~30k stars、ONNX Runtime ~21k stars / v1.27.0、FastAPI ~100k stars / 38% 开发者采用（JetBrains 调查口径）、K8s 在容器用户中生产使用约 82%（CNCF 2025 年调查，2026.1 公告）。另：HF Hub 上可统计参数量的模型下载中，约 92%+ 来自 1B 以下（含 embedding / 编码器等，不能直接读成「聊天 SLM 占 92%」）。完整数据及来源见 [ai-engineering.md §5](ai-engineering.md)。

#### 全景图

**五层工作流**：用户请求依次穿过这五层，每层有自己在做的事和主流技术。

```
用户: "这段代码什么意思"
  │
  ▼
第5层 应用增强       RAG 查资料 / Agent 调工具 / LangChain 编排
  │  (可选)           补知识、补行动力
  ▼
第4层 部署交付       API/SSE/Docker/K8s/Prometheus（学习常用 FastAPI；生产网关也常见 FastAPI）
  │                  把推理能力暴露成可访问、可观测的服务
  ▼
第3层 推理引擎/框架   生产 GPU 常见 vLLM / SGLang；CPU/边端常见 ONNX Runtime
  │                  决定"怎么跑、跑多快"（学习可用 HF generate，勿写成生产标配）
  ▼
第2层 模型           Qwen / Llama / DeepSeek 的权重文件（几十GB）
  │                  "谁的知识在回答"
  ▼
第1层 模型架构       Attention 让 token 互看 / FFN 深度加工每个 token
  │                  决定"模型内部怎么算"
  ▼
输出 token → 逐字推回用户
```

**第1层 — 模型架构**

Attention 和 FFN 各自有一个「目的」，不同方案是达成这个目的的不同实现：

| 目的 | 经典实现 | 后来的实现 |
|---|---|---|
| 让 token 之间互看（Attention） | 标准 Attention | MLA（DeepSeek V2/V3，压缩 K/V 省显存） |
| | | CSA+HCA（DeepSeek-V4，两级压缩+稀疏索引） |
| | | GatedDeltaNet（Qwen3-Next / Qwen3.6 等，线性注意力路径） |
| （辅助）加速、省显存 | — | FlashAttention（通用加速）、GQA（Llama 3，省显存） |
| 深度加工每个 token（FFN） | 标准 FFN | SwiGLU（Llama / Qwen / DeepSeek 主流） |
| | | MoE（DeepSeek-V4 / Qwen3.6 / Llama 4 等，每次只激活部分专家） |

> Attention 和 FFN 各自独立迭代，一个模型可以同时选两边的方案（如 DeepSeek-V4-Pro = CSA+HCA + MoE）。

**第2层 — 模型**（权重文件）

| 开源 | 闭源 |
|---|---|
| Qwen、Llama、DeepSeek、Gemma、Mistral | GPT、Claude、Gemini |

**第3层 — 推理引擎/框架**（把模型跑起来）

| 层级 | 做什么 | 主流选择 |
|---|---|---|
| 推理框架（上层） | 调度请求、管 KV Cache、对外服务 | vLLM、SGLang、TensorRT-LLM、Ollama、llama.cpp |
| 推理引擎（底层） | 加载模型、执行矩阵计算 | ONNX Runtime、TensorRT、PyTorch 原生 |

> GPU 生产：常见选 vLLM 或 SGLang（也可分工部署，见后文）；本地：Ollama；CPU/边缘：ONNX Runtime。学习路径可用 HF + FastAPI 把链路跑通，不等于生产 Serving 架构。

**第4-5层 — 部署交付 + 应用增强**

这两层没有竞品关系，各环节是串联的工具链，不是互替选项：
- 部署链：API 层（常 FastAPI）→ SSE（流式）→ Docker（容器）→ K8s（编排，平台岗常见）→ Prometheus（监控）
- 增强链：RAG（LangChain / LlamaIndex + ChromaDB / Milvus）→ Agent（LangGraph / AutoGen）

---

## 第二章：AI 是怎么一步步发展到今天的

本章只选对今天生态影响较大的转折点，不做完整编年史。

### 2.1 早期发展（1956–2016）

1956 达特茅斯提出 Artificial Intelligence；感知机证明「可训练分类」，但能力有限。1986 反向传播让多层网络可训；LeNet/LSTM 推进视觉与序列；2006 CUDA 让 GPU 成为通用并行平台。2012 AlexNet 证明「深网 + 大数据 + GPU」；word2vec、seq2seq、Bahdanau Attention 铺路；2016 AlphaGo 推高公众预期。

相关论文和资料见文末附录。

---

### 2.2 Transformer 登场，大模型的底座（2017–2019）

#### 2017：Transformer，真正的大拐点

Google 发表论文《Attention Is All You Need》，提出 Transformer 架构。

为什么它如此重要？因为它解决了以前 RNN/LSTM 体系的核心瓶颈：

| RNN/LSTM 的问题 | Transformer 的方案 |
|---|---|
| 必须按顺序一步步处理，无法高效并行训练 | 序列中各位置可同时处理，充分利用 GPU 并行能力 |
| 长序列中早期信息容易丢失 | Attention（回顾 1.4 第四步，每个 token 动态关注上下文中所有其他 token）让每个位置直接连接所有其他位置，不受距离限制 |
| 扩展到海量数据和超大模型受限 | 天生适合大规模扩展 |

**Transformer 让更大规模的语言建模成为工程现实。** GPT、Claude、Gemini、Llama、DeepSeek、Qwen 等当代大模型主要建立在 Transformer 及其变体上。

> 参考：[Transformer 原论文](https://proceedings.neurips.cc/paper_files/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf)

#### 2018：BERT，预训练 + 微调范式

BERT（Bidirectional Encoder Representations from Transformers）在多项 NLP 基准上显著推高了成绩。它代表的是预训练 + 微调范式的成功：

- 先在大规模语料上通用预训练
- 再在具体任务上稍作微调

从此**不需要为每个 NLP 任务都从零开始训练一个模型**。

> 参考：[BERT 论文](https://arxiv.org/abs/1810.04805v1)

#### 2019：T5 与「The Bitter Lesson」

- **T5**（Text-To-Text Transfer Transformer）：把几乎所有 NLP 任务统一成「输入文本 → 输出文本」的形式。这对后来的大模型产品很重要，今天看到的聊天、翻译、摘要、代码补全，本质上都在走「统一接口」这条路。
- **The Bitter Lesson**：Rich Sutton 的文章在思想层面影响巨大。核心观点：**长期看，依赖通用方法 + 大规模计算，通常比大量手工规则和领域技巧更有效。** 这一判断与后来「先预训练大模型、再堆算力扩展」的路线一致。

> 参考：[T5 论文](https://arxiv.org/abs/1910.10683v3) | [The Bitter Lesson](https://bitterlesson.ai/)

---

### 2.3 GPT-3 → ChatGPT，大模型进入大众产品（2020–2022）

#### 2020：GPT-3，规模带来涌现式通用能力

GPT-3（Generative Pre-trained Transformer 3）的行业意义，不只在于参数规模（1750 亿），更在于少样本学习（Few-shot）等任务上的表现：很多任务不必专门训练也能完成。

- 少样本学习（Few-shot）：给它几个例子，它就能举一反三
- 一个模型同时做翻译、问答、写作、摘要、代码生成
- 产业界开始把它当成可能覆盖多种任务的通用接口，而不只是 NLP 学术工具

> 参考：[GPT-3 论文](http://arxiv.org/abs/2005.14165)

#### 2020：RAG 提出

同年发表的 RAG（Retrieval-Augmented Generation）针对大模型知识截止与幻觉问题，提出「先检索外部证据、再生成答案」的路径。定义、局限与工程形态见 §1.3；评估见 §3.6。

> 参考：[RAG 论文](https://arxiv.org/abs/2005.11401)

#### 2021：Foundation Model 概念 + LoRA 微调

- **Foundation Model（基础模型）** 概念被正式提出：先训练一个通用大模型，再在很多具体场景上适配。模型开始像「底层平台」，下游应用不必每次重来。
- **LoRA**（Low-Rank Adaptation）：低成本微调方法。不是所有场景都要重新全量训练模型，很多时候可以更便宜地定制。

> 参考：[Foundation Models 报告](https://arxiv.org/abs/2108.07258) | [LoRA 论文](http://arxiv.org/pdf/2106.09685)

#### 2022：InstructGPT + RLHF → ChatGPT

GPT-3 很强，但并不一定「听话」。InstructGPT 引入 RLHF（Reinforcement Learning from Human Feedback，基于人类反馈的强化学习），让模型从「能生成语言」变成**「更会按照人的意图回答」**。

2022 年 11 月 ChatGPT 发布。对话界面自然、RLHF 调过的回复质量、免费试用与低门槛交互，把大模型能力带进了大众产品，体验明显区别于传统规则型客服机器人。

> 参考：[InstructGPT 论文](https://proceedings.neurips.cc/paper/2022/hash/b1efde53be364a73914f58805a001731-Abstract.html) | [ChatGPT 发布](https://www.openai.com/blog/chatgpt/)

---

### 2.4 产业生态扩展，从聊天到工具调用（2023–2024）

#### 2023：开源权重生态崛起

- **Llama 2**（Meta，2023.7）：开源权重发布，让更多公司和开发者可以自己部署模型。生态从「少数闭源 API」变成「闭源 + 开源并行」。
- **Mistral 7B**（2023.9）：证明了小模型也能有竞争力，降低部署门槛。
- **Gemini**（Google，2023.12）：Google 的多模态模型首次亮相，原生支持文本、图像、音频。

#### 推理服务优化（2023 起）

随着模型开始大规模上线，行业发现：**模型能跑，但太慢、太贵、显存占用太大。** 行业重心开始从「怎么训练更强模型」转向「怎么更高效地服务模型」。

| 技术 | 年份 | 解决的问题 | 当前状态 |
|------|------|-----------|---------|
| **FlashAttention** | 2022-2023 | 长上下文 Attention 的速度和显存问题，从 GPU 内存访问层面做优化 | v3 已发布；训练/推理栈中常见 |
| **vLLM / PagedAttention** | 2023 | KV Cache 浪费显存、碎片化导致 batch size 受限，借鉴操作系统分页思想管理显存 | PyTorch Foundation 托管；K8s 上常见生产 Serving 选择之一（「40 万+ GPU」类数字见 engineering §5.2，待核实） |
| **SGLang** | 2024 | 推理框架的编程模型和调度效率，RadixAttention 前缀缓存 | PyTorch Ecosystem 项目；DeepSeek V3 **特定配置**下相对 vLLM 约 3.1×（非全场景） |
| **HuggingFace TGI** | 2021-2026 | 早期的自托管 LLM 推理服务 | **2025.12.11 维护模式，2026.3.21 仓库归档只读**（GitHub API 核对于 2026-07-17）；新项目多转向 vLLM / SGLang |
| **Ollama** | 2023-2026 | 本地 LLM 推理工具，一键拉取运行开源模型 | 个人开发/本地实验首选；2026 起有部分 batching 基建，但仍不以生产级 Continuous Batching / 分布式部署见长 |

> 以上各项的 stars 数、版本号等精确数据：vLLM ~86k / SGLang ~30k / Ollama ~174k / TGI 已归档。完整数据见 [ai-engineering.md §5](ai-engineering.md)。

这些技术说明：行业重心已经从「只训更强模型」明显前移到「怎样更稳、更便宜地服务模型」。

> 参考：[FlashAttention 论文](https://papers.nips.cc/paper_files/paper/2022/hash/67d57c32e20fd0a7a302cb81d36e40d5-Abstract-Conference.html) | [vLLM / PagedAttention 论文](https://arxiv.org/abs/2309.06180)

#### 产品形态多元化

- **GitHub Copilot**（2021 技术预览 / 2022 GA）：大模型进入 IDE 编码补全；2023 起 Copilot Chat 把能力从补全扩展到对话
- **Cursor**（2024）：在补全之外，把模型、IDE 与工具调用收成编程代理
- **Claude Code**（Anthropic，2025）：终端原生 AI 编程助手，具备工具调用和自主编码能力的 CLI Agent
- **Claude 3.5 Sonnet**（Anthropic，2024.6）：在编码和推理上表现突出，200K 上下文

---

### 2.5 推理能力与架构演进（2025–2026）

这一阶段还在进行中。常见主题是：在堆更大模型之外，也更重视怎么用好已有模型（路由、推理时扩展、小模型承接高频路径等）。

#### 2025 上半年：推理能力的涌现

| 时间 | 事件 | 意义 |
|------|------|------|
| 2025.1 | **DeepSeek-R1** 发布 | 用纯强化学习 GRPO（组相对策略优化，不需要单独训练评估模型，通过组内比较自行学习）+ 无人工标注的推理过程，模型涌现出自主推理能力（「aha moment」，模型自己发现错误并修正）。权重开源 |
| 2025.4 | **Qwen3** 发布 | 开源 8 个模型（0.6B-235B），统一「思考/非思考」模式，Apache 2.0 协议 |
| 2025.4 | **Llama 4 Scout/Maverick** 发布 | Meta 首次在 Llama 引入 MoE；Scout 17B 激活、标称 1000 万上下文（训练约 256K，靠 iRoPE 等做长度泛化）；Maverick ~400B 总参 / 17B 激活、1M 上下文 |
| 2025.5 | **Claude 4** 系列发布 | Claude Opus 4 / Sonnet 4，200K 上下文，Anthropic 的旗舰推理模型 |

#### 2025 下半年：GPT-5 与架构趋同

| 时间 | 事件 | 意义 |
|------|------|------|
| 2025.8 | **GPT-5** 正式发布 | OpenAI 统一系统：路由在快速回复与深度推理之间切换；API 总上下文约 400K（约 272K 输入 + 128K 输出/推理，官方开发者说明）；架构细节（是否 MoE）官方未公开 |
| 2025.9 | **Qwen3-Next** 发布 | 80B 总参数仅约 3B 激活，引入 GatedDeltaNet 线性注意力；长上下文（>32K）相对 Qwen3-32B 可有约 10× 吞吐提升（厂商口径） |
| 2025 全年 | 多个新模型采用 MoE | DeepSeek V3 的 MLA+MoE 等路线被 Mistral Large 3、Kimi K2、GLM-5 等跟进；稠密模型仍在继续发布 |

#### 2026：效率优化与长上下文

| 时间 | 事件 | 意义 |
|------|------|------|
| 2026.2 | **Qwen3.5** 发布 | 原生多模态，GatedDeltaNet + MoE；如 35B-A3B（3B 激活）在多项公开基准上达到或超过前代 Qwen3-235B |
| 2026.4 | **DeepSeek-V4-Pro** 发布 | 系列含 Pro（1.6T 总参 / 49B 激活）与 Flash（284B / 13B）；CSA+HCA 混合注意力，1M 上下文；官方称 1M 设定下相对 V3.2 约 27% 单 token FLOPs、约 10% KV Cache；权重见 Hugging Face（许可证以模型卡为准） |
| 2026.4 | **Qwen3.6** 发布 | 如 35B-A3B（35B 总参 / 3B 激活），侧重 Agentic Coding；262K 原生上下文、可扩展至约 1M；Apache 2.0 |

#### 这一阶段的关键趋势

> 技术全景图见 [§1.5](#15-核心名词速览) 末尾，那里画出了各层竞品关系，读到后面忘了谁跟谁竞争可以随时翻回去看。

**1. MoE（混合专家）成为主流架构**

模型里有很多「专家子网络」，每次只激活其中几个。参数总量可以很大，但单次推理计算量相对较小。以 DeepSeek-V4-Pro（约 49B/1.6T ≈ 3%）与 Qwen3.6-35B-A3B（约 3B/35B ≈ 9%）为例。

**2. Attention 变体在抢 KV 与长上下文效率（标准 softmax Attention 仍是底座）**

| 方案 | 代表 | 思路 |
|------|------|------|
| MLA | DeepSeek V2/V3 | K/V 压缩再缓存，降低 KV Cache |
| CSA+HCA | DeepSeek V4 | 两级压缩 + 稀疏索引 |
| GatedDeltaNet 等 | 部分新模型 | 线性注意力等，降低长序列代价 |

入门仍应先学标准因果 Attention；上面是生产里常见的「省显存 / 提吞吐」变体，不是「标准 Attention 已经过时」。

**3. 能力密度（常称 Densing Law）**  
公开讨论里常举：2026 年约 3B 激活参数的模型，在部分任务上可接近 2023 年 200B+ 稠密模型的表现。规模仍重要，但架构与训练效率的权重在上升。入门阶段知道有这回事即可，不必当作定律背。

#### 延伸：当前技术栈的关键格局

以下三点影响 2026 年常见选型讨论，细节与数字以 [ai-engineering.md §5](ai-engineering.md) 为准。

**1. 推理框架：vLLM 与 SGLang 常被一起讨论（协同是一种实践，不是唯一标准）**

2025-2026 年，除了二选一，也有团队采用 vLLM **和** SGLang 分工：

- **vLLM** 做生产基座：扛并发、管 KV Cache、PagedAttention
- **SGLang** 做部分加速与结构化输出：JSON Schema、RadixAttention 等

有公开实战文给出协同方案在特定模型/硬件上的格式错误率对比；把它当作**一种可参考的架构选项**即可，完整选型数字见 [ai-engineering.md §5](ai-engineering.md)。

**国产芯片与中间层（公开报道口径，细节随版本变）：**
- 昆仑芯等厂商有 vLLM/SGLang 适配与集群演示（如 P800 相关报道）；插件化 XPU 后端是常见集成方式
- 云厂商方案里常见把 vLLM/SGLang 列为部署选项之一
- 昇腾/海光等国产芯片的框架适配，常出现在企业选型清单
- Xinference 等统一推理管理平台被部分团队用作多引擎中间层，并非唯一标准

> 参考：[vLLM+SGLang 协同部署实战](https://devpress.csdn.net/amd/6a38e5c310ee7a33f280bbe9.html)、[昆仑芯 + vLLM/SGLang](https://www.kunlunxin.com/news/5002.html)、[阿里云 GLM-5.2 部署方案](https://developer.aliyun.com/article/1742277)

**2. ONNX Runtime：与 vLLM/SGLang 不在同一层**

ONNX Runtime 是微软开源的跨平台推理引擎（§1.5 名词表提过：加载模型、执行矩阵计算）。它与 vLLM/SGLang 不在同一产品层级：ORT 偏底层引擎，vLLM/SGLang 偏上层 Serving 框架（框架内部可调用不同引擎）。GPU 高并发 Serving 常见 vLLM/SGLang；ORT（约 21k stars，v1.27.0，查询 2026-07-17）更常出现在边端、CPU、ARM 服务器、跨硬件部署。Arm + KleidiAI 集成后，ARM CPU 推理加速约 2.4×（ORT 官方博客口径）。常见分工可以记成：HF Transformers 做实验与加载，vLLM/SGLang 做 GPU 生产服务，ORT 做跨平台/CPU/边缘。场景对比与限制见 [ai-engineering.md §5.6](ai-engineering.md)。

**3. SLM（小语言模型）生产化**

2025-2026 年，可自托管的小模型在分类、抽取、简单问答等高频路径上更常被考虑。HF Hub 上约 **92%+** 的下载来自参数量 1B 以下的模型（Hub 下载分布，含 embedding 等，≠ 生产对话一律走 SLM；口径见 [engineering §5.7](ai-engineering.md)）。NVIDIA 论文对若干 agent 的案例估计约 **40-70%** 的调用可由更小模型承接（非全行业普查）。常见做法是按任务**混合路由**，SLM / 中等 API / 前沿 API 的比例因业务而异，不存在统一的企业配比。成本与模型列表见 [ai-engineering.md §5.7](ai-engineering.md)。

---

## 第三章：六个关键问题

§3.1–3.6 覆盖训练与推理、架构选择、RAG、KV Cache、请求链路与评估。读完后应能说明：模型为何能跑、卡点在哪、相关岗位在补哪一段。

---

### 3.1  训练和推理的区别是什么

**最短答案：** 训练是把模型「学出来」，推理是把学好的模型「拿来用」。

#### 用普通人能理解的话说

- **训练**：像「上学、做题、练习」，模型接触大量数据，不断调整自己的参数
- **推理**：像「毕业以后正式上岗回答问题」，参数固定，只根据输入生成输出

#### 训练为什么这么贵

训练不只是「算一遍结果」，而是每轮都要做四件事：

1. **前向计算**：先算出预测结果
2. **计算误差**：看与真实答案差多少
3. **反向传播**：把误差信号逐层往回传
4. **更新参数**：让模型下一次更接近正确答案

大模型训练需要：
- 几万亿 token 的训练数据
- 成百上千张 GPU 跑几周到几个月
- 几百万到几千万美元的成本

#### 推理为什么也不便宜

很多人以为「训练才贵，推理很轻」，这不对。大模型推理成本也很高：

1. 模型参数大（权重常驻显存）
2. 上下文可能很长（KV Cache 线性增长）
3. 生成是逐 token 串行进行的，不是一次性输出
4. 高并发下 GPU 和显存争抢激烈

产业界重视推理成本优化；规模上线后，推理成本往往占运营成本的大头，小幅优化也会带来可观的累计节省。

#### 与日常工作的关系

日常接触到的几乎都是推理，不是训练。很多岗位和项目只做推理，不做训练。比如企业内部问答、AI 助手、AI 编码工具，这些更多是在做模型部署、推理优化、请求调度、缓存管理、观测与发布。

---

### 3.2  Transformer 为什么比 LSTM 更适合大模型

#### LSTM 为什么曾经重要

LSTM 解决了普通 RNN「处理长序列时会失忆」的问题。比如一句话：「虽然前面铺垫了很多内容，但最后一个代词'它'到底指什么？」模型需要记住很远之前的信息，LSTM 通过门控机制改善了这个问题。

#### 但 LSTM 不适合今天的大模型

核心原因：**它天生不够适合超大规模并行训练。**

LSTM/RNN 是「顺序执行」的：先处理第 1 个词，再处理第 2 个，再处理第 3 个。这导致：
- 难以高效并行
- 训练速度受限于串行瓶颈
- 扩展到超长上下文和超大数据时很吃亏

#### Transformer 的关键优势

关键点是序列中各位置可以并行处理。Attention 让当前 token 直接「看」上下文中其他 token，不必像 RNN 那样一步一传。

| | LSTM | Transformer |
|---|---|---|
| **训练并行性** | 差（串行依赖） | 好（可并行处理所有位置） |
| **长距离依赖** | 改善但不完美 | 直接建模任意距离关系 |
| **扩展到超大规模** | 困难 | 天然适合 GPU 矩阵运算 |
| **大模型时代的角色** | 不再是通用 LLM 的主要架构 | 通用 LLM 的主要底座 |

当代通用大模型产品（GPT、Claude、DeepSeek 等）主要建立在 Transformer 及其变体上；并行训练能力是规模能做大的工程前提之一。

---

### 3.3  RAG 解决了什么、没解决什么

#### 最短定义

**RAG（Retrieval-Augmented Generation，检索增强生成）可以概括为：先检索外部证据，再让模型组织回答。**

#### 解决了什么

**问题 1：模型知识不是最新的。** 模型训练完后参数固定，新发生的事情它不知道。RAG 通过检索外部知识库来补这个问题。

**问题 2：企业私有知识不在通用模型里。** 公司内部 SOP、产品手册、合同模板、历史项目文档，这些不会出现在通用大模型训练数据里。RAG 把这些接进来。

**问题 3：需要出处和证据。** 纯模型回答时难以提供「信息来源」。RAG 可以把检索到的文档片段一起呈现，提升可追溯性。

#### 没解决什么

**没解决 1：模型本身推理能力不够。** RAG 只能补充外部信息，不能补足模型本身的推理能力。逻辑能力弱的模型，给了资料也总结不好。

**没解决 2：检索不准时，回答也会偏。** RAG 效果高度依赖检索质量。文档切分差、向量检索不准、排名错了，模型就基于错误资料回答。

**没解决 3：幻觉不会彻底消失。** 「上了 RAG 就没有幻觉」是误解。RAG 只能降低一部分幻觉，模型仍可能误读文档、过度推断、自己编造。

#### 实用理解

RAG 更像给一个会说话的模型配了「资料库」；它仍不能自动保证事实正确。从产品落地角度看，RAG 是企业较现实的切入点之一，不用从零训练模型，可以快速接企业知识，能直接形成问答、助手、搜索增强等产品。

RAG 能补充模型参数之外的证据，但不保证模型正确使用这些证据。生成阶段的显存与延迟，还受 KV Cache 等机制约束，见下一节。

---

### 3.4  为什么 KV Cache 是推理系统的核心瓶颈

#### 先讲生成式模型怎么输出内容

大语言模型不是一次把整段话吐出来。它是这样生成的：

```
看当前输入 → 预测下一个 token → 把新 token 接回输入 → 再预测下一个 → 重复
```

假设已经生成了 500 个 token。如果每生成第 501 个 token 都要把前面 500 个完整重算一遍，代价极高。

#### KV Cache 做什么

**缓存前面已经算过的中间状态，后续生成时直接读取，不重算。** KV Cache = Key-Value Cache，缓存的正是 Attention 计算中每个 token 在各层的 Key 和 Value。

有了 KV Cache，生成第 501 个 token 时只需要：读全量缓存 + 算 1 个新 token 的 Attention → 缓存末尾追加一行。

#### 但它为什么成了「大问题」

因为 KV Cache 非常占显存，且大致随下面因素增长（完整公式含 K/V 双份、dtype 字节数、GQA 等压缩，见 [ai-engineering.md §2.4](ai-engineering.md)）：

```
KV Cache 大小 ∝ 层数 × 序列长度 × KV 维度 × 2(K+V) × 字节数 × 并发请求数
（若使用 GQA/MLA 等，有效 KV 头数会再缩小）
```

- 单请求长对话：常见是几十到几百 MB 量级（视模型与长度）
- 高并发合计：才容易到十几 GB、几十 GB；别把「总占用」误当成「单请求」
- 不同请求长度不同，静态预分配会产生碎片浪费
- 合批不好做时，GPU 算力吃不满

#### 这为什么会催生推理基础设施

线上服务真正要回答的是：

- 同时 100 个用户来怎么办？
- 长上下文会不会把显存吃爆？
- 如何合批又不把延迟打爆？
- 如何减少 KV 预分配造成的浪费？

vLLM 的 PagedAttention（按块管理 KV，缓解碎片）、FlashAttention（优化显存访问）、DeepSeek 的 MLA/CSA（压缩 K/V）都是在打这些问题。注意：PagedAttention 论文里「利用率从约 20–40% 提到接近 100%」主要指 **KV 预分配浪费被压下去**，不是整张 GPU 的算力利用率变成满分。

#### 直觉类比

KV Cache 像**写长文时的草稿和记忆区**。如果每写一句话都要把前面所有内容重新从零推导一遍，效率极差。于是系统保留中间记忆，但记忆区很占地方，人一多就不够用，管理不好就混乱。AI 基础设施工程，很多就是在解决这个「记忆区如何高效管理」的问题。

---

### 3.5  一个 AI 产品从请求到回答的完整链路

以在 Cursor 中输入「帮我解释这个函数，并修复潜在空指针问题」为例：

```text
第一步：用户发请求
  → 前端/客户端接收输入

第二步：服务入口
  → 认证鉴权 → 会话管理 → 限流 → 路由 → 日志记录

第三步：上下文构建
  → 读取当前文件、项目结构、历史对话
  → 拼成模型的输入 Prompt

第四步：是否走 RAG / 检索（可选）
  → 需要查知识库？检索文档片段 → 拼进 Prompt

第五步：首次模型推理（进入 AI Infra 核心区）
  → 分词（Tokenize）
  → Prefill（预填充）：并行处理全部输入，建好 KV Cache
  → Decode（解码生成）：逐 token 生成，每次追加到 KV Cache
  → 返回普通文本，或返回 tool_call

第六步：执行工具并再次推理（可选循环）
  → 系统执行读文件/修改代码/命令等工具
  → 工具结果写回 messages
  → 再次 Prefill / Decode
  → 直到模型不再请求工具，返回最终内容

第七步：结果后处理
  → 格式化、安全过滤、转成代码 patch、结构化 JSON

第八步：返回给用户 + 写入日志/指标
  → 请求耗时、token 数量、错误码、模型版本、工具调用记录
```

整条链路可以写成：

```text
用户请求
  → 鉴权/路由
  → 上下文构建
  → 检索(RAG，可选)
  → 模型推理（Prefill → Decode）
  → [tool_call → 执行工具 → 回填结果 → 再推理]（可选循环）
  → 结果后处理
  → 返回用户
  → 日志/指标/追踪
```

所以说，**一个 AI 产品不只是「一个模型文件 + 一个聊天框」，背后是一套复杂的分布式系统。** 后面学 AI 工程，真正会接触到的正是这条链路上的某一段或几段。

应用层协议（messages / SSE / tool_call）的最小闭环见 [ai-engineering.md §1.6](ai-engineering.md)；延迟怎么观测见 [ai-engineering.md §3.4](ai-engineering.md)。

---

### 3.6  评估与质量：怎么知道「变好了」

> 延迟指标（TTFT/TPOT）证明「跑得动、能排障」；本节证明「答得对不对、忠不忠于资料」。二者不要混成一个「好不好」。TTFT/TPOT 口径见 [ai-engineering.md §3.4](ai-engineering.md)。

现代产品公式更完整的写法是：

**模型 + RAG + 工具 + Eval/观测**

没有 Eval，上了 RAG 也只是「感觉好一点」。

#### 幻觉可以分几类（方便定位）

| 类型 | 含义 | 常见对策方向 |
|------|------|-------------|
| 事实错误 | 与世界/常识不符 | 更大模型、检索、拒答策略 |
| 未忠于检索 | 资料在，答案却歪曲/编造 | 改 Prompt、强制引用、忠实度检查 |
| 过度推断 | 资料不足仍下结论 | 要求「不知则说不知」 |
| 归因错误 | 引用了错 chunk / 假来源 | 引用对齐、recall 检查 |

「上了 RAG 就没有幻觉」是误解（§3.3）；评估要分别测检索与生成。

#### 三张指标表（何时用哪个）

| 类别 | 例子 | 回答的问题 |
|------|------|-----------|
| 延迟 / 成本 | TTFT、TPOT、tokens、\$/请求 | 慢不慢、贵不贵 |
| 检索 | recall@k、命中标准 chunk | 找没找对资料 |
| 生成质量 | 引用准确率、忠实度、人工抽检 | 有没有胡编、有没有歪曲资料 |

RAGAS 等框架里常见 groundedness / context precision / answer relevance，对应的就是「忠于上下文 / 检索是否干净 / 答案是否切题」。入门不必上全套自动化 Judge；先把人工协议跑稳。

#### 最小评估集怎么建

1. 准备 **同一套** 10 个问题，每题标注「标准答案应来自哪几个 chunk」  
2. 含：应能答、应拒答/应说不知道、需要跨 chunk 的题  
3. 测：recall@3/5 + 引用是否对上 + 人工看是否忠实  
4. 再做 2–3 个故障注入：坏切块、top-k 过小、坏 Prompt，看指标是否按预期变差（归因）

动手步骤见 [ai-infra-project-plan.md Step 3.6](ai-infra-project-plan.md)。

---

## 第四章：产业、岗位与决策

这一章靠近「要不要转、往哪转、团队先投什么」。岗位薪资细节以 [ai-engineering.md §4](ai-engineering.md) 为附录；本文只保留判断框架。

---

### 4.1  为什么今天出现大量 AI 基础设施岗位

#### 以前的难点：把模型训出来

在早期，大问题主要是：模型怎么设计、模型怎么训练、准确率够不够高。

#### 今天的难点：把模型稳定地用起来

现在模型已经够强了，但企业落地时很快会遇到问题：GPU 昂贵、长上下文吃显存、高并发下延迟波动大、出问题不易定位、企业数据不在模型里、需要接外部工具。单纯「模型本体」已经不够了，必须有推理引擎、API 网关、缓存系统、调度系统、监控系统、发布系统。这整套东西，就叫 **AI 基础设施（AI Infrastructure）**。

> 详细的关键问题对照表和 6 层产品架构图见 [ai-engineering.md §3.1-3.2](ai-engineering.md)。

#### 对应岗位

| 工种 | 干什么 | 适合工程背景？ |
|------|--------|:---:|
| **推理部署工程师** | 把模型装到服务器上跑起来，搭环境、配服务、盯 QPS/延迟 | 是 |
| **ML Platform 工程师** | 做模型上线流水线平台，让算法工程师一键部署 | 是 |
| **MLOps / AI DevOps** | 模型从开发到上线的全流程自动化，自动测试→推模型→切流量→回滚 | 是 |
| **推理优化工程师** | 深入引擎内部提速：写 CUDA kernel、算子融合，需要 GPU 和 C++ 功底 | 门槛较高 |
| **AI 算法工程师** | 训练模型、设计架构、调参，造权重的人 | 否，非本指南方向 |

> 详细的岗位市场数据（薪资、技能需求、各城市/地区对比、搜索关键词）见 [ai-engineering.md §4](ai-engineering.md)。

---

### 4.2  DeepSeek 意味着什么

DeepSeek 的影响不限于模型榜单名次，在训练策略、架构压缩、开源协议和成本控制几方面都有代表性：

#### 正确理解

GPU 仍然关键；同时 DeepSeek 系列也说明：训练策略（如 R1 的 GRPO）、架构压缩（MLA → V4 的 CSA+HCA）、开源发布节奏与推理成本，都会改变竞争格局。公开讨论里常强调「训练成本相对同期闭源更低」，具体数字依赖各家披露口径，不宜单独当作行业通则。

常见误解是「没 GPU 也无所谓」。更稳妥的读法是：竞争从「主要拼资源」扩展到「资源 + 算法 + 工程 + 生态」，算力约束并未消失。

> 参考：[DeepSeek-R1 技术报告](https://arxiv.org/abs/2501.12948)；V4 参数与 KV 口径见 §2.5

---

### 4.3  今天的 AI 生态在争什么

模型能力仍是起点，但竞争不只停在榜单上。常见争夺面还包括：训练与推理成本、接到真实场景的速度、工具与开发者生态，以及企业环境里的稳定交付（可靠性、可运维）。芯片/算力、推理系统、产品交互会叠在同一条商业链上，强弱要分场景看，不宜压成单一「谁最大」。

---

### 4.4  学习顺序与项目路径

#### 认知落点

大模型是强的模式建模系统，离人类意识仍很远；配上工具、工作流和记忆后，在许多任务上已能当协作接口。工程上更常卡住的是：知识怎么接、工具怎么调、延迟与成本怎么控、故障怎么排、版本怎么发。价值多半在「把 AI 接进真实系统」，而不只在单次对话质量。

#### 学习路径

与文首一致：

1. 读完本文（至少 §1 + §3）
2. 再读 [ai-engineering.md](ai-engineering.md)：链路边界、Prefill/Decode、工种；§5 数据按需查
3. 动手看 [ai-infra-project-plan.md](ai-infra-project-plan.md)：Step 0→2C 为主；RAG / vLLM 按目标加减

目标是做产品而不是求职时，可先用商业 API 做出可演示闭环，再回头用 plan 的服务化与观测压成本、做私有化。

#### 项目路径选择

两条都成立：Python + HF Transformers（LLM 应用工程更常见），或 C + ONNX Runtime（边端/嵌入式）。当前 plan 选 Python。说明见 [ai-engineering.md §6](ai-engineering.md)。

---

### 4.5  决策卡：做产品 / 带队时先投什么

> plan 是**个人作品集手册**；带队或做产品时用下面两张卡，不要直接把 Step 0–4 当团队 roadmap。

#### A. 工种五格（统一术语）

| 名称 | 解决什么 | 最小证据举例 |
|------|----------|-------------|
| **AI 应用工程** | Prompt、RAG、工具调用、产品闭环 | 可演示垂直场景 + 引用/拒答策略 |
| **LLM 服务化** | 把模型变成可观测 API | OpenAI 兼容 + SSE + TTFT/TPOT |
| **ML Platform** | 上线流水线、多模型治理 | 灰度、回滚、配置与权限 |
| **推理部署** | GPU 上稳定跑 Serving | vLLM/SGLang + 队列/KV/显存 |
| **推理优化** | 极致吞吐与算子 | CUDA/量化算法；本套文档不主攻 |

同一技能在 JD 里可能被叫「AI Infra」或「平台后端」；先对表，再谈招人。

#### B. 成本 × 能力（独立开发 / 小团队）

| 场景 | 优先选 | 原因 |
|------|--------|------|
| 验证「有没有人用」 | 商业 API | 最快出 demo；先证明需求 |
| 高频简单任务、要控成本 | 本地 SLM / 小模型路由 | 见 engineering §5.7 |
| 私有知识、要可追溯 | RAG +（API 或本地）生成 | 先保证检索与引用，再抠模型大小 |
| 高并发、要自托管 | vLLM 等 Serving | 在 API 验证之后再上 |

**产品最短路径：** API 出可点的闭环 → 打点用量与成本 → 热点路径换 SLM/自托管 → 有私有知识再上 RAG。  
**求职最短路径：** 仍按 plan Step 0→2C（服务化可观测证据）。  
团队投资按本节决策卡取舍；plan 的 Step 编号只服务个人作品集，不要按 0→4 排团队排期。

#### C. Tech Lead 90 天取舍（极简）

1. **先**做「能观测的服务化」还是「能评估的 RAG」：有私有知识痛点 → RAG；要平台化接模型 → 服务化。  
2. **何时上 GPU/vLLM：** 延迟/成本/并发成为真实瓶颈，或客户要求私有化；此前用 API 即可。  
3. **招人 vs 转训：** 缺 Serving 深水区再招推理；多数团队先训现有后端做服务化 + 应用。  
4. **降级条件：** 个人 plan 的 Step 0（Ownership）若两周仍讲不清核心路径，团队试点同样应缩小 scope，而不是加 Agent。

PM 决策检查表（炒作 vs 现实）：能力够不够、成本能不能扛、延迟能不能接受、数据是否可控、要不要 Agent（默认先不要）。

---

## 附录  参考资料索引

### 学科与早期历史
- [IEEE：1956 AI 术语里程碑](https://ieeemilestones.ethw.org/Milestone-Proposal:Introduction_of_the_Term_%E2%80%98Artificial_Intelligence,%E2%80%99_1956)
- [Perceptron 1958](https://www.ling.upenn.edu/courses/cogs501/Rosenblatt1958.pdf)
- [Backpropagation 1986](https://www.nature.com/articles/323533a0)
- [LSTM 1997](https://direct.mit.edu/neco/article-abstract/9/8/1735/6109/Long-Short-Term-Memory)
- [CUDA 2006](https://www.cgw.com/Press-Center/News/2006/Nvidia-Introduces-CUDA-Architecture-for-Computin.aspx)

### 深度学习爆发
- [AlexNet 2012](https://proceedings.neurips.cc/paper_files/paper/2012/file/c399862d3b9d6b76c8436e924a68c45b-Paper.pdf)
- [word2vec 2013](https://arxiv.org/abs/1301.3781)
- [seq2seq 2014](https://papers.nips.cc/paper/5346-sequence-to-sequence-learning-with-neural-networks)
- [Bahdanau Attention 2014](https://arxiv.org/abs/1409.0473v7)
- [AlphaGo 2016](http://www.nature.com/articles/nature16961.pdf)

### Transformer 与大模型
- [Transformer 2017](https://proceedings.neurips.cc/paper_files/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf)
- [BERT 2018](https://arxiv.org/abs/1810.04805v1)
- [T5 2019](https://arxiv.org/abs/1910.10683v3)
- [The Bitter Lesson 2019](https://bitterlesson.ai/)
- [GPT-3 2020](http://arxiv.org/abs/2005.14165)
- [RAG 2020](https://arxiv.org/abs/2005.11401)
- [Foundation Models 2021](https://arxiv.org/abs/2108.07258)
- [LoRA 2021](http://arxiv.org/pdf/2106.09685)
- [InstructGPT / RLHF 2022](https://proceedings.neurips.cc/paper/2022/hash/b1efde53be364a73914f58805a001731-Abstract.html)
- [ChatGPT 2022](https://www.openai.com/blog/chatgpt/)

### 推理基础设施
- [FlashAttention 2022](https://papers.nips.cc/paper_files/paper/2022/hash/67d57c32e20fd0a7a302cb81d36e40d5-Abstract-Conference.html)
- [vLLM / PagedAttention 2023](https://arxiv.org/abs/2309.06180)
- [Meta Llama 2 2023](https://ai.meta.com/blog/llama-2)
- [NVIDIA Triton 文档](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html)

### 最新进展
- [DeepSeek-R1 2025](https://arxiv.org/abs/2501.12948)
- [DeepSeek-V4-Pro（Hugging Face 模型卡）](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro)；[技术报告 HTML](https://arxiv.org/html/2606.19348)
- [GPT-5 2025（OpenAI 官方）](https://openai.com/index/introducing-gpt-5/)
- [GPT-5 for developers（上下文等规格）](https://openai.com/index/introducing-gpt-5-for-developers/)
- [Qwen3 2025](https://qwen-ai.com/qwen-3/)
- [Llama 4 2025（Meta 官方）](https://ai.meta.com/blog/llama-4-multimodal-intelligence/)

### 推理系统与工程（2025-2026）
- [Transformers v5 官方博客 (2025.12)](https://huggingface.co/blog/transformers-v5) — 日 300 万+ pip 安装，400+ 架构，与 vLLM/SGLang/ORT 互操作
- [vLLM 官方文档](https://docs.vllm.ai/en/stable/) — OpenAI 兼容 API、Ray Serve 集成、Prometheus 指标
- [SGLang](https://github.com/sgl-project/sglang) — ~30.2k stars，RadixAttention，xAI Grok 选用
- [ONNX Runtime + KleidiAI (2025.5)](https://onnxruntime.ai/blogs/arm-microsoft-kleidiai) — ARM CPU 2.4× 推理加速
- [CNCF: K8s 与 AI (2026.1 公告 / 2026.3 解读)](https://www.cncf.io/announcements/2026/01/20/kubernetes-established-as-the-de-facto-operating-system-for-ai-as-production-use-hits-82-in-2025-cncf-annual-cloud-native-survey/) — 82% 容器用户生产使用 K8s；托管 GenAI 的组织中 66% 用 K8s 做部分或全部推理
- [EVAL #001: 推理引擎对比 (2026.3)](https://buttondown.com/ultradune/archive/eval-001-the-great-llm-inference-engine-showdown) — vLLM vs TGI vs TensorRT-LLM vs SGLang vs llama.cpp vs Ollama
- [FastAPI 2026: AI 服务化常用框架](https://www.programming-helper.com/tech/fastapi-2026-python-api-framework-ai-ml-adoption-enterprise) — 99.5k stars，38% Python 开发者采用
- [开源 LLM 推理引擎对比 2026](https://fish.audio/zh-CN/blog/open-source-llm-inference-engines-2026/) — SGLang vs vLLM vs MAX vs BentoML
- [SLM in Production 2026](https://dev.to/alexcloudstar/small-language-models-in-production-2026-where-slms-beat-frontier-models-and-where-they-quietly-3kn5) — 92.48% HF 下载为 1B 以下模型，Router 模式
- [AI 团队角色与人员配置 2026](https://www.e2enetworks.com/blog/ai-production-teams-2026) — 4 大核心角色：应用/基础设施/评估/数据
- [Open Responses 规范 (2026.1)](https://simonwillison.net/2026/jan/15/open-responses/) — 基于 OpenAI Responses API 的厂商中立 JSON API 规范；发起方含 OpenRouter、HF、LM Studio、vLLM、Ollama、Vercel 等（查询 2026-07-17）
- [PyPI transformers v5.14.1](https://pypi.org/project/transformers/) — 查询 2026-07-17
- [PyPI sglang v0.5.15.post1](https://pypi.org/project/sglang/) — 查询 2026-07-17；第三方「亿级周下载」口径不采信
- [TGI 维护模式 PR #3344 (2025-12-11)](https://github.com/huggingface/text-generation-inference/pull/3344)；仓库 2026-03-21 归档

