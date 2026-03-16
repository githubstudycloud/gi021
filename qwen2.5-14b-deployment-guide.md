# Qwen2.5-14B 部署指南：显存、内存与并发规划

> 版本：1.0 | 日期：2026-03-16 | 适用模型：Qwen2.5-14B-Instruct / Qwen2.5-14B-Instruct-1M

---

## 1. 显存（VRAM）需求

模型权重本身占用：14B 参数 × 精度字节数 = 基础显存。实际部署还需加上 KV Cache 和激活内存。

### 1.1 按精度划分（仅模型权重）

| 精度 | 每参数字节 | 模型权重显存 | 备注 |
|------|-----------|------------|------|
| FP32 | 4 B | ~56 GB | 训练用，推理不推荐 |
| BF16 / FP16 | 2 B | **~28 GB** | 官方推荐推理精度 |
| FP8 | 1 B | ~14 GB | Ampere/Hopper 支持 |
| INT8 (GPTQ/AWQ) | 1 B | ~14 GB | 精度损失极小 |
| INT4 (GPTQ/AWQ/Q4_K_M) | 0.5 B | **~7–10 GB** | 轻量化部署首选 |

### 1.2 加上 KV Cache 后的实际显存（vLLM，BF16）

KV Cache 大小 ∝ `batch_size × seq_len × num_layers × hidden_dim`。

以 Qwen2.5-14B（32 层，5120 隐藏维度）为例估算：

| 显卡 | 显存 | 最大并发 token 容量 | 单卡可跑？ |
|------|------|------------------|----------|
| RTX 4090 | 24 GB | 勉强放下权重（约 28 GB BF16，需量化）| 需 INT4/INT8 |
| RTX 4090 × 2 | 48 GB | 权重 28 GB + ~20 GB KV | ✅ BF16 可用 |
| A100 40 GB | 40 GB | 权重 28 GB + ~12 GB KV | ✅（上下文受限）|
| **A100 80 GB** | **80 GB** | 权重 28 GB + **~52 GB KV** | ✅ **推荐生产** |
| H100 SXM 80 GB | 80 GB | 同上，带宽更高 | ✅ 最优选 |

> **结论**：生产全量 BF16 部署，单实例推荐 **A100/H100 80 GB**；若用 INT4 量化，RTX 4090 单卡即可运行。

---

## 2. 系统内存（RAM）需求

| 部署场景 | 最低 RAM | 推荐 RAM | 说明 |
|---------|---------|---------|------|
| 单卡开发测试 | 32 GB | 64 GB | 模型加载 + 少量并发 |
| 单实例生产（vLLM） | 64 GB | **128 GB** | KV Cache 溢出到内存 + 请求队列 |
| 多实例 / 分布式 | 128 GB | **256 GB+** | 每实例独立内存空间 |

- vLLM 默认 `gpu_memory_utilization=0.9`，不足时会将 KV Cache 换出到 CPU 内存，导致延迟剧增。
- 建议 CPU 内存至少为单 GPU 显存的 **2–3 倍**。

---

## 3. 单实例并发能力（vLLM + A100 80 GB，BF16）

基于公开基准数据：

| 推理引擎 | 并发请求数 | 吞吐量（tokens/s） | 来源 |
|---------|-----------|-----------------|------|
| vLLM | 50 req | ~2,500–3,000 | A100 80 GB 实测 |
| vLLM | 100 req | ~3,000–4,000 | 接近性能峰值 |
| TensorRT-LLM | 100 req | ~5,700 | 最高性能，部署复杂 |
| SGLang | 100 req | ~5,200 | 高吞吐替代方案 |

### 典型业务场景假设

```
平均输入 token：512
平均输出 token：512
P99 TTFT 目标：< 3s
单实例稳定并发上限：~80–100 并发请求（A100 80GB + vLLM）
```

> **注意**：并发数受 `max_num_seqs`、`max_model_len`、KV Cache 容量共同制约。
> 实际业务中建议留 20% 余量，有效并发按 **80 并发/实例** 规划。

---

## 4. 300 并发部署规划

### 4.1 实例数量计算

```
目标并发：300
单实例有效并发：80（保守估计，留余量）
所需实例数：300 ÷ 80 = 3.75 → 取整 = 4 实例

加上高可用冗余（N+1）：4 + 1 = 5 实例
```

