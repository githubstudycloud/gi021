# RAG 技术栈调研与部署规划总结

> 日期：2026-03-17 | 作者：本地知识库项目规划

---

## 项目背景

基于个人 NAS 家庭实验室，构建一套全天候可用的私有 RAG（检索增强生成）系统。
主机（GPU）平时关机，NAS 常驻待机；正常时接入云端 LLM API，断网自动降级本地小模型。

---

## 硬件清单

| 设备 | 类型 | 规格 | 常驻 |
|------|------|------|------|
| 极空间 Z423 标准版 | NAS | R5 5625U / 64GB DDR4 / 2TB SSD + 8TB HDD | ✅ 常驻 |
| 绿联 DXP4800 Plus | NAS | Intel 8505 / 64GB DDR5 / 2TB SSD + 8TB HDD | ✅ 常驻 |
| PC-A | 主机 | RTX 5070 12GB | 按需开机 |
| PC-B | 主机 | RTX 4060 8GB | 按需开机 |

---

## 文档索引

| # | 文档 | 核心内容 |
|---|------|---------|
| 1 | [Qwen2.5-14B 完整部署指南](./qwen2.5-14b-deployment-guide.md) | 各精度显存需求、A100 并发基准、300 并发集群规划 |
| 2 | [32GB 显存并发分析](./qwen2.5-14b-32gb-vram-concurrency.md) | 32GB 单卡各精度实际可用并发数 |
| 3 | [BGE-M3 + Qwen2.5-14B RAG 架构](./rag-stack-deployment-guide.md) | 完整 RAG 链路、组件配合、300 并发集群方案 |
| 4 | [RAG 组件选型指南](./rag-components-selection-guide.md) | 向量数据库横评、Reranker 选型、最低资源需求 |
| 5 | [家庭实验室 GPU 部署](./home-lab-rag-deployment.md) | RTX 5070+4060+NAS 四节点方案（主机开机时）|
| 6 | [NAS 常驻部署（64GB 版）](./nas-64gb-rag-deployment.md) | **当前采用方案**，Docker Compose 完整配置 |
| 7 | [NAS 常驻部署（旧版）](./nas-only-rag-deployment.md) | 历史参考，已被 64GB 版取代 |

---

## 一、核心模型资源需求速查

### Qwen2.5-14B（LLM）

| 精度 | 显存 | 单 A100 80G 并发 | 单 32G 显存并发 |
|------|------|----------------|---------------|
| BF16 | 28 GB | 60–80 | ❌ 放不下 |
| INT8 | 14 GB | 80–100 | ~37（2K 上下文）|
| **INT4/AWQ** | **7.5 GB** | 80–100+ | **~53（2K 上下文）**|
| Q4_K_M（GGUF）| ~8.5 GB RAM | CPU 推理 | ~5–7 tok/s（R5 5625U）|

### BGE-M3（Embedding）

| 精度 | 显存/内存 | 吞吐（512 token）| 说明 |
|------|---------|----------------|------|
| GPU FP16 | ~1.1 GB 显存 | 3,000–5,000 句/s | RTX 4090 |
| CPU FP32 | ~2 GB 内存 | 30–50 QPS | NAS 场景 |

> 推荐服务框架：**HuggingFace TEI**（吞吐比原生 vLLM 高 3–5 倍，且无 CPU 瓶颈问题）

### BGE-Reranker-v2-M3（精排）

| 精度 | 显存/内存 | 每次精排（20 候选）| 说明 |
|------|---------|----------------|------|
| GPU FP16 | ~1.1 GB 显存 | ~30–50 req/s（RTX 4090）| |
| CPU FP32 | ~2 GB 内存 | ~5–10 req/s（NAS）| 可接受 |

---

## 二、向量数据库选型结论

| 场景 | 推荐方案 | 最低内存 |
|------|---------|---------|
| NAS 家庭部署 | **Qdrant** | ~1 GB |
| 已有 PostgreSQL | pgvector | ~512 MB |
| 超大规模（亿+）| Milvus | ~8 GB |
| 纯开发测试 | Chroma | ~256 MB |

**Qdrant 关键限制**：不支持 NFS，必须挂载本地磁盘（SSD/HDD 均可）。

---

## 三、当前采用方案（NAS 常驻，64GB 版）

### 服务分工

```
极空间 Z423（推理节点）           绿联 DXP4800 Plus（数据节点）
R5 5625U + 64GB DDR4             Intel 8505 + 64GB DDR5 + 万兆网口
─────────────────────────        ──────────────────────────────────
Ollama                           Qdrant（SSD热 + HDD冷）
  ├ Qwen2.5-7B  Q4（~8-12tok/s） Redis（16GB 大缓存）
  └ Qwen2.5-14B Q4（~5-7 tok/s） Nginx（智能路由网关）
BGE-M3 TEI（CPU Embedding）      Open WebUI（网页聊天界面）
bge-reranker-v2-m3 TEI           Prometheus + Grafana（监控）
─────────────────────────        ──────────────────────────────────
内存占用：~25 GB / 64 GB          内存占用：~20-40 GB / 64 GB
```

