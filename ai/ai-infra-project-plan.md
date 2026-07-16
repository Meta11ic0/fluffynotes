# AI 工程落地项目计划：LLM 服务化 + RAG + 可观测性

> 生成日期：2026-06-17 | 状态：执行中 | 最后更新：2026-07-16（v9.3：协议、技术准确性与文风复审）

---

## 一、总览

> **读这份计划前：** 先读 [ai-intro.md](ai-intro.md) 与 [ai-engineering.md](ai-engineering.md)（至少 engineering §1 + §2.4；做流式再读 §1.6；做观测再读 §3.4）。  
> **本文是什么：** 个人作品集 / 转岗执行手册，主线是 **LLM 服务化 + 可观测性**（平台后端 / 应用工程），不是纯推理 Infra 深水区路线图。  
> **读者分叉：**  
> - **求职** → 按 Step 0→2C  
> - **做产品验证** → 可先商业 API 出 demo（见下方「1 周可演示」），再把 Step 2 的服务化与观测迁到自托管  
> - **已会 RAG** → 可跳过 Step 3，Step 4 紧挨 Step 2

### 0.1 两条最短路径（先选一条）

| | 求职作品集 | 产品验证（非求职） |
|--|-----------|-------------------|
| 目标 | 可讲、可观测、可投递 | 用户能完成一次任务 |
| 第一周 | Step 0  Ownership + Step 1 最小推理 | API + 简单 UI/脚本 + 用量日志 |
| 随后 | Step 2A→2B→2C | 热点换 SLM；有私有知识再 RAG |
| 不要一上来 | 纯 Agent / 多卡 vLLM | 先自建 Serving 再验证需求 |

**1 周可演示（产品向）最小集：**

1. 一个垂直场景（如「基于 3 篇自己的文档问答」或「代码解释接口」）  
2. 先调商业 API（或本地 Ollama）跑通  
3. 记录：每次请求 tokens、大概成本、是否答对（人工 10 条）  
4. 可选：再迁到 plan Step 2 的 FastAPI + TTFT/TPOT

---

### 1.0 按问题驱动的小闭环交付

本计划从泛化的「AI Infra 学习路线」收敛为「AI 工程落地交付路线」。

- 如果只让 AI 生成脚本，再照着运行，项目即使完成也很难形成可复述、可排障的能力。
- 已有的系统工程经验适合用来收敛问题、验证链路和推动交付。
- 每个阶段都按「问题单 → 小闭环 → 可讲产物」推进，再继续扩大功能。

**新的执行原则：**

1. 代码可以让 AI 辅助生成，但核心路径必须能自己讲清。
2. 每个脚本都必须配一段 `notes/ownership.md`，说明它解决什么问题、核心 API 是什么、如何验证。
3. 每次只做一个小闭环，不以"完成整个项目"为日目标。
4. 正反馈来自"搞懂了一个机制 / 修掉一个问题 / 得到一个数字"，不是来自文件数量。
5. 每个 Step 都以一个真实问题开头，例如"接口为什么慢""RAG 为什么答不准""流式输出为什么不能直接 async generate"。

### 1.1 到底在做什么

目的是补齐 AI 产品从模型到业务落地的最小工程链路：

```
用户请求
  → API 服务
  → Prompt / 消息协议
  → 模型推理
  → 流式输出
  → 指标监控
  → 检索增强
  → 部署交付
```

本计划用 Step 0-1-2 先形成最小可行入场券，Step 3 作为应用落地加分，Step 4 仅作为推理部署概念加分：

| Step | 项目 | 对应 AI 产品链路模块 | 训练目的 |
|------|------|------|------|
| Step 0 | Ownership Recovery | AI 辅助边界、最小重写、手动验证 | 先确认项目是"可讲清的项目"，不是 AI 生成脚本集合 |
| Step 1 | HF Transformers LLM 推理基础 | 模型加载、tokenizer、生成参数、量化、benchmark | 先理解 LLM 本身如何被调用 |
| Step 2 | LLM 流式服务化（2A-2D） | Chat Completions 兼容子集、SSE、TTFT/TPOT、日志、Docker、可观测性 | 把模型包装成可交付服务，并能定位问题 |
| Step 3 | RAG 问答系统 | 文档切片、Embedding、向量库、检索、引用生成、质量评估 | 加上企业最常见的知识库问答场景 |
| Step 4 | vLLM 概念学习 | KV Cache、PagedAttention、Continuous Batching | 只做概念加分，不作为第一阶段主线 |

### 1.2 AI 产品全链路

一个真实 AI 产品通常是分层的：

| 层级 | 解决的问题 | 常见技术/产品 | 本计划覆盖 |
|------|------|------|------|
| 模型层 | 选模型、加载模型、控制生成质量和成本 | HuggingFace Transformers、PyTorch、Qwen、Llama | Step 1 |
| 推理服务层 | 把模型变成稳定 API，支持并发和流式输出 | FastAPI、OpenAI API、SSE、vLLM、SGLang | Step 2，Step 4 概念补充 |
| 应用编排层 | 组织 Prompt、上下文、工具调用、工作流 | LangChain、LlamaIndex、LangGraph、OpenAI/Claude SDK | Step 3 只覆盖 RAG，不做 Agent |
| 数据/知识层 | 把企业文档变成模型可用上下文 | Embedding、ChromaDB、pgvector、Milvus、Reranker | Step 3 |
| 部署与平台层 | 让服务能上线、扩缩容、回滚和持续交付 | Docker、K8s、CI/CD、Prometheus | Step 2 |
| 评估与观测层 | 证明效果、发现延迟/幻觉/成本问题 | TTFT/TPOT、tokens/s、Langfuse、LangSmith、RAGAS | Step 2、Step 3 |

这条链路的重点是：先做"能落地、能排障、能观测"的 AI 应用工程，再补少量高性能推理架构理解。

### 1.3 同类型方向有哪些，市场最热门是什么

| 方向 | 典型产品/框架 | 热度证据 | 是否作为主项目 |
|------|------|------|------|
| 模型加载与基础推理 | HuggingFace Transformers、PyTorch | Transformers GitHub 约 17 万 stars（2026.7），PyPI 月下载量过亿，是开源模型接入事实标准 | 是，Step 1 |
| LLM 应用编排 | LangChain、LlamaIndex、LangGraph | LangChain 约 10 万 stars，生态大；LlamaIndex 在 RAG 场景增长快 | 部分覆盖，Step 3 不重依赖框架 |
| 高性能推理服务 | vLLM、SGLang、llama.cpp、Ollama | vLLM ~86k stars（2026.7），主打 PagedAttention、Continuous Batching、OpenAI 兼容服务；Ollama ~174k stars 为本地开发首选 | 作为 P2 概念加分，Step 4 |
| 企业 RAG | LlamaIndex、ChromaDB、Milvus、pgvector、RAGAS | 企业落地最常见形态之一，招聘中“知识库问答 / RAG / Embedding / 向量检索”高频出现 | 是，Step 3 |
| Agent / 工具调用 | LangGraph、AutoGen、CrewAI、MCP | 2026 热度高，但链路复杂、评估困难，对初始作品集不够短路径 | 暂不作为第一阶段主线 |
| 训练 / 微调 | LoRA、QLoRA、TRL、Axolotl | 有需求，但依赖 GPU 和数据闭环；对当前预算与岗位目标不是最短路径 | 暂不做 |

结论：市场最热门是三条主线并行：

1. **HuggingFace Transformers**：模型接入基础设施，几乎所有开源 LLM 工程都绕不开。
2. **RAG / 知识库问答**：企业最容易落地、最容易出业务价值的 AI 应用形态。
3. **vLLM / SGLang**：自托管开源模型进入生产时的高性能推理核心。

本计划选择"HF Transformers → LLM 流式服务化 → RAG → vLLM 概念"，因为这套组合覆盖最常见 AI 产品链路，同时绕开 CUDA/C++/大规模训练这些高成本板块。

**证据来源：**
- HuggingFace Transformers：GitHub 约 17 万 stars（2026.7），PyPI 月下载量过亿，它是开源模型加载和推理的主流基础库。
- SLM 趋势：HF 上 92.48% 模型下载为 1B 以下，SLM 路由模式已成生产主流。本计划选用 0.6B-1B 小模型，既符合学习成本约束，也踩中产业趋势。
- LangChain / LlamaIndex：LangChain 约 10 万 stars，LlamaIndex 在 RAG 场景持续增长。
- vLLM/SGLang：vLLM ~86k stars，SGLang ~30k stars，国内已从二选一转向协同部署：vLLM 做生产基座，SGLang 做智能编排。
- 生产级 AI Stack 文章普遍把模型服务、RAG/Memory、Observability/Eval、部署层拆成独立模块。

### 1.4 为什么先做 Step 1

Step 1 选择 HuggingFace Transformers，因为它是后面所有项目的共同地基。

如果不先理解 tokenizer、chat_template、generate、temperature/top_p、max_new_tokens、量化和 benchmark，后面做 FastAPI 服务时只是在“包一层接口”，很难解释延迟、token 数、输出质量和内存占用从哪里来。

同类型替代项包括：

| 可选方向 | 为什么不作为第一步 |
|------|------|
| 直接用 OpenAI / Claude API | **求职作品集**不够：练不到本地加载、tokenizer、量化与自托管指标。**做产品验证**时，API 反而是合法第一步：先证明用户愿意用，再把热点路径换成 SLM/自托管压成本 |
| 直接上 LangChain / RAG | 会先学框架抽象，反而看不清模型调用底层 |
| 直接上 vLLM | vLLM 是服务引擎，不适合在没理解 HF generate 流程前学习 |
| 直接做微调 | 成本更高，且当前目标岗位更看重部署和应用链路 |

所以 Step 1 的定位是：用最小成本获得 LLM 工程的底层手感。

### 1.5 最终目标

数月内完成一个可讲的 LLM 服务化项目，简历上形成第一条可投递叙事线：**"C/C++ 系统工程背景 → HF Transformers 推理机制理解 → OpenAI 兼容流式服务 → TTFT/TPOT 可观测性 → Docker 交付"**。

