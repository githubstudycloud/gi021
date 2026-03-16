# NAS 全天候 RAG 部署方案（64GB 升级版）
## 绿联 DXP4800 Plus + 极空间 Z423 标准版
## 各 64GB 内存 + 2TB SSD + 8TB HDD

> 版本：2.0 | 日期：2026-03-16
> 升级亮点：64GB 内存彻底解放，可同时常驻多模型，本地 14B 模型离线可用

---

## 一、硬件规格（最终配置）

| 设备 | CPU | 内存 | SSD | HDD | 网口 |
|------|-----|------|-----|-----|------|
| **极空间 Z423 标准版** | R5 5625U（6核12线程，Zen3，AVX2）| **64GB** DDR4 3200 双通道 | 2TB NVMe | 8TB | 双 2.5GbE |
| **绿联 DXP4800 Plus** | Intel 8505（5核6线程，AVX2）| **64GB** DDR5 4800 双通道 | 2TB NVMe | 8TB | 10GbE + 2.5GbE |

### 内存带宽（推理关键指标）

```
极空间 Z423：DDR4 3200 双通道 → ~51 GB/s 内存带宽（推理性能核心）
绿联 4800 Plus：DDR5 4800 双通道 → ~76 GB/s 内存带宽（Intel 8505 双通道）

推理速度理论上限（内存带宽 ÷ 模型每 token 内存读取量）：
  Qwen2.5-3B  Q4 (~2GB)   极空间：~25 tok/s  绿联：~38 tok/s
  Qwen2.5-7B  Q4 (~4.5GB) 极空间：~11 tok/s  绿联：~17 tok/s
  Qwen2.5-14B Q4 (~8.5GB) 极空间：~6  tok/s  绿联：~9  tok/s
```

---

## 二、64GB 内存后的能力跃升

| 能力 | 8-16GB 时 | **64GB 后** |
|------|-----------|------------|
| 本地 LLM | 最大 7B Q4 | **14B Q4 可用，且可双模型常驻** |
| Qdrant 向量规模 | <100 万（内存模式）| **>1000 万（内存模式），亿级（mmap）** |
| Redis 缓存 | 512MB | **8–16GB，embedding 大规模缓存** |
| 同时运行服务数 | 受限 | **全套服务 + WebUI + 余量充足** |
| Reranker 版本 | bge-reranker-large（小）| **bge-reranker-v2-m3（最强）** |

---

## 三、服务分配方案（重新规划）

### 3.1 总体架构

```
                     ┌────────────────────────────────────────┐
                     │               互联网 / 云端 API           │
                     │  阿里云 DashScope / OpenAI / 月之暗面 等   │
                     └──────────────────┬─────────────────────┘
                                        │（正常时主用）
                     ┌──────────────────▼─────────────────────┐
                     │      绿联 DXP4800 Plus（万兆，数据+网关）   │
                     │  Nginx 智能路由 :80/:443                 │
                     │  Qdrant 向量库  :6333（SSD热+HDD冷）      │
                     │  Redis 缓存    :6379（16GB 大缓存）        │
                     │  Open WebUI   :3001（可选聊天界面）        │
                     │  监控面板      :3000                      │
                     └───────────────┬────────────────────────┘
                                     │ 局域网
                     ┌───────────────▼────────────────────────┐
                     │      极空间 Z423（推理引擎，R5 5625U）      │
                     │  Ollama        :11434                   │
                     │   ├ Qwen2.5-7B  Q4（快速响应）            │
                     │   └ Qwen2.5-14B Q4（高质量，可选常驻）      │
                     │  BGE-M3 TEI    :8080（Embedding）        │
                     │  Reranker TEI  :8081（bge-reranker-v2-m3）│
                     └────────────────────────────────────────┘
```

### 3.2 服务分配清单

#### 极空间 Z423（推理节点，64GB DDR4）

