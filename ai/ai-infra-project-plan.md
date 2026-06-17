# AI Infra 项目实战计划

> 生成日期：2026-06-17 | 状态：执行中 | 最后更新：2026-06-17（v4 市场校准版）

---

## 一、总览

### 1.1 最终目标

3-4 个月内完成 3-4 个有量化产出的项目，简历上形成完整叙事线：**「HF Transformers LLM 推理基础 → FastAPI LLM 流式推理服务（含 K8s/CI/CD）→ RAG 检索增强生成 → vLLM 架构学习」**。主攻 **AI 应用开发工程师 / AI 平台后端开发** 岗位（岗位数量多、匹配度高），辅攻大模型平台工程师和推理部署工程师。

> **定位说明（v4）**：本计划产出的技能组合最匹配「AI 应用开发工程师」和「AI 平台后端开发」（深圳 20-35K），而非「推理优化工程师」（需 CUDA/C++）。这不是降级，是瞄准了一个岗位更多、学历门槛更低的细分市场。初创公司对此类技能的需求尤其旺盛。

### 1.2 约束条件

| 约束 | 说明 |
|------|------|
| 无本地 GPU，可租用（预算约 30-40 元） | 核心路径 CPU 可完成；Step 3 可选租 GPU 2-4h 做实验验证（~10-20 元），让简历数据更有说服力 |
| 业余时间 | 工作日 1-2h，周末 3-4h |
| 目标工种 | 推理部署工程师 / ML Platform 工程师（不涉及训练、不写 CUDA kernel） |
| 技术栈偏好 | Python 为主，匹配岗位实际需求 |
| 网络环境 | 国内访问 HF Hub 可能受限，需配置镜像 |

### 1.3 推荐路径

```
P0 Step 1: HF Transformers LLM 基础      ← 现在开始，CPU，2-3 周
P0 Step 2: FastAPI LLM 流式推理服务      ← Step 1 完成后，CPU，3-4 周（含 K8s + CI/CD）
P0 Step 4: RAG 问答系统                  ← Step 2 完成后，CPU，2-3 周
P1 Step 3: vLLM 架构学习与源码分析       ← 加分项，视进度决定深度，CPU，1-3 周
```

> **P0/P1 说明**：Step 1-2-4 构成最小可行证据链，做完就能投 AI 应用开发岗。Step 3 是加分项——能讲清 PagedAttention 和 Continuous Batching 即可，源码笔记写到够面试用就停，不追求代码级精度。GPU benchmark 真正变成可选（不做也不影响投目标岗位）。

### 1.4 时间估算

| 步骤 | 优先级 | 预计耗时 | 日历时间 | GPU | 核心技能 |
|------|:---:|---------|---------|:---:|------|
| Step 1 | P0 | 12-18h | 2-3 周 | ❌ | HF Transformers, tokenizer, 量化, 多模型, benchmark |
| Step 2 | P0 | 20-26h | 3-4 周 | ❌ | FastAPI, SSE streaming, OpenAI API, Docker, K8s, CI/CD, Prometheus |
| Step 4 | P0 | 10-15h | 2-3 周 | ❌ | Embedding, ChromaDB, RAG pipeline |
| Step 3 | P1 | 8-16h（GPU 可选 +4h） | 1-3 周 | ⚠️ CPU 为主，GPU 可选 | vLLM 源码, PagedAttention, Scheduler, KV Cache |
| **合计（P0）** | | **42-59h** | **约 2.5-3 个月** | | |
| **合计（含 P1）** | | **50-75h** | **约 3-4 个月** | | |

---

## 二、为什么这样设计——基于真实 JD 的证据

### 2.1 JD 来源

以下分析基于 2026 年 6 家公司的真实招聘要求（数据采集于 2026-06-17）：

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

**本计划目标：推理部署 / ML Platform（岗位多、不写 CUDA）。** CUDA/C++ 明确跳过。

### 2.3 JD 技能要求 vs 本计划覆盖

| JD 核心要求 | 出现频率 | 对应 Step | 说明 |
|------|:---:|:---:|------|
| Python | 6/6 | 全流程 | 主力语言 |
| HuggingFace Transformers / PyTorch | 6/6 | Step 1 | 基础技能，JD 必选项 |
| vLLM | 5/6 | Step 3 | 高频要求，侧重架构理解 |
| SGLang | 3/6 | Step 3（概念对比） | 不实操，概念层了解与 vLLM 的差异 |
| Docker | 6/6 | Step 2 | 服务化标配 |
| FastAPI / HTTP 服务化 | 隐含要求 | Step 2 | 推理部署核心日常 |
| OpenAI 兼容 API 协议 | 行业标准 | Step 2 | vLLM/SGLang/各类 LLM 网关均采用此格式 |
| SSE 流式输出 | 4/6 | Step 2 | 腾讯 JD 明确要求 |
| Prometheus / 可观测性 | 4/6 | Step 2 | 腾讯、小鹏 JD 高频 |
| KV Cache 原理 | 高频 | Step 3 | 手算验证，不止于概念 |
| PagedAttention / Continuous Batching | 高频 | Step 3 | 百度 JD：「了解架构原理」 |
| TTFT / TPOT 指标 | 高频 | Step 2-3 | LLM 推理标准延迟指标 |
| 量化 (AWQ/GPTQ) | 4/6 | Step 1（实操 8-bit） | 概念 + 代码层面都接触 |
| RAG | 中频 | Step 4 | 企业落地常见场景 |
| 多模型经验 | 隐含要求 | Step 1（2 个模型） | 展示不只会上手一个模型 |
| Kubernetes | 4/6 | Step 2（Minikube 实操） | 原标注未覆盖，v4 加入 Step 2 末尾 |
| CUDA | 4/6 | ❌ 跳过 | 推理优化专属，不在目标范围 |

### 2.4 与原 P1 计划的差异