v9 先保证一个项目真正可讲、可运行、可排障：

1. **Step 0：Ownership Recovery** — 把已有 `hf-llm-benchmark` 中 AI 生成的代码重新读懂、拆小、验证。
2. **Step 1：HF LLM Benchmark** — 形成 tokenizer、generate、参数实验、benchmark 指标的真实理解。
3. **Step 2：LLM 流式服务化** — 拆成 2A/2B/2C/2D，形成 AI 平台后端岗位最有价值的工程证据。
4. **Step 3：RAG 问答系统** — 第二阶段优先加分项，更贴近 AI 应用工程。
5. **Step 4：vLLM 概念学习** — 只做面试概念加分，不急着源码精读。

**不做单一岗位锁定**。当前阶段的策略是：用项目产出覆盖市场最热门的技能交集，然后根据实际投递反馈调整方向。以下是技能组合与岗位的对应关系（基于 2026-06 深圳市场数据）：

| 岗位方向 | 匹配度 | 核心匹配项 | 需补的差距 |
|----------|:---:|------|------|
| **AI 平台后端开发** | ⭐⭐⭐⭐ | FastAPI + Docker/K8s + Prometheus + HF 基础 | Step 0-1-2 先覆盖入场门槛，RAG/vLLM 作为后续加分 |
| **大模型应用开发** | ⭐⭐⭐⭐ | HF + RAG + FastAPI + 服务化 | LangChain 实操（Step 3 已安排 LangChain 对比任务）、Agent 概念（了解但不实操） |
| **推理部署工程师** | ⭐⭐ | vLLM + Docker/K8s + TTFT/TPOT | 缺少 GPU 多卡部署经验、CUDA 概念；暂不作为主线 |
| **ML Platform** | ⭐⭐⭐ | K8s + CI/CD + Prometheus + 模型服务化 | 缺少大规模集群经验、GPU 调度 |

> **定位说明（v9）**：本计划产出的技能组合与"AI 平台后端 / LLM 服务化 / 大模型应用工程"重合度最高；不再把"纯推理部署 / AI Infra 深水区"作为第一主线。**做完 Step 0-1-2C 后即可投递第一批岗位，用面试反馈决定 Step 3 先做 RAG 还是补 vLLM。**

### 1.6 约束条件

| 约束 | 说明 |
|------|------|
| 无本地 GPU，可租用（预算约 30-40 元） | 核心路径 CPU 可完成；Step 4 可选租 GPU 2-4h 做实验验证（~10-20 元），让简历数据更有说服力 |
| 业余时间 | 工作日 1-2h，周末 3-4h |
| 目标工种 | 不做单一锁定，优先 AI 平台后端 / 大模型应用工程 / LLM 服务化，推理部署仅作为加分方向（不涉及训练、不写 CUDA kernel） |
| 技术栈偏好 | Python 为主，匹配岗位实际需求 |
| 网络环境 | 国内访问 HF Hub 可能受限，需配置镜像 |

### 1.7 推荐路径

```
P0 Step 0: Ownership Recovery             ← 现在开始，7 天，只读懂/重写/验证已有代码
P0 Step 1: HF Transformers LLM 基础       ← Step 0 后，1-2 周，形成可讲 benchmark
P0 Step 2A: LLM 服务 MVP                  ← /health + OpenAI 兼容端点 + SSE 跑通
P0 Step 2B: 可观测性闭环                  ← request_id + JSON log + TTFT/TPOT + /metrics
P0 Step 2C: 交付闭环                      ← Docker + README + curl 示例 + 踩坑记录
P1 Step 2D: 工程加分                      ← K8s / CI/CD / 并发压测，按精力选择
P1 Step 3: RAG 问答系统                   ← 应用开发方向优先加分
P2 Step 4: vLLM 概念学习                  ← 推理部署方向概念加分
```

> **P0/P1/P2 说明**：Step 0-1-2C 构成最小可行证据链。Step 2D、Step 3、Step 4 都是加分项，不允许在 Step 2C 前抢占注意力。

### 1.8 时间估算

| 步骤 | 优先级 | 预计耗时 | 日历时间 | GPU | 核心技能 |
|------|:---:|---------|---------|:---:|------|
| Step 0 | P0 | 数小时 | 约一周 | ❌ | 代码 ownership, 独立解释, 最小重写 |
| Step 1 | P0 | 约十余小时 | 数周 | ❌ | HF Transformers, tokenizer, 生成参数, benchmark |
| Step 2A | P0 | 数小时 | 约一周 | ❌ | FastAPI, SSE streaming, OpenAI compatible API |
| Step 2B | P0 | 数小时 | 约一周 | ❌ | request_id, JSON log, TTFT/TPOT, Prometheus |
| Step 2C | P0 | 数小时 | 约半周 | ❌ | Docker, README, curl examples, troubleshooting notes |
| Step 2D | P1 | 约十余小时 | 数周 | ❌ | K8s, CI/CD, concurrency benchmark |
| Step 3 | P1 | 十余小时 | 数周 | ❌ | Embedding, ChromaDB, RAG pipeline, LangChain 对比 |
| Step 4 | P2 | 约十余小时（GPU 可选加时） | 数周 | ⚠️ CPU 为主，GPU 可选 | vLLM 概念, PagedAttention, Scheduler, KV Cache |
| **合计（P0 到 Step 2C）** | | **数十小时** | **约两月** | | |
| **合计（含 Step 3）** | | **数十小时** | **约一季** | | |

### 1.9 执行者背景：从 C/C++ 到 Python AI 生态

本计划的执行者是 C/C++ 开发者，以下是最小化的过渡知识，不要求精通 Python，够做项目就行。

**Python 虚拟环境是什么。** Python 的包（`pip install`）默认装到全局，不同项目依赖同一个包的不同版本会冲突。`venv` 创建一个隔离的 Python 环境，类比 C++ 的 `LD_LIBRARY_PATH` 隔离，只不过它是按项目隔离的，不是按进程。

```bash
# 每个项目开始前（习惯性操作，类比 cmake -B build）
python -m venv venv
source venv/bin/activate    # Linux/Mac
# 此后 pip install 的所有包都装在这个 venv 里，不影响其他项目
```

**PyTorch CPU vs CUDA 是不同的 pip package。** 默认 `pip install torch` 装的是 CUDA 版本，在 CPU 机器上可能报错或行为异常。本计划全部 CPU 运行，注意装 CPU 版本：

