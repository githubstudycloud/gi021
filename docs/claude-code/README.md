# Claude Code 配置文档

> 基于官方文档 + 扩展源码（v2.1.76）分析
> 更新：2026-03-16

---

## 文档目录

| 文件 | 内容 |
|------|------|
| [01-settings-reference.md](./01-settings-reference.md) | 完整字段参考：settings.json 所有字段 + VS Code 插件所有设置项 + 环境变量速查表 |
| [02-recommended-configs.md](./02-recommended-configs.md) | 推荐配置模板：个人 / 项目共享 / 企业管控 / VS Code |
| [03-third-party-models.md](./03-third-party-models.md) | 第三方模型注意事项：8 类常见问题与规避方案 + 云厂商配置 |
| [04-multi-model.md](./04-multi-model.md) | 多模型 / 多实例方案：别名、Worktree、多窗口等变通方案 |
| [05-env-management.md](./05-env-management.md) | 环境变量管理策略：各平台永久/临时设置 + 安全建议 |

---

## 快速导航

### 我想…

**配置第三方中转（One-API / LiteLLM）**
→ [03-third-party-models.md #最小可用配置](./03-third-party-models.md#五最小可用配置第三方中转)

**了解所有 settings.json 字段**
→ [01-settings-reference.md](./01-settings-reference.md)

**了解 VS Code 插件 environmentVariables 格式**
→ [01-settings-reference.md #3.2](./01-settings-reference.md#32-environmentvariables-格式)
> ⚠️ 格式是**数组**，每项为 `{"name": "KEY", "value": "VALUE"}`，不是对象

**配置企业统一管控**
→ [02-recommended-configs.md #五企业管控](./02-recommended-configs.md#五企业管控配置managed-settingsjson)

**同时使用不同模型**
→ [04-multi-model.md](./04-multi-model.md)

**了解环境变量在各平台怎么永久生效**
→ [05-env-management.md](./05-env-management.md)

---

## 核心要点速记

### 配置优先级

```
会话命令(/model) > CLI 参数(--model) > 环境变量 > Local > Project > User > Managed > 默认
```

### VS Code 插件 environmentVariables 正确格式

```json
"claudeCode.environmentVariables": [
  { "name": "ANTHROPIC_AUTH_TOKEN",           "value": "your-token" },
  { "name": "ANTHROPIC_BASE_URL",             "value": "https://your-proxy.com/v1" },
  { "name": "ANTHROPIC_DEFAULT_SONNET_MODEL", "value": "claude-sonnet-4-6" }
]
```

### 使用第三方中转必做

1. 设置 `ANTHROPIC_BASE_URL`（只到 `/v1`，不加 `/messages`）
2. **固定模型别名**（`ANTHROPIC_DEFAULT_*_MODEL`）
3. 禁用 Prompt Cache（`DISABLE_PROMPT_CACHING=1`）
4. VS Code 设置 `claudeCode.disableLoginPrompt: true`

### env 变量是最后手段

优先顺序：`apiKeyHelper` > `settings.json env` > `VS Code environmentVariables` > Shell export

---

## 参考资料

- [官方文档：Settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [官方文档：IDE Integrations](https://docs.anthropic.com/en/docs/claude-code/ide-integrations)
- [官方文档：Bedrock/Vertex Proxies](https://docs.anthropic.com/en/docs/claude-code/bedrock-vertex-proxies)
- [扩展 package.json schema](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code)
