# 家庭实验室 RAG 部署方案
## 硬件：RTX 5070 12G + RTX 4060 8G + NAS×2（各 64GB 内存）

> 版本：1.0 | 日期：2026-03-16

---

## 一、硬件资源盘点

| 节点 | CPU | GPU | 内存 | 存储 | 定位 |
|------|-----|-----|------|------|------|
| **PC-A** | 宿主机 | RTX 5070 12GB GDDR7 | 取决于主机配置 | 本地 SSD | LLM 推理主节点 |
| **PC-B** | 宿主机 | RTX 4060 8GB GDDR6 | 取决于主机配置 | 本地 SSD | Embedding + Reranker 节点 |
| **NAS-1** | 低功耗 x86/ARM | ❌ 无 GPU | **64 GB** | HDD/SSD 阵列 | 向量数据库 + 缓存 |
| **NAS-2** | 低功耗 x86/ARM | ❌ 无 GPU | **64 GB** | HDD/SSD 阵列 | 网关 + 监控 + CPU 备用推理 |

---

## 二、各节点显存/内存详细分析

### 2.1 PC-A：RTX 5070 12GB — LLM 节点

```
RTX 5070 规格：
  显存：12 GB GDDR7
  带宽：672 GB/s（Blackwell 架构）
  性能：Qwen2.5-14B Q4_K_M 生成速度 ~20 tokens/s（实测）

Qwen2.5-14B INT4 (Q4_K_M) 显存分配：
  模型权重：~8.5 GB
  系统预留：~0.3 GB
  可用 KV Cache：12GB × 0.90 - 8.5GB = 2.3 GB

KV Cache 并发估算：
  上下文 1K  → 每 seq 160MB → 最大 ~14 路 → 实际可用 ~8-10 路
  上下文 2K  → 每 seq 320MB → 最大 ~7 路  → 实际可用 ~4-6 路
  上下文 4K  → 每 seq 640MB → 最大 ~3 路  → 实际可用 ~2-3 路
```

**结论：RTX 5070 12GB 单卡可运行 Qwen2.5-14B INT4，适合 4-8 个并发用户（取决于上下文长度）。**

---

### 2.2 PC-B：RTX 4060 8GB — Embedding + Reranker 节点

```
RTX 4060 规格：
  显存：8 GB GDDR6
  带宽：272 GB/s

显存分配（两模型共存）：
  BGE-M3 FP16：          ~1.1 GB
  bge-reranker-v2-m3 FP16：~1.1 GB
  激活缓冲（批处理）：     ~2.0 GB
  系统预留：              ~0.3 GB
  合计使用：             ~4.5 GB  ← 8GB 有余量
  剩余可用：             ~3.5 GB

并发能力：
  Embedding QPS（seq=512）：~300-600 QPS
  Reranker QPS（20候选，seq=512）：~15-25 req/s
  两模型分时运行，互不干扰
```

**结论：RTX 4060 8GB 足够运行 BGE-M3 + Reranker，8GB 显存绰绰有余。**

---

### 2.3 NAS-1：64GB 内存 — 向量数据库节点

```
Qdrant 内存占用（BGE-M3 向量，1024维，FP32）：
  10 万文档  → ~600 MB（纯内存模式）
  100 万文档 → ~6 GB（纯内存模式）
  1000 万文档→ ~60 GB（量化后 ~15 GB）

64GB 内存可支撑规模：
  纯内存模式：~1000 万向量
  INT8 量化模式：~4000 万向量
  mmap 磁盘模式：亿级向量（NAS HDD/SSD 作存储）

其他服务内存占用：
  Redis 缓存：~2-4 GB（缓存热门 embedding 结果）
  Qdrant 进程本身：~100 MB
  系统 OS：~2 GB
  可用于 Qdrant：~55-60 GB
```

**重要限制：Qdrant 不支持 NFS！** 向量数据必须存储在 NAS 本地磁盘（SATA SSD/NVMe/HDD均可），不能挂载网络文件系统。

---

### 2.4 NAS-2：64GB 内存 — 网关 + 监控 + CPU 推理备用