| 方案 | GPU 配置 | 实例数 | 总 GPU 数 | 说明 |
|------|---------|-------|---------|------|
| **推荐：标准 BF16** | 1× A100 80G/实例 | **5** | 5× A100 | 稳定、易维护 |
| 经济：INT4 量化 | 1× RTX 4090/实例 | **5–6** | 5–6× RTX4090 | 成本低，精度略降 |
| 高性能：双卡 BF16 | 2× A100 40G/实例 | **5** | 10× A100 40G | 适合旧有 40G 卡资产 |
| 极致性能 | 1× H100 80G/实例 + TRT-LLM | **3–4** | 3–4× H100 | 吞吐最高，成本最高 |

### 4.2 推荐架构（标准 BF16 方案）

```
                    ┌─────────────────────┐
                    │    负载均衡器        │
                    │  (Nginx / Gateway)   │
                    └──────────┬──────────┘
                               │ 300 并发
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
     ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
     │  vLLM 实例 #1   │ │  vLLM 实例 #2   │ │  vLLM 实例 #3   │
     │  A100 80G      │ │  A100 80G      │ │  A100 80G      │
     │  ~80 并发       │ │  ~80 并发       │ │  ~80 并发       │
     └────────────────┘ └────────────────┘ └────────────────┘
              ▼                ▼                ▼
     ┌────────────────┐ ┌────────────────┐
     │  vLLM 实例 #4   │ │  vLLM 实例 #5   │
     │  A100 80G      │ │  A100 80G      │   ← 冗余/备用
     │  ~80 并发       │ │  Standby       │
     └────────────────┘ └────────────────┘
```

### 4.3 vLLM 启动参数参考

```bash
# 单实例启动（BF16，A100 80G，标准 128K 上下文）
vllm serve Qwen/Qwen2.5-14B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --dtype bfloat16 \
  --max-model-len 32768 \
  --max-num-seqs 100 \
  --gpu-memory-utilization 0.90 \
  --tensor-parallel-size 1 \
  --served-model-name qwen2.5-14b

# 量化版本（INT4 AWQ，RTX 4090）
vllm serve Qwen/Qwen2.5-14B-Instruct-AWQ \
  --host 0.0.0.0 \
  --port 8000 \
  --quantization awq \
  --dtype float16 \
  --max-model-len 16384 \
  --max-num-seqs 80 \
  --gpu-memory-utilization 0.90
```

---

## 5. 软件依赖

| 组件 | 版本要求 |
|------|---------|
| CUDA | 12.1+ |
| cuDNN | 8.9+ |
| Python | 3.10+ |
| PyTorch | 2.3+ |
| vLLM | 0.5.0+ |
| Transformers | 4.43+ |

```bash
pip install vllm>=0.5.0 transformers>=4.43.0
```

---

## 6. 成本参考（云端 GPU 租用）

| GPU 类型 | 参考单价（/小时） | 5 实例月费（730h） |
|---------|---------------|----------------|
| A100 80G SXM | ~$2.5–3.5 | **~$9,125–$12,775** |
| H100 80G SXM | ~$4.0–5.5 | ~$14,600–$20,075 |
| RTX 4090 | ~$0.7–1.0 | ~$2,555–$3,650（6 卡） |

> 价格因云厂商、地区、合同方式不同有较大差异，以实际报价为准。

---

## 7. 部署检查清单

- [ ] GPU 驱动 ≥ 535，CUDA 12.1+
- [ ] 系统内存 ≥ 128 GB/节点
- [ ] NVMe SSD，模型加载速度 ≥ 2 GB/s
- [ ] 负载均衡配置（最少连接或一致性哈希）
- [ ] vLLM metrics 接入 Prometheus + Grafana
- [ ] 设置 `max-num-seqs` 防止 OOM
- [ ] 健康检查端点：`GET /health`
- [ ] 模型权重本地缓存，避免重启重下载

---

## 参考资料

- [Qwen2.5 官方速度基准](https://qwen.readthedocs.io/en/latest/getting_started/speed_benchmark.html)
- [Qwen2.5-14B-Instruct-1M HuggingFace](https://huggingface.co/Qwen/Qwen2.5-14B-Instruct-1M)
- [vLLM A100 80GB 基准测试](https://www.databasemart.com/blog/vllm-gpu-benchmark-a100-80gb)
- [A6000 vLLM 基准（50/100 并发）](https://www.databasemart.com/blog/vllm-gpu-benchmark-a6000)
- [GPUStack Qwen3-14B A100 性能实验室](https://docs.gpustack.ai/2.0/performance-lab/qwen3-14b/a100/)
- [Qwen 硬件需求指南](https://apxml.com/posts/gpu-system-requirements-qwen-models)