| | 原计划（v1） | v3 优化版 | 改动原因 |
|---|------|------|------|
| Step 1 模型 | ResNet-18（CV） | Qwen3-0.6B + **Llama-3.2-1B 对比** | JD 要 LLM + 展示多模型经验 |
| Step 1 框架 | ONNX Runtime | HuggingFace Transformers | 6/6 JD 要求 PyTorch/HF |
| Step 1 量化 | ❌ 无 | **✅ 8-bit 实操对比** | 4/6 JD 要求量化概念 |
| Step 2 模型 | ResNet-18 | Qwen3-0.6B | JD 要 LLM 非 CV |
| Step 2 输出 | JSON 一次性 | **SSE 流式 + OpenAI 兼容 API** | 腾讯 JD 要求 + 行业标准协议 |
| Step 2 压测 | Locust | **asyncio + httpx（更可控）** | Locust 对 SSE 支持不足 |
| Step 3 定位 | vLLM 调参 | **vLLM 架构学习 + 源码分析** | CPU+小模型下实验对比无意义 |
| Step 3 GPU | 必须租 T4（~32元） | **CPU 为主 + GPU 可选强化（~10-20 元）** | 核心学习 CPU 完成；租 2-4h GPU 跑实验验证理论 |
| 延迟指标 | QPS/P50/P99 | **TTFT + TPOT + tokens/s** | LLM 推理标准指标 |
| 国内下载 | 未提及 | **HF Mirror / ModelScope 镜像** | 执行可行性 |
| SGLang | 未提及 | **Step 3 概念对比** | 3/6 JD 提及 |
| 总 GPU 依赖 | Step 3 需租 32 元 | **0 元（核心） + 可选 10-20 元（强化）** | 满足低预算约束 |

---

## 三、Step 1：HuggingFace Transformers LLM 推理基础

### 3.1 目标

用 HuggingFace Transformers 加载 Qwen3-0.6B 和 Llama-3.2-1B 两个大语言模型，深入理解分词器、生成流程、参数影响、量化效果，编写 benchmark 脚本测量推理性能。

### 3.2 为什么这是第一步

- 6/6 JD 要求 PyTorch / HuggingFace Transformers，这是 AI Infra 的入门必备
- 所有 LLM 推理框架（vLLM、SGLang）都在此基础上构建
- 使用 2 个不同模型，展示处理多模型的通用能力
- 全程 CPU 可完成

### 3.3 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| 主模型 | Qwen3-0.6B-Instruct（~1.2GB） | 2026 主流开源，Apache 2.0 协议，CPU 友好 |
| 对比模型 | Llama-3.2-1B-Instruct（~2.5GB） | Meta 开源，不同 tokenizer 体系（BPE vs 中文优化），展示多样性 |
| 框架 | HuggingFace Transformers | 行业标准，JD 必选项 |
| 精度 | FP32 + 8-bit（BitsAndBytesConfig） | CPU 推理默认 + 量化入门 |
| 模型下载 | HF Mirror (`hf-mirror.com`) 或 ModelScope | 国内网络必需 |
| 语言 | Python 3.10+ | 岗位要求 |

### 3.4 环境准备：国内下载指南（重要）

HuggingFace Hub 在国内直接下载可能极慢或失败。两种方案：

**方案 A：使用 HF Mirror（推荐）**
```bash
export HF_ENDPOINT=https://hf-mirror.com
# 此后 huggingface_hub 的所有下载自动走镜像
```

**方案 B：使用 ModelScope**
```bash
pip install modelscope
# modelscope download --model Qwen/Qwen3-0.6B-Instruct
```

### 3.5 可参考项目

| 参考 | 类型 | 亮点 | 链接 |
|------|------|------|------|
| HF 官方 text-generation 教程 | 官方文档 | pipeline() → AutoModel → generate() 完整流程 | huggingface.co/docs/transformers |
| CSDN "5分钟快速上手LLM" | 中文博客 | 2026.6 新鲜，含量化/chatbot | blog.csdn.net/m0_65555479 |
| huggingface-llm-examples | GitHub | 多模型 benchmark，自动生成 Excel 报告 | github.com/ogunerkutay/huggingface-llm-examples |
| Gemma-3-1B Pipeline | 博客 | 2026.4，含 mini-benchmark 和延迟测量 | marktechpost.com |

### 3.6 详细任务清单

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 1.1 | 环境准备 | 创建 `hf-llm-benchmark/`，git init，虚拟环境，配置 HF Mirror 环境变量，安装 `torch transformers accelerate bitsandbytes` | 0.5h |
| 1.2 | 模型下载与文件结构 | 下载 Qwen3-0.6B-Instruct 和 Llama-3.2-1B-Instruct → 理解模型文件结构：`config.json`（hidden_size/num_layers/vocab_size）、`tokenizer.json`（词表映射）、`model.safetensors`（权重格式）、`*.safetensors.index.json`（分片索引）。对比两个模型的 config 差异 | 1.5h |
| 1.3 | 分词器入门 | Qwen3 tokenizer → tokenize 中英文示例 → 理解 input_ids / attention_mask / 特殊 token（`<|im_start|>`、`<|im_end|>`）→ **对比 Llama tokenizer**（BPE 体系差异、同一句中文的 token 数差异） | 2h |
| 1.4 | 单次推理跑通 | 加载 Qwen3 → `tokenizer.apply_chat_template()` → `model.generate()` → decode，打通完整推理流程。再切换到 Llama 跑同一条 prompt | 1h |
| 1.5 | 生成参数实验 | Qwen3 上系统对比：① temperature(0.1/0.7/1.5) ② top_p(0.5/0.9/1.0) ③ max_new_tokens(32/128/512) ④ do_sample(True/False)。记录每组输出质量和 tokens/s | 3h |
| 1.6 | 8-bit 量化实验 | 用 `BitsAndBytesConfig(load_in_8bit=True)` 加载 Qwen3-0.6B → 对比 FP32 vs INT8：模型加载内存、推理速度、输出质量差异 → 理解量化的本质：用精度换显存/内存 | 1.5h |
| 1.7 | Benchmark 脚本 | 准备 10-15 条不同长度 prompt（短问答/中篇总结/长文分析）→ 预热 10 次 → 每条 10 次取统计 → 两个模型分别跑 → 输出 avg/P50/P99 延迟 + tokens/s + 生成 token 数 | 3h |
| 1.8 | 结果整理 + 提交 | README（项目目的+环境+镜像配置+核心结果）+ 两模型对比表 + 量化对比表 | 1h |