| 服务 | 镜像 | 内存占用 | 端口 | 说明 |
|------|------|---------|------|------|
| Ollama（多模型）| `ollama/ollama` | **18–22 GB** | 11434 | 同时常驻 7B+14B |
| BGE-M3 Embedding | `text-embeddings-inference:cpu` | **2 GB** | 8080 | 全精度 FP32 |
| bge-reranker-v2-m3 | `text-embeddings-inference:cpu` | **2 GB** | 8081 | 最强 Reranker |
| **小计** | — | **~25 GB / 64 GB** | — | **余量 39 GB** |

#### 绿联 DXP4800 Plus（数据节点，64GB DDR5）

| 服务 | 镜像 | 内存占用 | 端口 | 说明 |
|------|------|---------|------|------|
| Qdrant | `qdrant/qdrant` | **8–20 GB** | 6333/6334 | 千万级向量内存模式 |
| Redis | `redis:7-alpine` | **16 GB** | 6379 | 大型 embedding 缓存 |
| Nginx 网关 | `nginx:alpine` | **256 MB** | 80/443 | 智能路由 |
| Open WebUI | `ghcr.io/open-webui/open-webui` | **1 GB** | 3001 | 聊天界面（可选）|
| Prometheus | `prom/prometheus` | **512 MB** | 9090 | 指标采集 |
| Grafana | `grafana/grafana` | **512 MB** | 3000 | 可视化 |
| **小计** | — | **~27 GB / 64 GB** | — | **余量 37 GB** |

---

## 四、本地小模型策略（64GB 充裕方案）

### 4.1 Ollama 多模型常驻

```
64GB RAM → 可同时加载并随时切换，无需重新加载

常驻配置（推荐）：
  ┌─────────────────────────────────────────────────────┐
  │  模型 1：Qwen2.5-7B-Instruct Q4_K_M                  │
  │    文件大小：~4.7 GB                                  │
  │    内存占用：~6 GB（含 KV Cache）                      │
  │    速度：~8–12 tok/s                                  │
  │    用途：快速问答，轻量 RAG，交互式使用                   │
  ├─────────────────────────────────────────────────────┤
  │  模型 2：Qwen2.5-14B-Instruct Q4_K_M                 │
  │    文件大小：~8.7 GB                                  │
  │    内存占用：~11 GB（含 KV Cache）                     │
  │    速度：~5–7 tok/s                                   │
  │    用途：高质量输出，复杂推理，文档分析                   │
  └─────────────────────────────────────────────────────┘
  合计内存：~17 GB / 64 GB  ✅ 绰绰有余
```

### 4.2 模型速度对比（R5 5625U，实测估算）

| 模型 | 生成速度 | 首 token 延迟 | 适合场景 |
|------|---------|-------------|---------|
| Qwen2.5-3B Q4_K_M | ~18–25 tok/s | ~0.5s | 不推荐，14B可用后无意义 |
| **Qwen2.5-7B Q4_K_M** | **~8–12 tok/s** | **~1s** | 日常 RAG 主力 ★ |
| **Qwen2.5-14B Q4_K_M** | **~5–7 tok/s** | **~2s** | 复杂任务，质量优先 ★ |
| Qwen2.5-14B Q2_K | ~8–10 tok/s | ~1.5s | 速度与质量折衷 |

---

## 五、存储布局规划（2TB SSD + 8TB HDD）

### 5.1 SSD（2TB NVMe，高速读写）

```
极空间 Z423 SSD（2TB）：
├── /ssd/models/           ~50 GB   AI 模型文件（GGUF/SafeTensors）
│   ├── Qwen2.5-7B-Q4_K_M.gguf     ~4.7 GB
│   ├── Qwen2.5-14B-Q4_K_M.gguf    ~8.7 GB
│   ├── bge-m3/                    ~1.1 GB
│   └── bge-reranker-v2-m3/        ~1.1 GB
├── /ssd/tei_cache/         ~5 GB   TEI 模型缓存
├── /ssd/ollama_data/       ~30 GB  Ollama 数据目录
└── /ssd/swap/              ~16 GB  交换分区（防 OOM 备用）

绿联 DXP4800 Plus SSD（2TB）：
├── /ssd/qdrant/hot/       ~200 GB  Qdrant 热数据（HNSW 索引）
├── /ssd/redis/             ~50 GB  Redis AOF 持久化
├── /ssd/docker/            ~30 GB  Docker 镜像和容器层
└── /ssd/prometheus/        ~20 GB  监控历史数据
```