```
主要服务内存占用：
  Nginx + OpenResty（API 网关）：~256 MB
  Prometheus（指标采集）：~512 MB
  Grafana（可视化）：~256 MB
  系统 OS：~2 GB

可选 CPU 备用推理（llama.cpp）：
  Qwen2.5-14B Q4_K_M 模型文件：~8.5 GB 加载到内存
  KV Cache 运行时：~2-4 GB
  合计：~12-13 GB（占 64GB 的 20%）
  速度：取决于 NAS CPU，预计 2-8 tokens/s（适合离线任务）

⚠️ NAS CPU 推理注意事项：
  - 需要 x86 架构且支持 AVX2 指令集
  - 常见 NAS CPU（Celeron J4125/N5105/N100）支持 AVX2 ✅
  - ARM 架构 NAS（如群晖 DS923+）也可运行 llama.cpp ARM 版
  - 速度：Celeron N5105 ≈ ~3-5 tokens/s（仅供批处理/备用）
```

---

## 三、完整部署架构

### 3.1 网络拓扑

```
                        ┌─────────────────────────────┐
                        │          用户/客户端           │
                        └──────────────┬──────────────┘
                                       │ HTTP/HTTPS
                        ┌──────────────▼──────────────┐
                        │  NAS-2：API 网关 (Nginx)      │
                        │  Port 443/80                 │
                        │  • 限流（100 req/s）          │
                        │  • 路由分发                   │
                        │  • SSL 终止                  │
                        └──┬──────────────┬────────────┘
                           │              │
             ┌─────────────▼──┐    ┌──────▼──────────────┐
             │  PC-B：嵌入服务  │    │  PC-A：LLM 推理服务   │
             │  RTX 4060 8GB  │    │  RTX 5070 12GB      │
             │  Port 8080      │    │  Port 8000           │
             │  • BGE-M3 TEI  │    │  • Qwen2.5-14B INT4 │
             │  • Reranker TEI │    │  • llama.cpp/Ollama │
             └────────────────┘    └─────────────────────┘
                                       ▲
                        ┌──────────────┼──────────────┐
                        │  NAS-1：数据服务              │
                        │  64 GB RAM                  │
                        │  • Qdrant :6333             │
                        │  • Redis  :6379             │
                        └─────────────────────────────┘

                        ┌─────────────────────────────┐
                        │  NAS-2：监控服务              │
                        │  • Prometheus :9090         │
                        │  • Grafana    :3000         │
                        │  • CPU LLM 备用（可选）        │
                        └─────────────────────────────┘
```

---

### 3.2 各节点部署清单

#### PC-A（RTX 5070）：LLM 服务

**方案 A：Ollama（最简单，推荐新手）**
```bash
# 安装 Ollama
curl https://ollama.ai/install.sh | sh

# 拉取 Qwen2.5-14B INT4
ollama pull qwen2.5:14b-instruct-q4_K_M

# 启动服务（监听所有网卡，允许局域网访问）
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

**方案 B：llama.cpp（性能更高，推荐进阶）**
```bash
# 编译（CUDA 加速版）
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j

# 下载模型
# Qwen2.5-14B-Instruct-Q4_K_M.gguf (~8.5GB)

# 启动 HTTP 服务
./build/bin/llama-server \
  -m ./models/Qwen2.5-14B-Instruct-Q4_K_M.gguf \
  --host 0.0.0.0 \
  --port 8000 \
  --ctx-size 8192 \
  --n-gpu-layers 999 \   # 全部层放 GPU
  --parallel 6 \         # 最大 6 路并发（12G 显存限制）
  --batch-size 512
```

**PC-A 显存占用预估：**
```
Q4_K_M 权重：8.5 GB
KV Cache（6并发×2K）：6 × 320MB = 1.9 GB
合计：~10.4 GB / 12 GB ✅
```

---

#### PC-B（RTX 4060）：Embedding + Reranker 服务

```bash
# 方式一：HuggingFace TEI（推荐）
# Embedding 服务 Port 8080
docker run -d --gpus all \
  --name bge-m3-embed \
  -p 8080:80 \
  -v ~/.cache/huggingface:/data \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-m3 \
  --dtype float16 \
  --max-concurrent-requests 256

