---
kind: config-dict
scope: configuration-and-environment
status: draft
origin: generated
source_paths: [nosql.py, .wally/config.json, .wally/install-state.json, .claude/hooks/_token_common.py, .claude/hooks/claude_token_collect.py, .codex/hooks/codex_token_collect.py, .cursor/hooks/cursor_token_collect.py, .codebuddy/hooks/stop_token_collect.py, .claude/scripts/token_report.py]
symbols: [HOST, PORT, execution_config, CLAIM_LEDGER_ENV, state_dir, repo_candidates, git_branch, token_work_disabled]
upstream: [INDEX.md, contracts/api-reference.md]
downstream: [business/sop-task-workflow.md, operations/deployment-and-runtime.md]
---

# 配置、环境变量与运行时开关手册

> 全仓仅用 `os.environ.get` 读取环境变量，未出现 `os.getenv`；所有读取点已逐项收录。密钥不明文落库、不回显；本仓当前无任何凭据配置。配套阅读：[错误码字典](error-codes.md) · [部署与运行时](../operations/deployment-and-runtime.md)。

## 1. 环境变量全量字典

| 变量名 | 读取点 | 用途 | 回退/优先级 |
|---|---|---|---|
| `CLAUDE_CONFIG_DIR` | `.claude/hooks/claude_token_collect.py:42` | 定位 Claude 配置目录 | 未设置时用默认推断 |
| `WALLY_TOKEN_CLAIMS` | `.claude/hooks/_token_common.py:67`、`:551` | 覆盖 claim 账本路径 | 未设置时用默认路径 |
| `CODEBUDDY_PROJECT_DIR` | `.claude/scripts/token_report.py:437`、`.codebuddy/hooks/stop_token_collect.py:377` | 定位仓库根 | 报表第一优先级；采集器默认来源 |
| `CLAUDE_PROJECT_DIR` | `.claude/scripts/token_report.py:438`、`.cursor/hooks/cursor_token_collect.py:131`、`.codebuddy/hooks/stop_token_collect.py:378` | 定位仓库根 | 报表第二优先级；Cursor 第二优先级 |
| `CURSOR_PROJECT_DIR` | `.claude/scripts/token_report.py:439`、`.cursor/hooks/cursor_token_collect.py:131` | 定位仓库根 | 报表第三优先级；Cursor 第一优先级 |
| `WALLY_CODEX_STATE_DIR` | `.codex/hooks/codex_token_collect.py:42` | 覆盖 Codex 采集状态目录 | 未设置时用用户目录下默认位置 |
| `CODEX_HOME` | `.codex/hooks/codex_token_collect.py:299` | 补充 Codex 自定义 home | 未设置时回落到 `Path.home() / ".codex"` |
| `CODEBUDDY_EXTENSION_DATA` | `.codebuddy/hooks/stop_token_collect.py:79` | 定位 CodeBuddy 扩展数据 | 未设置时用默认推断 |

> 同名变量在 `.codex/scripts/token_report.py:437-439` 与 `.codebuddy/scripts/token_report.py:437-439` 的副本中重复出现，来源为同一份脚本，不重复计数。

## 2. 变量优先级链

| 场景 | 优先级顺序 | 锚点 |
|---|---|---|
| Token 报表定位仓库 | 位置参数 `repo` → `CODEBUDDY_PROJECT_DIR` → `CLAUDE_PROJECT_DIR` → `CURSOR_PROJECT_DIR` → `os.getcwd()` → `find_repo` 向上查找 | `.claude/scripts/token_report.py:435-442` |
| Cursor 采集定位仓库 | `CURSOR_PROJECT_DIR` → `CLAUDE_PROJECT_DIR` → payload `workspace_roots` → payload `cwd` → `os.getcwd()` → `resolve_repo` | `.cursor/hooks/cursor_token_collect.py:128-142` |
| Codex 状态目录 | `WALLY_CODEX_STATE_DIR` → 默认目录；`CODEX_HOME` 另用于 homes 集合 | `.codex/hooks/codex_token_collect.py:40-42`、`:299` |
| claim 账本 | `WALLY_TOKEN_CLAIMS` → 默认路径 | `.claude/hooks/_token_common.py:550-551` |
| 分支推断 | 位置参数 `--branch` → `git branch --show-current` | `.claude/scripts/token_report.py:446` |

## 3. 应用硬编码常量