```bash
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

**Python async/await vs C++ 协程。** FastAPI 基于 Python 的 `async/await`。Python 的事件循环是**单线程**的：所有 async 任务在一个线程里轮转，没有真正的并行。这和 C++20 协程的模型不同（C++ 协程可以分布在多个线程上）。模型推理是 CPU 密集型操作，在 async 函数里直接调 `model.generate()` 会阻塞整个事件循环。Step 2 用 `TextIteratorStreamer` 在**后台线程**跑推理，正是为了解决这个问题。

**C/C++ 背景在这个领域的优势。** 不需要从零学起。以下是已有的 C/C++ 知识和 AI 工程概念的对应关系：

| C/C++ 已有的知识 | 对应 AI 工程概念 | 在哪一步用到 |
|------|------|:---:|
| 内存管理（malloc/free、内存池） | PagedAttention 的 Block Table（逻辑块→物理块映射 = 虚拟内存页表） | Step 4 |
| 生产者-消费者、线程同步 | TextIteratorStreamer 的推理线程 + 主线程消费 | Step 2 |
| 并发调度、请求队列 | Continuous Batching 的 Scheduler（请求入队→动态打包） | Step 4 |
| 缓存行、数据局部性 | KV Cache 的显存排布和访问模式 | Step 4 |
| 字符串编码（UTF-8、字节→字符） | Tokenizer（文本→token id→文本，同样的编解码思想） | Step 1 |
| 性能压测（perf、valgrind） | Benchmark 脚本设计（预热、多次采样、P50/P99） | Step 1 |
| 哈希表、倒排索引 | Embedding + 向量检索（ANN 索引） | Step 3 |

**前置技能自查。** 开始前确认能做到以下 4 件事：① 在终端里 `cd` 到项目目录并运行 `python xxx.py` ② 创建 venv 并 `pip install` 一个包 ③ 用 `git clone` 拉一个仓库 ④ 读懂 Python traceback：先看最后一行的异常类型和消息，再看其上方最靠下的业务代码栈帧，随后沿调用链向上定位输入来源。

**完美主义执行法则。** 本计划的设计哲学是"小闭环、快速交付、80% 即合格"。每个任务标注了预计时间，如果超时 1.5 倍还没完成，写一句"未完成"标记并跳过。做完永远比做完美重要。面试官更想听到一个完成了 80% 但有反思的项目，而不是一个"还在打磨"的完美计划。

**调试习惯。** 遇到异常先读 traceback 的最后一行，确认异常类型和消息；栈帧按调用展开，通常最靠下的业务代码帧最接近抛错位置。随后结合局部变量、日志或断点验证，不要仅凭栈顶/栈底猜原因。

这个对照表的目的不是在每个细节上对应，而是让读者在学每个 Step 时意识到：**底层思想已经掌握了，只需要换一层 Python API**。每个 Step 末尾的"C++ 背景加分讨论"会具体展开这种对应关系。

（读完本章后可用以下速查）

> **术语速查（看到不懂的词先翻这里）**：RAG = 先搜索文档再让 AI 回答 | SSE = 服务端逐字推送 | TTFT = 首 token 延迟 | TPOT = 后续每个 token 平均间隔 | Tokenizer = 把文字转成数字的工具 | PagedAttention = 把显存分页管理（像操作系统虚拟内存） | Continuous Batching = 动态打包多个请求一起推理 | KV Cache = 缓存推理中间结果避免重复计算 | Embedding = 把文字变成向量（一串数字）表示语义 | ChromaDB = 本地向量数据库 | HF = HuggingFace | venv = Python 虚拟环境，隔离项目依赖

---

## 二、为什么这样设计（基于真实 JD 的证据）

### 2.1 JD 来源

以下分析来自 2026 年 6 家公司的真实招聘要求（数据采集于 2026-06-17）。**注意**：这些 JD 用于提取 AI 工程岗位的通用技能要求模式，不代表目标雇主。实际投递优先面向 AI 初创和中厂，这些公司对学历门槛更低、对工程交付能力更看重。

- 腾讯 — 大模型推理后台开发工程师（深圳/北京）
- 百度昆仑芯 — 大模型推理架构研发工程师（北京/深圳）
- 小鹏汽车 — 大模型平台 & Infra 工程师（深圳/北京）
- 华为 — AI 推理框架研发工程师（深圳/北京）
- 字节跳动 — AI Platform 后端开发工程师（深圳）
- 天亿马 — AI 大模型平台工程师（深圳）

### 2.2 关键区分：两类岗位

| | 推理部署 / ML Platform | 推理优化工程师 |
|------|------|------|
| Python | ✅ 主力语言 | ✅ |
| HF Transformers / PyTorch | ✅ 必须 | ✅ |
| vLLM / SGLang 使用和原理 | ✅ 核心 | ✅ |
| OpenAI 兼容 API 协议 | ✅ 高频 | ✅ |
| FastAPI / Docker / Prometheus | ✅ 核心 | ⚠️ 次要 |
| SSE 流式输出 | ✅ 必须 | ⚠️ 次要 |
| C++ | ⚠️ 加分项 | ✅ 必须 |
| CUDA kernel 开发 | ❌ 不需要 | ✅ 必须 |
| 量化算法实现 | ❌ 不需要 | ✅ 必须 |
| **岗位数量** | **更多** | 少但薪资高 |

**本计划目标：AI 平台后端 / LLM 服务化 / 大模型应用工程**（岗位多、不写 CUDA、不做算法研究）。推理部署 / ML Platform 相关技能作为加分，不作为第一主线——与 §1.5 的 v9 定位一致。

> **关键澄清**：这个方向的核心竞争力是 **"能把开源模型跑起来、包成可观测的 API 服务、容器化部署、写出 CI/CD。整个过程有代码和数据，不是纸上谈兵"**。这正是纯算法背景候选人普遍缺的工程能力，也是 C/C++ 后端经验的直接延伸。不必跟硕士/博士竞争算法岗。竞争的岗位，后者大多数做不了。

> **学习 vs 生产：** Step 2 用 HF Transformers + FastAPI 练流式、线程模型与观测，是为了把链路讲清。生产 GPU 上跑高并发推理时，主进程更常见是 vLLM/SGLang（或等价 Serving）；FastAPI 多作网关/编排。不要把「学习样本」误写成「生产标配架构」。

### 2.3 JD 技能要求 vs 本计划覆盖

| JD 核心要求 | 出现频率 | 对应 Step | 说明 |
|------|:---:|:---:|------|
| Python | 6/6 | 全流程 | 主力语言 |
| HuggingFace Transformers / PyTorch | 6/6 | Step 1 | 基础技能，JD 必选项 |
| vLLM | 5/6 | Step 4 | 高频要求，侧重架构理解 |
| SGLang | 3/6 | Step 4（概念对比） | 不实操，概念层了解与 vLLM 的差异 |
| Docker | 6/6 | Step 2 | 服务化标配 |
| FastAPI / HTTP 服务化 | 隐含要求 | Step 2 | 服务化日常；生产推理进程常另用 vLLM 等 |
| OpenAI 风格 API | 常见接入形式 | Step 2 | vLLM、SGLang 与许多 LLM 网关提供不同程度的兼容实现 |
| SSE 流式输出 | 4/6 | Step 2 | 腾讯 JD 明确要求 |
| Prometheus / 可观测性 | 4/6 | Step 2 | 腾讯、小鹏 JD 高频 |
| KV Cache 原理 | 高频 | Step 4 | 手算验证，不止于概念 |
| PagedAttention / Continuous Batching | 高频 | Step 4 | 百度 JD："了解架构原理" |
| TTFT / TPOT 指标 | 高频 | Step 2-4 | LLM 推理标准延迟指标 |
| 量化 (AWQ/GPTQ) | 4/6 | Step 1（实操 8-bit） | 概念 + 代码层面都接触 |
| RAG | 中频 | Step 3 | 企业落地常见场景 |
| 多模型经验 | 隐含要求 | Step 1（2 个模型） | 展示不只会上手一个模型 |
| Kubernetes | 4/6 | Step 2（Minikube 实操） | 用 Minikube 覆盖基础部署、配置管理和端口转发 |
| CUDA | 4/6 | ❌ 跳过 | 推理优化专属，不在目标范围 |

### 2.4 训练路线选择结论

本计划选择”AI 应用工程 + 推理服务化优先”。

原因有三点：

1. **和目标岗位匹配**：AI 应用开发、AI 平台后端、大模型应用开发更常要求 Python、FastAPI、Docker、K8s、RAG、OpenAI API、HF/vLLM 使用经验。
2. **和资源约束匹配**：无本地 GPU 时，HF 小模型推理、FastAPI 服务化、RAG、CPU 模式 vLLM 学习都能推进。
3. **和简历证据匹配**：每个 Step 都能产出可展示代码、benchmark 数据、README、踩坑记录，而不是只停留在概念学习。

v9 后最终训练路径收缩为：

```
Ownership Recovery
  → HF Transformers 基础推理
  → FastAPI LLM 服务化
  → RAG 应用落地 / vLLM 推理架构加分（二选一）
```

---

## 三、Step 0：Ownership Recovery（新增，P0）

> **定位**：这一步的目的是确认项目 ownership。如果跳过这一步，后续很可能继续变成 AI 生成脚本、用户负责运行命令，最后面试讲不深。

### 3.1 目标

用 7 天把已有 `hf-llm-benchmark` 从"AI 生成的脚本集合"变成"可解释、可验证、可最小重写的学习项目"。

### 3.2 任务清单

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 0.1 | 代码盘点 | 列出当前仓库已有脚本，每个脚本只写一句话：输入是什么、输出是什么、解决什么问题 | 30min |
| 0.2 | 选一个最小入口 | 只选 `explore_tokenizer.py` 或最简单的单次推理脚本，不同时看多个文件 | 15min |
| 0.3 | 手动追踪数据流 | 打印 prompt → chat_template → input_ids → model.generate → decode 的每一步输出 | 1.5h |
| 0.4 | 最小重写 | 新建 `scratch/minimal_infer.py`，不复制原脚本，只写 30-50 行跑通一次 Qwen 推理 | 2h |
| 0.5 | ownership 笔记 | 写 `notes/ownership.md`，回答 5 个问题：脚本解决什么、核心 API 是什么、最容易错在哪里、如何验证、删掉 AI 后可重写哪部分 | 1h |
| 0.6 | 面试口述 | 用 5 分钟口述 tokenizer、chat_template、generate、decode 的链路，讲不顺就回到 0.3 | 1h |

### 3.3 验收标准

- [ ] 能不看 AI 解释 `tokenizer.apply_chat_template()` 做了什么
- [ ] 能说清 `input_ids` 和 `attention_mask` 的含义
- [ ] 能画出一次 `model.generate()` 的最小数据流
- [ ] 能自己写一个 30-50 行的最小推理脚本
- [ ] `notes/ownership.md` 完成，不少于 500 字

### 3.4 停止条件

如果 7 天后仍然无法完成最小重写，不继续扩大项目范围。此时应把 AI 工程落地优先级从 P0 降到 P1，先投平台工程 / 基础软件 / 数据库产品工程岗位，用外部面试反馈重新判断方向。

---

## 四、Step 1：HuggingFace Transformers LLM 推理基础

### 4.1 目标

用 HuggingFace Transformers 加载 Qwen3-0.6B 和 Llama-3.2-1B 两个大语言模型，深入理解分词器、生成流程、参数影响、量化效果，编写 benchmark 脚本测量推理性能。

### 4.2 为什么这是第一步

- 6/6 JD 要求 PyTorch / HuggingFace Transformers，这是 AI 工程落地的入门必备
- 所有 LLM 推理框架（vLLM、SGLang）都在此基础上构建
- 使用 2 个不同模型，展示处理多模型的通用能力
- 全程 CPU 可完成

### 4.3 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| 主模型 | Qwen3-0.6B（`Qwen/Qwen3-0.6B`，~1.2GB） | 小尺寸、Apache 2.0 协议，适合 CPU 学习 |
| 对比模型 | Llama-3.2-1B-Instruct（~2.5GB） | Meta 开源，不同 tokenizer 体系（BPE vs 中文优化），展示多样性 |
| 框架 | HuggingFace Transformers | 行业标准，JD 必选项 |
| 精度 | FP32 + 8-bit（BitsAndBytesConfig） | CPU 推理默认 + 量化入门 |
| 模型下载 | HF Mirror (`hf-mirror.com`) 或 ModelScope | 国内网络必需 |
| 语言 | Python 3.10+ | 岗位要求 |

### 4.4 环境准备：国内下载指南（重要）

HuggingFace Hub 在国内直接下载可能极慢或失败。两种方案：

**方案 A：使用 HF Mirror（推荐）**
```bash
export HF_ENDPOINT=https://hf-mirror.com
# 此后 huggingface_hub 的所有下载自动走镜像
```

**方案 B：使用 ModelScope**
```bash
pip install modelscope
# modelscope download --model Qwen/Qwen3-0.6B
```

### 4.5 可参考项目

| 参考 | 类型 | 亮点 | 链接 |
|------|------|------|------|
| HF 官方 text-generation 教程 | 官方文档 | pipeline() → AutoModel → generate() 完整流程 | huggingface.co/docs/transformers |
| CSDN "5分钟快速上手LLM" | 中文博客 | 2026.6 新鲜，含量化/chatbot | blog.csdn.net/m0_65555479 |
| huggingface-llm-examples | GitHub | 多模型 benchmark，自动生成 Excel 报告 | github.com/ogunerkutay/huggingface-llm-examples |
| Gemma-3-1B Pipeline | 博客 | 2026.4，含 mini-benchmark 和延迟测量 | marktechpost.com |

### 4.6 详细任务清单

> v9 保留 ownership 原则：Step 1 不再从"让 AI 生成完整 benchmark 脚本"开始，而是从"手动理解一个最小推理链路"开始。任何 AI 生成代码都必须经过解释、删减、重写、验证四步。

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 1.1 | 最小推理链路复现 | 基于 Step 0 的 `minimal_infer.py`，只跑一条 prompt，并打印 prompt / chat_template / input_ids / output_ids / decoded text | 1h |
| 1.2 | 模型文件结构 | 只看 Qwen3 的 `config.json`、`tokenizer.json`、`model.safetensors`，先解释清楚每个文件的角色；Llama 对比放到后面 | 1h |
| 1.3 | 分词器显微镜 | 用 3 条中英文句子观察 token 数、特殊 token、attention_mask；输出 `results/tokenizer_notes.md` | 2h |
| 1.4 | 单次推理跑通 | 加载 Qwen3 → `tokenizer.apply_chat_template()` → `model.generate()` → decode，打通完整推理流程；再切换到 Llama 跑同一条 prompt | 1h |
| 1.5 | 生成参数实验 | Qwen3 上系统对比：① temperature(0.1/0.7/1.5) ② top_p(0.5/0.9/1.0) ③ max_new_tokens(32/128/512) ④ do_sample(True/False)。记录每组输出质量和 tokens/s | 3h |
| 1.6 | Benchmark 最小版 | 先只做 3 条 prompt × 3 次，输出 avg/P50/P99/tokens/s；确认统计逻辑自己讲得清，再扩展到完整版 | 2h |
| 1.7 | 双模型对比 | 加入 Llama，对比同一句中文的 token 数和推理速度，记录一个明确结论 | 1.5h |
| 1.8 | 量化实验 / 理论推算 | CPU 上 8-bit 如有环境问题，不硬卡；允许改为参数量 × 字节数的理论内存推算，并诚实写入 README | 1h |
| 1.9 | 结果整理 + 提交 | README + `notes/ownership.md` + tokenizer 笔记 + benchmark 结果。README 必须包含"哪些代码是 AI 辅助、哪些部分已能独立解释" | 1h |

### 4.7 产出物

```
hf-llm-benchmark/
├── README.md                      # 项目说明 + benchmark 结果
├── requirements.txt
├── benchmark.py                   # 核心 benchmark（支持 --model 参数）
├── explore_tokenizer.py           # 分词器实验（两模型对比）
├── explore_params.py              # 生成参数实验
├── explore_quantization.py        # 8-bit 量化对比
├── scratch/
│   └── minimal_infer.py            # 从空文件最小重写的一次推理
├── notes/
│   └── ownership.md                # AI 辅助边界 + 独立理解边界
└── results/
    ├── tokenizer_notes.md
    └── report.md
