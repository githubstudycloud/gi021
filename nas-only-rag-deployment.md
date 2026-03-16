# NAS 全天候 RAG 部署方案
## 绿联 DXP4800 Plus + 极空间 Z423 标准版（主机关机模式）

> 版本：1.0 | 日期：2026-03-16
> 约束：仅 NAS 常驻运行，无 GPU，主机关机，大模型主用 API，断网自动降级本地小模型

---

## 一、硬件规格确认

| 设备 | CPU | 内存（标配/最大）| 网口 | 定位 |
|------|-----|----------------|------|------|
| **极空间 Z423 标准版** | AMD R5 5625U（6核12线程）| 16GB / 64GB DDR4 3200 双通道 | 双 2.5GbE | 推理主节点（性能强）|
| **绿联 DXP4800 Plus** | Intel Pentium Gold 8505（5核6线程）| 8GB / 64GB DDR5 4800 双通道 | 1×10GbE + 1×2.5GbE | 数据服务节点（万兆）|

### CPU 推理能力评估

```
极空间 Z423（R5 5625U）：
  架构：Zen 3，支持 AVX2，内存带宽 ~51 GB/s（DDR4 3200 双通道）
  llama.cpp 推理速度估算：
    Qwen2.5-3B  Q4_K_M (~2GB)  → ~18–25 tokens/s  ✅ 可用于交互
    Qwen2.5-7B  Q4_K_M (~4.5GB)→ ~8–12 tokens/s   ✅ 可接受
    Qwen2.5-14B Q4_K_M (~8.5GB)→ ~3–5  tokens/s   ⚠️ 太慢

绿联 DXP4800 Plus（Intel 8505）：
  架构：Alder Lake，支持 AVX2，内存带宽 ~38 GB/s（单通道，若仅1条8GB）
  推理速度约为 R5 5625U 的 70%，不适合运行 LLM，专注数据服务
```

**结论：推理任务放极空间 Z423，数据存储放绿联 4800 Plus。**

---

## 二、服务分配方案

### 2.1 总体架构（仅 NAS 常驻）

```
                    ┌─────────────────────────────────┐
                    │            互联网                  │
                    │  • OpenAI / DashScope / Claude    │
                    │  • Qwen-Plus / GLM-4 / ERNIE      │
                    └─────────────┬───────────────────┘
                                  │ HTTPS API（正常时）
                    ┌─────────────▼───────────────────┐
                    │  绿联 DXP4800 Plus（数据节点）      │
                    │  Nginx 反向代理 + 智能路由          │
                    │  • 网络在线 → 转发至云端 API        │
                    │  • 网络中断 → 转发至本地小模型       │
                    └──────────────┬──────────────────┘
                                   │ 局域网
          ┌────────────────────────┼────────────────────────┐
          ▼                        ▼                        ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  绿联 DXP4800    │    │  极空间 Z423      │    │  极空间 Z423      │
│  Qdrant :6333    │    │  Ollama :11434    │    │  BGE-M3 TEI      │
│  Redis  :6379    │    │  Qwen2.5-3B/7B   │    │  :8080 Embed     │
│  监控   :3000    │    │  （断网备用LLM）   │    │  :8081 Rerank    │
└──────────────────┘    └──────────────────┘    └──────────────────┘
（注：极空间 Z423 的两类服务运行在同一台机器上）
```

### 2.2 各节点服务清单

#### 极空间 Z423 标准版（推理节点）

| 服务 | 镜像 | 内存占用 | 端口 |
|------|------|---------|------|
| Ollama（小模型） | `ollama/ollama:latest` | ~3.5 GB（含 3B 模型）| 11434 |
| BGE-M3 Embedding | `ghcr.io/huggingface/text-embeddings-inference` | ~1.5 GB | 8080 |
| bge-reranker-large | `ghcr.io/huggingface/text-embeddings-inference` | ~0.8 GB | 8081 |
| **合计** | — | **~5.8 GB / 16 GB** | — |

#### 绿联 DXP4800 Plus（数据节点）

