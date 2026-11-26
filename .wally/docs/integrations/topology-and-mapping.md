---
kind: mapping
scope: upstream-to-downstream-field-mapping
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/sop_context.py, .claude/hooks/_token_common.py, .claude/scripts/token_report.py]
symbols: [parse_message, handle_put, route, checkpoint_issue, normalize_usage, merge_store, rollup_turns, render]
upstream: [INDEX.md, integrations/index.md]
downstream: [integrations/downstream/socket-wire-protocol.md, integrations/downstream/git-subprocess.md, integrations/downstream/codex-collector.md]
---

# 入参 DTO → 中间转换器 → 下游字典键映射

> 映射一律取自真实转换函数，不省略任何下游键。三条链互不共享转换器：KV 走 `parse_message`，SOP 走 `route`，Token 走 `normalize_usage`。配套阅读：[集成总览](index.md) · [API 参考](../contracts/api-reference.md)。

## 1. KV 报文到处理函数映射

上游入参为 `nosql.py:177` 的 `connection.recv(4096).decode()`，转换器为 `parse_message`（`nosql.py:50-64`）。

| 上游报文字段 | 转换规则 | 中间变量 | 下游处理函数入参 | 锚点 |
|---|---|---|---|---|
| 第 1 段 `command` | `str.strip`，无大小写归一 | `command` | 分派表键 | `nosql.py:54`、`:155-164` |
| 第 2 段 `key` | `str.strip` | `key` | `handler(key, ...)` | `nosql.py:54`、`:181-187` |
| 第 3 段 `value` | 依 `value_type` 转 `LIST` / `INT`，否则 `None` | `value` | `handler(key, value)` | `nosql.py:57-63` |
| 第 4 段 `value_type` | 只作为转换开关，不传给处理函数 | `value_type` | 无 | `nosql.py:54`、`:57` |

| 命令分支 | 下游函数 | 实际入参个数 | 返回值结构 |
|---|---|---|---|
| `STATS` | `handle_stats` | 0 | `(True, str(STATS))` |
| `GET` / `GETLIST` / `INCREMENT` / `DELETE` | 分派表对应函数 | 1（`key`） | `(bool, message)` |
| `PUT` / `PUTLIST` / `APPEND` | 分派表对应函数 | 2（`key`, `value`） | `(bool, message)` |
| 其他 | 无 | 0 | `(False, 'Unknown command type ...')` |

**返回解包**：`nosql.py:178` 按 `command, key, value` 三元组解包；`nosql.py:190` 取 `response[0]` 作为统计成功标志；`nosql.py:191` 用 `'{};\n{}\n'.format(response[0], response[1])` 组装两字段响应。完整字典见 [socket 线协议契约](downstream/socket-wire-protocol.md)。

## 2. Hook payload 到路由结果映射

上游入参为 `.wally/scripts/sop_context.py:405-407` 的 `json.load(sys.stdin)`，转换器为 `route`（`:87-173`）。

| payload 字段 | 转换函数 | 中间结果 | 下游消费点 |
|---|---|---|---|
| `tool_name` | `tool_kind`（`:201-207`） | `shell` / `edit` / `other` | `check_tool`（`:369-391`）的守卫条件 |
| `tool_input.file_path` / `filePath` / `path` | `check_tool`（`:379`） | `paths` 列表 | `plain_document`（`:210-256`）白名单判定 |
| `command` / `patch` / `input` | `check_tool`（`:380-383`） | patch 文本，正则提取 Add/Update/Delete File 路径 | 路径包含校验 `:387` |
| `prompt` | `accept_confirmation`（`:348-366`） | 12 位 token 正则匹配 | approved 状态迁移 |

| 路由输出键 | 来源 | 消费点 |
|---|---|---|
| `branch` | `subprocess.run` 的 `stdout.strip()`（`:90-93`） | `context_text` 头部（`:177`） |
| `task` | overview 父目录相对路径（`:125`、`:172`） | 任务目录定位 |
| `core` | overview 的 `SOP Rules` 字段（`:116-131`） | 规则文件读取列表 |
| `phase` | `PHASE` 正则（`:133-136`） | checkpoint 与角色映射（`:139-143`） |
| `documents` | `:144-171` 组装 | 拼接为 Agent 上下文正文 |
| `overview` | overview 全文 | implement checkpoint 证据（`:267-274`） |

**出向 JSON 键**：Cursor 事件用 `continue` / `user_message` / `permission` / `additional_context`（`:446-453`）；其他客户端用 `hookSpecificOutput.hookEventName` 加 `permissionDecision` 或 `additionalContext`（`:455-460`）。

## 3. Token payload 到账本与报表映射

上游为各客户端事件 payload，转换器为 `normalize_usage`（`.claude/hooks/_token_common.py:293-360`）。

| 上游字段 | 归一化键 | 合并规则 | 报表列 |
|---|---|---|---|
| 客户端原始 input | `input_tokens` | 累计口径取最大/合并（`:362-394`） | Total input |
| uncached input | `uncached_input_tokens` | 同上 | Uncached input |
| cache read | `cache_read_tokens` | 同一轮内各调用求和（`.claude/scripts/token_report.py:251-255`） | Cache read |
| cache write | `cache_write_tokens` | 同上 | Cache write |
| output | `output_tokens` | 同上 | Output |
| reasoning output | `reasoning_output_tokens` | 仅在字段存在时累加（`:256-257`） | of which reasoning |
| credit | `credit` | 浮点累加（`:252-253`） | Credit |
| 客户端标识 | `client` / `source` | `record_client`（`:175-182`）兜底早期 CodeBuddy 行 | Client |
| 请求 ID | `request_id` | claim 账本去重（`:570-602`） | 不直接展示 |

报表轮次合并规则见 `.claude/scripts/token_report.py:195-230`：Cursor 保留 `generation_id` 一轮一行；Codex 按 `(session_id, thread_id, agent_kind, agent_type, root_turn_id)` 分组求和；CodeBuddy 按同会话连续同 prompt 合并。会计恒等式在 `.claude/scripts/token_report.py:354-362` 固定。

## 4. 映射缺口与诚实声明

- 源码中未找到外部 HTTP/RPC/数据库下游；本页不生成任何无源码证据的占位边。
- `DELETE` 成功分支没有返回字典，映射表按 `nosql.py:139-148` 原样记录为「无返回值」，不补写臆测字段。
- `handle_increment` 成功消息使用自增前的值，属报文文案而非最终 `DATA` 值，映射时不可当作存储结果。
- Claude / Cursor / CodeBuddy 采集器的 usage 具体数值来自各自运行时，仓库内只含解析逻辑，不含真实账本数据。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 改 KV 字段切分：`nosql.py:50-64` → 本页第 1 节 → [socket 线协议](downstream/socket-wire-protocol.md) → [API 参考](../contracts/api-reference.md)。
2. 改 hook payload 字段：`.wally/scripts/sop_context.py:369-391` → 本页第 2 节 → [错误码字典](../contracts/error-codes.md) → [SOP 工作流](../business/sop-task-workflow.md)。
3. 改 usage 键：`.claude/hooks/_token_common.py:293-360` → 本页第 3 节 → [Codex 采集器](downstream/codex-collector.md) → [可观测性 Runbook](../operations/runbook-and-observability.md)。
4. 必跑验证：启动服务发送 `PUT; k; 1; INT` 再 `INCREMENT; k;;` 核对响应；向 SOP Hook 输入模拟 `tool_input` 验证路径白名单；用 `python3 -m py_compile` 覆盖三个转换模块。