# Reranker 服务 Port 8081
docker run -d --gpus '"device=0"' \
  --name bge-reranker \
  -p 8081:80 \
  -v ~/.cache/huggingface:/data \
  ghcr.io/huggingface/text-embeddings-inference:latest \
  --model-id BAAI/bge-reranker-v2-m3 \
  --dtype float16 \
  --max-concurrent-requests 64
```

**PC-B 显存占用预估：**
```
BGE-M3 FP16：       ~1.1 GB
Reranker FP16：     ~1.1 GB
批处理激活：         ~2.0 GB
合计：              ~4.2 GB / 8 GB ✅（富余 3.8 GB）
```

---

#### NAS-1（64GB RAM）：向量数据库 + 缓存

```bash
# Qdrant 向量数据库
# ⚠️ 必须挂载本地磁盘，不能用 NFS
docker run -d \
  --name qdrant \
  -p 6333:6333 \
  -p 6334:6334 \
  -v /volume1/qdrant/data:/qdrant/storage \
  -e QDRANT__SERVICE__GRPC_PORT=6334 \
  qdrant/qdrant

# Redis 缓存（缓存 embedding 结果，避免重复计算）
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v /volume1/redis/data:/data \
  redis:7-alpine \
  redis-server --appendonly yes --maxmemory 4gb --maxmemory-policy allkeys-lru
```

**NAS-1 内存占用预估：**
```
Qdrant（100万向量）：~6-8 GB
Redis 缓存：        ~4 GB
OS + 其他：         ~4 GB
可用余量：          ~48 GB（可扩展到 1000 万向量）
```

---

#### NAS-2（64GB RAM）：网关 + 监控

```bash
# docker-compose.yml
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

**Nginx 路由配置示例（nginx.conf）：**
```nginx
upstream llm_backend {
    server PC-A-IP:11434;        # Ollama 或 llama.cpp
}
upstream embed_backend {
    server PC-B-IP:8080;         # BGE-M3
}
upstream rerank_backend {
    server PC-B-IP:8081;         # Reranker
}
upstream qdrant_backend {
    server NAS-1-IP:6333;        # Qdrant
}

server {
    listen 80;
    location /api/chat    { proxy_pass http://llm_backend; }
    location /api/embed   { proxy_pass http://embed_backend; }
    location /api/rerank  { proxy_pass http://rerank_backend; }
    location /api/search  { proxy_pass http://qdrant_backend; }
}
```

---

## 四、纯内存部署（无 GPU）方案

> 适用场景：仅用 NAS，不开 PC，低功耗长期运行

| 服务 | 方案 | NAS 内存占用 | 速度 |
|------|------|------------|------|
| BGE-M3 | CPU ONNX 量化 | ~500 MB | ~50-200 QPS（短文本）|
| Reranker | CPU FP32 | ~1.1 GB | ~2-5 req/s |
| Qdrant | mmap 模式 | ~500 MB（活跃数据）| 正常 |
| Qwen2.5-14B | llama.cpp Q4_K_M | ~8.5 GB | **~3-5 tokens/s** ⚠️ 极慢 |
| Redis | 标准 | ~1 GB | 正常 |
| 合计 | — | **~12 GB / 64 GB** | LLM 非常慢 |

```bash
# CPU 模式下 llama.cpp 启动（NAS-1 或 NAS-2）
./llama-server \
  -m ./Qwen2.5-14B-Instruct-Q4_K_M.gguf \
  --host 0.0.0.0 \
  --port 8000 \
  --ctx-size 4096 \
  --n-gpu-layers 0 \   # 全 CPU 模式
  --threads 8 \        # NAS CPU 核心数
  --parallel 1         # 低并发，避免 RAM 不足
```

**⚠️ 纯 CPU 模式 LLM 推理速度极慢（3-5 tokens/s），仅适合离线批处理任务。**

---

## 五、各场景并发能力汇总

### GPU 模式（推荐）

