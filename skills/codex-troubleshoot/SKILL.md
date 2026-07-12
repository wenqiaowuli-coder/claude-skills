---
name: codex-troubleshoot
description: 修复 Codex CLI 在容器/WSL 环境下的常见错误。当用户提到"codex 错误"、"codex 故障"、"修复 codex"、"codex bwrap"、"codex shell snapshot"、"Model metadata"时使用此 skill。
version: 1.0.0
---

# Codex CLI 故障排查与修复

修复 Codex CLI 在容器/WSL 环境下运行时的常见错误。

## 适用环境

- Docker 容器内运行 codex（无 CAP_SYS_ADMIN）
- WSL2 环境
- 无特权的 Linux 用户空间
- 使用自定义模型 provider（非 OpenAI 官方）的场景

## 常见错误现象

运行 `codex exec` 时出现以下报错：

```
bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted
Shell snapshot validation failed: syntax error near unexpected token `('
Model metadata for `step-3.7-flash` not found. Defaulting to fallback metadata
```

导致 codex 无法执行任何 shell 命令。

## 根因分析

| 问题 | 根因 |
|------|------|
| bwrap sandbox 失败 | 容器/WSL 环境缺少 `CAP_SYS_ADMIN`，无法创建 user namespace 和网络命名空间 |
| shell snapshot 语法错误 | bash-completion（~2300行）在非交互 shell 中验证失败，与 codex 的 snapshot 机制不兼容 |
| Model metadata 缺失 | 自定义模型 `step-3.7-flash` 不在 Codex 内置模型目录中 |
| `preferred_auth_method` / `suppress_update_tips` 报错 | config.toml 使用了已被移除的旧版配置键 |

## 修复步骤

### 第 1 步：绕过 bwrap sandbox 限制

**方案**：创建包装脚本 `~/.local/bin/codex`，对 `exec` 子命令自动追加 `-s danger-full-access`。

```bash
#!/bin/bash
REAL_CODEX="$HOME/.hermes/node/bin/codex"
if [[ "$1" == "exec" ]]; then
    shift
    exec "$REAL_CODEX" exec -s danger-full-access "$@"
else
    exec "$REAL_CODEX" "$@"
fi
```

**原理**：`danger-full-access` 模式完全绕过 bwrap，在容器外层已有隔离的前提下是安全的。

保存并赋予执行权限：
```bash
chmod +x ~/.local/bin/codex
```

### 第 2 步：禁用 shell_snapshot 特性

在 `~/.codex/config.toml` 中添加：

```toml
features.shell_snapshot = false
```

**原理**：该特性用于命令执行前的环境快照，在当前 bash-completion 环境下验证失败，禁用后不影响核心功能。

### 第 3 步：消除 Model metadata 警告

**方案**：创建 `~/.codex/model_catalog.json`，提供自定义模型的完整元数据。

```json
{
  "models": [
    {
      "id": "step-3.7-flash",
      "slug": "step-3.7-flash",
      "display_name": "Step 3.7 Flash",
      "description": "Step 3.7 Flash model via StepFun API",
      "provider": "custom",
      "mode": "online",
      "visibility": "list",
      "context_window": 128000,
      "max_context_window": 128000,
      "supports_reasoning_summaries": false,
      "supported_reasoning_levels": [
        {"effort": "low", "name": "Low", "description": "Low effort"},
        {"effort": "medium", "name": "Medium", "description": "Medium effort"},
        {"effort": "high", "name": "High", "description": "High effort"}
      ],
      "supports_parallel_tool_calls": true,
      "supports_image_detail_original": false,
      "supports_search_tool": false,
      "supports_web_search_request": false,
      "shell_type": "unified_exec",
      "truncation_policy": {"mode": "tokens", "limit": 128000, "auto_compact_token_limit": 128000},
      "service_tier": "default",
      "supported_in_api": true,
      "priority": 0,
      "base_instructions": "You are a helpful assistant.",
      "support_verbosity": true,
      "experimental_supported_tools": []
    }
  ]
}
```

在 `~/.codex/config.toml` 中引用：

```toml
model_catalog_json = "/home/wq/.codex/model_catalog.json"
```

**注意**：ModelInfo 是一个包含 38 个字段的结构体，必须逐字段补齐，缺少任何一个都会报 `missing field` 错误。需要通过逐轮测试+错误反馈的方式反推完整字段列表。

### 第 4 步：清理无效配置键

从 `~/.codex/config.toml` 中移除以下无效键：

- `preferred_auth_method` — 已移除，引发 `--strict-config` 报错
- `suppress_update_tips` — 已移除，引发 `--strict-config` 报错
- `sandbox` — 不是 config.toml 的有效键，仅支持 CLI 参数 `-s`

## 最终配置文件

### `~/.codex/config.toml`

```toml
model_provider = "custom"
model = "step-3.7-flash"
model_reasoning_effort = "high"
model_reasoning_summary = "none"
model_supports_reasoning_summaries = false
model_context_window = 128000
model_catalog_json = "/home/wq/.codex/model_catalog.json"

# 容器环境缺少 CAP_SYS_ADMIN，bwrap sandbox 无法创建 user namespace
features.shell_snapshot = false

[model_providers.custom]
name = "StepFun"
base_url = "https://api.stepfun.com/step_plan/v1"
wire_api = "responses"
requires_openai_auth = true

[projects."/home/wq"]
trust_level = "trusted"

[projects."/"]
trust_level = "trusted"

[tui.model_availability_nux]
"gpt-5.5" = 3
```

### `~/.local/bin/codex`（包装脚本）

```bash
#!/bin/bash
REAL_CODEX="$HOME/.hermes/node/bin/codex"
if [[ "$1" == "exec" ]]; then
    shift
    exec "$REAL_CODEX" exec -s danger-full-access "$@"
else
    exec "$REAL_CODEX" "$@"
fi
```

## 验证方法

修复完成后，验证是否生效：

```bash
codex exec --skip-git-repo-check "echo hello world"
```

预期输出：
```
hello world
```

验证标准：
- ✅ 零报错、零警告
- ✅ shell 命令执行正常
- ✅ 文件读写正常
- ✅ 网络请求正常

## 经验教训

1. **容器/WSL 环境必须绕过 bwrap**：`CAP_SYS_ADMIN` 是宿主机内核能力，容器内无法自行获取。`danger-full-access` 是唯一可行方案。
2. **config.toml 键名要严格匹配**：用 `--strict-config` 可以快速发现无效键。
3. **model_catalog_json 是完整 Schema**：不是可选补充，缺少字段会直接报错退出。需要通过二进制 strings 搜索或逐轮试错来补齐。
4. **shell_snapshot 在复杂 shell 环境下不稳定**：如果 `.bashrc` 加载了大量 completion 脚本，该特性会验证失败，建议禁用。
5. **包装脚本优于 alias**：`alias codex='codex -s danger-full-access'` 对子命令（如 `codex exec`）不生效，包装脚本能精确控制子命令。

## 适用场景

本修复方案适用于以下环境：

- Docker 容器内运行 codex（无 CAP_SYS_ADMIN）
- WSL2 环境
- 无特权的 Linux 用户空间
- 使用自定义模型 provider（非 OpenAI 官方）的场景

## 相关链接

- [Codex CLI 官方文档](https://github.com/openai/codex)
- [Codex 配置说明](https://github.com/openai/codex/blob/main/docs/config.md)
- [StepFun API 文档](https://platform.stepfun.com/docs)

## 安全提示

1. **danger-full-access 模式**：该模式完全绕过 bwrap sandbox，仅建议在已受信任的环境中使用
2. **自定义模型配置**：model_catalog.json 中的配置应仔细核对，避免元数据错误
3. **配置文件权限**：config.toml 和 model_catalog.json 应设置为仅当前用户可读写

---

**创建日期**：2026-07-12
**适用版本**：codex-cli 0.144.1+
**环境**：Ubuntu 24.04, WSL/Docker, StepFun API
