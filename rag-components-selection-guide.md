# RAG 组件选型指南：向量数据库 + Reranker

> 版本：1.0 | 日期：2026-03-16
> 配套文档：[BGE-M3 + Qwen2.5-14B 部署指南](./rag-stack-deployment-guide.md)

---

## 一、向量数据库选型

### 1.1 主流方案对比

| 数据库 | 语言 | 内存模式 | 最低内存 | 部署复杂度 | 最大向量规模 | 适合场景 |
|--------|------|---------|---------|---------|------------|---------|
| **Qdrant** | Rust | 内存/mmap/磁盘 | **~1 GB** | ⭐ 极简（单二进制）| 亿级 | 中小规模生产首选 |
| **Milvus** | Go/C++ | 内存为主 | **~8 GB**（含etcd+MinIO）| ⭐⭐⭐⭐（需K8s）| 百亿级 | 超大规模企业 |
| **Weaviate** | Go | 内存/磁盘 | **~2 GB** | ⭐⭐⭐ | 十亿级 | 知识图谱+向量混合 |
| **pgvector** | C（PG扩展）| PostgreSQL内存 | **~512 MB** | ⭐⭐ | <100万（性能下降）| 已有PG、轻量场景 |
| **Chroma** | Python | 内存/SQLite | **~256 MB** | ⭐ 极简 | <100万 | 开发测试 |
| **FAISS** | C++ | 纯内存 | 视数据量 | ⭐⭐（需自己封装）| 亿级 | 离线批处理 |

---

### 1.2 内存消耗计算

以 BGE-M3 输出向量（1024 维，FP32）为例：

```
每个向量内存 = 1024 维 × 4 字节 = 4 KB
1 万个向量  = 4 KB × 10,000 = ~40 MB
10 万个向量 = ~400 MB
100 万个向量 = ~4 GB（含索引和元数据约 × 1.5 = ~6 GB）
1000 万向量 = ~60 GB

通用公式：
  内存(GB) = 向量数 × 1024维 × 4字节 × 1.5 ÷ 1024³
```

| 向量数 | 纯内存模式 | 量化(INT8)后 | mmap磁盘模式 |
|--------|----------|------------|------------|
| 10 万 | ~600 MB | ~150 MB | ~100 MB + SSD |
| 100 万 | ~6 GB | ~1.5 GB | ~500 MB + SSD |
| 1000 万 | ~60 GB | ~15 GB | ~2 GB + SSD |
| 1 亿 | ~600 GB | ~150 GB | ~10 GB + SSD |

---

### 1.3 各方案详细说明

#### Qdrant（推荐中小规模首选）

```
优势：
  ✅ 单二进制部署，Docker 一行启动
  ✅ 内存/mmap/磁盘三种模式自由切换
  ✅ 内置向量量化（节省 4–64x 内存）
  ✅ Rust 实现，极低内存开销（空跑 < 50 MB）
  ✅ 支持混合检索（向量 + 关键词）
  ✅ REST + gRPC API

劣势：
  ❌ 不支持 NFS/S3 存储（需本地 SSD）
  ❌ 分布式模式配置较复杂

最低部署要求：
  RAM: 1 GB（空跑）
  存储: SSD 推荐（NVMe 更佳），不支持 NFS
```

```bash
# 启动命令（极简）
docker run -p 6333:6333 -v ./qdrant_data:/qdrant/storage \
  qdrant/qdrant

# 内存映射模式（大数据量低内存）
# 在 collection 创建时设置 on_disk: true
```

---

#### Milvus（推荐超大规模）

```
优势：
  ✅ GPU 加速向量索引（NVIDIA GPU）
  ✅ 支持十亿级向量
  ✅ 最丰富的索引类型（IVF/HNSW/DiskANN）

劣势：
  ❌ 依赖 etcd + MinIO + 多组件，最低 8 GB RAM
  ❌ 部署和运维复杂，适合有 K8s 经验的团队
  ❌ 小数据量下资源浪费严重

最低部署要求：
  RAM: 8 GB（etcd 2G + MinIO 2G + Milvus 4G）
  推荐: 32 GB+
```

---

#### pgvector（推荐已有 PostgreSQL 用户）

```
优势：
  ✅ 无需新增服务，直接在 PG 中存向量
  ✅ 可与业务数据 JOIN 查询
  ✅ 成熟的 PostgreSQL 生态

劣势：
  ❌ 向量维度上限 2000（BGE-M3 的 1024 维OK，但余量不大）
  ❌ 超过 100 万向量后查询性能显著下降
  ❌ 高并发向量查询会占用大量 shared_buffers

最低部署要求：
  RAM: 512 MB（PostgreSQL 本身）+ 向量索引内存
```

---

### 1.4 选型决策树

```
数据量 < 100 万且已有 PostgreSQL？
  └─ 是 → pgvector（零新增服务）

数据量 < 1000 万，追求简单运维？
  └─ 是 → Qdrant ★推荐★

数据量 > 1000 万 或 需要 GPU 加速检索？
  └─ 是 → Milvus

需要知识图谱 + 向量混合？
  └─ 是 → Weaviate

仅开发测试？
  └─ 是 → Chroma（最简单）
```

---

## 二、Reranker 选型

### 2.1 Reranker 工作原理

```
Bi-encoder（BGE-M3 Embedding）：
  query → 向量化 → 向量相似度检索 → Top-100
  特点：快速，但粗排

Cross-encoder（Reranker）：
  [query, doc] → 逐对精排 → 相关性评分
  特点：慢（需逐对计算），但精度高

完整 RAG 链路：
  query → BGE-M3 召回 Top-100 → Reranker 精排 → Top-5 → LLM 生成
```

