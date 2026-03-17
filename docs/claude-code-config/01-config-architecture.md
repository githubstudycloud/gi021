# 配置层级架构：合并规则与调试诊断

---

## 一、四层配置文件全景

```
优先级 低 ←─────────────────────────────────── 高 →

[托管层]           [用户层]           [项目层]           [本地层]
managed-           ~/.claude/         <repo>/.claude/    <repo>/.claude/
settings.json      settings.json      settings.json      settings.local.json
（企业推送）        （个人全局）        （团队共享）        （本人私有）
不可被覆盖         可被项目覆盖        可被本地覆盖        最终生效
```

外加运行时覆盖（优先级最高）：
```
CLI 参数（--model opus）
环境变量（ANTHROPIC_MODEL=...）
会话命令（/model haiku）
```

---

## 二、合并规则详解

### 规则 1：数组字段——跨层合并（累加）

以下字段会把所有层的值**合并**在一起：

| 字段 | 合并行为 |
|------|---------|
| `permissions.allow` | 所有层的允许规则合并为并集 |
| `permissions.deny` | 所有层的拒绝规则合并（拒绝优先） |
| `hooks.PreToolUse` | 所有层的 Hook 列表合并执行 |
| `hooks.PostToolUse` | 同上 |
| `mcpServers` | 所有层的 MCP 服务器合并 |

**示例**：

```jsonc
// ~/.claude/settings.json（用户层）
{
  "permissions": {
    "allow": ["Bash(git *)"]
  }
}

// .claude/settings.json（项目层）
{
  "permissions": {
    "allow": ["Bash(npm *)"]
  }
}

// 最终生效的 allow 列表：["Bash(git *)", "Bash(npm *)"]
```

### 规则 2：标量字段——高优先级覆盖

以下字段遵循**高优先级完全覆盖**低优先级：

| 字段 | 覆盖行为 |
|------|---------|
| `model` | 项目层覆盖用户层 |
| `effortLevel` | 项目层覆盖用户层 |
| `env` 中的各个键 | 同名 key 高优先级覆盖低优先级 |
| `cleanupPeriodDays` | 最近层生效 |
| `skipWebFetchPreflight` | 最近层生效 |

**示例**：

```jsonc
// ~/.claude/settings.json
{ "model": "claude-sonnet-4-6", "env": { "ANTHROPIC_BASE_URL": "https://global-proxy.com/v1" } }

// .claude/settings.json（项目层）
{ "model": "claude-opus-4-6", "env": { "ANTHROPIC_BASE_URL": "https://project-proxy.com/v1" } }

// 最终生效：model=claude-opus-4-6，BASE_URL=https://project-proxy.com/v1
```

### 规则 3：env 字段的继承与覆盖

`env` 是对象（`{}`），每个 key 独立继承：

```jsonc
// 用户层
{ "env": { "ANTHROPIC_AUTH_TOKEN": "global-token", "DISABLE_TELEMETRY": "1" } }

// 项目层
{ "env": { "ANTHROPIC_BASE_URL": "https://project.com/v1" } }

// 本地层
{ "env": { "ANTHROPIC_AUTH_TOKEN": "local-token" } }

// 最终生效（三层合并，同名 key 以最近层为准）：
{
  "ANTHROPIC_AUTH_TOKEN": "local-token",    // 本地层覆盖
  "DISABLE_TELEMETRY": "1",                 // 来自用户层（未被覆盖）
  "ANTHROPIC_BASE_URL": "https://project.com/v1"  // 来自项目层
}
```

---

## 三、配置文件加载顺序（完整流程）

```
1. 加载 managed-settings.json（企业推送，不可被覆盖的策略）
2. 加载 ~/.claude/settings.json（用户全局）
3. 找到当前工作目录，向上遍历所有父目录，依次加载 .claude/settings.json
4. 加载当前项目目录的 .claude/settings.local.json
5. 读取环境变量（覆盖所有文件配置）
6. 读取 CLI 参数（最高优先级）
```

> 第 3 步的"向上遍历"意味着：如果你的项目在 `/home/user/projects/myapp/`，
> Claude 会加载 `/home/user/projects/myapp/.claude/settings.json`、
> `/home/user/projects/.claude/settings.json`（如果存在）、
> `/home/user/.claude/settings.json` 等。

---

## 四、settings.json 完整字段速查