### LLM 调用策略

```
正常联网  ─→  云端 API（DashScope Qwen-Plus / OpenAI）  ← 低延迟，高质量
断网降级  ─→  本地 Qwen2.5-14B Q4（~5-7 tok/s）         ← 高质量离线
紧急模式  ─→  本地 Qwen2.5-7B  Q4（~8-12 tok/s）        ← 快速响应
```

### 存储布局

```
SSD（2TB）：模型文件 + Qdrant 热索引 + Redis 持久化   ← 低延迟
HDD（8TB）：原始文档 + Qdrant 冷向量 + 备份归档       ← 大容量
```

### Qdrant 容量上限

| 存储模式 | 可存向量数（1024 维）|
|---------|-------------------|
| 纯内存（64GB 配额 20GB）| ~2400 万 |
| SSD mmap（200GB）| ~2.4 亿 |
| HDD mmap（4TB）| ~50 亿 |

---

## 四、GPU 主机开机时的能力提升

| 服务 | NAS 常驻（CPU）| PC-B RTX 4060 开机 | PC-A RTX 5070 开机 |
|------|-------------|-------------------|-------------------|
| Embedding QPS | ~30–50 | ~300–600 ↑10x | — |
| Reranker QPS | ~5–10 | ~15–25 ↑3x | — |
| LLM 并发 | ~1–2（本地）| — | ~6–8（14B INT4）↑4x |
| LLM 速度 | ~5–7 tok/s | — | ~20 tok/s ↑3x |

> GPU 主机开机时，各服务自动接管对应任务，NAS 退为数据层，无需修改配置（Nginx 路由切换）。

---

## 五、全套部署最低资源需求

| 部署模式 | 最低显存 | 最低内存 | 说明 |
|---------|---------|---------|------|
| 纯 NAS 常驻 | 0 GB | **16 GB**（极空间）| 7B 小模型 + Qdrant 基础 |
| NAS + 本地 14B | 0 GB | **32 GB**（极空间）| 14B CPU 推理稳定运行 |
| **当前方案（64GB）** | 0 GB | **64 GB × 2** | 全服务常驻，充裕 |
| GPU 加速（PC 开机）| **12 GB**（RTX 5070）| 64 GB | LLM GPU 加速 |

---

## 六、RAG 请求链路（端到端）

```
用户输入 query
    │
    ▼ [BGE-M3 CPU，~20–50ms]
向量化 query → 1024 维向量
    │
    ▼ [Qdrant HNSW，~10–30ms]
召回 Top-100 候选文档
    │
    ▼ [bge-reranker-v2-m3 CPU，~100–300ms，20 篇精排]
精排 → Top-5 最相关文档
    │
    ▼ [云端 API ~1-3s / 本地 14B ~30-60s]
拼接 Prompt → LLM 生成答案
    │
    ▼
返回结果

端到端延迟：
  正常联网：~1.5–4s（云端 LLM 主导）
  断网本地：~35–65s（14B CPU 推理主导）
```

---

## 七、后续扩展方向

| 方向 | 方案 | 优先级 |
|------|------|-------|
| 可视化 RAG 编排 | 部署 **Dify**（Docker，免费）| ⭐⭐⭐ |
| 文档解析增强 | Docling / marker-pdf | ⭐⭐⭐ |
| 多模态支持 | Qwen2.5-VL（需 GPU）| ⭐⭐ |
| 知识图谱增强 | GraphRAG / Neo4j | ⭐⭐ |
| 自动化测试评估 | RAGAs 框架 | ⭐⭐ |
| GPU 主机自动唤醒 | WoL（网络唤醒）+ 任务队列 | ⭐ |

---

## 参考来源

- [Qwen2.5 官方速度基准](https://qwen.readthedocs.io/en/latest/getting_started/speed_benchmark.html)
- [LocalScore RTX 5070 实测](https://www.localscore.ai/accelerator/168)
- [HuggingFace TEI](https://github.com/huggingface/text-embeddings-inference)
- [Qdrant 容量规划](https://qdrant.tech/documentation/guides/capacity-planning/)
- [BAAI/bge-m3](https://huggingface.co/BAAI/bge-m3)
- [BAAI/bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3)
- [Ollama](https://github.com/ollama/ollama)
- [Open WebUI](https://github.com/open-webui/open-webui)
- [极空间 Z423 标准版评测](https://post.smzdm.com/p/admvgook/)
- [绿联 DXP4800 Plus 官方](https://www.ugnas.com/products-detail/id-39.html)