### 3.7 产出物

```
hf-llm-benchmark/
├── README.md                      # 项目说明 + benchmark 结果
├── requirements.txt
├── benchmark.py                   # 核心 benchmark（支持 --model 参数）
├── explore_tokenizer.py           # 分词器实验（两模型对比）
├── explore_params.py              # 生成参数实验
├── explore_quantization.py        # 8-bit 量化对比
└── results/
    └── report.md
```

### 3.8 验收标准

- [ ] `python benchmark.py --model qwen` 一键运行
- [ ] 理解 tokenizer 的 chat_template 和特殊 token
- [ ] 能解释 temperature / top_p 的影响
- [ ] 能说出 Qwen3 和 Llama tokenizer 的一个具体差异（如中文切分方式）
- [ ] 量化实验：INT8 内存降幅有具体数字
- [ ] 报告包含两个模型的 avg/P50/P99 延迟和 tokens/s

### 3.9 面试一句话

> 「我用 HuggingFace Transformers 加载了 Qwen3-0.6B 和 Llama-3.2-1B 跑 benchmark，系统对比了生成参数对延迟和质量的影响，还做了 FP32 vs INT8 量化的内存/速度对比。过程中理解了 tokenizer 机制、chat_template 作用和自回归生成流程。」

### 3.10 对标的 JD 技能

Python ✅ | PyTorch/HF ✅ | Tokenizer ✅ | 多模型经验 ✅ | 量化(8-bit) ✅ | Benchmark ✅

### 3.11 执行提示

**CPU benchmark 太慢怎么办。** Qwen3-0.6B 在普通笔记本 CPU 上约 2-5 tokens/s，生成 512 token 需要 1-3 分钟。按任务 1.7 的 15 prompt × 10 次 = 150 次，一次 2 分钟 = 5 小时 wall clock。**建议首次运行时降低采样**：10 prompt × 5 次 = 50 次，先出数据。后续可以挂一晚上跑完整版。

**tokenizer 对比抓住一个点。** Qwen3 和 Llama tokenizer 对中文的处理差异是最容易观察的——同一句中文，Llama 的 token 数通常是 Qwen3 的 1.5-2 倍（因为 Llama 的 BPE 词表以英文为主）。把这个对比数字写进报告，面试时一句话就能展示「我理解 tokenizer 对推理成本和速度的影响」。

**量化实验注意。** `BitsAndBytesConfig(load_in_8bit=True)` 在纯 CPU 环境下可能需要额外配置（默认依赖 CUDA）。如果遇到报错，改为对比 `torch_dtype=torch.float32` vs `torch_dtype=torch.float16` 来展示精度对内存的影响，概念层面等价。

---

## 四、Step 2：FastAPI LLM 流式推理服务

### 4.1 目标

基于 Step 1 的模型和推理代码，用 FastAPI 构建 LLM 流式推理 HTTP 服务：SSE 逐 token 推送、OpenAI 兼容 API 格式、TTFT/TPOT 打点、Prometheus 指标、Docker 部署，并发压测。

### 4.2 为什么是第二步

- 腾讯 JD：「流式推理」「可观测性体系」
- 字节 JD：「端到端流程」
- OpenAI 兼容 API 是行业事实标准——vLLM、SGLang、各类 LLM 网关走的都是 `/v1/chat/completions`
- 在 Step 1 模型基础上叠加服务化，技能连续积累

### 4.3 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| Web 框架 | FastAPI | 异步原生支持 SSE StreamingResponse |
| 模型加载 | HuggingFace Transformers | 沿用 Step 1 |
| 流式生成 | `TextIteratorStreamer`（transformers 内置） | HF generate() 是阻塞的，Streamer 通过后台线程逐 token 产出 |
| 流式协议 | SSE (Server-Sent Events) | LLM 服务标准输出方式 |
| API 格式 | 自定 `/generate` + **OpenAI 兼容 `/v1/chat/completions`** | 行业标准协议 |
| 指标 | prometheus_client | JD 高频 |
| 压测 | Python asyncio + httpx（替代 Locust） | Locust 对 SSE 支持不足，asyncio 更可控 |
| 容器化 | Docker + docker-compose | 6/6 JD 要求 |

> **为什么不用 Locust？** Locust 没有原生 SSE 支持，需要大量自定义代码才能解析 SSE event 并计算 TTFT。直接用 asyncio + httpx 写并发脚本更可控、代码更少、逻辑更透明。

### 4.4 可参考项目

| 参考 | 类型 | 亮点 |
|------|------|------|
| FastAPI SSE 教程 | 官方文档 | StreamingResponse 实现 |
| vLLM API Server 源码 | 源码 | `/v1/chat/completions` 请求/响应格式 |
| OpenAI API 文档 | 规范 | chat/completions 的标准请求体和响应体 |

