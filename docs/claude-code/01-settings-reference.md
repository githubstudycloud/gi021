# Claude Code 配置参考手册

> 版本：v2.1.76 | 来源：官方文档 + 本地扩展 package.json / schema 分析
> 更新：2026-03-16

---

## 一、配置层级与优先级

Claude Code 有四层配置文件，优先级从高到低：

```
Managed（企业管理）
    ↓ 覆盖
User（用户全局）   ~/.claude/settings.json
    ↓ 覆盖
Project（项目共享） <repo>/.claude/settings.json
    ↓ 覆盖
Local（本地私有）  <repo>/.claude/settings.local.json
```

除文件外还有：
- **CLI 参数** (`--model`, `--effort`)：覆盖所有文件配置
- **环境变量** (`ANTHROPIC_MODEL` 等)：覆盖 settings.json，但低于 CLI 参数
- **会话内命令** (`/model`, `/effort`)：仅影响当前对话

完整优先级（高→低）：
```
会话命令 > CLI 参数 > 环境变量 > Local > Project > User > Managed > 默认值
```

> **注意**：`permissions`、`hooks`、`mcpServers` 等数组类配置会**跨层合并**，
> 而 `env`、`model` 等标量配置**高优先级覆盖低优先级**。

---

## 二、settings.json 完整字段说明

### 2.1 认证与连接

```jsonc
{
  // 执行此脚本获取临时 API Key（推荐安全方案）
  "apiKeyHelper": "/path/to/get-token.sh",

  // 强制使用指定登录方式："claudeai" | "console"
  "forceLoginMethod": "console",

  // 自动选择指定组织 UUID（OAuth 场景）
  "forceLoginOrgUUID": "org-xxxxxxxx"
}
```

### 2.2 模型配置

```jsonc
{
  // 默认模型（别名或完整 ID）
  "model": "claude-sonnet-4-6",

  // 限制可选模型列表（留空=不限制）
  "availableModels": [
    "claude-sonnet-4-6",
    "claude-haiku-4-5-20251001",
    "claude-opus-4-6"
  ],

  // 将 Anthropic 模型 ID 映射到厂商专属 ID（Bedrock ARN 等）
  "modelOverrides": {
    "claude-opus-4-6": "us.anthropic.claude-opus-4-6-20250514-v1:0",
    "claude-sonnet-4-6": "us.anthropic.claude-sonnet-4-6-20250514-v1:0"
  },

  // 持久化努力级别："low" | "medium" | "high"
  "effortLevel": "medium"
}
```

### 2.3 环境变量

```jsonc
{
  "env": {
    // 认证（二选一）
    "ANTHROPIC_AUTH_TOKEN": "...",   // Authorization: Bearer
    "ANTHROPIC_API_KEY":   "...",   // X-Api-Key

    // 自定义端点
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",

    // 模型别名固定（使用中转时必须设置）
    "ANTHROPIC_DEFAULT_OPUS_MODEL":   "claude-opus-4-6",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL":  "claude-haiku-4-5-20251001",

    // 代理
    "HTTPS_PROXY": "http://proxy.example.com:8080",
    "HTTP_PROXY":  "http://proxy.example.com:8080",
    "NO_PROXY":    "localhost,127.0.0.1,.internal.com",

    // TLS（自签名证书时）
    "NODE_EXTRA_CA_CERTS":           "/path/to/ca.pem",
    "NODE_TLS_REJECT_UNAUTHORIZED":  "0",   // 仅开发环境

    // 行为控制
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS":            "32000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "DISABLE_PROMPT_CACHING":                   "1",
    "DISABLE_AUTOUPDATER":                      "1",
    "DISABLE_TELEMETRY":                        "1",
    "DISABLE_ERROR_REPORTING":                  "1",
    "DISABLE_BUG_COMMAND":                      "1"
  }
}
```

### 2.4 权限系统

```jsonc
{
  "permissions": {
    // 默认模式："bypassPermissions" | "acceptEdits" | "plan" | "default"
    "defaultMode": "default",

    // 允许列表（无需确认直接执行）
    "allow": [
      "bash(git *)",
      "bash(npm *)",
      "read(**)",
      "write(src/**)"
    ],

    // 拒绝列表（永远拒绝）
    "deny": [
      "bash(rm -rf *)",
      "bash(sudo *)"
    ],

    // 追加可访问目录
    "additionalDirectories": [
      "/data/shared"
    ]
  }
}
```

权限规则语法：
| 模式 | 说明 |
|------|------|
| `bash(git *)` | 允许/拒绝 bash 执行 git 开头的命令 |
| `read(**/secret*)` | 文件读权限（支持 glob） |
| `write(src/**)` | 文件写权限 |
| `mcp__server__tool` | MCP 工具权限 |

