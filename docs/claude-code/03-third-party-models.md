# Claude Code 第三方模型配置注意事项

> 适用于 One-API、New-API、LiteLLM、LM Studio 等中转服务，
> 以及 AWS Bedrock、Google Vertex AI、Microsoft Foundry 等云厂商。

---

## 一、问题全景图

```
Claude Code
    │
    ├─ ANTHROPIC_BASE_URL ──→ 第三方中转（One-API/LiteLLM/...）
    │                              │
    │                              └─→ Anthropic API / 其他 LLM
    │
    └─ 直连模式 ──→ api.anthropic.com（需科学上网）
```

使用中转时，以下 8 类问题最为常见：

---

## 二、问题详解与规避

### 问题 1：模型别名漂移（最高频）

**现象**：`404 model not found`、`invalid_model`

**原因**：Claude Code 内部使用 `"sonnet"`、`"haiku"`、`"opus"` 别名，
Anthropic 发布新模型时别名自动指向新版本，但中转服务的模型列表未同步。

**规避**：在 `settings.json` 中固定别名指向：

```json
{
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "claude-opus-4-6"
  }
}
```

同时在 `settings.json` 中锁定可用模型：

```json
{
  "availableModels": [
    "claude-sonnet-4-6",
    "claude-haiku-4-5-20251001",
    "claude-opus-4-6"
  ]
}
```

---

### 问题 2：Base URL 路径格式错误

**现象**：`404`、`path not found`、`Cannot POST /v1/messages/messages`

**原因**：`ANTHROPIC_BASE_URL` 填写错误，导致路径重复或缺失。

Claude Code 拼接规则：`BASE_URL + /messages`

```
✅ ANTHROPIC_BASE_URL = "https://proxy.example.com/v1"
   → 实际请求：https://proxy.example.com/v1/messages

❌ ANTHROPIC_BASE_URL = "https://proxy.example.com/v1/messages"
   → 实际请求：https://proxy.example.com/v1/messages/messages （错误）

❌ ANTHROPIC_BASE_URL = "https://proxy.example.com"
   → 实际请求：https://proxy.example.com/messages （缺少 /v1）
```

**不同中转的正确格式**：

| 中转服务 | 正确 BASE_URL |
|----------|--------------|
| One-API / New-API | `https://your-domain.com/v1` |
| LiteLLM | `https://your-litellm.com/v1` |
| Cloudflare AI Gateway | `https://gateway.ai.cloudflare.com/v1/xxx/anthropic` |
| 自建 Nginx 代理 | `https://your-domain.com/api/v1`（视配置） |

---

### 问题 3：认证 Header 不兼容

**现象**：`401 Unauthorized`

**原因**：中转服务只识别特定的认证 Header：
- `ANTHROPIC_AUTH_TOKEN` → 发送 `Authorization: Bearer xxx`
- `ANTHROPIC_API_KEY` → 发送 `X-Api-Key: xxx`

部分中转仅支持其中一种。

**规避**：根据中转服务文档选择变量：

```jsonc
// 中转支持 X-Api-Key（OpenAI 兼容接口常见）
{ "env": { "ANTHROPIC_API_KEY": "sk-your-key" } }

// 中转支持 Authorization: Bearer（标准 Anthropic 格式）
{ "env": { "ANTHROPIC_AUTH_TOKEN": "sk-your-key" } }
```

> ⚠️ 两个变量**不要同时设置**，否则可能发送两个 Header 导致冲突。

---

### 问题 4：Prompt Cache 字段不兼容

**现象**：`unknown parameter: cache_control`、`400 Bad Request`

**原因**：Claude Code 默认开启 Prompt Cache，请求体中注入 `cache_control` 字段。
大多数第三方中转不支持此字段，转发时直接报错。

**规避**：

```json
{ "env": { "DISABLE_PROMPT_CACHING": "1" } }
```

---

### 问题 5：SSL/TLS 证书验证失败

**现象**：`UNABLE_TO_VERIFY_LEAF_SIGNATURE`、`CERT_HAS_EXPIRED`、`self-signed certificate`

**原因**：中转服务使用自签名或企业内网证书。

**规避方案（按安全性排序）**：

```jsonc
// 方案A：信任指定 CA（推荐）
{ "env": { "NODE_EXTRA_CA_CERTS": "/etc/ssl/certs/corp-ca.pem" } }

// 方案B：完全跳过验证（仅内网/开发环境）
{ "env": { "NODE_TLS_REJECT_UNAUTHORIZED": "0" } }
```

---

### 问题 6：流式响应被截断

**现象**：响应中途断开，长代码生成失败，Claude "说到一半停止"

**原因**：中转服务或反向代理超时/缓冲区配置不当。

**客户端规避**：
```json
{
  "env": {
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "32000"
  }
}
```

