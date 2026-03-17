# 自定义模型接入：第三方代理、本地模型、云厂商、兼容层

---

## 一、接入类型全景

```
Claude Code
    │
    ├─ 1. 官方直连 ──────────────→ api.anthropic.com
    │      需要科学上网 / Claude.ai 订阅
    │
    ├─ 2. 第三方中转代理 ─────────→ One-API / New-API / LiteLLM / 88code 等
    │      支持 Claude 模型，OpenAI 格式中转
    │
    ├─ 3. 云厂商托管 ─────────────→ AWS Bedrock / Google Vertex / Azure Foundry
    │      企业合规、数据主权场景
    │
    ├─ 4. OpenAI 兼容接口 ────────→ OpenRouter / Together AI / Groq / Fireworks
    │      通过适配层调用非 Claude 模型
    │
    └─ 5. 本地模型 ──────────────→ Ollama / LM Studio / vLLM / LocalAI
           完全私有，零成本，离线可用
```

---

## 二、官方直连配置

```jsonc
// ~/.claude/settings.json
{
  "env": {
    // Claude.ai 订阅（推荐，每月固定费用）
    "ANTHROPIC_AUTH_TOKEN": "sk-ant-..."
    // 或 Anthropic Console API Key
    // "ANTHROPIC_API_KEY": "sk-ant-..."
  }
}
```

国内用户需要代理：

```jsonc
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-ant-...",
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "HTTP_PROXY":  "http://127.0.0.1:7890",
    "NO_PROXY":    "localhost,127.0.0.1"
  }
}
```

---

## 三、第三方 Anthropic 格式中转

适用于国内可访问的中转服务（88code、newapi、one-api 等自建服务）。

### 3.1 最小配置

```jsonc
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "your-proxy-token",
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",

    // 必须固定模型别名（防止别名漂移导致 404）
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "claude-opus-4-6",

    // 大多数中转不支持 Prompt Cache
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

### 3.2 BASE_URL 格式规则

```
Claude Code 拼接规则：BASE_URL + "/messages"

✅ "https://proxy.com/v1"       → 请求 https://proxy.com/v1/messages
❌ "https://proxy.com/v1/"      → 请求 https://proxy.com/v1//messages（双斜杠）
❌ "https://proxy.com/v1/messages" → 请求 https://proxy.com/v1/messages/messages
❌ "https://proxy.com"          → 请求 https://proxy.com/messages（缺少 /v1）
```

### 3.3 认证 Header 选择

| 变量 | 发送的 Header | 适用场景 |
|------|-------------|---------|
| `ANTHROPIC_AUTH_TOKEN` | `Authorization: Bearer xxx` | 标准 Anthropic 格式 |
| `ANTHROPIC_API_KEY` | `X-Api-Key: xxx` | OpenAI 兼容格式的中转 |

> 两个变量**不要同时设置**。

---

## 四、AWS Bedrock

### 4.1 直接使用 AWS 凭证

```jsonc
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_ACCESS_KEY_ID":     "AKIA...",
    "AWS_SECRET_ACCESS_KEY": "your-secret",
    "AWS_REGION":            "us-east-1",

    // Bedrock 模型 ID 格式（跨区域推理）
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "us.anthropic.claude-sonnet-4-6-20250514-v1:0",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "us.anthropic.claude-haiku-4-5-20251001-v1:0",
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "us.anthropic.claude-opus-4-6-20250514-v1:0"
  }
}
```

### 4.2 通过企业 LLM 网关（跳过 AWS 认证）

```jsonc
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "ANTHROPIC_BEDROCK_BASE_URL":    "https://your-bedrock-gateway.com",
    "CLAUDE_CODE_SKIP_BEDROCK_AUTH": "1",
    "ANTHROPIC_AUTH_TOKEN":          "gateway-token",
    "AWS_REGION":                    "us-east-1",

    "ANTHROPIC_DEFAULT_SONNET_MODEL": "us.anthropic.claude-sonnet-4-6-20250514-v1:0"
  },
  "modelOverrides": {
    "claude-sonnet-4-6": "us.anthropic.claude-sonnet-4-6-20250514-v1:0",
    "claude-haiku-4-5-20251001": "us.anthropic.claude-haiku-4-5-20251001-v1:0"
  }
}
```

---

## 五、Google Vertex AI

```jsonc
{
  "env": {
    "CLAUDE_CODE_USE_VERTEX":    "1",
    "CLOUD_ML_REGION":           "us-east5",
    "ANTHROPIC_VERTEX_PROJECT_ID": "your-gcp-project-id"
  }
}
```

通过企业网关：

```jsonc
{
  "env": {
    "CLAUDE_CODE_USE_VERTEX":       "1",
    "ANTHROPIC_VERTEX_BASE_URL":    "https://vertex-gateway.internal.com",
    "CLAUDE_CODE_SKIP_VERTEX_AUTH": "1",
    "ANTHROPIC_AUTH_TOKEN":         "gateway-token"
  }
}
```

---

## 六、Microsoft Azure Foundry

```jsonc
{
  "env": {
    "CLAUDE_CODE_USE_FOUNDRY":       "1",
    "ANTHROPIC_FOUNDRY_RESOURCE":    "your-azure-resource",
    "ANTHROPIC_FOUNDRY_API_KEY":     "azure-api-key",
    "ANTHROPIC_FOUNDRY_REGION":      "eastus"
  }
}
```

---

## 七、OpenAI 兼容接口（调用非 Claude 模型）

> ⚠️ **重要限制**：Claude Code 本身是为 Claude 模型优化的，使用非 Claude 模型会导致部分功能降级（工具调用格式差异、上下文窗口不同）。以下方案适合**实验性使用**或**成本优化**场景。

### 7.1 通过 LiteLLM 代理统一接口

LiteLLM 可以将任意 LLM 包装为 Anthropic API 格式：

```bash
# 安装 LiteLLM
pip install litellm[proxy]

