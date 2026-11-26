---
kind: integration
scope: integrations-overview
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .claude/hooks/_token_common.py]
symbols: [parse_message, route, git_branch, refresh_codex, sync_task_store]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [integrations/topology-and-mapping.md, integrations/downstream/git-subprocess.md, integrations/downstream/socket-wire-protocol.md, integrations/downstream/codex-collector.md]
---

# 上下游集成与调用拓扑总览

> 本仓只有三条出向链路：TCP 响应回写、git 分支子进程、Codex 采集器回填。不存在 BaseAdapter / HTTP SDK 之类的中间业务封装层，因此不存在「三明治夹心」遗漏。配套阅读：[字段映射](topology-and-mapping.md) · [模块依赖矩阵](../architecture/module-dep-matrix.md)。

## 1. 双向拓扑总表

| 方向 | 上游发起方 | 传输机制 | 下游目标 | 契约页 |
|---|---|---|---|---|
| 入向 | 本机 socket 客户端 | TCP `recv(4096)` | `nosql.py` 引擎 | [API 参考](../contracts/api-reference.md) |
| 入向 | IDE Hook 运行时 | stdin JSON | `sop_context.py` `--hook` | [API 参考](../contracts/api-reference.md) |
| 出向 | `nosql.py:191-192` | `sendall` | TCP 客户端 | [socket 线协议](downstream/socket-wire-protocol.md) |
| 出向 | `.wally/scripts/sop_context.py:90-92` | `subprocess.run` | `git` CLI | [git 子进程](downstream/git-subprocess.md) |
| 出向 | `.claude/scripts/token_report.py:68-76` | `subprocess.check_output` | `git` CLI | [git 子进程](downstream/git-subprocess.md) |
| 出向 | `.claude/scripts/token_report.py:406-407` | `subprocess.run` | Codex 采集器 | [Codex 采集器](downstream/codex-collector.md) |

## 2. 上游清单（谁进入本仓）

| 上游 | 入口锚点 | 请求形态 | 是否需鉴权 |
|---|---|---|---|
| 本机客户端 | `nosql.py:173-178` | `COMMAND; KEY; VALUE; VALUE_TYPE` 文本 | 无 |
| Claude Hook | `.claude/settings.json:3-38` | 客户端事件 payload | 无 |
| Codex Hook | `.codex/hooks.json:3-62` | 客户端事件 payload | 无 |
| Cursor Hook | `.cursor/hooks.json:3-40` | 客户端事件 payload | 无 |
| CodeBuddy Hook | `.codebuddy/settings.json:3-30` | 客户端事件 payload | 无 |
| Agent / 开发者 CLI | `.wally/scripts/create_task.py:164-173`、`.wally/scripts/sop_context.py:394-401` | argparse 参数 | 无 |

## 3. 下游清单（本仓调谁）

| 下游目标 | 调用方 | 请求字典 Key 数 | 返回解包字段 | 契约页 |
|---|---|---|---|---|
| TCP 客户端 | `nosql.py:191-192` | 4 个报文字段 + 2 个响应字段 | `bool`、`message` | [socket 线协议](downstream/socket-wire-protocol.md) |
| git CLI（路由） | `.wally/scripts/sop_context.py:90-92` | 6 个 `subprocess.run` 键 | `returncode`、`stdout`、`stderr` | [git 子进程](downstream/git-subprocess.md) |
| git CLI（报表） | `.claude/scripts/token_report.py:68-76` | 5 个 `check_output` 键 | 分支字符串或空串 | [git 子进程](downstream/git-subprocess.md) |
| Codex 采集器 | `.claude/scripts/token_report.py:401-412` | 4 个 `subprocess.run` 键 | `returncode` → 告警串 | [Codex 采集器](downstream/codex-collector.md) |

## 4. 契约不变量

1. **零外部网络出向**：除本机 `localhost:50505` 外，源码中没有 `requests`、`httpx`、`grpc` 等外部 HTTP/RPC 调用；跨仓 `repoDeps` 为空。
2. **零凭据落库**：`.wally/config.json:3-27` 的 deploy/it 均为 pending 且 command 为空，未写入任何密钥；`.wally/install-state.json:2-7` 只记录哈希与版本，账本底座 `.claude/hooks/_token_common.py:293-360` 不处理凭据。
3. **出向失败降级**：子进程失败分别降级为 `ValueError`（路由）、空串（报表分支）、告警文本（采集器回填），见对应契约页。
4. **字典 Key 全展开**：每个下游页均列出真实请求组装键与返回解包字段，不以「其余类似」省略。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 增加出向调用：先建 `integrations/downstream/<目标>.md`，再更新本页第 3 节与 [字段映射](topology-and-mapping.md)。
2. 改调用参数：对应下游契约页 → [字段映射](topology-and-mapping.md) → [错误码字典](../contracts/error-codes.md) → `.wiki-meta.json` 的 `fileToDocs`。
3. 改入向接口：[API 参考](../contracts/api-reference.md) → [组件总表](../architecture/index.md) → [根路由](../INDEX.md)。
4. 必跑验证：`python3 -m py_compile nosql.py .wally/scripts/sop_context.py .claude/scripts/token_report.py`；`python3 .wally/scripts/sop_context.py --hook --client codex --event PreToolUse` 输入 `{}` 观察 JSON 输出。
