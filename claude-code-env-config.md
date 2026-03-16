# Claude Code 环境变量与第三方 URI 配置完整指南

> 整理自官方文档 https://docs.anthropic.com/en/docs/claude-code/settings
> 更新日期：2026-03-16

---

## 一、核心环境变量速查表

### 1.1 认证变量

| 变量名 | Header | 适用场景 | 说明 |
|--------|--------|----------|------|
| `ANTHROPIC_AUTH_TOKEN` | `Authorization` | Claude Code CLI 交互式使用 | **推荐**，官方优先方式 |
| `ANTHROPIC_API_KEY` | `X-Api-Key` | SDK / 编程调用 | 用于第三方中转/API Key 场景 |
| `ANTHROPIC_FOUNDRY_API_KEY` | - | Microsoft Foundry | Foundry 专用 Key |

> ⚠️ 两者选其一，同时设置可能导致冲突。

### 1.2 Base URL 变量

| 变量名 | 适用场景 |
|--------|----------|
| `ANTHROPIC_BASE_URL` | 通用自定义端点（One-API、LiteLLM、国内中转等） |
| `ANTHROPIC_BEDROCK_BASE_URL` | AWS Bedrock 网关/代理 |
| `ANTHROPIC_VERTEX_BASE_URL` | Google Vertex AI 网关/代理 |
| `ANTHROPIC_FOUNDRY_BASE_URL` | Microsoft Foundry 网关 |

### 1.3 模型 Pin 变量

| 变量名 | 说明 |
|--------|------|
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | 固定 Opus 别名指向的模型版本 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | 固定 Sonnet 别名指向的模型版本 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | 固定 Haiku 别名指向的模型版本 |
| `ANTHROPIC_MODEL` | 当前会话使用的模型 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子 Agent 使用的模型 |

### 1.4 云厂商开关变量

| 变量名 | 值 | 说明 |
|--------|-----|------|
| `CLAUDE_CODE_USE_BEDROCK` | `1` | 启用 AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | `1` | 启用 Google Vertex AI |
| `CLAUDE_CODE_USE_FOUNDRY` | `1` | 启用 Microsoft Foundry |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | `1` | 跳过 AWS 认证（网关代为处理） |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | `1` | 跳过 GCP 认证（网关代为处理） |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | `1` | 跳过 Azure 认证（网关代为处理） |

### 1.5 网络代理变量

| 变量名 | 示例 |
|--------|------|
| `HTTPS_PROXY` | `http://user:pass@proxy.corp.com:8080` |
| `HTTP_PROXY` | `http://proxy.corp.com:8080` |
| `NO_PROXY` | `localhost,127.0.0.1,.internal.com` |
| `NODE_EXTRA_CA_CERTS` | `/path/to/ca-cert.pem` |
| `NODE_TLS_REJECT_UNAUTHORIZED` | `0`（仅开发环境，有安全风险） |

### 1.6 行为控制变量

| 变量名 | 说明 |
|--------|------|
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 最大输出 token 数 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 上下文自动压缩触发阈值（1-100） |
| `BASH_MAX_TIMEOUT_MS` | Bash 命令最大超时时间（ms） |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | `1` 禁用自适应推理 |
| `DISABLE_PROMPT_CACHING` | 全局禁用 Prompt 缓存 |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | API Key 刷新间隔（默认 300000ms） |

---

## 二、CLI 配置完整指南

### 2.1 配置优先级（从高到低）

```
Shell 环境变量 > ~/.claude/settings.json (user) > .claude/settings.json (project) > 默认值
```

### 2.2 方式一：Shell 环境变量

```bash
# 临时（当前会话）
export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"
export ANTHROPIC_API_KEY="sk-xxx"
claude

# 持久化（写入 ~/.bashrc 或 ~/.zshrc）
echo 'export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"' >> ~/.zshrc
echo 'export ANTHROPIC_API_KEY="sk-xxx"' >> ~/.zshrc
source ~/.zshrc
```

### 2.3 方式二：settings.json（推荐）

```bash
# 用户级别（全局生效）
~/.claude/settings.json

# 项目级别（团队共享，提交到 git）
.claude/settings.json

# 本地项目级别（个人，不提交 git，已在 .gitignore）
.claude/settings.local.json
```

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",
    "ANTHROPIC_API_KEY": "sk-xxx",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "HTTPS_PROXY": "http://proxy.internal.com:8080",
    "NO_PROXY": "localhost,127.0.0.1"
  }
}
```

### 2.4 方式三：CLI 命令配置

```bash
claude config set env.ANTHROPIC_BASE_URL "https://your-proxy.com/v1"
claude config set env.ANTHROPIC_API_KEY "sk-xxx"
claude config set env.ANTHROPIC_DEFAULT_SONNET_MODEL "claude-sonnet-4-6"

# 查看当前配置
claude config list

