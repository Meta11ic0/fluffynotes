# AI 工程入门：从打字到 AI Infra

> 生成日期：2026-07-14 | 状态：v2.2（第二波：协议/观测/评估/决策） | 基于 2026 年中实际数据

---

## 本文定位

这份文档写给**已经读过（或正在并行浏览）[ai-intro.md](ai-intro.md)、有编程基础、想把「请求 → 模型回复」拆开看清楚的人**。比 intro 更深：机制、边界、工种与技术栈数据。

读完应能回答：

1. 输入的那行字经历了什么变成回复
2. LLM 内部如何工作：Attention、FFN、Prefill、Decode 的核心直觉
3. AI Infra 大致管什么、和应用工程差在哪
4. 有哪些相关工种、各自要什么技能
5. 2026 年常见技术栈与区域岗位参考数据（§5 / §4，可当附录）

**同目录职责（唯一读序）：**

| 顺序 | 文档 | 职责 |
|:---:|------|------|
| 1 | [ai-intro.md](ai-intro.md) | 认知地图 |
| 2 | **本文** | 工程深读 |
| 3 | [ai-infra-project-plan.md](ai-infra-project-plan.md) | 动手（LLM 服务化主线；纯 Infra 为加分） |

**主线一句话：** AI 应用工程 + 轻量推理概念；CUDA / 多卡调度是支线。

**全文约 75-90 分钟（可跳过 §4–§5）。**

**各章建议：**
- §1 链路与边界：必读；前端必读 **§1.6**
- §2 机制：想搞懂 Prefill/Decode / KV 的人必读；只做产品决策可只读 §2.4–2.5
- §3 为什么难运维 + **§3.4 观测**：建议读；DevOps 必读 §3.4
- §4 岗位市场：求职附录；做产品可跳
- §5 技术栈数据：权威数字出处，按需查
- §6 学习路径：读完前三章后看

**跳读捷径：**
- 已会 RAG → 直接 §1 + §2.4–2.5 → plan Step 2 / Step 4
- 前端/全栈 → §1 + **§1.6** + §2.4–2.5 → plan Step 2
- DevOps → §1.3 + §2.4 + **§3.4** → plan Step 2B
- 关心评估 → [ai-intro.md §3.6](ai-intro.md) → plan Step 3.6

---

## 1. AI 工具，底层在做什么

### 1.1 一条聊天消息的完整旅程

在网页上给 ChatGPT 发一句话："这段代码什么意思"。从按下发送键到看到回复，经历了这些：

```
你的浏览器
  │ （把文本打包成 HTTP 请求，发到服务器）
  │
  ▼
后端服务
  │ ── 通用后端逻辑：检查你是谁、有没有额度、有没有被封
  │    这部分不是 AI Infra，只是普通 Web 后台
  │
  │ ═══════════════ AI Infra 管辖范围 — 从这里开始 ═══════════════
  │
  │ ── 分词（Tokenize）：把"这段代码什么意思"切成 token 编号
  │    大模型不认识汉字，只认数字编号
  │
  │ ── 拼装 Prompt：系统提示词 + 历史对话 + 你的问题 → 完整 token 序列
  │
  │ ── 转发给推理服务器
  │
  ▼
推理服务器 —— 一台插着 GPU 的机器                          ← AI Infra 核心
  │ ── 加载模型：从硬盘读模型文件（几十到几百 GB）到 GPU 显存
  │ ── 推理计算（Prefill）：token 序列 → 几十层矩阵乘法 → 输出
  │ ── 循环生成（Decode）：每生成一个新 token，拼到末尾再跑一轮
  │    过程中需要 KV Cache 记住已生成的全部内容
  │ ── 解码（Detokenize）：token 编号 → 可读文字
  │
  ▼
生成的文字逐字推回浏览器（看到字一个一个蹦出来）
```

**收束：** 看似只是打字和收到回复。中间真正吃 GPU、管 KV、跑 Prefill/Decode 的那一段，才是 AI Infra / Serving 的核心；鉴权与普通限流是通用后端。拼 Prompt、工具调用更常算应用工程（见 §1.3 三档边界）。

### 1.2 ChatGPT / OpenCode / Cursor：三条链路对比

可同时用 ChatGPT 网页版、OpenCode 终端、Cursor IDE。从 AI Infra 的视角看，它们的链路有本质区别。

#### ChatGPT 网页版：全托管

```
你的浏览器 → OpenAI 后端（鉴权+分词+拼Prompt）
  → OpenAI 推理集群（插满 GPU，跑 GPT 模型）
  → 逐字返回
```

→ AI Infra 全在 OpenAI 机房。模型、GPU、推理引擎全是 OpenAI 的。用户只是消费者。

#### OpenCode 终端：客户端 + 工具链

```
你的电脑
  │ OpenCode CLI 在本机运行，这是个客户端程序
  │ ── 读代码库、跑 shell 命令、管理文件
  │ ── 把你的问题 + 代码上下文 + 工具结果拼成 Prompt
  │
  │ HTTP/SSE 请求（发到 Anthropic/DeepSeek 的服务器）
  ▼
API 网关 → 推理服务器（别人的 Claude / DeepSeek 模型）
  │
  ▼
OpenCode CLI 收到回复 → 解析工具调用 → 执行工具 → 显示结果
```

→ AI Infra 部分（推理服务器、GPU、模型）是远程的、别人管。本地跑的是客户端 + 工具链。OpenCode 团队是"做工具"的人，不是"做推理"的人。

#### Cursor IDE：IDE + 模型接入

```
你的电脑
  │ Cursor 客户端（Electron App，VSCode 改的）
  │ ── 代码索引（语法树、LSP）
  │ ── 上下文收集（打开的文件、光标位置、选中代码）
  │
  │ HTTP 请求
  ▼
Cursor 后端服务器（Cursor 公司自己的）
  │ ── 鉴权/计费/订阅管理
  │ ── 模型路由：根据订阅等级决定调哪个模型、哪家 API
  │
  ▼
推理 API（Anthropic / OpenAI 的推理服务器）→ 返回结果
  │
  ▼
Cursor 客户端展示（内联补全 / Chat 面板）
```

→ 这张图描述的是一种典型链路：客户端和产品后端负责索引、上下文与路由，模型推理由远程服务完成。它不用于断言某家公司是否自建了模型或推理能力。

#### 一张表看清三条链路

| | **ChatGPT 网页版** | **OpenCode** | **Cursor** |
|---|---|---|---|
| **AI Infra 在哪** | 全在 OpenAI 机房 | 远程（API 提供商） | 远程（API 提供商） |
| **使用者本地跑什么** | 浏览器 | CLI + 工具链 | IDE + 代码索引 |
| **普通使用者直接接触 Serving 吗** | 通常不接触 | 通常不接触 | 通常不接触 |
| **开发商角色** | 做推理 + 做产品 | 做产品，调别人推理 | 做产品，调别人推理 |

这张表用于区分两类关注点：
- 做 OpenCode / Cursor 这类工具的 → "AI 应用开发"
- 做推理服务器、模型部署、GPU 算力管理的 → "AI Infra"
- 两者工作重点不同，但在网关、路由、观测等位置可能由同一团队共同负责。

### 1.3 AI Infra 边界：核心 / 常共担 / 通常不算

业界没有唯一标准，面试与分工时用三档比二元 ✅❌ 更稳：

```
核心（几乎总是 AI Infra / Serving）:
   - 推理引擎与推理框架（加载模型、矩阵计算、KV Cache、Continuous Batching、多卡）
   - GPU / 显存调度与模型版本灰度
   - TTFT / TPOT / tokens/s、排队与 KV 占用等推理指标

常共担（平台后端与 Infra 都会碰到）:
   - 分词 / 解码（可在网关，也可在推理进程）
   - SSE / 流式协议（应用要会消费；Serving 要会正确吐出）
   - 按 token / 并发槽的限流与路由（比 Nginx QPS 限流更贴 LLM）

通常不算 AI Infra（通用后端 / 应用）:
   - 客户端 UI / IDE
   - 拼 Prompt / 上下文收集 / 工具调用编排
   - 账号鉴权、会话、普通计费账单
```

**产品链路 ≠ Infra 边界：** intro 里「从请求到回答」会写鉴权与拼 Prompt，那是产品全链路；Infra 管的是其中推理与资源那一段。

### 1.4 五个核心零件

把整条链路拆成五个独立零件，就能看清谁属于 AI Infra、谁不属于：

