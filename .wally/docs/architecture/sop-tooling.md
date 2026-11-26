---
kind: component
scope: sop-cli
status: draft
origin: generated
source_paths: [.wally/scripts/create_task.py, .wally/scripts/sop_context.py, .wally/config.json, .wally/install-state.json]
symbols: [create, execution_config, templates, route, current_section, checkpoint_issue, require_confirmation, check_tool, plain_document]
upstream: [architecture/index.md, overview/system-architecture.md]
downstream: [integrations/downstream/git-subprocess.md, business/sop-task-workflow.md, contracts/error-codes.md]
---

# SOP 工作流 CLI 与路由网关

> 框架模型为 `工作流 CLI + Hook 网关`：两个短命 Python 进程，一个负责从不可变模板创建任务，一个负责按 Git 分支路由并守阶段门槛。业务阶段语义见 [SOP 任务工作流](../business/sop-task-workflow.md)，本页只讲实现契约。

## 1. 双入口与职责边界

| 入口 | 命令 | 输出 | 退出码 |
|---|---|---|---|
| 建任务 | `python3 .wally/scripts/create_task.py YYYY-MM-DD_title --branch <分支> [--repo-root .]` | 标准输出 `Created ... (SOP 2.0 / revision)` | 成功 0，参数/校验失败 1（`.wally/scripts/create_task.py:170-173`） |
| 上下文路由 | `python3 .wally/scripts/sop_context.py [--repo .]` | 绑定文件全文拼接 | 成功 0，路由失败 1（`.wally/scripts/sop_context.py:412-415`） |
| 阶段门槛 | `python3 .wally/scripts/sop_context.py --checkpoint implement|deploy|it` | 就绪或确认提示 | 就绪 0，待确认 3（`.wally/scripts/sop_context.py:416-419`） |
| Hook 模式 | `... --hook --client <四选一> --event <事件>` | Hook JSON 到 stdout | 恒 0（`.wally/scripts/sop_context.py:469-474` 兜底） |

`create_task.py` 不做任何 Git 操作；`sop_context.py` 只读取分支名，不切换分支。两者都不执行客户配置里的 deploy/it 命令，依据 `.wally/scripts/create_task.py:26-27` 的文档串声明。

## 2. 建任务校验链

```text
.wally/scripts/create_task.py:164 main() -> create(root, name, branch)
  ├─ [:71-75] 名称正则 \d{4}-\d{2}-\d{2}_[\w.-]+ + 日期真实性 + 非默认分支
  ├─ [:77-81] inside(rel) 路径包含校验
  ├─ [:83-94] install-state schema=2/steps=6 + revision 16 位 hex + 4 个规则文件存在
  ├─ [:97-104] 任务目录不重复 + Branch 字段不重复
  ├─ [:105-113] 加载六阶段模板 + 校验共享 profile
  ├─ [:114-115] execution_config() -> JSON 快照写入 execution.json
  ├─ [:117-146] deploy/it 注入「执行状态」段落
  └─ [:149-159] 临时目录写全 -> rename 原子发布
```

关键守卫：`.wally/scripts/create_task.py:84-85` 存在未完成的 `translation_sources` 时拒绝建任务；`:86-87` 要求 SOP 已完成 2.0 初始化；`:92-94` 要求 `workflow.md`、`gates.md`、`interaction.md`、`clarify.md` 四个绑定规则文件齐全。

## 3. 执行方法判定字典

`execution_config`（`.wally/scripts/create_task.py:26-53`）读取 `.wally/config.json:2-27`；状态基线由 `.wally/install-state.json:2-7` 提供，当前仓实际值如下：

| 字段 | `deploy` | `it` | 判据 |
|---|---|---|---|
| `command` | 空字符串 | 空字符串 | `method_configured` 要求非空（`.wally/scripts/create_task.py:22-23`） |
| `guide` | `.wally/docs/devops/test-env-deploy.md` | `.wally/docs/integration_test/integration_test.md` | 仅有 guide 不足以判定已配置 |
| `readiness` | `pending`（`.wally/config.json:8`） | `pending`（`.wally/config.json:20`） | pending 强制 BLOCKED，不自动跳过 |
| `gaps` | 4 条（`.wally/config.json:9-14`） | 4 条（`.wally/config.json:21-26`） | 缺口文本进入确认提示 |
| 任务状态 | BLOCKED | BLOCKED | `.wally/scripts/create_task.py:120` 的三值判定 |