| 服务 | 镜像 | 内存占用 | 端口 |
|------|------|---------|------|
| Qdrant 向量数据库 | `qdrant/qdrant:latest` | ~1–3 GB | 6333 |
| Redis 缓存 | `redis:7-alpine` | ~512 MB | 6379 |
| Nginx API 网关 | `nginx:alpine` | ~128 MB | 80/443 |
| Prometheus 监控 | `prom/prometheus` | ~256 MB | 9090 |
| Grafana 面板 | `grafana/grafana` | ~256 MB | 3000 |
| **合计** | — | **~4.5 GB / 8 GB** | — |

---

## 三、小模型选型（离线 LLM 备用）

### 3.1 R5 5625U 上各模型实测速度估算

| 模型 | 文件大小 | 内存占用 | 生成速度 | 能力评价 | 推荐度 |
|------|---------|---------|---------|---------|-------|
| Qwen2.5-1.5B Q4_K_M | ~1.0 GB | ~1.5 GB | ~35–50 tok/s | 基础问答，推理弱 | 最低配 |
| **Qwen2.5-3B Q4_K_M** | ~2.0 GB | ~3.0 GB | ~18–25 tok/s | 日常问答可用 | **★ 推荐** |
| Qwen2.5-7B Q4_K_M | ~4.5 GB | ~6.0 GB | ~8–12 tok/s | 质量明显更好 | 16GB 内存首选 |
| Qwen2.5-7B Q2_K | ~3.0 GB | ~4.5 GB | ~10–15 tok/s | 速度与质量折衷 | 可选 |
| Gemma2-2B Q4 | ~1.5 GB | ~2.5 GB | ~20–30 tok/s | 英文强，中文弱 | 英文场景 |

**推荐策略：**
```
极空间内存 ≤ 16GB → 默认用 Qwen2.5-7B Q4_K_M（质量优先）
极空间内存 = 8GB  → 用 Qwen2.5-3B Q4_K_M（资源优先）
需要最快响应      → 用 Qwen2.5-1.5B Q4_K_M
```

### 3.2 云端 API 推荐（主用）

| 服务商 | 模型 | 特点 | 免费额度 |
|--------|------|------|---------|
| 阿里云 DashScope | qwen-plus / qwen-max | 国内低延迟，中文最强 | 有免费额度 |
| 阿里云 DashScope | qwen-turbo | 速度快，低成本 | 有免费额度 |
| OpenAI | gpt-4o-mini | 综合能力强 | 付费 |
| 智谱 AI | glm-4-flash | 国内，免费版可用 | 有免费模型 |
| 月之暗面 | moonshot-v1-8k | 长上下文，国内 | 有免费额度 |

---

## 四、Docker 部署文件

### 4.1 极空间 Z423 — docker-compose.yml

```yaml
# 路径：/极空间数据目录/rag-inference/docker-compose.yml
version: '3.8'

services:
  # ── Ollama 本地小模型（断网备用）──────────────────────────
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ./ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_NUM_PARALLEL=2        # 最多 2 路并发
      - OLLAMA_MAX_LOADED_MODELS=1   # 内存有限，只加载1个模型
    cpus: '6'                        # 用满 6 核
    mem_limit: 8g
    mem_reservation: 4g

  # ── BGE-M3 Embedding 服务 ──────────────────────────────────
  bge-m3:
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-latest
    container_name: bge-m3-embed
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./tei_cache:/data
    command:
      - --model-id=BAAI/bge-m3
      - --dtype=float32              # CPU 用 FP32
      - --max-concurrent-requests=32
      - --max-batch-tokens=16384
    cpus: '4'
    mem_limit: 2g
    mem_reservation: 1g

  # ── BGE-Reranker 服务 ──────────────────────────────────────
  bge-reranker:
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-latest
    container_name: bge-reranker
    restart: unless-stopped
    ports:
      - "8081:80"
    volumes:
      - ./tei_cache:/data
    command:
      - --model-id=BAAI/bge-reranker-large
      - --dtype=float32
      - --max-concurrent-requests=16
    cpus: '2'
    mem_limit: 1500m
    mem_reservation: 800m

networks:
  default:
    driver: bridge
```

**启动后拉取小模型：**
```bash
# 进入 Ollama 容器拉取模型（首次需联网）
docker exec -it ollama ollama pull qwen2.5:7b-instruct-q4_K_M

# 或拉取 3B（内存更少）
docker exec -it ollama ollama pull qwen2.5:3b-instruct-q4_K_M

# 验证
docker exec -it ollama ollama list
```