| 零件 | 是什么 | 类比 | 是 AI Infra 吗 |
|------|--------|------|:---:|
| **模型文件** | 超大文件（几 MB 到几百 GB），存着一堆浮点数 | 电子书（数据） | ✅ |
| **推理引擎** | 程序库，加载模型文件 + 执行矩阵计算。如 ONNX Runtime、TensorRT | 阅读器 App | ✅ |
| **推理框架** | 推理引擎 + KV Cache 管理 + 请求调度 + 多卡并行。如 vLLM、SGLang | 阅读器 Plus（能做笔记、翻译） | ✅ |
| **推理服务器** | 7×24 运行、绑定端口、接收网络请求的服务程序 | 便利店店员 | ✅ |
| **客户端** | 用户打字的界面（网页、App、IDE） | 外卖 App | ❌ |

**推理引擎 ≠ 推理服务器。** 引擎只管"跑计算"，服务器要管"开门迎客"（收钱、记账、开门关门）。这两个概念常被混淆，记住它们的区别很重要。

**KV Cache 的作用。** 大模型逐 token 生成时，每一步都要看到「前面已经生成了什么」。KV Cache 缓存已算过的 K/V，避免每步重算。它很占显存：单请求常见几十到几百 MB；高并发合计才容易到十几、几十 GB。推理框架（vLLM 等）的核心工作之一就是管好这块内存。机制与公式见 §2.4。

### 1.5 纠正一个常见误解：AI ≠ if-else + 数据库

> "AI 是不是就是程序接一个数据库，用户问 A 就去数据库查相关答案？"

**不是。** 规则系统的知识在人写的 if-else 里，没写到的就不会。现代 AI 是**学习系统**。知识在模型的参数（几亿到几万亿个训练出来的浮点数）里，遇到没见过的情况也可能泛化应对。

上述"接数据库查答案再回答"更接近 **RAG**（检索增强生成）。这是大模型时代弥补模型局限（知识过时、可能幻觉）的增强方式，不是 AI 的基础原理。

> 规则系统的知识在 if-else 里，学习系统的知识在模型参数中。这是 AI 与经典编程的根本区别。详见 [ai-intro.md §1.3](ai-intro.md)。

### 1.6 应用协议最小闭环：messages → SSE → tool_call

本节只给出实现所需的最小协议模型。不同服务对字段的支持并不完全相同；机制细节见 §2.4，动手任务见 [项目计划 Step 2](ai-infra-project-plan.md)。

#### messages：Prompt 不是一串纯文本

OpenAI 兼容接口里，输入是一组带角色的消息：

```json
{
  "model": "qwen3-0.6b",
  "stream": true,
  "messages": [
    {"role": "system", "content": "你是严谨的代码助手。"},
    {"role": "user", "content": "解释这段函数"},
    {"role": "assistant", "content": "……上一轮回复……"},
    {"role": "user", "content": "再指出空指针风险"}
  ]
}
```

服务端通常会：拼 system + 历史 + 本轮 user（可选再塞入 RAG 片段）→ `apply_chat_template` → token 序列 → Prefill → Decode。  
**拼 Prompt / 上下文收集算应用工程**（§1.3）；真正吃 GPU 的 Prefill/Decode 才是 Serving 核心。

如果允许工具调用，请求还会包含 `tools`。每个工具用 JSON Schema 描述名称、用途和参数：

```json
{
  "tools": [{
    "type": "function",
    "function": {
      "name": "read_file",
      "description": "读取指定文件",
      "parameters": {
        "type": "object",
        "properties": {"path": {"type": "string"}},
        "required": ["path"]
      }
    }
  }]
}
```

#### SSE：事件、网络 chunk 和 token 不是一回事

`stream: true` 时，响应通常使用 `text/event-stream`。SSE 以**空行**划分事件；一个事件可以包含多行 `data:`。网络层收到的 TCP/HTTP chunk 可能只含半个事件，也可能含多个事件，因此客户端需要先缓冲并解析完整事件，不能把每次 `read()` 或每一行直接当成一个 token。

```text
data: {"id":"chatcmpl-…","choices":[{"delta":{"content":"你"},"index":0}]}

data: {"id":"chatcmpl-…","choices":[{"delta":{"content":"好"},"index":0}]}

data: [DONE]
```

`delta.content` 是服务端解码并封装后的一小段文本，可能对应零个、一个或多个模型 token。首个 SSE 事件也可能只有 `role` 或元数据。浏览器看到的首包时间和包间隔只能近似用户体验，不能直接替代服务端按 token 时间戳计算的 **TTFT/TPOT**。

非流式则等整段生成完，一次返回完整 `message.content`。学习/排障时两种都要会。

#### tool_call：字段关联与状态循环

工具调用不是魔法，是多轮 messages：

```
1. user →「修复空指针」
2. assistant → 输出 tool_calls（包含 id、name、arguments）
3. 客户端/运行时执行工具，得到结果
4. 追加 role=tool 的消息，并用 tool_call_id 关联原调用
5. assistant → 基于工具结果继续写，或再调工具，直到结束
```

非流式响应的关键结构可以简化为：

```json
{
  "tool_calls": [{
    "id": "call_123",
    "type": "function",
    "function": {
      "name": "read_file",
      "arguments": "{\"path\":\"src/app.py\"}"
    }
  }]
}
```

工具执行后回填：

```json
{"role": "tool", "tool_call_id": "call_123", "content": "工具结果"}
```

流式响应里，`name` 和 `arguments` 可能分多个 delta 到达。运行时必须按 `tool_calls[index]` 或调用 ID 累积参数，确认 `finish_reason` 后再解析 JSON 和执行工具。还要设置最大循环次数，处理工具异常、客户端取消与断连；取消后应尽量把信号传到生成任务，避免客户端已经离开而服务端继续耗算力。

这一循环可以概括为：

```text
messages += user
  → Prefill / Decode
  → 有 tool_calls：累积参数 → 执行 → messages += tool 结果 → 再推理
  → 无 tool_calls：输出最终回复
```

协议层决定模型收到什么，推理层决定生成的延迟与资源消耗。

---

## 2. LLM 内部如何工作

> 上一章追踪了一条消息从浏览器到推理服务器的完整旅程，其中提到了分词（Tokenize）、推理计算、KV Cache、解码等环节。这一章拆开每个环节，看具体是怎么算的。本章是全文最难的部分，会引入约 30 个新术语，涉及矩阵乘法和形状追踪。第一次接触的话，建议先快速扫一遍建立印象，不用卡在某个公式上，读完 §3-§6 后回来看会有新理解。如果完全不关心数学细节，看 [ai-intro.md §1.4](ai-intro.md) 的纯文字版即可。
>
> **读完本章可回答的问题：** 为什么大模型回答越长越贵？KV Cache 为什么是系统瓶颈？为什么同样是 LLM，有的擅长写代码有的擅长聊天？这些都是 AI Infra 面试的核心问题。
>
> 这一章从"我喜欢吃红烧肉"这一句话出发，追踪它如何变成 token、如何在 Attention 中让每个字互相"看到"、如何在 FFN 中被深度加工、如何在 Prefill 中一次过完全部层、如何在 Decode 中一个字一个字蹦出来。
>
> 不要求数学背景，只需要理解"矩阵乘法 = 对应乘再求和"这个基本操作。每一步都标明形状变化。

### 2.1 分词 → 向量 → 位置

**第一步：分词（Tokenization）。** 模型不读汉字，只认数字。分词器把文本切成 token（最小处理单位），然后查词表映射成编号。

```
"我喜欢吃红烧肉"
    ↓ 切分
[我, 喜欢, 吃, 红烧肉]          ← 4 个 token（字符串）
    ↓ 查词表
[8247, 15302, 901, 43891]       ← 4 个词表编号（整数）
```

- Token 不一定等于"一个字"，"红烧肉"可能是一个 token，也可能被切成"红 + 烧 + 肉"
- 收费按 token 数算，速度按 tokens/s 算
- 看到的"蹦字"，底层是"蹦 token"

**第二步：变向量（Embedding）。** 每个词表编号被映射成一个固定长度的浮点数数组（向量）。语义接近的词，向量也接近。比如"猫"和"狗"的向量距离，就比"猫"和"电脑"近。

实际模型的向量维度很大（Llama-3-8B 用 d=4096），但原理一样。每个 token 变成 d 个浮点数。

