---
kind: operations
scope: deployment-and-runtime
status: draft
origin: generated
source_paths: [nosql.py, .wally/config.json, .wally/install-state.json, .wally/profiles/project.md, .wally/docs/devops/test-env-deploy.md, .wally/scripts/create_task.py]
symbols: [main, execution_config, token_work_disabled]
upstream: [INDEX.md, architecture/index.md]
downstream: [operations/boundary.md, contracts/configuration.md]
---

# 部署、构建产物与运行时契约

> 仓库内没有 Makefile、Dockerfile/Compose、CI workflow、包清单或部署脚本；`.wally/config.json:8-14` 的 deploy 为 `readiness: pending`，未配置任何命令、目标环境、凭据或回滚。本页只描述可证实的运行期形态，不补写部署步骤。配套阅读：[构建与单测](dev-and-test.md) · [认知边界](boundary.md)。

## 1. 构建产物

| 项 | 证据 | 结论 |
|---|---|---|
| 编译产物 | 无 `setup.py` / `pyproject.toml` / `requirements.txt` | 无构建步骤，直接以源码运行 |
| 可执行入口 | `nosql.py:196-197` 的 `if __name__ == '__main__'` | `python3 nosql.py` 即启动 |
| 工具入口 | `.wally/scripts/create_task.py:176-177`、`.wally/scripts/sop_context.py:465-467`、`.claude/scripts/token_report.py:485-486` | 均为独立 CLI |
| 依赖 | 仅标准库 `socket` / `time` / `argparse` / `json` / `pathlib` / `subprocess` 等 | 无需安装第三方包 |
| 版本锁定 | 无锁文件 | 解释器版本未锁定 |

## 2. 运行期磁盘目录树契约

| 路径 | 来源 | 内容 | 是否仓库快照包含 |
|---|---|---|---|
| `nosql.py` | 跟踪文件 | 服务源码 | 是 |
| `.wally/core/` | 跟踪文件 | `entry.md`、`scope.md` 路由入口 | 是 |
| `.wally/versions/3f84e7245abcc75d/core/` | 跟踪文件 | 6 份不可变规则 | 是 |
| `.wally/versions/3f84e7245abcc75d/plans-template/` | 跟踪文件 | 六阶段模板 | 是 |
| `.wally/profiles/project.md:8` | 跟踪文件 | 项目策略 | 是 |
| `.wally/config.json` | 跟踪文件 | 执行方法配置 | 是 |
| `.wally/install-state.json:2-7` | 跟踪文件 | 初始化状态与 revision | 是 |
| `.wally/plans/` | 运行期生成 | 任务目录与阶段产物、`execution.json`、`token.json` | 否，当前快照不存在 |
| `.wally/local/sop-confirmations.json` | 运行期生成 | pending / approved 确认状态 | 否，未跟踪 |
| 各客户端 transcript 与状态目录 | 外部运行时 | Token 采集来源 | 否，外部黑盒 |

`.wally/scripts/create_task.py:149-159` 用 `TemporaryDirectory` 写全后 `rename` 发布任务目录，保证路由不会看到半写任务。`.wally/scripts/sop_context.py:97-104` 按 `*/00-overview.md` 遍历任务目录，跳过下划线与点开头的目录。

## 3. 网络与 IPC 边界

| 边界 | 实现 | 约束 |
|---|---|---|
| TCP 监听 | `nosql.py:9-11` 绑定 `localhost:50505` | 仅本机，无 TLS、无鉴权 |
| 连接模型 | `nosql.py:172-193` 串行 `accept()` → `recv(4096)` → `sendall` → `close` | 单连接、单线程，无并发 |
| git 子进程 | `.wally/scripts/sop_context.py:90-92`、`.claude/scripts/token_report.py:68-76` | 3 秒超时，分别 PIPE 与 DEVNULL |
| 采集器子进程 | `.claude/scripts/token_report.py:406-407` | 10 秒超时，捕获输出 |
| 文件锁 | `.claude/hooks/_token_common.py:396-441` | 跨 Hook 进程串行化账本 |
| 跨仓网络 | 未找到 | 无 HTTP/RPC/MQ 出向，无跨仓边 |

## 4. 部署配置现状（未配置）

| 字段 | 当前值 | 锚点 |
|---|---|---|
| `deploy.command` | 空串 | `.wally/config.json:4` |
| `deploy.guide` | `.wally/docs/devops/test-env-deploy.md` | `.wally/config.json:5` |
| `deploy.readiness` | `pending` | `.wally/config.json:8` |
| `deploy.gaps` | 4 条：无构建/部署入口、无目标环境、无凭据来源、无验证与回滚 | `.wally/config.json:9-14` |
| `deploy.verify` | `Not configured: ...` | `.wally/config.json:6` |
| `deploy.rollback` | `Not configured: ...` | `.wally/config.json:7` |

`.wally/docs/devops/test-env-deploy.md:24` 的静态核验段明确：未发现真实部署配置，未猜测命令，未执行任何构建或部署。SOP 权威部署规范保留在该文件，本 Wiki 不复制或改写正文，依据 `.wally/versions/3f84e7245abcc75d/core/knowledge.md:17-18`。

## 5. 生效验证与回滚

| 能力 | 现状 | 依据 |
|---|---|---|
| 生效版本验证 | 未配置；源码无版本端点或健康接口 | `.wally/config.json:6` |
| 健康检查 | 仅可通过 TCP 连接并发送 `STATS` 间接确认服务存活 | `nosql.py:171`、`:179-180` |
| 数据持久化 | 无，`DATA` / `STATS` 随进程退出丢失 | `nosql.py:47`、`:12-45` |
| 回滚版本 | 未配置 | `.wally/config.json:7` |
| 回滚触发 | 未配置 | `.wally/config.json:7` |
| 数据清理 | 无迁移、无备份、无清理脚本 | 仓库全量文件清单 |

## 6. 共变耦合与修改清单（Co-Change Playbook）

1. 改启动地址：`nosql.py:9-11` → [配置手册](../contracts/configuration.md) → [系统架构](../overview/system-architecture.md) → 本页第 3 节。
2. 改任务目录布局：`.wally/scripts/create_task.py:95-159` → `.wally/scripts/sop_context.py:97-104` → [数据模型](../business/domain-models/index.md) → 本页第 2 节。
3. 补齐部署配置：`.wally/config.json` → `.wally/docs/devops/test-env-deploy.md`（SOP 权威页）→ 本页第 4/5 节 → [排障 Runbook](runbook-and-observability.md)。
4. 必跑验证：`python3 nosql.py` 观察监听输出；另开终端发送 `STATS;;;` 确认响应；核对 `git ls-files` 中不存在 `.wally/plans/` 与部署脚本。
