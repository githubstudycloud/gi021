# 按任务路由模型：多模型共存与成本优化

> 不同任务对模型能力的需求不同。合理路由可以在保证质量的前提下大幅降低成本。

---

## 一、模型选择矩阵

| 任务类型 | 推荐模型 | 原因 | 相对成本 |
|---------|---------|------|---------|
| 简单问答、代码补全 | Haiku | 速度快，成本低 | 1x |
| 功能开发、代码审查 | Sonnet | 质量/成本最佳平衡 | 5x |
| 复杂架构、深度推理 | Opus | 最高推理能力 | 25x |
| 本地开发测试 | Ollama | 零成本，完全私有 | 0 |
| 企业合规场景 | Bedrock/Vertex | 数据不离开云账户 | 可变 |

---

## 二、会话内模型路由

### 2.1 手动切换（斜杠命令）

在同一会话中无缝切换模型：

```
# 快速问答用 Haiku
/model haiku
有哪些常见的排序算法？

# 切回 Sonnet 做实现
/model sonnet
请实现一个快速排序，要求：时间复杂度 O(nlogn)，原地排序

# 复杂架构决策用 Opus
/model opus
请 ultrathink 分析这个分布式缓存设计的潜在问题
```

### 2.2 启动时指定模型

```bash
# 不同终端，不同模型
claude --model haiku   "快速回答：Python list 和 tuple 的区别"
claude --model sonnet  "帮我实现用户认证模块"
claude --model opus    "分析这个微服务架构设计的扩展性"
```

---

## 三、Sub-agent 模型路由

主线程用高质量模型，子任务用轻量模型：

```jsonc
// ~/.claude/settings.json
{
  "model": "claude-sonnet-4-6",           // 主对话用 Sonnet
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5-20251001",

    // 子 Agent（后台并行任务）用 Haiku
    "CLAUDE_CODE_SUBAGENT_MODEL": "claude-haiku-4-5-20251001"
  }
}
```

**效果**：
- 主对话的代码生成、架构设计 → Sonnet（高质量）
- 后台并行的文件读取、搜索、简单分析 → Haiku（低成本）
- 典型场景节省 60-70% 成本

---

## 四、项目级模型路由

不同项目使用不同模型，通过项目 settings.json 配置：

```
projects/
├── customer-dashboard/       ← 高要求，用 Sonnet
│   └── .claude/settings.json
├── internal-tools/           ← 普通任务，用 Haiku
│   └── .claude/settings.json
├── ml-research/              ← 复杂推理，用 Opus
│   └── .claude/settings.json
└── experiments/              ← 实验性，用本地 Ollama
    └── .claude/settings.json
```

```jsonc
// customer-dashboard/.claude/settings.json
{
  "model": "claude-sonnet-4-6",
  "availableModels": ["claude-sonnet-4-6", "claude-opus-4-6"]  // 不允许用 Haiku
}
```

```jsonc
// internal-tools/.claude/settings.json
{
  "model": "claude-haiku-4-5-20251001",
  "availableModels": ["claude-haiku-4-5-20251001", "claude-sonnet-4-6"]
}
```

```jsonc
// ml-research/.claude/settings.json
{
  "model": "claude-opus-4-6",
  "effortLevel": "high",   // 最大推理预算
  "env": {
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "128000"
  }
}
```

---

## 五、任务类型路由策略

### 5.1 按复杂度路由

```
简单任务（< 5 分钟）:   Haiku  → 代码格式化、简单重命名、文档更新
中等任务（5-30 分钟）:  Sonnet → 功能实现、bug 修复、代码审查
复杂任务（> 30 分钟）:  Opus   → 架构设计、系统重构、复杂算法
```

实践：

```bash
# 简单任务用 Haiku（别名）
cc-fast "把这个函数的变量名从驼峰改为下划线"

# 中等任务用默认 Sonnet
cc "实现用户登录功能，包括 JWT 认证"

# 复杂任务用 Opus
cc-best "ultrathink：分析我们的 API 架构，提出微服务化方案"
```

### 5.2 按成本预算路由

```bash
# 在 .zshrc 中定义带预算提示的别名
cc-budget() {
  echo "当前模型：Haiku（低成本模式）"
  echo "适用：简单查询、格式化、小修改"
  ANTHROPIC_DEFAULT_SONNET_MODEL=claude-haiku-4-5-20251001 \
  claude --model haiku "$@"
}

cc-premium() {
  echo "当前模型：Opus（高质量模式）"
  echo "适用：复杂架构、深度分析"
  read -p "确认使用高成本 Opus 模型？[y/N] " confirm
  [ "$confirm" = "y" ] && claude --model opus "$@"
}
```

### 5.3 按隐私要求路由

```bash
# 敏感代码（不发到外部）→ 本地模型
alias cc-private='ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:4000/v1 claude'

# 普通代码 → 中转代理
alias cc='claude'
```