| 组件 | 节点 | 并发能力 | 瓶颈 |
|------|------|---------|------|
| Qwen2.5-14B INT4 | PC-A RTX 5070 | **4-8 并发**（2K上下文）| 12GB 显存 KV Cache |
| BGE-M3 Embedding | PC-B RTX 4060 | **300-600 QPS** | GPU 算力 |
| Reranker | PC-B RTX 4060 | **15-25 req/s** | 与 Embedding 共享显存 |
| Qdrant | NAS-1 64GB | **500+ QPS**（HNSW）| CPU 搜索核心数 |

### 系统整体 RAG 链路并发

```
瓶颈：LLM 生成（4-8 并发）

完整 RAG 请求处理时间：
  BGE-M3 编码：      ~5-15 ms
  Qdrant 检索：      ~10-50 ms
  Reranker 精排：    ~40-100 ms（20篇候选）
  Qwen 生成（512 output token）：~25-30s（20 tokens/s，单请求）

端到端延迟估算：
  最快（短答案 100 tokens）：~5s
  典型（300 tokens）：        ~15-20s
  慢（500 tokens）：          ~25-30s
```

---

## 六、资源消耗总表（GPU 模式）

| 节点 | 显存 | 内存 | 主要服务 |
|------|------|------|---------|
| PC-A RTX 5070 | **10.4 / 12 GB** | ~8 GB | Qwen2.5-14B INT4 |
| PC-B RTX 4060 | **4.2 / 8 GB** | ~6 GB | BGE-M3 + Reranker |
| NAS-1 | 无 | **~12 / 64 GB** | Qdrant + Redis |
| NAS-2 | 无 | **~2 / 64 GB** | Nginx + 监控 |

**最低启动资源（仅必需服务）：**
```
PC-A：RTX 5070 12GB → Qwen INT4（必须）
PC-B 或 NAS：BGE-M3 + Reranker（GPU 推荐，CPU 可用）
NAS-1：Qdrant（仅需 ~500 MB 内存起步）
NAS-2：Nginx（~256 MB，可与 NAS-1 合并）

最低必需显存：~10 GB（PC-A 独跑 LLM）
最低必需内存：~16 GB（NAS 各服务合并）
```

---

## 七、启动顺序 Checklist

```
第一步：NAS-1
  [ ] docker run qdrant（等待健康检查通过）
  [ ] docker run redis
  [ ] 验证：curl http://NAS-1-IP:6333/health

第二步：PC-B
  [ ] docker run bge-m3-embed（等待模型加载，约30s）
  [ ] docker run bge-reranker（等待模型加载，约20s）
  [ ] 验证：curl http://PC-B-IP:8080/health

第三步：PC-A
  [ ] ollama serve（或 llama-server）
  [ ] 等待 Qwen2.5-14B 模型加载（约 60-90s）
  [ ] 验证：curl http://PC-A-IP:11434/api/tags

第四步：NAS-2
  [ ] docker-compose up（nginx + prometheus + grafana）
  [ ] 检查路由配置
  [ ] 访问 http://NAS-2-IP:3000 验证 Grafana

第五步：端到端测试
  [ ] 发送测试 embedding 请求
  [ ] 测试向量写入 Qdrant
  [ ] 测试完整 RAG 查询链路
```

---

## 参考资料

- [LocalScore RTX 5070 LLM 基准](https://www.localscore.ai/accelerator/168)
- [RTX 5070 LLM 性能评测](https://www.techreviewer.com/tech-specs/nvidia-rtx-5070-gpu-for-llms/)
- [Qdrant 内存消耗分析](https://qdrant.tech/articles/memory-consumption/)
- [Qdrant 容量规划](https://qdrant.tech/documentation/guides/capacity-planning/)
- [llama.cpp 官方仓库](https://github.com/ggml-org/llama.cpp)
- [HuggingFace TEI](https://github.com/huggingface/text-embeddings-inference)
- [Qwen2.5 llama.cpp 部署指南](https://qwen.readthedocs.io/en/latest/run_locally/llama.cpp.html)
- 本项目相关文档：
  - [RAG 组件选型指南](./rag-components-selection-guide.md)
  - [BGE-M3 + Qwen2.5-14B 完整部署指南](./rag-stack-deployment-guide.md)
