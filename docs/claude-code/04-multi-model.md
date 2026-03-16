# Claude Code 多模型 / 多实例方案

> 结论先行：Claude Code **原生不支持命名 Profile 或同时连接多个端点**。
> 本文介绍所有可行的变通方案。

---

## 一、原生能力

### 1.1 会话内切换模型

```bash
# 启动时指定模型
claude --model claude-sonnet-4-6
claude --model claude-opus-4-6

# 会话内切换（不中断对话）
/model sonnet
/model opus
/model claude-haiku-4-5-20251001
```

### 1.2 VS Code 插件切换

- 插件面板右上角下拉菜单选择模型
- 或修改 `claudeCode.selectedModel` 后重启会话
- 每个 VS Code 窗口可以运行独立的 Claude Code 会话

---

## 二、多实例方案

### 方案 A：多个终端窗口（最简单）

每个终端使用不同的环境变量启动 Claude Code：

```bash
# 终端 1：连接主力模型（Sonnet）
ANTHROPIC_BASE_URL=https://proxy-a.com/v1 \
ANTHROPIC_AUTH_TOKEN=token-a \
ANTHROPIC_DEFAULT_SONNET_MODEL=claude-sonnet-4-6 \
claude

# 终端 2：连接强力模型（Opus）
ANTHROPIC_BASE_URL=https://proxy-b.com/v1 \
ANTHROPIC_AUTH_TOKEN=token-b \
ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-6 \
claude --model opus
```

---

### 方案 B：Shell 别名（推荐）

在 `~/.bashrc` 或 `~/.zshrc` 中定义别名，一键切换：

```bash
# ~/.zshrc

# 默认配置（走全局 settings.json）
alias cc='claude'

# 快速版（Haiku，低延迟）
alias cc-fast='ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-haiku-4-5-20251001 claude --model haiku'

# 强力版（Opus，复杂任务）
alias cc-power='ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-6 claude --model opus'

# 备用中转（当主中转故障时）
alias cc-backup='ANTHROPIC_BASE_URL=https://backup-proxy.com/v1 claude'

# 直连 Anthropic（需代理）
alias cc-direct='ANTHROPIC_BASE_URL=https://api.anthropic.com claude'
```

使用：

```bash
cc          # 默认
cc-fast     # 切换到 Haiku
cc-power    # 切换到 Opus
```

---

### 方案 C：多份 settings.json 手动切换

维护多个配置文件，需要时手动切换：

```bash
# 配置文件目录
~/.claude/profiles/
├── default.json     # 日常使用
├── opus.json        # 复杂任务
├── haiku.json       # 快速响应
└── backup.json      # 备用端点

# 切换脚本
cat > ~/.local/bin/claude-switch << 'EOF'
#!/bin/bash
PROFILE="${1:-default}"
PROFILE_DIR="$HOME/.claude/profiles"

if [ ! -f "$PROFILE_DIR/$PROFILE.json" ]; then
    echo "可用 profiles: $(ls $PROFILE_DIR/*.json | xargs -n1 basename | sed 's/.json//')"
    exit 1
fi

cp "$PROFILE_DIR/$PROFILE.json" "$HOME/.claude/settings.json"
echo "已切换到 profile: $PROFILE"
EOF
chmod +x ~/.local/bin/claude-switch

# 使用
claude-switch opus    # 切换到 Opus 配置
claude-switch haiku   # 切换到 Haiku 配置
claude-switch         # 恢复默认
```

---

### 方案 D：Worktree 并行代理（高级）

使用 Git Worktree 在不同目录运行独立 Claude 实例，
每个 Worktree 有独立的 `.claude/settings.local.json`：

```
project/
├── .claude/settings.json          ← 共享配置（无认证信息）
├── feature-a/                     ← Worktree A
│   └── .claude/settings.local.json  ← 使用 Sonnet
└── feature-b/                     ← Worktree B
    └── .claude/settings.local.json  ← 使用 Opus
```

```jsonc
// feature-a/.claude/settings.local.json
{
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6"
  },
  "model": "claude-sonnet-4-6"
}
```

```jsonc
// feature-b/.claude/settings.local.json
{
  "env": {
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6"
  },
  "model": "claude-opus-4-6"
}
```

在各 Worktree 目录中分别启动 `claude`，互不干扰。

---

### 方案 E：VS Code 多窗口

每个 VS Code 窗口打开不同目录（含不同的 `.claude/settings.local.json`），
插件会读取各自目录的配置，实现"多窗口多模型"。

```
窗口1: 打开 /projects/fast-tasks   → settings.local.json 指向 Haiku
窗口2: 打开 /projects/complex-task → settings.local.json 指向 Opus
```

---

## 三、各方案对比

| 方案 | 难度 | 同时运行 | CLI | VS Code | 推荐场景 |
|------|------|:---:|:---:|:---:|------|
| A：多终端 | 低 | ✅ | ✅ | ✅ | 临时测试不同模型 |
| B：Shell 别名 | 低 | ✅ | ✅ | ❌ | 日常 CLI 快速切换 |
| C：配置文件切换 | 中 | ❌ | ✅ | ✅ | 长期维护多个端点 |
| D：Worktree 并行 | 高 | ✅ | ✅ | ✅ | 并行开发不同功能 |
| E：VS Code 多窗口 | 低 | ✅ | ❌ | ✅ | 多项目并行 |

---

## 四、子 Agent 使用不同模型

在同一个 Claude Code 会话中，可以让子 Agent 使用更轻量的模型：

```json
{
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-haiku-4-5-20251001"
  }
}
```

这样主对话用 Sonnet，后台并行子任务用 Haiku，节省成本。
