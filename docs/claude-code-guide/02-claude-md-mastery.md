# CLAUDE.md 深度指南

> CLAUDE.md 是 Claude Code 的"项目记忆"。每次对话开始时自动加载，
> 相当于给 Claude 的永久系统提示。**把它当成产品代码来维护。**

---

## 一、文件加载优先级

```
~/.claude/CLAUDE.md              ← 全局（所有项目通用）
    ↓ 追加
<repo>/CLAUDE.md                 ← 项目（团队共享，提交 git）
    ↓ 追加
<repo>/CLAUDE.local.md           ← 本地（个人私有，gitignore）
    ↓ 追加
<repo>/src/CLAUDE.md             ← 子目录（模块级，按需加载）
```

所有文件内容**叠加**，不互相覆盖。Claude 会读取当前工作目录及所有父目录的 CLAUDE.md。

---

## 二、生成与维护

```bash
# 自动生成（在项目根目录）
> /init

# 会话中动态添加记忆
# <这是需要记住的内容>
# Claude 会将其写入 CLAUDE.md

# 定期审查清理
> 请审查 CLAUDE.md，删除过时内容，精简冗余描述
```

**维护原则**：
- 保持简洁，每条规则一行
- 定期审查，删除不再适用的内容
- 假设未来读者（包括 Claude）没有当前上下文

---

## 三、推荐内容结构

```markdown
# 项目名称

## 项目概述
一句话描述这是什么项目，技术栈，主要功能。

## 常用命令
```bash
npm run dev      # 启动开发服务器
npm test         # 运行测试
npm run build    # 构建生产版本
npm run lint     # 代码检查
```

## 架构说明
- 前端：React + TypeScript，状态管理用 Zustand（见 src/stores/）
- 后端：Node.js + Express，路由在 src/routes/
- 数据库：PostgreSQL，ORM 用 Prisma

## 代码规范
- 使用 ES Modules，不用 CommonJS
- 函数命名：camelCase；组件：PascalCase
- 每个函数不超过 50 行
- 必须写单元测试，覆盖率 > 80%

## 测试指南
- 单元测试：Jest + Testing Library
- E2E 测试：Playwright（见 tests/e2e/）
- 运行单个测试：`npm test -- --testNamePattern="测试名"`

## 重要约定
- 不要修改 src/generated/ 目录（自动生成）
- API 响应统一用 src/utils/response.ts 的 helper
- 环境变量统一在 .env.example 中声明

## 已知问题
- 登录页面在 Safari 上有样式问题（待修复）
- 数据库连接池在高并发下偶发超时

## 项目文件结构
- src/components/ — React 组件
- src/api/        — API 路由处理
- src/stores/     — Zustand 状态
- tests/          — 测试文件
```

---

## 四、全局 CLAUDE.md 模板

`~/.claude/CLAUDE.md` 适用于所有项目的通用偏好：

```markdown
# 全局开发规范

## 我的偏好
- 代码语言：中文注释，英文变量名
- 提交信息：Conventional Commits 格式（feat/fix/docs/chore）
- 回复风格：简洁，直接给解决方案，不要过多解释

## 代码质量铁律
- 不写投机性代码（未被要求的功能）
- 不写过早抽象（3条相似代码才考虑抽象）
- 不跳过测试（每个新函数至少一个测试）
- 函数长度 < 50 行，文件长度 < 300 行

## 工具使用
- Git：使用 GitHub CLI（gh）处理所有 GitHub 操作
- 包管理：Node 项目优先 pnpm，Python 项目用 uv
- 测试：先写测试，再写实现

## 安全规范
- 永远不要将密钥、Token 写入代码
- 敏感信息用环境变量，在 .env.example 中说明
- SQL 查询必须参数化，防注入

## 工作流程
1. 先理解需求和现有代码
2. 提出方案，确认后再动手
3. 小步提交，每个 commit 只做一件事
4. 完成后运行测试确认无回归
```

---

## 五、子目录 CLAUDE.md 示例

针对特定模块添加模块级说明：

```markdown
# src/api/ 模块规范

## 路由约定
- 所有路由文件以 .router.ts 结尾
- 使用 express-validator 做参数验证
- 统一错误处理在 middleware/errorHandler.ts

## 安全要求
- 所有写操作路由必须有 authMiddleware
- 用户输入必须经过 sanitize 处理
- 禁止在日志中打印用户密码或 Token
```

---

## 六、CLAUDE.md 提示词优化技巧

### 用约束语言，不用请求语言

```markdown
# ❌ 弱约束（容易被忽略）
请尽量写测试

# ✅ 强约束（明确指令）
所有新函数必须有单元测试，无测试的 PR 不予合并
```

### 引用项目文件

```markdown
## 架构参考
参见 @docs/architecture.md 获取系统架构详情
参见 @src/types/index.ts 了解核心类型定义
```

### 用 `#` 在会话中实时更新

```
# 记住：登录功能使用 JWT，token 存在 httpOnly cookie 中，不是 localStorage
```

Claude 会将这条信息追加到当前项目的 CLAUDE.md 中。

---

## 七、常见错误

| 错误 | 问题 | 修正 |
|------|------|------|
| 内容太多 | 超过 2000 词 Claude 开始忽略末尾 | 精简到核心规则，细节用 `@file` 引用 |
| 太笼统 | "写好代码" | 具体："函数 < 50 行，圈复杂度 < 10" |
| 过期内容 | 删掉的功能还在 CLAUDE.md | 定期清理，`/init` 可重新生成 |
| 重复啰嗦 | 同一规则说三遍 | 每条规则只说一次，简洁即力量 |