### 4.5 详细任务清单

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 2.1 | 项目初始化 | 创建 `llm-inference-service/`，git init，虚拟环境 | 0.5h |
| 2.2 | 模型层：流式生成器 | `model.py`：封装 HF 模型加载 + `generate_stream()` 生成器函数。核心实现：用 `TextIteratorStreamer` 开后台线程跑 `model.generate()`，主线程 yield 每个 token 的 text。记录 TTFT（第一个 token 的 wall time）和每个后续 token 的间隔 | 3h |
| 2.3 | 单文件原型 | `main.py`：启动加载模型到全局 → `/health` → `/generate`（SSE 流式，自定格式）→ 验证 `curl` 可收到逐 token 推送 | 2h |
| 2.4 | OpenAI 兼容端点 | 新增 `/v1/chat/completions`：接收 `{"model":"qwen3","messages":[...],"stream":true}` → 返回 SSE 格式 `data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"content":"..."}}]}` → 支持 `stream: false` 的非流式模式。直接对标 vLLM/SGLang 的 API 格式 | 2h |
| 2.5 | Pydantic 模型 + 错误处理 | ChatCompletionRequest / ChatCompletionResponse / GenerateRequest / GenerateResponse，处理空 messages、超长 prompt（超过 max_model_len 拒绝并返回明确错误） | 1.5h |
| 2.6 | 延迟指标打点 | TTFT = 请求到达 → 首 token；TPOT = 后续每个 token 间隔（记录 avg/P99）。每个请求结束后输出 JSON 日志：`{request_id, ttft_ms, tpot_avg_ms, tpot_p99_ms, total_tokens, total_time_ms}` | 1.5h |
| 2.7 | 中间件 | ① request_id（uuid4，注入每个请求并在响应头返回）② 结构化日志（JSON 格式）③ Prometheus `/metrics`：请求总数、TTFT histogram、TPOT histogram、tokens/s gauge | 2h |
| 2.8 | 配置管理 | pydantic-settings：MODEL_PATH / HOST / PORT / MAX_TOKENS_LIMIT / LOG_LEVEL，环境变量覆盖 | 0.5h |
| 2.9 | Docker 化 | Dockerfile（python:3.11-slim）+ docker-compose.yml，挂载模型文件。**注意**：不使用 `--reload`，避免多 worker 各自加载一份模型 | 1.5h |
| 2.10 | asyncio 并发压测 | 写 `bench.py`：asyncio + httpx 并发发送 10/50/100 个流式请求 → 解析 SSE → 统计每个请求的 TTFT/TPOT/total_time → 汇总输出 avg/P50/P99。对比 1 worker vs 4 workers | 3h |
| 2.11 | K8s 部署（Minikube） | `minikube start` → 写 Deployment（1-2 replicas）+ Service（ClusterIP）+ ConfigMap（模型路径/参数）→ `kubectl apply` 部署 → 验证 `kubectl port-forward` 后可调 `/v1/chat/completions`。**核心产出**：对比 Docker Compose vs K8s 部署差异的笔记（服务发现、配置管理、扩缩容方式的差异） | 3h |
| 2.12 | CI/CD（GitHub Actions） | 写 `.github/workflows/ci.yml`：push 触发 → build Docker 镜像 → 启动容器 → 跑 `test_chat_completions.py`（验证 `/health` + 流式 + 非流式）→ 输出测试报告。可选：`docker build` 成功后推送到 Docker Hub 或 ghcr.io | 2h |
| 2.13 | 结果整理 + 提交 | README 含架构图、压测结果表格、OpenAI API 调用示例（Python + curl）、K8s 部署说明、CI badge | 1h |

### 4.6 产出物

```
llm-inference-service/
├── README.md
├── Dockerfile
├── docker-compose.yml
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI + /health + /generate + /v1/chat/completions
│   ├── config.py          # pydantic-settings
│   ├── model.py           # HF 模型 + TextIteratorStreamer 流式生成
│   ├── schemas.py         # OpenAI 兼容 + 自定义 Pydantic 模型
│   └── middleware.py      # request_id + 日志 + Prometheus
├── k8s/                    # 【v4 新增】
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── .github/workflows/      # 【v4 新增】
│   └── ci.yml
├── tests/
│   ├── test_health.py
│   └── test_chat_completions.py
├── bench.py               # asyncio + httpx 并发压测
└── results/
    └── load_test_report.md
```

### 4.7 验收标准

- [ ] `docker-compose up` 一键启动
- [ ] `curl localhost:8000/health` 返回模型状态
- [ ] `curl -X POST localhost:8000/v1/chat/completions -d '{"model":"qwen3","messages":[{"role":"user","content":"你好"}],"stream":true}'` SSE 流式输出
- [ ] 同一端点 `"stream":false` 返回完整 JSON
- [ ] `/metrics` 暴露 TTFT/TPOT histogram
- [ ] `python bench.py` 压测出 3 组并发的 TTFT/TPOT 对比
- [ ] 日志每条有 request_id
- [ ] **【v4 新增】** `kubectl apply -f k8s/` 部署成功，`kubectl port-forward` 后可调 API
- [ ] **【v4 新增】** GitHub Actions CI 绿色，包含构建 + 测试 + 报告

### 4.8 面试一句话

> 「我用 FastAPI 构建了 LLM 流式推理服务：实现了 OpenAI 兼容的 `/v1/chat/completions` 端点、SSE 逐 token 推送、TTFT/TPOT 延迟打点、Prometheus 指标和 Docker 部署。用 asyncio 并发压测了不同 worker 数下的延迟分布。」

### 4.9 对标的 JD 技能

FastAPI ✅ | SSE 流式 ✅ | OpenAI 兼容 API ✅ | TTFT/TPOT ✅ | Docker ✅ | K8s ✅ | CI/CD ✅ | Prometheus ✅ | 压测 ✅

### 4.10 执行提示

**TextIteratorStreamer 的正确用法（最容易卡住的地方）。** `TextIteratorStreamer` 的工作模式是：创建一个 streamer 对象 → 传给 `model.generate(streamer=streamer)` → **在另一个线程**调用 generate → 主线程循环 `for text in streamer` 读 token。不是同步调用 generate 再读 streamer，那样会死锁。最可靠的参考不是博客，是 transformers 仓库的测试文件 `tests/test_streamers.py`，在 GitHub 上搜索 "TextIteratorStreamer test" 即可找到官方用法。

**asyncio 压测脚本的常见坑。** 流式请求的超时需要区分"连接超时"和"读取超时"。第一次 token（TTFT）可能在 2-3 秒后才来，所以读取超时要设大（30s+），但连接超时保持默认（5s）。另外 `asyncio.gather` 跑 100 个并发任务时，异常不会自动抛出——需要用 `return_exceptions=True` 然后手动检查每个结果。