### 5.2 HDD（8TB 机械，大容量冷存储）

```
极空间 Z423 HDD（8TB）：
├── /hdd/documents/         TB 级   原始文档（PDF/Word/TXT 等）
├── /hdd/model_backup/     ~50 GB  模型文件备份
└── /hdd/data_backup/       TB 级   数据备份

绿联 DXP4800 Plus HDD（8TB）：
├── /hdd/qdrant/cold/       TB 级   Qdrant 冷数据（mmap 模式向量文件）
├── /hdd/backups/           TB 级   全系统备份
└── /hdd/archive/           TB 级   历史文档归档
```

### 5.3 Qdrant SSD+HDD 混合存储（关键配置）

```
策略：索引（HNSW 图）放 SSD → 搜索速度快
      向量数据放 HDD（mmap）→ 容量超大，几乎无限

可存向量规模：
  SSD 200GB + HDD 8TB → 支持 ~20 亿向量（1024 维）
  内存仅需 8–16 GB（HNSW 索引常驻内存）
```

---

## 六、Docker Compose 配置

### 6.1 极空间 Z423 — docker-compose.yml

```yaml
version: '3.8'

services:
  # ── Ollama（多模型并行常驻）────────────────────────────────
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - /ssd/ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_NUM_PARALLEL=4          # 4 路并发推理
      - OLLAMA_MAX_LOADED_MODELS=2     # 同时保持 7B + 14B 常驻
      - OLLAMA_KEEP_ALIVE=24h          # 模型常驻不卸载
    cpus: '10'                         # 留 2 核给其他服务
    mem_limit: 30g
    mem_reservation: 18g

  # ── BGE-M3 Embedding ─────────────────────────────────────
  bge-m3:
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-latest
    container_name: bge-m3
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - /ssd/tei_cache:/data
    command:
      - --model-id=BAAI/bge-m3
      - --dtype=float32
      - --max-concurrent-requests=64
      - --max-batch-tokens=32768
      - --max-batch-requests=32
    cpus: '4'
    mem_limit: 3g
    mem_reservation: 2g

  # ── BGE-Reranker-v2-M3（最强 Reranker）──────────────────
  bge-reranker:
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-latest
    container_name: bge-reranker
    restart: unless-stopped
    ports:
      - "8081:80"
    volumes:
      - /ssd/tei_cache:/data
    command:
      - --model-id=BAAI/bge-reranker-v2-m3
      - --dtype=float32
      - --max-concurrent-requests=32
      - --max-batch-tokens=16384
    cpus: '4'
    mem_limit: 3g
    mem_reservation: 2g

networks:
  default:
    driver: bridge
```

**初始化（首次运行）：**
```bash
# 拉取两个本地模型（需联网，各 5-9 GB，约 30 分钟）
docker exec -it ollama ollama pull qwen2.5:7b-instruct-q4_K_M
docker exec -it ollama ollama pull qwen2.5:14b-instruct-q4_K_M

# 验证两个模型都已加载
docker exec -it ollama ollama list
```

---

### 6.2 绿联 DXP4800 Plus — docker-compose.yml

