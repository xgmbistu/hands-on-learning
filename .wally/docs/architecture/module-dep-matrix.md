---
kind: component
scope: module-dependency-matrix
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/hooks/_token_common.py, .claude/scripts/token_report.py, .wally/config.json]
symbols: [main, create, route, execution_config, sync_task_store, refresh_codex]
upstream: [INDEX.md, architecture/index.md]
downstream: [integrations/index.md, operations/pitfalls-and-adr.md]
---

# 模块依赖矩阵与环检测

> 依赖边一律取自真实 `import` / `subprocess` / 文件读取，不画跨仓占位边。本仓为单仓，`repo_deps` 为空。配套阅读：[组件总表](index.md) · [上下游集成总览](../integrations/index.md)。

## 1. 模块级依赖边

| 源模块 | 依赖类型 | 目标 | 源码证据 | 传递内容 |
|---|---|---|---|---|
| `nosql.py` | 标准库 | `socket`、`time` | `nosql.py:6-7` | TCP 套接字与时间戳日志 |
| `.wally/scripts/sop_context.py` | 模块导入 | `create_task.method_configured` | `.wally/scripts/sop_context.py:8` | 执行方法判定复用同一函数 |
| `.wally/scripts/sop_context.py` | 子进程 | `git branch --show-current` | `.wally/scripts/sop_context.py:90-92` | 当前分支字符串 |
| `.wally/scripts/sop_context.py` | 文件读取 | `.wally/core/entry.md` | `.wally/scripts/sop_context.py:88` | 路由入口摘要 |
| `.wally/scripts/sop_context.py` | 文件读取 | `.wally/plans/.../00-overview.md` | `.wally/scripts/sop_context.py:97-104` | Branch / Rules / Phase 发现字段 |
| `.wally/scripts/create_task.py` | 文件读取 | `.wally/install-state.json` | `.wally/scripts/create_task.py:83` | revision、steps、translation 状态 |
| `.wally/scripts/create_task.py` | 文件读取 | `.wally/config.json:2` | `.wally/scripts/create_task.py:28-34` | deploy / it 执行方法 |
| `.wally/scripts/create_task.py` | 文件读取 | `.wally/versions/.../plans-template/` | `.wally/scripts/create_task.py:105` | 六阶段模板正文 |
| `.claude/scripts/token_report.py` | 子进程 | `.codex/hooks/codex_token_collect.py --refresh-task` | `.claude/scripts/token_report.py:401-412` | Codex 日志回填 |
| `.claude/scripts/token_report.py` | 子进程 | `git branch --show-current` | `.claude/scripts/token_report.py:68-76` | 当前分支，用于定位任务 |
| `.claude/scripts/token_report.py` | 文件写入 | 任务的 `00-overview.md` | `.claude/scripts/token_report.py:415-425` | Token 汇总表 |
| `.claude/hooks/_token_common.py` | 文件读写 | 任务 `token.json` + claim 账本 | `.claude/hooks/_token_common.py:443-541`、`:550-602` | usage 记录与请求去重 |

## 2. 高扇入底座

| 底座 | 被依赖方 | 复用机制 | 变更影响面 |
|---|---|---|---|
| `.claude/hooks/_token_common.py` | `.claude/`、`.codex/`、`.cursor/`、`.codebuddy/` 四套采集器 | 同名文件复制到各 hooks 目录 | 一处 usage 字段变更需同步 4 处副本与报表解析 |
| `.wally/scripts/create_task.py` 的 `method_configured` | `.wally/scripts/sop_context.py:8` | 直接 import | 改判定会同时改变建任务状态与 checkpoint 结论 |
| `.wally/config.json` | `.wally/scripts/create_task.py:26-53`、`.wally/scripts/sop_context.py:275-284` | 配置快照写入 `execution.json` | 改 schema 或 readiness 影响任务生成与部署门槛 |
| `.wally/install-state.json` | `.wally/scripts/create_task.py:83-94`、`.wally/scripts/sop_context.py:130-132` | revision 绑定不可变规则 | 改 revision 必须同步 rules 目录存在性 |

## 3. 环检测与拓扑分层

```text
L0 标准库 / 外部进程
  socket, time, subprocess(git), subprocess(codex collector)
        │
L1 共享底座
  _token_common.py ── create_task.method_configured
        │                       │
L2 执行入口                      │
  token_report.py ──refresh──> codex_token_collect.py
  sop_context.py ──import──────┘
        │
L3 规则与数据（无出边）
  versions/core/*.md, config.json, install-state.json, token.json
```

- **无环结论**：`sop_context.py` 只单向依赖 `create_task.py`；`create_task.py` 不反向导入 `sop_context.py`。Token 报表只调用采集器进程，采集器不回读报表。L3 资产无代码出边。
- **潜在环风险**：若未来给 `create_task.py` 增加 `import sop_context`，将立即形成 L2 内双向环，须改为共享模块下沉到 L1。
- **跨仓依赖**：`.wiki-meta.json` 的 `repoDeps` 为空；源码中未找到任何跨仓 RPC/HTTP/MQ/SDK 调用，不生成占位边。

## 4. 未映射目录零遗漏对账

| 目录 | 是否有可执行源码 | 归档位置 |
|---|---|---|
| `.agent/`、`.windsurf/`、`.github/` | 否，规则与指令文本 | [认知边界](../operations/boundary.md) |
| `.agents/skills/` | 否，Skill 定义 | [认知边界](../operations/boundary.md) |
| `.claude/scripts/`、`.codex/scripts/`、`.codebuddy/scripts/` | 是 | [ide-integration-hooks.md](ide-integration-hooks.md) |
| `.claude/hooks/` 等四套 hooks | 是 | [ide-integration-hooks.md](ide-integration-hooks.md) |
| `.wally/docs/` | 否，知识资产 | [根路由](../INDEX.md) |

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 改 import 关系：`.wally/scripts/sop_context.py:8` → 本页 → [component 总表](index.md) → [调用链全景](../overview/callchain-panorama.md)。
2. 改配置读取：`.wally/scripts/create_task.py:26-53` → [配置手册](../contracts/configuration.md) → [git 子进程契约](../integrations/downstream/git-subprocess.md)。
3. 改 Token 底座：`.claude/hooks/_token_common.py:293-360` → [ide-integration-hooks.md](ide-integration-hooks.md) → [共变清单](../operations/pitfalls-and-adr.md)。
4. 必跑验证：对四个采集器与报表脚本执行 `python3 -m py_compile`；用 `python3 .wally/scripts/sop_context.py` 确认 import 未破坏路由。