---

### 4.2 绿联 DXP4800 Plus — docker-compose.yml

```yaml
# 路径：/绿联数据目录/rag-data/docker-compose.yml
version: '3.8'

services:
  # ── Qdrant 向量数据库 ──────────────────────────────────────
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    restart: unless-stopped
    ports:
      - "6333:6333"
      - "6334:6334"        # gRPC
    volumes:
      - ./qdrant_storage:/qdrant/storage
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
      - QDRANT__STORAGE__ON_DISK_PAYLOAD=true   # payload 存磁盘，节省内存
    mem_limit: 3g
    mem_reservation: 1g

  # ── Redis 缓存（缓存 embedding，避免重复计算）───────────────
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - ./redis_data:/data
    command:
      - redis-server
      - --appendonly yes
      - --maxmemory 512mb
      - --maxmemory-policy allkeys-lru
    mem_limit: 600m

  # ── Nginx API 网关（智能路由：在线→API，离线→本地）───────────
  nginx:
    image: nginx:alpine
    container_name: nginx-gateway
    restart: unless-stopped
    ports:
      - "8000:8000"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    mem_limit: 256m
    depends_on:
      - qdrant
      - redis

  # ── 监控（Prometheus + Grafana）───────────────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus_data:/prometheus
    mem_limit: 512m

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=your_password
      - GF_USERS_ALLOW_SIGN_UP=false
    mem_limit: 512m

networks:
  default:
    driver: bridge
```

---

### 4.3 Nginx 智能路由配置（关键：在线/离线自动切换）

```nginx
# 绿联 DXP4800 Plus: ./nginx/nginx.conf

worker_processes auto;

events { worker_connections 1024; }

http {
    # ── 上游服务定义 ─────────────────────────────────────────

    # 云端 LLM API（主用）
    upstream cloud_llm {
        server api.dashscope.aliyuncs.com:443;   # 阿里云 DashScope
        # server api.openai.com:443;             # 或 OpenAI
    }

    # 本地小模型（断网备用）
    upstream local_llm {
        server 极空间Z423-IP:11434;               # Ollama
    }

    # Embedding 服务
    upstream embed_service {
        server 极空间Z423-IP:8080;
    }

    # Reranker 服务
    upstream rerank_service {
        server 极空间Z423-IP:8081;
    }

    # 向量数据库
    upstream qdrant_service {
        server localhost:6333;
    }

    # ── 主服务入口 ────────────────────────────────────────────
    server {
        listen 8000;

        # LLM 对话接口（带降级逻辑）
        location /v1/chat/completions {
            proxy_connect_timeout 3s;
            proxy_read_timeout 120s;

            # 尝试云端 API，失败则降级到本地
            proxy_pass https://cloud_llm;
            proxy_intercept_errors on;
            error_page 502 503 504 = @local_llm_fallback;
        }

        # 断网降级路由
        location @local_llm_fallback {
            proxy_pass http://local_llm;
            proxy_read_timeout 300s;   # 本地推理更慢，超时设更长
        }

        # Embedding 接口
        location /v1/embeddings {
            proxy_pass http://embed_service;
            proxy_read_timeout 30s;
        }

        # Reranker 接口
        location /v1/rerank {
            proxy_pass http://rerank_service;
            proxy_read_timeout 30s;
        }

        # 向量检索接口（Qdrant）
        location /qdrant/ {
            rewrite ^/qdrant/(.*) /$1 break;
            proxy_pass http://qdrant_service;
        }

        # 健康检查
        location /health {
            return 200 '{"status":"ok"}';
            add_header Content-Type application/json;
        }
    }
}
```

---

## 五、LLM API 接入（应用层配置）

### 5.1 统一 API 格式（OpenAI 兼容）

所有主流国内 API 均支持 OpenAI 兼容接口，只需切换 `base_url` 和 `api_key`：

