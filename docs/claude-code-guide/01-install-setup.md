# 安装与初始配置

---

## 一、安装

### 前提条件

| 依赖 | 最低版本 | 说明 |
|------|----------|------|
| Node.js | 18+ | 推荐 20 LTS |
| npm | 8+ | 随 Node.js 安装 |
| Git | 2.x | 可选，但强烈推荐 |
| OS | Win10/macOS 12/Ubuntu 20.04 | |

### 安装命令

```bash
# 全局安装
npm install -g @anthropic-ai/claude-code

# 验证安装
claude --version

# 更新到最新版
npm update -g @anthropic-ai/claude-code
```

### Windows 特殊说明

```powershell
# 推荐在 WSL2 中运行（最佳体验）
wsl --install
# 然后在 WSL2 中执行 npm install

# 原生 Windows（Git Bash / PowerShell）也支持，但部分功能受限
# 确保 npm 全局目录在 PATH 中：
$env:PATH += ";$env:APPDATA\npm"
```

---

## 二、认证

### 方式一：交互式登录（推荐个人用户）

```bash
claude
# 首次运行会引导完成 Anthropic 账号登录
# 凭证加密存储在系统密钥链（macOS Keychain / Windows Credential Manager）
```

### 方式二：API Key（推荐第三方中转）

```bash
# ~/.claude/settings.json
{
  "env": {
    "ANTHROPIC_API_KEY":   "sk-ant-...",   # 发送 X-Api-Key header
    # 或
    "ANTHROPIC_AUTH_TOKEN": "sk-ant-...",   # 发送 Authorization: Bearer header
    "ANTHROPIC_BASE_URL":  "https://your-proxy.com/v1"
  }
}
```

> ⚠️ `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN` 二选一，不要同时设置。

### 方式三：apiKeyHelper（企业/团队推荐）

```json
// ~/.claude/settings.json
{
  "apiKeyHelper": "~/.claude/scripts/get-token.sh"
}
```

```bash
#!/bin/bash
# get-token.sh：从密钥管理服务动态获取 Token
curl -s "https://vault.corp.com/v1/secret/claude" \
  -H "X-Vault-Token: $VAULT_TOKEN" | jq -r '.data.token'
```

---

## 三、国内使用配置

### 最小可用配置

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN":           "your-token",
    "ANTHROPIC_BASE_URL":             "https://your-proxy.com/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "claude-opus-4-6",
    "DISABLE_PROMPT_CACHING":         "1",
    "DISABLE_TELEMETRY":              "1"
  }
}
```

> 📌 **必须固定模型别名**：Claude Code 内部使用 `sonnet`/`haiku` 等别名，
> Anthropic 发布新模型时别名自动漂移，导致中转服务 404。

### 代理配置

```jsonc
{
  "env": {
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "HTTP_PROXY":  "http://127.0.0.1:7890",
    "NO_PROXY":    "localhost,127.0.0.1,*.your-proxy.com"
  }
}
```

### URL 格式规范

```
✅ ANTHROPIC_BASE_URL = "https://proxy.com/v1"
   → 实际请求：https://proxy.com/v1/messages

❌ ANTHROPIC_BASE_URL = "https://proxy.com/v1/messages"   # 路径重复
❌ ANTHROPIC_BASE_URL = "https://proxy.com"               # 缺少 /v1
```

---

## 四、项目初始化

```bash
cd your-project
claude
> /init       # 自动分析项目生成 CLAUDE.md
> /status     # 查看当前配置和连接状态
```

`/init` 会：
- 检测构建系统（package.json、Makefile、pyproject.toml 等）
- 识别测试框架
- 生成 `CLAUDE.md` 草稿

---

## 五、VS Code 插件

```
插件 ID：anthropic.claude-code
最低 VS Code 版本：1.98.0
```

### 关键设置

```jsonc
// VS Code settings.json
{
  // ⚠️ 格式是数组，不是对象！
  "claudeCode.environmentVariables": [
    { "name": "ANTHROPIC_AUTH_TOKEN",           "value": "your-token" },
    { "name": "ANTHROPIC_BASE_URL",             "value": "https://your-proxy.com/v1" },
    { "name": "ANTHROPIC_DEFAULT_SONNET_MODEL", "value": "claude-sonnet-4-6" },
    { "name": "ANTHROPIC_DEFAULT_HAIKU_MODEL",  "value": "claude-haiku-4-5-20251001" },
    { "name": "DISABLE_PROMPT_CACHING",         "value": "1" }
  ],
  "claudeCode.disableLoginPrompt":    true,     // 第三方认证必须设为 true
  "claudeCode.selectedModel":         "claude-sonnet-4-6",
  "claudeCode.preferredLocation":     "panel",
  "claudeCode.initialPermissionMode": "default"
}
```

> ⚠️ VS Code 图形进程**不继承** Shell 的 `export` 变量，必须通过上面的方式注入。

### 快捷键

| 操作 | 快捷键 |
|------|--------|
| 打开 / 聚焦 Claude | `Ctrl+Esc` |
| 新建对话 | `Ctrl+N`（Claude 聚焦时） |
| 在新标签页打开 | `Ctrl+Shift+Esc` |
| 插入 @-mention | `Alt+K` |

---

## 六、卸载与清理

```bash
# 卸载 CLI
npm uninstall -g @anthropic-ai/claude-code

# 清理用户数据（可选）
rm -rf ~/.claude/

# Windows
rmdir /s /q %USERPROFILE%\.claude
```
