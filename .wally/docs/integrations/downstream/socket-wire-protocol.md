---
kind: downstream-contract
scope: tcp-socket-client
status: draft
origin: generated
source_paths: [nosql.py]
symbols: [parse_message, COMMAND_HANDERS, update_stats, main, handle_get, handle_getlist, handle_increment, handle_append, handle_delete, handle_stats]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: [contracts/api-reference.md, contracts/error-codes.md, architecture/nosql-engine.md]
---

# 下游契约：TCP 客户端线协议

> 下游为发起连接的本机客户端程序。请求字典由客户端拼装、服务端 `parse_message` 解包；响应字典由 `nosql.py:191` 拼装并经 `sendall` 回写。协议无版本字段、无长度前缀、无编码协商，证据全部来自 `nosql.py`。配套阅读：[引擎实现](../../architecture/nosql-engine.md) · [字段映射](../topology-and-mapping.md)。

## 1. 请求字典键（客户端 → 服务端）

| 键位 | 字段名 | 类型 | 来源与转换 | 锚点 |
|---|---|---|---|---|
| 1 | `command` | 字符串 | `parse_message` 第 1 段 `strip`，不做大写归一 | `nosql.py:54` |
| 2 | `key` | 字符串 | 第 2 段 `strip` | `nosql.py:54` |
| 3 | `value` | 字符串 / list / int / None | `value_type == 'LIST'` 走 `split(',')`；`== 'INT'` 走 `int()`；否则 `None` | `nosql.py:57-63` |
| 4 | `value_type` | 字符串 | 第 4 段 `strip`，仅作转换开关，不传递 | `nosql.py:54`、`:57` |

报文格式字面为 `COMMAND; [KEY]; [VALUE]; [VALUE TYPE]`（`nosql.py:56` 的错误提示）。接收端 `recv(4096)`（`nosql.py:177`）一次读取即视为一条完整命令；超长报文会被截断且无重组逻辑。

## 2. 响应字典键（服务端 → 客户端）

| 键位 | 字段名 | 类型 | 组装规则 | 锚点 |
|---|---|---|---|---|
| 1 | `response[0]` | 布尔 | 处理函数返回的 success 标志 | `nosql.py:191` |
| 2 | `response[1]` | 字符串 | 处理函数返回的消息或数据 | `nosql.py:191` |

帧格式：`'{};\n{}\n'.format(response[0], response[1])`（`nosql.py:191`），即第一行布尔值加分号，第二行消息，末尾换行；随后 `bytearray(data, 'utf-8')`（`nosql.py:192`）编码发送，`connection.close()`（`:193`）结束连接。

## 3. 八命令请求/响应对照

| 命令 | 请求键使用 | 成功响应体 | 失败响应体 |
|---|---|---|---|
| `PUT` | `command`、`key`、`value` | `key [k] set to [v]` | 无失败分支 |
| `GET` | `command`、`key` | `DATA[key]` 原值 | `Error: Key [k] not found` |
| `PUTLIST` | `command`、`key`、`value` | 转调 PUT 的文案 | 无失败分支 |
| `GETLIST` | `command`、`key` | 原 list | `ERROR: Key [k] contains non-list value ([v])` |
| `INCREMENT` | `command`、`key` | `Key [k] incremented`（引用自增前值） | `ERROR: Key [k] contains non-list value ([v])` |
| `APPEND` | `command`、`key`、`value` | `Key [k] had value [v] appended` | `ERROR: Key [k] contains non-list value ([v])` |
| `DELETE` | `command`、`key` | 无返回值，触发 `TypeError` | `ERROR: Key [k] not found and could not be deleted.` |
| `STATS` | `command` | `str(STATS)` 字典字符串 | 无失败分支 |

`INCREMENT` 与 `APPEND` 的失败文案字面都写 `non-list value`，前者实际校验的是 `int`（`nosql.py:117-119`），这是必须原样保留的报文历史。

## 4. 返回解包与统计联动

```text
[nosql.py:178] command, key, value = parse_message(data)
[nosql.py:179-189] 分派 -> response = (bool, message)
[nosql.py:190] update_stats(command, response[0])
   ├─ True  -> STATS[command]['success'] += 1   (nosql.py:70-71)
   └─ False -> STATS[command]['error']   += 1   (nosql.py:72-73)
[nosql.py:191-192] 组帧 -> sendall
```

`STATS` 预置 8 个键（`nosql.py:12-45`），所以第 8 行 `STATS` 命令自身也会被计数；未知命令在 `nosql.py:190` 越界。解析错误时 `parse_message` 返回单个字符串（`nosql.py:55-56`），在 `nosql.py:178` 解包失败，客户端收不到任何响应。

## 5. 边界与不可假设项

- 无鉴权、无 TLS、无请求 ID、无重试/幂等机制：`nosql.py:9-11` 直接绑定 `localhost:50505`。
- 无持久化：`DATA` 与 `STATS` 均为模块级内存对象（`nosql.py:47`、`:12-45`），进程重启即归零。
- 无并发：`.wally/docs` 内任何页都不得把 `nosql.py:172-193` 描述为多线程或事件循环。
- 客户端实现不在本仓：仓库内只有服务端，客户端拼装代码属 `[外部黑盒：本仓未包含客户端实现]`，不推测其字段来源。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改报文字段：`nosql.py:50-64` → [字段映射](../topology-and-mapping.md) → [API 参考](../../contracts/api-reference.md) → [引擎实现](../../architecture/nosql-engine.md)。
2. 改响应文案：`nosql.py:76-152` → [错误码字典](../../contracts/error-codes.md) → [KV 工作流](../../business/kv-command-workflow.md) → 本页第 3 节。
3. 改统计键：`nosql.py:12-45` → [数据模型](../../business/domain-models/index.md) → [可观测性 Runbook](../../operations/runbook-and-observability.md)。
4. 必跑验证：起服后发送 `PUT; alpha; 1; INT`、`INCREMENT; alpha;;`、`GETLIST; alpha;;`，核对三条响应布尔与文案；发送 `STATS;;;` 观察 8 键计数变化。
