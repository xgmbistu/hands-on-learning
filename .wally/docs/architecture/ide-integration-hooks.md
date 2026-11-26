---
kind: component
scope: ide-token-tooling
status: draft
origin: generated
source_paths: [.claude/hooks/_token_common.py, .claude/hooks/claude_token_collect.py, .codex/hooks/codex_token_collect.py, .cursor/hooks/cursor_token_collect.py, .codebuddy/hooks/stop_token_collect.py, .claude/scripts/token_report.py]
symbols: [normalize_usage, sync_task_store, file_lock, record_claims, parse_transcript, scan_transcript, cursor_usage, scan_ide, render]
upstream: [architecture/index.md, architecture/module-dep-matrix.md]
downstream: [integrations/downstream/git-subprocess.md, operations/runbook-and-observability.md, contracts/configuration.md]
---

# IDE 接入层：Token 采集与报表工具

> 框架模型为 `事件驱动短命工具`。四套采集器分别适配 Claude / Codex / Cursor / CodeBuddy，共享一份同名 `_token_common.py` 账本库；报表脚本跨客户端汇总。本页聚焦真实事件注册、环境变量和账本契约，不推断未配置的客户端行为。

## 1. 事件注册与入口

| 客户端 | 配置文件 | 注册事件 | 采集器入口 |
|---|---|---|---|
| Claude | `.claude/settings.json:3-38` | `Stop`、`UserPromptSubmit`、`PreToolUse` | `.claude/hooks/claude_token_collect.py:166` (`main`) |
| Codex | `.codex/hooks.json:3-62` | `UserPromptSubmit`、`SubagentStart`、`Stop`、`SubagentStop`、`Interrupt`、`SessionEnd`、`PreToolUse` | `.codex/hooks/codex_token_collect.py:592` (`main`) |
| Cursor | `.cursor/hooks.json:3-40` | `beforeSubmitPrompt`、`afterAgentResponse`、`stop`、`sessionStart`、`preToolUse` | `.cursor/hooks/cursor_token_collect.py:200` (`main`) |
| CodeBuddy | `.codebuddy/settings.json:3-30` | `Stop`、`UserPromptSubmit`、`PreToolUse` | `.codebuddy/hooks/stop_token_collect.py:373` (`main`) |

SOP 路由 Hook 与 Token 采集 Hook 并列注册，但互不调用：前者统一走 `.wally/scripts/sop_context.py --hook --client <客户端> --event <事件>`，后者只写账本。超时分别为 3 秒（采集）与 5 秒（SOP）。

## 2. 共享账本底座

| 能力 | 符号与锚点 | 不变量 |
|---|---|---|
| usage 归一化 | `.claude/hooks/_token_common.py:293-360` (`normalize_usage`) | 统一 `input_tokens` / `cache_read_tokens` / `output_tokens` / `credit` 等字段 |
| 累计值合并 | `.claude/hooks/_token_common.py:362-394` (`merge_cumulative_usage`) | 同一会话的累计口径只取最大/合并规则，避免重复累加 |
| 文件锁 | `.claude/hooks/_token_common.py:396-441` (`file_lock`) | 超时抛 `LockTimeout`，不无限等待 |
| 写盘 | `.claude/hooks/_token_common.py:443-453` (`write_json`, `empty_store`) | 统一 JSON 结构，空账本有固定初值 |
| 读写与合并 | `.claude/hooks/_token_common.py:455-541` (`load_store`, `merge_store`) | 坏 JSON 不炸毁账本 |
| claim 去重 | `.claude/hooks/_token_common.py:550-602` (`claim_ledger_path`, `record_claims`) | 请求 ID 账本路径由 `WALLY_TOKEN_CLAIMS` 覆盖 |
| 全量同步 | `.claude/hooks/_token_common.py:638` (`sync_task_store`) | 事件到账本的唯一收口 |

## 3. 客户端专属解析路径