# 删除配置
claude config remove env.ANTHROPIC_BASE_URL
```

### 2.5 使用第三方中转（One-API / LiteLLM / NewAPI）

```json
// ~/.claude/settings.json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://your-oneapi.com",
    "ANTHROPIC_API_KEY": "sk-your-forwarded-key",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6"
  }
}
```

> ⚠️ **必须 pin 模型版本**。不 pin 的话，Claude Code 使用别名（如 "sonnet"），当 Anthropic 发布新模型时别名自动更新，导致中转服务返回 404/model not found。

### 2.6 AWS Bedrock 配置

```bash
# 直连
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
claude

# 通过 LLM 网关
export CLAUDE_CODE_USE_BEDROCK=1
export ANTHROPIC_BEDROCK_BASE_URL='https://your-gateway.com/bedrock'
export CLAUDE_CODE_SKIP_BEDROCK_AUTH=1
claude
```

### 2.7 Google Vertex AI 配置

```bash
# 直连
export CLAUDE_CODE_USE_VERTEX=1
export CLOUD_ML_REGION=us-east5
export ANTHROPIC_VERTEX_PROJECT_ID=your-gcp-project-id
claude

# 通过 LLM 网关
export CLAUDE_CODE_USE_VERTEX=1
export ANTHROPIC_VERTEX_BASE_URL='https://your-gateway.com/vertex'
export CLAUDE_CODE_SKIP_VERTEX_AUTH=1
claude
```

### 2.8 验证配置

```bash
# 进入 Claude Code 后执行
/status
```

---

## 三、VS Code 插件配置完整指南

### 3.1 安装要求

- VS Code 版本 ≥ 1.98.0
- 插件 ID：`anthropic.claude-code`
- 安装后包含 CLI，可在集成终端直接使用 `claude`

### 3.2 VS Code 专属设置项

打开方式：`Ctrl+,` → 搜索 "Claude Code"，或编辑 VS Code 的 `settings.json`

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `claudeCode.environmentVariables` | `object` | `{}` | **注入环境变量给 Claude 进程** |
| `claudeCode.selectedModel` | `string` | `default` | 新会话使用的模型 |
| `claudeCode.useTerminal` | `boolean` | `false` | 使用终端模式而非图形面板 |
| `claudeCode.initialPermissionMode` | `string` | `default` | 权限模式：`default`/`plan`/`acceptEdits`/`bypassPermissions` |
| `claudeCode.preferredLocation` | `string` | `panel` | 打开位置：`sidebar`/`panel` |
| `claudeCode.autosave` | `boolean` | `true` | Claude 读写前自动保存文件 |
| `claudeCode.useCtrlEnterToSend` | `boolean` | `false` | 用 Ctrl+Enter 发送（默认 Enter） |
| `claudeCode.enableNewConversationShortcut` | `boolean` | `true` | Cmd/Ctrl+N 新建会话 |
| `claudeCode.disableLoginPrompt` | `boolean` | `false` | **跳过登录提示**（第三方服务必须设为 true） |
| `claudeCode.allowDangerouslySkipPermissions` | `boolean` | `false` | 跳过所有权限确认（慎用） |
| `claudeCode.respectGitIgnore` | `boolean` | `true` | 排除 .gitignore 中的文件 |
| `claudeCode.claudeProcessWrapper` | `string` | - | 启动 Claude 进程的包装器路径 |

### 3.3 VS Code settings.json 配置示例

```json
// .vscode/settings.json 或 用户 settings.json
{
  "claudeCode.environmentVariables": {
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",
    "ANTHROPIC_API_KEY": "sk-xxx",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6"
  },
  "claudeCode.disableLoginPrompt": true,
  "claudeCode.selectedModel": "claude-sonnet-4-6",
  "claudeCode.preferredLocation": "panel",
  "claudeCode.initialPermissionMode": "default"
}
```

### 3.4 常用快捷键

| 操作 | 快捷键 |
|------|--------|
| 聚焦输入框 | `Cmd/Ctrl+Esc` |
| 新建对话 | `Cmd/Ctrl+N`（Claude 已聚焦时） |
| 在新标签页打开 | `Cmd/Ctrl+Shift+Esc` |
| 插入 @-mention | `Option/Alt+K` |

---

## 四、配置第三方 URI 常见问题与规避方案

### 4.1 问题一：模型别名解析失败（最常见）

**现象**：中转服务报 `404 model not found` 或 `invalid model`

**原因**：Claude Code 内部使用 `sonnet`/`haiku`/`opus` 等别名，这些别名会随 Anthropic 发布新模型自动更新，中转服务的模型列表没有同步更新。

**规避方案**：必须通过以下变量固定模型版本：

```json
{
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6"
  }
}
```

---

### 4.2 问题二：URL 路径格式不匹配

**现象**：请求返回 `404` 或 `path not found`

**原因**：不同中转服务的 API 路径不同：
- Anthropic 原生：`https://api.anthropic.com/v1/messages`
- LiteLLM：路径通常为 `/v1`
- One-API / New-API：可能为 `/v1` 或自定义路径