**第三步：准备位置信息。** 仅有词向量无法区分词序。"我 喜欢 吃 红烧肉"和"红烧肉 吃 喜欢 我"使用相同的一组 token，但顺序不同。主流 decoder-only 模型常用 RoPE（旋转位置编码）：它通常不直接修改输入 embedding，而是在各层 Attention 中按位置旋转 Q/K，把相对位置信息带入打分。

4 个 token、每个 d 维，按顺序摞起来，这就是**输入矩阵 X[4×d]**，准备进入模型。

### 2.2 Attention：token 之间互相"看"

**为什么需要 Attention？** 每个 token 的向量目前只有自己的孤立含义。"红烧肉"只知道自己是"红烧肉"，不知道前面有"喜欢"在修饰它、有"吃"在描述动作。Attention 让每个 token **看到并加权利用其他 token 的信息**。

Attention 的完整计算分六步。全程输入 X[4×d]，输出形状不变 [4×d]，但每个向量已经融合了上下文。

> **先理解一个核心概念：权重矩阵。** 后面将看到 WQ、WK、WV、WO 这些字母，它们是训练阶段学出来的数字矩阵（§2.6 会讲怎么训练的）。可将每个 W 理解为一套"映射规则"：输入一个向量，输出另一个向量，不同的 W 代表不同的映射角度。不需要知道矩阵里的具体数字，只需要知道它们存在、而且所有 token 共享同一套 W。

---

**第 1 步：三组投影。** `X[4×d] × WQ/WK/WV[d×d] → Q[4×d], K[4×d], V[4×d]`

把同一个输入 X 分别乘以三个不同的权重矩阵，得到三个不同的"视角"：

| | 角色 | 类比 |
|---|---|---|
| **Q（Query）** | "这个 token 需要什么信息" | 搜索框里打的词 |
| **K（Key）** | "这个 token 能提供什么" | 每篇文章的标题 |
| **V（Value）** | "这个 token 的实际内容" | 文章正文 |

**第 2 步：打分。** 单头简化写法是 `Q × Kᵀ ÷ sqrt(d_head) → scores[4×4]`

Q 的每一行（某个 token "要什么"）跟 K 的每一列（某个 token "有什么"）做点积。分数越高，匹配越强。实际缩放项使用每个注意力头的维度 `d_head`；本节其余形状仍用 `d` 做教学简化。

scores[4×4] 的含义变了：行列不再是 token×维度，而是 **token×token**（每对 token 之间的相关度）。

**第 3 步：因果掩码。** scores 右上角全设成 -∞。因为语言模型是"根据前面猜下一个"，当前 token 不能偷看未来 token。`-∞` 过 softmax 后变成 0，彻底屏蔽。

```
加 mask 后的 scores（示意）：
              K("我") K("喜欢") K("吃") K("红烧肉")
Q("我")     [  0.9     -∞       -∞       -∞     ]  ← 只能看自己
Q("喜欢")   [  0.3     0.8      -∞       -∞     ]  ← 能看"我"和自己
Q("吃")     [  0.2     0.4      0.7      -∞     ]  ← 能看前三个
Q("红烧肉") [  0.1     0.2      0.5      0.8    ]  ← 能看全部
```

**第 4 步：softmax。** 每行做 softmax，变成「注意力比例」。下面数字是**教学示意**（便于读出「每行和为 1」），不要拿去验算成精确 softmax 结果。

```
weights（注意力权重，示意）：
              "我"   "喜欢"  "吃"  "红烧肉"
"我"        [ 1.00   0.00   0.00   0.00 ]  ← 只看自己
"喜欢"      [ 0.38   0.62   0.00   0.00 ]  ← 38% 给"我"，62% 给自己
"吃"        [ 0.27   0.33   0.40   0.00 ]
"红烧肉"    [ 0.20   0.22   0.27   0.31 ]  ← 相对均匀关注前文
```

**第 5 步：加权汇总 V。** `weights[4×4] × V[4×d] → context[4×d]`

每个 token 的新向量 = 按注意力权重，对所有 token 的 V 加权求和。

```
context("红烧肉") = 0.20×V("我") + 0.22×V("喜欢") + 0.27×V("吃") + 0.31×V("红烧肉")
```

"红烧肉"从孤立的食物概念，变成了"有人喜欢吃的食物"的意思。

**第 6 步：输出投影。** `context[4×d] × WO[d×d] → attn_out[4×d]`

WO 把 context 变换回主干网络能读懂的格式。

**残差连接：** `attn_out + X → hidden[4×d]`。Attention 只需学"需要修改什么"，原始信息通过加法原样保留。这让深层网络训练稳定，梯度有一条不经过矩阵乘法的直通路径。

---

**Attention 全程形状汇总：**

```
X[4×d]（进来：4个token，每个d维）
  ↓ × WQ/WK/WV
Q[4×d]  K[4×d]  V[4×d]          ← 还是 token×dim
  ↓ Q × Kᵀ ÷ sqrt(d_head)
scores[4×4]（token×token，含义变了！）
  ↓ 因果 mask + softmax
weights[4×4]（每行和为1）
  ↓ × V
context[4×d]（token×dim，含义变回来）
  ↓ × WO + X（残差）
hidden[4×d]（出去，传给 FFN）
```

**进来 X[4×d]，出去 hidden[4×d]。形状不变，但每个向量已经融合了上下文信息。**

> **多头 Attention（Multi-Head）：** 上面是单头简化版。实际模型在同一层内并行运行多套 WQ/WK/WV（每套叫一个"头"），各头在更低维度独立做 Attention，最后拼接。不同头自然学会关注不同东西：一个偏句法、一个偏指代、一个偏情感。
>
> **GQA（分组查询注意力，Llama 3 等）：** Q 仍有多头，但多组 Q 共享同一组 K/V。推理时 KV Cache 按 K/V 头数存，能省约 4 倍显存。

### 2.3 FFN：深度加工每个 token

Attention 让 token 之间通信。对一组已经确定的注意力权重，输出是 V 的加权和；但这些权重由输入经 softmax 动态产生，所以 Attention 整体不是线性函数。FFN（前馈网络）随后对每个 token **各自独立**做密集计算。

FFN 里存着大量「可被检索式激活」的模式（流行说法常把 FFN 比作知识库）。更准确地说：事实与技能是分布式写在整网权重里的，FFN 往往贡献了很大一部分参数容量；不要理解成「查 FFN = 查一张百科表」。Attention 负责在 token 之间传信号，FFN 负责对单个 token 做深度非线性变换。

主流模型（Llama、Qwen、DeepSeek）用 **SwiGLU FFN**，对同一份输入走两条并行路径再汇合：

```
hidden[5]（来自 Attention，假设 d=5）
  ├─ × W_gate[5×10] → gate_raw[10] → silu → gate[10]    ← 门控路径："每个维度放行多少"
  └─ × W_up[5×10]   → up[10]                             ← 值路径："每个维度携带什么内容"
gate[10] ⊙ up[10] → mid[10]                               ← 逐元素乘（非线性在这里！）
  ↓ × W_down[10×5]
out[5]（压缩回 5 维）
  ↓ + hidden[5]（残差）
新 hidden[5]（传下一层）
```

**核心直觉：膨胀 → 门控 → 压缩。**

- **膨胀**（5→10）：给模型足够空间表达复杂模式
- **门控**（silu(gate) × up）：silu 让负值被衰减而非直接归零，gate 决定每个维度放行 up 里的多少内容。这是**非线性的关键**。没有这一步，多层矩阵乘可以合并成一层，深度就失去意义
- **压缩**（10→5）：压回模型主干维度

> **MoE（混合专家）替换的是 FFN 这块。** 标准 FFN 对所有 token 用同一套权重；MoE 准备多份"专家"权重，由路由器根据 token 内容决定分配给哪几个专家。总参数量可以极大（DeepSeek V4 1.6T），但每次只激活少量（49B，约 3%）。

### 2.4 Prefill 与 Decode：一次输入，逐字输出

一层 Attention+FFN 是一个 Transformer 层。实际模型堆几十到上百层（Llama-3-8B 有 32 层）。这 4 个 token 如何过完全部层、产出第一个输出字？之后又如何一个字一个字蹦出来？

#### Prefill（预填充）：输入一次过完所有层，产出第 1 个 token