| 客户端 | 数据来源 | 解析函数 | 特殊规则 |
|---|---|---|---|
| Claude | 本地 transcript | `.claude/hooks/claude_token_collect.py:93` (`parse_transcript`) | `:38-60` 校验可信根目录，拒绝越界 transcript |
| Codex | 线程 transcript + meta | `.codex/hooks/codex_token_collect.py:154` (`scan_transcript`)、`:113` (`identity`) | `:322` (`discover_guardians`) 识别审批代理；`:401` (`ledger_digest`) 做账本摘要 |
| Cursor | Hook payload 本身 | `.cursor/hooks/cursor_token_collect.py:161` (`cursor_usage`)、`:174` (`build_record`) | `:91-111` 用 generation_id 暂存/读取 prompt；`:128-159` 多候选定位仓库 |
| CodeBuddy | IDE 消息 + CLI JSONL | `.codebuddy/hooks/stop_token_collect.py:236` (`scan_ide`)、`:319` (`scan_cli`) | `:281` (`cli_usage`) 单独归一 CLI 口径；`:168` (`clean_prompt`) 清洗 prompt |

Codex 的 `state_dir` 位于 `.codex/hooks/codex_token_collect.py:40-42`，优先读 `WALLY_CODEX_STATE_DIR`，否则回落到 `CODEX_HOME` 或用户主目录。

## 4. Token 报表渲染链

| 步骤 | 符号与锚点 | 语义 |
|---|---|---|
| 定位仓库 | `.claude/scripts/token_report.py:81-88` (`find_repo`) | 向上查找含 `.wally` 目录的路径 |
| 跳过分支 | `.claude/scripts/token_report.py:58-62` (`token_work_disabled`) | main、master、release* 返回 0 并 `[SKIP]` |
| 找任务 | `.claude/scripts/token_report.py:91-106` (`find_overview`) | 按 Branch 正则匹配，多个取修改时间最新 |
| 回填 Codex | `.claude/scripts/token_report.py:401-412` (`refresh_codex`) | 子进程调用采集器 `--refresh-task`，失败只告警 |
| 轮次合并 | `.claude/scripts/token_report.py:195-230` (`rollup_turns`) | Cursor 一轮一行不求和；Codex 按 thread+root_turn_id；CodeBuddy 按同会话同 prompt |
| 表格渲染 | `.claude/scripts/token_report.py:278-365` (`render`) | 表头含 13 列，代理 Credit 记 `—`，不做成本估算 |
| 健康说明 | `.claude/scripts/token_report.py:368-398` (`render_health`) | 区分 `ok`/`partial`/`missing`/`unsupported`/`pending` |
| 写回 | `.claude/scripts/token_report.py:415-425` (`patch_overview`) | 仅替换 `TOKEN_USAGE_START` 到 `TOKEN_USAGE_END` 之间内容 |

会计恒等式由 `.claude/scripts/token_report.py:354-362` 明确写死：总输入 = 未缓存输入 + 缓存读 + 缓存写；Tokens = 总输入 + 输出；推理 token 属于输出，不二次计数。

## 5. 失败降级与边界

- Hook 模式顶层异常兜底在 `.wally/scripts/sop_context.py:469-474`：输出 `{}` 且退出 0，避免 Hook 失败变成全局工具禁止。
- 采集侧坏 payload 经 `.claude/hooks/_token_common.py:153-194` 的 `read_stdin_json` 被吞掉；`.codex/hooks/codex_token_collect.py:304-309` 的 `discovery_budget_low` 限制代理发现开销。
- 报表侧缺失账本只追加告警（`.claude/scripts/token_report.py:466-467`），绝不输出零用量结论。全部变量见 [配置手册](../contracts/configuration.md)，排障路径见 [Runbook](../operations/runbook-and-observability.md)。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改账本字段：`.claude/hooks/_token_common.py:293-360` → 四套采集器的 usage 构造 → `.claude/scripts/token_report.py:278-365` → [字段映射](../integrations/topology-and-mapping.md)。
2. 改事件注册：对应 `settings.json` / `hooks.json` → 本页 → [API 参考](../contracts/api-reference.md)。
3. 改报表表头：`.claude/scripts/token_report.py:346-365` → [可观测性 Runbook](../operations/runbook-and-observability.md) → [认知边界](../operations/boundary.md) 的计量口径声明。
4. 必跑验证：对四套采集器与报表脚本执行 `python3 -m py_compile`；向报表脚本传入测试分支确认 `[SKIP]` 路径；核对 `TOKEN_USAGE_START` 标记在目标 overview 中唯一。
