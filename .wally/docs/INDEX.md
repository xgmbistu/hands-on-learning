# hands-on-learning · Agent 知识库总路由表（INDEX）

> **Agent 读取规程（必读）**：
> 1. **禁止一次性全量读取 `.wally/docs/`**。先查下方「任务意图路由表」或「源码路径反向索引」，只打开命中的 1–3 篇目标文档（可用 `Read(limit=15)` 先看顶部 YAML 头确认 `source_paths`、`symbols` 与 `status`）。
> 2. **草稿核验纪律**：全站页面均为 `status: draft` / `origin: generated`，是从源码提取的草稿。拿到 `路径:行号`（`符号名`）、字典结构与共变清单后，修改源码前必须调用 `Read(path, offset, limit)` 复核字面；近邻词线索不等于原词覆盖。
> 3. **人工精修保护**：若后续出现人工确认或已核验页面，重跑知识库严禁覆盖。本次基准提交：`chore/wally-init-20261126-c107` @ `14fea139ecbc1b7509d19a6d4fbcb83502c3be6a`。
> 4. **外部权威页**：`apis/`、`sqls/`、`unittest/`、`integration_test/`、`devops/` 归 SOP 规范所有，本 Wiki 只链接不改写，见文末清单。

本仓是 `hands-on-learning` 单仓：唯一业务执行体是 `nosql.py` 提供的本机 TCP 键值服务（8 个命令、内存 `DATA`/`STATS`），另有 SOP 六阶段工作流 CLI 与四客户端 Token 采集工具链。两条执行链互不调用，共享的只是「仓库根路径发现」机制；部署与 IT 均为 `readiness: pending`，不声称环境验证通过。

---

## 1. 任务意图最短路由表（Agent Intent Router）

| Agent 当前待办任务类型 | 按序必读文档（1 → 2 → 3） |
|---|---|
| **修改 KV 命令 / 报文解析 / 响应文案** | [KV 命令工作流](business/kv-command-workflow.md) → [nosql-engine](architecture/nosql-engine.md) → [socket 线协议契约](integrations/downstream/socket-wire-protocol.md) |
| **新增或排查入向接口、Hook 事件、CLI 参数** | [API 参考](contracts/api-reference.md) → [sop-tooling](architecture/sop-tooling.md) → [错误码字典](contracts/error-codes.md) |
| **对接或修改出向子进程契约** | [集成总览](integrations/index.md) → [字段映射](integrations/topology-and-mapping.md) → 对应下游契约（[git](integrations/downstream/git-subprocess.md) / [Codex 采集器](integrations/downstream/codex-collector.md) / [socket](integrations/downstream/socket-wire-protocol.md)） |
| **修改 SOP 规则、阶段或建任务流程** | [SOP 任务工作流](business/sop-task-workflow.md) → [业务架构全景](overview/business-architecture.md) → [配置手册](contracts/configuration.md) |
| **定位报错、异常、退出码或降级原因** | [错误码字典](contracts/error-codes.md) → [排障 Runbook](operations/runbook-and-observability.md) → [主干调用链全景](overview/callchain-panorama.md) |
| **评估跨模块联动与必跑验证** | [模块依赖矩阵](architecture/module-dep-matrix.md) → [高危雷区与历史陷阱](operations/pitfalls-and-adr.md) → [构建与单测映射](operations/dev-and-test.md) |
| **了解系统分层与启动装配** | [系统技术架构](overview/system-architecture.md) → [组件总表](architecture/index.md) → [主干调用链全景](overview/callchain-panorama.md) |
| **查领域术语、字段字典、数据归属** | [术语表](overview/glossary.md) → [领域与数据模型](business/domain-models/index.md) → [配置手册](contracts/configuration.md) |
| **确认部署、运行期目录与边界** | [部署与运行时](operations/deployment-and-runtime.md) → [认知边界](operations/boundary.md) → [构建与单测映射](operations/dev-and-test.md) |

---

## 2. 源码路径 → 文档反向索引（Source-to-Doc Reverse Index）