```
输入: X₀[4×d]（"我喜欢吃红烧肉"）
KV Cache: 空

第 1 层: Attention（4个token互相看）→ 💾存K/V[4×d] → FFN → X₁[4×d]
第 2 层: Attention → 💾存K/V → FFN → X₂[4×d]
...
第 32 层: Attention → 💾存K/V → FFN → X₃₂[4×d]

末尾采样:
  取最后一个位置（"红烧肉"）的向量
  → × lm_head[d×vocab_size] → vocab_size 个分数
  → softmax → 概率分布
  → 按策略挑一个 → 第 1 个输出 token，比如"真"
```

**Prefill 的特征是：** 所有输入 token 同时处理（矩阵并行）。计算密集型，时间花在矩阵乘法上。全程只产出 1 个 token。

> **为什么只取最后一个位置？** Prefill 时每个位置都算了 Attention+FFN，但推理接龙只需要"整句输入之后，下一个字是什么"，所以只对最后一个位置做 lm_head + 采样。前面位置的 K/V 已经存进 Cache，供后续 Decode 复用。
>
> **为什么只存 K/V，不存 Q？** Decode 每步只算 1 个新 token，只有这个新 token 需要"问"（产生 Q）。历史 token 的 Q 已经用过，不会再被问到。但历史 token 的 K 和 V 每次都要被新 token 查询，所以 K/V 必须缓存，Q 不用。

#### Decode（解码循环）：一个字一个字蹦

```
Prefill 产出 "真"（词表编号 21056）

Decode 第 1 步:
  "真" → Embedding → X_new[1×d]（只有 1 个 token！）

  第 1 层:
    Q = X_new[1×d] × WQ → Q[1×d]        ← 只有 1 行
    K = X_new[1×d] × WK → K[1×d]
    V = X_new[1×d] × WV → V[1×d]
    💾 读Cache: K_hist[4×d], V_hist[4×d]（Prefill 存的）
    拼接: K_full[5×d], V_full[5×d]       ← 追加，不重写
    Q[1×d] × K_fullᵀ → scores[1×5]       ← 1个新Q 跟 5个K 打分
    softmax → × V_full → context[1×d]
    × WO + X_new → hidden[1×d]
    FFN → [1×d]

  第 2-32 层: 同样流程，各自读Cache、追加、输出
  末尾采样 → 第 2 个输出 token "好"

Decode 第 2 步: "好" → Embedding → ... → 第 3 个 token
...重复直到产出 <eos> 或达到长度上限
```

**Prefill vs Decode 对比：**

| | Prefill | Decode（每一步） |
|---|---|---|
| **输入** | 矩阵 [N×d]（所有 token 同时进） | 向量 [1×d]（1 个新 token） |
| **K/V** | 当场全量计算 | 新 token 自己算，历史从 Cache 读 |
| **因果 mask** | 需要 | 不需要（新 token 天然在最末） |
| **并行度** | 高（所有 token 同时算） | 低（只有 1 个 token） |
| **瓶颈** | **计算密集型**（大矩阵乘法） | **常受内存带宽限制**（反复读取模型权重，并读取历史 KV Cache） |
| **产出** | 第 1 个 token | 每步 1 个 token |

**为什么输出必须串行？** 第 N 个 token 依赖第 N-1 个 token，不知道上一个字是什么就算不出下一个。这叫**自回归（Autoregressive）**，本质是超大号 Autocomplete。

#### KV Cache 到底占多大

```
输入 token 数 N = 1000
层数 L = 50（简化估算，未扣除 GQA 压缩）
每层 KV 维度 d_kv = 1024

KV Cache = L × N × d_kv × 2(K+V) × 2字节(FP16)
         = 50 × 1000 × 1024 × 2 × 2
         ≈ 200 MB（一个请求）
```

100 个用户同时来 ≈ **20 GB，光 KV Cache，不算权重。** 这就是为什么 vLLM 的 PagedAttention 重要：它借鉴操作系统分页，按块管理 KV，把**因静态预分配造成的 KV 显存浪费**从大约 20–40% 压到很低（论文口径常写成「利用率接近 100%」）。这不是说整卡 GPU 算力利用率变成满分；模型权重仍然占显存大头。

#### Continuous Batching（连续批处理）一句话

静态 batching：凑齐一批请求才一起跑，短请求要等长请求，GPU 常闲着。Continuous Batching：请求随时进队，Decode 步里动态组批，做完的请求立刻腾出槽位。它和 Prefill/Decode、KV 分页绑在一起：没有高效的 KV 管理，动态组批也撑不起高并发。细节放到 [ai-infra-project-plan.md Step 4](ai-infra-project-plan.md) 概念笔记；这里只要建立「吞吐主要靠调度，不只靠更大的卡」即可。

### 2.5 采样：分数变文字

所有层走完后，取最后一个位置的向量 [1×d]，乘 `lm_head[d×vocab_size]` 得到 vocab_size 个分数（logits），再 softmax 变成概率分布，最后按策略挑一个 token。

- **贪心（Greedy）：** 永远选概率最高的，稳定但死板
- **Temperature：** 采样前先除以温度值。>1 更随机（创意），<1 更保守
- **Top-p：** 按概率从高到低累加，取累计概率达到阈值 `p` 的最小候选集合，再从中采样

挑出的 token 编号通过分词器查表还原成可读文字（Detokenize），然后通过 SSE 流式推给客户端，看到字一个一个蹦出来。

### 2.6 权重怎么来的：训练四阶段

上面讲的 Attention、FFN、Prefill、Decode 是推理（模型已经在跑）。但那些权重矩阵（WQ、WK、WV、WO、W_gate、W_up、W_down、lm_head）里的数字是怎么来的？答案是**训练**。

> 日常接触到的几乎都是推理，不是训练。但理解训练的基本流程，才能回答面试中"模型怎么来的""为什么不同模型能力不同"这类问题。

训练分四个阶段，从零到能对话：

#### 阶段 1：预训练（Pre-training）：学语言

**任务：** 海量文本上猜下一个字。若你做过传统 ML：这就是词表规模上的多分类，loss 通常是交叉熵（和分类 CE 同构），只是类别数变成整张词表。

```
随机抽一段文本 → 给前面 N 个字让模型猜第 N+1 个字
  → 对比正确答案 → 猜错了
  → 反向传播：从最后一层往回，精确算出每个权重对猜错贡献了多少
  → 微调所有权重（贡献大的多拧，贡献小的少拧）
  → 重复万亿次
```

- **数据：** 维基百科、书籍、代码仓库、新闻网页……数万亿 token。这些数据**不进模型文件**，训练跑完后，文件里只有训练产出的数字（权重）
- **成本：** 成百上千张 GPU、几周到几个月、几百万到几千万美元
- **产出：** Base Model，会接话但不会对话。问"这段代码什么意思"，它可能继续往下写代码而不是解释，因为它只学会了"猜下一个字"，还没学会"回答问题的格式"

#### 阶段 2：SFT（监督微调）：学对话格式

用人工写的"问题→标准答案"对子（通常 1 千到 5 万条）继续训练：

```
输入: "请解释这段代码的功能"
目标输出: "这段代码实现了一个二分查找算法，时间复杂度 O(logn)..."
```

这一步后模型学会了"用户问问题→模型回答"的格式，但风格可能还很生硬。

#### 阶段 3：偏好对齐（RLHF / DPO）：学风格和品味

同一个问题给多个回答，人类标注哪个更好。模型学习"为什么 A 优于 B"，不再是简单对错判断。DPO（直接偏好优化）是近年常见做法：可跳过单独训练奖励模型，直接从偏好数据里学。

#### 阶段 4：推理增强（GRPO / RLVR）：学思考

DeepSeek-R1 用的方法。用数学题答案、代码运行结果等**可验证的奖励信号**，让模型在试错中自己学会推理，没人逐步教它"怎么思考"，强化学习过程中可能涌现出更长的中间推理链。

#### 训练阶段一览

| 阶段 | 做什么 | 数据规模 | 本质 |
|------|--------|----------|:---:|
| **预训练** | 猜下一个字 → 反向传播微调 | 数万亿 token | ✅ 错了往对调 |
| **SFT** | 模仿人工标注的问答对 | 1K~50K 条 | ✅ 有标准答案 |
| **偏好对齐** | 人类偏好排序，学"哪个更好" | 偏好对数据 | ⚠️ 不是对错，是风格 |
| **推理增强** | 可验证奖励自学推理（数学题、代码运行） | 可验证题目 | ⚠️ 无标准答案，自己试 |

#### 模型文件里到底有什么

