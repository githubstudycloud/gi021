# 核心工作流

> 从探索到提交的完整工作流程，以及 TDD、UI 迭代、并行开发、CI/CD 集成。

---

## 一、标准工作流：探索 → 计划 → 编码 → 验证 → 提交

这是 Claude Code 最核心的工作循环，适用于绝大多数开发任务。

### 第一步：探索（Explore）

```
请阅读项目的 README.md、package.json 和主要目录，帮我了解这个项目的架构和技术栈，但暂时不要编写任何代码。
```

目标：建立上下文，让 Claude 了解代码库现状。

```
# 理解特定功能
find the files that handle user authentication
how do these authentication files work together?
trace the login process from front-end to database
```

### 第二步：计划（Plan）

在交互模式中按两次 **Shift+Tab** 进入 PLAN 模式（自动开启扩展思考）。

```
请 ultrathink 并制定详细的实现计划，列出：
1. 需要修改/新建的文件
2. 每个文件的变更内容
3. 潜在风险和依赖关系
不要开始编码，等我确认计划。
```

> **原则**：计划越详细，实现越准确。对于复杂任务，花 5 分钟确认计划，节省 50 分钟返工。

### 第三步：编码（Code）

```
计划已确认，请按计划实现功能。每完成一个文件后暂停，等待我确认。
```

### 第四步：验证（Verify）

```
请运行测试套件，确认没有引入回归。
npm test

# 如果有测试失败
请分析失败的测试，修复问题，不要删除测试。
```

### 第五步：提交（Commit）

```
请查看当前修改，编写 Conventional Commits 格式的提交信息并提交。
然后创建 Pull Request，包含：修改摘要、测试说明、相关 Issue 链接。
```

---

## 二、测试驱动开发（TDD）

对于有明确输入/输出的功能，TDD 是最可靠的工作流：

```bash
# 第 1 步：写测试（确保测试失败）
"请基于以下期望行为编写测试，确保测试会失败，不要写实现代码：
- 输入：用户名和密码
- 输出：JWT token
- 边界条件：空密码、用户不存在、密码错误"

# 第 2 步：运行确认失败
"运行测试，确认它们失败，不要修改测试。"

# 第 3 步：提交测试
"请提交测试代码。"

# 第 4 步：实现功能
"请实现代码使测试通过，不要修改测试文件。"

# 第 5 步：提交实现
"测试通过了，请提交实现代码。"
```

**优势**：
- 测试作为活的规范文档
- 防止 Claude 偷换测试来"让测试通过"
- 明确的完成标准（所有测试绿）

---

## 三、UI 开发迭代流程

利用 Claude Code 的图像识别能力进行视觉迭代：

```bash
# 1. 提供设计稿（拖拽图片到 Claude Code 窗口，或 Ctrl+V 粘贴）
"这是设计图，请实现这个界面。"

# 2. 运行并截图
"请截图当前实现的效果。"

# 3. 对比改进
"与设计图对比，列出差异并逐一修复：
- 字体大小
- 间距
- 颜色
- 布局"

# 4. 循环直到满意
"继续优化，直到与设计图一致。"
```

**图像传入方式**：
- 拖拽图片到 Claude Code 终端窗口
- 复制图片后在 CLI 中按 `Ctrl+V`（不是 Cmd+V）
- 提供图片路径：`analyze this image: /path/to/design.png`

---

## 四、代码理解工作流

接手新项目时快速建立上下文：

```
# 广度优先
give me an overview of this codebase
explain the main architecture patterns used here
what are the key data models?
how is authentication handled?

# 深度优先
find the files that handle [具体功能]
trace the [用户操作] flow from [起点] to [终点]
explain how [A模块] and [B模块] interact
```

---

## 五、调试工作流

```bash
# 1. 提供完整错误信息
"我运行 npm test 遇到了这个错误：[粘贴完整堆栈跟踪]"

# 2. 让 Claude 分析根因
"请分析错误的根本原因，不要急于修复，先解释清楚。"

# 3. 获取修复建议
"基于分析，请提供 2-3 种修复方案，说明各方案的优缺点。"

# 4. 应用修复
"请应用方案 1，同时确保不影响其他功能。"

# 5. 验证
"请运行相关测试确认问题已解决。"
```

