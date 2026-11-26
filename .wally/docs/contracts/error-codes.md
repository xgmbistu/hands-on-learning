---
kind: error-dict
scope: error-code-dictionary
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .claude/hooks/_token_common.py, .wally/config.json]
symbols: [parse_message, handle_get, handle_getlist, handle_increment, handle_append, handle_delete, create, route, checkpoint_issue, token_work_disabled]
upstream: [INDEX.md, contracts/api-reference.md]
downstream: [integrations/downstream/git-subprocess.md, operations/runbook-and-observability.md]
---

# 错误码、异常字面量与降级字典

> 本仓没有数字错误码体系；错误以文本字面、`ValueError` 消息、退出码和降级告警四形式存在。下表全部按源码字面记录，配套阅读：[API 参考](api-reference.md) · [排障 Runbook](../operations/runbook-and-observability.md)。

## 1. Socket 文本错误字面

| 字面量 | 触发条件 | 返回方 | 锚点 |
|---|---|---|---|
| `Invalid input! Expected: COMMAND; [KEY]; [VALUE]; [VALUE TYPE]` | 报文按 `;` 切分不为 4 段 | `parse_message` | `nosql.py:55-56` |
| `Error: Key [{}] not found` | `GET` 键不存在 | `handle_get` | `nosql.py:87` |
| `ERROR: Key [{}] contains non-list value ([{}])` | `GETLIST` 值非 list | `handle_getlist` | `nosql.py:105-106` |
| `ERROR: Key [{}] contains non-list value ([{}])` | `INCREMENT` 值非 int（文案沿用 non-list） | `handle_increment` | `nosql.py:118-119` |
| `ERROR: Key [{}] contains non-list value ([{}])` | `APPEND` 值非 list | `handle_append` | `nosql.py:132-133` |
| `ERROR: Key [{}] not found and could not be deleted.` | `DELETE` 键不存在 | `handle_delete` | `nosql.py:143-145` |
| `Unknown command type {}` | 命令不在分派表 | `main` | `nosql.py:189` |

前 6 项以 `response[0] = False` 返回；第 7 项后续在 `nosql.py:190` 触发 `KeyError`。

## 2. 运行时异常（无显式码）

| 异常 | 触发链 | 后果 | 锚点 |
|---|---|---|---|
| `ValueError` | `parse_message` 返回单字符串却在 `main` 解包三元组 | 客户端无响应 | `nosql.py:55-56` → `:178` |
| `KeyError` | 未知命令进入 `STATS[command]` | 连接处理中断 | `nosql.py:189-190` |
| `TypeError` | `DELETE` 成功分支无返回值，`response[0]` 索引 None | 数据已删但响应失败 | `nosql.py:147` → `:190` |
| `ValueError` | 任务名称/日期/分支校验失败 | `[ERROR]` 退出 1 | `.wally/scripts/create_task.py:71-75` |
| `ValueError` | 配置 schema、readiness、gaps 类型非法 | `[ERROR]` 退出 1 | `.wally/scripts/create_task.py:39-52` |
| `ValueError` | revision 非法或绑定规则文件缺失 | `[ERROR]` 退出 1 | `.wally/scripts/create_task.py:88-94` |
| `ValueError` | 无 Git 分支 / 多任务匹配 / 规则路径非法 | stdout 警告，退出 1 | `.wally/scripts/sop_context.py:95`、`:107`、`:132` |
| `ValueError` | Core 导出标记缺失或重复 | 路由中断 | `.wally/scripts/sop_context.py:31-35` |
| `ValueError` | 确认状态结构非法 | 确认不可用 | `.wally/scripts/sop_context.py:299-305` |
| `LockTimeout` | 账本文件锁等待超时 | 采集降级，不阻断 | `.claude/hooks/_token_common.py:89`、`:396-441` |

## 3. 退出码语义

| 退出码 | 入口 | 含义 | 锚点 |
|---|---|---|---|
| `0` | `create_task.py` | 任务创建成功 | `.wally/scripts/create_task.py:160-161` |
| `1` | `create_task.py` | `parser.exit(1, "[ERROR] ...")` | `.wally/scripts/create_task.py:172-173` |
| `0` | `sop_context.py` 普通模式 | 路由成功 | `.wally/scripts/sop_context.py:420-421` |
| `1` | `sop_context.py` 普通模式 | 路由警告 | `.wally/scripts/sop_context.py:413-415` |
| `3` | `sop_context.py --checkpoint` | 需用户确认 | `.wally/scripts/sop_context.py:419` |
| `0` | `sop_context.py --hook` | 恒 0，异常兜底 | `.wally/scripts/sop_context.py:469-474` |
| `0` | `token_report.py` | 报表完成或分支跳过 | `.claude/scripts/token_report.py:449`、`:482` |
| `1` | `token_report.py` | 仓库/分支/overview 缺失 | `.claude/scripts/token_report.py:445`、`:453` |