---

### 2.2 主流 Reranker 对比

| 模型 | 参数量 | FP16 显存 | 语言 | 精度 | 速度 | 适用场景 |
|------|--------|---------|------|------|------|---------|
| **bge-reranker-v2-m3** | 568M | ~1.1 GB | 100+语言 | ⭐⭐⭐⭐⭐ | 中 | 多语言生产首选 |
| **bge-reranker-large** | 335M | ~0.7 GB | 中英文 | ⭐⭐⭐⭐ | 快 | 中英文高速场景 |
| **bge-reranker-base** | 278M | ~0.6 GB | 中英文 | ⭐⭐⭐ | 最快 | 资源极度受限 |
| **bge-reranker-v2-minicpm-layerwise** | 2.7B | ~2.5 GB | 多语言 | ⭐⭐⭐⭐⭐ | 慢 | 精度极致优先 |
| **ms-marco-MiniLM-L-6-v2** | 22M | ~45 MB | 英文 | ⭐⭐⭐ | 最快 | 纯英文超低资源 |
| **jina-reranker-v2-base** | 278M | ~0.6 GB | 多语言 | ⭐⭐⭐⭐ | 快 | Jina 生态用户 |

---

### 2.3 bge-reranker-v2-m3 详细规格

| 项目 | 数值 |
|------|------|
| 架构 | BGE-M3 backbone（XLM-RoBERTa-Large）|
| 参数量 | **568M** |
| FP16 显存 | **~1.1 GB** |
| INT8 显存 | ~0.6 GB |
| CPU 推理（batch=16，512 token）| ~130 ms |
| GPU 推理（A100，batch=23，512 token）| ~5.4 req/s |
| GPU 推理（RTX 4090，batch=32）| ~30–50 req/s |
| 每次查询候选数建议 | 10–30 篇文档 |

```bash
# FlagEmbedding 部署
from FlagEmbedding import FlagReranker
reranker = FlagReranker('BAAI/bge-reranker-v2-m3', use_fp16=True)
scores = reranker.compute_score([['query', 'doc1'], ['query', 'doc2']])

# vLLM 部署
vllm serve BAAI/bge-reranker-v2-m3 \
  --task score \
  --dtype float16 \
  --max-num-batched-tokens 32768

# HuggingFace TEI 部署（推荐）
docker run --gpus all -p 8081:80 \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-reranker-v2-m3 \
  --dtype float16
```

---

### 2.4 Reranker 性能与并发

| GPU | 每请求候选数 | 每候选 seq_len | 并发 QPS |
|-----|------------|--------------|---------|
| RTX 4060 8G | 20 篇 | 512 | ~15–25 req/s |
| RTX 4090 24G | 20 篇 | 512 | ~40–60 req/s |
| A100 80G | 20 篇 | 512 | ~100–150 req/s |
| CPU（16核）| 20 篇 | 512 | ~2–5 req/s |

> 每次重排序 = 20 对 (query, doc) 同时计算，实际用户 QPS 可达到上述数值。

---

### 2.5 是否需要 Reranker？

```
数据量 < 1万文档，查询简单         → 不需要，BGE-M3 Dense 已足够
数据量 1万–100万，追求精度         → 推荐 bge-reranker-large（快）
数据量 > 100万 或多语言业务        → 推荐 bge-reranker-v2-m3（精度最高）
显存极度受限（< 2GB）              → bge-reranker-base 或 CPU 部署
```

---

## 三、完整 RAG 组件栈汇总

### 推荐技术栈（生产级）

| 组件 | 推荐方案 | 最低显存 | 最低内存 | 备注 |
|------|---------|---------|---------|------|
| **Embedding** | BGE-M3 + TEI | 1.5 GB GPU | 8 GB | 支持三路检索 |
| **向量数据库** | Qdrant | 0 GB GPU | **1 GB** | 纯 CPU 运行 |
| **Reranker** | bge-reranker-v2-m3 | 1.5 GB GPU | 4 GB | 可与 BGE-M3 共卡 |
| **LLM** | Qwen2.5-14B INT4 | **7.5 GB GPU** | 16 GB | llama.cpp/vLLM |
| **缓存** | Redis | 0 GB GPU | 512 MB | 缓存 embedding 结果 |
| **网关** | Nginx | 0 GB GPU | 256 MB | 限流 + 路由 |

### 最低资源总需求

```
显存（合并 Embedding + Reranker + LLM）：
  BGE-M3(1.1G) + Reranker(1.1G) + Qwen14B-INT4(7.5G) = ~10 GB 显存

内存（不含操作系统）：
  Qdrant(1G) + Redis(1G) + 服务进程(4G) + 系统余量(8G) = ~14 GB 内存

最低推荐配置：
  GPU: 12 GB 显存（放 LLM + 部分 Embedding）
  内存: 32 GB RAM
```

---

## 参考资料

- [Qdrant 内存消耗分析](https://qdrant.tech/articles/memory-consumption/)
- [Qdrant 容量规划文档](https://qdrant.tech/documentation/guides/capacity-planning/)
- [BAAI/bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3)
- [BGE Reranker 官方文档](https://bge-model.com/tutorial/5_Reranking/5.2.html)
- [bge-reranker-v2-m3 吞吐基准 Issue](https://github.com/FlagOpen/FlagEmbedding/issues/1088)
- [向量数据库对比 2025](https://liquidmetal.ai/casesAndBlogs/vector-comparison/)
- [Milvus vs pgvector 对比](https://zilliz.com/comparison/milvus-vs-pgvector)
