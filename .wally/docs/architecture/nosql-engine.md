---
kind: component
scope: nosql-engine
status: draft
origin: generated
source_paths: [nosql.py]
symbols: [main, parse_message, update_stats, handle_put, handle_get, handle_getlist, handle_increment, handle_append, handle_delete, handle_stats, COMMAND_HANDERS]
upstream: [architecture/index.md, overview/system-architecture.md]
downstream: [integrations/downstream/socket-wire-protocol.md, business/kv-command-workflow.md, contracts/api-reference.md]
---

# 核心引擎：单文件 TCP 键值服务

> 框架模型为 `Core Engine`，但传输层是自实现的阻塞式 socket 循环，没有第三方框架、没有线程、没有持久化。本页只深潜 `nosql.py` 全文；协议字典见 [socket 线协议下游契约](../integrations/downstream/socket-wire-protocol.md)，流程分支见 [KV 命令工作流](../business/kv-command-workflow.md)。

## 1. 组件定位与全局状态

| 项 | 值 | 锚点 |
|---|---|---|
| 入口 | `python3 nosql.py` → `main()` | `nosql.py:196-197` |
| 监听 | `localhost:50505`、`AF_INET`、`SOCK_STREAM` | `nosql.py:9-11` |
| 数据存储 | `DATA = {}` 进程内字典 | `nosql.py:47` |
| 统计存储 | `STATS` 8 键 × `success`/`error` | `nosql.py:12-45` |
| 分派表 | `COMMAND_HANDERS` 8 项 | `nosql.py:155-164` |
| 接收缓冲 | 4096 字节，一次 `recv` 对应一次命令 | `nosql.py:177` |
| 生命周期 | `while 1` 串行接受，处理完关闭连接 | `nosql.py:172-193` |

## 2. 报文解析与类型转换字典

`parse_message` 位于 `nosql.py:50-64`，按分号切分 4 段并逐段 `strip`：

| 段位 | 变量 | 类型转换规则 | 源码锚点 |
|---|---|---|---|
| 1 | `command` | 保持字符串，不做大写归一 | `nosql.py:54` |
| 2 | `key` | 保持字符串 | `nosql.py:54` |
| 3 | `value` | `value_type == 'LIST'` → `value.split(',')`；`== 'INT'` → `int(value)` | `nosql.py:57-61` |
| 4 | `value_type` | 非空才触发转换，为空则 `value = None` | `nosql.py:57-63` |

切分片数不等于 4 时，`except` 分支返回一个提示字符串（`nosql.py:55-56`），而调用方按三元组解包（`nosql.py:178`），形成真实的 `ValueError` 短路。`LIST` / `INT` 是仅有的两个字面类型转换分支，其他类型字符串被静默忽略。

## 3. 命令处理函数对照

| 命令 | 处理函数 | 入参 | 成功返回 | 失败返回 |
|---|---|---|---|---|
| `PUT` | `handle_put` `nosql.py:76-80` | `key, value` | `(True, 'key [...] set to [...]')` | 无 |
| `GET` | `handle_get` `nosql.py:83-89` | `key` | `(True, DATA[key])` | `(False, 'Error: Key [...] not found')` |
| `PUTLIST` | `handle_putlist` `nosql.py:92-95` | `key, value` | 转调 `handle_put` | 无 |
| `GETLIST` | `handle_getlist` `nosql.py:98-108` | `key` | `(True, list)` | 键缺失转 `handle_get`；非 list 回 `(False, 'ERROR: Key [...] contains non-list value')` |
| `INCREMENT` | `handle_increment` `nosql.py:111-122` | `key` | `(True, 'Key [...] incremented')` | 非 int 回 `(False, 'ERROR: Key [...] contains non-list value')` |
| `APPEND` | `handle_append` `nosql.py:125-136` | `key, value` | `(True, 'Key [...] had value [...] appended')` | 非 list 回 `(False, 'ERROR: Key [...] contains non-list value')` |
| `DELETE` | `handle_delete` `nosql.py:139-148` | `key` | 无返回值（缺陷） | `(False, 'ERROR: Key [...] not found and could not be deleted.')` |
| `STATS` | `handle_stats` `nosql.py:150-152` | 无 | `(True, str(STATS))` | 无 |

`INCREMENT` 的成功消息使用自增前的值（`nosql.py:122` 的 `format(key, value)` 在赋值后仍引用旧局部量），这是报文文案与最终存储值不一致的真实细节。

## 4. 状态机与统计计数

```text
[连接建立] nosql.py:173 accept()
  └─ [报文] parse_message
       ├─ [STATS] -> handle_stats -> 不读写 DATA
       ├─ [读类 GET/GETLIST/INCREMENT/DELETE] -> handler(key)
       ├─ [写类 PUT/PUTLIST/APPEND] -> handler(key, value)
       └─ [未知命令] -> (False, 'Unknown command type ...')
  → update_stats(command, response[0]) nosql.py:190
  → 'True/False;\n<message>\n' nosql.py:191
  → sendall + close nosql.py:192-193
```

`update_stats` 位于 `nosql.py:67-73`：成功对 `STATS[command]['success'] += 1`，失败对 `['error'] += 1`。`STATS` 预置 8 个键（`nosql.py:12-45`），任何不在分派表中的命令都会在 `nosql.py:190` 触发 `KeyError`。计数器无上限、无重置接口，`STATS` 命令自身也会被计数。

## 5. 已知缺陷与不可假设行为

| 缺陷 | 触发条件 | 后果 | 锚点 |
|---|---|---|---|
| 解析错误解包失败 | 报文段数不为 4 | `ValueError`，连接无响应 | `nosql.py:55-56` + `nosql.py:178` |
| 未知命令统计越界 | command 不在 `STATS` 预置键中 | `KeyError` | `nosql.py:189-190` |
| 删除成功无返回 | `DELETE` 命中键 | `response[0]` 抛 `TypeError` | `nosql.py:147` + `nosql.py:190` |
| 套接字单例 | 模块导入即创建 `SOCKET` | 进程内只能绑定一次，不可重复起服 | `nosql.py:11` |
| 无鉴权与无 TLS | 全部请求 | 仅适合本机学习环境 | `nosql.py:9-11` |

上述缺陷按现状如实记录，不在本 Wiki 中推断修复方案；治理建议见 [高危雷区与历史陷阱](../operations/pitfalls-and-adr.md)。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改命令集合或分派：`nosql.py:155-164` → [API 参考](../contracts/api-reference.md) → [错误码字典](../contracts/error-codes.md) → [KV 工作流](../business/kv-command-workflow.md) → [线协议契约](../integrations/downstream/socket-wire-protocol.md)。
2. 改报文字段：`nosql.py:50-64` → [字段映射](../integrations/topology-and-mapping.md) → [API 参考](../contracts/api-reference.md) → [数据模型](../business/domain-models/index.md)。
3. 改统计键：`nosql.py:12-45` → [数据模型](../business/domain-models/index.md) → [可观测性 Runbook](../operations/runbook-and-observability.md)。
4. 必跑验证：`python3 -m py_compile nosql.py`；启动服务后依次发送 `PUT`/`GET`/`GETLIST`/`PUTLIST`/`INCREMENT`/`APPEND`/`DELETE`/`STATS` 八条报文，逐条比对第 3 节返回文案。
