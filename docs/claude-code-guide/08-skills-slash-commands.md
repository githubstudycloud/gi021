# Skills 与自定义斜杠命令

> 自定义斜杠命令是 Claude Code 的"可编程宏"，将常用工作流封装为一行命令。

---

## 一、什么是自定义斜杠命令

自定义斜杠命令是存储在 `.claude/commands/` 目录下的 Markdown 文件。每个文件：
- 文件名 = 命令名（如 `fix-issue.md` → `/project:fix-issue`）
- 文件内容 = 发送给 Claude 的提示词
- 支持 `$ARGUMENTS` 占位符接收参数

**两种作用域**：
- **项目级**（`.claude/commands/`）：对所有团队成员可用，随 git 提交共享
- **用户级**（`~/.claude/commands/`）：仅个人使用，跨所有项目有效

---

## 二、快速入门

### 创建第一个命令

```bash
mkdir -p .claude/commands

cat > .claude/commands/review.md << 'EOF'
请对当前修改进行代码审查，检查：
- 代码规范和风格
- 潜在的 bug 和边界条件
- 安全漏洞
- 性能问题
- 测试覆盖是否足够
EOF
```

### 使用命令

```
> /project:review
```

---

## 三、带参数的命令

使用 `$ARGUMENTS` 占位符接收任意文本：

```bash
cat > .claude/commands/fix-issue.md << 'EOF'
请分析并修复 GitHub Issue #$ARGUMENTS，步骤：

## 分析
1. 用 `gh issue view $ARGUMENTS` 获取 Issue 详情
2. 搜索相关代码和历史 PR
3. 理解根本原因

## 实现
- 创建新分支：`fix/issue-$ARGUMENTS`
- 小步骤实现修复
- 每步提交

## 测试
- 编写复现 bug 的测试用例
- 确认测试先失败，再通过
- 运行完整测试套件

## 提交
- 推送分支并创建 PR
- PR 描述中关联 Issue：`Fixes #$ARGUMENTS`
EOF
```

使用：
```
> /project:fix-issue 123
```

`$ARGUMENTS` 会被替换为 `123`。

---

## 四、常用命令模板

### 4.1 版本发布命令

```markdown
<!-- .claude/commands/release.md -->
请执行版本发布流程，版本号：$ARGUMENTS

步骤：
1. 检查所有测试通过：`npm test`
2. 更新 package.json 版本号为 $ARGUMENTS
3. 更新 CHANGELOG.md（汇总本次变更）
4. 提交：`chore: release v$ARGUMENTS`
5. 创建 Git tag：`v$ARGUMENTS`
6. 推送到远程：`git push && git push --tags`
7. 创建 GitHub Release（用 gh CLI）
```

### 4.2 安全审查命令

```markdown
<!-- ~/.claude/commands/security-review.md -->
请对以下代码进行深度安全审查：

检查清单：
- [ ] SQL 注入漏洞
- [ ] XSS 跨站脚本
- [ ] CSRF 防护
- [ ] 敏感信息硬编码
- [ ] 不安全的依赖
- [ ] 未授权访问风险
- [ ] 输入验证是否完整
- [ ] 密码/Token 存储方式

对每个发现的问题，提供：
1. 严重程度（高/中/低）
2. 漏洞描述
3. 修复建议
4. 代码示例
```

### 4.3 PR 审查命令

```markdown
<!-- ~/.claude/commands/review-pr.md -->
请审查 PR #$ARGUMENTS

步骤：
1. `gh pr view $ARGUMENTS` 查看 PR 描述
2. `gh pr diff $ARGUMENTS` 查看代码变更
3. 分析：
   - 变更是否符合 PR 描述
   - 代码质量和规范
   - 测试是否充分
   - 是否有潜在问题
4. 用 `gh pr review $ARGUMENTS --comment` 发布评论
```

### 4.4 性能分析命令

```markdown
<!-- .claude/commands/perf-audit.md -->
请对 $ARGUMENTS 进行性能审计

分析维度：
1. 时间复杂度和空间复杂度
2. 数据库查询效率（N+1 问题、缺少索引）
3. 网络请求优化（合并请求、缓存策略）
4. 内存使用和泄漏风险
5. 渲染性能（如果是前端代码）