| 文件 | 内容 | 占比 |
|------|------|------|
| **权重文件** | 训练产出的所有浮点数（WQ/WK/WV/WO/W_gate/W_up/W_down/lm_head……） | 99% |
| **config** | 架构说明书：多少层、每层多大、Attention 类型 | 极小 |
| **tokenizer** | 字↔编号对照表，训练时绑定 | 几 MB |

**关键结论：** 文件 100 GB ≠ 训练了 100 GB 语料。文件大是因为参数个数多，不是因为塞进了训练数据。部署的模型文件里**只有权重、config、tokenizer**，没有原始语料。

#### 各主流模型对比（2026 年公开信息）

| | GPT-5 | Llama 4 Maverick | DeepSeek V4-Pro | Qwen3.6-35B-A3B | Claude Opus 4.6 |
|---|---|---|---|---|---|
| **架构公开？** | ❌ | ✅ | ✅ | ✅ | ❌ |
| **MoE？** | 推测有 | ✅ 128 路由专家 + 1 共享 | ✅ 384 路由 + 1 共享（top-6 激活） | ✅ 256 专家（8 路由 + 1 共享） | 推测有 |
| **总参数** | 未公开 | ~400B | **1.6T** | 35B | 未公开 |
| **每次激活** | 未公开 | **17B** | **49B** | **3B** | 未公开 |
| **上下文（标称）** | ~400K；GPT-5.5 约 1M | 1M | 1M | 262K 原生，可扩展至约 1M | 1M |

> 数据来源：各厂商 model card / 技术博客（截至 2026 年中）。GPT-5、Claude Opus 4.6 架构与总参数量官方未公开。

---

## 3. 为什么今天出现大量 AI Infra 岗位

### 3.1 从"把模型训出来"到"把模型稳定用起来"

以前的难点：模型怎么设计、怎么训练、准确率够不够高。

今天的难点：**模型已经够强了，但企业落地时很快会遇到问题：**

| 问题 | 具体表现 |
|------|---------|
| **太贵** | GPU 昂贵，长上下文吃显存，用户量上来成本飙升 |
| **太慢** | 响应延迟高，高并发排队严重，尾延迟（P95/P99，即 95%/99% 用户的最差响应时间）波动大 |
| **难运维** | 出问题不易定位，日志不统一，升级回滚风险大 |
| **需要接企业数据** | 企业资料不在模型里，需要 RAG、权限控制、知识库更新 |
| **需要接工具** | 不只是回答，还要读文件、查数据库、调 API、执行工作流 |

这时候单纯"模型本体"已经不够，还需要推理引擎、API 网关、缓存系统、调度系统、监控系统、发布系统。这整套东西，就叫 **AI 基础设施（AI Infrastructure）**。

### 3.2 一个现代大模型产品的分层架构

```
1. 模型层         GPT / Claude / Llama / DeepSeek / Qwen
2. 推理层         vLLM / SGLang / TensorRT-LLM / ONNX Runtime
3. 服务层         API 网关、限流、认证、路由、重试
4. 数据增强层     RAG、向量检索、知识库、缓存
5. 工具执行层     调数据库、调搜索、调浏览器、调代码工具
6. 观测与运维层   日志、指标、追踪、告警、版本管理、灰度发布、回滚
```

从软件工程的角度看，今天的 AI 产品越来越像一个复杂分布式系统，而不只是一个模型文件。这正是工程型开发者的核心价值所在。

### 3.3 核心瓶颈：KV Cache 为什么是显存杀手

LLM 推理的一个关键矛盾（详见 §2.4）：

- 模型权重已经很大（7B 参数 ≈ 14GB FP16）
- 推理时还需要 KV Cache，缓存中间状态避免重复计算
- KV Cache 随对话长度**线性增长**：多 1 个 token，各层多存一行 K/V
- 单请求长对话 → 常见几十到几百 MB 缓存
- 100 个并发合计 → 才容易到十几、几十 GB

这就解释了为什么 vLLM 的 PagedAttention 重要：按块管理 KV，缓解预分配碎片。论文里「利用率接近 100%」主要指 **KV 这块分配更紧**，不是整卡算力拉满。Continuous Batching 则解决「短请求被长请求拖死」的调度问题，见 §2.4 末。

### 3.4 推理服务怎么观测（给运维 / 平台后端）

> 传统 APM 看 QPS、P99、错误率；LLM 服务还要拆 **首字慢还是后续慢、显存/KV 是否顶满、队列是否堆住**。动手对齐 [plan Step 2B](ai-infra-project-plan.md)。

#### 三类指标，别混用

| 类别 | 看什么 | 典型用途 |
|------|--------|----------|
| **延迟（LLM 专用）** | TTFT、TPOT、端到端耗时 | 「接口慢」拆因 |
| **吞吐 / 负载** | tokens/s、并发数、队列深度 waiting/running | 容量与扩缩容 |
| **资源** | GPU util、显存、KV 占用、OOM/失败原因 | 会不会爆、要不要降并发 |

传统 **QPS / HTTP P99** 仍然要：用来看网关与非推理段。但「用户感觉字蹦得慢」优先看 TTFT/TPOT。

#### 慢请求怎么拆

```
用户说慢
  → 先拆端到端时间：
      网关/鉴权 → Prompt/RAG → 排队 → tokenize → Prefill → Decode → SSE/网络 → 客户端
  → TTFT 高：
      逐段看网关、Prompt 构建、queue wait、tokenize、Prefill、首个有效内容事件
  → TTFT 正常、TPOT 高：
      看 Decode、批调度、模型权重/KV 带宽、抢占、多卡通信、客户端背压
  → 两者都高：
      结合队列、scheduled tokens、batch size、preemption 与 GPU/显存指标
  → 伴随 OOM / 429 → 并发槽、KV 上限、限流策略
```

仅看 TTFT/TPOT 只能发现症状，无法确认根因。学习路径（HF + FastAPI）至少统一 `request_id`，记录各阶段时间、prompt/output tokens、TTFT、TPOT 与模型版本。生产路径再对齐 Serving runtime 的原生指标，例如 queue wait、running/waiting 请求数、scheduled tokens、batch size、KV cache 使用率、preemption 和前缀缓存命中。FastAPI 多作网关时，网关延迟与引擎延迟要分开记录。

#### 最小看板（建议）

1. TTFT p50/p95、TPOT p50/p95  
2. queue wait、活跃请求数、等待队列、batch size  
3. GPU 显存、KV 占用、preemption（若 runtime 暴露）  
4. 错误分类：超时、OOM、4xx/5xx、工具异常、客户端取消  
5. 按模型版本分组；可选记录单次请求成本（tokens × 单价）

质量侧（幻觉、RAG 召回）见 [ai-intro.md §3.6](ai-intro.md)，不要和延迟指标混在一张「好不好」里。

---

## 4. AI Infra 各工种与各地区岗位市场

> **阅读级别：求职附录。** 做产品 / 只补机制可整章跳过。权威数字与选型以 §5 为准；带队分工见 [ai-intro.md §4.5](ai-intro.md)。

### 4.1 工种对照表

| 工种 | 工作内容 | 适合有工程背景的人？ |
|------|---------|:---:|
| **推理部署工程师** | 把模型装到服务器上跑起来，搭环境、配服务、盯 QPS/延迟 | ✅ |
| **ML Platform 工程师** | 做模型上线流水线平台，让算法工程师一键部署 | ✅ |
| **MLOps / AI DevOps** | 模型从开发到上线的全流程自动化（自动测试→推模型→切流量→回滚） | ✅ |
| **推理优化工程师** | 深入引擎内部提速：写 CUDA kernel、算子融合，需要 GPU 和 C++ 功底 | ⚠️ |
| **AI 算法工程师** | 训练模型、设计架构、调参，是造权重的人 | ❌ 不是工程方向 |

**注意：** 2025-2026 年业界趋势是"AI 工程师""LLM 工程师""提示词工程师"等 hype 期岗位名称正在被传统工程学科吸收，核心技能分别归入应用工程、基础设施工程、ML/评估工程和数据工程四大类。

### 4.2 中国大陆岗位市场（2025-2026）

**核心技能需求分布（基于 Boss 直聘、猎聘 JD 分析）：**