**为简历加分：在 README 加一段"多模型版本管理"讨论。** 不需要写代码。在 README 末尾加一段 200 字的讨论：「如果线上同时服务 v1 和 v2 两个版本的模型，可以怎么做：模型文件按版本号目录存储 → FastAPI 根据请求中的 `model` 参数路由到对应实例 → Prometheus 指标按 `model_version` 标签区分」。4/6 JD 隐含对模型版本管理的期望，这段讨论证明你思考过生产环境的问题。

**Docker 多 worker 的内存陷阱。** Gunicorn 多 worker 模式下，每个 worker 各自加载一份模型到内存。0.6B FP32 约 2.4GB，4 workers ≈ 10GB。`docker-compose.yml` 如果限制内存不足会 OOM。建议：本地测试用 1-2 workers，压测对比时在文档中说明"生产环境需根据模型大小调整 worker 数"。

---

## 五、Step 3：vLLM 架构学习与源码分析（P1 加分项）

> **定位（v4）**：本 Step 从「核心路径」降为「加分项」。AI 应用开发岗对 vLLM 的要求是「会用 + 理解原理」，不是「读过源码」。核心交付物是**能用自己的话讲清 PagedAttention 和 Continuous Batching**。源码笔记写到够面试用就停，不追求代码级精度。GPU benchmark 真正变成可选——不做也不影响投目标岗位。优先完成 Step 1-2-4，有剩余精力再做 Step 3。

### 5.1 目标

**核心路径（CPU，12-16h）：** 跑通 vLLM CPU 模式，精读核心模块源码（PagedAttention 内存管理、Scheduler 调度逻辑），手算 KV Cache，理论推演 GPU vs CPU 场景差异。产出架构分析文档。

**强化路径（租 GPU 2-4h，约 10-20 元，可选）：** 在 T4 上跑通 vLLM GPU 模式，做 HF vs vLLM 的真实 GPU benchmark 对比，用实验数据验证理论推演的结论。产出：一份有真实 GPU 数据的对比报告。

**两步互补：** 核心路径保证深度理解，强化路径让简历上有真实的 GPU benchmark 数据。面试时既能讲源码，又能出示性能对比数字。

### 5.2 最低交付标准（做什么就够）

**你不需要完成全部 5.5 的任务。** 最低交付足以应对 AI 应用开发岗面试：

1. 用 `vllm serve` 启动模型，调通 `/v1/chat/completions` ✅
2. 阅读官方 Architecture Overview 文档，能画出 LLMEngine → Scheduler → Worker 调用链 ✅
3. 用自己的话讲清 PagedAttention（Block Table + 逻辑→物理映射 + Copy-on-Write） ✅
4. 用自己的话讲清 Continuous Batching（请求队列 → 动态打包 → 为什么比 static batching 好） ✅
5. 手算一次 KV Cache（用 Qwen3-0.6B 的 config 参数） ✅
6. 能说出 vLLM 和 SGLang 的一个核心差异 ✅

以上约 8-10h 可完成。剩下的源码精读和 GPU benchmark 是「有余力再做」

### 5.3 技术选型

| 组件 | 选择 | 原因 |
|------|------|------|
| 推理框架 | vLLM（CPU 跑通 + GPU 实验） | CPU 完成核心学习，GPU 验证理论 |
| 模型 | Qwen3-0.6B-Instruct（CPU）/ Llama-3.2-1B 或 Qwen3-1.7B（GPU） | CPU 用小模型快速跑通；GPU 上有显存余量可用更大模型 |
| 源码重点 | `scheduler.py`、`block_manager.py` | PagedAttention 和调度逻辑的核心 |
| 概念对比 | SGLang | 3/6 JD 提及，了解核心差异 |
| GPU 租用 | AutoDL T4 16GB（~2元/h） | 2-4h 足够完成实验验证 |

### 5.4 可参考资源

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

### 5.5 详细任务清单

