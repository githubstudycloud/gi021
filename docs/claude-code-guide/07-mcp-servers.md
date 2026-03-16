# MCP 服务器：配置、推荐与最佳实践

> MCP（Model Context Protocol）是 Claude Code 的扩展协议，让 Claude 能访问外部工具、数据库、API。

---

## 一、MCP 基础概念

### 什么是 MCP？

MCP 是 Anthropic 发布的开放协议，允许 Claude 与外部服务进行标准化通信：
- **工具调用**：执行外部动作（截图、查询 DB、搜索网页）
- **资源读取**：访问外部数据源（文件系统、数据库表）
- **提示模板**：注入预设上下文

### 代价：Token 消耗

> MCP 服务器注入的工具定义会消耗上下文窗口。**超过 20k token 的 MCP 配置会严重影响 Claude 的表现。**

**原则**：
- 只安装当前任务需要的 MCP
- 避免同时启用 5+ 个重量级 MCP
- 定期清理不用的 MCP 服务器

---

## 二、配置方式

### 方式 1：命令行添加

```bash
# 添加用户级 MCP（所有项目可用）
claude mcp add <name> -s user -- <command>

# 添加项目级 MCP（当前项目可用，写入 .mcp.json）
claude mcp add <name> -s project -- <command>

# 添加带环境变量的 MCP
claude mcp add <name> -s user -e API_KEY=xxx -- <command>
```

### 方式 2：手动编辑配置文件

**用户级**（`~/.claude/settings.json`）：

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"]
    }
  }
}
```

**项目级**（`.mcp.json`，可提交到 git）：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"]
    },
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

> **安全提示**：Token 等敏感信息使用 `${ENV_VAR}` 引用环境变量，不要硬编码。

---

## 三、推荐 MCP 服务器

### 3.1 Context7 — 最新官方文档

解决 Claude 训练数据过期问题，实时获取第三方库的最新文档。

```bash
claude mcp add context7 -s user -- npx -y @upstash/context7-mcp@latest
```

**使用方法**：

```
# 在提示词中加 "use context7"
请用 Next.js 15 实现服务端渲染，use context7

# 指定特定库
请查阅 React 19 的 Server Components 文档，use context7
```

**适用场景**：处理快速迭代的框架（Next.js、React、Prisma、shadcn/ui 等）

### 3.2 Playwright — 浏览器自动化

让 Claude 直接控制浏览器，截图、点击、填写表单。

```bash
claude mcp add playwright -s user -- npx @playwright/mcp@latest --headless
```

**使用场景**：
- UI 开发对比截图
- E2E 测试自动化
- 网页内容抓取
- 调试浏览器渲染问题

```
请截图当前的登录页面，与设计稿对比，列出差异
```

### 3.3 Exa — 语义网络搜索

AI 原生的搜索引擎，返回结构化结果。

```bash
claude mcp add exa -s user \
  -e EXA_API_KEY=your-key \
  -- npx -y exa-mcp-server
```

**使用场景**：研究最新技术方案、查找代码示例、获取最新信息

### 3.4 PostgreSQL / 数据库

直接查询和分析数据库。

```bash
# PostgreSQL
claude mcp add postgres -s project \
  -- npx -y @modelcontextprotocol/server-postgres \
  postgresql://user:password@localhost:5432/dbname

# SQLite
claude mcp add sqlite -s project \
  -- npx -y @modelcontextprotocol/server-sqlite \
  --db-path ./data.db
```

> **安全提示**：数据库连接字符串包含密码，建议通过环境变量传入，不要写入 `.mcp.json`。

### 3.5 文件系统

扩展文件访问范围（Claude Code 默认只访问项目目录）。

```bash
claude mcp add filesystem -s user \
  -- npx -y @modelcontextprotocol/server-filesystem \
  ~/Documents ~/Desktop ~/Downloads
```

### 3.6 GitHub

直接操作 GitHub 仓库、Issue、PR。

```bash
claude mcp add github -s user \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=your_token \
  -- npx @modelcontextprotocol/server-github
```

**所需权限**：
- `repo`：访问私有仓库
- `issues`：读写 Issue
- `pull_requests`：读写 PR

```
我的 GitHub 仓库有哪些最近的 Issue？
帮我查看 PR #123 的变更内容
```

### 3.7 Sequential Thinking — 结构化思考

强制 Claude 进行多步骤推理，适合复杂问题。

```bash
claude mcp add sequential-thinking -s user \
  -- npx -y @modelcontextprotocol/server-sequential-thinking
```

### 3.8 Puppeteer

轻量级浏览器自动化（相比 Playwright 更简单）。

```bash
claude mcp add puppeteer -s user \
  -- npx -y @modelcontextprotocol/server-puppeteer
```

### 3.9 Firecrawl — 网页爬取

高质量网页内容抓取，转换为 Markdown。

```bash
claude mcp add firecrawl -s user \
  -e FIRECRAWL_API_KEY=fc-YOUR_API_KEY \
  -- npx -y firecrawl-mcp
```

---

## 四、MCP 管理命令

```bash
claude mcp list                      # 查看所有 MCP
claude mcp get playwright            # 查看某个 MCP 详情
claude mcp remove playwright -s user # 删除某个 MCP
```

在交互模式中：
```
> /mcp    # 查看 MCP 状态（连接状态、可用工具数量）
```

---

## 五、项目级 MCP 配置示例

适合团队共享的 `.mcp.json`：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"],
      "description": "浏览器自动化，用于 UI 测试和截图对比"
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"],
      "description": "获取第三方库最新文档"
    },
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      },
      "description": "GitHub 集成，需要设置 GITHUB_TOKEN 环境变量"
    }
  }
}
```

---

## 六、自定义 MCP 服务器

可以用 Node.js 或 Python 编写自定义 MCP 服务器：

```javascript
// my-mcp-server.js
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  { name: "my-tools", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "query_internal_api",
    description: "查询内部 API",
    inputSchema: {
      type: "object",
      properties: {
        endpoint: { type: "string" },
        params: { type: "object" }
      }
    }
  }]
}));

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;
  if (name === "query_internal_api") {
    const result = await callInternalAPI(args.endpoint, args.params);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

```bash
# 注册自定义 MCP
claude mcp add my-tools -s user -- node /path/to/my-mcp-server.js
```

---

## 七、最佳实践

### Token 优化

```
# 任务前：只启用需要的 MCP
claude mcp list  # 确认只有必要的 MCP 在运行

# 任务后：如果不再需要，考虑移除
claude mcp remove heavy-mcp -s user
```

### 安全实践

- 数据库 MCP 优先使用只读账户
- Token 通过环境变量传入，不写入配置文件
- `.mcp.json` 可提交 git，但不包含任何密钥
- 定期审查 MCP 工具权限

### 调试 MCP

```bash
# 查看 MCP 连接状态
> /mcp

# 输出 MCP 服务器日志
claude mcp get context7  # 查看配置是否正确

# 测试 MCP 是否正常工作
请列出所有可用的 MCP 工具
```