| 技能 | 需求程度 | 说明 |
|------|:---:|------|
| **Python** | 🟢 必备 | 几乎 100% 岗位标明，主力语言 |
| **Kubernetes** | 🟢 高频 | 平台型 AI Infra 岗位常见，部分岗位明确要求 K8s 算力调度 |
| **Docker** | 🟢 必备 | 容器化 AI 工作负载 |
| **vLLM / SGLang** | 🟢 必备 | 国内企业已从"二选一"转向**双框架协同部署**，vLLM 做生产基座，SGLang 做智能编排 |
| **FastAPI** | 🟢 高频 | AI 推理服务网关/API 层首选框架 |
| **RAG / Agent** | 🟢 高频 | LangChain、LlamaIndex、向量数据库(Milvus/FAISS)、Agent 编排 |
| **PyTorch / HF** | 🟢 高频 | 模型加载与实验 |
| **C++** | 🟡 加分 | GPU kernel、推理框架底层开发 |
| **国产芯片适配** | 🟡 加分 | 昆仑芯/昇腾/海光，vLLM-Kunlun Plugin 等 |

**薪资数据（深圳，猎聘 6,409 样本，数据采集 2025 Q3-2026 Q1）：**

| 经验级别 | 月薪范围 |
|---------|---------|
| 1 年以下 | ~¥15,800 |
| 1-3 年 | ~¥18,400 |
| **3-5 年** | **~¥24,200** |
| 5 年以上 | ~¥26,700 |
| 全行业均值 | **¥18,680** |
| 薪资分布 | **20K-30K 占比最高（25%）**，15K-30K 合计占 46% |
| AI Infra 工程师（细分岗位，含推理优化） | **月均 ¥75,800**（注：样本量小，主要为资深岗，不代表入门薪资） |

**岗位热度：**
- 2026 年 AI 岗位同比增长 **~12 倍**
- 大模型相关岗位占 AI 招聘 **45%**
- 企业更看重**实际项目经验**（52.5%）而非名校学历（仅 28.8%）
- 主流开源模型：DeepSeek、Qwen、GLM

**Boss 直聘/猎聘 搜索关键词（深圳）：**

可以搜的：
- 大模型推理部署 / AI Infra / 模型部署工程师
- AI 平台研发 / MLOps / LLM 服务化工程师
- Python 后端开发（AI 方向）/ RAG 开发工程师 / AI 应用开发工程师

若搜到以下岗位，注意它们不是 AI Infra 方向：
- **AI 算法工程师 / 深度学习工程师**，偏模型设计和训练，不是工程化方向
- **大模型训练工程师**，需要 GPU 集群调度经验，不适合入门
- **数据标注工程师**，不同的工种，不涉及模型部署和推理

> **数据来源：** 猎聘薪资统计、脉脉《2025-2026 人才市场洞察报告》、Boss 直聘 2025 薪资报告、CSDN 行业分析、掘金社区 JD 调研

### 4.3 香港岗位市场

| 维度 | 数据 |
|------|------|
| 月薪范围 | **HK$60,000-88,000**（AI Engineering Lead） |
| 核心技能 | Python, LangChain, RAG, Docker, K8s, vLLM, Terraform, AWS/Azure/GCP |
| 语言要求 | **English** 必备，Cantonese/Mandarin 加分 |
| 行业侧重 | **金融/银行/保险** 占主导，重视 security、governance |
| 本地化部署 | 企业明确要求部署开源 LLM（Qwen, DeepSeek, Gemma）而非仅用公共 API |

> **数据来源：** Michael Page Hong Kong, CTgoodjobs, Apple HK, Randstad

### 4.4 新加坡及东南亚市场

| 地区 | 月薪范围（USD 等值） | 特点 |
|------|---------|------|
| **新加坡** | SGD $6K-14K（~$72K-168K USD/年）；顶级科技公司 $14K-24K | 东南亚枢纽，**92% AI 融资集中于此**，~11,000 AI 工程师 |
| **马来西亚** | $35K-70K USD/年 | ~10,600 AI 工程师，半导体/云基础设施驱动 |
| **印度尼西亚** | $12.5K-60K USD/年 | ~12,800 AI 工程师，最大人才池 |

**东南亚整体趋势：**
- **~67,000** AI 工程师有生产经验（2023 年翻倍）
- LLM 工程师需求 2025 年涨 **340%**
- AI/ML 薪资 2025 年涨 **18%**，预计 2027 年前年涨 12-15%
- 仅 **43%** AI 工程师有实际 LLM 经验，LLM 专长者有 +28% 薪资溢价
- 技能优先招聘成为趋势，更看重实践经验而非学历

> **数据来源：** Second Talent《AI Engineering Talent in Southeast Asia 2026》、MyCareersFuture.sg、Glassdoor Singapore、Aon 2025 SEA Study

### 4.5 日本与韩国市场

| 地区 | 月薪范围 | 核心技术栈 |
|------|---------|-----------|
| **日本（东京）** | ¥10M-18M/年（~$67K-120K USD） | Python, K8s, vLLM, LangChain, RAG, BigQuery/Databricks |
| **韩国（首尔）** | 未完全公开 | Python, K8s, vLLM, GPU 集群(NVIDIA A100/H100), Triton, TensorRT-LLM |

**日韩共同特点：**
- vLLM 在 LLM serving 岗位中出现频率高
- Kubernetes 在平台与集群型岗位中常见，具体权重取决于团队部署方式
- 多家公司提供 remote + relocation 路径
- 日语/韩语不总是硬性要求，但提供学习支持

> **数据来源：** 禾蛙 (Hewa), MangoBoost, CAST AI, Shine.com, jaabz.com

### 4.6 各地区对比总览

| 地区 | 中级月薪 | 最大需求方向 | 核心差异化技能 |
|------|---------|-------------|--------------|
| **深圳/大陆一线** | ¥18K-30K | 大模型推理部署 + RAG + Agent | 国产芯片适配、DeepSeek/Qwen 经验 |
| **香港** | HK$60K-88K | 金融 AI + 本地化 LLM 部署 | English + Cantonese、合规经验 |
| **新加坡** | SGD $6K-14K | MLOps + LLM 部署 + GPU 优化 | 多语言、国际团队协作 |
| **东京** | ¥10M-18M/年 | AI 数据平台 + LLM 编排 | 日企文化适应、数据工程 |
| **首尔** | 未公开 | GPU 推理优化 + AI 基础设施 | NVIDIA 硬件深度经验 |

### 4.7 Python vs C++ 的角色分工

| 维度 | Python | C++ |
|------|--------|-----|
| **在 AI Infra 中的主要用途** | API 服务、模型加载、推理编排、数据处理、自动化 | GPU kernel、推理引擎底层、高性能网络通信 |
| **对应岗位** | AI 平台后端、ML Platform、LLM 服务化 | 推理优化工程师、AI 框架开发 |
| **岗位数量** | 更多 | 少但薪资高 |
| **是否需要 GPU 经验** | 不强制 | 通常需要 CUDA |
| **学习曲线** | 低 | 高 |

**关键结论：** 对于有 C/C++ 后端经验转 AI 工程的开发者，Python 路径岗位更多、更友好；C++ 背景在推理优化方向是隐藏优势（PagedAttention ≈ 虚拟内存页表、Continuous Batching Scheduler ≈ 连接池调度）。

---

## 5. 2026 年 AI 工程技术栈速览

以下数据均来自 2026 年中的公开可查来源，每个数字都有出处。

### 5.1 模型加载：HuggingFace Transformers

| 指标 | 数据 |
|------|------|
| GitHub Stars | **~170,000+**（截至 2026.7 实际已远超 v5 发布时的 162k；全局排名 #48） |
| PyPI 月下载量 | **~1.67 亿次** |
| 累计安装 | **超 12 亿次** |
| 日 pip 安装量 | **>300 万/天**（v5 发布后增长 150×） |
| 支持模型架构 | **400+** |
| 社区模型数 | **75 万+** |
| 最新版本 | v5.12.1（2026.6） |
| 协议 | Apache 2.0 |

**重要变化（v5，2025.12）：** Transformers v5 明确了自身定位，作为模型定义和加载的标准接口，不替代专用推理引擎。vLLM、SGLang、ONNX Runtime 团队均与 HF 团队有合作和互操作引用。