**规避方案**：`ANTHROPIC_BASE_URL` 不要带 `/messages`，只写到 `/v1`，Claude Code 会自动拼接剩余路径。

```bash
# ✅ 正确
ANTHROPIC_BASE_URL="https://your-proxy.com/v1"

# ❌ 错误（多了 /messages）
ANTHROPIC_BASE_URL="https://your-proxy.com/v1/messages"

# ❌ 错误（有时中转要求去掉 /v1，需测试）
ANTHROPIC_BASE_URL="https://your-proxy.com"
```

---

### 4.3 问题三：SSL/TLS 证书验证失败

**现象**：`UNABLE_TO_VERIFY_LEAF_SIGNATURE` 或 `certificate verify failed`

**原因**：中转服务使用自签名证书或企业内网证书

**规避方案**：

```bash
# 方案A（推荐）：指定 CA 证书
NODE_EXTRA_CA_CERTS=/path/to/your-ca.pem

# 方案B（仅开发环境，有安全风险）
NODE_TLS_REJECT_UNAUTHORIZED=0
```

---

### 4.4 问题四：认证 Header 格式不兼容

**现象**：`401 Unauthorized`，日志显示认证失败

**原因**：
- `ANTHROPIC_AUTH_TOKEN` 发送 `Authorization: Bearer xxx` header
- `ANTHROPIC_API_KEY` 发送 `X-Api-Key: xxx` header
- 部分中转服务只识别其中一种

**规避方案**：根据中转服务支持的格式选择对应变量：

```bash
# 中转服务支持 X-Api-Key 时
ANTHROPIC_API_KEY="sk-xxx"

# 中转服务支持 Authorization: Bearer 时
ANTHROPIC_AUTH_TOKEN="sk-xxx"
```

---

### 4.5 问题五：Prompt Cache 与第三方服务冲突

**现象**：请求报错 `cache_control not supported` 或 `unknown parameter`

**原因**：Claude Code 默认开启 Prompt Cache，向请求中注入 `cache_control` 字段，第三方中转服务不支持转发此参数。

**规避方案**：

```json
{
  "env": {
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

---

### 4.6 问题六：流式响应（Streaming）被截断

**现象**：响应中途断开，长任务无法完成

**原因**：中转服务或反向代理的超时/缓冲区配置不当

**规避方案**：
1. 中转服务端设置足够大的超时（建议 ≥ 600s）
2. 禁用代理的响应缓冲（Nginx：`proxy_buffering off`）
3. 检查 `CLAUDE_CODE_MAX_OUTPUT_TOKENS` 是否设置过大

---

### 4.7 问题七：VS Code 插件仍弹出登录提示

**现象**：即使配置了 `ANTHROPIC_BASE_URL` 和 Key，插件仍要求登录 Anthropic 账号

**原因**：插件默认走 Anthropic OAuth 登录流程

**规避方案**：必须在 VS Code settings.json 中设置：

```json
{
  "claudeCode.disableLoginPrompt": true
}
```

---

### 4.8 问题八：环境变量在 VS Code 插件中不生效

**现象**：Shell 里 export 的变量，在 VS Code 图形界面插件中不生效

**原因**：VS Code 图形进程不继承用户 Shell 的环境变量

**规避方案**：不要依赖 Shell export，改用两种方式之一：

**方式A**：VS Code settings.json 的 `claudeCode.environmentVariables`

```json
{
  "claudeCode.environmentVariables": {
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",
    "ANTHROPIC_API_KEY": "sk-xxx"
  }
}
```

**方式B**：`~/.claude/settings.json`（CLI 和插件都读）

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",
    "ANTHROPIC_API_KEY": "sk-xxx"
  }
}
```

---

## 五、最佳实践配置模板

### 5.1 国内用户使用第三方中转（推荐配置）

**`~/.claude/settings.json`（CLI + 插件通用）**：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",
    "ANTHROPIC_API_KEY": "sk-your-key",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

**VS Code `settings.json`（插件专属追加）**：

```json
{
  "claudeCode.disableLoginPrompt": true,
  "claudeCode.environmentVariables": {
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",
    "ANTHROPIC_API_KEY": "sk-your-key",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001"
  }
}
```

### 5.2 企业内网代理配置

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://internal-gateway.corp.com/v1",
    "ANTHROPIC_API_KEY": "sk-internal-key",
    "HTTPS_PROXY": "http://proxy.corp.com:8080",
    "NO_PROXY": "localhost,127.0.0.1,.corp.com",
    "NODE_EXTRA_CA_CERTS": "/etc/ssl/certs/corp-ca.pem",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001"
  }
}
```

---

## 六、调试与验证

```bash
# 1. 进入 Claude Code 查看当前配置状态
/status

# 2. 检查环境变量是否生效（在 Claude 内执行）
/tools bash
env | grep ANTHROPIC

# 3. 查看 CLI 配置
claude config list

# 4. VS Code 插件日志
# 命令面板 → "Claude Code: Show Logs"
```

---

*文档来源：https://docs.anthropic.com/en/docs/claude-code/settings*
