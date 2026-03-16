# 环境变量管理策略

> **核心原则**：环境变量是**最后手段或临时手段**，优先用 settings.json。
> 本文梳理各平台设置方法、生命周期和最佳实践。

---

## 一、优先级建议

```
❌ 避免         Shell export（临时、易丢失、不跨进程）
⚠️  次选         VS Code claudeCode.environmentVariables（仅插件）
✅  推荐         ~/.claude/settings.json → env（CLI + 插件通用）
✅✅ 最推荐       ~/.claude/settings.json → apiKeyHelper（安全动态获取）
```

**为什么 env 是"最后手段"**：

| 问题 | 说明 |
|------|------|
| VS Code 不继承 Shell 变量 | 图形进程启动早于 Shell 初始化 |
| 临时 export 重启即失效 | 忘记写入 profile 会莫名断连 |
| 明文存储安全风险 | `.bashrc` 中的 Key 被其他程序读取 |
| 多项目冲突 | 全局变量影响所有项目，难以隔离 |
| 版本控制困难 | 无法 `git diff` 环境变量变更历史 |

---

## 二、临时设置（当次会话有效）

### 2.1 单次命令

```bash
# 临时覆盖，不修改任何文件
ANTHROPIC_BASE_URL=https://backup.com/v1 claude

# 多个变量
ANTHROPIC_BASE_URL=https://... ANTHROPIC_AUTH_TOKEN=sk-... claude --model sonnet
```

### 2.2 当前 Shell 会话

```bash
export ANTHROPIC_BASE_URL="https://test-proxy.com/v1"
export ANTHROPIC_AUTH_TOKEN="sk-test-xxx"

claude        # 使用测试配置
# 关闭终端后失效
```

---

## 三、永久设置（各平台）

### 3.1 推荐方式：settings.json

**跨平台，CLI + 插件均生效，推荐优先使用**：

```jsonc
// ~/.claude/settings.json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "your-token",
    "ANTHROPIC_BASE_URL":   "https://your-proxy.com/v1"
  }
}
```

或用 CLI 命令写入：

```bash
claude config set env.ANTHROPIC_AUTH_TOKEN "your-token"
claude config set env.ANTHROPIC_BASE_URL   "https://your-proxy.com/v1"
claude config list   # 查看所有配置
```

---

### 3.2 Windows 永久环境变量

**方式A：系统 GUI（推荐给 Windows 用户）**

```
Win+S → 搜索"环境变量" → 编辑系统环境变量
→ 用户变量 → 新建
  ANTHROPIC_BASE_URL = https://your-proxy.com/v1
```

**方式B：PowerShell（永久写入注册表）**

```powershell
# 用户级别（仅当前用户）
[Environment]::SetEnvironmentVariable(
    "ANTHROPIC_BASE_URL",
    "https://your-proxy.com/v1",
    "User"
)

# 系统级别（需要管理员权限）
[Environment]::SetEnvironmentVariable(
    "ANTHROPIC_BASE_URL",
    "https://your-proxy.com/v1",
    "Machine"
)

# 查看
[Environment]::GetEnvironmentVariable("ANTHROPIC_BASE_URL", "User")
```

**方式C：Git Bash / MSYS2**

```bash
# 写入 ~/.bashrc（Git Bash）
echo 'export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"' >> ~/.bashrc
source ~/.bashrc
```

> ⚠️ Windows Git Bash 的 `~/.bashrc` 中的 export **不会影响 VS Code 图形进程**

---

### 3.3 macOS 永久环境变量

**方式A：Shell Profile（最常用）**

```bash
# zsh（macOS 默认）
echo 'export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"' >> ~/.zshrc
source ~/.zshrc

# bash（旧版）
echo 'export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"' >> ~/.bash_profile
source ~/.bash_profile
```

**方式B：launchctl（影响图形应用，包括 VS Code）**

