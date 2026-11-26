---
kind: downstream-contract
scope: codex-token-collector
status: draft
origin: generated
source_paths: [.claude/scripts/token_report.py, .codex/hooks/codex_token_collect.py]
symbols: [refresh_codex, refresh_task, main, synchronize, ledger_digest]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: [architecture/ide-integration-hooks.md, operations/runbook-and-observability.md]
---

# 下游契约：Codex 采集器回填子进程

> 下游为本仓内的 `.codex/hooks/codex_token_collect.py`，以独立 Python 子进程方式被 Token 报表调用。调用是可选的：采集器文件不存在时直接跳过，不报错。证据来自 `.claude/scripts/token_report.py:401-412` 与 `.codex/hooks/codex_token_collect.py:574-590`。配套阅读：[集成总览](../index.md) · [IDE 接入层](../../architecture/ide-integration-hooks.md)。

## 1. 调用方与触发条件

| 项 | 值 | 锚点 |
|---|---|---|
| 调用方 | `refresh_codex(repo, overview)` | `.claude/scripts/token_report.py:401` |
| 触发时机 | `main()` 在读账本之前 | `.claude/scripts/token_report.py:455-457` |
| 前置条件 | 采集器文件存在 | `.claude/scripts/token_report.py:402-404` |
| 下游入口 | `codex_token_collect.py --refresh-task <任务目录>` | `.claude/scripts/token_report.py:406` |
| 下游处理函数 | `refresh_task(task)` | `.codex/hooks/codex_token_collect.py:574` |

## 2. 请求组装字典

`subprocess.run` 的完整参数字典：

| 参数键 | 实际值 | 语义 | 锚点 |
|---|---|---|---|
| `args[0]` | `sys.executable` | 与当前解释器一致 | `.claude/scripts/token_report.py:406` |
| `args[1]` | `str(repo / ".codex/hooks/codex_token_collect.py")` | 采集器绝对路径 | `.claude/scripts/token_report.py:402`、`:406` |
| `args[2]` | `"--refresh-task"` | 下游子命令开关 | `.claude/scripts/token_report.py:406` |
| `args[3]` | `str(overview.parent)` | 任务目录路径 | `.claude/scripts/token_report.py:406` |
| `capture_output` | `True` | 捕获 stdout/stderr，不回显 | `.claude/scripts/token_report.py:407` |
| `timeout` | `10` | 10 秒超时 | `.claude/scripts/token_report.py:407` |

等价命令：`<当前 Python> .codex/hooks/codex_token_collect.py --refresh-task <任务目录>`。任务目录来自 `find_overview(repo, branch)` 的父目录（`.claude/scripts/token_report.py:450-453`），路径由 `find_repo` 的向上查找确定（`:81-88`）。

## 3. 返回解包字段

| 返回字段 | 读取方式 | 取值 | 翻译结果 |
|---|---|---|---|
| `returncode` | `result.returncode`（`.claude/scripts/token_report.py:408`） | `0` | 返回空串，无告警 |
| `returncode` | 同上 | 非 `0` | `Codex backfill is incomplete; values include saved usage only.` |
| 异常 `OSError` | `except` 分支（`:410`） | — | `Codex backfill is incomplete or timed out; values include saved usage only.` |
| 异常 `TimeoutExpired` | 同上 | — | 同上 |

告警串拼到 `warning` 变量，最终进入 `render_health` 的 `Collection note` 行（`.claude/scripts/token_report.py:394-395`）。`capture_output` 捕获的 stdout/stderr 在失败时不被解析，只依据退出码判断。

## 4. 下游内部契约

| 能力 | 符号与锚点 | 入参 | 出参 |
|---|---|---|---|
| 刷新入口 | `refresh_task`（`.codex/hooks/codex_token_collect.py:574`） | 任务目录 Path | 无返回值，内部写账本 |
| 状态同步 | `synchronize`（`:431`） | state 字典 + `only_task` | 同步后的 state |
| 账本摘要 | `ledger_digest`（`:401`） | 任务目录 + session_id | 摘要字符串 |
| 主入口 | `main`（`:592`） | argparse 参数 | 退出码 |

`refresh_task` 只回填已绑定到该任务的 Codex 日志；报表不假设回填成功，也不把缺失日志解释为零用量（`.claude/scripts/token_report.py:396-397`）。

## 5. 错误翻译与降级表

| 场景 | 报表行为 | 用户可见 | 锚点 |
|---|---|---|---|
| 采集器文件不存在 | 早退返回空串 | 无告警 | `.claude/scripts/token_report.py:402-404` |
| 子进程非 0 退出 | 告警串 | `Codex backfill is incomplete...` | `.claude/scripts/token_report.py:408-409` |
| 启动失败或超时 | 告警串 | `Codex backfill is incomplete or timed out...` | `.claude/scripts/token_report.py:410-411` |
| 账本缺失或坏 JSON | 追加告警 | `Usage ledger missing or unreadable; do not assume zero usage.` | `.claude/scripts/token_report.py:466-467` |
| 无请求记录 | 健康行 | `No usage records collected; this does not mean complete zero usage.` | `.claude/scripts/token_report.py:396-397` |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改子进程参数：`.claude/scripts/token_report.py:401-412` → 本页第 2/3 节 → [字段映射](../topology-and-mapping.md) → [IDE 接入层](../../architecture/ide-integration-hooks.md)。
2. 改下游刷新语义：`.codex/hooks/codex_token_collect.py:574-590` → 本页第 4 节 → [数据模型](../../business/domain-models/index.md) → [可观测性 Runbook](../../operations/runbook-and-observability.md)。
3. 改告警文案：`.claude/scripts/token_report.py:408-411` → [错误码字典](../../contracts/error-codes.md) → [排障决策树](../../operations/runbook-and-observability.md)。
4. 必跑验证：对两个脚本执行 `python3 -m py_compile`；临时移走采集器文件后运行报表，确认不因缺失而失败；构造非 0 退出的桩脚本确认 `Collection note` 行出现。
