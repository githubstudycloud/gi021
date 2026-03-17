# Profile 化配置管理：多端点共存与一键切换

> Claude Code 没有原生 Profile 系统，但可以通过以下方案实现完整的 Profile 管理。

---

## 一、Profile 系统设计思路

```
~/.claude/
├── settings.json          ← 当前激活的配置（由脚本管理）
├── profiles/
│   ├── default.json       ← 日常开发（Sonnet + 主中转）
│   ├── opus.json          ← 复杂任务（Opus + 主中转）
│   ├── haiku.json         ← 快速响应（Haiku + 主中转）
│   ├── backup.json        ← 备用端点（当主中转故障）
│   ├── local.json         ← 本地 Ollama 模型
│   ├── bedrock.json       ← AWS Bedrock（企业合规）
│   └── offline.json       ← 完全离线本地模型
└── scripts/
    └── switch-profile.sh  ← 切换脚本
```

---

## 二、Profile 文件模板

### 2.1 default.json（日常开发）

```jsonc
{
  "_profile": "default",
  "_description": "日常开发，Sonnet 模型，主中转",

  "model": "claude-sonnet-4-6",
  "effortLevel": "medium",

  "env": {
    "ANTHROPIC_AUTH_TOKEN": "YOUR_PRIMARY_TOKEN",
    "ANTHROPIC_BASE_URL": "https://your-primary-proxy.com/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6",
    "DISABLE_PROMPT_CACHING": "1",
    "DISABLE_TELEMETRY": "1",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "32000"
  },

  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}
```

### 2.2 opus.json（复杂任务）

```jsonc
{
  "_profile": "opus",
  "_description": "复杂架构/深度推理，Opus 模型，高成本",

  "model": "claude-opus-4-6",
  "effortLevel": "high",

  "env": {
    "ANTHROPIC_AUTH_TOKEN": "YOUR_PRIMARY_TOKEN",
    "ANTHROPIC_BASE_URL": "https://your-primary-proxy.com/v1",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "DISABLE_PROMPT_CACHING": "1",
    "DISABLE_TELEMETRY": "1",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "128000"
  }
}
```

### 2.3 haiku.json（快速轻量）

```jsonc
{
  "_profile": "haiku",
  "_description": "快速响应，Haiku 模型，低成本",

  "model": "claude-haiku-4-5-20251001",
  "effortLevel": "low",

  "env": {
    "ANTHROPIC_AUTH_TOKEN": "YOUR_PRIMARY_TOKEN",
    "ANTHROPIC_BASE_URL": "https://your-primary-proxy.com/v1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "DISABLE_PROMPT_CACHING": "1",
    "DISABLE_TELEMETRY": "1",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "8000"
  }
}
```

### 2.4 backup.json（备用端点）

```jsonc
{
  "_profile": "backup",
  "_description": "备用中转（主中转故障时使用）",

  "model": "claude-sonnet-4-6",

  "env": {
    "ANTHROPIC_AUTH_TOKEN": "YOUR_BACKUP_TOKEN",
    "ANTHROPIC_BASE_URL": "https://backup-proxy.com/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

### 2.5 local.json（Ollama 本地模型）

```jsonc
{
  "_profile": "local",
  "_description": "本地 Ollama 模型，零成本，完全私有",

  "model": "qwen2.5-coder:32b",

  "env": {
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_BASE_URL": "http://localhost:4000/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "qwen2.5-coder:32b",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "qwen2.5-coder:7b",
    "DISABLE_PROMPT_CACHING": "1",
    "NODE_TLS_REJECT_UNAUTHORIZED": "0"
  }
}
```

---

## 三、Profile 切换脚本

```bash
#!/bin/bash
# ~/.claude/scripts/switch-profile.sh

PROFILES_DIR="$HOME/.claude/profiles"
SETTINGS_FILE="$HOME/.claude/settings.json"

# 颜色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