## 4. 降级与未配置字面量

| 字面量 | 来源 | 语义 | 锚点 |
|---|---|---|---|
| `SOP context warning: ` | 路由失败 | 普通模式输出，不写 stderr | `.wally/scripts/sop_context.py:411` |
| `No SOP task for branch ...` | 零匹配 | 附建任务指引，非阻断咨询 | `.wally/scripts/sop_context.py:108-114` |
| `Classify the request first...` | Hook 无任务 | 仅提示分类 | `.wally/scripts/sop_context.py:438-441` |
| `SOP checkpoint failed: ` | 顶层异常 | stderr，hook 模式不输出 | `.wally/scripts/sop_context.py:473-474` |
| `{}` | hook 模式兜底 | 避免工具全局禁止 | `.wally/scripts/sop_context.py:470-471` |
| `Not configured: no deployment verification ...` | deploy.verify | 未配置生效验证 | `.wally/config.json:6` |
| `Not configured: no rollback version ...` | deploy.rollback | 未配置回滚 | `.wally/config.json:7` |
| `Not configured: no integration test acceptance criteria ...` | it.success_criteria | 未配置验收标准 | `.wally/config.json:19` |
| `Codex backfill is incomplete; values include saved usage only.` | 采集器非 0 退出 | 只含已保存用量 | `.claude/scripts/token_report.py:409` |
| `Codex backfill is incomplete or timed out; values include saved usage only.` | 启动失败/超时 | 同上 | `.claude/scripts/token_report.py:411` |
| `Usage ledger missing or unreadable; do not assume zero usage.` | 账本坏 JSON | 不得当作零用量 | `.claude/scripts/token_report.py:467` |
| `No usage records collected; this does not mean complete zero usage.` | 无记录 | 同上 | `.claude/scripts/token_report.py:397` |
| `_No prompt captured_` | prompt 为空 | 报表占位文案 | `.claude/scripts/token_report.py:140` |
| `unknown`（模型列） | model 字段缺失 | 报表模型列兜底 | `.claude/scripts/token_report.py:311` |

## 5. 错误到文档映射

| 错误族 | 定位页 | 修复影响页 |
|---|---|---|
| Socket 文本错误 | [KV 工作流](../business/kv-command-workflow.md) 第 4 节 | [线协议契约](../integrations/downstream/socket-wire-protocol.md)、[引擎实现](../architecture/nosql-engine.md) |
| 任务创建 `ValueError` | [SOP 工作流](../business/sop-task-workflow.md) 第 2 节 | [API 参考](api-reference.md)、[配置手册](configuration.md) |
| 路由 `ValueError` | [git 子进程契约](../integrations/downstream/git-subprocess.md) 第 5 节 | [排障决策树](../operations/runbook-and-observability.md) |
| 退出码 3 | [SOP 工作流](../business/sop-task-workflow.md) 第 3 节 | [配置手册](configuration.md) |
| 采集降级字面 | [IDE 接入层](../architecture/ide-integration-hooks.md) 第 5 节 | [Codex 采集器契约](../integrations/downstream/codex-collector.md) |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改 socket 错误文案：`nosql.py:87-189` → 本页第 1 节 → [线协议契约](../integrations/downstream/socket-wire-protocol.md) → [KV 工作流](../business/kv-command-workflow.md)。
2. 改 `ValueError` 文本：`.wally/scripts/create_task.py:69-115` 与 `.wally/scripts/sop_context.py:87-173` → 本页第 2 节 → [API 参考](api-reference.md) → [排障 Runbook](../operations/runbook-and-observability.md)。
3. 改降级字面：`.claude/scripts/token_report.py:396-467` → 本页第 4 节 → [IDE 接入层](../architecture/ide-integration-hooks.md)。
4. 必跑验证：起服后发送未知命令与不存在的键，核对第 1 节文案；`python3 .wally/scripts/sop_context.py --checkpoint implement` 捕获退出码；用无效名称运行 `create_task.py` 观察 `[ERROR]` 输出。