**服务端必须配置**（以 Nginx 为例）：
```nginx
location /v1/ {
    proxy_pass         http://upstream;
    proxy_read_timeout 600s;
    proxy_buffering    off;        # 关键：禁用响应缓冲
    proxy_cache        off;
    chunked_transfer_encoding on;
}
```

---

### 问题 7：VS Code 插件仍弹出 Anthropic 登录框

**现象**：配置了 URL 和 Key，插件仍要求用 Anthropic 账号登录。

**原因**：插件默认走 OAuth 登录流程。

**规避**：

```json
{
  "claudeCode.disableLoginPrompt": true
}
```

---

### 问题 8：VS Code 图形进程不继承 Shell 变量

**现象**：终端 `export` 的变量，在 VS Code 图形界面的 Claude 插件里不生效。

**原因**：VS Code 图形主进程启动时不经过 Shell 初始化，无法继承 `.bashrc` / `.zshrc` 中的 `export`。

**规避**：不依赖 Shell export，改用以下任一方式：

```jsonc
// 方式A：~/.claude/settings.json（CLI + 插件均读取）
{ "env": { "ANTHROPIC_BASE_URL": "https://..." } }

// 方式B：VS Code settings.json（仅插件）
{
  "claudeCode.environmentVariables": [
    { "name": "ANTHROPIC_BASE_URL", "value": "https://..." }
  ]
}
```

---

## 三、云厂商专项配置

### 3.1 AWS Bedrock

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_REGION": "us-east-1",

    // 通过 LLM 网关时（跳过 AWS 认证）
    "ANTHROPIC_BEDROCK_BASE_URL":  "https://your-bedrock-gateway.com",
    "CLAUDE_CODE_SKIP_BEDROCK_AUTH": "1",

    // 必须 pin 模型（Bedrock 使用 ARN 格式）
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "us.anthropic.claude-sonnet-4-6-20250514-v1:0",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "us.anthropic.claude-haiku-4-5-20251001-v1:0"
  },

  // 或使用 modelOverrides 映射
  "modelOverrides": {
    "claude-sonnet-4-6": "us.anthropic.claude-sonnet-4-6-20250514-v1:0"
  }
}
```

### 3.2 Google Vertex AI

```jsonc
{
  "env": {
    "CLAUDE_CODE_USE_VERTEX": "1",
    "CLOUD_ML_REGION": "us-east5",
    "ANTHROPIC_VERTEX_PROJECT_ID": "your-gcp-project-id",

    // 通过网关（跳过 GCP 认证）
    "ANTHROPIC_VERTEX_BASE_URL":    "https://your-vertex-gateway.com",
    "CLAUDE_CODE_SKIP_VERTEX_AUTH": "1"
  }
}
```

### 3.3 Microsoft Azure Foundry

```jsonc
{
  "env": {
    "CLAUDE_CODE_USE_FOUNDRY": "1",
    "ANTHROPIC_FOUNDRY_RESOURCE": "your-azure-resource-name",
    "ANTHROPIC_FOUNDRY_API_KEY":  "your-azure-key",

    // 通过网关（跳过 Azure 认证）
    "ANTHROPIC_FOUNDRY_BASE_URL":    "https://your-foundry-gateway.com",
    "CLAUDE_CODE_SKIP_FOUNDRY_AUTH": "1"
  }
}
```

---

## 四、调试清单

配置问题时按以下顺序排查：

```bash
# 1. 进入 Claude Code 查看生效配置
/status

# 2. 检查环境变量是否注入
# 在 Claude 内执行 bash：
env | grep -E "ANTHROPIC|CLAUDE|PROXY|NODE_TLS"

# 3. 手动测试中转端点连通性
curl -v \
  -H "Authorization: Bearer your-token" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}' \
  https://your-proxy.com/v1/messages

# 4. 检查 CLI 配置
claude config list

# 5. VS Code 插件日志
# 命令面板 → "Claude Code: Show Logs"
```

---

## 五、最小可用配置（第三方中转）

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN":           "your-token",
    "ANTHROPIC_BASE_URL":             "https://your-proxy.com/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "claude-opus-4-6",
    "DISABLE_PROMPT_CACHING":         "1"
  }
}
```

```jsonc
// VS Code settings.json（额外追加）
{
  "claudeCode.disableLoginPrompt": true,
  "claudeCode.environmentVariables": [
    { "name": "ANTHROPIC_AUTH_TOKEN",           "value": "your-token" },
    { "name": "ANTHROPIC_BASE_URL",             "value": "https://your-proxy.com/v1" },
    { "name": "ANTHROPIC_DEFAULT_SONNET_MODEL", "value": "claude-sonnet-4-6" },
    { "name": "ANTHROPIC_DEFAULT_HAIKU_MODEL",  "value": "claude-haiku-4-5-20251001" },
    { "name": "DISABLE_PROMPT_CACHING",         "value": "1" }
  ]
}
```