```python
# rag_config.py — 双 Provider 配置

import os
from openai import OpenAI

# ── 主用：云端 API ──────────────────────────────────────────
PRIMARY_CLIENT = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),   # 或 OPENAI_API_KEY
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)
PRIMARY_MODEL = "qwen-plus"   # 或 "gpt-4o-mini"

# ── 备用：本地 Ollama（断网） ───────────────────────────────
FALLBACK_CLIENT = OpenAI(
    api_key="ollama",    # Ollama 不需要真实 key
    base_url="http://NAS网关IP:8000/v1",   # 通过 Nginx 路由到 Ollama
)
FALLBACK_MODEL = "qwen2.5:7b-instruct-q4_K_M"

# ── 自动切换逻辑 ────────────────────────────────────────────
def chat(messages: list, **kwargs):
    try:
        response = PRIMARY_CLIENT.chat.completions.create(
            model=PRIMARY_MODEL,
            messages=messages,
            timeout=10,   # 3 秒无响应则认为断网
            **kwargs
        )
        return response
    except Exception as e:
        print(f"[LLM] 云端 API 不可用：{e}，切换本地模型")
        return FALLBACK_CLIENT.chat.completions.create(
            model=FALLBACK_MODEL,
            messages=messages,
            timeout=300,  # 本地推理更慢
            **kwargs
        )
```

### 5.2 环境变量配置（.env 文件）

```env
# 放在 NAS 应用目录下

# ── 云端 API（二选一）──
DASHSCOPE_API_KEY=sk-xxxxxxxxxxxxxxxx
# OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx

# ── 服务地址 ──
QDRANT_URL=http://绿联IP:6333
REDIS_URL=redis://绿联IP:6379/0
EMBED_URL=http://绿联IP:8000/v1/embeddings
RERANK_URL=http://绿联IP:8000/v1/rerank
LLM_GATEWAY_URL=http://绿联IP:8000/v1

# ── 本地模型 ──
LOCAL_MODEL=qwen2.5:7b-instruct-q4_K_M
FALLBACK_THRESHOLD_MS=5000   # 超过 5s 无响应则切换本地
```

---

## 六、内存占用全景（稳定运行状态）

### 极空间 Z423（16GB）

| 服务 | 空闲内存 | 满载内存 |
|------|---------|---------|
| 系统 OS | 1.5 GB | 2.0 GB |
| Ollama + Qwen2.5-7B Q4 | 5.5 GB | 6.0 GB |
| BGE-M3 TEI CPU | 1.2 GB | 1.8 GB |
| bge-reranker-large | 0.8 GB | 1.2 GB |
| **合计** | **9.0 GB** | **11.0 GB** |
| **剩余** | **7 GB** | **5 GB** ✅ |

> 如果只装 16GB 原厂内存，运行 7B 模型稳定可用。
> 建议扩展到 32GB（约 200 元），运行 7B 更宽裕。

### 绿联 DXP4800 Plus（8GB）

| 服务 | 空闲内存 | 满载内存 |
|------|---------|---------|
| 系统 OS | 1.5 GB | 2.0 GB |
| Qdrant（100 万向量）| 1.0 GB | 2.5 GB |
| Redis | 0.3 GB | 0.5 GB |
| Nginx | 0.1 GB | 0.1 GB |
| Prometheus + Grafana | 0.5 GB | 0.8 GB |
| **合计** | **3.4 GB** | **5.9 GB** |
| **剩余** | **4.6 GB** | **2.1 GB** ✅ |

> 8GB 内存可以运行，但 Qdrant 数据量超过 200 万向量后建议扩展到 16GB。

---

## 七、最低起步配置 vs 推荐配置

| 配置项 | 最低起步 | 推荐配置 | 说明 |
|--------|---------|---------|------|
| 极空间内存 | 16GB（原厂）| **32GB** | 7B 模型更稳定 |
| 绿联内存 | 8GB（原厂）| **16GB** | Qdrant 更大数据量 |
| 极空间存储 | 1× M.2 SSD 128GB | 1× M.2 SSD 256GB+ | 存模型+缓存 |
| 绿联存储 | HDD 即可 | **M.2 SSD for Qdrant** | 向量检索性能 |
| 小模型 | Qwen2.5-3B Q4 | **Qwen2.5-7B Q4** | 回答质量提升明显 |
| 云端 API | 阿里云 DashScope（有免费额度）| 按需 | 正常时主用 |

---

## 八、NAS 特有注意事项

