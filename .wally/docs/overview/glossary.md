---
kind: glossary
scope: overview-glossary
status: draft
origin: generated
source_paths: [nosql.py, .wally/scripts/sop_context.py, .wally/scripts/create_task.py, .wally/config.json, .wally/install-state.json]
symbols: [parse_message, COMMAND_HANDERS, DATA, STATS, route, checkpoint_issue, create]
upstream: [INDEX.md]
downstream: [overview/system-architecture.md, overview/business-architecture.md]
---

# 领域术语与代码符号映射

> 本页锁定全站命名。自然语言术语一律映射到本仓真实符号；无实现处直写「未找到」，不做语义外推。源码锚点为仓库根相对路径，修改前用 `Read` 复核字面。配套阅读：[系统技术架构](system-architecture.md) · [主干调用链全景](callchain-panorama.md)。

## 1. 应用引擎术语

| 需求说法 | 精确符号与锚点 | 含义与边界 |
|---|---|---|
| 数据库服务 / KV 存储 | `nosql.py:167-193` (`main`) | 单文件 TCP 回显式键值服务，`localhost:50505` 单连接循环 |
| 写键值 | `nosql.py:76-80` (`handle_put`) | `DATA[key] = value`，无覆盖校验、无持久化 |
| 读键值 | `nosql.py:83-89` (`handle_get`) | 键不存在返回 `(False, 'Error: Key [...] not found')` |
| 列表写入 | `nosql.py:92-95` (`handle_putlist`) | 直接复用 `handle_put`，不强制 value 为 list |
| 列表读取 | `nosql.py:98-108` (`handle_getlist`) | 额外校验 `isinstance(value, list)` |
| 自增 | `nosql.py:111-122` (`handle_increment`) | 校验 `isinstance(value, int)` 后 `value + 1` |
| 追加 | `nosql.py:125-136` (`handle_append`) | 对 list 调 `DATA[key].append(value)` |
| 删除 | `nosql.py:139-148` (`handle_delete`) | `del DATA[key]`；成功分支没有 `return` |
| 统计 | `nosql.py:150-152` (`handle_stats`) | `str(STATS)`，把字典字符串作为响应体 |
| 命令分派表 | `nosql.py:155-164` (`COMMAND_HANDERS`) | 8 个命令字面量到处理函数的唯一映射 |
| 报文解析 | `nosql.py:50-64` (`parse_message`) | 按 `;` 切 4 段，支持 `LIST` / `INT` 类型转换 |
| 内存数据 | `nosql.py:47` (`DATA`) | 进程内全局 dict，重启即丢失 |
| 计数器 | `nosql.py:12-45` (`STATS`) | 每命令 `success` / `error` 两个计数槽 |
| 监听地址 | `nosql.py:9-10` (`HOST` / `PORT`) | 硬编码 `localhost` / `50505`，无配置项覆盖 |
| 套接字 | `nosql.py:11` (`SOCKET`) | 模块级单例，`SOCK_STREAM`，未设 SO_REUSEADDR |

## 2. SOP 工作流术语

| 需求说法 | 精确符号与锚点 | 含义与边界 |
|---|---|---|
| 开发六阶段 | `.wally/versions/3f84e7245abcc75d/core/workflow.md:26-54` | 01 Clarify → 06 Review + Commit，新任务不可减阶段 |
| 请求分类 | `.wally/core/scope.md:1-3` | 开发类进 SOP；问答、只读分析、纯文档编辑、Git 操作不进 |
| 分支路由 | `.wally/scripts/sop_context.py:87-173` (`route`) | 以 `git branch --show-current` 匹配 `00-overview.md` 的 Branch 字段 |
| 当前阶段 | `.wally/scripts/sop_context.py:48-52` (`current_section`) | 从 overview 的 `## Current phase` 段提取 |
| 阶段校验 | `.wally/scripts/sop_context.py:260-291` (`checkpoint_issue`) | implement 查 01/02 非模板；deploy/it 查执行方法配置 |
| 显式确认 | `.wally/scripts/sop_context.py:325-345` (`require_confirmation`) | 生成 `CONFIRM SOP <token>` 提示，退出码 3 |
| 确认回收 | `.wally/scripts/sop_context.py:348-366` (`accept_confirmation`) | 12 位十六进制 token，校验 scope 与 fingerprint 一致 |
| 确认状态 | `.wally/scripts/sop_context.py:293-308` (`confirmation_state`) | `.wally/local/sop-confirmations.json` 的 `pending` / `approved` |
| 建任务 | `.wally/scripts/create_task.py:69-161` (`create`) | 从不可变版本模板生成 `.wally/plans/<name>`，原子 rename |
| 执行方法 | `.wally/scripts/create_task.py:20-23` (`method_configured`) | `readiness != pending` 且 command 或 guide 非空才算已配置 |
| 配置读取 | `.wally/scripts/create_task.py:26-53` (`execution_config`) | 只读 `.wally/config.json:2`，校验 schema 1 与字段类型 |
| 规则快照 | `.wally/install-state.json:2-6` | `revision=3f84e7245abcc75d`、`steps=6` 绑定不可变规则 |
| 质量门禁 | `.wally/versions/3f84e7245abcc75d/core/gates.md:1-52` | 证据、Red/Green、增量覆盖率、时间日志、Token 汇总 |
| 知识路由 | `.wally/versions/3f84e7245abcc75d/core/knowledge.md:1-25` | `INDEX.md` 入口、`fileToDocs` 仅作影响提示 |

