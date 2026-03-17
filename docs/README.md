# Claude Code 文档总览

> 三个目录，涵盖从入门到进阶的完整知识体系。
> 版本：v2.1.76 | 更新：2026-03-17

---

## 文档目录结构

```
docs/
├── README.md                    ← 本文：总入口与快速导航
│
├── claude-code/                 ← 配置参考手册（精确、权威）
│   ├── 01-settings-reference    字段全集：settings.json + VS Code 设置 + 环境变量
│   ├── 02-recommended-configs   推荐模板：个人 / 项目 / 企业 / VS Code
│   ├── 03-third-party-models    第三方中转 8 类问题与规避
│   ├── 04-multi-model           多实例方案：别名、Worktree、多窗口
│   └── 05-env-management        各平台环境变量永久/临时设置
│
├── claude-code-guide/           ← 完整使用指南（全面、实践导向）
│   ├── 01-install-setup         安装、认证、国内代理、VS Code
│   ├── 02-claude-md-mastery     CLAUDE.md 深度指南
│   ├── 03-commands-reference    CLI 命令与斜杠命令速查
│   ├── 04-workflows             工作流：TDD、UI 迭代、并行开发
│   ├── 05-context-management    上下文管理与成本控制
│   ├── 06-hooks-system          Hooks 事件钩子
│   ├── 07-mcp-servers           MCP 服务器推荐与配置
│   ├── 08-skills-slash-commands 自定义斜杠命令
│   ├── 09-advanced-tips         100+ 进阶技巧
│   ├── 10-multi-agent           多 Agent 并行架构
│   ├── 11-security-sandbox      安全与权限管理
│   └── 12-resources             社区资源
│
└── claude-code-config/          ← 配置进阶指南（深度、场景化）
    ├── 01-config-architecture   层级架构与合并规则
    ├── 02-project-isolation     项目配置隔离策略
    ├── 03-custom-models         自定义模型接入
    ├── 04-profile-system        Profile 化管理与切换脚本
    └── 05-model-routing         多模型共存与任务路由
```

---

## 按角色的阅读路径

### 刚安装 Claude Code

```
guide/01-install-setup     → 安装并连上 API
guide/02-claude-md-mastery → 建立项目记忆
guide/03-commands-reference → 掌握常用命令
guide/04-workflows          → 建立标准工作流
```

### 遇到配置问题（中转/代理/VS Code）

```
claude-code/03-third-party-models  → 8 类常见问题速查
claude-code/01-settings-reference  → 查字段含义
claude-code/05-env-management      → 各平台永久设置
config/01-config-architecture      → 理解层级合并规则
```

### 要接入自定义模型（Ollama/Bedrock/私有代理）

```
config/03-custom-models      → 各类模型接入方法
config/04-profile-system     → 管理多个端点配置
config/05-model-routing      → 按任务路由不同模型
```

### 团队共享配置 / 企业部署

```
config/02-project-isolation   → 团队共享 vs 个人私有策略
claude-code/02-recommended-configs → 企业管控配置模板
guide/11-security-sandbox     → 权限与安全
```

### 提升日常开发效率

```
guide/08-skills-slash-commands → 自定义命令加速工作流
guide/09-advanced-tips         → 100+ 实用技巧
guide/06-hooks-system          → 自动化守护
guide/07-mcp-servers           → 扩展工具能力
```

### 遇到性能/成本问题

```
guide/05-context-management    → 控制 Token 消耗
config/05-model-routing        → 按任务选择合适模型
config/03-custom-models        → 接入本地模型零成本
```

---

## 高频问题速查

### Q1：VS Code 插件配置 `environmentVariables` 怎么写？

**格式是数组，每项为 `{name, value}` 对象**（源码验证）：

```json
"claudeCode.environmentVariables": [
  { "name": "ANTHROPIC_AUTH_TOKEN",           "value": "your-token" },
  { "name": "ANTHROPIC_BASE_URL",             "value": "https://your-proxy.com/v1" },
  { "name": "ANTHROPIC_DEFAULT_SONNET_MODEL", "value": "claude-sonnet-4-6" },
  { "name": "DISABLE_PROMPT_CACHING",         "value": "1" }
]
```

> 详见 → [claude-code/01-settings-reference § 3.2](./claude-code/01-settings-reference.md)

---

### Q2：使用第三方中转的最小配置是什么？

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

**必做四件事**：
1. `ANTHROPIC_BASE_URL` 只写到 `/v1`，不加 `/messages`
2. 固定三个模型别名（防漂移 → 404）
3. 禁用 Prompt Cache（大多数中转不支持）
4. VS Code 加 `"claudeCode.disableLoginPrompt": true`

> 详见 → [claude-code/03-third-party-models](./claude-code/03-third-party-models.md)

---

### Q3：配置优先级是什么？

```
会话命令 /model
  > CLI 参数 --model
  > 环境变量 ANTHROPIC_MODEL
  > 项目本地 .claude/settings.local.json    ← 个人私有，不提交
  > 项目共享 .claude/settings.json          ← 团队共享，提交 git
  > 用户全局 ~/.claude/settings.json        ← 个人全局
  > 企业托管 managed-settings.json
  > 默认值
```

