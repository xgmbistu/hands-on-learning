---
kind: api-contract
scope: inbound-api-reference
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .codex/hooks/codex_token_collect.py, .claude/settings.json, .codex/hooks.json, .cursor/hooks.json, .codebuddy/settings.json]
symbols: [main, create, route, parse_message, COMMAND_HANDERS, accept_confirmation, check_tool]
upstream: [INDEX.md, integrations/index.md]
downstream: [contracts/error-codes.md, business/kv-command-workflow.md, architecture/ide-integration-hooks.md]
---

# 入向接口参考（全量）

> 全仓入向接口分三类：socket 文本命令、CLI 参数入口、IDE Hook 事件。本页逐条列全，不使用省略表述。协议细节见 [线协议契约](../integrations/downstream/socket-wire-protocol.md)，错误出口见 [错误码字典](error-codes.md)。

## 1. Socket 文本命令（8 个）

入口：`nosql.py:167-193` `main()`，监听 `localhost:50505`（`nosql.py:9-10`），报文格式 `COMMAND; KEY; VALUE; VALUE_TYPE`。

| 命令 | 参数键 | 成功响应 | 失败响应 | 处理函数锚点 |
|---|---|---|---|---|
| `PUT` | `key`, `value` | `key [k] set to [v]` | 无 | `nosql.py:76-80` |
| `GET` | `key` | `DATA[key]` | `Error: Key [k] not found` | `nosql.py:83-89` |
| `GETLIST` | `key` | list 值 | 键缺失或非 list | `nosql.py:98-108` |
| `PUTLIST` | `key`, `value` | 复用 PUT 文案 | 无 | `nosql.py:92-95` |
| `INCREMENT` | `key` | `Key [k] incremented` | 键缺失或非 int | `nosql.py:111-122` |
| `APPEND` | `key`, `value` | `Key [k] had value [v] appended` | 键缺失或非 list | `nosql.py:125-136` |
| `DELETE` | `key` | 无返回值 | `ERROR: Key [k] not found and could not be deleted.` | `nosql.py:139-148` |
| `STATS` | 无 | `str(STATS)` | 无 | `nosql.py:150-152` |

响应帧固定为 `'{};\n{}\n'.format(response[0], response[1])`（`nosql.py:191`）。`value_type` 仅接受 `LIST`、`INT` 两种转换字面（`nosql.py:58-61`）。分派表见 `nosql.py:155-164`。

## 2. CLI 参数入口

| 入口 | 参数 | 必填 | 默认值/取值域 | 锚点 |
|---|---|---|---|---|
| `create_task.py` | 位置参数 `name` | 是 | `YYYY-MM-DD_title` 正则 | `.wally/scripts/create_task.py:166`、`:71-73` |
| `create_task.py` | `--branch` | 是 | 非 main/master/HEAD，不含管道符/反引号/换行 | `.wally/scripts/create_task.py:167`、`:74-75` |
| `create_task.py` | `--repo-root` | 否 | `.` | `.wally/scripts/create_task.py:168` |
| `sop_context.py` | `--repo` | 否 | `.` | `.wally/scripts/sop_context.py:396` |
| `sop_context.py` | `--hook` | 否 | 开关，从 stdin 读 JSON | `.wally/scripts/sop_context.py:397`、`:405-407` |
| `sop_context.py` | `--checkpoint` | 否 | `implement` / `deploy` / `it` | `.wally/scripts/sop_context.py:398` |
| `sop_context.py` | `--client` | 否 | `codex` / `codebuddy` / `cursor` / `claude`，默认 `codex` | `.wally/scripts/sop_context.py:399` |
| `sop_context.py` | `--event` | 否 | 任意字符串，默认 `PreToolUse` | `.wally/scripts/sop_context.py:400` |
| `token_report.py` | 位置参数 `repo` | 否 | 未传时读环境变量或 cwd | `.claude/scripts/token_report.py:430`、`:435-441` |
| `token_report.py` | `--branch` | 否 | 未传时由 `git_branch` 推断 | `.claude/scripts/token_report.py:431`、`:446` |
| `token_report.py` | `--overview` | 否 | 未传时 `find_overview(repo, branch)` | `.claude/scripts/token_report.py:432`、`:450` |
| `codex_token_collect.py` | `--refresh-task` | 否 | 任务目录路径，由报表子进程传入 | `.claude/scripts/token_report.py:406`、`.codex/hooks/codex_token_collect.py:574` |

`--hook` 特殊语义：顶层异常兜底输出 `{}` 且退出 0（`.wally/scripts/sop_context.py:469-474`），不阻断编辑器。

## 3. Hook 事件（20 个注册）