```yaml
version: '3.8'

services:
  # ── Qdrant（SSD热数据 + HDD冷数据 混合）───────────────────
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    restart: unless-stopped
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - /ssd/qdrant/hot:/qdrant/storage    # 热数据走 SSD
      - /hdd/qdrant/cold:/qdrant/snapshots # 快照走 HDD
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
      - QDRANT__STORAGE__ON_DISK_PAYLOAD=true  # payload 存磁盘
      - QDRANT__STORAGE__MEMMAP_THRESHOLD_KB=1024  # >1MB 的向量用 mmap
    mem_limit: 24g
    mem_reservation: 8g

  # ── Redis（16GB 大缓存，embedding 结果持久化）──────────────
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - /ssd/redis:/data
    command:
      - redis-server
      - --appendonly yes
      - --maxmemory 16gb
      - --maxmemory-policy allkeys-lru
      - --save 900 1
      - --save 300 10
    mem_limit: 18g
    mem_reservation: 8g

  # ── Nginx（智能路由，云端/本地自动切换）─────────────────────
  nginx:
    image: nginx:alpine
    container_name: nginx-gateway
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "8000:8000"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    mem_limit: 512m

  # ── Open WebUI（网页聊天界面，支持 Ollama + API）────────────
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3001:8080"
    volumes:
      - ./open_webui_data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://极空间Z423-IP:11434
      - OPENAI_API_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
      - OPENAI_API_KEY=${DASHSCOPE_API_KEY}
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
    mem_limit: 2g

  # ── Prometheus ────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /ssd/prometheus:/prometheus
    mem_limit: 1g

  # ── Grafana ───────────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
    mem_limit: 1g

networks:
  default:
    driver: bridge
```

---

### 6.3 Nginx 智能路由（在线/离线自动切换）

```nginx
# ./nginx/nginx.conf

worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
}

http {
    # ── 连接超时：判断云端是否可达 ────────────────────────────
    proxy_connect_timeout   3s;
    proxy_read_timeout      120s;
    proxy_send_timeout      30s;

    # ── 上游定义 ─────────────────────────────────────────────

    # 云端主 LLM（阿里云 DashScope，国内低延迟）
    upstream cloud_llm_primary {
        server dashscope.aliyuncs.com:443;
        keepalive 32;
    }

    # 本地备用 LLM（极空间 Z423，断网时使用）
    upstream local_llm_fallback {
        server 极空间Z423-IP:11434;
        keepalive 16;
    }

    # Embedding 服务（极空间）
    upstream embed_svc { server 极空间Z423-IP:8080; }

    # Reranker 服务（极空间）
    upstream rerank_svc { server 极空间Z423-IP:8081; }

    # Qdrant（本机）
    upstream qdrant_svc { server localhost:6333; }

    # ── 主 API 服务器 ─────────────────────────────────────────
    server {
        listen 8000;
        server_name _;

        # LLM 对话接口：优先云端，失败降级本地
        location /v1/chat/completions {
            proxy_pass https://cloud_llm_primary/compatible-mode/v1/chat/completions;
            proxy_ssl_server_name on;
            proxy_set_header Host dashscope.aliyuncs.com;
            proxy_set_header Authorization "Bearer ${DASHSCOPE_API_KEY}";
            proxy_intercept_errors on;
            error_page 502 503 504 = @local_llm;
        }

        location @local_llm {
            # 降级到本地 14B（更高质量）或 7B（更快）
            proxy_pass http://local_llm_fallback/api/chat;
            proxy_read_timeout 600s;   # 本地推理慢，超时设长
            add_header X-LLM-Source "local" always;
        }

        # Embedding 接口
        location /v1/embeddings {
            proxy_pass http://embed_svc;
            proxy_read_timeout 60s;
        }

        # Reranker 接口
        location /v1/rerank {
            proxy_pass http://rerank_svc;
            proxy_read_timeout 60s;
        }

        # Qdrant REST 接口
        location /qdrant/ {
            rewrite ^/qdrant/(.*) /$1 break;
            proxy_pass http://qdrant_svc;
        }

        location /health {
            return 200 '{"status":"ok","mode":"gateway"}';
            add_header Content-Type application/json;
        }
    }
}
```

---

## 七、内存使用全景（64GB × 2）

### 极空间 Z423（64GB DDR4，推理节点）

