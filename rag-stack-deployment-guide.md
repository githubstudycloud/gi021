# RAG 技术栈部署指南：BGE-M3 + Qwen2.5-14B

> 版本：1.0 | 日期：2026-03-16
> 覆盖：嵌入模型（BGE-M3）+ 推理模型（Qwen2.5-14B）完整部署规划

---

## 一、BGE-M3 部署需求

### 1.1 模型基本信息

| 项目 | 参数 |
|------|------|
| 架构 | XLM-RoBERTa-Large（持续预训练） |
| 参数量 | **568M**（约 Qwen2.5-14B 的 1/25） |
| 最大输入 | 8,192 tokens |
| 输出维度 | 1,024 |
| 检索模式 | Dense / Sparse / ColBERT（可单独启用）|
| 支持语言 | 100+ |

---

### 1.2 显存需求

BGE-M3 模型极小，显存几乎不是瓶颈：

| 精度 | 模型显存 | 批处理显存（batch=64）| 单卡可运行实例数（24G）|
|------|---------|---------------------|----------------------|
| FP32 | ~2.1 GB | +2–4 GB | ~4 |
| **FP16/BF16** | **~1.1 GB** | +1–2 GB | **~8–10** |
| INT8 | ~0.6 GB | +0.5–1 GB | ~15+ |
| INT4 | ~0.27 GB | +0.3 GB | ~20+ |

> **关键结论**：BGE-M3 几乎可以忽略显存限制。一张 24GB 显卡可以同时跑 **8–10 个 FP16 实例**，或跑 1 个实例服务极高并发。

---

### 1.3 系统内存需求

| 场景 | 最低内存 | 推荐内存 |
|------|---------|---------|
| 单实例 CPU 推理 | 4 GB | 8 GB |
| 单实例 GPU 推理 | 8 GB | 16 GB |
| 多实例 / 高并发队列 | 16 GB | **32 GB** |
| 含向量数据库（如 Milvus）| 32 GB | **64 GB+** |

---

### 1.4 单机并发能力

BGE-M3 是编码器（Encoder-only），**无 KV Cache，无自回归解码**，吞吐特性与 LLM 完全不同。

#### 影响吞吐的核心因素
```
吞吐量 ∝ batch_size / seq_len
瓶颈顺序：CPU 分词(tokenizer) → GPU 矩阵计算 → 结果返回
```

#### 实测吞吐参考（单实例，FP16，Dense-only 模式）

| GPU | batch_size | seq_len | 吞吐（句/秒）| 备注 |
|-----|-----------|---------|------------|------|
| RTX 4090 (24G) | 64 | 512 | **~3,000–5,000** | 消费级最优 |
| RTX 4090 (24G) | 32 | 128 | **~8,000–12,000** | 短文本场景 |
| A100 80G | 128 | 512 | **~10,000–18,000** | 批处理极限 |
| A100 80G | 64 | 128 | **~25,000–40,000** | 短文本极限 |
| CPU（32核）| 32 | 512 | ~200–500 | 不推荐高并发 |

> Dense + Sparse + ColBERT 全模式吞吐约为 Dense-only 的 **1/3**，建议按需启用。

#### 单机 QPS（在线服务，单请求不批处理）

| GPU | seq_len 128 | seq_len 512 | seq_len 2048 |
|-----|-----------|------------|-------------|
| RTX 4090 | ~500–800 QPS | ~150–300 QPS | ~30–60 QPS |
| A100 80G | ~1,000–2,000 QPS | ~400–800 QPS | ~80–150 QPS |

> **注**：使用 HuggingFace TEI（Text Embeddings Inference）+ 动态批处理可将吞吐提升 **3–5 倍**，强烈推荐生产部署使用 TEI 而非原生 vLLM。

#### vLLM 的 CPU 瓶颈警告
```
已知问题（vllm issue #11320）：
  BGE-M3 在 vLLM 下，并发 > 12 TPS 时 Python tokenizer CPU 达到 100%
  GPU 反而处于空闲状态（GPU 利用率 < 30%）

解决方案：
  1. 改用 HuggingFace TEI（专为 embedding 模型优化）
  2. 增加 --tokenizer-pool-size（vLLM 临时缓解方案）
  3. 预处理阶段将 tokenization 移出关键路径
```

---

### 1.5 推荐部署方式（BGE-M3）

