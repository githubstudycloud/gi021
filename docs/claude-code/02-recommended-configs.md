# Claude Code 推荐配置模板

> 覆盖场景：个人开发、项目共享、企业管控、多平台

---

## 一、配置文件位置速查

| 场景 | 文件路径 | 是否提交 git | 优先级 |
|------|----------|:---:|:---:|
| 企业管控 | `C:\ProgramData\ClaudeCode\managed-settings.json` (Win) | 否 | 最高 |
| 个人全局 | `~/.claude/settings.json` | 否 | 低 |
| 项目共享 | `<repo>/.claude/settings.json` | **是** | 中 |
| 本地私有 | `<repo>/.claude/settings.local.json` | 否（gitignore）| 中高 |
| VS Code 插件 | VS Code `settings.json` | 视情况 | — |

---

## 二、个人全局配置（~/.claude/settings.json）

适用：所有项目通用的个人偏好，含认证信息。

```jsonc
// ~/.claude/settings.json
{
  "env": {
    // === 认证（填入实际值，不要提交到 git）===
    "ANTHROPIC_AUTH_TOKEN": "your-token-here",
    "ANTHROPIC_BASE_URL":   "https://your-proxy.com/v1",

    // === 固定模型别名（使用第三方中转时必须）===
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "claude-opus-4-6",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "claude-haiku-4-5-20251001",

    // === 隐私保护 ===
    "DISABLE_TELEMETRY":                        "1",
    "DISABLE_ERROR_REPORTING":                  "1",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "DISABLE_AUTOUPDATER":                      "1"
  },

  "permissions": {
    "defaultMode": "default"
  },

  "cleanupPeriodDays": 30,
  "respectGitignore": true,
  "includeGitInstructions": true,
  "autoUpdatesChannel": "stable"
}
```

---

## 三、项目共享配置（.claude/settings.json）

适用：团队统一的权限规则、MCP 服务器、工具白名单。**提交到 git**，不含敏感信息。

```jsonc
// <repo>/.claude/settings.json
{
  // 模型偏好（可选，不含认证信息）
  "model": "claude-sonnet-4-6",

  // 项目允许的工具操作
  "permissions": {
    "allow": [
      "bash(git *)",
      "bash(npm *)",
      "bash(pnpm *)",
      "bash(yarn *)",
      "bash(python *)",
      "bash(pytest *)",
      "read(**)",
      "write(src/**)",
      "write(tests/**)",
      "write(docs/**)"
    ],
    "deny": [
      "bash(rm -rf /)",
      "bash(sudo rm *)",
      "bash(curl * | bash)"
    ]
  },

  // 项目 MCP 服务器（团队共享）
  "enableAllProjectMcpServers": false,
  "enabledMcpjsonServers": ["github", "filesystem"],

  // 钩子示例：提交前 lint
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "bash(git commit*)",
        "hooks": [
          { "type": "command", "command": "npm run lint --if-present" }
        ]
      }
    ]
  },

  "respectGitignore": true,
  "includeGitInstructions": true,
  "cleanupPeriodDays": 7
}
```

---

## 四、本地私有配置（.claude/settings.local.json）

适用：个人在项目中的覆盖配置，**不提交 git**（应在 .gitignore 中排除）。

```jsonc
// <repo>/.claude/settings.local.json
{
  // 本地测试时放宽权限
  "permissions": {
    "defaultMode": "bypassPermissions"
  },

  // 本地调试用环境变量（不含真实密钥）
  "env": {
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "64000",
    "DISABLE_PROMPT_CACHING": "1"
  },

  // Worktree 本地优化
  "worktree": {
    "symlinkDirectories": ["node_modules", ".venv", ".cache"]
  }
}
```

确保 .gitignore 包含：

```gitignore
.claude/settings.local.json
.claude/*.local.json
```

---

## 五、企业管控配置（managed-settings.json）

适用：IT 管理员统一部署，员工无法覆盖。

**文件位置：**
- Windows：`C:\ProgramData\ClaudeCode\managed-settings.json`
- macOS：`/Library/Application Support/ClaudeCode/managed-settings.json`
- Linux：`/etc/claude-code/managed-settings.json`

