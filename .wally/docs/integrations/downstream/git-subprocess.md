---
kind: downstream-contract
scope: git-cli
status: draft
origin: generated
source_paths: [.wally/scripts/sop_context.py, .claude/scripts/token_report.py]
symbols: [route, git_branch, token_work_disabled, find_overview]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: [contracts/error-codes.md, business/sop-task-workflow.md]
---

# 下游契约：git CLI 子进程

> 下游系统为本机 `git` 可执行文件，非远程服务。本仓有两处独立调用，分别服务于 SOP 路由与 Token 报表。请求字典键与返回解包字段逐项展开，证据来自 `.wally/scripts/sop_context.py:90-92` 与 `.claude/scripts/token_report.py:65-78`。配套阅读：[字段映射](../topology-and-mapping.md) · [错误码字典](../../contracts/error-codes.md)。

## 1. 调用方清单

| 调用方 | 用途 | 实现锚点 | 触发时机 |
|---|---|---|---|
| `sop_context.py` `route` | 读取当前分支以匹配任务 | `.wally/scripts/sop_context.py:90-95` | 每次路由、每次 Hook、每次 checkpoint |
| `token_report.py` `git_branch` | 读取当前分支以定位 overview | `.claude/scripts/token_report.py:65-78` | 生成 Token 报表时 |

## 2. 请求组装字典（调用 A：SOP 路由）

`subprocess.run` 的完整参数字典：

| 参数键 | 实际值 | 语义 | 锚点 |
|---|---|---|---|
| `args[0]` | `"git"` | 可执行文件 | `.wally/scripts/sop_context.py:90` |
| `args[1]` | `"-C"` | 指定工作目录开关 | `.wally/scripts/sop_context.py:90` |
| `args[2]` | `str(root)` | 仓库根绝对路径 | `.wally/scripts/sop_context.py:90` |
| `args[3]` | `"branch"` | git 子命令 | `.wally/scripts/sop_context.py:90` |
| `args[4]` | `"--show-current"` | 只输出当前分支名 | `.wally/scripts/sop_context.py:90` |
| `stdout` | `subprocess.PIPE` | 捕获标准输出 | `.wally/scripts/sop_context.py:91` |
| `stderr` | `subprocess.PIPE` | 捕获标准错误（未消费） | `.wally/scripts/sop_context.py:91` |
| `universal_newlines` | `True` | 返回文本而非字节 | `.wally/scripts/sop_context.py:92` |
| `timeout` | `3` | 3 秒超时 | `.wally/scripts/sop_context.py:92` |

等价命令：`git -C <仓库根> branch --show-current`。仓库根经 `.wally/scripts/sop_context.py:402` 的 `Path(args.repo).resolve()` 得到，并由 `inside`（`:71-77`）做包含校验。

## 3. 请求组装字典（调用 B：Token 报表）

`subprocess.check_output` 的完整参数字典：

| 参数键 | 实际值 | 语义 | 锚点 |
|---|---|---|---|
| `args[0]` | `"git"` | 可执行文件 | `.claude/scripts/token_report.py:69` |
| `args[1]` | `"branch"` | git 子命令 | `.claude/scripts/token_report.py:69` |
| `args[2]` | `"--show-current"` | 只输出当前分支名 | `.claude/scripts/token_report.py:69` |
| `cwd` | `str(repo)` | 工作目录由 `find_repo` 定位 | `.claude/scripts/token_report.py:70` |
| `stderr` | `subprocess.DEVNULL` | 丢弃错误输出 | `.claude/scripts/token_report.py:71` |
| `timeout` | `3` | 3 秒超时 | `.claude/scripts/token_report.py:72` |

返回链：`check_output` → `.decode("utf-8", errors="replace")`（`.claude/scripts/token_report.py:74`）→ `.strip()`（`:75`）→ 分支字符串；任意 `Exception` 捕获后返回空串（`:77-78`）。

## 4. 返回解包字段与消费

| 调用 | 返回字段 | 提取方式 | 消费逻辑 |
|---|---|---|---|
| A | `returncode` | `git.returncode == 0` | 非 0 时 `branch` 置空 |
| A | `stdout` | `git.stdout.strip()`（`:93`） | 参与 `00-overview.md` 的 Branch 正则匹配（`:97-104`） |
| A | `stderr` | 捕获但未读取 | 不进入任何错误文本 |
| B | bytes 输出 | `decode(..., errors="replace").strip()` | `token_work_disabled`（`:58-62`）与 `find_overview`（`:91-106`） |
| B | 异常分支 | 返回 `""` | `find_overview` 无匹配 → `[ERROR] Cannot find 00-overview.md`（`:450-453`） |

## 5. 错误翻译表

| 下游状态 | 调用方翻译 | 用户可见结果 | 锚点 |
|---|---|---|---|
| A：分支为空（含失败） | 抛 `ValueError` | `Cannot route SOP without an actual Git branch; resolve Git state first.` | `.wally/scripts/sop_context.py:94-95` |
| A：分支匹配 0 条任务 | 抛 `ValueError` | 建任务指引，含 `create_task.py YYYY-MM-DD_title --branch ...` | `.wally/scripts/sop_context.py:105-114` |
| A：分支匹配多条任务 | 抛 `ValueError` | `Multiple SOP tasks match branch ...` | `.wally/scripts/sop_context.py:105-107` |
| A：Hook 模式下失败 | 顶层兜底 | 输出 `{}` 且退出 0，不阻断编辑器 | `.wally/scripts/sop_context.py:404-411`、`:469-474` |
| B：分支为 `main`/`master`/`release*` | `token_work_disabled` 返回真 | `[SKIP] Token rollup is disabled`，退出码 0 | `.claude/scripts/token_report.py:446-449` |
| B：异常或空分支 | 空串 → `find_overview` 无匹配 | `[ERROR] Cannot find 00-overview.md`，退出码 1 | `.claude/scripts/token_report.py:77-78`、`:451-453` |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改 git 参数：`.wally/scripts/sop_context.py:90-92` → 本页第 2 节 → [字段映射](../topology-and-mapping.md) → [SOP 工作流](../../business/sop-task-workflow.md)。
2. 改报表分支定位：`.claude/scripts/token_report.py:65-78` → 本页第 3 节 → [IDE 接入层](../../architecture/ide-integration-hooks.md) → [可观测性 Runbook](../../operations/runbook-and-observability.md)。
3. 改错误文本：`.wally/scripts/sop_context.py:94-114` → [错误码字典](../../contracts/error-codes.md) → [排障决策树](../../operations/runbook-and-observability.md)。
4. 必跑验证：`git -C repo branch --show-current` 应输出 `chore/wally-init-20261126-c107`；`python3 .wally/scripts/sop_context.py` 应进入无任务绑定分支并给出指引；`python3 .claude/scripts/token_report.py` 在 main 分支应输出 `[SKIP]`。
