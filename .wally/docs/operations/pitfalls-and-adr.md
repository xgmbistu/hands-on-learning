---
kind: operations
scope: pitfalls-and-adr
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/hooks/_token_common.py, .wally/config.json, .wally/versions/3f84e7245abcc75d/core/knowledge.md]
symbols: [parse_message, handle_delete, update_stats, create, inside, scope_summary, file_lock]
upstream: [INDEX.md, architecture/module-dep-matrix.md]
downstream: [operations/boundary.md, contracts/error-codes.md]
---

# 架构决策、高危雷区与存量未挂载代码

> 本页记录源码可证实的决策与陷阱，不推测作者意图之外的设计理由。配套阅读：[模块依赖矩阵](../architecture/module-dep-matrix.md) · [认知边界](boundary.md)。

## 1. 架构决策记录（ADR）

| ID | 决策 | 证据锚点 | 后果 |
|---|---|---|---|
| ADR-1 | 键值服务采用单文件自实现 TCP 循环，不引入 Web 框架 | `nosql.py:1-197`；仓库无包清单 | 零依赖、易读；但单连接串行、无并发 |
| ADR-2 | SOP 规则按不可变 revision 绑定，禁止用最新版兜底 | `.wally/install-state.json:4-5`；`.wally/scripts/sop_context.py:130-132`；`.wally/scripts/create_task.py:91-94` | 历史任务可复现；升级需显式迁移 |
| ADR-3 | 任务目录用临时目录写全后原子 rename 发布 | `.wally/scripts/create_task.py:149-159` | 路由不会看到半写任务 |
| ADR-4 | Hook 失败降级为空 JSON，不阻断编辑器 | `.wally/scripts/sop_context.py:469-474` | 工具可用性优先；代价是错误被静默 |
| ADR-5 | 客户执行方法只读不推断，pending 时 BLOCKED | `.wally/config.json:8-25`；`.wally/scripts/create_task.py:26-53`、`:120` | 不编造部署/测试命令；当前无从验证环境 |
| ADR-6 | Token 请求 ID 用 claim 账本去重 | `.claude/hooks/_token_common.py:550-602` | 避免重复计数；依赖账本路径正确 |
| ADR-7 | 新任务强制保留六阶段 | `.wally/scripts/create_task.py:57-58`；`.wally/versions/3f84e7245abcc75d/core/workflow.md:48` | 简小改动也不减阶段；SKIPPED 仍留零时长行 |
| ADR-8 | CodeWiki 与 SOP 权威文档分离 | `.wally/versions/3f84e7245abcc75d/core/knowledge.md:17-18` | `apis/`、`sqls/`、`unittest/`、`integration_test/`、`devops/` 只链接不复制 |

## 2. 高危雷区（Danger Zones）

| 编号 | 雷区 | 触发条件 | 后果 | 锚点 |
|---|---|---|---|---|
| D1 | 裸 `except` | 报文段数不为 4 | 吞掉具体解析异常，返回字符串 | `nosql.py:55-56` |
| D2 | `COMMAND_HANDERS` 拼写 | 引用时写成 `HANDLERS` | `NameError`，全部分派失效 | `nosql.py:155` |
| D3 | 模块级 `SOCKET` 单例 | 同一进程重复导入或起服 | 只能绑定一次，无法并行实例 | `nosql.py:11` |
| D4 | `DELETE` 成功无返回值 | 删除命中键 | 响应阶段 `TypeError`，但数据已删 | `nosql.py:147` → `:190` |
| D5 | 未知命令统计越界 | 命令不在 `STATS` 8 键中 | `KeyError` | `nosql.py:189-190` |
| D6 | 错误文案错标类型 | `INCREMENT` 非 int | 返回 `non-list value`，误导排障 | `nosql.py:117-119` |
| D7 | 单次 `recv(4096)` 即一命令 | 报文超过 4096 字节 | 截断且无重组 | `nosql.py:177-178` |
| D8 | 无持久化 | 进程退出 | `DATA` 与 `STATS` 全部丢失 | `nosql.py:47`、`:12-45` |
| D9 | 分支匹配依赖文档表格字面 | overview 的 Branch 行格式被改动 | 路由零匹配 | `.wally/scripts/sop_context.py:19`、`:102-104` |
| D10 | `_token_common.py` 四份副本 | 只改其中一份 | 四客户端账本口径漂移 | `.claude/hooks/_token_common.py:1` 及另外三套 hooks 目录 |
| D11 | 路径包含校验依赖 `resolve()` | 符号链接指向仓库外 | 抛 `ValueError`，不越界写 | `.wally/scripts/create_task.py:11-17`、`.wally/scripts/sop_context.py:62-77` |
| D12 | `Path.is_relative_to` 兼容实现 | 误改为 3.9+ 原生方法 | 丢失低版本兼容 | `.wally/scripts/create_task.py:12`、`.wally/scripts/sop_context.py:63` |

