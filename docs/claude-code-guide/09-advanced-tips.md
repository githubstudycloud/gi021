# 进阶技巧：100+ 实用技巧精选

> 综合来自 claude-cn.org、ykdojo/claude-code-tips、januff/claude-code-tips 社区精华。

---

## 一、提示词技巧

### 1.1 探索阶段锁定不编码

```
请阅读相关文件了解当前架构，但暂时不要编写任何代码。
```
防止 Claude 在不理解代码库的情况下盲目修改。

### 1.2 扩展思考触发词

```
ultrathink           ← 最大推理预算（复杂架构决策）
think harder         ← 深层思考（疑难 bug）
think more           ← 更多思考（方案评估）
think               ← 基础扩展思考（一般任务）
```

交互模式快捷键：按两次 **Shift+Tab** 进入计划模式（自动开启扩展思考）。

### 1.3 约束语言 vs 请求语言

```markdown
# ❌ 弱约束（容易被忽略）
请尽量写测试

# ✅ 强约束（明确指令）
所有新函数必须有单元测试，无测试不予合并
```

### 1.4 设定明确边界

```
只修改 src/auth/ 目录，不要触碰 src/routes/ 中的文件。
```

### 1.5 控制节奏

```
每完成一个文件后暂停，等待我确认再继续。
每完成一步都要暂停等待进一步指令。
```

### 1.6 让 Claude 先解释再动手

```
请先分析现有实现，解释为什么会有这个 bug，然后再修复。
```

### 1.7 指定参考对象

```
遵循 @src/api/users.ts 中的错误处理模式实现新接口。
```

---

## 二、CLAUDE.md 技巧

### 2.1 全局 CLAUDE.md 推荐内容

```markdown
# ~/.claude/CLAUDE.md

## 沟通风格
- 永远使用中文回复
- 简洁直接，直接给解决方案，不要过多铺垫

## 代码质量铁律
- 不写投机性代码（未被要求的功能）
- 不写过早抽象（3条相似代码才考虑抽象）
- 函数长度 < 50 行，文件长度 < 300 行
- 不跳过测试（每个新函数至少一个测试）

## 工具
- Git 操作使用 GitHub CLI（gh）
- Node 项目使用 pnpm，Python 项目使用 uv
- 提交信息遵循 Conventional Commits 格式

## 安全铁律
- 永远不要将密钥写入代码
- 敏感信息用环境变量，在 .env.example 说明
- SQL 查询必须参数化

## 工作流程
1. 先理解需求和现有代码
2. 提出方案，确认后再动手
3. 小步提交，每个 commit 只做一件事
```

### 2.2 项目 CLAUDE.md 推荐约束

```markdown
## Code Architecture
- Python/JS/TS 文件不超过 300 行
- 每层目录文件不超过 8 个
- 避免循环依赖

## 运行规范
- 所有启停操作使用 scripts/ 下的 .sh 脚本
- 日志统一输出到 logs/ 目录

## 禁止事项
- 不要修改 src/generated/ 目录（自动生成）
- 不要直接使用 npm start，使用 scripts/start.sh
```

### 2.3 用 `#` 实时添加记忆

在会话中输入：
```
# 记住：认证 token 存在 httpOnly cookie 中，不是 localStorage
# 记住：测试环境 DB 在 .env.test 中，不要用生产库
```

Claude 会将这些追加到 CLAUDE.md 中。

---

## 三、上下文管理技巧

### 3.1 精准 @ 引用

```
# ✅ 精准引用必要文件
请参考 @src/auth/login.ts 和 @src/types/user.ts

# ❌ 避免过宽引用
分析整个 src/ 目录（会消耗大量 token）
```

### 3.2 上下文使用率告警

当出现以下信号时立即 `/compact`：
- Claude 开始"忘记"前面讨论的内容
- 回答开始重复之前已经解决的问题
- 响应明显变慢
- `/status` 显示超过 70%

### 3.3 长任务持久化计划

```
请将实现计划写入 docs/impl-plan.md，然后按计划执行。
每完成一步，更新文件中的进度标记。
```

即使清空上下文，计划仍在文件中。

---

## 四、工具调用技巧

### 4.1 管道组合

```bash
# 代码审查
git diff | claude "审查这次变更，重点关注安全问题"

# 日志分析
tail -100 app.log | claude -p "分析这段日志中的异常模式"

# 文档翻译
cat README.md | claude -p "翻译成中文，保持 Markdown 格式"
```

### 4.2 脚本化批处理

```bash
# 批量代码审查
for file in src/**/*.ts; do
  echo "=== $file ===" >> review.md
  cat "$file" | claude -p "简要审查这个文件" >> review.md
done
```

### 4.3 非交互模式 + JSON 输出

```bash
# 结构化输出方便后处理
claude --output-format json -p "分析这段代码的复杂度" < code.py | \
  jq '.result.complexity'
```

---

## 五、图像与视觉技巧

### 5.1 UI 迭代工作流

```bash
# 1. 粘贴设计稿（Ctrl+V）
# 2. 实现
# 3. 截图当前效果
# 4. 对比改进
```

### 5.2 截图方式

- 拖拽图片到 Claude Code 终端
- `Ctrl+V` 粘贴剪贴板图片
- 提供路径：`analyze this image: /path/to/screenshot.png`

### 5.3 图像分析应用

```
这是当前界面截图，与 @designs/homepage.png 的设计稿对比，
列出所有差异并逐一修复。
```

---

## 六、Git 工作流技巧

### 6.1 智能提交信息

```bash
git diff --staged | claude -p "生成 Conventional Commits 格式提交信息，不超过 72 字符"
```

