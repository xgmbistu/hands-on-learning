---
kind: panorama
scope: callchain-panorama
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/sop_context.py, .wally/scripts/create_task.py, .claude/scripts/token_report.py, .claude/hooks/_token_common.py]
symbols: [main, parse_message, route, checkpoint_issue, require_confirmation, sync_task_store]
upstream: [INDEX.md]
downstream: [business/kv-command-workflow.md, business/sop-task-workflow.md, integrations/topology-and-mapping.md]
---

# 主干调用链全景

> 以符号级条件调用树表达三条主干链：KV 请求处理、SOP 路由与确认、Token 采集与汇总。异常短路出口与跨进程边界单独标注。配套阅读：[业务流程总表](../business/index.md) · [字段映射](../integrations/topology-and-mapping.md)。

## 1. KV 请求主干链

```mermaid
sequenceDiagram
    participant C as 本机客户端
    participant S as nosql.main
    participant P as parse_message
    participant H as COMMAND_HANDERS
    participant D as DATA / STATS
    C->>S: TCP 连接 + 报文
    S->>P: recv(4096).decode()
    P-->>S: (command, key, value) 或错误字符串
    S->>H: 按 command 取处理函数
    H->>D: 读写数据 / 计数
    D-->>S: (bool, message)
    S-->>C: "True/False; message"
```

符号树（分支条件取自 `nosql.py:179-189`）：

```text
nosql.py:167 main()
  ├─ [bind/listen] nosql.py:169-170 -> 阻塞接受一次连接
  ├─ [recv] nosql.py:177 data = connection.recv(4096).decode()
  ├─ [parse] nosql.py:50-64 parse_message(data)
  │    ├─ [if 切分失败] -> 返回 1 个错误字符串（非三元组）
  │    ├─ [if value_type == LIST] nosql.py:58-59 -> value.split(',')
  │    ├─ [elif value_type == INT] nosql.py:60-61 -> int(value)
  │    └─ [else] nosql.py:62-63 -> value = None
  ├─ [if command == STATS] nosql.py:179-180 -> handle_stats()
  ├─ [elif in GET/GETLIST/INCREMENT/DELETE] nosql.py:181-182 -> handler(key)
  ├─ [elif in PUT/PUTLIST/APPEND] nosql.py:183-187 -> handler(key, value)
  ├─ [else] nosql.py:188-189 -> (False, 'Unknown command type ...')
  ├─ [stats] nosql.py:190 update_stats(command, response[0])
  └─ [send] nosql.py:191-193 -> sendall -> close
```

**异常短路**：解析失败返回字符串却在 `nosql.py:178` 按三元组解包，会抛 `ValueError`；未知命令在 `nosql.py:190` 触发 `STATS[command]` 的 `KeyError`；`handle_delete` 成功分支无返回值，使 `nosql.py:190` 的 `response[0]` 抛 `TypeError`。三处均为真实可执行陷阱，细节见 [高危雷区](../operations/pitfalls-and-adr.md)。

## 2. SOP 路由与确认链

```text
.wally/scripts/sop_context.py:394 main()
  ├─ [hook?] .wally/scripts/sop_context.py:405-407 payload = json.load(stdin)
  ├─ [route] .wally/scripts/sop_context.py:409 route(root)
  │   ├─ [read] .wally/scripts/sop_context.py:88 .wally/core/entry.md
  │   ├─ [subprocess] .wally/scripts/sop_context.py:90-95 git branch --show-current
  │   │    └─ 空分支 -> ValueError「Cannot route SOP without an actual Git branch」
  │   ├─ [scan] .wally/scripts/sop_context.py:97-104 遍历 .wally/plans/*/00-overview.md
  │   │    ├─ 0 匹配 -> ValueError + 建任务指引（含 create_task.py 命令）
  │   │    └─ >1 匹配 -> ValueError「Multiple SOP tasks match branch」
  │   ├─ [legacy] .wally/scripts/sop_context.py:116-129 无 SOP Rules 字段 -> compat/v1 快照
  │   └─ [modern] .wally/scripts/sop_context.py:130-173 校验规则路径正则 -> 解析阶段 -> 组装 documents
  ├─ [checkpoint?] .wally/scripts/sop_context.py:416-419
  │   ├─ checkpoint_issue(root, result, phase)
  │   │    ├─ implement: .wally/scripts/sop_context.py:267-274 比对 01/02 与模板
  │   │    └─ deploy/it: .wally/scripts/sop_context.py:275-284 检查 method_configured
  │   └─ require_confirmation -> 退出码 3 + CONFIRM SOP token
  └─ [hook 出口] .wally/scripts/sop_context.py:422-462
       ├─ UserPromptSubmit -> accept_confirmation(root, result, prompt)
       ├─ PreToolUse -> check_tool(root, payload, result)
       └─ cursor 用 permission JSON，其他客户端用 hookSpecificOutput
```