**证据来源：** [HF 官方博客 (2025.12)](https://huggingface.co/blog/transformers-v5)、[Star History](https://www.star-history.com/huggingface/transformers/)、pypistats.org

### 5.2 推理框架：vLLM vs SGLang

> 注：vLLM/SGLang 按 §1.4 五零件分类属于"推理框架"（引擎 + KV Cache 管理 + 调度 + 多卡并行）。但行业日常交流中常直接叫"推理引擎"，两者混用是普遍现象，本文 §5 沿用行业习惯但以"框架"为准。

| | vLLM | SGLang |
|------|------|------|
| GitHub Stars | **~86,200**（截至 2026.7） | **~30,200** |
| Forks | 18,700+ | 7,100+ |
| 贡献者 | 2,000+ | — |
| 部署 GPU 数 | **400,000+** | — |
| PyPI 周下载 | — | **~1.02 亿次** |
| 最新版本 | 持续活跃更新 | v0.5.13.post1 |
| 组织归属 | PyTorch Foundation（2025.5 起） | PyTorch Ecosystem（2025.3 起） |
| 生产用户 | Meta, Mistral, Cohere, Google, IBM, Databricks, Cloudflare | xAI（Grok）, NVIDIA, LinkedIn, Cursor |
| 核心优势 | 最稳定、社区最大、KV Cache 分页管理 | RadixAttention（多轮对话快 10-20%）、前缀缓存 |
| 性能参考 | Llama-70B 1,000-2,000 tok/s（A100） | DeepSeek V3 上比 vLLM 快 3.1× |

> **vLLM PagedAttention 演进说明：** KV Cache 分页管理的**设计理念保留**在新架构中，换的是实现引擎（FlashAttention + Triton 替代旧版 CUDA kernel），不是设计思想。文档中"PagedAttention"指代这一设计理念。

**HuggingFace TGI（Text Generation Inference）已于 2025.12.11 进入维护模式、2026.3.21 归档只读**，自托管 LLM 服务市场正式进入 vLLM vs SGLang 双雄时代。

**2025-2026 一种常见实践：vLLM + SGLang 分工。** 不一定「二选一」：vLLM 做生产基座（并发、KV、PagedAttention），SGLang 做部分结构化输出与加速场景。有实战文给出特定配置下的格式错误率对比；把它当架构选项，不要背成「业界唯一主流」。选型总表仍以本节数据为准。

> **一个影响行业的事件：DeepSeek。** 2025 年初 DeepSeek-R1 的发布改变了行业认知，证明了架构创新（MLA、MoE）和训练方法创新（GRPO）可以显著缩小与前沿模型的资源差距。今天 AI 竞争从"单纯拼 GPU 数量"转向"资源 + 算法 + 工程 + 生态"的综合竞争。详细分析见 [ai-intro.md §4.2](ai-intro.md)。

**选型参考：** 追求稳定 → vLLM；追求极致性能/结构化输出 → SGLang；国内企业推荐 vLLM + SGLang 协同；本地开发/个人体验 → Ollama（~174,000+ stars，但不支持 Continuous Batching 和多 GPU 张量并行，不适合生产）。

**证据来源：** [AI Wiki - vLLM](https://aiwiki.ai/wiki/vllm)、[Socket - sglang](https://socket.dev/pypi/package/sglang)、[EVAL #001: 推理引擎对比](https://buttondown.com/ultradune/archive/eval-001-the-great-llm-inference-engine-showdown)、[开源 LLM 推理引擎对比 2026](https://fish.audio/zh-CN/blog/open-source-llm-inference-engines-2026/)、[vLLM+SGLang 协同部署实战](https://devpress.csdn.net/amd/6a38e5c310ee7a33f280bbe9.html)

### 5.3 API 框架：FastAPI

| 指标 | 数据 |
|------|------|
| GitHub Stars | **~99,500**（接近 100k） |
| PyPI 月下载 | **~900 万** |
| Python 开发者采用率 | **38%**（JetBrains 2025 调查，2023 年仅 29%） |
| ML 工程师用于模型服务 | **42%**（vs Django 22%, Flask 28%） |
| 年增长 | **40%** |
| Fortune 500 采用 | **50%+** |
| 岗位年增长 | **150%**（2024-2025） |
| 知名用户 | Uber, Netflix, Microsoft, vLLM, Dify (132k stars), Open WebUI (130k stars) |

**FastAPI 为什么常用于学习与网关层：** 支持 async/ASGI、Pydantic 数据校验与自动 OpenAPI 文档；配合 `StreamingResponse` 或 SSE 库可以返回增量文本。生产 GPU 推理进程通常仍由专用 Serving runtime 承担。

**实际 benchmark：** FastAPI + Uvicorn 在 Llama 3-8B 服务化场景中，比 Flask + Gunicorn 高 3.2× 吞吐量、P95 延迟低 68%。

**证据来源：** [FastAPI 2026](https://www.programming-helper.com/tech/fastapi-2026-python-api-framework-ai-ml-adoption-enterprise)、Byteiota、JetBrains Survey

### 5.4 部署：Docker + Kubernetes

| 指标 | 数据 |
|------|------|
| K8s 生产使用率 | **82%** 的容器用户 |
| GenAI 推理在 K8s 上 | **66%** 的企业（CNCF 2025 调查，样本量有限；不同调查数据有差异） |
| 数据工作负载在 K8s 上 | **~50%** 的企业 |

**重要趋势：** CNCF 社区已启动 K8s「AI 一致性」计划，尝试定义运行 AI 工作负载的基线能力。vLLM、SGLang 常被用于 K8s 上的高吞吐 LLM 服务；KEDA 等基于队列的自动缩放方案也在相关架构中出现。

**证据来源：** [CNCF 博客 (2026.3)](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes/)

### 5.5 OpenAI 风格 API：常见兼容接口

许多推理服务实现了 OpenAI 风格的 `/v1/chat/completions` 端点，因此应用层可以复用一部分客户端代码。但更换引擎通常不只改 `base_url`：模型名、支持参数、tool schema、错误结构、usage、流式 delta 和取消语义都可能不同。

因此项目需要写明自己实现的是完整协议还是**兼容子集**。2026 年 1 月提出的 Open Responses 规范（OpenRouter、HuggingFace、LM Studio、vLLM、Ollama、Vercel 共同发起）尝试减少这些差异；具体支持情况仍以各服务的官方文档为准。

**证据来源：** [vLLM 官方文档](https://docs.vllm.ai/en/stable/serving/openai_compatible_server/)、[Open Responses (Simon Willison)](https://simonwillison.net/2026/jan/15/open-responses/)

### 5.6 ONNX Runtime 的角色：不是过时，是不同赛道

| 指标 | 数据 |
|------|------|
| GitHub Stars | **~21,100** |
| 最新版本 | v1.27.0（2026.6） |
| 年增长 | ~16%（慢于 vLLM/SGLang，但稳步增长） |
| 贡献者 | 727 |

**ONNX Runtime 在 LLM 推理生态中的实际位置：**

- **没有过时。** 在边端部署、CPU 推理、ARM 服务器、跨硬件平台、嵌入式设备场景仍然不可替代。
- **ONNX Runtime GenAI** 扩展提供了专用 LLM API（tokenization + generation loop + KV Cache 管理），CPU 上部署代码量减少 ~90%。
- **Arm + Microsoft KleidiAI** 集成后，ARM CPU 上 Phi-3 推理 2.4× 加速。Azure Cobalt 100 (Arm) + ORT 性价比是 AMD Genoa (x86) 的 2.8×。
- **学术研究（2026）：** ONNX 转换在 NVIDIA GPU 上相比原生 PyTorch 延迟降 27.6%、GFLOP/s 吞吐提升 48.3%。
- **HF 生态：** Transformers v5 明确与 ONNX Runtime 深度互操作。HF 官方 `optimum-benchmark` 将 ORT 列为与 PyTorch、vLLM、TensorRT-LLM 同等的一等后端。

**HF Transformers vs ONNX Runtime 的使用场景：**

| | HF Transformers | ONNX Runtime |
|------|------|------|
| **适合** | 模型实验、加载、微调、快速原型 | 生产部署优化、跨平台、边端、CPU 推理 |
| **不适合** | 高并发生产服务（应配合 vLLM） | 模型实验和快速迭代 |
| **关系** | 模型的"源格式" | 模型的"部署格式" |

**一句话：HF Transformers 做实验和加载，vLLM/SGLang 做 GPU 生产服务，ONNX Runtime 做跨平台/CPU/边缘设备部署。三者是同一模型在不同场景下的不同用法。**

**证据来源：** [Snyk - onnxruntime](https://security.snyk.io/package/pip/onnxruntime)、[Arm + KleidiAI (2025.5)](https://onnxruntime.ai/blogs/arm-microsoft-kleidiai)、[ONNX Runtime GenAI](https://developer.baidu.com/article/detail.html?id=7322579)、IEEE 基准研究 (2026)、[optimum-benchmark](https://github.com/huggingface/optimum-benchmark)

### 5.7 小模型（SLM）的生产化趋势

2025-2026 年最重要的产业趋势之一：**不是所有场景都需要大模型。**

| 数据点 | 数值 |
|------|------|
| HF 上 1B 以下模型下载占比 | **92.48%** |
| 企业 AI 任务可在 10B 以下完成 | **40-70%** 的 agentic AI 调用可被 SLM 替代（NVIDIA Research, 2025.6） |
| 微调 SLM vs GPT-4o 分类准确率 | **94% vs 91%**（SLM 胜） |
| 成本差距（1000 万次分类） | SLM $600/月 vs GPT-4o $300,000/月 |
| 主流 SLM 模型 | Qwen3-0.6B, Phi-4-mini (3.8B), Gemma 3 (4B), Llama 3.2 (3B) |

**企业主流模式：混合路由（Router）**
- **~70%** 请求走本地 SLM 快通道（分类、提取、简单 Q&A）
- **~20%** 走中等 API
- **~10%** 走前沿 API（复杂推理、长上下文）

**意义：** 用 0.6B 小模型做学习项目是完全合理的，它不仅是实验配置，还是真实生产趋势的一部分。面试时说"用了 0.6B 模型跑 benchmark"也不用心虚，加一句"2026 年 92.48% 的 HF 模型下载是 1B 以下模型，SLM 路由是生产主流模式之一"就行。

**证据来源：** [SLMs in Production 2026](https://dev.to/alexcloudstar/small-language-models-in-production-2026-where-slms-beat-frontier-models-and-where-they-quietly-3kn5)、[SLM Enterprise Guide 2026](https://ortemtech.com/blog/small-language-models-enterprise-guide-2026)、NVIDIA Research (2025.6)、PER 学术研究 (2026)

---

## 6. 学习路径建议

**唯一顺序：** [ai-intro.md](ai-intro.md)（地图）→ **本文**（机制与边界）→ [ai-infra-project-plan.md](ai-infra-project-plan.md)（动手）。不要把本文当成第一站再回头补 intro。

**怎么用本文：**
- 先搞清「应用工程 vs 推理 Infra」边界（§1），再决定主攻哪条
- 机制至少读懂 Prefill / Decode / KV（§2.4）；矩阵六步按需精读
- 不要同时学 vLLM + SGLang + TensorRT；先选一个（生产常见 vLLM）
- 代码可以 AI 辅助，但核心路径必须自己能讲清

**项目路径选择：**

**路径 A：Python + HF Transformers（推荐，岗位更多）**
- 第一天：Colab 上加载小模型跑通生成
- 再用 FastAPI 做成 OpenAI 兼容流式服务（**学习**用 HF+FastAPI；**生产** GPU 推理主进程更常见是 vLLM/SGLang，FastAPI 多作网关/编排）
- 对应 plan 的 Step 1→2

**路径 B：C + ONNX Runtime（边端/嵌入式）**
- 适合边端、CPU、跨平台；与当前 plan 主线不同，本文只作选项说明

当前国内岗位数量上，路径 A 更常见。

---

## 附录 A：自检清单

读完本文后应能回答：

1. 从发消息到看到回复，中间经过了哪些环节？哪个环节算 AI Infra、哪个不算？
2. ChatGPT、OpenCode、Cursor 在 AI Infra 链路上有什么本质区别？
3. "推理引擎"和"推理服务器"的区别是什么？五个核心零件分别是什么？
4. AI ≠ if-else + 数据库，那本质区别是什么？RAG 在其中的位置？
5. Attention 的六步计算分别做什么？Q、K、V、WO 各是什么角色？
6. FFN 的"膨胀→门控→压缩"三步分别做了什么？非线性在哪里？
7. Prefill 和 Decode 的核心区别是什么？为什么 Decode 叫 memory-bound？
8. KV Cache 存什么、为什么不存 Q？100 个用户同时来的 KV Cache 大约多大？
9. AI Infra 有哪些工种？中国大陆哪些岗位需求最大？
10. 深圳 AI Infra 岗位月薪大致范围？和香港/新加坡/东京的差异？
11. 2026 年 LLM 服务化的主流技术栈是什么？vLLM 和 SGLang 的关系？
12. ONNX Runtime 过时了吗？它和 HF Transformers、vLLM 是什么关系？
13. 为什么小模型（0.5B-7B）是 2026 年的重要趋势？
14. 采样策略中 Temperature 和 Top-p 分别控制什么？
15. messages / SSE / tool_call 最小闭环各自解决什么？和 Prefill/Decode 怎么衔接？
16. TTFT 高和 TPOT 高分别优先怀疑什么？和 HTTP P99 有何不同？
17. 延迟指标与 RAG 质量指标为什么不能混成一张「好不好」？

---

## 附录 B：数据来源索引

| 数据 | 来源 | 时间 |
|------|------|------|
| HF Transformers 170k+ stars, v5 日 300 万下载 | [HF 官方博客](https://huggingface.co/blog/transformers-v5), Star History | 2025.12 / 2026.7 |
| vLLM 86.2k stars, 40 万 GPU | [AI Wiki](https://aiwiki.ai/wiki/vllm), GitHub | 2026 中 |
| SGLang 30.2k stars, 1.02 亿周下载 | [Socket](https://socket.dev/pypi/package/sglang) | 2026 |
| FastAPI 99.5k stars, 38% 采用率 | [Programming Helper](https://www.programming-helper.com/tech/fastapi-2026-python-api-framework-ai-ml-adoption-enterprise) | 2026 |
| K8s 82% 生产使用, 66% GenAI（注：样本量有限，不同调查有差异） | [CNCF 博客](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes/) | 2026.3 |
| ONNX Runtime 21.1k stars, v1.27.0 | [Snyk](https://security.snyk.io/package/pip/onnxruntime) | 2026 |
| Ollama 174k+ stars | GitHub, 多方交叉验证 | 2026.6 |
| ONNX + KleidiAI 2.4× 提速 | [ONNX Runtime 博客](https://onnxruntime.ai/blogs/arm-microsoft-kleidiai) | 2025.5 |
| ONNX vs PyTorch 延迟降 27.6% | IEEE 基准研究 | 2026 |
| TGI 归档 | 社区报告 | 2026 |
| SLM 92.48% 下载量, Router 模式 | [SLM in Production 2026](https://dev.to/alexcloudstar/small-language-models-in-production-2026-where-slms-beat-frontier-models-and-where-they-quietly-3kn5) | 2026 |
| RAG 市场规模 | MarketResearch, MarketsandMarkets（注：不同机构估算差异极大，从几亿到几十亿美元不等，任一单一数据不可采信） | 2025 |
| 深圳 AI Infra 薪资 ¥18,680/月 | 猎聘 (6,409 样本) | 2025-2026 |
| 大模型岗位占 AI 招聘 45%, 增长 12× | Boss 直聘, 脉脉, CSDN | 2026 |
| vLLM+SGLang 协同部署成国内主流 | [CSDN](https://devpress.csdn.net/amd/6a38e5c310ee7a33f280bbe9.html), 阿里云, 百度百舸 | 2026 |
| 香港 AI 工程 HK$60K-88K/月 | Michael Page HK, CTgoodjobs | 2026 |
| 新加坡 11,000 AI 工程师, 92% 融资集中 | [Second Talent](https://www.secondtalent.com/resources/the-state-of-ai-engineering-talent-in-southeast-asia-data-report/) | 2026 |
| 东南亚 LLM 需求涨 340%, ~67,000 AI 工程师 | Second Talent, Aon SEA Study | 2025-2026 |
| 东京 AI 平台 ¥10M-18M/年 | 禾蛙 (Hewa) | 2026 |
| 首尔 vLLM/K8s/GPU | MangoBoost, CAST AI | 2026 |
| OpenAI 兼容 API 协议 | [vLLM 官方文档](https://docs.vllm.ai/en/stable/serving/openai_compatible_server/) | 2026.6 |
| Open Responses 规范 | [Simon Willison](https://simonwillison.net/2026/jan/15/open-responses/) | 2026.1 |
| EVAL #001 推理引擎对比 | [Buttondown](https://buttondown.com/ultradune/archive/eval-001-the-great-llm-inference-engine-showdown) | 2026.3 |