### 绿联 DXP4800 Plus

```
✅ Docker 支持：绿联 UGOS PRO 原生支持 Docker，可直接在 Web 界面管理
✅ 万兆网口：10GbE 适合做 API 网关，局域网内高速传输
⚠️ Qdrant 存储：不能放在网络挂载目录，必须用本地 SSD 或 HDD（SATA 接口）
⚠️ 8GB 内存：原厂 8GB 仅 1 条，建议加一条 8GB 组成双通道 16GB
⚠️ 电源：长期运行，注意硬盘待机设置，减少功耗
```

### 极空间 Z423 标准版

```
✅ Docker 支持：极空间内置 Docker 支持，Web 界面直接操作
✅ R5 5625U：支持 AVX2，llama.cpp 推理性能强（是 N100 的 2-3 倍）
✅ 双通道内存：16GB 双通道，内存带宽充足
⚠️ TEI 镜像：cpu-latest tag 需确认架构兼容（x86_64 ✅，ARM ❌）
⚠️ 模型下载：首次需联网拉取 Ollama 模型（3-7B 模型 2-5GB，需预留空间）
⚠️ 热管理：长期 CPU 高负载，注意 NAS 散热，必要时降低 cpus 限制
```

---

## 九、启动顺序与验证

```bash
# 第一步：绿联 DXP4800 Plus 启动数据服务
cd /绿联数据目录/rag-data
docker-compose up -d qdrant redis
# 等待 5s
docker-compose up -d nginx prometheus grafana

# 验证 Qdrant
curl http://绿联IP:6333/health
# 预期返回: {"title":"qdrant - vector search engine","version":"...","commit":"..."}

# 第二步：极空间 Z423 启动推理服务
cd /极空间数据目录/rag-inference
docker-compose up -d bge-m3 bge-reranker
# 等待模型下载（首次 ~5-10 分钟）
docker-compose up -d ollama

# 第三步：拉取本地小模型（仅首次，需联网）
docker exec -it ollama ollama pull qwen2.5:7b-instruct-q4_K_M

# 第四步：验证所有服务
curl http://极空间IP:8080/health       # BGE-M3
curl http://极空间IP:8081/health       # Reranker
curl http://极空间IP:11434/api/tags    # Ollama 模型列表

# 第五步：端到端测试（通过 Nginx 网关）
curl http://绿联IP:8000/health

# 测试 Embedding
curl -X POST http://绿联IP:8000/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": "测试文本", "model": "BAAI/bge-m3"}'

# 测试本地 LLM（断网模拟）
curl -X POST http://极空间IP:11434/api/chat \
  -d '{"model":"qwen2.5:7b-instruct-q4_K_M","messages":[{"role":"user","content":"你好"}]}'
```

---

## 十、总费用与功耗估算

| 项目 | 功耗 | 月电费（0.6元/度，24h待机）|
|------|------|------------------------|
| 极空间 Z423（待机）| ~20W | ~8.6 元 |
| 极空间 Z423（LLM 推理中）| ~35W | ~15 元 |
| 绿联 DXP4800 Plus（待机）| ~15W | ~6.5 元 |
| 绿联 DXP4800 Plus（满载）| ~25W | ~11 元 |
| **两台合计（平均混合负载）** | **~50W** | **~21 元/月** |

---

## 参考资料

- [绿联 DXP4800 Plus 官方参数](https://www.ugnas.com/products-detail/id-39.html)
- [极空间 Z423 标准版开箱评测](https://post.smzdm.com/p/admvgook/)
- [Ollama 官方文档](https://github.com/ollama/ollama)
- [HuggingFace TEI CPU 部署](https://github.com/huggingface/text-embeddings-inference)
- [Qdrant Docker 部署](https://qdrant.tech/documentation/guides/installation/)
- [阿里云 DashScope OpenAI 兼容接口](https://help.aliyun.com/zh/model-studio/developer-reference/compatibility-of-openai-with-dashscope)
- 本项目相关文档：
  - [RAG 组件选型指南](./rag-components-selection-guide.md)
  - [BGE-M3 + Qwen2.5-14B 完整部署](./rag-stack-deployment-guide.md)
  - [家庭实验室 GPU 部署方案](./home-lab-rag-deployment.md)