```

### 4.8 验收标准

- [ ] `python benchmark.py --model qwen` 一键运行
- [ ] 理解 tokenizer 的 chat_template 和特殊 token
- [ ] 能解释 temperature / top_p 的影响
- [ ] 能说出 Qwen3 和 Llama tokenizer 的一个具体差异（如中文切分方式）
- [ ] 量化实验：INT8 内存降幅有具体数字
- [ ] 报告包含两个模型的 avg/P50/P99 延迟和 tokens/s
- [ ] 能从空文件写出一个 30-50 行最小推理脚本
- [ ] `notes/ownership.md` 记录 AI 辅助边界和独立理解边界
- [ ] 能用 5 分钟口述一次 prompt → token → generate → decode → benchmark 的完整链路

### 4.9 面试一句话

> "用 HuggingFace Transformers 加载了 Qwen3-0.6B 和 Llama-3.2-1B 跑 benchmark，系统对比了生成参数对延迟和质量的影响，还做了 FP32 vs INT8 量化的内存/速度对比。过程中理解了 tokenizer 机制、chat_template 作用和自回归生成流程。"

### 4.10 对标的 JD 技能

Python ✅ | PyTorch/HF ✅ | Tokenizer ✅ | 多模型经验 ✅ | 量化(8-bit) ✅ | Benchmark ✅

### 4.11 执行提示

**CPU benchmark 太慢怎么办。** Qwen3-0.6B 在普通笔记本 CPU 上约 2-5 tokens/s，生成 512 token 需要 1-3 分钟。按任务 1.7 的 15 prompt × 10 次 = 150 次，一次 2 分钟 = 5 小时 wall clock。**建议首次运行时降低采样**：10 prompt × 5 次 = 50 次，先出数据。后续可以挂一晚上跑完整版。

**tokenizer 对比抓住一个点。** Qwen3 和 Llama tokenizer 对中文的处理差异是最容易观察的：同一句中文，Llama 的 token 数通常是 Qwen3 的 1.5-2 倍（因为 Llama 的 BPE 词表以英文为主）。把这个对比数字写进报告，面试时一句话就能展示"理解 tokenizer 对推理成本和速度的影响"。

**量化实验注意。** `BitsAndBytesConfig(load_in_8bit=True)` 在纯 CPU 环境下可能需要额外配置（默认依赖 CUDA）。如果遇到报错，备选方案（按优先级）：① 改 `torch_dtype=torch.float16` 加载（注意：PyTorch CPU 的 float16 需要较新版本才支持，如不支持则用方案②）② 不改代码，用模型权重文件大小推算理论内存差异：FP32 模型文件 ≈ 参数数量 × 4 字节，INT8 ≈ 参数数量 × 1 字节，文献计量即可说明量化效果。

**C++ 背景加分讨论。** Tokenizer 的本质是一次字符串编解码（encode → token ids → decode）。C++ 开发者对字符编码（UTF-8 code point → 字节序列）有天然的直觉。tokenizer 不过是把"字符→字节"的映射换成了"子词→token id"的映射，词表（`tokenizer.json`）就是这张映射表。面试时可以说："理解了 tokenizer 之后发现，它的核心就是个查表操作，和 C++ 里做 Unicode 编解码没有本质区别。中文 token 数比英文多的根本原因是词表覆盖度问题。"这个视角是纯 Python 背景的候选人很难自然想到的。

---

## 五、Step 2：LLM 流式服务化（P0 主体）

### 5.1 目标

用 Step 1 的模型和推理代码，在 FastAPI 上构建一个可交付的 LLM 流式推理服务。第一阶段不追求完整平台，只证明三件事：

1. 模型能被包装成 OpenAI Chat Completions 风格的兼容子集。
2. 服务能流式输出，并解释为什么要用后台线程。
3. 当用户说"接口慢"时，能用 TTFT/TPOT、日志和指标定位问题。

### 5.2 为什么是第二步

- 腾讯 JD："流式推理""可观测性体系"
- 字节 JD："端到端流程"
- `/v1/chat/completions` 是常见接入形式，但不同引擎支持的字段与流式语义并不完全一致
- 在 Step 1 模型基础上叠加服务化，最贴近 AI 平台后端 / LLM 服务化岗位
- 这一步训练的是把链路收敛为可运行、可观测、可交付服务的能力

### 5.3 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| Web 框架 | FastAPI | 异步原生支持 SSE StreamingResponse |
| 模型加载 | HuggingFace Transformers | 沿用 Step 1 |
| 流式生成 | `TextIteratorStreamer`（transformers 内置） | HF generate() 是阻塞的，Streamer 通过后台线程逐 token 产出 |
| 流式协议 | SSE (Server-Sent Events) | LLM 服务标准输出方式 |
| API 格式 | 自定 `/generate` + `/v1/chat/completions` **兼容子集** | 便于复用常见客户端，同时明确实现边界 |
| 指标 | prometheus_client | JD 高频 |
| 压测 | Python asyncio + httpx（P1 加分） | Locust 对 SSE 支持不足，asyncio 更可控 |
| 容器化 | Docker + docker-compose | 先做 Docker 交付，K8s 放到 P1 |

> **为什么不用 Locust？** Locust 没有原生 SSE 支持，需要大量自定义代码才能解析 SSE event 并计算 TTFT。直接用 asyncio + httpx 写并发脚本更可控、代码更少、逻辑更透明。

### 5.4 可参考项目

| 参考 | 类型 | 亮点 |
|------|------|------|
| FastAPI SSE 教程 | 官方文档 | StreamingResponse 实现 |
| vLLM API Server 源码 | 源码 | `/v1/chat/completions` 请求/响应格式 |
| OpenAI API 文档 | 规范 | chat/completions 的标准请求体和响应体 |

### 5.5 问题单驱动拆分

#### Step 2A：服务 MVP（P0）

**问题单**：怎么把本地模型从"脚本调用"变成"可被业务系统调用的 HTTP 服务"？

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 2A.1 | 项目初始化 | 创建 `llm-inference-service/`，虚拟环境，复用 Step 1 模型路径 | 0.5h |
| 2A.2 | 模型层封装 | `model.py`：封装模型加载与 `generate_stream()`，只支持 Qwen 单模型 | 1.5h |
| 2A.3 | 流式原型 | `/health` + `/generate`，用 SSE 返回增量文本，`curl` 可看到流式结果 | 2h |
| 2A.4 | Chat Completions 兼容子集 | `/v1/chat/completions` 支持 `messages` 与 `stream: true`；README 明确暂不支持的字段 | 2h |
| 2A.5 | MVP README | 写清启动方式、curl 示例、当前边界 | 1h |

**验收标准**：
- [ ] `uvicorn app.main:app` 可启动
- [ ] `/health` 返回模型状态
- [ ] `/v1/chat/completions` 支持 SSE 流式输出
- [ ] README 有一条可复制运行的 curl 示例

#### Step 2B：可观测性闭环（P0）

**问题单**：如果接口很慢，如何判断时间花在网关、排队、Prompt 构建、Prefill、Decode 还是传输？

> 指标分类与慢请求拆法总表见 [ai-engineering.md §3.4](ai-engineering.md)。本 Step 先落 **学习路径最小集**；上生产后再对齐 vLLM 原生队列/KV 指标。

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 2B.1 | request_id | 中间件生成 request_id，写入响应头和日志 | 0.5h |
| 2B.2 | 分段 JSON 日志 | 每个请求输出 `{request_id, model, prompt_tokens, output_tokens, queue_ms, prefill_ms, ttft_ms, tpot_ms, total_time_ms, status}`；学习版拿不到的阶段标为 `null`，不伪造 | 1h |
| 2B.3 | TTFT/TPOT | 服务端在生成器边界记录首个模型 token 与后续 token 时间；另记首个有效 SSE 内容事件，避免把 role/元数据包当首 token | 1.5h |
| 2B.4 | `/metrics` | prometheus_client 暴露：请求数、TTFT、TPOT、tokens/s、错误计数 | 2h |
| 2B.5 | 慢请求案例 | 人为构造长/短 prompt 与长/短输出，区分 Prompt 长度影响 TTFT、输出长度影响总耗时；不能仅凭 TTFT/TPOT 就断言底层根因 | 1h |
| 2B.6 | 生产指标对照 | 在 notes 里列「学习指标 ↔ Serving runtime 指标」：queue wait、running/waiting、scheduled tokens、batch size、KV 占用、preemption | 0.5h |

**验收标准**：
- [ ] 每条日志都有 request_id
- [ ] 能说清 TTFT 与 TPOT 的区别，以及和 HTTP P99 的关系
- [ ] `/metrics` 能看到至少 3 个指标
- [ ] README 有一段「如何定位慢请求」（对齐 engineering §3.4 拆法）
- [ ] 不把「延迟好看」写成「回答质量好」（质量见 Step 3 / intro §3.6）

#### Step 2C：交付闭环（P0）

**问题单**：如果把这个服务交给别人，能否一键启动、验证、知道出问题去哪看？

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 2C.1 | Dockerfile | 基于 python slim 镜像，安装依赖，启动服务 | 1h |
| 2C.2 | docker-compose | 挂载模型目录和配置环境变量，避免把模型打进镜像 | 1h |
| 2C.3 | 故障记录 | README 记录至少 3 个踩坑：模型路径、依赖、流式输出、内存 | 1h |
| 2C.4 | 最小测试 | `tests/test_health.py` + 一个流式接口 smoke test | 1h |

**验收标准**：
- [ ] `docker-compose up` 可启动
- [ ] 容器内 `/health` 正常
- [ ] README 有"快速开始 / 验证 / 常见问题"三段
- [ ] 至少 1 个测试能验证服务活着

#### Step 2D：工程加分（P1）

**问题单**：如果要把这个服务放到团队环境里，下一步最该补什么？

按面试反馈和精力三选一，不强制全做：

| 选项 | 内容 | 适合场景 |
|------|------|----------|
| K8s 部署 | Minikube Deployment + Service + ConfigMap | 投 AI 平台 / DevOps / LLMOps |
| GitHub Actions | 构建镜像 + 启动容器 + 跑 smoke test | 投平台后端 / 工程效率 |
| 并发压测 | asyncio + httpx 统计并发下 TTFT/TPOT | 投推理服务 / 后端性能 |

**停止规则**：Step 2A-2C 没完成前，不做 Step 2D。

### 5.6 产出物

```
llm-inference-service/
├── README.md
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI + /health + /generate + /v1/chat/completions
│   ├── config.py          # pydantic-settings
│   ├── model.py           # HF 模型 + TextIteratorStreamer 流式生成
│   ├── schemas.py         # OpenAI 兼容 + 自定义 Pydantic 模型
│   └── middleware.py      # request_id + 日志 + Prometheus
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── .github/workflows/
│   └── ci.yml
├── tests/
│   ├── test_health.py
│   └── test_chat_completions.py
├── bench.py               # asyncio + httpx 并发压测
└── results/
    ├── slow_request_analysis.md
    └── load_test_report.md