| 服务 | 空闲占用 | 满负载占用 |
|------|---------|---------|
| 系统 OS | 2 GB | 3 GB |
| Ollama + Qwen2.5-7B Q4 | 6 GB | 8 GB |
| Ollama + Qwen2.5-14B Q4 | 11 GB | 13 GB |
| BGE-M3 TEI（CPU FP32）| 2 GB | 3 GB |
| bge-reranker-v2-m3 TEI | 2 GB | 2.5 GB |
| **合计** | **~23 GB** | **~29.5 GB** |
| **剩余** | **41 GB ✅** | **34.5 GB ✅** |

> **剩余 34-41GB**：可额外运行文档处理服务、OCR、更多模型，或作为系统缓冲。

### 绿联 DXP4800 Plus（64GB DDR5，数据节点）

| 服务 | 空闲占用 | 满负载占用 |
|------|---------|---------|
| 系统 OS | 2 GB | 2.5 GB |
| Qdrant（1000万向量）| 8 GB | 20 GB |
| Redis（大缓存）| 8 GB | 16 GB |
| Open WebUI | 0.5 GB | 1 GB |
| Prometheus + Grafana | 0.8 GB | 1.5 GB |
| Nginx | 0.1 GB | 0.2 GB |
| **合计** | **~19.4 GB** | **~41.2 GB** |
| **剩余** | **44.6 GB ✅** | **22.8 GB ✅** |

---

## 八、Qdrant 向量容量规划（SSD+HDD 混合）

```
策略：HNSW 索引常驻内存，向量数据 mmap 到 SSD/HDD

内存 8GB（Qdrant 配额）可支撑：
  ├── 纯内存模式：~1200 万向量（1024 维）
  └── mmap 模式（SSD/HDD 存向量）：
      SSD 200GB  → ~2.4 亿向量
      HDD 4TB    → ~50 亿向量

实际业务规划：
  10 万文档，每段 ~500 字，每段 1 个向量：
    文档总向量数：~10-30 万 → 内存 ~2-6GB → 纯内存模式完全可用

  100 万文档：
    向量数：~300-500 万 → 内存 ~6-10GB → 仍可全内存

  1000 万文档：
    向量数：~3000-5000 万 → 建议 mmap 模式，SSD 存储
```

---

## 九、LLM API 配置（主用 + 断网降级）

```python
# rag_llm_client.py

import os, time
from openai import OpenAI

# ── 主用：阿里云 DashScope（国内延迟低，中文最强）
PRIMARY = OpenAI(
    api_key=os.environ["DASHSCOPE_API_KEY"],
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    timeout=8.0,
)

# ── 备用：本地 14B（断网时自动切换）
FALLBACK = OpenAI(
    api_key="ollama",
    base_url="http://极空间Z423-IP:11434/v1",
    timeout=600.0,
)

CLOUD_MODEL  = "qwen-plus"                        # 或 qwen-max / qwen-turbo
LOCAL_FAST   = "qwen2.5:7b-instruct-q4_K_M"      # 快速响应
LOCAL_QUALITY= "qwen2.5:14b-instruct-q4_K_M"     # 高质量

def chat(messages: list, quality: str = "fast", **kwargs) -> str:
    """
    quality: "fast" → 优先云端，断网用 7B
             "high" → 优先云端，断网用 14B
    """
    fallback_model = LOCAL_QUALITY if quality == "high" else LOCAL_FAST
    try:
        resp = PRIMARY.chat.completions.create(
            model=CLOUD_MODEL, messages=messages, **kwargs
        )
        return resp.choices[0].message.content
    except Exception as e:
        print(f"[LLM] 云端不可达({e})，切换本地 {fallback_model}")
        resp = FALLBACK.chat.completions.create(
            model=fallback_model, messages=messages, **kwargs
        )
        return resp.choices[0].message.content
```

---

## 十、启动顺序与验证