## 3. 集成与可观测术语

| 需求说法 | 精确符号与锚点 | 含义与边界 |
|---|---|---|
| Token 采集 | `.claude/hooks/_token_common.py:293-360` (`normalize_usage`) | 把客户端原始 usage 归一化为统一字段 |
| 请求去重 | `.claude/hooks/_token_common.py:550-568` (`claim_ledger_path`, `record_claims`) | 用 `WALLY_TOKEN_CLAIMS` 指向的账本防重复计数 |
| 文件锁 | `.claude/hooks/_token_common.py:396-441` (`file_lock`) | 并发写账本的自旋锁，超时抛 `LockTimeout` |
| 会话汇总 | `.claude/scripts/token_report.py:195-230` (`rollup_turns`) | 一轮用户对话一行；Cursor 不再二次求和 |
| 报告渲染 | `.claude/scripts/token_report.py:278-365` (`render`) | 输出 `TOKEN_USAGE_START` / `TOKEN_USAGE_END` 包裹的 Markdown 表 |
| 子命令进程 | `.wally/scripts/sop_context.py:90-92` | `git -C <root> branch --show-current`，3 秒超时 |
| 文档白名单 | `.wally/scripts/sop_context.py:210-256` (`plain_document`) | 仅 `.wally/docs/**` 与被动根文档视为纯文档 |

## 4. 易混淆对照

- **`DATA` 与 `STATS`**：前者存用户键值（`nosql.py:47`），后者存命令成功/失败计数（`nosql.py:12-45`），两者互不联动写入失败统计之外的状态。
- **`readiness: pending` 与「未配置」**：前者要求 BLOCKED 并向用户索取缺口，后者才允许自动 SKIPPED；判据在 `.wally/scripts/create_task.py:20-23`。
- **SKIPPED 与 PASS**：`.wally/versions/3f84e7245abcc75d/core/gates.md:23-36` 明确 SKIPPED 不是通过，未配置部署/IT 不得声称端到端验证。
- **`COMMAND_HANDERS` 拼写**：源码字面为 `HANDERS`（`nosql.py:155`），不是 `HANDLERS`，跨页引用必须保持原拼写。
- **SOP 权威页与 CodeWiki 页**：`apis/`、`sqls/`、`unittest/`、`integration_test/`、`devops/` 归 SOP 规范所有；本页集只链接不改写，依据 `.wally/versions/3f84e7245abcc75d/core/knowledge.md:13-18`。

## 5. 共变耦合与修改清单（Co-Change Playbook）

1. 改命令字面量：`nosql.py:155-164` → `contracts/api-reference.md` → `business/kv-command-workflow.md` → `contracts/error-codes.md` → `business/domain-models/index.md`。
2. 改 SOP 阶段或退出码：`.wally/scripts/sop_context.py:394-462` → `business/sop-task-workflow.md` → `contracts/api-reference.md` → `contracts/error-codes.md`。
3. 改 Token 字段：`.claude/hooks/_token_common.py:293-360` → `.claude/scripts/token_report.py:278-365` → `integrations/downstream/git-subprocess.md` → `operations/runbook-and-observability.md`。
4. 必跑验证：`python3 -m py_compile nosql.py .wally/scripts/create_task.py .wally/scripts/sop_context.py`；起服后逐条发送 8 个命令核对响应；`python3 .wally/scripts/sop_context.py --checkpoint implement` 观察当前分支返回码。
