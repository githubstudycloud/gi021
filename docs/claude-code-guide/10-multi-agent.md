# 多 Agent 与 Sub-agents：并行架构

> Claude Code 支持在单次会话中启动多个 Sub-agent 并行工作，或通过多 Worktree 实现真正的并行开发。

---

## 一、Sub-agents 概述

### 什么是 Sub-agents？

Sub-agents 是 Claude Code 的内置功能，允许主 Claude 实例**委派子任务**给独立的 Agent 实例：
- 每个 Sub-agent 有独立的上下文
- 可以并行执行多个任务
- 子任务完成后结果返回主线程

### 触发方式

```
# 自然语言触发
请指示 sub-agents 并行验证关键细节

# 明确委派
请并行完成以下三个任务：
1. 分析认证模块的安全漏洞
2. 检查 API 接口的参数验证
3. 运行测试套件并报告失败用例
```

---

## 二、Sub-agent 文件（.claude/agents/）

可以在 `.claude/agents/` 目录下定义专门化的 Sub-agent：

```markdown
---
name: code-reviewer
description: 专门进行代码质量审查，检查代码规范、潜在bug和性能问题。使用时请主动调用。
tools: Read, Grep, Glob
---

# 代码审查专家

你是一个经验丰富的代码审查专家，专注于：

1. **代码质量检查**
   - 遵循项目的编码规范
   - 识别潜在的 bug 和安全问题
   - 提出性能优化建议

2. **最佳实践**
   - 确保代码可读性和可维护性
   - 检查错误处理是否完善
   - 验证测试覆盖率

3. **审查流程**
   - 首先理解代码的整体结构
   - 逐个检查每个函数和模块
   - 提供具体的改进建议
```

---

## 三、实用 Sub-agent 示例

### 3.1 调试专家

```markdown
<!-- .claude/agents/debugger.md -->
---
name: debugger
description: 专门处理错误、测试失败和异常行为的调试专家。遇到任何问题时主动使用。
tools: Read, Edit, Bash, Grep, Glob
---

# 调试专家

你是一个专精于根因分析的调试专家。被调用时的工作流程：

1. 捕获错误信息和堆栈跟踪
2. 识别复现步骤
3. 定位故障位置（具体到文件:行号）
4. 实施最小化修复
5. 验证解决方案有效性

调试原则：
- 分析错误信息和日志
- 检查最近的代码变更
- 形成并测试假设
- 添加策略性日志排查
- 专注于根本问题，而非表面症状

对每个问题提供：
- 根本原因解释
- 支持诊断的证据
- 具体的代码修复方案
- 测试验证方法
- 预防建议
```

### 3.2 测试生成专家

```markdown
<!-- .claude/agents/test-writer.md -->
---
name: test-writer
description: 专门为代码编写高质量测试用例。在需要增加测试覆盖率时主动调用。
tools: Read, Write, Bash
---

# 测试生成专家

你是一个专精于测试驱动开发的专家。

工作流程：
1. 分析被测代码的行为
2. 识别所有边界条件
3. 编写覆盖正常路径的测试
4. 编写覆盖异常路径的测试
5. 运行测试确认通过

测试质量标准：
- 每个函数至少 3 个测试用例
- 必须覆盖：正常输入、边界值、异常输入
- 测试名称清晰描述预期行为
- 不依赖外部状态（可独立运行）
```

### 3.3 数据分析专家

```markdown
<!-- .claude/agents/data-analyst.md -->
---
name: data-analyst
description: 数据分析专家，专精于 SQL 查询和数据洞察。处理数据分析任务时主动使用。
tools: Bash, Read, Write
---

# 数据分析专家

工作流程：
1. 理解数据分析需求
2. 编写高效的 SQL 查询
3. 分析并总结结果
4. 清晰呈现发现

关键实践：
- 编写带有适当过滤的优化查询
- 为复杂逻辑添加注释
- 格式化结果提高可读性
- 提供数据驱动的建议
- 确保查询高效且成本可控
```