### 2.5 Hooks（钩子）

```jsonc
{
  "hooks": {
    // 工具执行前后触发
    "PreToolUse": [
      {
        "matcher": "bash",
        "hooks": [
          { "type": "command", "command": "echo 'pre-bash hook'" }
        ]
      }
    ],
    "PostToolUse": [],

    // 会话级别
    "UserPromptSubmit": [],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "/path/to/notify.sh" }
        ]
      }
    ]
  },

  // 禁用所有 Hooks
  "disableAllHooks": false,

  // HTTP Hook 允许访问的 URL 模式
  "allowedHttpHookUrls": [
    "https://api.internal.com/*"
  ],

  // HTTP Hook 可读取的环境变量白名单
  "httpHookAllowedEnvVars": ["CI_TOKEN", "BUILD_ENV"]
}
```

### 2.6 MCP 服务器

```jsonc
{
  // 自动信任项目 .mcp.json 中所有服务器
  "enableAllProjectMcpServers": false,

  // 仅信任指定服务器
  "enabledMcpjsonServers": ["github", "postgres"],

  // 拒绝指定服务器
  "disabledMcpjsonServers": ["untrusted-server"]
}
```

### 2.7 工作区与会话

```jsonc
{
  // 历史清理天数（0 = 不清理）
  "cleanupPeriodDays": 30,

  // 计划文件存放路径
  "plansDirectory": "~/.claude/plans",

  // 自动内存目录
  "autoMemoryDirectory": "~/.claude/memory",

  // 是否遵守 .gitignore
  "respectGitignore": true,

  // 是否包含内置 git 工作流指引
  "includeGitInstructions": true,

  // 更新通道："stable" | "latest"
  "autoUpdatesChannel": "stable",

  // 跳过 web fetch 预检
  "skipWebFetchPreflight": false,

  // 跳过危险模式权限提示
  "skipDangerousModePermissionPrompt": false
}
```

### 2.8 Worktree（工作树）

```jsonc
{
  "worktree": {
    // 需要软链接共享的目录（避免复制大文件）
    "symlinkDirectories": ["node_modules", ".venv"],

    // git sparse-checkout 仅检出指定路径
    "sparsePaths": ["src/", "docs/"]
  }
}
```

### 2.9 企业/托管专用字段

```jsonc
{
  // 以下字段仅在 managed-settings.json 中有效

  // 仅允许托管 Hooks
  "allowManagedHooksOnly": true,

  // 仅允许托管权限规则
  "allowManagedPermissionRulesOnly": true,

  // 仅允许托管 MCP 服务器
  "allowManagedMcpServersOnly": true,

  // 仅允许托管域名
  "allowManagedDomainsOnly": true,

  // 强制 OAuth 组织
  "forceLoginOrgUUID": "org-xxxxxxxx",

  // 允许的 MCP 服务器列表
  "allowedMcpServers": [],

  // 公司启动公告
  "companyAnnouncements": [
    { "message": "请使用内网代理访问 API", "type": "info" }
  ]
}
```

### 2.10 外观与输出

```jsonc
{
  // 首选语言
  "language": "chinese",

  // Git 提交归属
  "attribution": {
    "commit": "Co-Authored-By: Claude <noreply@anthropic.com>",
    "pr": "Generated with Claude Code"
  },

  // 显示每轮耗时
  "showTurnDuration": true,

  // 减少动画（无障碍）
  "prefersReducedMotion": false
}
```

---

## 三、VS Code 插件专属设置

插件设置存放在 VS Code 的 `settings.json`（不是 `~/.claude/settings.json`）。

### 3.1 完整设置项

| 设置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `claudeCode.selectedModel` | string | `"default"` | 当前会话使用的模型 |
| `claudeCode.environmentVariables` | array | `[]` | 注入给 Claude 进程的环境变量 |
| `claudeCode.useTerminal` | boolean | `false` | 使用终端模式而非图形面板 |
| `claudeCode.allowDangerouslySkipPermissions` | boolean | `false` | 跳过所有权限确认（仅沙箱环境） |
| `claudeCode.claudeProcessWrapper` | string | `""` | 启动 Claude 进程的包装器可执行文件 |
| `claudeCode.respectGitIgnore` | boolean | `true` | @文件选择器遵守 .gitignore |
| `claudeCode.initialPermissionMode` | string | `"default"` | 初始权限模式 |
| `claudeCode.disableLoginPrompt` | boolean | `false` | 跳过登录提示（第三方认证必须设为 true） |
| `claudeCode.autosave` | boolean | `true` | Claude 读写文件前自动保存 |
| `claudeCode.useCtrlEnterToSend` | boolean | `false` | Ctrl+Enter 发送（默认 Enter） |
| `claudeCode.preferredLocation` | string | `"panel"` | 打开位置：`sidebar` / `panel` |
| `claudeCode.enableNewConversationShortcut` | boolean | `true` | Cmd/Ctrl+N 新建对话 |
| `claudeCode.hideOnboarding` | boolean | `false` | 隐藏入门引导清单 |
| `claudeCode.usePythonEnvironment` | boolean | `true` | 自动激活 Python 环境（需 Python 插件） |