```bash
# 方式一：HuggingFace TEI（推荐，吞吐最高）
docker run --gpus all \
  -p 8080:80 \
  -v ~/.cache/huggingface:/data \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-m3 \
  --dtype float16 \
  --max-batch-tokens 131072 \
  --max-concurrent-requests 512

# 方式二：FlagEmbedding 原生（灵活，支持三路检索）
from FlagEmbedding import BGEM3FlagModel
model = BGEM3FlagModel('BAAI/bge-m3', use_fp16=True)
embeddings = model.encode(sentences, batch_size=64, max_length=512)

# 方式三：vLLM（统一接口，但注意 CPU 瓶颈）
vllm serve BAAI/bge-m3 \
  --task embedding \
  --dtype float16 \
  --max-num-batched-tokens 131072 \
  --tokenizer-pool-size 8
```

---

## 二、Qwen2.5-14B 部署需求（摘要）

> 详细数据见：[qwen2.5-14b-deployment-guide.md](./qwen2.5-14b-deployment-guide.md)
> 32GB 显存专项分析：[qwen2.5-14b-32gb-vram-concurrency.md](./qwen2.5-14b-32gb-vram-concurrency.md)

### 2.1 显存需求速查

| 精度 | 权重显存 | 推荐GPU | 备注 |
|------|---------|--------|------|
| BF16 | 28 GB | A100/H100 80G | 精度最高 |
| INT8 | 14 GB | 任意 32G+ GPU | 推荐平衡方案 |
| INT4/AWQ | ~7.5 GB | RTX 4090 | 轻量化 |

### 2.2 32 GB 显存单卡并发速查

| 精度 | 2K 上下文并发 | 4K 上下文并发 |
|------|------------|------------|
| BF16 | ❌ 不可用 | ❌ |
| INT8 | **~37** | **~18** |
| INT4 | **~53** | **~26** |

---

## 三、联合部署方案（RAG 系统）

### 3.1 典型 RAG 请求链路

```
用户请求
    │
    ▼
[BGE-M3] 向量化 query（~5–20ms）
    │
    ▼
[向量数据库] 召回 Top-K 文档（~5–50ms）
    │
    ▼
[BGE-M3 Reranker] 重排序（可选，~20–100ms）
    │
    ▼
[Qwen2.5-14B] 生成答案（~500ms–5s）
    │
    ▼
返回结果
```

**瓶颈分析**：Qwen2.5-14B 生成时间占整体链路 **80–95%**，BGE-M3 编码时间可忽略。

---

### 3.2 单机（单节点）显存分配方案

#### 方案 A：A100 80G 单卡混合部署

```
┌─────────────────────────────────────────┐
│           A100 80 GB 显存分配             │
├──────────────────┬──────────────────────┤
│ BGE-M3 (FP16)   │       ~1.5 GB        │
│ Qwen2.5-14B INT8│      ~14.0 GB        │
│ 两模型 KV Cache  │      ~57.5 GB        │
│ 系统预留         │       ~7.0 GB        │
└──────────────────┴──────────────────────┘

LLM 并发：~60–80（2K 上下文）
Embedding QPS：~600–1,000（共享 GPU，有竞争）
推荐：BGE-M3 和 LLM 分进程部署，各占固定显存分区
```

#### 方案 B：32 GB 显存单卡分工部署

```
┌─────────────────────────────────────────┐
│         32 GB 显存分配（INT8）            │
├──────────────────┬──────────────────────┤
│ BGE-M3 (FP16)   │       ~1.5 GB        │
│ Qwen2.5-14B INT8│      ~14.0 GB        │
│ LLM KV Cache    │      ~13.0 GB        │
│ 系统预留         │       ~3.5 GB        │
└──────────────────┴──────────────────────┘

LLM 并发：~25–35（2K 上下文，显存被 BGE 占用部分）
Embedding QPS：~300–500
注意：两模型争抢显存带宽，实际吞吐低于独立部署
```

#### 方案 C：双卡分离部署（推荐生产）

```
GPU 0（专用 BGE-M3）              GPU 1（专用 Qwen2.5-14B）
┌──────────────────┐               ┌──────────────────┐
│ BGE-M3 × 4 实例  │               │ Qwen INT8        │
│ 显存：4 × 1.5G   │               │ 权重：14G        │
│ = ~6 GB 使用     │               │ KV Cache：~14G   │
│ QPS：~2,000+     │               │ 并发：~35–45     │
└──────────────────┘               └──────────────────┘
互不影响，各自满载运行
```

---

### 3.3 支撑 300 并发 RAG 的完整集群规划

#### 需求分析
```
目标：300 并发 RAG 请求
链路假设：
  - 每次 RAG 调用触发 1 次 BGE-M3 编码（~10ms）
  - 1 次向量检索
  - 1 次 Qwen2.5-14B 生成（平均 2s，2K 上下文）
```

#### 各组件实例数