```bash
# 对当前用户所有应用生效（包括 VS Code 图形进程）
launchctl setenv ANTHROPIC_BASE_URL "https://your-proxy.com/v1"
launchctl setenv ANTHROPIC_AUTH_TOKEN "your-token"

# 永久保存（重启后仍生效，写入 plist）
cat > ~/Library/LaunchAgents/claude-env.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.user.claude-env</string>
  <key>EnvironmentVariables</key>
  <dict>
    <key>ANTHROPIC_BASE_URL</key>
    <string>https://your-proxy.com/v1</string>
  </dict>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
EOF
launchctl load ~/Library/LaunchAgents/claude-env.plist
```

---

### 3.4 Linux 永久环境变量

**用户级别**：

```bash
# ~/.bashrc（交互式 bash）
echo 'export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"' >> ~/.bashrc

# ~/.profile（登录 shell，影响范围更广）
echo 'export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"' >> ~/.profile

# ~/.config/environment.d/（systemd 用户环境，影响 GUI 应用）
mkdir -p ~/.config/environment.d/
cat > ~/.config/environment.d/claude.conf << 'EOF'
ANTHROPIC_BASE_URL=https://your-proxy.com/v1
EOF
```

**系统级别（需 root）**：

```bash
# /etc/environment（所有用户）
echo 'ANTHROPIC_BASE_URL=https://your-proxy.com/v1' >> /etc/environment

# /etc/profile.d/（登录时加载）
cat > /etc/profile.d/claude-code.sh << 'EOF'
export ANTHROPIC_BASE_URL="https://your-proxy.com/v1"
EOF
```

---

## 四、各方式生效范围对比

| 方式 | CLI | VS Code 终端 | VS Code 图形插件 | 重启后 | 其他程序 |
|------|:---:|:---:|:---:|:---:|:---:|
| 临时 `export` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `~/.bashrc` / `.zshrc` | ✅ | ✅ | ❌ | ✅ | ❌ |
| `~/.profile` | ✅ | ✅ | ⚠️ | ✅ | ⚠️ |
| Win 用户环境变量 | ✅ | ✅ | ✅ | ✅ | ✅ |
| macOS `launchctl` | ✅ | ✅ | ✅ | ✅ | ✅ |
| Linux `environment.d` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `~/.claude/settings.json` env | ✅ | ✅ | ✅ | ✅ | ❌ |
| VS Code `environmentVariables` | ❌ | ❌ | ✅ | ✅ | ❌ |

> **结论**：`~/.claude/settings.json` 的 `env` 字段是 **最平衡的方案**，
> 只影响 Claude Code，不污染其他程序环境。

---

## 五、安全建议

### 5.1 永远不要做

```bash
# ❌ 不要提交含密钥的文件
git add ~/.claude/settings.json

# ❌ 不要在 .env 提交密钥
echo "ANTHROPIC_AUTH_TOKEN=sk-real-key" >> .env
git add .env

# ❌ 不要在脚本里硬编码
echo "curl -H 'Authorization: Bearer sk-hardcoded'"
```

### 5.2 应该做

```bash
# ✅ .gitignore 中排除敏感文件
echo "~/.claude/settings.json" >> .gitignore
echo ".claude/settings.local.json" >> .gitignore
echo ".env.local" >> .gitignore

# ✅ 使用 apiKeyHelper 动态获取
# ~/.claude/settings.json
# { "apiKeyHelper": "~/.claude/scripts/get-token.sh" }

# ✅ 敏感 Key 存入系统密钥管理
# macOS Keychain:
security add-generic-password -a claude -s anthropic-token -w "your-token"
# 读取脚本:
# security find-generic-password -a claude -s anthropic-token -w

# ✅ 使用 git-secret 或 SOPS 加密配置
```

### 5.3 项目级别的 .gitignore 模板

```gitignore
# Claude Code 私有配置
.claude/settings.local.json
.claude/*.local.json

# 密钥文件
.env
.env.local
.env.*.local
*.token
*.key
```
