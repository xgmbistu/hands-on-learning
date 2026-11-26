---
kind: panorama
scope: business-architecture
status: draft
origin: generated
source_paths: [AGENTS.md, .wally/core/scope.md, .wally/versions/3f84e7245abcc75d/core/workflow.md, .wally/versions/3f84e7245abcc75d/core/gates.md, .wally/profiles/project.md, .wally/config.json]
symbols: [token_work_disabled, method_configured, route, checkpoint_issue, create]
upstream: [INDEX.md]
downstream: [business/index.md, overview/system-architecture.md]
---

# 业务架构全景

> 本仓存在两条互不相交的业务线：一是对外提供 8 个键值命令的 `nosql.py` 服务，二是对内治理开发交付的 SOP 六阶段流程与 Token 计量。触发角色、流程总表与全局不变量在此锁定。配套阅读：[业务流程总表](../business/index.md) · [系统技术架构](system-architecture.md)。

## 1. 触发角色与业务域

| 角色 | 触发动作 | 进入的业务域 | 入口锚点 |
|---|---|---|---|
| 本机客户端 | 建立 TCP 连接并发送 `COMMAND; KEY; VALUE; VALUE_TYPE` | KV 存储域 | `nosql.py:167-193` (`main`) |
| 开发者提交需求 | 在 IDE 中发出开发请求 | SOP 治理域 | `.wally/core/scope.md:1-3` |
| AI Coding Agent | 进入 Clarify / Plan / Implement 阶段 | SOP 治理域 | `.wally/versions/3f84e7245abcc75d/core/workflow.md:26-54` |
| IDE 事件钩子 | Stop / UserPromptSubmit / PreToolUse 等 | Token 计量域 | `.claude/settings.json:3-38` |
| SOP 阶段执行者 | 在 Deploy / IT 前调用 checkpoint | SOP 治理域 | `.wally/scripts/sop_context.py:260-291` (`checkpoint_issue`) |

| 业务域 | 状态归属 | 权威数据来源 | 失败语义 |
|---|---|---|---|
| KV 存储域 | 进程内 `DATA` / `STATS` | `nosql.py:47`、`nosql.py:12-45` | 命令级布尔标志 + 文本错误，非异常 |
| SOP 治理域 | `.wally/plans/<任务>/` 下 6 个阶段产物 | `.wally/install-state.json:2-6` 绑定的规则快照 | 校验失败抛 `ValueError`；确认待决退出码 3 |
| Token 计量域 | 任务目录 `token.json` 与 claim 账本 | `.claude/hooks/_token_common.py:550-568` | 采集缺失写告警，禁止当作零用量 |

## 2. 全量工作流总表（WF-*）

| 流程 ID | 业务域 | 入口符号 | 成功终点 | 失败出口 |
|---|---|---|---|---|
| WF-KV-PUT | KV | `nosql.py:76-80` `handle_put` | `DATA[key]` 写入并回 `key set to` | 无失败分支 |
| WF-KV-GET | KV | `nosql.py:83-89` `handle_get` | 返回值 | 键不存在回 `Error: Key [...] not found` |
| WF-KV-PUTLIST | KV | `nosql.py:92-95` `handle_putlist` | 复用写流程 | 无失败分支 |
| WF-KV-GETLIST | KV | `nosql.py:98-108` `handle_getlist` | 返回 list | 键不存在或非 list 回 `ERROR` |
| WF-KV-INCREMENT | KV | `nosql.py:111-122` `handle_increment` | `DATA[key] = value + 1` | 键不存在或非 int 回 `ERROR` |
| WF-KV-APPEND | KV | `nosql.py:125-136` `handle_append` | `DATA[key].append(value)` | 键不存在或非 list 回 `ERROR` |
| WF-KV-DELETE | KV | `nosql.py:139-148` `handle_delete` | `del DATA[key]` | 键不存在回不可删除提示 |
| WF-KV-STATS | KV | `nosql.py:150-152` `handle_stats` | 返回 `str(STATS)` | 无失败分支 |
| WF-SOP-ROUTE | SOP | `.wally/scripts/sop_context.py:87-173` `route` | 返回 branch/task/core/phase 字典 | 无分支、多任务匹配、规则路径非法均抛错 |
| WF-SOP-CREATE | SOP | `.wally/scripts/create_task.py:69-161` `create` | 原子发布任务目录 | 名称/分支/配置/规则缺失均抛错 |
| WF-SOP-CHECKPOINT | SOP | `.wally/scripts/sop_context.py:260-291` `checkpoint_issue` | 空字典表示前置满足 | 有缺口则生成指纹并索取确认 |
| WF-TOKEN-COLLECT | Token | `.claude/hooks/_token_common.py:638` `sync_task_store` | 归一化 usage 落入任务账本 | 锁超时或坏 JSON 不阻断编辑器 |
| WF-TOKEN-REPORT | Token | `.claude/scripts/token_report.py:428-482` `main` | 渲染并写回 overview 标记区 | 找不到仓库 / 分支 / overview 即返回 1 |