**合并规则**：
- `permissions.allow/deny`、`hooks`、`mcpServers` → **累加合并**（所有层都生效）
- `model`、`env`、`effortLevel` → **高优先级覆盖**低优先级

> 详见 → [claude-code-config/01-config-architecture](./claude-code-config/01-config-architecture.md)

---

### Q4：如何隔离不同项目的配置？

```
提交到 git（团队共享，无密钥）：
  .claude/settings.json      → 工具权限、Hooks、模型、行为配置
  .mcp.json                  → MCP 服务器（Token 用环境变量引用）
  CLAUDE.md                  → 项目规范、架构说明、常用命令

不提交（个人私有）：
  .claude/settings.local.json → 个人 Token、代理、权限偏好
  CLAUDE.local.md             → 个人工作笔记、本地环境信息
```

> 详见 → [claude-code-config/02-project-isolation](./claude-code-config/02-project-isolation.md)

---

### Q5：怎么管理多个模型端点（主/备/本地）？

**方案 A：Shell 别名（轻量，推荐个人）**

```bash
# ~/.zshrc
alias cc='claude'
alias cc-fast='claude --model haiku'
alias cc-best='claude --model opus'
alias cc-backup='ANTHROPIC_BASE_URL=https://backup.com/v1 claude'
alias cc-local='ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:4000/v1 claude'
```

**方案 B：Profile 切换脚本（适合多端点管理）**

```bash
claude-profile use opus    # 永久切换
claude-profile use haiku
claude-profile list
```

> 详见 → [claude-code-config/04-profile-system](./claude-code-config/04-profile-system.md)
> 多模型方案对比 → [claude-code/04-multi-model](./claude-code/04-multi-model.md)

---

### Q6：如何接入本地模型（Ollama）？

```bash
# 1. 安装并下载模型
ollama pull qwen2.5-coder:32b

# 2. 启动 Anthropic 格式代理
litellm --model ollama/qwen2.5-coder:32b --port 4000

# 3. 配置 Claude Code
```

```jsonc
{ "env": {
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_BASE_URL": "http://localhost:4000/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "qwen2.5-coder:32b",
    "DISABLE_PROMPT_CACHING": "1"
} }
```

> 详见 → [claude-code-config/03-custom-models § 八](./claude-code-config/03-custom-models.md)

---

### Q7：上下文快满了怎么办？

| 使用率 | 操作 |
|--------|------|
| > 70% | `/compact` 压缩，保留关键信息 |
| 任务切换 | `/clear` 重置 |
| 重要约定 | `# <内容>` 写入 CLAUDE.md（永久记忆） |
| 跨天恢复 | `claude --resume` 选择历史会话 |

> 详见 → [claude-code-guide/05-context-management](./claude-code-guide/05-context-management.md)

---

### Q8：CLAUDE.md 写什么最有效？

```markdown
## 常用命令       ← Claude 每次都能用正确命令
## 架构说明       ← 不需要重复解释代码结构
## 代码规范       ← 用约束语言（"必须"/"禁止"），不用请求语言
## 禁止事项       ← 明确告知不能做什么
## 已知问题       ← 避免 Claude 重复踩坑
```

**铁律**：内容 < 2000 词（超出后末尾内容被忽略），细节用 `@file` 引用。

> 详见 → [claude-code-guide/02-claude-md-mastery](./claude-code-guide/02-claude-md-mastery.md)

---

## 核心概念一张图

```
                    ┌─────────────────────────────────────────┐
                    │           Claude Code 核心概念           │
                    └─────────────────────────────────────────┘

  记忆层                工具层                  执行层
  ─────────            ─────────              ─────────
  CLAUDE.md            MCP 服务器              Sub-agents
  项目规范             扩展外部能力             并行子任务
  永久加载             Context7/Playwright      Worktree 隔离

  配置层                防护层                  工作流层
  ─────────            ─────────              ─────────
  settings.json        Hooks                  探索→计划→编码→验证→提交
  四层优先级            PreToolUse/PostToolUse  TDD / UI迭代 / CI/CD
  Profile 切换          确定性守护              /compact /clear 上下文管理

  模型层
  ─────────
  Haiku → 快速轻量（成本 1x）
  Sonnet → 日常开发（成本 5x）
  Opus   → 复杂推理（成本 25x）
  本地   → Ollama（成本 0）
```

---

## 五条核心原则

1. **上下文是生命线** — 70% 时开始降精度，85% 时幻觉增加，定期 `/compact`
2. **CLAUDE.md 是项目记忆** — 不要依赖 Claude 自己记住规范，写进文件
3. **settings.json 优于环境变量** — 跨平台统一，不污染其他程序，可追溯
4. **密钥永远在本地层** — `settings.local.json` 或 `~/.claude/settings.json`，绝不提交 git
5. **合适的模型做合适的事** — Sub-agent 用 Haiku，主线用 Sonnet，架构决策才用 Opus

---

## 参考来源

- [Anthropic 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [claude-cn.org 中文教程](https://www.claude-cn.org)
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)
- [trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config)
- [feiskyer/claude-code-settings](https://github.com/feiskyer/claude-code-settings)
- VS Code 扩展源码分析（anthropic.claude-code-2.1.76）
