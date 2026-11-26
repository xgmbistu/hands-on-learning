---
kind: component
scope: architecture-registry
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/hooks/_token_common.py, .claude/scripts/token_report.py]
symbols: [main, create, route, file_lock, render]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [architecture/nosql-engine.md, architecture/sop-tooling.md, architecture/module-dep-matrix.md]
---

# 多态组件总表与工具归档

> 按框架机制而非目录名分类：本仓唯一的业务执行体是 TCP 引擎，其余为工作流 CLI 与编辑器事件工具。组件深潜页各自读取专属源码，避免同料复写。配套阅读：[模块依赖矩阵](module-dep-matrix.md) · [系统技术架构](../overview/system-architecture.md)。

## 1. 组件注册表

| 组件标识 | 框架模型 | 核心源码路径 | 入口符号与锚点 | 上游 | 下游 | 深潜页 |
|---|---|---|---|---|---|---|
| `nosql-engine` | Core Engine / 自实现 TCP 服务 | `nosql.py` | `main` (`nosql.py:167-193`) | 本机 socket 客户端 | `DATA`、`STATS` | [nosql-engine.md](nosql-engine.md) |
| `sop-cli` | 工作流 CLI + Hook 网关 | `.wally/scripts/create_task.py`、`.wally/scripts/sop_context.py` | `create` (`.wally/scripts/create_task.py:69-161`)、`route` (`.wally/scripts/sop_context.py:87-173`) | 编辑器 Hook、Agent | `git` 子进程、规则快照 | [sop-tooling.md](sop-tooling.md) |
| `ide-token` | 事件驱动的采集/报表工具 | `.claude/hooks/`、`.codex/hooks/`、`.cursor/hooks/`、`.codebuddy/hooks/`、各 `scripts/token_report.py` | `sync_task_store` (`.claude/hooks/_token_common.py:638`)、`render` (`.claude/scripts/token_report.py:278-365`) | 编辑器事件 | `token.json`、claim 账本 | [ide-integration-hooks.md](ide-integration-hooks.md) |
| `sop-rules` | 纯知识资产（不可变快照） | `.wally/versions/3f84e7245abcc75d/core/` | `.wally/versions/3f84e7245abcc75d/core/workflow.md:26-54` | `sop-cli` 按 revision 绑定 | 阶段产物 | [业务架构](../overview/business-architecture.md) |

## 2. 框架模型判据

| 判据 | nosql-engine | sop-cli | ide-token |
|---|---|---|---|
| 进程生命周期 | 长驻 `while 1`（`nosql.py:172`） | 每次调用短命 | Hook 触发短命，3–5 秒超时 |
| 输入形态 | TCP 字节流 `recv(4096)`（`nosql.py:177`） | 命令行参数 + stdin JSON（`.wally/scripts/sop_context.py:405-407`） | 客户端事件 payload |
| 输出形态 | `sendall` 的文本响应（`nosql.py:191-192`） | 标准输出文本或 JSON（`.wally/scripts/sop_context.py:454-461`） | Hook JSON（`permission` / `hookSpecificOutput`） |
| 状态持久化 | 无，全内存 | 任务目录 + 确认状态文件 | 任务账本 `token.json` |
| 失败处理 | 布尔标志 + 文本错误 | `ValueError` / `parser.exit(1)` / 退出码 3 | 降级为空串或告警，不阻断编辑器 |

## 3. Tier 3 轻量工具归档

| 工具 | 源码锚点 | 职责 | 是否业务关键 |
|---|---|---|---|
| Token 报表 | `.claude/scripts/token_report.py:428-482` | 渲染 `00-overview.md` 的 Token 区 | 否，交付证据工具 |
| 通用账本库 | `.claude/hooks/_token_common.py:293-360` | usage 归一化、锁、claim 去重 | 否，被 4 客户端复用 |
| Claude 采集器 | `.claude/hooks/claude_token_collect.py:42` | 读 `CLAUDE_CONFIG_DIR` 定位配置 | 否 |
| Codex 采集器 | `.codex/hooks/codex_token_collect.py:42` | 读 `WALLY_CODEX_STATE_DIR` 定位状态 | 否 |
| Cursor 采集器 | `.cursor/hooks/cursor_token_collect.py:132` | 逐键读取环境变量 | 否 |
| CodeBuddy 采集器 | `.codebuddy/hooks/stop_token_collect.py:79` | 读 `CODEBUDDY_EXTENSION_DATA` | 否 |
| SOP 校验脚本 | `.wally/scripts/sop_context.py:394-462` | 路由与阶段门槛 | 是，治理链关键 |

## 4. 目录覆盖对账

| 顶层目录 | 归档去向 | 说明 |
|---|---|---|
| 根目录 `nosql.py` | [nosql-engine.md](nosql-engine.md) | 唯一业务源码 |
| `.wally/scripts/` | [sop-tooling.md](sop-tooling.md) | 工作流 CLI |
| `.wally/versions/`、`.wally/core/`、`.wally/profiles/`、`.wally/rules/` | [业务架构](../overview/business-architecture.md)、[边界声明](../operations/boundary.md) | 规则与配置资产 |
| `.wally/docs/` | 本页集 + [认知边界](../operations/boundary.md) 的外部权威页清单 | 知识库本体 |
| `.claude/`、`.codex/`、`.cursor/`、`.codebuddy/` | [ide-integration-hooks.md](ide-integration-hooks.md) | 采集器与报表 |
| `.agent/`、`.agents/`、`.github/`、`.windsurf/` | [边界声明](../operations/boundary.md) | 规则入口与技能定义，非可执行业务代码 |

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 新增组件目录：本页 → [模块依赖矩阵](module-dep-matrix.md) → [根路由](../INDEX.md) → `.wiki-meta.json` 的 `moduleTree`。
2. 改组件入口符号：对应深潜页 → [调用链全景](../overview/callchain-panorama.md) → [术语表](../overview/glossary.md)。
3. 改 Tier 3 工具：对应工具页 → [配置手册](../contracts/configuration.md) → [排障 Runbook](../operations/runbook-and-observability.md)。
4. 必跑验证：`python3 -m py_compile nosql.py .wally/scripts/create_task.py .wally/scripts/sop_context.py`；核对本表每个 `source_paths` 在正文有真实行号引用。
