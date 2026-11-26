---
kind: workflow
scope: business-flow-registry
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/sop_context.py, .wally/scripts/create_task.py, .claude/hooks/_token_common.py, .claude/scripts/token_report.py]
symbols: [main, route, create, checkpoint_issue, sync_task_store, rollup_turns]
upstream: [INDEX.md, overview/business-architecture.md]
downstream: [business/kv-command-workflow.md, business/sop-task-workflow.md, business/domain-models/index.md]
---

# 业务流程与接口交叉索引

> 全量 WF-* 流程在此登记，逐条链接到规则、状态机与接口契约。本仓共 13 个流程：8 个 KV 命令、3 个 SOP 治理流程、2 个 Token 流程。配套阅读：[业务架构全景](../overview/business-architecture.md) · [API 参考](../contracts/api-reference.md)。

## 1. KV 存储域流程

| WF ID | 业务意图 | 规则页 | 入向接口 | 错误出口 |
|---|---|---|---|---|
| WF-KV-PUT | 设置键值 | [KV 命令工作流](kv-command-workflow.md) | `PUT` | 无 |
| WF-KV-GET | 读取键值 | 同上 | `GET` | 键不存在 |
| WF-KV-PUTLIST | 写入列表 | 同上 | `PUTLIST` | 无 |
| WF-KV-GETLIST | 读取列表 | 同上 | `GETLIST` | 键不存在 / 非 list |
| WF-KV-INCREMENT | 整数自增 | 同上 | `INCREMENT` | 键不存在 / 非 int |
| WF-KV-APPEND | 列表追加 | 同上 | `APPEND` | 键不存在 / 非 list |
| WF-KV-DELETE | 删除键 | 同上 | `DELETE` | 键不存在 |
| WF-KV-STATS | 读取命令计数 | 同上 | `STATS` | 无 |

各流程共享报文解析 `nosql.py:50-64` 与分派表 `nosql.py:155-164`，差异集中在 `nosql.py:76-152` 的处理函数。

## 2. SOP 治理域流程

| WF ID | 业务意图 | 规则页 | 入口 | 门槛 |
|---|---|---|---|---|
| WF-SOP-CREATE | 从不可变模板创建六阶段任务 | [SOP 任务工作流](sop-task-workflow.md) | `.wally/scripts/create_task.py:164` | 名称/分支/状态/规则/配置校验 |
| WF-SOP-ROUTE | 按 Git 分支定位唯一任务与阶段 | 同上 | `.wally/scripts/sop_context.py:394` | 唯一分支匹配、规则路径正则、阶段链接一致 |
| WF-SOP-CHECKPOINT | Implement/Deploy/IT 前置校验 | 同上 | `.wally/scripts/sop_context.py:260` | 01/02 完成且非模板；执行方法已配置 |

## 3. Token 计量域流程

| WF ID | 业务意图 | 规则页 | 入口 | 数据归属 |
|---|---|---|---|---|
| WF-TOKEN-COLLECT | 事件到账本的归一化与去重 | [SOP 任务工作流](sop-task-workflow.md) 第 5 节、`.claude/hooks/_token_common.py:638` | 四套采集器 | 任务 `token.json` |
| WF-TOKEN-REPORT | 账本到 overview 的轮次汇总 | [IDE 接入层](../architecture/ide-integration-hooks.md) | `.claude/scripts/token_report.py:428` | `00-overview.md` 标记区 |

## 4. 接口与错误码交叉引用

| 契约类型 | 文档 | 覆盖范围 |
|---|---|---|
| 入向接口 | [API 参考](../contracts/api-reference.md) | 8 个 socket 命令、2 个 CLI 入口、4 类 Hook 事件 |
| 出向契约 | [集成总览](../integrations/index.md) | TCP 客户端、git CLI、Codex 采集器 |
| 错误字典 | [错误码字典](../contracts/error-codes.md) | 文本错误、`ValueError`、退出码 1/3、Hook 降级 |
| 配置与开关 | [配置手册](../contracts/configuration.md) | 环境变量、config.json、install-state.json |
| 数据结构 | [领域与数据模型](domain-models/index.md) | DATA、STATS、任务产物、账本 |

## 5. 全局约束索引

| 约束 | 权威锚点 | 影响流程 |
|---|---|---|
| 六阶段不可减 | `.wally/scripts/create_task.py:57-58` | WF-SOP-CREATE |
| 规则绑定不可变 revision | `.wally/install-state.json:4-5` | WF-SOP-ROUTE |
| pending 执行方法必须 BLOCKED | `.wally/config.json:8`、`:20` | WF-SOP-CREATE、WF-SOP-CHECKPOINT |
| SKIPPED 不等于 PASS | `.wally/versions/3f84e7245abcc75d/core/gates.md:23-36` | WF-SOP-CHECKPOINT |
| 采集缺失不等于零用量 | `.claude/scripts/token_report.py:396-397` | WF-TOKEN-REPORT |
| 纯内存状态不持久化 | `nosql.py:47` | 全部 KV 流程 |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 新增流程：先写专属工作流页 → 更新本表 → 更新 [业务架构](../overview/business-architecture.md) 的 WF 总表 → 更新 `.wiki-meta.json` 的 `pageOrder`。
2. 改任一流程规则：专属页 → [错误码字典](../contracts/error-codes.md) → [API 参考](../contracts/api-reference.md) → [调用链全景](../overview/callchain-panorama.md)。
3. 改全局约束：[配置手册](../contracts/configuration.md) → [认知边界](../operations/boundary.md) → 本页第 5 节。
4. 必跑验证：`python3 -m py_compile nosql.py .wally/scripts/create_task.py .wally/scripts/sop_context.py`；起服后按第 1 节 8 个命令逐条验证；运行 `python3 .wally/scripts/sop_context.py` 确认路由输出。