```jsonc
// managed-settings.json（IT 管理员部署）
{
  // 强制走企业内部 API 网关
  "env": {
    "ANTHROPIC_BASE_URL": "https://api-gateway.corp.internal/v1",
    "ANTHROPIC_AUTH_TOKEN": "corp-managed-token",
    "HTTPS_PROXY": "http://proxy.corp.com:8080",
    "NO_PROXY": "localhost,127.0.0.1,.corp.internal",
    "NODE_EXTRA_CA_CERTS": "/etc/ssl/certs/corp-ca.pem",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "DISABLE_TELEMETRY": "1"
  },

  // 固定可用模型（员工只能选这些）
  "availableModels": [
    "claude-sonnet-4-6",
    "claude-haiku-4-5-20251001"
  ],

  // 权限锁定
  "allowManagedPermissionRulesOnly": true,
  "permissions": {
    "defaultMode": "default",
    "deny": [
      "bash(curl * | sh)",
      "bash(wget * | sh)",
      "bash(sudo *)"
    ]
  },

  // Hook 锁定（只允许企业定义的 hook）
  "allowManagedHooksOnly": true,

  // MCP 锁定（只允许企业批准的服务器）
  "allowManagedMcpServersOnly": true,
  "allowedMcpServers": [
    { "name": "github-enterprise", "url": "https://mcp.corp.internal/github" }
  ],

  // 网络域名锁定
  "allowManagedDomainsOnly": true,

  // 强制登录方式
  "forceLoginMethod": "console",
  "forceLoginOrgUUID": "org-your-corp-uuid",

  // 公司公告
  "companyAnnouncements": [
    { "message": "请使用内部代理，禁止直连外网 API", "type": "warning" }
  ],

  // 历史保留
  "cleanupPeriodDays": 90
}
```

---

## 六、VS Code 插件推荐配置

### 6.1 个人开发（用户 settings.json）

```jsonc
// VS Code 用户 settings.json（Ctrl+Shift+P → Open User Settings JSON）
{
  "claudeCode.environmentVariables": [
    { "name": "ANTHROPIC_AUTH_TOKEN",             "value": "your-token-here" },
    { "name": "ANTHROPIC_BASE_URL",               "value": "https://your-proxy.com/v1" },
    { "name": "ANTHROPIC_DEFAULT_SONNET_MODEL",   "value": "claude-sonnet-4-6" },
    { "name": "ANTHROPIC_DEFAULT_HAIKU_MODEL",    "value": "claude-haiku-4-5-20251001" },
    { "name": "ANTHROPIC_DEFAULT_OPUS_MODEL",     "value": "claude-opus-4-6" },
    { "name": "DISABLE_PROMPT_CACHING",           "value": "1" },
    { "name": "DISABLE_TELEMETRY",                "value": "1" }
  ],

  "claudeCode.disableLoginPrompt":             true,
  "claudeCode.selectedModel":                  "claude-sonnet-4-6",
  "claudeCode.preferredLocation":              "panel",
  "claudeCode.initialPermissionMode":          "default",
  "claudeCode.autosave":                       true,
  "claudeCode.respectGitIgnore":               true,
  "claudeCode.allowDangerouslySkipPermissions": false
}
```

### 6.2 项目共享（.vscode/settings.json）

```jsonc
// <repo>/.vscode/settings.json（可提交 git，不含密钥）
{
  "claudeCode.selectedModel":         "claude-sonnet-4-6",
  "claudeCode.preferredLocation":     "panel",
  "claudeCode.initialPermissionMode": "default",
  "claudeCode.respectGitIgnore":      true,
  "claudeCode.autosave":              true,

  // 不在这里放 environmentVariables（含密钥）
  // 密钥通过 ~/.claude/settings.json 或用户级 VS Code settings 注入
  "claudeCode.environmentVariables": []
}
```

### 6.3 最简认证配置（第三方中转最小集）

```json
{
  "claudeCode.environmentVariables": [
    { "name": "ANTHROPIC_AUTH_TOKEN",           "value": "your-token" },
    { "name": "ANTHROPIC_BASE_URL",             "value": "https://your-proxy.com/v1" },
    { "name": "ANTHROPIC_DEFAULT_SONNET_MODEL", "value": "claude-sonnet-4-6" },
    { "name": "ANTHROPIC_DEFAULT_HAIKU_MODEL",  "value": "claude-haiku-4-5-20251001" }
  ],
  "claudeCode.disableLoginPrompt": true
}
```

---

## 七、安全 API Key 管理（apiKeyHelper）

**推荐用于团队/生产环境**，避免明文 Key 存在配置文件。

```jsonc
// ~/.claude/settings.json
{
  "apiKeyHelper": "~/.claude/scripts/get-token.sh"
}
```

```bash
#!/bin/bash
# ~/.claude/scripts/get-token.sh
# 从密钥管理服务获取临时 Token
curl -s "https://vault.internal.com/v1/secret/claude-token" \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  | jq -r '.data.token'
```

Token 刷新逻辑：
- 默认每 5 分钟（300000ms）刷新一次
- 收到 HTTP 401 时立即刷新
- 自定义刷新间隔：`CLAUDE_CODE_API_KEY_HELPER_TTL_MS=600000`