```jsonc
{
  // ═══════════════════════════════════════
  // 认证
  // ═══════════════════════════════════════
  "apiKeyHelper": "/path/to/get-token.sh",   // 动态获取 key 的脚本
  "forceLoginMethod": "console",             // "claudeai" | "console"
  "forceLoginOrgUUID": "org-xxxxxxxx",       // 强制使用指定组织

  // ═══════════════════════════════════════
  // 模型
  // ═══════════════════════════════════════
  "model": "claude-sonnet-4-6",              // 默认模型
  "effortLevel": "medium",                   // "low" | "medium" | "high"
  "availableModels": [                       // 限制可选模型列表
    "claude-sonnet-4-6",
    "claude-haiku-4-5-20251001"
  ],
  "modelOverrides": {                        // 模型 ID 映射（Bedrock ARN 等）
    "claude-opus-4-6": "us.anthropic.claude-opus-4-6-20250514-v1:0"
  },

  // ═══════════════════════════════════════
  // 环境变量（注入给 Claude 进程）
  // ═══════════════════════════════════════
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "...",
    "ANTHROPIC_BASE_URL": "https://proxy.com/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "128000",
    "DISABLE_PROMPT_CACHING": "1",
    "DISABLE_TELEMETRY": "1"
  },

  // ═══════════════════════════════════════
  // 权限
  // ═══════════════════════════════════════
  "permissions": {
    "defaultMode": "default",               // "bypassPermissions" | "acceptEdits" | "plan" | "default"
    "allow": ["Bash(git *)", "Bash(npm *)"],
    "deny": ["Bash(rm -rf *)"],
    "additionalDirectories": ["/data/shared"]
  },

  // ═══════════════════════════════════════
  // Hooks
  // ═══════════════════════════════════════
  "hooks": {
    "PreToolUse": [{ "matcher": "Bash", "hooks": [{ "type": "command", "command": "..." }] }],
    "PostToolUse": [],
    "Stop": [],
    "UserPromptSubmit": []
  },
  "disableAllHooks": false,

  // ═══════════════════════════════════════
  // MCP
  // ═══════════════════════════════════════
  "enableAllProjectMcpServers": false,
  "enabledMcpjsonServers": ["github"],
  "disabledMcpjsonServers": [],

  // ═══════════════════════════════════════
  // 工作区
  // ═══════════════════════════════════════
  "cleanupPeriodDays": 30,
  "respectGitignore": true,
  "includeGitInstructions": true,
  "skipWebFetchPreflight": false,
  "skipDangerousModePermissionPrompt": false,

  // ═══════════════════════════════════════
  // Worktree
  // ═══════════════════════════════════════
  "worktree": {
    "symlinkDirectories": ["node_modules", ".venv"],
    "sparsePaths": ["src/", "docs/"]
  },

  // ═══════════════════════════════════════
  // 外观
  // ═══════════════════════════════════════
  "language": "chinese",
  "showTurnDuration": true,

  // ═══════════════════════════════════════
  // 企业托管专用（仅 managed-settings.json）
  // ═══════════════════════════════════════
  "allowManagedHooksOnly": true,
  "allowManagedPermissionRulesOnly": true,
  "allowManagedMcpServersOnly": true,
  "companyAnnouncements": [{ "message": "请使用内网代理", "type": "info" }]
}
```

---

## 五、调试配置问题

### 5.1 查看当前生效配置

```bash
# 方式 1：在 Claude 会话中查看
> /status

# 方式 2：查看 CLI 配置
claude config list

# 方式 3：查看环境变量是否正确注入
# 在 Claude 会话中执行：
env | grep -E "ANTHROPIC|CLAUDE|PROXY|NODE_TLS"
```

### 5.2 测试端点连通性

```bash
# 直接用 curl 测试，不依赖 Claude Code
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}' \
  https://your-proxy.com/v1/messages
# 期望返回 200
```

### 5.3 排查配置优先级冲突

```bash
# 查看所有生效的配置文件（包含路径）
cat ~/.claude/settings.json
cat .claude/settings.json       # 项目层（如果存在）
cat .claude/settings.local.json # 本地层（如果存在）

# 检查环境变量是否意外覆盖了文件配置
echo $ANTHROPIC_BASE_URL
echo $ANTHROPIC_AUTH_TOKEN
```

### 5.4 VS Code 插件日志

```
命令面板（Ctrl/Cmd+Shift+P）→ Claude Code: Show Logs
```

### 5.5 常见问题快速排查表

| 症状 | 可能原因 | 排查命令 |
|------|---------|---------|
| 401 Unauthorized | Token 错误或 Header 类型不匹配 | `curl` 测试 |
| 404 model not found | 模型别名漂移，未固定 | 检查 `ANTHROPIC_DEFAULT_*` 变量 |
| VS Code 仍弹登录框 | 缺少 `disableLoginPrompt: true` | VS Code settings.json |
| 配置不生效 | 环境变量覆盖了文件配置 | `echo $ANTHROPIC_BASE_URL` |
| 流式响应截断 | 代理超时/缓冲 | 联系中转服务商 |
| 自签名证书错误 | TLS 验证失败 | `NODE_TLS_REJECT_UNAUTHORIZED: "0"` |