#### 核心路径（CPU，12-16h）

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 3.1 | vLLM CPU 环境搭建 | `pip install vllm`，验证 CPU 后端可用，启动 `vllm serve Qwen/Qwen3-0.6B-Instruct --host 0.0.0.0 --port 8000`，curl 调通 `/v1/chat/completions` | 1.5h |
| 3.2 | 离线推理跑通 | Python 调 `vllm.LLM` 类 → `llm.generate([...])` 验证输出，对比与 Step 1 HF 的输出差异（同一 prompt、temperature=0） | 1h |
| 3.3 | 阅读官方架构文档 | 精读 [vLLM Architecture Overview](https://docs.vllm.ai/en/latest/design/arch_overview.html)，理解：LLMEngine → Scheduler → Worker → ModelRunner 的调用链；Centralized Controller 设计；KV Cache Manager 的角色 | 1.5h |
| 3.4 | **源码精读 1：PagedAttention** | 阅读 `block_manager.py`：Block Table 数据结构、逻辑块→物理块映射、块的分配与回收、Copy-on-Write。输出源码笔记 | 3h |
| 3.5 | **源码精读 2：Scheduler** | 阅读 `scheduler.py`：请求队列管理、抢占策略（Preemption）、Continuous Batching 实现。输出源码笔记 | 3h |
| 3.6 | KV Cache 理论计算 | 取 Qwen3-0.6B 的 config，用 [ai_from_zero.md:451](ai_from_zero.md#L451) 公式手动计算 512 token 输入的 KV Cache 大小 | 1h |
| 3.7 | SGLang 概念对比 | 阅读 SGLang 文档，了解 RadixAttention 与 PagedAttention 的核心差异 | 1h |
| 3.8 | 理论推演文档 | 假设场景（7B、100 并发、2000in+500out、A10 GPU），推演 HF 瓶颈 → vLLM 解决方案 → CPU 局限 | 2h |
| 3.9 | 核心路径结果整理 | 架构分析文档：架构全景图 + PagedAttention 笔记 + Scheduler 笔记 + KV Cache 手算 + SGLang 对比 + 理论推演 | 2h |

#### 强化路径（租 GPU 2-4h，约 10-20 元，可选）

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| G.1 | 租 GPU + 环境搭建 | AutoDL 租 T4 16GB，安装 vLLM，下载模型（建议 Llama-3.2-1B 或 Qwen3-1.7B，利用 GPU 显存余量） | 0.5h |
| G.2 | GPU benchmark 实验 | 用与 Step 1 相同的 benchmark 脚本 + 同一批 prompt，**在 GPU 上跑 HF 和 vLLM 两组实验**。记录：TTFT、TPOT、tokens/s、GPU 显存占用、GPU 利用率。**关键：对比不同并发（1/16/32/64）下两框架的吞吐差异** | 2h |
| G.3 | 实验结果与理论推演对照 | 对比 G.2 的实验数据 vs 3.8 的理论推演结论：推演中预测的趋势是否在实验中验证？为什么有偏差？输出一份 1-2 页的「理论 vs 实验」对照笔记 | 1h |
| G.4 | GPU 结果整合到主文档 | 将 GPU 实验结果补充到架构分析文档，README 中突出展示核心基准数字 | 0.5h |

> **GPU 实验策略**：先在 CPU 上完成核心路径的源码学习和理论推演 → 准备好 benchmark 脚本 → 租 GPU 一次性跑完所有实验（2-4h 连续工作）→ 关机释放实例。不要边学边开着 GPU 烧钱。

### 5.6 产出物

```
vllm-architecture-study/
├── README.md                        # 架构分析报告入口 + GPU 实验结果（如有）
├── notes/
│   ├── 01_arch_overview.md          # 架构全景 + 调用链
│   ├── 02_paged_attention.md        # PagedAttention 源码笔记
│   ├── 03_scheduler.md              # Scheduler 源码笔记
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

### 5.7 验收标准

**核心路径（必达）：**
- [ ] `vllm serve` 成功启动并提供 OpenAI 兼容 API
- [ ] PagedAttention 源码笔记能用自己的话解释 Block Table 和 Copy-on-Write
- [ ] Scheduler 源码笔记能描述 Continuous Batching 的执行流程
- [ ] KV Cache 手算值与 config 推导一致
- [ ] 理论推演能说清楚：GPU 上 vLLM 为什么快、CPU 上为什么优势消失
- [ ] 能说出 vLLM 和 SGLang 的一个核心架构差异

**强化路径（可选）：**
- [ ] GPU benchmark 有 HF vs vLLM 在不同并发下的吞吐和延迟对比数据
- [ ] 「理论 vs 实验」对照笔记，分析了预测偏差的原因
- [ ] README 中突出展示 GPU 实验核心数字

### 5.8 面试一句话

> 「我系统学习过 vLLM 架构：读过 scheduler 和 block_manager 源码，能讲清楚 PagedAttention 和 Continuous Batching。还在 T4 上做了 HF vs vLLM 的 benchmark 对比，在不同并发下 vLLM 吞吐提升 X 倍，KV Cache 碎片率从 Y% 降到 Z%。」

### 5.9 对标的 JD 技能

vLLM ✅ | PagedAttention ✅ | Continuous Batching ✅ | KV Cache ✅ | SGLang(概念) ✅ | 源码级理解 ✅ | GPU benchmark ✅

### 5.10 执行提示

**源码阅读的正确姿势：不要逐行读。** vLLM 代码库约 20 万行，`scheduler.py` 自己就 ~1500 行。目标是**理解核心设计决策**，不是读懂每一行。具体方法：

1. **先看官方 Architecture Overview 文档**（约 10 页），建立全局视图
2. **Scheduler 入口**：打开 `scheduler.py`，搜索 `def _schedule(` ——这是 Continuous Batching 的核心入口方法。理解它的输入（请求队列）、核心循环（逐个取出请求、分配到 block）、输出（scheduled requests + 空闲 block 列表）
3. **Block Manager 入口**：打开 `block_manager.py`，搜索 `def allocate(` 和 `def free(` ——理解 Block Table 的两级映射（逻辑块→物理块）和 Copy-on-Write 的触发条件
4. **记三个关键设计决策**就够了：①为什么用 Block Table 两级映射（允许逻辑连续、物理分散）②Preemption 什么时候触发（新请求需要的 block 超过空闲数时，回收哪些旧请求的 block）③Continuous Batching 的打包逻辑（哪些请求可以放进同一个 batch）

**如果源码实在看不懂。** 搜索引擎搜索 "vLLM architecture explained" 或 "PagedAttention illustrated"，找带图的社区解读文章。理解了全局后再回来看源码对应位置。不要硬啃。

**理论推演加 2 个 2026 年热点段落（加分但不费时）。**

一、**MoE 模型推理**。在 3.8 的理论推演文档中加一段（200 字即可）：「如果是 MoE 模型（如 DeepSeek V4，1.6T 总参数但每次只激活 49B），推理部署额外面临：①专家路由——每个 token 需要决定走哪几个专家，路由计算本身有时间开销 ②专家负载不均——热门专家（如通用语言专家）被大量 token 选中，成为瓶颈 ③EP（专家并行）——不同专家分布在不同 GPU 上，token 需要跨 GPU 通信」。百度/华为 JD 均涉及 MoE。

二、**PD 分离（Prefill-Decode Disaggregation）**。在上述文档再加一段（150 字）：「PD 分离是 2026 年推理部署的热点架构——Prefill（计算密集）和 Decode（内存密集）拆分到不同的 GPU 实例上，各自独立扩缩容。好处是资源利用率更高（不会因为长 prompt 的 Prefill 占用 GPU 导致 Decode 排队），代价是 Prefill 产出的 KV Cache 需要跨网络传输给 Decode 实例」。百度 JD 明确提及 PD 分离。

> 以上两段不需要跑任何代码，纯概念讨论。写在理论推演文档中，面试被问到 MoE 或 PD 分离时，你已经比 90% 的候选人准备得更充分。

---

## 六、Step 4：RAG 问答系统（P0 推荐）

> **定位（v4）**：从「可选」升级为「推荐」。RAG 是当前 AI 应用开发岗的最高频场景——你搜到的深圳岗位（顺丰、腾讯、尹硕）几乎全部涉及。且 Step 4 复用了 Step 1 的模型和 Step 2 的服务化经验，边际成本低。

### 6.1 目标

构建端到端 RAG 问答系统：文档切片 → Embedding 向量化 → ChromaDB 检索 → LLM 生成带引用答案，含检索质量评估。LLM 生成部分直接复用 Step 1 的 Qwen3-0.6B 模型，不需要外部 API。

### 6.2 前置条件

- 完成 Step 2（服务化经验）
- 全部 CPU 可完成
- `pip install chromadb sentence-transformers`
- 复用 Step 1 下载的 Qwen3-0.6B 模型

### 6.3 详细任务清单

| # | 任务 | 内容 | 预计 |
|---|------|------|------|
| 4.1 | 准备知识库 | 选取文档集（建议 ai_from_zero.md 拆成 10-20 个章节），转纯文本 | 1h |
| 4.2 | 文档切片 | 按段落/标题分块，每块 200-500 token，50 token 重叠；输出 chunk 列表（文本+来源+位置） | 2h |
| 4.3 | Embedding + 向量存储 | 用 BGE-small 或 all-MiniLM-L6-v2 转向量 → 存入 ChromaDB，支持增删查 | 2h |
| 4.4 | 检索 pipeline | `retrieve(question)` → 向量化 → ChromaDB top-k 检索 → 返回 chunk + 相似度分数 | 1.5h |
| 4.5 | 生成 pipeline | 检索结果 + 用户问题拼 Prompt → 调**本地 Qwen3-0.6B**（复用 Step 1 模型）→ 返回带引用来源的答案 | 2h |
| 4.6 | 评估 | 准备 10 个测试问题（含标准答案来源）→ 评估 recall@3/5、答案引用准确率、幻觉率 | 2h |
| 4.7 | FastAPI 服务化 | 复用 Step 2 经验：`/ask` 端点 → 检索+生成 → JSON（含答案+引用+耗时） | 2h |
| 4.8 | 结果整理 + 提交 | README + 10 问评估结果 + 架构图 | 1h |

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
- [ ] 评估报告含 recall@3/5 和幻觉率
- [ ] 生成 pipeline 使用本地模型（非外部 API）

### 6.6 面试一句话

> 「我搭了端到端 RAG 系统：文档切片 → BGE-small Embedding → ChromaDB 检索 → 本地 Qwen3-0.6B 生成带引用答案，召回率 recall@3 达到 X%。」

---

## 七、简历叙事汇总

### 7.1 项目线

| 项目 | 优先级 | 一行描述 | 关键词 |
|------|:---:|---------|--------|
| HF LLM Benchmark | P0 | 用 HF Transformers 加载 Qwen3-0.6B/Llama-3.2-1B 双模型做 benchmark，含量化对比 | PyTorch, HF, Tokenizer, 8-bit Quantization, Multi-model |
| FastAPI LLM 流式服务 | P0 | 构建 OpenAI 兼容 LLM 推理服务：SSE 流式、TTFT/TPOT、K8s 部署、CI/CD、Prometheus、Docker、并发压测 | FastAPI, SSE, OpenAI API, Docker, K8s, CI/CD, Prometheus, TTFT/TPOT |
| RAG 问答系统 | P0 | 文档切片→Embedding→ChromaDB→本地 LLM 生成，含检索质量评估 | RAG, Embedding, ChromaDB, LLM |
| vLLM 架构源码分析 | P1 | 精读 vLLM 核心源码（PagedAttention/Scheduler），手算 KV Cache | vLLM, PagedAttention, Scheduler, KV Cache, SGLang |

### 7.2 完整叙事模板

> 「我独立完成了三个 AI 应用工程项目的端到端交付：从 HF Transformers 双模型推理基础（tokenizer、参数调优、8-bit 量化），到 LLM 流式推理服务化（OpenAI 兼容 API + SSE + TTFT/TPOT + K8s 部署 + GitHub Actions CI/CD + Prometheus 监控），再到 RAG 检索增强生成（文档切片→Embedding→ChromaDB→本地 LLM 生成带引用答案）。另外系统学习过 vLLM 架构，能讲清 PagedAttention 和 Continuous Batching 原理。所有项目代码在 GitHub 公开，有完整的 benchmark 数据和踩坑记录。」

### 7.3 目标岗位与搜索词

**主攻：**
- `AI 应用开发工程师`
- `AI 平台后端开发`
- `大模型应用开发`
- `RAG 开发工程师`

**辅攻：**
- `大模型推理部署`
- `ML Platform`
- `vLLM`

**关注公司类型**：AI 初创（20-200 人）、中厂 AI 部门、大厂 AI 应用团队

---

## 八、常见问题

### Q1: 为什么不用 ONNX Runtime 做项目？

JD 中 vLLM/HF Transformers 出现频率远高于 ORT。ORT 的核心概念在 [ai_from_zero.md](ai_from_zero.md) 附录 C 已覆盖概念层。简历上 HF Transformers + vLLM 的组合对 HR 的命中率远高于 ORT + ResNet。

### Q2: Step 3 为什么是先源码后实验，而不是直接做性能对比？

**CPU 模式下 vLLM 的性能优势无法体现。** 0.6B 模型的 KV Cache 只有几十 MB，碎片化不存在；CPU 计算瓶颈 > 内存瓶颈（和 GPU 相反）。CPU 上实测 vLLM 可能比 HF 更慢——这不证明 vLLM 不好，只说明场景不对。

**所以设计了两步：** 核心路径在 CPU 上跑通 + 精读源码 + 理论推演（免费，深度理解）；强化路径租 GPU 2-4h 跑真实 benchmark（10-20 元，验证理论，给简历数据）。两步互补——面试时既能讲源码原理，又有实验数据佐证。

**租 GPU 策略：** 先在 CPU 上完成所有源码学习和理论推演 → 准备好 benchmark 脚本 → 租 T4 一次性跑完（2-4h 连续工作）→ 关机释放。不要边学边开着 GPU。

### Q3: 用 0.6B 小模型，HR 会不会觉得浅？

单独看会。所以 v3 做了两件事来对冲：① Step 1 用了两个不同模型（Qwen3 + Llama），展示多模型通用能力 ② Step 3 的理论推演是以 7B 模型为假设场景的。面试时你展示的是方法论和架构理解，模型大小只是实验配置。

### Q4: 要不要学 Kubernetes？

**要，而且已经在 Step 2 末尾安排了。** K8s 是 JD 高频词（4/6），v4 在 Step 2 增加了 Minikube 实操部署（+3h）——把 Docker Compose 服务翻译为 K8s Deployment + Service + ConfigMap，`kubectl apply` 跑通。不是"了解概念"，是"动手做过"。这足以应对 AI 应用开发岗的 K8s 要求。更深的 K8s（Operator 开发、GPU 调度策略、多节点集群）入职后结合实际环境学习。

### Q5: 要不要学量化（AWQ/GPTQ 等）？

Step 1 已有 8-bit 量化的实操对比。更深入的量化算法实现（AWQ/GPTQ/SmoothQuant）是推理优化工程师的专业领域，推理部署工程师知道「什么是量化、为什么能省显存、实际效果（内存和速度的数字）」就够。

### Q6: 每个项目需要写单元测试吗？

Step 1 不需要（benchmark 脚本即测试）。Step 2 写 2-3 个关键端点测试。Step 3 是分析文档为主，不需要测试。Step 4 不强制。

### Q7: 项目是每个独立仓库还是 monorepo？

推荐**每个 Step 独立仓库**，方便面试时分别展示。每个仓库 README 自包含。README 底部加「前置项目」链接形成项目链。

### Q8: SGLang 需要实操吗？

不需要。Step 3 的概念对比已经足够。3/6 JD 提到 SGLang，但你展示的是「了解生态、知道差异」而不是每个都要写代码。

### Q9: 每个项目的 README 应该写什么？

统一模板：
1. **项目目的**（一句话）
2. **架构图**（ASCII 或 mermaid）
3. **快速开始**（3 步跑起来：安装→启动→验证）
4. **核心结果**（表格/图表）
5. **踩坑记录**（至少 2-3 个实际遇到的问题和解决方案——这是面试时最有话聊的部分）
6. **前置项目**（链接到上一步的仓库）

---

## 附录：执行检查清单

### Step 1 完成标准
- [ ] `benchmark.py` 可运行，输出 avg/P50/P99 + tokens/s
- [ ] 至少对比了 temperature/top_p/max_tokens 4 组参数组合
- [ ] 量化实验有 FP32 vs INT8 的具体数字
- [ ] 能说出 Qwen3 和 Llama tokenizer 的一个具体差异
- [ ] GitHub 公开仓库 + README（含踩坑记录）

### Step 2 完成标准
- [ ] `docker-compose up` 一键启动
- [ ] `/v1/chat/completions` 支持 SSE 流式和完整 JSON 两种模式
- [ ] `/metrics` 有 TTFT/TPOT histogram
- [ ] `python bench.py` 压测有 3 组并发的 TTFT/TPOT 对比
- [ ] 日志每条有 request_id
- [ ] **【v4 新增】** `kubectl apply -f k8s/` 部署成功
- [ ] **【v4 新增】** GitHub Actions CI 绿色（构建 + 测试 + 报告）
- [ ] README 含踩坑记录（至少记录 TextIteratorStreamer 的线程坑 + K8s 部署坑）

### Step 3 完成标准（P1 加分项，完成前 6 项即达标）

**最低交付（8-10h）：**
- [ ] `vllm serve` 成功启动并提供 OpenAI 兼容 API
- [ ] 能画出 LLMEngine → Scheduler → Worker 调用链
- [ ] PagedAttention 笔记：用自己的话解释 Block Table 和 Copy-on-Write
- [ ] Scheduler 笔记：能描述 Continuous Batching 的执行流程
- [ ] KV Cache 手算过程与 config 参数一致
- [ ] 能说出 vLLM 和 SGLang 的一个架构差异

**有余力再做：**
- [ ] 完整的源码精读笔记（block_manager.py + scheduler.py）
- [ ] 理论推演文档（7B + 100 并发场景分析）
- [ ] GPU benchmark：HF vs vLLM 在不同并发下的 TTFT/TPOT/吞吐对比

### Step 4 完成标准（P0 推荐）
- [ ] 检索 recall@3 可量化
- [ ] 生成答案带引用来源
- [ ] 生成 pipeline 使用本地模型（非外部 API）
- [ ] 10 个测试问题的评估报告
- [ ] `/ask` 端点 FastAPI 服务化（复用 Step 2 经验）

---

## 版本历史

| 版本 | 日期 | 主要变更 |
|------|------|---------|
| v1 | 2026-06-17 | 初版：ONNX + ResNet → vLLM 路径 |
| v2 | 2026-06-17 | 切换到 HF Transformers + LLM 模型 |
| v3 | 2026-06-17 | 双模型对比 + 量化实验 + 国内镜像 + 理论推演 |
| v4 | 2026-06-17 | **市场校准版**：① 目标岗从「推理部署」调整为主攻「AI 应用开发」② Step 2 新增 K8s + CI/CD 实操（+6h）③ Step 3 从 P0 降为 P1 加分项 ④ Step 4 从可选升级为 P0 推荐 ⑤ 优先级重新排序：1-2-4 为最小可行证据链 ⑥ FAQ + 搜索词 + 叙事模板同步更新 |
