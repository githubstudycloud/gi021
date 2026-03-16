# CLI 命令与斜杠命令完整速查

> 版本：Claude Code v2.1.76 | 更新：2026-03-16

---

## 一、CLI 启动命令

### 基础用法

```bash
claude                          # 进入交互模式
claude "描述任务"               # 一次性执行后退出
claude -p "生成SQL" > query.sql # 纯文本输出到文件（-p = --print）
claude --help                   # 查看帮助信息
claude --version                # 查看版本
```

### 会话管理

```bash
claude --continue               # 自动继续最近一次会话
claude --resume                 # 弹出历史会话选择器
claude --continue --print "继续" # 非交互模式下继续（适合脚本）
```

### 权限控制

```bash
# 启动时指定允许的工具
claude --allowedTools Edit,Bash,WebFetch

# 允许特定 Bash 子命令
claude --allowedTools "Bash(git commit:*),Edit"

# 跳过权限确认（危险，仅受信任环境使用）
claude --dangerously-skip-permissions
```

### 输出格式

```bash
claude --output-format text       # 纯文本（默认）
claude --output-format json       # JSON（含元数据）
claude --output-format stream-json # 流式 JSON（实时输出）
```

### 模型选择

```bash
claude --model claude-opus-4-6      # 指定模型
claude --model claude-sonnet-4-6    # Sonnet（默认）
claude --model claude-haiku-4-5-20251001 # Haiku（最快）
```

### 调试与诊断

```bash
claude --verbose "问题"      # 详细日志模式
claude --debug "测试命令"    # 调试模式
```

---

## 二、管道与 Unix 集成

### 文件处理

```bash
# 分析文件内容
cat main.py | claude "检查这段代码的安全问题"

# 翻译文档
cat README.md | claude "翻译成中文"

# 分析错误日志
cat build-error.txt | claude -p "简要解释根本原因" > output.txt
```

### Git 集成

```bash
# 解释变更
git diff | claude "解释这次改动"

# 生成提交信息
git diff --staged | claude "生成 Conventional Commits 格式的提交信息"

# 分析日志
git log --oneline -20 | claude "总结最近的开发动态"
```

### 批处理

```bash
# 批量审查 Python 文件
for file in *.py; do
  cat "$file" | claude "检查代码质量" > "${file%.py}_review.txt"
done

# 链式管道
git diff | claude "总结改动" | claude "用一句话概括"
```

### CI/CD 集成

```json
// package.json
{
  "scripts": {
    "lint:claude": "claude -p 'you are a linter. check the diff vs main for typos. report filename:line and description only.'",
    "review": "git diff main | claude -p 'code review this diff, focus on bugs and security'"
  }
}
```

---

## 三、配置管理命令

```bash
claude config list              # 查看所有配置
claude config list --project    # 查看项目配置
claude config get api-key       # 查看某项配置
claude config set model claude-sonnet-4-6   # 设置全局配置
claude config set --project model claude-opus-4-6  # 设置项目配置
claude config reset             # 重置配置
```

---

## 四、MCP 管理命令

```bash
claude mcp list                 # 列出所有 MCP 服务器
claude mcp get playwright       # 查看某个 MCP 的详细信息
claude mcp remove playwright -s user  # 删除某个 MCP

# 添加 MCP（-s user = 用户级；-s project = 项目级）
claude mcp add playwright -s user -- npx @playwright/mcp@latest
claude mcp add context7 -s user -- npx -y @upstash/context7-mcp@latest
claude mcp add filesystem -s user -- npx -y @modelcontextprotocol/server-filesystem ~/Documents

# 添加带环境变量的 MCP
claude mcp add github-server \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=YOUR_TOKEN \
  -- npx @modelcontextprotocol/server-github
```

---

## 五、交互模式斜杠命令

在 `claude` 交互会话中输入：

### 基础命令