| 仓库源码路径 | 架构模型 / 角色 | 首选关联文档 | 关联上下游/业务文档 |
|---|---|---|---|
| `nosql.py` | Core Engine / 自实现 TCP 服务 | [nosql-engine.md](architecture/nosql-engine.md) | [KV 工作流](business/kv-command-workflow.md) · [线协议契约](integrations/downstream/socket-wire-protocol.md) |
| `.wally/scripts/create_task.py` | 工作流 CLI | [sop-tooling.md](architecture/sop-tooling.md) | [SOP 工作流](business/sop-task-workflow.md) · [数据模型](business/domain-models/index.md) |
| `.wally/scripts/sop_context.py` | Hook 网关 / 路由 | [sop-tooling.md](architecture/sop-tooling.md) | [git 子进程契约](integrations/downstream/git-subprocess.md) · [错误码字典](contracts/error-codes.md) |
| `.wally/config.json` | 执行方法配置 | [配置手册](contracts/configuration.md) | [SOP 工作流](business/sop-task-workflow.md) · [部署与运行时](operations/deployment-and-runtime.md) |
| `.wally/install-state.json` | 不可变 revision 快照 | [认知边界](operations/boundary.md) | [高危雷区](operations/pitfalls-and-adr.md) · [配置手册](contracts/configuration.md) |
| `.wally/versions/3f84e7245abcc75d/core/` | SOP 规则资产 | [业务架构全景](overview/business-architecture.md) | [SOP 工作流](business/sop-task-workflow.md) · [排障 Runbook](operations/runbook-and-observability.md) |
| `.wally/docs/apis/`、`.wally/docs/sqls/`、`.wally/docs/unittest/`、`.wally/docs/integration_test/`、`.wally/docs/devops/` | SOP 外部权威页 | 见文末外部引用清单 | 不复制正文 |
| `.claude/hooks/_token_common.py`（及三套副本） | 账本底座 | [ide-integration-hooks.md](architecture/ide-integration-hooks.md) | [字段映射](integrations/topology-and-mapping.md) · [数据模型](business/domain-models/index.md) |
| `.claude/scripts/token_report.py`（及两套副本） | Token 报表 | [ide-integration-hooks.md](architecture/ide-integration-hooks.md) | [Codex 采集器契约](integrations/downstream/codex-collector.md) · [可观测性 Runbook](operations/runbook-and-observability.md) |
| `.claude/settings.json`、`.codex/hooks.json`、`.cursor/hooks.json`、`.codebuddy/settings.json` | Hook 事件注册 | [API 参考](contracts/api-reference.md) | [集成总览](integrations/index.md) |
| `.agent/`、`.agents/`、`.github/`、`.windsurf/` | 规则入口与技能定义 | [认知边界](operations/boundary.md) | [组件总表](architecture/index.md) |

---

## 3. 模块技术能力清单（Component Registry）

| 组件标识 | 框架模型 | 核心源码路径 | 核心入口符号 | 上游调用方 | 下游依赖 | 文档 |
|---|---|---|---|---|---|---|
| `nosql-engine` | Core Engine / TCP 服务 | `nosql.py` | `main` (`nosql.py:167-193`) | 本机 socket 客户端 | `DATA`、`STATS` | [nosql-engine.md](architecture/nosql-engine.md) |
| `sop-cli` | 工作流 CLI + Hook 网关 | `.wally/scripts/` | `create` (`.wally/scripts/create_task.py:69-161`)、`route` (`.wally/scripts/sop_context.py:87-173`) | IDE Hook、Agent | `git` 子进程、规则快照 | [sop-tooling.md](architecture/sop-tooling.md) |
| `ide-token` | 事件驱动采集/报表 | 四套 `hooks/` 与 `scripts/` | `sync_task_store`、`render` | 编辑器事件 | `token.json`、claim 账本 | [ide-integration-hooks.md](architecture/ide-integration-hooks.md) |
| `sop-rules` | 不可变规则资产 | `.wally/versions/3f84e7245abcc75d/core/` | `.wally/versions/3f84e7245abcc75d/core/workflow.md:26-54` | `sop-cli` | 阶段产物 | [业务架构全景](overview/business-architecture.md) |

---

## 4. 业务流程、契约与验证速查

- **全景认知**：[系统技术架构](overview/system-architecture.md) · [业务架构全景](overview/business-architecture.md) · [主干调用链全景](overview/callchain-panorama.md) · [领域术语与符号映射](overview/glossary.md)
- **上下游与契约**：[集成总览](integrations/index.md) · [字段映射](integrations/topology-and-mapping.md) · [API 参考](contracts/api-reference.md) · [错误码字典](contracts/error-codes.md) · [配置手册](contracts/configuration.md)
- **业务与数据**：[业务流程总表](business/index.md) · [KV 命令工作流](business/kv-command-workflow.md) · [SOP 任务工作流](business/sop-task-workflow.md) · [领域与数据模型](business/domain-models/index.md)
- **验证、雷区与边界**：[构建与单测映射](operations/dev-and-test.md) · [部署与运行时](operations/deployment-and-runtime.md) · [排障 Runbook](operations/runbook-and-observability.md) · [高危雷区与历史陷阱](operations/pitfalls-and-adr.md) · [认知边界](operations/boundary.md)

---

## 5. 外部权威页（SOP 专用，不改写）

| 路径 | 职责 |
|---|---|
| [apis/index.md](apis/index.md) | API 索引模板 |
| [apis/_template.md](apis/_template.md) | 单接口契约模板 |
| [sqls/README.md](sqls/README.md) | 数据库与 SQL 规范 |
| [unittest/unittest.md](unittest/unittest.md) | 单测规范 |
| [integration_test/integration_test.md](integration_test/integration_test.md) | 集成测试规范（含静态核验段） |
| [devops/env.md](devops/env.md) | 开发环境规范 |
| [devops/test-env-deploy.md](devops/test-env-deploy.md) | 测试环境部署规范（含静态核验段） |