```

### 5.7 验收标准

- [ ] Step 2A：OpenAI 兼容流式接口可用
- [ ] Step 2B：日志、request_id、TTFT/TPOT、`/metrics` 可用
- [ ] Step 2C：Docker 一键启动，README 可交付
- [ ] 能口述"请求进入 → 模型生成 → streamer 输出 → 指标记录 → SSE 返回"完整链路
- [ ] Step 2D 至少完成一个加分项（K8s / CI / 并发压测三选一，可选）

### 5.8 面试一句话

> "用 FastAPI 把本地 Qwen 模型封装成 Chat Completions 风格的流式服务，实现了 SSE 增量输出、request_id、分段 JSON 日志、TTFT/TPOT 指标和 Docker 交付。README 明确记录兼容子集与指标边界，并用长短 Prompt、长短输出实验分析延迟。"

### 5.9 对标的 JD 技能

FastAPI ✅ | SSE 流式 ✅ | OpenAI 兼容 API ✅ | TTFT/TPOT ✅ | Docker ✅ | Prometheus ✅ | request_id / JSON logging ✅ | K8s/CI/压测（P1 加分）✅

### 5.10 执行提示

**TextIteratorStreamer 的正确用法。** `TextIteratorStreamer` 的工作模式是：创建 streamer（内部用队列传递已解码文本）→ 传给 `model.generate(streamer=streamer)` → **在后台线程**调用 generate → 请求处理侧迭代 streamer 并转发结果。

不要先同步等待 `generate()` 返回，再开始迭代 streamer。`generate()` 是阻塞调用；这样写会等生成全部结束后才消费，失去实时流式效果，并阻塞 FastAPI 事件循环。后台线程让生成与消费并行。是否出现队列阻塞取决于具体 streamer 实现与配置，不应把它固定解释为「有限队列导致死锁」。

最可靠的参考是 transformers 仓库的测试文件 `tests/test_streamers.py`，在 GitHub 上搜索 "TextIteratorStreamer test" 就能找到官方用法。

**asyncio 压测脚本的常见坑。** 流式请求的超时需要区分连接超时和读取超时。首个有效内容可能数秒后才到，因此读取超时通常应高于连接超时。`asyncio.gather()` 默认会把首个异常传播给等待方；若希望所有任务都跑完并把异常作为结果收集，才使用 `return_exceptions=True`，随后逐项检查。

**为简历加分：在 README 加一段"多模型版本管理"讨论。** 不需要写代码。在 README 末尾加一段 200 字的讨论："如果线上同时服务 v1 和 v2 两个版本的模型，可以怎么做：模型文件按版本号目录存储 → FastAPI 根据请求中的 `model` 参数路由到对应实例 → Prometheus 指标按 `model_version` 标签区分"。4/6 JD 隐含对模型版本管理的期望，这段讨论证明思考过生产环境的问题。

**Docker 多 worker 的内存陷阱。** Gunicorn 多 worker 模式下，每个 worker 各自加载一份模型到内存。0.6B FP32 约 2.4GB，4 workers ≈ 10GB。`docker-compose.yml` 如果限制内存不足会 OOM。建议：本地测试用 1-2 workers，压测对比时在文档中说明"生产环境需根据模型大小调整 worker 数"。

**C++ 背景加分讨论。** Python 的 `async/await` 和 C++20 协程虽然语法相似，但运行模型不同。Python 是单线程事件循环，C++ 协程可以分布到线程池。理解这一点后，可明白为什么 FastAPI 里**不能在 async 函数中直接调 `model.generate()`**（阻塞事件循环），以及为什么 `TextIteratorStreamer` 用后台线程是一个必要的设计选择。面试时："用 asyncio + httpx 写压测脚本时，处理了流式请求的连接超时和读取超时分离，这和 C++ 里做非阻塞 I/O 时的 socket timeout 分层是同一个思路。"这个系统级调试经验，是只做 Python 的候选人不容易具备的。

---

## 六、Step 3：RAG 问答系统（P1 推荐）

> **定位**：RAG 是 Step 2C 之后的优先加分项，比 vLLM 更贴近用户当前 P0 方向（AI 应用工程 / LLM 服务化）。它能放大用户的文档整理、复杂信息结构化、评估闭环能力。

### 6.1 目标

构建端到端 RAG 问答系统：文档切片 → Embedding 向量化 → ChromaDB 检索 → LLM 生成带引用答案，含检索质量评估。LLM 生成部分默认复用 Step 1 的 Qwen3-0.6B，不强制外部 API。

**期望管理（重要）：** 0.6B 本地生成验证的是 **管道与指标**（切片 → 检索 → 拼 Prompt → 生成 → recall/引用检查），不是 SOTA 问答质量。答不准时，先用故障注入分清是切片、检索还是生成能力不足，再决定要不要把生成换成更大模型或 API。

**问题单**：当 RAG 回答不准时，问题到底出在文档切片、向量检索、Prompt 拼接，还是 LLM 生成幻觉？

### 6.2 前置条件

- 完成 Step 2（服务化经验）
- 全部 CPU 可完成
- `pip install chromadb sentence-transformers`
- 复用 Step 1 下载的 Qwen3-0.6B 模型

### 6.3 详细任务清单

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 3.1 | 准备知识库 | 选取文档集（建议把本目录 [ai-intro.md](ai-intro.md) / [ai-engineering.md](ai-engineering.md) 按章节拆成 10-20 块），转纯文本 | 1h |
| 3.2 | 文档切片 | 按段落/标题分块，每块 200-500 token，50 token 重叠；输出 chunk 列表（文本+来源+位置） | 2h |
| 3.3 | Embedding + 向量存储 | 用 BGE-small 或 all-MiniLM-L6-v2 转向量 → 存入 ChromaDB，支持增删查 | 2h |
| 3.4 | 检索 pipeline | `retrieve(question)` → 向量化 → ChromaDB top-k 检索 → 返回 chunk + 相似度分数 | 1.5h |
| 3.5 | 生成 pipeline | 检索结果 + 用户问题拼 Prompt → 调**本地 Qwen3-0.6B**（复用 Step 1 模型）→ 返回带引用来源的答案 | 2h |
| 3.6 | 评估 | 准备 **同一套** 10 个测试问题（含标准答案来源 chunk）→ 评估：① recall@3/5 ② 引用准确率（自动比对）③ 人工抽查这 10 条是否忠实于检索（无胡编、无过度推断）。**强制**再做 2–3 个故障注入（坏切块 / top-k 过小 / 坏 Prompt），写清指标如何变化。完整 RAGAS / LLM-as-Judge 不在必做范围，但 README 要点名与 intro §3.6 三类指标的对应 | 2.5h |
| 3.7 | FastAPI 服务化 | 复用 Step 2 经验：`/ask` 端点 → 检索+生成 → JSON（含答案+引用+耗时） | 2h |
| 3.8 | LangChain 等价实现对比 | 用 LangChain 的 `RecursiveCharacterTextSplitter` + `Chroma` + `HuggingFaceEmbeddings` + `create_retrieval_chain` 重写 Step 3 的核心链路（仅对比，不替代手写版）。产出：一份 300 字对比笔记：LangChain 封装了什么、手写版多理解到的细节。目的是：① 简历上出现 LangChain 关键词 ② 面试时能说"两版都写过" | 2h |
| 3.9 | 结果整理 + 提交 | README + 10 问评估结果 + 架构图 + LangChain 对比笔记 | 1h |

### 6.4 产出物

```
rag-qa/
├── README.md
├── requirements.txt
├── data/docs/              # 原始文档
├── scripts/
│   ├── chunk.py
│   ├── embed.py
│   └── evaluate.py
├── app/
│   ├── main.py            # FastAPI /ask
│   ├── retriever.py
│   ├── generator.py       # 调用本地 Qwen3-0.6B
│   └── prompt.py
└── results/
    └── eval_report.md