show_help() {
  echo -e "${BLUE}Claude Code Profile 管理器${NC}"
  echo ""
  echo "用法: claude-profile [命令] [profile名称]"
  echo ""
  echo "命令:"
  echo "  list         列出所有可用 profile"
  echo "  use <name>   切换到指定 profile"
  echo "  show         显示当前激活的 profile"
  echo "  info <name>  显示指定 profile 的详情"
  echo ""
  echo "示例:"
  echo "  claude-profile use opus"
  echo "  claude-profile use haiku"
  echo "  claude-profile list"
}

list_profiles() {
  echo -e "${BLUE}可用的 Profile:${NC}"
  echo ""
  for f in "$PROFILES_DIR"/*.json; do
    name=$(basename "$f" .json)
    desc=$(jq -r '._description // "无描述"' "$f" 2>/dev/null)
    model=$(jq -r '.model // .env.ANTHROPIC_DEFAULT_SONNET_MODEL // "未指定"' "$f" 2>/dev/null)

    # 高亮当前激活的 profile
    current=$(jq -r '._profile // ""' "$SETTINGS_FILE" 2>/dev/null)
    if [ "$name" = "$current" ]; then
      echo -e "  ${GREEN}▶ $name${NC} - $desc"
      echo -e "    模型: $model"
    else
      echo -e "  ${YELLOW}  $name${NC} - $desc"
      echo -e "    模型: $model"
    fi
    echo ""
  done
}

switch_profile() {
  local profile="$1"
  local profile_file="$PROFILES_DIR/$profile.json"

  if [ ! -f "$profile_file" ]; then
    echo -e "${RED}错误: Profile '$profile' 不存在${NC}"
    echo ""
    list_profiles
    exit 1
  fi

  # 备份当前配置
  if [ -f "$SETTINGS_FILE" ]; then
    cp "$SETTINGS_FILE" "$SETTINGS_FILE.bak"
  fi

  # 切换配置
  cp "$profile_file" "$SETTINGS_FILE"
  echo -e "${GREEN}✓ 已切换到 Profile: $profile${NC}"

  # 显示关键配置
  base_url=$(jq -r '.env.ANTHROPIC_BASE_URL // "官方直连"' "$SETTINGS_FILE")
  model=$(jq -r '.model // "默认"' "$SETTINGS_FILE")
  desc=$(jq -r '._description // ""' "$SETTINGS_FILE")
  echo "  端点: $base_url"
  echo "  模型: $model"
  [ -n "$desc" ] && echo "  说明: $desc"
}

show_current() {
  if [ ! -f "$SETTINGS_FILE" ]; then
    echo -e "${YELLOW}未找到配置文件${NC}"
    exit 1
  fi

  profile=$(jq -r '._profile // "未知"' "$SETTINGS_FILE")
  base_url=$(jq -r '.env.ANTHROPIC_BASE_URL // "官方直连"' "$SETTINGS_FILE")
  model=$(jq -r '.model // "默认"' "$SETTINGS_FILE")

  echo -e "${BLUE}当前激活的 Profile:${NC} ${GREEN}$profile${NC}"
  echo "  端点: $base_url"
  echo "  模型: $model"
}

# 主逻辑
case "$1" in
  list|ls)     list_profiles ;;
  use|switch)
    [ -z "$2" ] && { show_help; exit 1; }
    switch_profile "$2"
    ;;
  show|current) show_current ;;
  info)
    [ -z "$2" ] && { show_help; exit 1; }
    jq . "$PROFILES_DIR/$2.json"
    ;;
  *)           show_help ;;
esac
```

### 安装脚本

```bash
# 创建目录
mkdir -p ~/.claude/profiles ~/.claude/scripts

# 保存脚本
cp switch-profile.sh ~/.claude/scripts/
chmod +x ~/.claude/scripts/switch-profile.sh

# 创建全局命令
sudo ln -sf ~/.claude/scripts/switch-profile.sh /usr/local/bin/claude-profile
# 或追加到 PATH
echo 'export PATH="$HOME/.claude/scripts:$PATH"' >> ~/.zshrc
```

### 使用方式

```bash
claude-profile list        # 查看所有 profile
claude-profile use opus    # 切换到 opus
claude-profile use haiku   # 切换到 haiku
claude-profile use local   # 切换到本地 Ollama
claude-profile show        # 查看当前 profile
```

---

## 四、Shell 别名（轻量版）

不需要切换全局 settings，通过别名按需覆盖：

```bash
# ~/.zshrc 或 ~/.bashrc

# 基础别名（使用全局 settings.json）
alias cc='claude'

# 按模型
alias cc-fast='ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-haiku-4-5-20251001 claude --model haiku'
alias cc-smart='ANTHROPIC_DEFAULT_SONNET_MODEL=claude-sonnet-4-6 claude --model sonnet'
alias cc-best='ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-6 claude --model opus'

# 按端点
alias cc-backup='ANTHROPIC_BASE_URL=https://backup-proxy.com/v1 claude'
alias cc-local='ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:4000/v1 claude'

# 组合别名（端点 + 模型）
alias cc-local-big='ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:4000/v1 ANTHROPIC_DEFAULT_SONNET_MODEL=qwen2.5-coder:32b claude --model sonnet'
alias cc-bedrock='CLAUDE_CODE_USE_BEDROCK=1 AWS_REGION=us-east-1 claude'

# 项目相关（指定工作目录）
alias cc-work='cd ~/work/main-project && claude'
alias cc-side='cd ~/work/side-project && claude'
```

---

## 五、VS Code 多工作区 Profile

每个 VS Code 工作区（`.code-workspace` 文件）可以有独立的 Claude 配置：

```json
// myproject.code-workspace
{
  "folders": [{ "path": "." }],
  "settings": {
    "claudeCode.selectedModel": "claude-haiku-4-5-20251001",
    "claudeCode.environmentVariables": [
      { "name": "ANTHROPIC_BASE_URL", "value": "https://fast-proxy.com/v1" },
      { "name": "ANTHROPIC_DEFAULT_HAIKU_MODEL", "value": "claude-haiku-4-5-20251001" }
    ]
  }
}
```

不同工作区文件对应不同配置：

```
workspaces/
├── fast-work.code-workspace   → Haiku + 快速中转
├── deep-work.code-workspace   → Opus + 稳定中转
└── local-work.code-workspace  → Ollama 本地模型
```

---

## 六、自动化 Profile 切换

### 基于目录自动切换（direnv）

```bash
# 安装 direnv
brew install direnv  # macOS
# 或 apt install direnv

# 在项目目录创建 .envrc
cat > /path/to/project/.envrc << 'EOF'
# 进入此目录时自动切换 Claude Profile
claude-profile use project-specific
EOF

direnv allow /path/to/project
```

### 基于时间的自动切换

```bash
# crontab -e
# 工作时间使用主端点（Sonnet）
0 9 * * 1-5 /home/user/.claude/scripts/switch-profile.sh use default

# 下班后切换到本地模型节省成本
0 18 * * 1-5 /home/user/.claude/scripts/switch-profile.sh use local
```

---

## 七、Profile 共享（团队）

```bash
# 将 profiles 目录纳入版本控制（不含密钥的模板）
mkdir -p .claude/profiles-template/

# 模板中用占位符代替真实密钥
cat > .claude/profiles-template/default.json << 'EOF'
{
  "_profile": "default",
  "_description": "日常开发（填入个人 token）",
  "model": "claude-sonnet-4-6",
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "${ANTHROPIC_TOKEN}",
    "ANTHROPIC_BASE_URL": "https://team-proxy.com/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
EOF

# 个人安装脚本（用环境变量替换占位符）
envsubst < .claude/profiles-template/default.json \
  > ~/.claude/profiles/default.json
```