## 3. 存量未挂载历史函数

| 函数 | 锚点 | 状态 | 处理建议 |
|---|---|---|---|
| `scope_summary` | `.wally/scripts/sop_context.py:44-45` | **未挂载**：全仓仅定义处出现，无任何调用点 | 保留原样；不得在生成文档中描述为运行时使用；如需启用须显式接入 `context_text` 并经 SOP 授权 |
| `exported_section` | `.wally/scripts/sop_context.py:28-36` | 已挂载：被 `workflow_summary`（`:41`）与 `scope_summary`（`:45`）调用 | 因 `scope_summary` 未挂载，其 `sop-scope` 导出路径实际不生效 |
| `workflow_summary` | `.wally/scripts/sop_context.py:39-41` | 已挂载：`context_text` 在 `:182-183` 调用 | 正常 |

核验方式：对 `scope_summary` 做全仓精确符号检索，命中仅定义行；对照 `workflow_summary` 在 `context_text` 中的真实调用行。

## 4. 版本与升级雷区

- `.wally/install-state.json:4-5` 的 `revision` 与 `.wally/versions/3f84e7245abcc75d/` 目录名必须一致；`.wally/scripts/create_task.py:91-94` 校验四个 core 文件存在，但目录整体缺失时由 `.wally/scripts/create_task.py:93` 逐个文件报错。
- `.wally/install-state.json` 中 `translation_sources` 非空会阻断建任务（`.wally/scripts/create_task.py:84-85`），这是升级中间态保护。
- `.wally/config.json` 的 `schema` 变更必须同步 `.wally/scripts/create_task.py:39-52` 的类型校验，否则已有配置被判非法。
- 旧任务通过 `.wally/compat/v1/` 恢复（`.wally/scripts/sop_context.py:116-129`）；快照缺失即阻断，不隐式套用 2.0 规则。

## 5. 决策边界与不做事项

| 不做事项 | 依据 |
|---|---|
| 不推测外部客户端实现 | 客户端源码不在本仓；[线协议契约](../integrations/downstream/socket-wire-protocol.md) 标注为外部黑盒 |
| 不生成跨仓占位边 | 源码无跨仓调用；`repoDeps` 为空 |
| 不把 SKIPPED 写成 PASS | `.wally/versions/3f84e7245abcc75d/core/gates.md:23-36` |
| 不把采集缺失写成零用量 | `.claude/scripts/token_report.py:396-397` |
| 不覆盖人工页 | 人工确认或已核验页面受保护；本仓当前无此类页 |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改引擎陷阱相关代码：`nosql.py:55-56`、`:147`、`:155`、`:177-190` → 本页第 2 节 → [引擎实现](../architecture/nosql-engine.md) → [错误码字典](../contracts/error-codes.md)。
2. 改 rule 绑定：`.wally/install-state.json:4-5` → 本页第 1/4 节 → [SOP 工作流](../business/sop-task-workflow.md) → [认知边界](boundary.md)。
3. 改账本副本：`.claude/hooks/_token_common.py` → 同步另外三套 hooks 目录 → 本页 D10 → [IDE 接入层](../architecture/ide-integration-hooks.md)。
4. 必跑验证：起服发送 `DELETE; absent;;` 观察文本错误；发送 `FOO;;;` 观察连接中断；对全部 Python 脚本执行 `python3 -m py_compile`；用符号检索确认 `scope_summary` 仍无调用点。