```

### 6.5 验收标准

- [ ] 上传新文档后可被检索
- [ ] `curl /ask` 返回带引用来源的答案
- [ ] 评估报告含 recall@3/5、引用准确率、人工忠实度抽查（同一套 10 题）
- [ ] 至少 2 个故障注入实验，能说明「坏在切片 / 检索 / 生成」哪一层
- [ ] README 写明：0.6B 验证的是管道与指标，不是 SOTA 问答质量
- [ ] 生成 pipeline 使用本地模型（非外部 API）；若对比过 API 生成，单独注明

### 6.6 面试一句话

> "搭建了端到端 RAG 系统：文档切片 → BGE-small Embedding → ChromaDB 检索 → 本地 Qwen3-0.6B 生成带引用答案，召回率 recall@3 达到 X%。也用 LangChain 重写了核心链路，能说清框架封装了什么、手写版多理解到的细节。"

### 6.7 对标的 JD 技能

RAG ✅ | Embedding ✅ | ChromaDB ✅ | 检索评估 ✅ | FastAPI 服务化 ✅ | LangChain ✅

### 6.8 执行提示

**文档切片的质量比数量重要。** 切片太小（< 100 token）会丢失上下文，太大（> 1000 token）会稀释检索精度。建议先用 [ai-intro.md](ai-intro.md) 按自然章节边界切，再人工检查每个 chunk 是否能独立理解。面试时常被问："如何决定 chunk size？"答案是"按文档的自然边界切，同一主题的内容不跨 chunk，然后再用 overlap 保证边界附近的上下文不丢失"。

**C++ 背景加分讨论。** 向量检索的核心，ANN（近似最近邻）索引，在数据结构层面和 C++ 里做空间索引（KD-Tree、R-Tree）是同一类问题。Embedding 向量是高维空间中的点，ChromaDB 的 HNSW 索引和 C++ 里做最近邻搜索是同一类算法。不需要实现这些算法，但面试时："Embedding 检索本质是一个高维空间最近邻搜索问题。这和游戏引擎里用 KD-Tree 做碰撞检测在数学上是同一个问题，只是维度从 3 变成了 768。"这个类比展示了跨领域的架构直觉。

---

## 七、Step 4：vLLM 概念学习（P2 加分项）

> **定位**：本 Step 是 P2 加分项。AI 应用工程岗对 vLLM 的要求通常是"知道它解决什么问题，会用 OpenAI 兼容服务，能讲核心概念"，不是读源码。优先完成 Step 0-1-2C 和 Step 3，再考虑本 Step。

### 7.1 目标

**核心路径（CPU，6-10h）：** 跑通 vLLM CPU 模式，阅读官方架构文档，能讲清 PagedAttention、Continuous Batching、KV Cache 的基本含义。产出概念笔记。

**强化路径（租 GPU 2-4h，约 10-20 元，强烈建议）：** 在 T4 上跑通 vLLM GPU 模式，做 HF vs vLLM 的真实 GPU benchmark 对比。这是投入产出比最高的改进，产出一份 GPU 对比数据，面试时竞争力提升一档，用实验数据验证理论推演的结论。产出：一份有真实 GPU 数据的对比报告。

**执行原则**：只做到够面试解释，不把它变成新的源码阅读大坑。

### 7.2 最低交付标准（做什么就够）

**最低交付足以应对 AI 应用开发岗面试：**

1. 用 `vllm serve` 启动模型，调通 `/v1/chat/completions` ✅
2. 阅读官方 Architecture Overview 文档，能画出 LLMEngine → Scheduler → Worker 调用链 ✅
3. 用自己的话讲清 PagedAttention（Block Table + 逻辑→物理映射 + Copy-on-Write） ✅
4. 用自己的话讲清 Continuous Batching（请求队列 → 动态打包 → 为什么比 static batching 好） ✅
5. 手算一次 KV Cache（用 Qwen3-0.6B 的 config 参数） ✅
6. 能说出 vLLM 和 SGLang 的一个核心差异 ✅

以上约 6-10h 可完成。源码精读和 GPU benchmark 都是有余力再做，不进入 P0。

### 7.3 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| 推理框架 | vLLM（CPU 跑通 + GPU 实验） | CPU 完成核心学习，GPU 验证理论 |
| 模型 | Qwen3-0.6B（CPU）/ Llama-3.2-1B 或 Qwen3-1.7B（GPU） | CPU 用小模型快速跑通；GPU 上有显存余量可用更大模型 |
| 源码重点 | 只定位 `scheduler.py`、`block_manager.py` 的入口 | 不逐行读，只建立概念映射 |
| 概念对比 | SGLang | 3/6 JD 提及，了解核心差异 |
| GPU 租用 | AutoDL T4 16GB（~2元/h） | 2-4h 足够完成实验验证 |

### 7.4 可参考资源

| 参考 | 类型 | 亮点 |
|------|------|------|
| [vLLM 官方文档 - Architecture Overview](https://docs.vllm.ai/en/latest/design/arch_overview.html) | 官方文档 | 架构全景图，必读 |
| [vLLM 源码 - vllm/worker/model_runner.py](https://github.com/vllm-project/vllm) | 源码 | 模型执行的核心入口 |
| [vLLM 源码 - vllm/core/scheduler.py](https://github.com/vllm-project/vllm) | 源码 | 请求调度和抢占逻辑 |
| [vLLM 源码 - vllm/core/block_manager.py](https://github.com/vllm-project/vllm) | 源码 | PagedAttention 内存管理 |
| [PagedAttention 论文 (vLLM)](https://arxiv.org/abs/2309.06180) | 论文 | 理论基础 |
| [SGLang 官方文档](https://sglang.readthedocs.io/) | 文档 | 与 vLLM 的架构差异对比 |
| Red Hat vLLM CPU 教程 | 技术文章 | CPU 模式部署参考 |
| [VLLM-K8s-Deployment](https://github.com/doronmak/VLLM-Kubernetes-Deployment) | GitHub | Qwen3-0.6B + vLLM CPU 参考 |

### 7.5 详细任务清单

#### 核心路径（CPU，6-10h）

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 4.1 | vLLM CPU 环境搭建 | 按当前 vLLM CPU 安装文档选择 CPU wheel / 专用 index 或源码构建（通用 `pip install vllm` 不保证得到可用的 CPU 后端）；启动 `vllm serve Qwen/Qwen3-0.6B --host 0.0.0.0 --port 8000`，curl 调通 `/v1/chat/completions` | 1.5h |
| 4.2 | 离线推理跑通 | Python 调 `vllm.LLM` 类 → `llm.generate([...])` 验证输出，对比与 Step 1 HF 的输出差异（同一 prompt、temperature=0） | 1h |
| 4.3 | 阅读官方架构文档 | 精读 [vLLM Architecture Overview](https://docs.vllm.ai/en/latest/design/arch_overview.html)，理解：LLMEngine → Scheduler → Worker → ModelRunner 的调用链；Centralized Controller 设计；KV Cache Manager 的角色 | 1.5h |
| 4.4 | PagedAttention 概念笔记 | 理解 Block Table、逻辑块→物理块映射、Copy-on-Write，不逐行读源码 | 1.5h |
| 4.5 | Scheduler 概念笔记 | 理解请求队列、动态 batching、为什么比 static batching 好，不逐行读源码 | 1.5h |
| 4.6 | KV Cache 理论计算 | 取 Qwen3-0.6B 的 config，用 [ai-engineering.md §2.4](ai-engineering.md) 的公式手动计算 512 token 输入的 KV Cache 大小 | 1h |
| 4.7 | SGLang 概念对比 | 阅读 SGLang 文档，了解 RadixAttention 与 PagedAttention 的核心差异 | 1h |
| 4.8 | 理论推演文档 | 假设场景（7B、100 并发、2000in+500out、A10 GPU），推演 HF 瓶颈 → vLLM 解决方案 → CPU 局限 | 2h |
| 4.9 | 核心路径结果整理 | 概念笔记：架构全景图 + PagedAttention + Scheduler + KV Cache 手算 + SGLang 对比 | 1h |

#### 强化路径（租 GPU 2-4h，约 10-20 元，可选）

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| G.1 | 租 GPU + 环境搭建 | AutoDL 租 T4 16GB，安装 vLLM，下载模型（建议 Llama-3.2-1B 或 Qwen3-1.7B，利用 GPU 显存余量） | 0.5h |
| G.2 | GPU benchmark 实验 | 用与 Step 1 相同的 benchmark 脚本 + 同一批 prompt，**在 GPU 上跑 HF 和 vLLM 两组实验**。记录：TTFT、TPOT、tokens/s、GPU 显存占用、GPU 利用率。**关键：对比不同并发（1/16/32/64）下两框架的吞吐差异** | 2h |
| G.3 | 实验结果与理论推演对照 | 对比 G.2 的实验数据 vs 4.8 的理论推演结论：推演中预测的趋势是否在实验中验证？为什么有偏差？输出一份 1-2 页的"理论 vs 实验"对照笔记 | 1h |
| G.4 | GPU 结果整合到主文档 | 将 GPU 实验结果补充到架构分析文档，README 中突出展示核心基准数字 | 0.5h |

> **GPU 实验策略**：先在 CPU 上完成核心路径的源码学习和理论推演 → 准备好 benchmark 脚本 → 租 GPU 一次性跑完所有实验（2-4h 连续工作）→ 关机释放实例。不要边学边开着 GPU 烧钱。

### 7.6 产出物

```
vllm-architecture-study/
├── README.md                        # 架构分析报告入口 + GPU 实验结果（如有）
├── notes/
│   ├── 01_arch_overview.md          # 架构全景 + 调用链
│   ├── 02_paged_attention.md        # PagedAttention 概念笔记
│   ├── 03_scheduler.md              # Scheduler 概念笔记
│   ├── 04_kv_cache_calc.md          # KV Cache 手算过程
│   ├── 05_sglang_comparison.md      # 与 SGLang 的概念对比
│   ├── 06_scenario_analysis.md      # 7B + 100 并发场景推演
│   └── 07_theory_vs_experiment.md   # 【GPU 强化】理论推演 vs 实验数据对照
├── scripts/
│   ├── vllm_serve.sh
│   ├── test_vllm.py
│   └── gpu_bench.py                 # 【GPU 强化】HF vs vLLM GPU benchmark
├── prompts/
│   └── test_prompts.jsonl           # 共享测试数据
├── results/
│   └── gpu_comparison.md            # 【GPU 强化】GPU 实验对比报告
└── diagrams/
    └── arch.mermaid                 # 架构图源码
