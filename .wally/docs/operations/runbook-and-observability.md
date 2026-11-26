---
kind: operations
scope: runbook-and-observability
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .claude/hooks/_token_common.py, .wally/config.json]
symbols: [main, update_stats, handle_stats, route, require_confirmation, render_health, file_lock]
upstream: [INDEX.md, contracts/error-codes.md]
downstream: [operations/boundary.md, integrations/downstream/git-subprocess.md]
---

# 排障 Runbook 与可观测性

> 本仓只有三类观测信号：服务 stdout 日志、进程内 `STATS` 计数、SOP/Token 工具的文本与 JSON 输出。没有指标端点、结构化日志、审计留存或告警规则。配套阅读：[错误码字典](../contracts/error-codes.md) · [部署与运行时](deployment-and-runtime.md)。

## 1. 观测信号清单

| 信号 | 产出点 | 内容 | 留存 |
|---|---|---|---|
| 启动日志 | `nosql.py:171` | `Listening on ('localhost', 50505)` | stdout，无文件 |
| 连接日志 | `nosql.py:174-176` | 时间戳 + `New connection from <地址>` | stdout，无文件 |
| 命令计数 | `nosql.py:12-45`、`:67-73` | 每命令 `success` / `error` | 进程内存 |
| 计数导出 | `nosql.py:150-152` | `STATS` 命令返回字典字符串 | 随响应返回 |
| SOP 路由警告 | `.wally/scripts/sop_context.py:411-415` | `SOP context warning: ...` | stdout |
| SOP 确认提示 | `.wally/scripts/sop_context.py:325-345` | `CONFIRM SOP <token>` + 退出码 3 | stdout |
| 采集健康行 | `.claude/scripts/token_report.py:368-398` | `ok` / `partial` / `missing` / `unsupported` / `pending` | overview 标记区 |
| 报表结果 | `.claude/scripts/token_report.py:477-481` | `[OK] Token rollup: ...` | stdout |

时间戳格式由 `nosql.py:175` 固定为 `%Y/%m/%d %H:%M:%S INFO`。

## 2. 健康检查步骤

1. 确认进程：`python3 nosql.py` 的 stdout 出现 `Listening on ('localhost', 50505)`（`nosql.py:171`）。
2. 发起探测：连接 `localhost:50505` 并发送 `STATS;;;`，期望收到 `True;\n{...}\n`（`nosql.py:179-180`、`:191`）。
3. 校验计数：响应中应包含 8 个命令键；缺失说明 `STATS` 初始化被改动（对照 `nosql.py:12-45`）。
4. 工具链检查：`python3 .wally/scripts/sop_context.py` 应给出当前分支路由结果或明确的建任务指引（`.wally/scripts/sop_context.py:105-114`）。
5. 报表检查：在功能分支运行 `python3 .claude/scripts/token_report.py`，观察 `[OK]` 与 Token 标记区是否更新。

没有独立的健康端点或就绪探针；上述为源码可证实的全部探测手段。

## 3. 确定性故障定位决策树

```text
现象 1：客户端连接后无响应
  ├─ 报文是否恰好 4 段？否 → parse_message 返回字符串，main 解包失败（nosql.py:55-56,178）
  ├─ 命令是否为 DELETE 且命中？是 → 删除成功但无返回值，统计阶段 TypeError（nosql.py:147,190）
  └─ 命令是否为未知值？是 → update_stats 的 STATS[command] 触发 KeyError（nosql.py:189-190）

现象 2：STATS 计数异常
  ├─ 某命令 error 增长 → 对照 contracts/error-codes.md 的文本错误字面
  ├─ 计数不增长 → 检查是否在响应前短路（解析失败、DELETE 成功）
  └─ 计数归零 → 服务重启，DATA 与 STATS 均为内存对象（nosql.py:47,12-45）

现象 3：SOP 无任务绑定
  ├─ git branch --show-current 为空 → 先修复 Git 状态（.wally/scripts/sop_context.py:94-95）
  ├─ 0 条 overview 匹配 → 按提示建任务（.wally/scripts/sop_context.py:108-114）
  └─ 多条匹配 → 解决分支归属歧义（.wally/scripts/sop_context.py:105-107）

现象 4：checkpoint 退出码 3
  ├─ 缺口为 01/02 空模板或标记未完成 → 补齐 Clarify/Plan（.wally/scripts/sop_context.py:267-274）
  ├─ 缺口为 deploy/it 未配置 → 本仓当前状态，readiness=pending
  └─ 需继续 → 用户原样回复 CONFIRM SOP <token>（.wally/scripts/sop_context.py:325-366）

现象 5：Token 报表提示不完整
  ├─ Collection note 为 Codex backfill incomplete → 采集器非 0 退出（.claude/scripts/token_report.py:408-409）
  ├─ Usage ledger missing → 账本缺失或坏 JSON，不得当作零用量（.claude/scripts/token_report.py:466-467）
  └─ [SKIP] → 分支为 main/master/release*（.claude/scripts/token_report.py:58-62）
```

## 4. 错误语义与处理边界

| 错误族 | 是否可重试 | 处理原则 | 锚点 |
|---|---|---|---|
| KV 文本错误 | 是（修正报文后重连） | 读 `response[0]` 布尔与 `response[1]` 文案 | `nosql.py:191` |
| KV 运行时异常 | 否 | 连接已中断，需重连；数据可能已变更 | `nosql.py:55-56`、`:147`、`:189-190` |
| SOP 零匹配 | 是 | 按提示创建任务，不在默认分支实现 | `.wally/scripts/sop_context.py:108-114` |
| SOP 多匹配 | 否 | 解决分支归属后重试 | `.wally/scripts/sop_context.py:105-107` |
| 确认待决 | 是 | 补齐证据或用户确认；确认不是通过 | `.wally/scripts/sop_context.py:325-366` |
| 账本锁超时 | 是 | `LockTimeout` 后重试，不阻断编辑器 | `.claude/hooks/_token_common.py:89`、`:396-441` |

## 5. 可观测性缺口

| 缺口 | 证据 | 影响 |
|---|---|---|
| 无持久日志 | `nosql.py:171`、`:174-176` 只 print | 故障后无法回溯 |
| 无请求 ID | 报文字段只有 4 段（`nosql.py:54`） | 无法跨响应追踪 |
| 无指标端点 | 仅 `STATS` 命令（`nosql.py:150-152`） | 无时序与告警 |
| 无 KV 写审计 | `handle_put` 等直接改 `DATA`（`nosql.py:79` 等） | 无法追溯写入者 |
| 无部署健康检查 | `.wally/config.json:6` | 生效版本不可验证 |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改日志/计数：`nosql.py:171-190` → 本页第 1 节 → [引擎实现](../architecture/nosql-engine.md) → [错误码字典](../contracts/error-codes.md)。
2. 改确认提示：`.wally/scripts/sop_context.py:325-366` → 本页第 3 节 → [SOP 工作流](../business/sop-task-workflow.md) → [配置手册](../contracts/configuration.md)。
3. 改健康行：`.claude/scripts/token_report.py:368-398` → 本页第 1/3 节 → [Codex 采集器契约](../integrations/downstream/codex-collector.md)。
4. 必跑验证：按第 2 节五步执行；故意发送 3 段报文观察无响应；`python3 .wally/scripts/sop_context.py --checkpoint deploy` 记录退出码与缺口文本。