---

## 六、代码重构工作流

```
# 1. 先分析现状
find deprecated API usage in our codebase

# 2. 制定重构计划
suggest how to refactor utils.js to use modern JavaScript features
—— 分析需要多大改动？有哪些风险？

# 3. 安全重构（小步走）
refactor utils.js to use ES2024 features while maintaining the same behavior

# 4. 验证
run tests for the refactored code
```

**重构原则**：
- 每次只重构一个文件/模块
- 重构前确保有测试覆盖
- 每步完成后运行测试
- 不在重构中同时添加功能

---

## 七、并行开发工作流（Git Worktrees）

需要同时开发多个独立功能时：

```bash
# 创建独立工作树
git worktree add ../project-feature-a -b feature/auth
git worktree add ../project-bugfix -b bugfix/login-error

# 在不同终端启动各自的 Claude Code
cd ../project-feature-a && claude   # 终端 1
cd ../project-bugfix && claude       # 终端 2

# 查看所有工作树
git worktree list

# 完成后清理
git worktree remove ../project-feature-a
```

**优势**：
- 每个 Claude 实例有独立的文件状态，互不干扰
- 共享同一个 Git 历史
- 可以同时运行多个耗时任务

> **注意**：新工作树需要单独初始化依赖（`npm install`、虚拟环境等）。

---

## 八、PR 工作流

```
# 生成 PR
summarize the changes I've made to the authentication module
create a pr

# 增强描述
enhance the PR description with more context about the security improvements
add information about how these changes were tested
```

Claude 会调用 `gh pr create` 自动创建 PR，包含：
- 清晰的标题和描述
- 修改内容摘要
- 测试计划
- 相关 Issue 链接

---

## 九、CI/CD 集成工作流

### 作为 Linter 使用

```json
// package.json
{
  "scripts": {
    "lint:claude": "git diff main | claude -p 'you are a linter. report any typos, bugs, or code style issues. format: filename:line - description'",
    "review": "git diff main | claude -p 'do a thorough code review. focus on bugs, security issues, and performance problems'"
  }
}
```

### GitHub Actions 集成

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on:
  pull_request:
    branches: [main]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Run Review
        env:
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.ANTHROPIC_AUTH_TOKEN }}
        run: |
          git diff origin/main...HEAD | \
          claude -p 'review this diff for bugs and security issues, output JSON' \
          --output-format json > review.json
      - name: Post Review Comment
        uses: actions/github-script@v7
        with:
          script: |
            const review = require('./review.json')
            // 解析并发布 PR 评论
```

---

## 十、Vibe Coding 工作流（规范驱动）

基于 [feiskyer/claude-code-settings](https://github.com/feiskyer/claude-code-settings) 的 Kiro 工作流：

```
# 1. 需求阶段
/kiro:spec 用户认证系统
  → 产出：需求文档和验收标准

# 2. 设计阶段
/kiro:design 用户认证系统
  → 产出：架构设计和组件规划

# 3. 任务分解
/kiro:task 用户认证系统
  → 产出：具体实施任务清单

# 4. 逐步执行
/kiro:execute 用户认证系统 实现用户注册接口
  → 产出：可运行的代码
```

每一步都有明确产出，前一步的结果作为下一步的输入，形成完整的开发闭环。

---

## 十一、提示词最佳实践

### 探索阶段

```
请阅读相关文件了解当前架构，但暂时不要编写代码。
```

### 边界设定

```
请只修改 src/auth/ 目录，不要触碰 src/routes/ 中的文件。
```

### 控制节奏

```
每完成一步都要暂停，等待我确认再继续。
```

### 保持一致性

```
遵循项目中现有的错误处理模式，参考 @src/utils/errors.ts。
```

### 渐进迭代

```
先实现最核心的功能，不要一次性写太多代码。
```