### 3.2 `environmentVariables` 格式

**类型：数组，每项为 `{name, value}` 对象**（源码验证）

```json
"claudeCode.environmentVariables": [
  { "name": "ANTHROPIC_AUTH_TOKEN",  "value": "sk-ant-..." },
  { "name": "ANTHROPIC_BASE_URL",    "value": "https://your-proxy.com/v1" }
]
```

> ⚠️ 官方文档建议优先使用 `~/.claude/settings.json` 的 `env` 字段，
> `claudeCode.environmentVariables` 作为 VS Code 专属补充。

### 3.3 `initialPermissionMode` 枚举值

| 值 | 说明 |
|----|------|
| `"default"` | 每次操作询问 |
| `"acceptEdits"` | 自动接受文件编辑，其他询问 |
| `"plan"` | 仅展示计划，不执行 |
| `"bypassPermissions"` | 跳过所有确认（危险） |

---

## 四、环境变量完整速查表

### 4.1 认证

| 变量 | Header | 说明 |
|------|--------|------|
| `ANTHROPIC_AUTH_TOKEN` | `Authorization: Bearer` | CLI 交互推荐 |
| `ANTHROPIC_API_KEY` | `X-Api-Key` | SDK / 第三方中转 |

### 4.2 端点

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_BASE_URL` | 通用自定义端点 |
| `ANTHROPIC_BEDROCK_BASE_URL` | Bedrock 网关 |
| `ANTHROPIC_VERTEX_BASE_URL` | Vertex AI 网关 |
| `ANTHROPIC_FOUNDRY_BASE_URL` | Microsoft Foundry 网关 |

### 4.3 模型别名

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_MODEL` | 覆盖默认模型 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | 固定 opus 别名 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | 固定 sonnet 别名 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | 固定 haiku 别名 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子 Agent 模型 |
| `CLAUDE_CODE_EFFORT_LEVEL` | 推理强度 |

### 4.4 云厂商

| 变量 | 值 | 说明 |
|------|-----|------|
| `CLAUDE_CODE_USE_BEDROCK` | `1` | 启用 AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | `1` | 启用 Google Vertex AI |
| `CLAUDE_CODE_USE_FOUNDRY` | `1` | 启用 Microsoft Foundry |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | `1` | 跳过 AWS 认证 |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | `1` | 跳过 GCP 认证 |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | `1` | 跳过 Azure 认证 |
| `AWS_REGION` | `us-east-1` | Bedrock 区域 |
| `CLOUD_ML_REGION` | `us-east5` | Vertex AI 区域 |

### 4.5 网络

| 变量 | 说明 |
|------|------|
| `HTTPS_PROXY` | HTTPS 代理 |
| `HTTP_PROXY` | HTTP 代理 |
| `NO_PROXY` | 代理排除列表 |
| `NODE_EXTRA_CA_CERTS` | 自定义 CA 证书路径 |
| `NODE_TLS_REJECT_UNAUTHORIZED` | `0` 跳过 TLS 验证 |

### 4.6 行为控制

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 最大输出 token |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 上下文压缩阈值 1-100 |
| `BASH_MAX_TIMEOUT_MS` | Bash 超时 ms |
| `DISABLE_PROMPT_CACHING` | 禁用 Prompt 缓存 |
| `DISABLE_PROMPT_CACHING_OPUS/SONNET/HAIKU` | 按模型禁用缓存 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | `1` 禁用自适应推理 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `1` 禁用非必要网络请求 |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | Key 刷新间隔 ms |
| `DISABLE_AUTOUPDATER` | `1` 禁用自动更新 |
| `DISABLE_TELEMETRY` | `1` 禁用遥测 |
| `DISABLE_ERROR_REPORTING` | `1` 禁用错误上报 |
| `DISABLE_BUG_COMMAND` | `1` 禁用 /bug 命令 |

---

*→ 配置推荐模板见 [02-recommended-configs.md](./02-recommended-configs.md)*
*→ 第三方模型注意事项见 [03-third-party-models.md](./03-third-party-models.md)*
*→ 多模型 / 多实例方案见 [04-multi-model.md](./04-multi-model.md)*
*→ 环境变量管理策略见 [05-env-management.md](./05-env-management.md)*