### 6.2 变更解释

```bash
git diff main | claude "解释这次改动的目的和潜在影响"
```

### 6.3 Worktree 并行开发

```bash
# 功能 A 和 bugfix 并行
git worktree add ../feat-a feature/user-auth
git worktree add ../bugfix bugfix/login-error

# 两个终端分别进入各自目录运行 claude
```

---

## 七、会话管理技巧

### 7.1 恢复中断的任务

```bash
claude --continue --print "继续之前的任务，我们完成到了步骤 3，接下来处理步骤 4"
```

### 7.2 会话历史利用

```bash
# 从历史中找到某个解决方案
claude --resume  # 选择包含该方案的历史会话
```

### 7.3 跨天任务管理

在结束工作前：
```
请将今天的工作进度、待解决的问题和明天的计划写入 docs/work-log.md
```

明天启动时：
```
请阅读 @docs/work-log.md，了解当前进度，然后继续未完成的任务。
```

---

## 八、调试技巧

### 8.1 提供完整错误上下文

```
我运行 npm test 时遇到以下错误：
[完整堆栈跟踪]

环境：Node.js 20.11，测试框架 Jest
最近的变更：修改了 src/auth/middleware.ts
```

### 8.2 让 Claude 形成假设

```
请列出可能导致这个错误的 3 种假设，
对每种假设说明如何验证，然后逐一测试。
```

### 8.3 二分法定位

```
从最近一次可用的提交开始，通过二分法找出引入 bug 的提交。
```

---

## 九、代码质量技巧

### 9.1 避免过度工程

```
实现时遵循 YAGNI 原则：
- 不实现未被明确要求的功能
- 不为"可能的未来需求"添加抽象
- 保持代码的直接性
```

### 9.2 强制测试

```
在 CLAUDE.md 中加入：
## 测试铁律
所有新函数必须有至少一个单元测试。
函数不超过 50 行，测试覆盖率不低于 80%。
```

### 9.3 代码坏味道检测

在 CLAUDE.md 中加入：
```
一旦识别出以下代码坏味道，立即询问用户是否需要优化：
- 循环依赖
- 僵化（一处改动引发连锁修改）
- 冗余代码（相似逻辑出现 3 次以上）
- 函数超过 50 行
- 文件超过 300 行
```

---

## 十、效率技巧

### 10.1 Shift+Tab 快速切换模式

在交互模式中按 **Shift+Tab**：
- 第一次：进入自动接受编辑模式（Claude 直接写代码，不需要确认）
- 第二次：进入计划模式（只思考，不执行）
- 第三次：回到普通模式

### 10.2 快速初始化新项目

```bash
cd new-project
claude
> /init                          # 自动生成 CLAUDE.md
> 请根据 CLAUDE.md 检查是否有遗漏的重要信息
```

### 10.3 多窗口并行

- VS Code 多 Panel：不同任务在不同 Claude 面板
- 多终端：Worktree + 各自的 Claude 实例
- tmux：在服务器上保持长会话

### 10.4 自动化重复任务

```bash
# 每天自动运行代码审查
cat > daily-review.sh << 'EOF'
#!/bin/bash
git diff HEAD~1 | \
  claude -p "对比昨天的变更，检查是否有问题" \
  > reviews/$(date +%Y%m%d).md
echo "代码审查完成"
EOF
```

---

## 十一、安全与隐私技巧

### 11.1 防止敏感信息泄露

```
在 .gitignore 中：
.env
*.key
*.pem
secrets/
CLAUDE.local.md
```

### 11.2 审查 Claude 的操作

```
在执行任何文件修改之前，请先展示将要做的全部变更，等待我确认。
```

### 11.3 沙箱测试

在受信任的环境执行前，先在测试分支验证：
```bash
git checkout -b claude/test-$(date +%s)
claude "实现功能 X"
# 检查变更后再合并到主分支
```

---

## 十二、团队协作技巧

### 12.1 统一 CLAUDE.md 规范

将团队约定集中在项目 CLAUDE.md 中：
```markdown
## 团队规范
- PR 必须有至少一个 Reviewer 批准
- 所有 API 变更需要更新 @docs/api.md
- 数据库迁移必须经过 DBA 审查
```

### 12.2 共享自定义命令

将常用命令放在 `.claude/commands/`，随 git 提交：
```
.claude/commands/
├── review.md      # 代码审查
├── release.md     # 发版流程
├── fix-issue.md   # 修复 Issue
└── deploy.md      # 部署流程
```

### 12.3 Prompt 版本控制

```
.claude/prompts/
├── onboarding.md   # 新人引导提示词
├── review.md       # 审查标准
└── commit.md       # 提交规范
```

---

## 十三、提示词模板库

### 快速理解代码库

```
请阅读项目的 README.md、package.json 和主要目录，
帮我了解这个项目的架构和技术栈，但暂时不要编写任何代码。
```

### 新功能开发

```
我需要开发 [功能描述]，请按以下步骤：
1. 先阅读相关代码了解现有架构
2. 制定详细的实现计划
3. 实现核心功能
4. 编写测试
每完成一步都要暂停等待我确认。
```

### 错误诊断

```
我遇到了这个错误：[错误信息]
环境：[Node.js/Python版本等]
重现步骤：[如何复现]
请分析根本原因，提供 2-3 种修复方案。
```

### 代码重构

```
请重构 [文件名] 中的 [函数/类]，目标是：
- 提高代码可读性
- 减少重复代码
- 遵循现有项目模式（参考 @src/existing-example.ts）
- 保持功能不变
请先分析现有代码，然后提供重构计划等待确认。
```