`.wally/scripts/create_task.py:46-52` 还校验 `readiness` 只能取 `None`/`pending`/`ready`，且 `ready` 必须至少配置 command 或 guide，否则抛 `ValueError`。

## 4. 路由与阶段解析

`route`（`.wally/scripts/sop_context.py:87-173`）的发现字段为三条正则常量：

| 常量 | 锚点 | 匹配内容 |
|---|---|---|
| `BRANCH` | `.wally/scripts/sop_context.py:19` | overview 表中的 `Branch` 或 `分支` 行 |
| `RULES` | `.wally/scripts/sop_context.py:20` | overview 表中的 `SOP Rules` 或 `SOP 规则` 行 |
| `PHASE` | `.wally/scripts/sop_context.py:21` | `## Current phase` 段内的 `- Phase: 0N.` |

`current_section`（`:48-52`）先切出当前阶段段，`phase_link`（`:55-59`）再取相对链接。`.wally/scripts/sop_context.py:139-143` 交叉校验阶段号与链接文件名一致，不一致则要求修复 overview。规则路径必须匹配 `.wally/versions/[0-9a-f]{16}/core`（`:131-132`）。

`documents` 组装顺序固定：`.wally/core/entry.md` → overview → `workflow.md` → `gates.md` → `interaction.md` → profile → 当前阶段文件 →（阶段 03 时）`.wally/rules/` 下所有规则文件 →（存在时）`knowledge.md` →（阶段 01 时）`clarify.md`。见 `.wally/scripts/sop_context.py:144-171`。

## 5. 门槛、确认与工具审视

| 函数 | 锚点 | 触发逻辑 | 失败出口 |
|---|---|---|---|
| `checkpoint_issue` | `.wally/scripts/sop_context.py:260-291` | implement 比对 01/02 是否完成且非模板；deploy/it 检查方法是否配置 | 返回含 `scope`/`fingerprint` 的 issue |
| `confirmation_state` | `.wally/scripts/sop_context.py:293-308` | 读 `.wally/local/sop-confirmations.json` 的 pending/approved | 结构非法抛 `ValueError` |
| `require_confirmation` | `.wally/scripts/sop_context.py:325-345` | 写 pending token，要求用户原样回复 `CONFIRM SOP <token>` | 返回提示串，调用方退出码 3 |
| `accept_confirmation` | `.wally/scripts/sop_context.py:348-366` | 从输入中提取 12 位 token 并校验 scope+fingerprint | 过期或不匹配返回「unknown or stale」 |
| `check_tool` | `.wally/scripts/sop_context.py:369-391` | 仅 `edit` 类工具；纯文档路径与任务目录内改动放行 | 其他改动走确认流程 |
| `plain_document` | `.wally/scripts/sop_context.py:210-256` | 只认 `.wally/docs/**` 与被动根文档 | 排除 `.agents/`、`.codex/` 等控制目录 |

`fingerprint` 由 `scope` 与 `evidence` 规范化后做 SHA-256（`.wally/scripts/sop_context.py:287-288`），保证证据变化后旧确认失效。

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改名称/分支校验：`.wally/scripts/create_task.py:69-75` → [SOP 任务工作流](../business/sop-task-workflow.md) → [错误码字典](../contracts/error-codes.md)。
2. 改正则发现字段：`.wally/scripts/sop_context.py:19-25` → [术语表](../overview/glossary.md) → [SOP 任务工作流](../business/sop-task-workflow.md)。
3. 改确认格式：`.wally/scripts/sop_context.py:293-366` → [配置手册](../contracts/configuration.md) → [排障 Runbook](../operations/runbook-and-observability.md)。
4. 必跑验证：`python3 -m py_compile .wally/scripts/create_task.py .wally/scripts/sop_context.py`；`python3 .wally/scripts/sop_context.py --checkpoint implement`；`python3 .wally/scripts/sop_context.py --hook --client codebuddy --event UserPromptSubmit` 输入空 JSON 观察返回结构。