---

## 四、多 Worktree 并行架构

当需要真正的**文件隔离并行**时，使用 Git Worktree：

### 架构图

```
主仓库 (main branch)
    ├── worktree-A (feature/auth)     ← Claude 实例 A
    ├── worktree-B (feature/payment)  ← Claude 实例 B
    └── worktree-C (bugfix/login)     ← Claude 实例 C
```

### 操作流程

```bash
# 创建并行工作树
git worktree add ../project-auth feature/auth
git worktree add ../project-payment feature/payment
git worktree add ../project-bugfix bugfix/login

# 在各个工作树中分别启动 Claude
# 终端 1
cd ../project-auth && npm install && claude

# 终端 2
cd ../project-payment && npm install && claude

# 终端 3
cd ../project-bugfix && npm install && claude

# 查看所有工作树
git worktree list

# 完成后清理
git worktree remove ../project-auth
```

### 优势

- 每个 Claude 实例有独立文件状态，互不干扰
- 可以让 Claude 并行处理耗时任务（测试、构建）
- 所有工作树共享同一个 Git 历史
- 随时可以 `git merge` 合并成果

---

## 五、后台任务模式

对于耗时的任务，可以让 Claude 在后台运行：

```bash
# 启动后台任务
tmux new-session -d -s claude-task-1 'cd ../project-auth && claude "实现用户认证模块，完成后运行测试"'

# 另一个后台任务
tmux new-session -d -s claude-task-2 'cd ../project-payment && claude "实现支付模块，完成后运行测试"'

# 查看进度
tmux attach -t claude-task-1

# 让所有任务使用 --print 模式输出到文件
claude -p "实现功能 X" > output.txt 2>&1 &
```

---

## 六、Headless 模式（自动化）

Claude Code 可以在完全非交互的情况下运行，适合 CI/CD：

```bash
# 非交互模式执行
claude -p "分析代码并生成报告" --output-format json > report.json

# 带权限的非交互执行
claude -p "修复所有 lint 错误" \
  --allowedTools "Edit,Bash(npm run lint:*)" \
  --dangerously-skip-permissions

# 脚本中使用
#!/bin/bash
RESULT=$(claude -p "检查 $1 文件的安全问题" --output-format text)
if echo "$RESULT" | grep -q "HIGH SEVERITY"; then
  echo "发现高危安全问题，中止部署"
  exit 1
fi
```

---

## 七、Sub-agent 链式调用

```
# 让主 Claude 协调多个专家 Agent
请按以下流程处理这个 PR：
1. 调用 code-reviewer 检查代码质量
2. 根据审查结果，调用 test-writer 补充测试
3. 调用 debugger 修复发现的 bug
4. 最后创建包含所有变更的 PR
```

---

## 八、并行验证模式

适合在计划阶段验证多个方案：

```
请 ultrathink 并制定实现计划，同时指示 sub-agents 并行验证：
- 方案 A：使用 Redis 缓存
- 方案 B：使用本地内存缓存
- 方案 C：使用 CDN 缓存

每个 sub-agent 验证：
1. 技术可行性
2. 性能预期
3. 维护成本

汇总结果后给出推荐方案。
```

---

## 九、最佳实践

### 何时使用 Sub-agents
- 任务可以明确拆分为独立子任务
- 需要多角度分析同一问题
- 子任务之间没有强依赖关系

### 何时使用多 Worktree
- 需要修改不同的文件集
- 需要在不同分支同时工作
- 需要隔离实验性变更

### 避免的陷阱
- Sub-agents 之间共享资源可能冲突（同时修改同一文件）
- 多 Worktree 需要手动管理环境初始化
- 后台任务难以监控状态，建议输出到日志文件

### 成本控制
- 每个 Sub-agent 都会消耗独立的 token
- 评估并行带来的效率提升是否值得额外成本
- 简单任务不需要 Sub-agents，直接顺序执行更经济
