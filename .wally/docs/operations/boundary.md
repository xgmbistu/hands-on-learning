---
kind: operations
scope: knowledge-boundary
status: draft
origin: generated
source_paths: [nosql.py, .wally/install-state.json, .wally/config.json, .wally/scripts/sop_context.py, .wally/versions/3f84e7245abcc75d/core/knowledge.md]
symbols: [main, route, find_repo, sync_task_store]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [operations/dev-and-test.md, operations/pitfalls-and-adr.md]
---

# 认知边界与快照声明

> 本页声明本知识库的基准快照、仓库外黑盒、动态分发局限与外部权威页清单。任何跨出本边界的结论都需要先取得源码证据。配套阅读：[高危雷区](pitfalls-and-adr.md) · [部署与运行时](deployment-and-runtime.md)。

## 1. 基准快照

| 项 | 值 |
|---|---|
| 工作区模式 | 单仓（`single-repo`），根目录 `repo/` |
| 分支 | `chore/wally-init-20261126-c107` |
| 提交 SHA | `14fea139ecbc1b7509d19a6d4fbcb83502c3be6a` |
| 工作树状态 | 生成前 `git status --short` 为空（干净） |
| `TARGET_LANG` | `zh-CN`（用户提问语言；`install-state.language` 为 `en`，仅影响建任务文案） |
| 生成模式 | `FULL`（无既有 `.wiki-meta.json` 基线） |
| 规则 revision | `3f84e7245abcc75d`（`.wally/install-state.json:4-5`） |
| 受保护人工页 | 0 篇；无人工确认或已核验页面 |

工作树在本次生成中只新增/更新 `.wally/docs/` 内容；业务源码与 `.wally/config.json` 等均未改动，也未执行 Git 提交或推送。

## 2. 仓库外黑盒

| 黑盒 | 本仓是否含实现 | 可观察部分 | 文档处理 |
|---|---|---|---|
| socket 客户端程序 | 否 | 仅服务端 `nosql.py:167-193` | [线协议契约](../integrations/downstream/socket-wire-protocol.md) 标注为外部黑盒 |
| Claude / Codex / Cursor / CodeBuddy 运行时 | 否 | 仅事件 payload 消费逻辑 | [API 参考](../contracts/api-reference.md) 只列本仓注册与字段 |
| 真实 Token 账本与 transcript | 否 | 仅解析与汇总代码 | [数据模型](../business/domain-models/index.md) 注明快照未包含 |
| 目标测试/部署环境 | 否 | 仅 `.wally/config.json:3-27` 的 pending 与 gaps | [部署与运行时](deployment-and-runtime.md) 不编写部署步骤 |
| 历史任务与 compat/v1 快照 | 否 | 仅恢复路径规则 | [高危雷区](pitfalls-and-adr.md) 第 4 节 |
| 外部 HTTP / RPC / MQ / 数据库 | 否，未找到调用 | 无 | 不生成跨仓依赖边 |

## 3. 动态分发局限

1. **命令分派是静态表**：`nosql.py:155-164` 的 `COMMAND_HANDERS` 固定 8 键；表外命令无动态注册机制，且会在统计阶段越界。
2. **任务路由依赖运行时 Git 分支**：`.wally/scripts/sop_context.py:90-104` 以 `git branch --show-current` 匹配 overview 的 Branch 字段；分支切换或字段格式变化会改变结论。
3. **Hook 事件由客户端决定**：本仓只能枚举已写入 `settings.json` / `hooks.json` 的 20 条注册；客户端新增事件不会自动出现在文档中。
4. **模型与用量来自运行时**：`.claude/scripts/token_report.py:311` 对缺失 model 显示 `unknown`；本仓不含真实模型清单或用量数值。
5. **账本路径可被环境变量覆盖**：`WALLY_TOKEN_CLAIMS` 与 `WALLY_CODEX_STATE_DIR` 改变读写位置，静态快照无法预知实际文件。
6. **配置快照冻结在建任务时**：`.wally/scripts/create_task.py:114-115` 把当时的 `config.json` 写入 `execution.json`；后续配置变更不会回写旧任务。

## 4. 外部权威页清单（SOP 专用）

以下页面属于 SOP 执行规范，本 Wiki 只链接不复制正文、不改写为 codewiki 模板，依据 `.wally/versions/3f84e7245abcc75d/core/knowledge.md:17-18`：

| 路径 | 职责 | 未计入 `pageOrder` 的理由 |
|---|---|---|
| `.wally/docs/apis/index.md` | API 索引模板 | SOP 专用权威页 |
| `.wally/docs/apis/_template.md` | 单接口契约模板 | SOP 专用模板 |
| `.wally/docs/sqls/README.md` | 数据库与 SQL 规范 | SOP 专用权威页 |
| `.wally/docs/unittest/unittest.md` | 单测规范 | SOP 专用权威页 |
| `.wally/docs/integration_test/integration_test.md` | 集成测试规范（含静态核验段） | SOP 专用权威页 |
| `.wally/docs/devops/env.md` | 开发环境规范 | SOP 专用权威页 |
| `.wally/docs/devops/test-env-deploy.md` | 测试环境部署规范（含静态核验段） | SOP 专用权威页 |

上述 7 页为「外部引用页」，在 [根路由](../INDEX.md) 中直接链接；G1 的 `pageOrder` / `pageCount` 只统计本 Wiki 管理的页面。

## 5. 未验证与不声称项

| 项 | 状态 | 理由 |
|---|---|---|
| 自动化测试 | 未找到 | 仓库无测试文件与框架 |
| 增量覆盖率 | 未验证 | 无覆盖率工具与命令 |
| 部署与 IT | BLOCKED | `.wally/config.json:8`、`:20` 为 `pending`，4 条 gaps 未解决 |
| 生效版本验证 | 未配置 | `.wally/config.json:6` |
| 回滚 | 未配置 | `.wally/config.json:7` |
| 跨仓依赖 | 不存在 | 单仓，源码无跨仓调用 |
| 人工页漂移 | 无 | 本次无受保护页面 |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改基准快照：本页第 1 节 → `.wiki-meta.json` 的 `commitHash` / `branch` → [根路由](../INDEX.md)。
2. 新增外部权威页：本页第 4 节 → [根路由](../INDEX.md) 的外部引用清单 → `.wiki-meta.json` 的 `externalPages`。
3. 改黑盒边界：本页第 2 节 → 对应契约页 → [认知边界总览](../overview/system-architecture.md)。
4. 必跑验证：`git rev-parse HEAD` 应返回本页 SHA；`git status --short` 确认仅 `.wally/docs/` 变更；`git ls-files` 核对第 4 节 7 个外部页路径真实存在。
