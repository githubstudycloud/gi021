# 项目级配置隔离：团队共享 vs 个人私有 vs 多环境

---

## 一、隔离设计原则

```
共享给所有人 → .claude/settings.json（提交 git）
              .mcp.json（提交 git）
              CLAUDE.md（提交 git）

个人私有     → .claude/settings.local.json（gitignore）
              CLAUDE.local.md（gitignore）
              ~/.claude/settings.json（用户全局）

环境相关     → .claude/settings.dev.json（可提交，无密钥）
              .claude/settings.prod.json（可提交，无密钥）
              由启动脚本选择加载
```

---

## 二、标准项目配置结构

### 2.1 文件职责划分

```
<repo>/
├── .claude/
│   ├── settings.json         ← 团队共享：工具权限、MCP、Hooks、代码规范
│   ├── settings.local.json   ← 个人私有：本地密钥、个人偏好（gitignore）
│   └── commands/             ← 团队共享的自定义命令
│       ├── review.md
│       ├── fix-issue.md
│       └── release.md
├── .mcp.json                 ← 团队共享的 MCP 配置（无密钥）
├── CLAUDE.md                 ← 团队共享的项目记忆（提交 git）
└── CLAUDE.local.md           ← 个人笔记（gitignore）
```

### 2.2 `.gitignore` 必备项

```gitignore
# Claude Code 私有配置
.claude/settings.local.json
.claude/*.local.json
CLAUDE.local.md

# 永远不提交用户全局配置
# （~/.claude/settings.json 已在 Home 目录，本身不在仓库中）
```

---

## 三、团队共享配置（`.claude/settings.json`）

**原则**：只写不含密钥的内容。

```jsonc
// .claude/settings.json
{
  // 团队约定的默认模型
  "model": "claude-sonnet-4-6",
  "effortLevel": "medium",

  // 限制可用模型（防止误用高价模型）
  "availableModels": [
    "claude-sonnet-4-6",
    "claude-haiku-4-5-20251001"
  ],

  // 团队统一权限策略
  "permissions": {
    "defaultMode": "default",
    "allow": [
      "Bash(git *)",
      "Bash(npm *)",
      "Bash(pnpm *)",
      "Bash(yarn *)",
      "Bash(npx *)",
      "Bash(node *)"
    ],
    "deny": [
      "Bash(rm -rf /)",
      "Bash(sudo *)",
      "Bash(curl * | bash)",
      "Bash(wget * | sh)"
    ]
  },

  // 团队共享 Hooks（代码格式化、审计日志）
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{
          "type": "command",
          "command": "cd \"$CLAUDE_PROJECT_DIR\" && npx prettier --write \"$TOOL_INPUT_path\" 2>/dev/null || true"
        }]
      }
    ]
  },

  // 共享的行为配置（不含密钥）
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6",
    "DISABLE_TELEMETRY": "1",
    "DISABLE_AUTOUPDATER": "1"
  },

  // 工作区配置
  "respectGitignore": true,
  "cleanupPeriodDays": 14
}
```

---

## 四、个人私有配置（`.claude/settings.local.json`）

**原则**：密钥、个人偏好、不适合团队的本地设置。

```jsonc
// .claude/settings.local.json（不提交 git）
{
  // 个人认证信息
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "your-personal-token",
    "ANTHROPIC_BASE_URL": "https://your-proxy.com/v1",

    // 个人代理（可能与团队成员不同）
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "HTTP_PROXY": "http://127.0.0.1:7890",
    "NO_PROXY": "localhost,127.0.0.1"
  },

  // 个人权限偏好（比团队配置更宽松）
  "permissions": {
    "defaultMode": "bypassPermissions"   // 个人开发机不需要每次确认
  }
}
```

---

## 五、多环境配置策略

### 5.1 基于文件的多环境

为不同环境维护不同的配置文件（无密钥，可提交）：

```
.claude/
├── settings.json           ← 基础配置（所有环境共享）
├── settings.dev.json       ← 开发环境覆盖
├── settings.staging.json   ← 测试环境覆盖
└── settings.prod.json      ← 生产环境覆盖（更严格权限）
```

```jsonc
// .claude/settings.dev.json
{
  "model": "claude-haiku-4-5-20251001",   // 开发阶段用便宜模型
  "permissions": {
    "defaultMode": "bypassPermissions"    // 开发环境跳过确认
  },
  "env": {
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "8000"
  }
}
```

```jsonc
// .claude/settings.prod.json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "defaultMode": "default",             // 生产环境每次确认
    "deny": [
      "Bash(git push *)",                 // 禁止直接推送
      "Bash(kubectl delete *)",           // 禁止删除 k8s 资源
      "Bash(terraform destroy *)"
    ]
  }
}
```