| 组件 | 单实例并发 | 目标并发 | 所需实例 | 含冗余（N+1）|
|------|---------|---------|---------|------------|
| **Qwen2.5-14B INT8**（32G GPU）| ~37 | 300 | 9 | **10 实例** |
| **Qwen2.5-14B INT4**（32G GPU）| ~53 | 300 | 6 | **7 实例** |
| **Qwen2.5-14B INT8**（A100 80G）| ~70 | 300 | 5 | **6 实例** |
| **BGE-M3 TEI**（RTX 4090）| ~2,000 QPS | 300 QPS | 1 | **2 实例** |
| **向量数据库**（Milvus）| - | - | - | **3 节点（集群）**|

> BGE-M3 的吞吐远超 LLM，**2 个 BGE-M3 实例即可支撑 300+ 并发**，无需水平扩展。

#### 推荐集群拓扑（32G 显存，INT8）

```
                     ┌──────────────────────┐
                     │     API Gateway       │
                     │   (限流 + 鉴权)        │
                     └──────────┬───────────┘
                                │
           ┌────────────────────┼─────────────────────┐
           ▼                    ▼                      ▼
  ┌─────────────────┐  ┌─────────────────┐   ┌─────────────────┐
  │  BGE-M3 节点 #1  │  │  BGE-M3 节点 #2  │   │  Milvus 集群    │
  │  TEI + RTX 4090 │  │  TEI + RTX 4090 │   │  3 节点         │
  │  ~2000 QPS      │  │  备用/冗余       │   │  向量存储        │
  └────────┬────────┘  └────────┬────────┘   └────────┬────────┘
           └────────────────────┘                      │
                       │                               │
                       └──────────────┬────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼           ▼           ▼           ▼           ▼
     ┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
     │ Qwen 实例 #1  │ │ #2       │ │ #3       │ │ #4 ... #9│ │ #10 备用 │
     │ 32G GPU INT8 │ │ 32G INT8 │ │ 32G INT8 │ │ 32G INT8 │ │ Standby  │
     │ ~37 并发     │ │ ~37 并发  │ │ ~37 并发  │ │ ~37 并发  │ │          │
     └──────────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

---

### 3.4 系统内存规划（各节点）

| 节点类型 | 推荐内存 | 说明 |
|---------|---------|------|
| BGE-M3 节点 | **32 GB** | 模型 + TEI 运行时 + 请求队列 |
| Qwen LLM 节点 | **128 GB** | vLLM KV 溢出保护 + 系统余量 |
| Milvus 节点 | **64–256 GB** | 向量索引内存映射（依数据量） |

---

## 四、软件栈推荐

| 组件 | 推荐方案 | 备选 |
|------|---------|------|
| BGE-M3 服务 | **HuggingFace TEI** | vLLM（注意 CPU 瓶颈）|
| Qwen LLM 服务 | **vLLM** | SGLang（更高吞吐）|
| 向量数据库 | **Milvus** | Qdrant、Weaviate |
| API 网关 | **Nginx + OpenResty** | Kong、APISIX |
| 监控 | **Prometheus + Grafana** | - |
| 容器编排 | **Kubernetes + GPU Operator** | Docker Compose（小规模）|

---

## 五、成本对比总结

### 场景：支撑 300 并发 RAG，使用 32G GPU

| 方案 | BGE-M3 | Qwen2.5-14B | 总 GPU 数 | 参考月费（云端）|
|------|--------|------------|---------|--------------|
| INT8 + TEI | 2× RTX4090 | 10× 32G GPU | **12** | ~$3,000–5,000 |
| INT4 + TEI | 2× RTX4090 | 7× 32G GPU | **9** | ~$2,200–3,800 |
| INT8 + A100 80G | 2× RTX4090 | 6× A100 80G | **8** | ~$5,000–9,000 |

---

## 六、参考资料

- [BAAI/bge-m3 HuggingFace 模型卡](https://huggingface.co/BAAI/bge-m3)
- [BGE-M3 官方文档](https://bge-model.com/bge/bge_m3.html)
- [HuggingFace Text Embeddings Inference](https://github.com/huggingface/text-embeddings-inference)
- [vLLM BGE-M3 CPU 瓶颈 Issue #11320](https://github.com/vllm-project/vllm/issues/11320)
- [NVIDIA NIM BGE-M3 模型卡](https://build.nvidia.com/baai/bge-m3/modelcard)
- [Qwen2.5 速度基准](https://qwen.readthedocs.io/en/latest/getting_started/speed_benchmark.html)
- [A100 80GB vLLM 基准](https://www.databasemart.com/blog/vllm-gpu-benchmark-a100-80gb)
- 本项目相关文档：
  - [Qwen2.5-14B 完整部署指南](./qwen2.5-14b-deployment-guide.md)
  - [Qwen2.5-14B 32GB 显存并发分析](./qwen2.5-14b-32gb-vram-concurrency.md)
