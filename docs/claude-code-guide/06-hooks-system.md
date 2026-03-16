# Hooks 系统：事件钩子与自动化守护

> Hooks 是 Claude Code 的自动化守护机制。在特定事件发生时执行 Shell 命令，实现确定性防护和工作流自动化。

---

## 一、Hooks 概述

### 什么是 Hooks？

Hooks 在以下事件发生时**自动执行 Shell 命令**：
- Claude 准备调用工具之前
- Claude 调用工具之后
- Claude 完成一轮回复时
- 用户提交消息时

### Hooks vs 提示词的区别

| 特性 | 提示词约束 | Hooks |
|------|-----------|-------|
| 可靠性 | 概率性（可能被忽略） | 确定性（每次都执行） |
| 用途 | 引导行为偏好 | 强制执行规则 |
| 典型应用 | "请写测试" | 强制运行 lint |
| 执行者 | Claude 模型 | 操作系统 |

> **重要**：Hooks 是**工作流辅助工具**，不是安全边界。不能依赖 Hooks 防范恶意提示词注入。

---

## 二、配置位置

Hooks 在 `~/.claude/settings.json`（用户级）或 `.claude/settings.json`（项目级）中配置：

```json
{
  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...],
    "Stop": [...],
    "UserPromptSubmit": [...]
  }
}
```

---

## 三、四种事件类型

### 1. PreToolUse — 工具调用前

在 Claude 执行工具（Edit、Bash 等）**之前**触发。可以：
- 阻止危险操作（exit code 2 = 阻止）
- 注入额外上下文
- 记录操作日志

### 2. PostToolUse — 工具调用后

在工具执行**完成后**触发。可以：
- 验证操作结果
- 触发后续自动化（如自动格式化）
- 记录变更

### 3. Stop — Claude 完成响应时

在 Claude **完成一轮回复**后触发。可以：
- 运行测试套件
- 发送通知
- 更新状态

### 4. UserPromptSubmit — 用户提交消息时

在用户**发送消息后**、Claude 处理前触发。可以：
- 注入额外上下文
- 预处理输入

---

## 四、配置格式详解

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '即将执行 Bash 命令'",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

**字段说明**：
- `matcher`：匹配工具名称（`"Bash"`、`"Edit"`、`"Write"`、`"*"` 等）
- `type`：目前只支持 `"command"`
- `command`：要执行的 Shell 命令
- `timeout`：超时秒数（默认 60）

### 退出码语义

| 退出码 | 含义 |
|--------|------|
| 0 | 成功，继续执行 |
| 2 | **阻止工具调用**（PreToolUse 专用） |
| 其他非 0 | 错误，Claude 会看到输出但继续执行 |

---

## 五、实用配置示例

### 5.1 自动代码格式化

每次 Edit 操作后自动格式化：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "cd \"$CLAUDE_PROJECT_DIR\" && prettier --write \"$TOOL_INPUT_path\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### 5.2 阻止修改受保护文件

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$TOOL_INPUT_path\" | grep -qE '(migrations/|.env$|package-lock.json)'; then echo '禁止修改此文件'; exit 2; fi"
          }
        ]
      }
    ]
  }
}
```

### 5.3 自动运行测试

Claude 完成修改后自动运行相关测试：

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "cd \"$CLAUDE_PROJECT_DIR\" && npm test -- --passWithNoTests 2>&1 | tail -5"
          }
        ]
      }
    ]
  }
}
```

### 5.4 记录操作日志

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"[$(date)] Bash: $TOOL_INPUT_command\" >> ~/.claude/audit.log"
          }
        ]
      }
    ]
  }
}
```

### 5.5 阻止危险 Bash 命令

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$TOOL_INPUT_command\" | grep -qE '(rm -rf|git push --force|DROP TABLE|DELETE FROM.*WHERE 1)'; then echo '危险命令被阻止'; exit 2; fi"
          }
        ]
      }
    ]
  }
}
```

### 5.6 注入环境上下文

每次对话开始时注入当前状态：

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"当前分支: $(git branch --show-current 2>/dev/null); Node: $(node -v 2>/dev/null)\""
          }
        ]
      }
    ]
  }
}
```

---

## 六、可用环境变量

Hooks 执行时可使用以下环境变量：

| 变量 | 说明 |
|------|------|
| `$CLAUDE_PROJECT_DIR` | 当前项目根目录 |
| `$TOOL_NAME` | 正在调用的工具名称 |
| `$TOOL_INPUT_command` | Bash 工具的命令内容 |
| `$TOOL_INPUT_path` | Edit/Write 工具的文件路径 |
| `$TOOL_INPUT_content` | Write 工具的写入内容 |
| `$TOOL_RESULT_output` | PostToolUse 中的工具输出 |

---

## 七、Trail of Bits 推荐的 Hooks 配置

来自 Trail of Bits 安全团队的三层防护配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/check-bash.sh",
            "timeout": 5
          }
        ]
      },
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/check-edit.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/post-edit.sh"
          }
        ]
      }
    ]
  }
}
```

```bash
# ~/.claude/hooks/check-bash.sh
#!/bin/bash
DANGEROUS_PATTERNS=(
  "rm -rf /"
  "curl.*| bash"
  "wget.*| sh"
  "> /etc/passwd"
  "chmod 777"
)

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$TOOL_INPUT_command" | grep -qi "$pattern"; then
    echo "危险命令被 Hook 阻止: $pattern"
    exit 2
  fi
done
```

---

## 八、Hooks 安全注意事项

1. **Hooks 不是安全边界**：Claude 可能绕过 Hooks（通过不同的工具调用路径）
2. **命令注入风险**：Hooks 命令中包含用户输入时，注意转义
3. **超时设置**：避免 Hook 运行时间过长阻塞工作流
4. **exit code 2 用法**：只在 PreToolUse 中有阻止效果，其他事件不适用
5. **最小权限原则**：Hook 脚本只授予必要权限

```bash
# ❌ 命令注入风险
command: "echo $TOOL_INPUT_command | process"

# ✅ 安全写法（先写文件再处理）
command: "printf '%s' \"$TOOL_INPUT_command\" > /tmp/cmd.txt && process /tmp/cmd.txt"
```

---

## 九、调试 Hooks

```bash
# 查看 Hooks 执行日志
cat ~/.claude/audit.log

# 临时禁用 Hooks
claude --no-hooks "测试命令"

# 测试 Hook 脚本
TOOL_INPUT_command="rm -rf /" bash ~/.claude/hooks/check-bash.sh
echo "exit code: $?"
```