# 启动代理（将 OpenAI GPT-4 包装为 Anthropic 格式）
litellm --model gpt-4o \
        --port 4000 \
        --add_function_to_prompt  # 适配工具调用格式
```

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-openai-key",
    "ANTHROPIC_BASE_URL": "http://localhost:4000/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "gpt-4o",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "gpt-4o-mini",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

### 7.2 通过 OpenRouter 聚合

OpenRouter 提供统一接口访问数百个模型：

```jsonc
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-or-v1-...",
    "ANTHROPIC_BASE_URL": "https://openrouter.ai/api/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "anthropic/claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "google/gemini-2.0-flash-exp",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

---

## 八、本地模型（Ollama）

### 8.1 安装和配置 Ollama

```bash
# 安装 Ollama
# macOS/Linux
curl -fsSL https://ollama.ai/install.sh | sh

# 下载模型
ollama pull qwen2.5-coder:32b    # 代码模型（推荐）
ollama pull llama3.3:70b         # 通用模型
ollama pull deepseek-r1:32b      # 推理模型

# 启动 Ollama 服务（默认 localhost:11434）
ollama serve
```

### 8.2 配置 Claude Code 使用 Ollama

Ollama 提供 OpenAI 兼容接口，需要通过 LiteLLM 或直接适配：

```bash
# 方式 1：通过 LiteLLM 包装为 Anthropic 格式
litellm --model ollama/qwen2.5-coder:32b --port 4000

# 方式 2：使用 ollama-anthropic-proxy
npm install -g ollama-anthropic-proxy
ollama-anthropic-proxy --port 4001 --model qwen2.5-coder:32b
```

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "ollama",   // 随意填，本地不验证
    "ANTHROPIC_BASE_URL": "http://localhost:4001/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "qwen2.5-coder:32b",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "qwen2.5-coder:14b",
    "DISABLE_PROMPT_CACHING": "1",
    "NODE_TLS_REJECT_UNAUTHORIZED": "0"
  }
}
```

### 8.3 推荐本地模型选择

| 场景 | 推荐模型 | 最低 VRAM |
|------|---------|---------|
| 代码生成 | qwen2.5-coder:32b | 20GB |
| 通用对话 | llama3.3:70b | 40GB |
| 快速响应 | qwen2.5-coder:7b | 8GB |
| 复杂推理 | deepseek-r1:32b | 20GB |
| 低端设备 | phi4:14b | 10GB |

---

## 九、LM Studio（GUI 本地模型）

LM Studio 提供图形界面，启动后内置 OpenAI 兼容服务器：

```bash
# LM Studio 默认地址：http://localhost:1234
# 通过 LiteLLM 包装为 Anthropic 格式
litellm \
  --model openai/lm-studio-model \
  --api_base http://localhost:1234/v1 \
  --port 4000
```

```jsonc
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "lm-studio",
    "ANTHROPIC_BASE_URL": "http://localhost:4000/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "your-loaded-model-name",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

---

## 十、配置多端点共存

在同一个 `settings.json` 中配置主端点，通过 shell 别名按需覆盖：

```jsonc
// ~/.claude/settings.json（主端点）
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "primary-token",
    "ANTHROPIC_BASE_URL": "https://primary-proxy.com/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

```bash
# ~/.zshrc（覆盖端点的别名）
alias cc='claude'
alias cc-backup='ANTHROPIC_BASE_URL=https://backup-proxy.com/v1 claude'
alias cc-local='ANTHROPIC_BASE_URL=http://localhost:4000/v1 ANTHROPIC_AUTH_TOKEN=ollama claude'
alias cc-opus='ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-6 claude --model opus'
alias cc-bedrock='CLAUDE_CODE_USE_BEDROCK=1 AWS_REGION=us-east-1 claude'
```

---

## 十一、自定义模型调试清单

```bash
# 1. 测试端点连通性
curl -s -w "\nHTTP_CODE: %{http_code}\n" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"YOUR_MODEL_ID","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}' \
  https://your-endpoint.com/v1/messages

# 2. 检查模型 ID 是否正确
# 期望响应：{"content":[{"type":"text","text":"Hi!"}],...}
# 如果 404：模型 ID 错误，查看中转服务支持的模型列表

# 3. 检查是否需要禁用 Prompt Cache
# 如果报 400 unknown parameter cache_control：
# 添加 "DISABLE_PROMPT_CACHING": "1"

# 4. 检查 TLS
# 如果报证书错误：
# 添加 "NODE_TLS_REJECT_UNAUTHORIZED": "0"（仅内网）

# 5. 检查生效的配置
claude config list
```