输出格式：
- 性能评分（1-10）
- Top 3 优化建议（按影响从大到小）
- 每条建议的代码示例
```

### 4.5 文档生成命令

```markdown
<!-- .claude/commands/doc.md -->
请为 $ARGUMENTS 生成完整文档

包含：
- 功能概述
- 参数说明（类型、是否必填、默认值）
- 返回值说明
- 使用示例（至少 3 个）
- 错误处理说明
- 注意事项

格式：JSDoc（如果是 JavaScript/TypeScript）
```

---

## 五、Vibe Coding 工作流命令（Kiro 风格）

参考 [feiskyer/claude-code-settings](https://github.com/feiskyer/claude-code-settings)，实现规范驱动开发：

```markdown
<!-- .claude/commands/kiro/spec.md -->
为功能 "$ARGUMENTS" 创建需求规范文档

输出文件：docs/specs/$ARGUMENTS.md

包含：
## 用户故事
作为 [用户角色]，我希望 [功能]，以便 [价值]

## 验收标准
- [ ] 标准 1
- [ ] 标准 2

## 技术约束
- 性能要求
- 安全要求
- 兼容性要求

## 边界条件
- 正常路径
- 异常路径
- 边界值
```

```markdown
<!-- .claude/commands/kiro/task.md -->
基于 docs/specs/$ARGUMENTS.md 生成任务清单

输出文件：docs/tasks/$ARGUMENTS-tasks.md

格式：
- [ ] 任务 1（预估：Xh）
  - 涉及文件
  - 完成标准
- [ ] 任务 2
```

---

## 六、命令组织结构

命令文件支持子目录，子目录名会成为命令前缀：

```
.claude/commands/
├── review.md              → /project:review
├── fix-issue.md           → /project:fix-issue
├── kiro/
│   ├── spec.md            → /project:kiro:spec
│   ├── design.md          → /project:kiro:design
│   └── task.md            → /project:kiro:task
└── infra/
    ├── deploy.md          → /project:infra:deploy
    └── rollback.md        → /project:infra:rollback
```

---

## 七、用户级命令（跨项目通用）

在 `~/.claude/commands/` 下创建的命令以 `/user:` 前缀调用，在所有项目中都可使用：

```bash
mkdir -p ~/.claude/commands

# 常用个人命令
~/.claude/commands/
├── security-review.md     → /user:security-review
├── explain.md             → /user:explain
├── refactor.md            → /user:refactor
└── commit.md              → /user:commit
```

---

## 八、高级技巧

### 8.1 命令中引用文件

```markdown
<!-- .claude/commands/arch-review.md -->
请审查项目架构，参考：
- @docs/architecture.md 中的设计原则
- @src/types/index.ts 中的核心类型

检查：
1. 新代码是否遵循架构原则
2. 类型定义是否一致
3. 模块边界是否清晰
```

### 8.2 链式工作流

```markdown
<!-- .claude/commands/full-feature.md -->
请完整实现功能：$ARGUMENTS

按以下步骤执行：
1. 先理解现有代码，不要编写任何代码
2. 制定实现计划，等待我确认
3. 确认后再开始编码
4. 每完成一个文件后暂停
5. 所有文件完成后运行测试
6. 测试通过后创建 PR
```

### 8.3 创建命令的命令

```markdown
<!-- ~/.claude/commands/create-command.md -->
请创建一个名为 $ARGUMENTS 的自定义命令

询问我：
1. 命令的用途
2. 是否需要参数
3. 作用域（项目/用户）

然后生成命令文件并保存到正确位置。
```

---

## 九、最佳实践

**命名规范**：
- 动词开头：`fix-`, `review-`, `generate-`, `deploy-`
- 简短明确：`pr` 而非 `create-pull-request`
- 使用连字符：`fix-issue` 而非 `fixIssue`

**内容原则**：
- 明确的步骤顺序
- 包含验证/确认节点
- 指定输出格式
- 使用约束语言（"必须"、"不要"）而非请求语言（"请尽量"）

**维护原则**：
- 命令文件随代码一起提交
- 在 CLAUDE.md 中列出常用命令
- 定期审查和更新过时命令