**确认闭环**：`.wally/scripts/sop_context.py:325-345` 用 scope + fingerprint 生成 12 位 token 写入 pending；`.wally/scripts/sop_context.py:348-366` 只在 token、scope、fingerprint 三者同时匹配时移入 approved。确认只放行缺口，不会把未验证阶段标记为通过。

## 3. Token 采集与汇总链

```text
[采集] .claude/hooks/_token_common.py:638 sync_task_store()
  ├─ .claude/hooks/_token_common.py:106-141 install_deadline() -> 超时自停
  ├─ .claude/hooks/_token_common.py:153-194 read_stdin_json(timeout) -> 坏 payload 不崩
  ├─ .claude/hooks/_token_common.py:210-234 find_repo + token_work_disabled -> release/main 跳过
  ├─ .claude/hooks/_token_common.py:396-441 file_lock(tag) -> 串行写账本
  ├─ .claude/hooks/_token_common.py:550-568 claim_ledger_path + record_claims -> 请求 ID 去重
  └─ .claude/hooks/_token_common.py:443-541 write_json / load_store / merge_store -> 落盘

[汇总] .claude/scripts/token_report.py:428 main()
  ├─ .claude/scripts/token_report.py:435-445 find_repo + git_branch + token_work_disabled
  ├─ .claude/scripts/token_report.py:450-453 find_overview(repo, branch)
  ├─ .claude/scripts/token_report.py:401-412 refresh_codex() -> 子进程 --refresh-task，10 秒超时
  ├─ .claude/scripts/token_report.py:109-121 load_requests(token.json)
  ├─ .claude/scripts/token_report.py:195-230 rollup_turns() -> 一轮一行的合并规则
  ├─ .claude/scripts/token_report.py:278-365 render() + render_health()
  └─ .claude/scripts/token_report.py:415-425 patch_overview() -> 替换 TOKEN_USAGE 标记区
```

## 4. 线程、队列与进程边界

| 边界类型 | 真实实现 | 并发/隔离语义 |
|---|---|---|
| 阻塞式单连接循环 | `nosql.py:172-193` `while 1` + `accept()` | 无线程、无队列；一个连接处理完才接受下一个 |
| 子进程调用 | `.wally/scripts/sop_context.py:90-92`、`.claude/scripts/token_report.py:68-76`、`.claude/scripts/token_report.py:406-407` | 分别为 3 秒 / 3 秒 / 10 秒超时，失败降级为空串或告警 |
| 文件锁 | `.claude/hooks/_token_common.py:396-441` | 跨 Hook 进程串行化账本写入 |
| 原子目录发布 | `.wally/scripts/create_task.py:150-159` | 临时目录 + rename，避免半写任务被路由看到 |
| 跨仓桥接 | 仓库内未找到跨仓调用 | 单仓工作区，`repo_deps` 为空；不做笛卡尔积占位边 |

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 改 KV 分派条件：`nosql.py:167-193` → `business/kv-command-workflow.md` → `contracts/api-reference.md` → `overview/callchain-panorama.md`。
2. 改路由错误文本：`.wally/scripts/sop_context.py:87-173` → `contracts/error-codes.md` → `business/sop-task-workflow.md`。
3. 改确认流程：`.wally/scripts/sop_context.py:293-366` → `operations/runbook-and-observability.md` → `contracts/configuration.md`。
4. 必跑验证：`python3 -m py_compile nosql.py .wally/scripts/sop_context.py .claude/scripts/token_report.py`；`python3 .wally/scripts/sop_context.py --checkpoint implement` 检查退出码与提示；新建连接发送 `STATS` 观察计数变化。