```

### 7.7 验收标准

**核心路径（必达）：**
- [ ] `vllm serve` 成功启动并提供 OpenAI 兼容 API
- [ ] PagedAttention 概念笔记能用自己的话解释 Block Table 和 Copy-on-Write
- [ ] Scheduler 概念笔记能描述 Continuous Batching 的执行流程
- [ ] KV Cache 手算值与 config 推导一致
- [ ] 理论推演能说清楚：GPU 上 vLLM 为什么快、CPU 上为什么优势消失
- [ ] 能说出 vLLM 和 SGLang 的一个核心架构差异

**强化路径（可选）：**
- [ ] GPU benchmark 有 HF vs vLLM 在不同并发下的吞吐和延迟对比数据
- [ ] "理论 vs 实验"对照笔记，分析了预测偏差的原因
- [ ] README 中突出展示 GPU 实验核心数字

### 7.8 面试一句话

> "系统学习过 vLLM 架构：读过 scheduler 和 block_manager 源码，能讲清楚 PagedAttention 和 Continuous Batching。还在 T4 上做了 HF vs vLLM 的 benchmark 对比，在不同并发下 vLLM 吞吐提升 X 倍，KV Cache 碎片率从 Y% 降到 Z%。"

### 7.9 对标的 JD 技能

vLLM ✅ | PagedAttention ✅ | Continuous Batching ✅ | KV Cache ✅ | SGLang(概念) ✅ | 源码级理解 ✅ | GPU benchmark ✅

### 7.10 执行提示

**源码阅读的正确姿势：不要逐行读。** vLLM 代码库约 20 万行，`scheduler.py` 自己就 ~1500 行。目标是**理解核心设计决策**，不是读懂每一行。按这个顺序来：

1. **先看官方 Architecture Overview 文档**（约 10 页），建立全局视图
2. **Scheduler 入口**：打开 `scheduler.py`，搜索 `def _schedule(`，这是 Continuous Batching 的核心入口方法。理解它的输入（请求队列）、核心循环（逐个取出请求、分配到 block）、输出（scheduled requests + 空闲 block 列表）
3. **Block Manager 入口**：打开 `block_manager.py`，搜索 `def allocate(` 和 `def free(`，理解 Block Table 的两级映射（逻辑块→物理块）和 Copy-on-Write 的触发条件
4. **记三个关键设计决策**就够了：①为什么用 Block Table 两级映射（允许逻辑连续、物理分散）②Preemption 什么时候触发（新请求需要的 block 超过空闲数时，回收哪些旧请求的 block）③Continuous Batching 的打包逻辑（哪些请求可以放进同一个 batch）

**如果源码实在看不懂。** 搜索引擎搜索 "vLLM architecture explained" 或 "PagedAttention illustrated"，找带图的社区解读文章。理解了全局后再回来看源码对应位置。不要硬啃。

**理论推演加 2 个 2026 年热点段落（加分但不费时）。**

一、**MoE 模型推理**。在 4.8 的理论推演文档中加一段（200 字即可）："如果是 MoE 模型（如 DeepSeek V4，1.6T 总参数但每次只激活 49B），推理部署额外面临：①专家路由：每个 token 需要决定走哪几个专家，路由计算本身有时间开销 ②专家负载不均：热门专家（如通用语言专家）被大量 token 选中，成为瓶颈 ③EP（专家并行）：不同专家分布在不同 GPU 上，token 需要跨 GPU 通信"。百度/华为 JD 均涉及 MoE。

二、**PD 分离（Prefill-Decode Disaggregation）**。在上述文档再加一段（150 字）："PD 分离是 2026 年推理部署的热点架构。Prefill（计算密集）和 Decode（内存密集）拆分到不同的 GPU 实例上，各自独立扩缩容。好处是资源利用率更高（不会因为长 prompt 的 Prefill 占用 GPU 导致 Decode 排队），代价是 Prefill 产出的 KV Cache 需要跨网络传输给 Decode 实例"。百度 JD 明确提及 PD 分离。

> 以上两段不需要跑任何代码，纯概念讨论。写在理论推演文档中，面试被问到 MoE 或 PD 分离时，已有准备，比只盯着一个框架的候选人更全面。

**C++ 背景加分讨论：Step 4 是 C/C++ 开发者的隐藏优势。** PagedAttention 的本质是一个内存管理问题：KV Cache 按 block 分配，每个 block 有固定大小（如 16 个 token 的 KV 数据），Block Table 做两级映射（逻辑块→物理块）。这和操作系统的虚拟内存页表是**完全同构的设计**：逻辑上连续的 KV 序列，物理上可以分散在显存各处，按需分配、用完回收。vLLM 的 Preemption 策略（当新请求需要 block 但空闲不够时，选择收回某些旧请求的 block）本质上是内存回收策略，和操作系统的 page reclaim 逻辑一致。

面试时："PagedAttention 的 Block Table 两级映射，解决的是 KV 静态预分配带来的碎片浪费，思路和操作系统页表很像。Continuous Batching 则是动态组批，短请求不必干等长请求——吞吐往往卡在调度，而不只是卡在更大的 GPU。"

这是纯 Python 背景的候选人几乎不可能自然建立的连接。不需要写任何 C++ 代码，但这个视角展示了系统级思维，这正是 AI 工程落地和平台后端岗位看重的素质。

---

## 八、简历叙事汇总

### 8.1 项目线

| 项目 | 优先级 | 一行描述 | 关键词 |
|------|:---:|---------|--------|
| HF LLM Benchmark | P0 | 用 HF Transformers 加载 Qwen3-0.6B/Llama-3.2-1B 双模型做 benchmark，含量化对比 | PyTorch, HF, Tokenizer, 8-bit Quantization, Multi-model |
| LLM 流式服务化 | P0 | 构建 OpenAI 兼容 LLM 服务：SSE 流式、request_id、JSON 日志、TTFT/TPOT、Prometheus、Docker 交付 | FastAPI, SSE, OpenAI API, Docker, Prometheus, TTFT/TPOT, Observability |
| RAG 问答系统 | P1 | 文档切片→Embedding→ChromaDB→本地 LLM 生成，含检索质量评估 | RAG, Embedding, ChromaDB, LLM |
| vLLM 概念学习 | P2 | 跑通 vLLM serve，讲清 PagedAttention / Continuous Batching / KV Cache | vLLM, PagedAttention, Scheduler, KV Cache, SGLang |

### 8.2 完整叙事模板

> "有 N 年的 C/C++ 后端开发经验，擅长在复杂系统里收敛问题并推动交付。近期完成了一个 LLM 服务化项目：先用 HuggingFace Transformers 跑通本地 Qwen/Llama 推理，理解 tokenizer、chat_template 和 generate 流程；再用 FastAPI 封装成 Chat Completions 风格的流式接口，实现 SSE 增量输出、request_id、分段 JSON 日志、TTFT/TPOT、Prometheus 指标和 Docker 交付。项目能解释从请求到文本增量的链路，并通过分段数据分析首包、持续生成、模型加载或容器配置等问题。后续可补 RAG 检索增强和 vLLM 概念。"

### 8.3 目标岗位与搜索词

**第一梯队（技能匹配度最高，优先投递）：**
- `AI 平台后端开发`
- `LLM 服务化工程师`
- `Python 后端开发（AI 方向）`

**第二梯队（大部分匹配，部分需补 Agent/微调概念）：**
- `大模型应用开发`
- `RAG 开发工程师`
- `AI 应用开发工程师`

**第三梯队（技术兴趣方向，部分匹配）：**
- `ML Platform`
- `vLLM`

**关注公司类型**：AI 初创（20-200 人，对全栈工程能力需求最强）、中厂 AI 部门、大厂 AI 应用团队

> **投递策略**：做完 Step 0-1-2C 后先投第一梯队 + 第二梯队各 3-5 家，根据面试反馈确定最终方向。不要等 RAG / vLLM 全部做完再投。

---

## 九、常见问题

### Q1: 为什么不用 ONNX Runtime 做项目？

JD 中 vLLM/HF Transformers 出现频率远高于 ORT。ORT 适合边端/跨平台，与本计划主线不同；引擎与服务的概念区分见 [ai-engineering.md §1.4](ai-engineering.md)。简历上 HF Transformers +（概念层）vLLM 对应用/平台后端岗的命中率通常更高。

### Q2: Step 4 为什么降为 P2？

因为 P0 是 AI 工程落地。vLLM 很有价值，但它容易把注意力拉向源码、GPU、调度、性能优化这些更硬的方向。当前阶段只需要会用 `vllm serve`，能讲 PagedAttention / Continuous Batching / KV Cache 的作用即可。若你已会 RAG、主缺口是推理机制，可以把 Step 4 紧挨 Step 2 做，Step 3 后移。

### Q3: 用 0.6B 小模型，HR 会不会觉得浅？

单独看会。所以本计划用两件事来对冲：① Step 1 用两个不同模型（Qwen3 + Llama），展示多模型通用能力 ② Step 2 重点展示服务化、流式输出、可观测性和排障闭环。面试时展示的是工程落地能力，模型大小只是实验配置。Step 3 用 0.6B 时，明确告诉面试官：验证的是管道与指标，不是生成 SOTA。

### Q4: 要不要学 Kubernetes？

要了解，但不放进 P0。P0 先完成 Docker 交付和可观测性闭环。K8s 放在 Step 2D，作为投 AI 平台 / DevOps / LLMOps 时的加分项。不要让 K8s 抢占 LLM 服务化主线。

### Q5: 要不要学量化（AWQ/GPTQ 等）？

Step 1 已有 8-bit 量化的实操对比。更深入的量化算法实现（AWQ/GPTQ/SmoothQuant）是推理优化工程师的专业领域，服务化/平台后端岗知道"什么是量化、为什么能省显存、实际效果（内存和速度的数字）"就够。

### Q6: 每个项目需要写单元测试吗？

Step 1 不需要（benchmark 脚本即测试）。Step 2C 至少写 `/health` 和一个流式接口 smoke test。Step 3 可以写检索评估脚本。Step 4 是概念笔记，不需要测试。

### Q7: 项目是每个独立仓库还是 monorepo？

推荐**每个 Step 独立仓库**，方便面试时分别展示。每个仓库 README 自包含。README 底部加"前置项目"链接形成项目链。

### Q8: SGLang 需要实操吗？

不需要。Step 4 的概念对比已经足够。3/6 JD 提到 SGLang，但展示的是"了解生态、知道差异"而不是每个都要写代码。注意：国内有一种实践是 vLLM+SGLang 协同：vLLM 负责服务治理与调度，SGLang 处理特定加速场景（如 JSON Schema 强制输出）。这是架构选项，不是「不协同就落后」。

### Q9: 每个项目的 README 应该写什么？

统一模板：
1. **项目目的**（一句话）
2. **架构图**（ASCII 或 mermaid）
3. **快速开始**（3 步跑起来：安装→启动→验证）
4. **核心结果**（表格/图表）
5. **踩坑记录**（至少 2-3 个实际遇到的问题和解决方案，这是面试时最有话聊的部分。至少含一个"认知型"的坑，即因为理解错了导致行为异常，而不只是环境问题）
6. **前置项目**（链接到上一步的仓库）

### Q10: 为什么不学 Ollama？

Ollama stars 很高，主打一键体验、封装层级较高。本计划 Step 4 选择 vLLM，是为了理解 PagedAttention、Continuous Batching 等机制。本地快速实验可以用 Ollama；学推理架构从 vLLM 入手更合适。
---

## 附录：执行检查清单

### Step 0 完成标准
- [ ] 完成当前 `hf-llm-benchmark` 脚本盘点：每个脚本一句话说明输入、输出、解决的问题
- [ ] 能不看 AI 解释 `tokenizer.apply_chat_template()` 做了什么
- [ ] 能说清 `input_ids` 和 `attention_mask` 的含义
- [ ] 能从空文件写出 30-50 行 `scratch/minimal_infer.py`
- [ ] `notes/ownership.md` 完成，不少于 500 字

### Step 1 完成标准
- [ ] `benchmark.py` 可运行，输出 avg/P50/P99 + tokens/s
- [ ] 至少对比了 temperature/top_p/max_tokens 4 组参数组合
- [ ] 量化实验有 FP32 vs INT8 的具体数字
- [ ] 能说出 Qwen3 和 Llama tokenizer 的一个具体差异
- [ ] 能用 5 分钟口述 prompt → token → generate → decode → benchmark 的完整链路
- [ ] GitHub 公开仓库 + README（含踩坑记录）

### Step 2A-2C 完成标准（P0）
- [ ] `/health` 返回模型状态
- [ ] `/v1/chat/completions` 支持 SSE 流式输出
- [ ] 日志每条有 request_id
- [ ] `/metrics` 有 TTFT/TPOT 指标
- [ ] `docker-compose up` 一键启动
- [ ] README 含快速开始、curl 示例、常见问题和至少 3 个踩坑记录
- [ ] 能口述"请求进入 → 模型生成 → streamer 输出 → 指标记录 → SSE 返回"完整链路

### Step 2D 完成标准（P1，三选一）
- [ ] K8s：`kubectl apply -f k8s/` 部署成功
- [ ] CI：GitHub Actions 能构建镜像并跑 smoke test
- [ ] 压测：`python bench.py` 输出并发下 TTFT/TPOT 对比

### Step 3 完成标准（P1 推荐）
- [ ] 检索 recall@3 可量化
- [ ] 生成答案带引用来源
- [ ] 生成 pipeline 使用本地模型（非外部 API）
- [ ] 10 个测试问题的评估报告
- [ ] `/ask` 端点 FastAPI 服务化（复用 Step 2 经验）

### Step 4 完成标准（P2 加分项，完成前 6 项即达标）

**最低交付（6-10h）：**
- [ ] `vllm serve` 成功启动并提供 OpenAI 兼容 API
- [ ] 能画出 LLMEngine → Scheduler → Worker 调用链
- [ ] PagedAttention 概念笔记：用自己的话解释 Block Table 和 Copy-on-Write
- [ ] Scheduler 概念笔记：能描述 Continuous Batching 的执行流程
- [ ] KV Cache 手算过程与 config 参数一致
- [ ] 能说出 vLLM 和 SGLang 的一个架构差异

**有余力再做：**
- [ ] 完整的源码精读笔记（block_manager.py + scheduler.py，仅有余力再做）
- [ ] 理论推演文档（7B + 100 并发场景分析）
- [ ] GPU benchmark：HF vs vLLM 在不同并发下的 TTFT/TPOT/吞吐对比

---

## 版本历史

| 版本 | 日期 | 主要变更 |
|------|------|---------|
| v1 | 2026-06-17 | 初版：ONNX + ResNet → vLLM 路径 |
| v2 | 2026-06-17 | 切换到 HF Transformers + LLM 模型 |
| v3 | 2026-06-17 | 双模型对比 + 量化实验 + 国内镜像 + 理论推演 |
| v4 | 2026-06-17 | **市场校准版**：① 目标岗从"推理部署"调整为主攻"AI 应用开发"② Step 2 新增 K8s + CI/CD 实操（+6h）③ Step 3 从 P0 降为 P1 加分项 ④ Step 4 从可选升级为 P0 推荐 ⑤ 优先级重新排序：1-2-4 为最小可行证据链 ⑥ FAQ + 搜索词 + 叙事模板同步更新 |
| v5 | 2026-06-18 | **产品链路版**：在总览补充 AI 产品全链路、同类方向热度证据、Step 与模块映射；将 2.4 改为训练路线选择结论 |
| v6 | 2026-06-18 | **步骤重编号版**：RAG 调整为 Step 3、vLLM 调整为 Step 4，章节顺序与执行顺序统一为 1→2→3→4 |
| v7 | 2026-06-18 | **执行者视角 + 市场校准版**：① 新增 1.9 节"C/C++ 背景过渡指南"+ 每个 Step 的 C++ 加分讨论 ② 目标岗位从单一锁定改为多方向映射（第一/二/三梯队）③ Step 3 新增 LangChain 等价实现对比任务（+2h）④ 修正时间估算（区分编码时间 vs wall clock）⑤ Step 3 幻觉率替换为可操作评估指标 ⑥ 强化 TextIteratorStreamer 死锁解释（生产者-消费者模型）⑦ 量化备选方案细化 |
| v8 | 2026-06-28 | **Ownership Recovery 版**：① 新增 Step 0，先读懂/重写/验证已有 AI 生成代码 ② 将路线从 4 个项目大计划收缩为 Step 0-1-2 最小入场券 ③ Step 1 改为小闭环执行方式，先最小推理、tokenizer 显微镜、最小 benchmark，再扩展双模型和量化 ④ 明确 AI 辅助边界：每个脚本必须配 `notes/ownership.md` ⑤ 增加停止条件：若 7 天无法完成最小重写，降低 AI Infra 优先级 |
| v9 | 2026-06-28 | **C 型交付版**：① 标题和定位从 AI Infra 改为 AI 工程落地 ② Step 2 拆成 2A 服务 MVP、2B 可观测性、2C 交付闭环、2D 工程加分 ③ 每个核心阶段增加问题单驱动 ④ RAG 调整为 Step 2C 后优先加分项 ⑤ vLLM 降为 P2 概念学习，不再强制源码精读/GPU benchmark |
| v9.2 | 2026-07-16 | **第二波**：读者分叉（求职/产品）+ 1 周可演示；Step 2B 对齐 engineering §3.4；Step 3.6 强制故障注入与期望管理 |
| v9.3 | 2026-07-16 | **复审修正**：修正 Qwen 模型 ID、TextIteratorStreamer、`asyncio.gather`、traceback 与 vLLM CPU 安装说明；补全 SSE/tool_call 协议边界和分段观测；减少第三人称画像与绝对化表述 |