---

## 六、多模型对比工作流

同一问题同时问多个模型，对比结果：

```bash
#!/bin/bash
# compare-models.sh
QUESTION="$1"
OUTPUT_DIR="./model-compare-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "正在向三个模型提问：$QUESTION"

# 并行提问
claude --model haiku -p "$QUESTION" > "$OUTPUT_DIR/haiku.md" &
claude --model sonnet -p "$QUESTION" > "$OUTPUT_DIR/sonnet.md" &
ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-6 \
  claude --model opus -p "$QUESTION" > "$OUTPUT_DIR/opus.md" &

wait

echo "三个模型的回答已保存到 $OUTPUT_DIR/"
echo "haiku:  $(wc -w < $OUTPUT_DIR/haiku.md) 词"
echo "sonnet: $(wc -w < $OUTPUT_DIR/sonnet.md) 词"
echo "opus:   $(wc -w < $OUTPUT_DIR/opus.md) 词"
```

```bash
./compare-models.sh "如何设计一个高可用的消息队列系统？"
```

---

## 七、成本监控与控制

### 7.1 设置 Token 预算

```jsonc
// 按环境设置不同的 Token 上限
// .claude/settings.dev.json（开发环境节省成本）
{
  "env": {
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "8000"    // 限制输出长度
  }
}

// .claude/settings.prod.json（生产分析不限制）
{
  "env": {
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "128000"
  }
}
```

### 7.2 禁用非必要消耗

```jsonc
{
  "env": {
    "DISABLE_PROMPT_CACHING": "1",              // 中转不支持时必须禁用
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",  // 禁用遥测等流量
    "DISABLE_TELEMETRY": "1",
    "DISABLE_ERROR_REPORTING": "1",
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

### 7.3 使用 Sub-agent 降低主线程成本

```
主线程（Sonnet）：规划、决策、代码生成核心逻辑
Sub-agents（Haiku）：
  - 文件扫描和读取
  - 简单的模式匹配
  - 测试用例生成
  - 文档更新
```

---

## 八、完整多模型共存方案汇总

### 场景 A：个人开发者（最常见）

```
日常任务    → Sonnet（全局默认）
代码审查    → cc-fast（Haiku，便宜）
架构设计    → cc-best（Opus，高质量）
私有项目    → cc-local（Ollama，零成本）
主中转故障  → cc-backup（备用端点）
```

```bash
# ~/.zshrc 全配置
alias cc='claude'
alias cc-fast='ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-haiku-4-5-20251001 claude --model haiku'
alias cc-best='ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-6 claude --model opus'
alias cc-local='ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:4000/v1 ANTHROPIC_DEFAULT_SONNET_MODEL=qwen2.5-coder:32b claude'
alias cc-backup='ANTHROPIC_BASE_URL=https://backup-proxy.com/v1 claude'
alias cc-switch='claude-profile use'    # 永久切换
```

### 场景 B：团队 / 企业

```
前端团队    → Haiku（快速迭代）            .claude/settings.json: model=haiku
后端团队    → Sonnet（复杂业务逻辑）        .claude/settings.json: model=sonnet
架构师      → Opus（系统设计）              手动 claude-profile use opus
CI/CD       → Haiku（批量处理，成本可控）    headless 模式固定 haiku
合规审计    → Bedrock（数据不出企业账户）    专用 settings
```

### 场景 C：成本极度敏感

```
主力模型    → Ollama（qwen2.5-coder:32b）   完全免费
降级方案    → Haiku                         非常便宜
紧急需求    → Sonnet                        按需使用
```

```jsonc
// ~/.claude/settings.json（以本地为主）
{
  "model": "qwen2.5-coder:32b",
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_BASE_URL": "http://localhost:4000/v1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "qwen2.5-coder:32b",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

```bash
# 需要更高质量时临时切换
cc-smart "实现这个复杂的分布式锁算法"
```

---

## 九、多模型共存架构图

```
用户
 │
 ├─ 终端 A ─→ cc（全局 settings = Sonnet + 主中转）
 │
 ├─ 终端 B ─→ cc-fast（临时变量覆盖 = Haiku）
 │
 ├─ 终端 C ─→ cc-local（临时变量覆盖 = Ollama）
 │
 ├─ VS Code 工作区 1 ─→ .claude/settings.local.json = Haiku
 │
 ├─ VS Code 工作区 2 ─→ .claude/settings.local.json = Opus
 │
 └─ CI/CD 环境 ─→ 环境变量注入 = Haiku + 企业中转
```

**关键点**：
- 不同方式互不干扰（各自作用域隔离）
- 项目配置优先于全局配置
- 临时环境变量覆盖所有文件配置
- Profile 切换是修改全局 settings.json 的永久方式
