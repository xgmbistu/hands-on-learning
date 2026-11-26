---
kind: operations
scope: development-and-test
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .wally/config.json, README.md]
symbols: [main, create, route, main, token_work_disabled]
upstream: [INDEX.md, architecture/index.md]
downstream: [operations/pitfalls-and-adr.md, contracts/api-reference.md]
---

# 构建、运行与单测映射

> 本仓没有 `pyproject.toml`、`requirements.txt`、`Makefile`、Dockerfile、CI 工作流或任何测试文件；README 仅有标题无正文（`README.md:1-2`）。本页只列源码可证实的命令，不编造测试框架或版本。配套阅读：[部署与运行时](deployment-and-runtime.md) · [高危雷区](pitfalls-and-adr.md)。

## 1. 运行时与版本证据

| 项 | 证据 | 结论 |
|---|---|---|
| 解释器 | `.wally/scripts/sop_context.py:12`、`.wally/scripts/create_task.py:12` 注释说明 `Path.is_relative_to` 是 3.9+，故自实现兼容函数 | 源码保留 Python 3.6/3.7 兼容写法 |
| 注解语法 | `.claude/scripts/token_report.py:19` 使用 `from __future__ import annotations`，`:81` 用 `Path \| None` | 依赖延迟注解求值 |
| 依赖清单 | 仓库内无 `pyproject.toml` / `requirements.txt` | 仅用标准库，无第三方运行依赖 |
| CI | `.github/` 下只有 `copilot-instructions.md` 与技能符号链接，无 workflows | 未找到 CI 配置 |
| 测试 | `git ls-files` 无 `test*.py` / `tests/` / `conftest.py` | 未找到自动化测试 |

具体 Python 小版本在仓库内没有可验证证据，按 `.wally/profiles/project.md:6` 的约束「遵循实际规则，不假设最新版本」处理。

## 2. 可执行的验证命令

| 目的 | 命令 | 预期 | 锚点 |
|---|---|---|---|
| 语法检查 | `python3 -m py_compile nosql.py .wally/scripts/create_task.py .wally/scripts/sop_context.py` | 无输出、退出 0 | 三个入口文件 |
| 语法检查（工具） | `python3 -m py_compile .claude/scripts/token_report.py .claude/hooks/_token_common.py` | 无输出、退出 0 | 工具脚本 |
| 启动服务 | `python3 nosql.py` | 输出 `Listening on ('localhost', 50505)` | `nosql.py:171` |
| 路由检查 | `python3 .wally/scripts/sop_context.py` | 当前分支无绑定任务，输出建任务指引 | `.wally/scripts/sop_context.py:108-114` |
| 门槛检查 | `python3 .wally/scripts/sop_context.py --checkpoint implement` | 退出 0 或 3（需确认） | `.wally/scripts/sop_context.py:416-419` |
| Hook 冒烟 | `echo '{}' | python3 .wally/scripts/sop_context.py --hook --client codebuddy --event UserPromptSubmit` | 输出合法 Hook JSON | `.wally/scripts/sop_context.py:422-462` |
| Token 分支跳过 | `git branch -M main` 后 `python3 .claude/scripts/token_report.py` | `[SKIP]`，退出 0 | `.claude/scripts/token_report.py:446-449` |

`git branch -M main` 仅建议在一次性临时克隆中执行；本仓当前分支为 `chore/wally-init-20261126-c107`，不要在工作树中直接改分支。

## 3. 源码模块 → 必跑验证映射

| 源码模块 | 变更类型 | 必跑验证 | 对照文档 |
|---|---|---|---|
| `nosql.py:50-64` | 报文解析 | 8 条命令往返 + 错误报文 | [KV 工作流](../business/kv-command-workflow.md) |
| `nosql.py:76-152` | 命令处理 | 每条命令成功与失败路径 | [线协议契约](../integrations/downstream/socket-wire-protocol.md) |
| `nosql.py:155-193` | 分派与循环 | 未知命令与 `STATS` 计数 | [引擎实现](../architecture/nosql-engine.md) |
| `.wally/scripts/create_task.py` | 建任务 | 语法检查 + 无效名称校验 | [SOP 工作流](../business/sop-task-workflow.md) |
| `.wally/scripts/sop_context.py` | 路由/门槛 | 普通模式 + checkpoint + hook 三出口 | [API 参考](../contracts/api-reference.md) |
| `.claude/hooks/_token_common.py` | 账本底座 | 四套采集器语法检查 | [IDE 接入层](../architecture/ide-integration-hooks.md) |
| `.claude/scripts/token_report.py` | 报表 | 跳过分支 + 标记唯一性检查 | [Codex 采集器契约](../integrations/downstream/codex-collector.md) |
| `.wally/config.json` | 执行方法 | 核对 readiness 与任务状态 | [配置手册](../contracts/configuration.md) |

## 4. Mock 与隔离规范

- 无单元测试框架，因此不存在既有 mock 边界；若后续补测试，外部依赖仅限 `git` 子进程与 Codex 采集器子进程，分别在 `.wally/scripts/sop_context.py:90-92` 与 `.claude/scripts/token_report.py:406-407` 注入桩。
- socket 测试使用独立本机端口进程；`nosql.py:9-11` 端口硬编码，不能并行起两个实例，须串行执行。
- 账本测试使用隔离的 `WALLY_TOKEN_CLAIMS` 路径（`.claude/hooks/_token_common.py:67`、`:551`），避免污染真实任务账本。
- 不提交原始 Token 账本或会话日志；`.wally/local/` 属于本地状态。

## 5. 覆盖率与门禁说明

`.wally/profiles/project.md:13-14` 要求绑定真实测试命令与增量覆盖率阈值，默认普通业务 80%/70%、核心 90%/80%、高风险基础设施 90%/85%。本仓无测试框架与覆盖率工具，因此这些阈值**无法验证**，不能声称达标。`.wally/config.json:20-25` 的 IT 因 `readiness: pending` 处于 BLOCKED，属于未配置而非已通过。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 新增测试框架：本页第 1/5 节 → `.wally/profiles/project.md` → [认知边界](boundary.md) 记录新增依赖理由 → [SOP 工作流](../business/sop-task-workflow.md)。
2. 改源码入口：第 3 节映射表 → [组件总表](../architecture/index.md) → [根路由](../INDEX.md) → `.wiki-meta.json` 的 `fileToDocs`。
3. 改验证命令：第 2 节 → [排障 Runbook](runbook-and-observability.md) → [高危雷区](pitfalls-and-adr.md)。
4. 必跑验证：按第 2 节顺序执行全部命令，记录退出码与输出；确认无 `py_compile` 报错、无意外分支变更。