```bash
# ════════════════════════════════════════
# 第一步：绿联 DXP4800 Plus（数据先行）
# ════════════════════════════════════════
cd /绿联数据目录/rag-data
docker-compose up -d qdrant redis
sleep 10
docker-compose up -d nginx open-webui prometheus grafana

# 验证
curl http://localhost:6333/health          # Qdrant
redis-cli -p 6379 ping                    # Redis → PONG
curl http://localhost:8000/health         # Nginx 网关

# ════════════════════════════════════════
# 第二步：极空间 Z423（推理服务）
# ════════════════════════════════════════
cd /极空间数据目录/rag-inference
docker-compose up -d bge-m3 bge-reranker  # 等模型下载 ~15min
docker-compose up -d ollama

# 拉取本地模型（首次）
docker exec -it ollama ollama pull qwen2.5:7b-instruct-q4_K_M
docker exec -it ollama ollama pull qwen2.5:14b-instruct-q4_K_M

# ════════════════════════════════════════
# 第三步：端到端验证
# ════════════════════════════════════════
# Embedding 测试
curl -X POST http://极空间IP:8080/embed \
  -H "Content-Type: application/json" \
  -d '{"inputs": "hello world"}'

# Reranker 测试
curl -X POST http://极空间IP:8081/rerank \
  -H "Content-Type: application/json" \
  -d '{"query":"NAS部署","texts":["本地部署方案","云端API接入"]}'

# 本地 LLM 测试
curl http://极空间IP:11434/api/tags

# 云端 API 测试
curl -X POST http://绿联IP:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-plus","messages":[{"role":"user","content":"你好"}]}'
```

---

## 十一、各场景并发能力

| 场景 | 并发能力 | 说明 |
|------|---------|------|
| **云端 LLM（正常联网）** | **无限制**（受 API 配额）| DashScope/OpenAI 限速 |
| **本地 7B 离线**（R5 5625U）| ~2–3 并发 | 速度 ~8–12 tok/s |
| **本地 14B 离线**（R5 5625U）| ~1–2 并发 | 速度 ~5–7 tok/s |
| **BGE-M3 Embedding** | ~30–50 QPS | CPU FP32，batch=16 |
| **Reranker-v2-M3** | ~5–10 req/s | 每次 20 候选，seq=512 |
| **Qdrant 向量检索** | **200–500 QPS** | 内存模式 HNSW |

---

## 十二、功耗与运行成本

| 设备 | 待机功耗 | AI 推理功耗 | 月电费（0.6元/度，24h）|
|------|---------|-----------|----------------------|
| 极空间 Z423（待机）| ~20W | ~50W（R5 高负载）| ~8.6–21.6 元 |
| 绿联 DXP4800 Plus（待机）| ~18W | ~30W（数据服务）| ~7.8–13 元 |
| **两台合计（日常混合）** | **~40W** | **~80W** | **~17–35 元/月** |

---

## 十三、升级建议

当前配置已非常充裕，未来可选升级方向：

| 升级项 | 效果 | 成本估算 |
|--------|------|---------|
| 绿联加装 M.2 NVMe SSD | Qdrant 索引更快 | ¥200–500 |
| 接入 PC（RTX 5070）时 | 14B 提速 3–4 倍（GPU推理）| 按需开机 |
| 接入 PC（RTX 4060）时 | BGE-M3 GPU 提速 10 倍 | 按需开机 |
| 添加 Dify 平台 | 可视化 RAG 流程编排 | 免费 Docker 部署 |

---

## 参考资料

- [极空间 Z423 标准版详测](https://post.smzdm.com/p/admvgook/)
- [绿联 DXP4800 Plus 官方页](https://www.ugnas.com/products-detail/id-39.html)
- [Qdrant 容量规划](https://qdrant.tech/documentation/guides/capacity-planning/)
- [Ollama 多模型配置](https://github.com/ollama/ollama)
- [Open WebUI](https://github.com/open-webui/open-webui)
- 本项目配套文档：
  - [RAG 组件选型](./rag-components-selection-guide.md)
  - [完整 GPU 部署方案](./home-lab-rag-deployment.md)
  - [原 NAS 部署方案（旧版）](./nas-only-rag-deployment.md)