### 5.2 环境切换脚本

```bash
#!/bin/bash
# scripts/use-env.sh
ENV="${1:-dev}"
CLAUDE_DIR=".claude"

case "$ENV" in
  dev|staging|prod)
    cp "$CLAUDE_DIR/settings.${ENV}.json" "$CLAUDE_DIR/settings.local.json"
    echo "已切换到 $ENV 环境配置"
    ;;
  *)
    echo "用法: $0 [dev|staging|prod]"
    echo "当前可用环境:"
    ls "$CLAUDE_DIR"/settings.*.json | sed 's/.*settings\.\(.*\)\.json/  \1/'
    exit 1
    ;;
esac
```

```bash
# 切换环境
./scripts/use-env.sh dev
./scripts/use-env.sh prod
```

---

## 六、Worktree 项目隔离

每个 Worktree 有独立的 `.claude/settings.local.json`，实现不同功能分支使用不同模型：

```bash
# 创建并行 Worktree
git worktree add ../myapp-feature-a feature/user-auth
git worktree add ../myapp-feature-b feature/payment
```

```jsonc
// ../myapp-feature-a/.claude/settings.local.json
{
  "model": "claude-haiku-4-5-20251001",  // 快速迭代用 Haiku
  "env": { "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001" }
}
```

```jsonc
// ../myapp-feature-b/.claude/settings.local.json
{
  "model": "claude-opus-4-6",            // 复杂支付逻辑用 Opus
  "env": { "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6" }
}
```

---

## 七、CLAUDE.md 的项目隔离

### 团队共享的 `CLAUDE.md`

```markdown
# MyApp 项目规范

## 架构说明
- 前端：React 19 + TypeScript，状态管理用 Zustand
- 后端：Node.js + Express，路由在 src/routes/
- 数据库：PostgreSQL + Prisma ORM

## 常用命令
```bash
pnpm dev          # 启动开发服务器
pnpm test         # 运行测试套件
pnpm build        # 构建生产版本
pnpm db:migrate   # 运行数据库迁移
```

## 代码规范
- 文件长度 < 300 行，函数长度 < 50 行
- 所有新函数必须有单元测试
- 不要修改 src/generated/（自动生成）

## 禁止事项
- 不要直接操作生产数据库
- 不要将密钥写入代码
- 不要修改 package-lock.json（使用 pnpm）
```

### 个人私有的 `CLAUDE.local.md`

```markdown
# 我的本地工作笔记

## 本地环境
- 数据库运行在 localhost:5433（与默认端口不同）
- 测试账号：test@local.dev / password123

## 我的工作习惯
- 代码审查结果写到 ~/Desktop/reviews/
- 不要自动提交，等我确认

## 当前任务进度
- 认证模块已完成 JWT 发放，还需要做 refresh token
```

---

## 八、企业级团队配置方案

### 8.1 仓库结构

```
company-claude-config/          ← 独立仓库，团队成员克隆
├── base/
│   └── settings.json           ← 基础安全策略
├── teams/
│   ├── frontend/settings.json  ← 前端团队配置
│   ├── backend/settings.json   ← 后端团队配置
│   └── devops/settings.json    ← DevOps 团队配置（更多权限）
└── install.sh                  ← 安装脚本
```

```bash
#!/bin/bash
# install.sh
TEAM="${1:-base}"
TARGET_DIR="$HOME/.claude"
mkdir -p "$TARGET_DIR"

# 合并基础配置和团队配置
jq -s '.[0] * .[1]' \
  "base/settings.json" \
  "teams/$TEAM/settings.json" \
  > "$TARGET_DIR/settings.json"

echo "已安装 $TEAM 团队配置到 ~/.claude/settings.json"
echo "请在 settings.local.json 中添加个人密钥"
```

### 8.2 Managed Settings（企业推送）

企业可以通过 MDM 或集中管理将 `managed-settings.json` 推送到所有设备：

```
macOS:   /Library/Application Support/claude-code/managed-settings.json
Linux:   /etc/claude-code/managed-settings.json
Windows: C:\ProgramData\claude-code\managed-settings.json
```

```jsonc
// managed-settings.json（不可被用户覆盖）
{
  "allowManagedPermissionRulesOnly": true,
  "allowManagedMcpServersOnly": true,
  "permissions": {
    "deny": [
      "Bash(curl * | bash)",
      "Bash(wget * | sh)",
      "Bash(nc *)",
      "WebFetch(*.competitor.com)"
    ]
  },
  "companyAnnouncements": [
    { "message": "请使用内网代理 https://claude-proxy.internal.com/v1", "type": "info" }
  ]
}
```