| 命令 | 说明 |
|------|------|
| `/help` | 显示所有可用命令 |
| `/exit` 或 `/quit` | 退出交互模式 |
| `/clear` | 清空当前对话历史（保留 CLAUDE.md） |
| `/status` | 查看当前配置和连接状态 |

### 项目与初始化

| 命令 | 说明 |
|------|------|
| `/init` | 自动分析项目，生成 CLAUDE.md 草稿 |
| `/permissions` | 管理工具权限 |

### 上下文管理

| 命令 | 说明 |
|------|------|
| `/compact` | 压缩对话历史（摘要化旧内容） |
| `/compact <focus>` | 带焦点的压缩（如 `/compact 保留认证相关内容`） |
| `/resume` | 从历史会话中选择并恢复 |

### MCP

| 命令 | 说明 |
|------|------|
| `/mcp` | 查看 MCP 服务器状态 |

### 思考模式（扩展推理）

在提示中使用关键词触发：

| 关键词 | 思考深度 |
|--------|----------|
| `think` | 基础扩展思考 |
| `think harder` / `think more` | 更深层思考 |
| `think longer` / `think a lot` | 最深层思考 |
| `ultrathink` | 最大推理预算 |

```
请 ultrathink 并制定详细的实现计划
think harder about the security implications
```

---

## 六、自定义斜杠命令

### 创建项目命令

```bash
mkdir -p .claude/commands

# 创建命令文件（文件名即命令名）
cat > .claude/commands/optimize.md << 'EOF'
分析以下代码的性能并给出三个具体的优化建议：
EOF

# 使用
> /project:optimize
```

### 带参数的命令（`$ARGUMENTS`）

```bash
cat > .claude/commands/fix-issue.md << 'EOF'
请分析并修复 GitHub Issue #$ARGUMENTS，步骤：
1. 用 gh issue view $ARGUMENTS 获取详情
2. 理解问题根因
3. 在新分支实现修复
4. 编写相关测试
5. 提交并创建 PR
EOF

# 使用
> /project:fix-issue 123
```

### 创建个人命令（所有项目通用）

```bash
mkdir -p ~/.claude/commands

cat > ~/.claude/commands/security-review.md << 'EOF'
对以下代码进行安全审查，重点检查：
- SQL 注入
- XSS 漏洞
- 不安全的反序列化
- 敏感信息泄露
EOF

# 使用（前缀为 /user:）
> /user:security-review
```

### 命令组织（子目录）

```
.claude/commands/
├── optimize.md              → /project:optimize
├── fix-issue.md             → /project:fix-issue
└── frontend/
    └── component.md         → /project:frontend:component
```

---

## 七、上下文引用语法

在提示中引用文件：

```
请参考 @src/components/UserProfile.tsx 的结构实现新组件
对比 @docs/api.md 检查实现是否符合规范
```

引用目录：

```
分析 @src/api/ 目录的整体架构
```

添加记忆（写入 CLAUDE.md）：

```
# 记住：登录功能使用 JWT，token 存在 httpOnly cookie 中，不是 localStorage
```

---

## 八、完整命令速查表

| 功能 | 命令 |
|------|------|
| 启动交互 | `claude` |
| 一次性询问 | `claude "问题"` |
| 纯文本输出 | `claude -p "问题"` |
| 继续上次会话 | `claude --continue` |
| 选择历史会话 | `claude --resume` |
| 查看帮助 | `claude --help` |
| 查看版本 | `claude --version` |
| 配置列表 | `claude config list` |
| MCP 列表 | `claude mcp list` |
| 添加 MCP | `claude mcp add <name> -s user -- <cmd>` |
| 初始化项目 | `/init`（交互模式） |
| 清空上下文 | `/clear`（交互模式） |
| 压缩上下文 | `/compact`（交互模式） |
| 查看权限 | `/permissions`（交互模式） |
| MCP 状态 | `/mcp`（交互模式） |
| 查看状态 | `/status`（交互模式） |
| 项目命令 | `/project:<name>`（交互模式） |
| 个人命令 | `/user:<name>`（交互模式） |