每个 WF 的精确条件、状态迁移与补偿说明见 [业务流程总表](../business/index.md)。

## 3. 全局不变量

1. **规则不可变**：`.wally/install-state.json:4-5` 以 `revision=3f84e7245abcc75d` 绑定规则；`.wally/scripts/sop_context.py:130-132` 拒绝非 `versions/<16位hex>/core` 的路径，禁止用最新规则兜底。
2. **六阶段不可减**：`.wally/scripts/create_task.py:57-58` 对非 6 步模板直接抛错；`.wally/versions/3f84e7245abcc75d/core/workflow.md:48` 重申新任务保留 01–06。
3. **配置不猜测**：`.wally/config.json:8-13` 的 deploy 与 `.wally/config.json:20-25` 的 it 均为 `readiness: pending` 且 command 为空；`.wally/scripts/create_task.py:20-23` 因此判为未配置，任务生成时写 SKIPPED / BLOCKED，不编造命令。
4. **分支唯一匹配**：`.wally/scripts/sop_context.py:105-114` 要求当前分支恰好匹配一条任务；零匹配给建任务指引，多匹配直接阻断。
5. **证据优于结论**：`.wally/versions/3f84e7245abcc75d/core/gates.md:3-9` 要求报告 passed / failed / unverified / inapplicable，缺失工具与全跳过运行不算通过。
6. **文档单一权威**：`.wally/versions/3f84e7245abcc75d/core/knowledge.md:13-18` 规定 SOP 执行规范留在原路径，CodeWiki 只链接不复制正文；入口见 `AGENTS.md:1-4` 与 `.wally/profiles/project.md:7-8`。

## 4. 阶段与门槛

| 阶段 | 绑定规则 | 关键门槛 | 数据源 |
|---|---|---|---|
| 01 Clarify | `core/clarify.md` | 可度量目标与未决问题 | `.wally/versions/3f84e7245abcc75d/core/workflow.md:29` |
| 02 Plan | `core/workflow.md` + `core/gates.md` | 调用链、状态机、兼容与回滚 | `.wally/versions/3f84e7245abcc75d/core/gates.md:12-22` |
| 03 Implement | `.wally/rules/` | Red → Green → 增量覆盖率 | `.wally/versions/3f84e7245abcc75d/core/gates.md:24-34` |
| 04 Deploy | `core/gates.md` | 版本、生效证据、可执行回滚；本仓未配置 | `.wally/config.json:3-14` |
| 05 IT | `core/gates.md` | 隔离、鉴权、逐条断言；本仓未配置 | `.wally/config.json:16-27` |
| 06 Review | `core/gates.md` + `.wally/review.md` | 六行时间日志、Token 汇总、Docs 核验 | `.wally/versions/3f84e7245abcc75d/core/gates.md:38-52` |

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 改业务分派：`.wally/scripts/sop_context.py:87-173` → `business/sop-task-workflow.md` → `contracts/api-reference.md` → `overview/callchain-panorama.md`。
2. 改全局门槛：`.wally/versions/3f84e7245abcc75d/core/gates.md:38-52` → `operations/runbook-and-observability.md` → `business/index.md`。
3. 改 KV 命令集合：`nosql.py:155-164` → `business/kv-command-workflow.md` → `contracts/api-reference.md` → `business/domain-models/index.md`。
4. 必跑验证：发一条 `PUT` 再 `GET` 确认往返；`python3 .wally/scripts/sop_context.py` 确认当前分支返回「无开发任务绑定」类提示且退出码 0；核对 `python3 -m py_compile` 三个入口脚本。