| 常量 | 值 | 类型 | 是否可覆盖 | 锚点 |
|---|---|---|---|---|
| `HOST` | `localhost` | 字符串 | 否 | `nosql.py:9` |
| `PORT` | `50505` | 整数 | 否 | `nosql.py:10` |
| `SOCKET` | `socket.socket(AF_INET, SOCK_STREAM)` | 模块级对象 | 否，导入即创建 | `nosql.py:11` |
| `STATS` 键 | 8 个命令名 | 字典 | 否 | `nosql.py:12-45` |
| `DATA` | 空字典 | 字典 | 否 | `nosql.py:47` |
| `COMMAND_HANDERS` | 8 项命令映射 | 字典 | 否 | `nosql.py:155-164` |
| `CLAIM_LEDGER_ENV` | `"WALLY_TOKEN_CLAIMS"` | 字符串 | 否 | `.claude/hooks/_token_common.py:67` |
| `CLIENT_*` | `codebuddy` / `cursor` / `codex` / `claude` | 字符串 | 否 | `.claude/scripts/token_report.py:37-40` |
| `CODEBUDDY_SOURCES` | `("ide_index", "cli_jsonl")` | 元组 | 否 | `.claude/scripts/token_report.py:41` |
| `START` / `END` | `TOKEN_USAGE_START` / `TOKEN_USAGE_END` | 字符串 | 否 | `.claude/scripts/token_report.py:31-32` |
| `TOKEN_FILE_NAME` | `token.json` | 字符串 | 否 | `.claude/scripts/token_report.py:33` |
| `BRANCH` / `RULES` / `PHASE` | 三条发现正则 | 字符串 | 否 | `.wally/scripts/sop_context.py:19-21` |
| `FILENAMES` | 六阶段文件名映射 | 字典 | 否 | `.wally/scripts/sop_context.py:22-23` |

## 4. config.json 执行方法字典

| 键路径 | deploy | it | 校验规则 | 锚点 |
|---|---|---|---|---|
| `schema` | `1`（文件顶层） | 同左 | 必须等于 1 | `.wally/config.json:2`、`.wally/scripts/create_task.py:39-40` |
| `command` | 空串 | 空串 | 字符串类型；非空才算已配置 | `.wally/config.json:4`、`:17`、`:22-23` |
| `guide` | `.wally/docs/devops/test-env-deploy.md` | `.wally/docs/integration_test/integration_test.md` | 字符串类型 | `.wally/config.json:5`、`:18` |
| `verify` / `success_criteria` | `Not configured: ...` | `Not configured: ...` | 字符串类型 | `.wally/config.json:6`、`:19` |
| `rollback` | `Not configured: ...` | 无此键 | 字符串类型 | `.wally/config.json:7` |
| `readiness` | `pending` | `pending` | 取值域 `None`/`pending`/`ready` | `.wally/config.json:8`、`:20`、`.wally/scripts/create_task.py:46-47` |
| `gaps` | 4 条字符串 | 4 条字符串 | 必须是字符串数组 | `.wally/config.json:9-14`、`:21-26`、`:48-50` |

`ready` 状态必须至少配置 command 或 guide（`.wally/scripts/create_task.py:51-52`）；`pending` 强制 BLOCKED（`:120`），不自动跳过。

## 5. install-state.json 与运行时策略

| 字段 | 当前值 | 影响 | 锚点 |
|---|---|---|---|
| `schema` | `2` | 建任务前置校验 | `.wally/install-state.json:2`、`.wally/scripts/create_task.py:86` |
| `steps` | `6` | 模板必须六阶段 | `.wally/install-state.json:5`、`:57-58` |
| `revision` | `3f84e7245abcc75d` | 绑定不可变规则目录 | `.wally/install-state.json:4`、`:91` |
| `language` | `en` | 决定建任务文案语言 | `.wally/install-state.json`、`:110`、`:123-128` |
| `translation_sources` | 空 | 非空时拒绝建任务 | `:84-85` |
| `docs` | 7 个文档域 | 知识路由边界 | `.wally/install-state.json` |
| `commit` | `no` | 提交授权不由工具代决 | `.wally/install-state.json` |

运行时策略开关：`token_work_disabled`（`.claude/scripts/token_report.py:58-62`）对 `main`、`master`、`release` 及 `release/` 前缀返回真；`.claude/hooks/_token_common.py:223-234` 存在同名同语义实现。`discovery_budget_low`（`.codex/hooks/codex_token_collect.py:304-309`）限制 Codex 代理发现开销。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改环境变量：本页第 1/2 节 → [API 参考](api-reference.md) → [IDE 接入层](../architecture/ide-integration-hooks.md) → [部署与运行时](../operations/deployment-and-runtime.md)。
2. 改 `config.json`：`.wally/config.json` → 本页第 4 节 → [SOP 工作流](../business/sop-task-workflow.md) → [错误码字典](error-codes.md)。
3. 改硬编码地址：`nosql.py:9-11` → 本页第 3 节 → [系统架构](../overview/system-architecture.md) → [认知边界](../operations/boundary.md)。
4. 必跑验证：在 `main` 分支运行 `python3 .claude/scripts/token_report.py` 应输出 `[SKIP]`；`python3 -m py_compile` 覆盖全部脚本；核对本页环境变量清单与源码 `os.environ.get` 命中数一致。