| 客户端 | 事件 | 处理入口 | 超时 | 配置锚点 |
|---|---|---|---|---|
| Claude | `Stop` | `.claude/hooks/claude_token_collect.py:166` | 3s | `.claude/settings.json:3`、`:9` |
| Claude | `UserPromptSubmit` | `.wally/scripts/sop_context.py:394` | 5s | `.claude/settings.json:14`、`:19` |
| Claude | `PreToolUse` | `.wally/scripts/sop_context.py:394` | 5s | `.claude/settings.json:25`、`:31` |
| Codex | `UserPromptSubmit` | `.codex/hooks/codex_token_collect.py:592` | 3s | `.codex/hooks.json:3`、`:9` |
| Codex | `UserPromptSubmit` | `.wally/scripts/sop_context.py:394` | 5s | `.codex/hooks.json:3`、`:17` |
| Codex | `SubagentStart` | `.codex/hooks/codex_token_collect.py:592` | 3s | `.codex/hooks.json:23`、`:29` |
| Codex | `Stop` | `.codex/hooks/codex_token_collect.py:592` | 3s | `.codex/hooks.json:34`、`:40` |
| Codex | `SubagentStop` | `.codex/hooks/codex_token_collect.py:592` | 3s | `.codex/hooks.json:45`、`:51` |
| Codex | `Interrupt` | `.codex/hooks/codex_token_collect.py:592` | 3s | `.codex/hooks.json:56`、`:62` |
| Codex | `SessionEnd` | `.codex/hooks/codex_token_collect.py:592` | 3s | `.codex/hooks.json:67`、`:73` |
| Codex | `PreToolUse` | `.wally/scripts/sop_context.py:394` | 5s | `.codex/hooks.json:78`、`:83` |
| Cursor | `beforeSubmitPrompt` | `.cursor/hooks/cursor_token_collect.py:200` | 3s | `.cursor/hooks.json:3`、`:7` |
| Cursor | `beforeSubmitPrompt` | `.wally/scripts/sop_context.py:394` | 5s | `.cursor/hooks.json:3`、`:11` |
| Cursor | `afterAgentResponse` | `.cursor/hooks/cursor_token_collect.py:200` | 3s | `.cursor/hooks.json:14`、`:18` |
| Cursor | `stop` | `.cursor/hooks/cursor_token_collect.py:200` | 3s | `.cursor/hooks.json:21`、`:25` |
| Cursor | `sessionStart` | `.wally/scripts/sop_context.py:394` | 5s | `.cursor/hooks.json:28`、`:31` |
| Cursor | `preToolUse` | `.wally/scripts/sop_context.py:394` | 5s | `.cursor/hooks.json:34`、`:37` |
| CodeBuddy | `Stop` | `.codebuddy/hooks/stop_token_collect.py:373` | 3s | `.codebuddy/settings.json:3`、`:9` |
| CodeBuddy | `UserPromptSubmit` | `.wally/scripts/sop_context.py:394` | 5s | `.codebuddy/settings.json:15`、`:20` |
| CodeBuddy | `PreToolUse` | `.wally/scripts/sop_context.py:394` | 5s | `.codebuddy/settings.json:26`、`:31` |

共 20 条注册：其中 Token 采集 11 条、SOP 路由 9 条。Cursor 的 `preToolUse` 设 `failClosed: false`（`.cursor/hooks.json:38`）。

## 4. Hook payload 消费字段

| 字段 | 类型 | 消费函数 | 用途 |
|---|---|---|---|
| `tool_name` | 字符串 | `tool_kind`（`.wally/scripts/sop_context.py:201-207`） | 区分 shell / edit / other |
| `tool_input` | 字典或字符串 | `check_tool`（`:374-377`） | 取 `file_path` / `filePath` / `path` 与 patch 文本 |
| `prompt` | 字符串 | `accept_confirmation`（`:348-366`） | 解析 12 位确认 token |
| stdin 任意 JSON | 字典 | `read_stdin_json`（`.claude/hooks/_token_common.py:153-194`） | Token 采集器通用输入 |

## 5. 输出契约

| 入口 | 成功输出 | 失败输出 |
|---|---|---|
| socket 命令 | `bool;\nmessage\n` | 同帧格式（失败消息）或连接异常 |
| `create_task.py` | `Created <路径> (SOP 2.0 / <revision>)` | `[ERROR] ...` 到 stderr，退出 1 |
| `sop_context.py` 普通模式 | 绑定文档拼接文本 | `SOP context warning: ...` 到 stdout，退出 1 |
| `sop_context.py` checkpoint | 就绪提示或确认提示 | 待确认时退出 3 |
| `sop_context.py` hook | Cursor 用 `permission` / `additional_context`；其他用 `hookSpecificOutput` | 异常兜底 `{}` |
| `token_report.py` | `[OK] Token rollup: ...` 到 stdout | `[ERROR]` 退出 1 或 `[SKIP]` 退出 0 |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改 socket 命令：`nosql.py:155-164` → 本页第 1 节 → [KV 工作流](../business/kv-command-workflow.md) → [错误码字典](error-codes.md)。
2. 改 CLI 参数：`.wally/scripts/create_task.py:164-169`、`.wally/scripts/sop_context.py:394-401` → 本页第 2 节 → [SOP 工作流](../business/sop-task-workflow.md)。
3. 改 Hook 注册：对应 settings/hooks JSON → 本页第 3 节 → [IDE 接入层](../architecture/ide-integration-hooks.md) → `.wiki-meta.json` 的 `fileToDocs`。
4. 必跑验证：起服后逐条发送 8 个命令；对 4 个 Python 入口执行 `python3 -m py_compile`；`python3 .wally/scripts/sop_context.py --hook --client cursor --event preToolUse` 输入 `{}` 检查 JSON 结构。
