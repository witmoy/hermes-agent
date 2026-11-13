# Hermes 特性提取表

> **被提取底座**：Nous Research `hermes-agent`  
> **本地源码快照**：`main`，`a84025ade22d2a4085f575d4353072e5b702635d`（2026-11-11）；`git status --short --branch` 显示工作区干净、本地分支落后 `origin/main` 632 个提交。  
> **源码根目录**：`D:\project\Agent\hermes-agent`。路径均相对此根目录；`路径:行号` 可定位源码/文档证据。  
> **结论口径**：`[已证实]` 有代码路径/依赖声明支撑；`[文档声明]` 来自仓库文档但未在本机运行验证；`[推断/待验证]` 是源码结构所暗示而非行为测试结论；`[需测量]` 必须在目标部署实测。此表不评分、不决定 NMC 产品取舍，不把已有需求草稿当成已确认需求。

## 0. 评估边界

- 已完整读取 `agent-scenario\template\特性提取表模板.md`，按其 21 个主维度、14 个细化子维度和证据章节组织结论。评分栏不填分：评分体系衡量“候选与已确认需求的适配度”，而 NMC 设备规格、产品规则及取舍均尚未由本任务确认。
- 本次仅静态检查本地源码、依赖清单、仓库 README/文档和 Git 元数据；**没有安装依赖、启动服务或跑测试**。能定位到代码不等于验证了实际运行、目标平台兼容或性能收益。
- NMC/NetOps 检查只检索了 `tools\`、`plugins\`、`gateway\` 下 Python 源码中的 `NMC`、`SNMP`、`NETCONF`、`YANG`、`OpenConfig` 等标识；未找到专用设备协议实现。此项是有限搜索的阴性证据，不证明所有插件目录、技能或外部系统都没有相关能力。
- 不读取 `.env`、真实 `config.yaml`、OAuth 文件或其他密钥文件；不写入 `hermes-agent` 仓库、XMind 或对齐台账。

## 1. 技术栈、架构与目录结构

### 1.1 技术栈总览

| 类别 | 已证实内容 | 证据/限制 |
|---|---|---|
| 主语言与运行时 | Python Agent/CLI/Gateway；`pyproject.toml` 声明 `>=3.11,<3.15`，注释说明新安装支持基线与旧版本更新迁移兼容并非同一概念；`asyncio` 服务入口与同步 Agent turn loop 并存。 | `pyproject.toml:5-18`；循环见 `agent\conversation_loop.py:1646-1790`。版本范围不等于每种 Python/OS 组合均已测试；目标安装器/PM 版本还需另核。 |
| UI/客户端 | 仓库含 Web、Electron Desktop、Ink TUI 及 ACP 客户端/适配器，使用独立 pnpm workspace/锁文件。 | `pnpm-workspace.yaml`、`package.json`、`web\package.json`、`apps\desktop\package.json`、`ui-tui\package.json`、`acp_adapter\`。各包依赖未安装、未统计运行包体积。 |
| Python 构建/入口 | setuptools/wheel；命令入口 `hermes`、`hermes-agent`、`hermes-acp`。运行依赖由 PM/uv 管理，桌面/Web/TUI 另有 pnpm。 | `pyproject.toml:585-615`、`pm\`、`uv.lock`、`pnpm-lock.yaml`。 |
| 依赖规模（静态） | `pyproject.toml` 有 45 条 core direct dependency 声明；`uv.lock` 有 355 个锁定 package 条目。存在 Python/平台条件、optional extras 和 PM 安装配置，不能把 355 当作每个部署实际安装数。 | `pyproject.toml:19-222,224-540`；锁文件数由本地 TOML 解析得出。 |
| 数据库 | 会话/状态主库为 SQLite `state.db`，启用 WAL（不兼容环境可切 `journal_mode`）；FTS5 提供会话搜索。Cron job 使用独立 `cron\jobs.json`；Kanban 是另一个 SQLite 子系统。 | `hermes_state.py:181,464-471,797-815`；`hermes_state_fts.py:46-59`；`cron\jobs.py:1-20,95`。无证据表明 PostgreSQL/MySQL/Redis 是 core 必需服务。 |
| 向量库 | 未发现必需向量数据库依赖；外部 memory provider 可采用自己的存储/服务。 | `pyproject.toml:19-222` 无必需向量库；`agent\memory_provider.py` 是外部记忆接口。不能据此断言所有第三方插件均不使用向量库。 |
| HTTP/协议 | OpenAI-compatible HTTP API server 是 aiohttp adapter；Desktop/Web 及其他服务还用 FastAPI/Uvicorn 等依赖。MCP 有 stdio/HTTP 客户端；ACP 使用 stdio JSON-RPC；TUI Gateway 有 JSON-RPC/WebSocket。 | `gateway\platforms\api_server.py:1-5,1034-1052,1555-1575`；`pyproject.toml:19-222`；`website\docs\developer-guide\programmatic-integration.md:21-40,174-196`。 |
| 模型传输 | Provider/Model 解析后按 transport 发出请求；代码声明 chat completions、Codex Responses、Anthropic Messages 等，并有 Bedrock/Codex app server 路径。每个 provider 的可用 API/模型能力不同。 | `hermes_cli\runtime_provider.py:128-178,1061`；`agent\transports\__init__.py:1-49`；`providers\base.py:142-238`。 |
| 日志/指标 | Python logging 写 profile 日志并滚动；有调用/平台/任务等内部观测事件。OTLP Gateway health export 和共享指标收集/上报均可配置，默认关闭；OTLP 依赖是 optional extra。 | `hermes_logging.py:357,467-485,623-705`；`hermes_cli\config_defaults.py:2010-2015,2050-2076,2310-2316`；`pyproject.toml:461-469`。未做 Prometheus/Tracing 性能核验。 |
| License/版本状态 | `pyproject.toml` 标注 MIT，版本字段为开发占位 `0.0.0`；本任务不对许可证权利、义务或商业使用作法律结论。 | `pyproject.toml:5-18`、`LICENSE`。上游 release 成熟度/贡献者活跃度未从远端核实。 |

### 1.2 架构与调用拓扑

```text
CLI / TUI / Desktop-Web / Gateway平台 / ACP / HTTP API / batch / Python调用
          │
          ├─ 按入口解析 profile、config、session、model/provider
          ▼
       AIAgent 初始化 ── Prompt/Context：身份、context files、skills、memory
          │                         └─ context engine / 压缩
          ▼
       run_conversation 同步 turn loop
          ├─ provider runtime → chat_completions / codex_responses / anthropic 等传输
          ├─ tool schemas ← 工具注册表 ← 工具模块 / Plugin / MCP
          └─ 校验 tool name → middleware/hooks/approval → registry handler → backend
          │
          └─ SessionDB (SQLite/WAL/FTS5)；工具调用块在副作用执行前尝试持久化

Gateway / Desktop 后台另启动 cron scheduler；任务建立独立 Agent/session；
Kanban 是另一套持久化任务队列，delegate_task 则启动子 Agent。
```

`AIAgent` 是共享 Agent 门面，turn 阶段分布在 `agent\turn_*.py`；工具 schema、选取、执行分在 `model_tools.py`、`tools\registry.py` 和 `agent\tool_executor.py`。源码拓扑及各入口数据流：`website\docs\developer-guide\architecture.md:11-49,142-177,194-268`；实际 turn：`agent\conversation_loop.py:1646-1802`。

### 1.3 关键目录

| 路径 | 职责（源码可定位） |
|---|---|
| `run_agent.py`、`agent\` | `AIAgent`、turn loop、prompt/context、transport、memory、compression、delegation。 |
| `tools\`、`toolsets.py`、`model_tools.py` | 工具实现与注册、schema/工具集过滤、dispatch、执行后端。 |
| `hermes_cli\` | CLI/config/认证/provider 解析、plugin manager、技能/工具选择、观测与 profile。 |
| `gateway\`、`plugins\platforms\` | 多平台适配、消息收发、授权/session 路由、网关生命周期。 |
| `hermes_state*.py`、`cron\` | SQLite 会话状态与恢复；Cron job 的 JSON 状态和触发/执行。 |
| `providers\`、`plugins\model-providers\` | `ProviderProfile` registry 与模型 provider 插件。 |
| `plugins\`、`plugin-catalog\`、`skills\`、`optional-skills\` | 各类可扩展后端、平台、插件目录及技能指令库。 |
| `acp_adapter\`、`tui_gateway\`、`web\`、`apps\desktop\`、`ui-tui\` | IDE/RPC/HTTP/UI 集成表面。 |
| `pm\`、`docker\`、`nix\`、`scripts\` | 依赖环境、镜像/部署、构建/检查/发布工具。 |

### 1.4 部署与交付

| 项目 | 能力/状态 | 证据 |
|---|---|---|
| 安装/运行 | 源码/PM 管理的 Python 环境；Windows native、Linux/WSL2、macOS、Docker 等正式安装路径。 | `README.md:35-68`；`website\docs\getting-started\platform-support.md:13-50`。 |
| 容器/守护服务 | 有 `Dockerfile`、Compose；Gateway 有 systemd、LaunchAgent、Windows 等服务生命周期实现。 | `Dockerfile`、`docker-compose.yml`、`docker-compose.windows.yml`、`hermes_cli\gateway.py` 及 `gateway_*` siblings。未据此推断 NMC firmware/嵌入式目标可运行。 |
| 桌面 | Electron Desktop 是独立前端/后端连接形态；Desktop 后端与消息 Gateway 生命周期不同。 | `apps\desktop\`；`gateway\AGENTS.md` “Gateway lifecycle vs. Desktop app”；运行方式详见 `website\docs\user-guide\windows-native.md`、`desktop.md`。 |
| Kubernetes/嵌入式 | 本次未看到正式 Helm/K8s/目标设备构建证据；没有设备规格，不能判定不可部署。 | 需目标 OS/架构、RAM/CPU/磁盘/系统库与启动方式后实测。 |
| 产物大小/冷启动 | 未测量。Python lock/前端依赖数量不能换算运行内存、镜像体积或启动耗时。 | `[需测量]` 目标构建、空闲、并发与启动基准。 |

## 2. 特性总览主表（不评分）

### A. 通用能力

| 编号 | 维度 | 提取结论 | 关键证据 |
|---|---|---|---|
| A01 | DEV 开发难度 | Python core + TS 客户端；Agent/工具/平台各有可扩展接口，但本地代码面较广，很多行为经门面及 siblings 分散。`AIAgent` 有大量构造参数；不等于“简单嵌入库”。 | `run_agent.py:214-280`；`agent\conversation_loop.py:1646-1802`；`website\docs\developer-guide\architecture.md:51-109`。 |
| A02 | ECO 生态成熟度 | MIT 声明、GitHub Actions/CI、本地插件/技能目录均可见；贡献者数、社区活跃度、release 节奏未核实。 | `pyproject.toml:5-18`、`.github\workflows\`、`tests\`。不作成熟度分数。 |
| A03 | COST 成本效率 | 核心 direct 声明 45 项、锁文件 355 package entries；optional extras/PM profiles 可缩小安装选择。未测常驻资源、并发成本、镜像大小。工具集关闭不会卸载 Python 包或源码。 | `pyproject.toml:19-69,224-540`、`uv.lock`；`model_tools.py:151`。 |
| A04 | EXT 可扩展性 | Skills、native plugins、MCP、`ProviderProfile`、工具集、memory/context/terminal/platform providers、hooks 均有接口；自建 Skill/Plugin 可承载通用流程或表格/创作等域外能力，但必须有模型/工具/后端/依赖支持；整个 Agent loop/SessionDB 没有由这些接口整体替换的配置项。 | `hermes_cli\plugins_context.py:36`、`providers\base.py:142-238`、`tools\registry.py:845-900`、`website\docs\user-guide\features\plugins.md:103-131`。 |
| A05 | SYS 系统复杂度 | 多入口、多 profile、长运行 Gateway、多插件注册表、SQLite 状态与多个独立服务能力；不是单一静态二进制。不同入口读取配置的 loader 不完全相同。 | `gateway\run.py:2842-2854`；`hermes_cli\config_effective.py:1-17,52-82`；`hermes_cli\config.py:1978-1989`。 |
| A06 | REL 可靠性 | 有 provider retry/fallback、SessionDB WAL/FTS 修复、durable turn lease、自动状态恢复；文件 checkpoint 是 opt-in 的工作目录快照，不是所有副作用的交易回滚或 Agent run replay。 | `agent\turn_facade_lease.py:1-60`；`hermes_state.py:464-471,797-815`；`tools\checkpoint_manager.py:1-45`。 |
| A07 | OBS 可观测性 | 轮转日志、事件/hooks、trajectory 选项、内部共享指标；Gateway health OTLP 可选。共享指标 collect/send 默认皆 false；未找到强制 Prometheus exporter。 | `hermes_logging.py:357,623-705`；`hermes_cli\config_defaults.py:2050-2076,2310-2316`；`pyproject.toml:461-469`。 |
| A08 | PERF 性能 | registry `check_fn` TTL、工具定义 memo、SQLite reader pool/WAL、独立/可选 `uvloop`、MCP 并发限制等优化存在；对 NMC 设备、具体关闭项或并发数没有测量结论。 | `tools\registry.py:256-336,477-574`；`model_tools.py:227-302`；`hermes_state.py:471,604`；`pyproject.toml:224-232`。 |
| A09 | SEC 安全/权限 | secret scope、Gateway allowlist/DM 授权、危险命令审批、不可绕过的 hardline/deny、文件路径保护、MCP env 过滤；安全不是统一的 OS sandbox，local terminal 默认与 Agent 同用户权限。 | `agent\secret_scope.py:121-159,244-278`；`tools\approval.py:1088-1224`；`agent\file_safety.py:320-360,388-425`；`tools\terminal_tool_backends.py:52`。 |
| A10 | INT 集成能力 | 26 个 bundled platform `plugin.yaml`、39 个模型 Provider manifest；MCP（本地 `uv.lock` 锁到 `mcp 2.3.0`）、OpenAI-compatible API、ACP、HTTP/WebSocket RPC。目录计数不等于已启用、凭据可用或 upstream 保证支持。 | 本地 manifest 清点；`uv.lock`；`website\docs\developer-guide\programmatic-integration.md:21-40,174-196`；`providers\README.md:20-25`。 |
| A11 | MAG 多智能体 | `delegate_task` 子 Agent，支持 batch/并发/route/leaf-orchestrator；`max_spawn_depth=1` 默认平铺，background delegation 进程内存态；Kanban 是另一个可选、持久化队列。 | `hermes_cli\config_defaults.py:1311-1373`；`tools\delegate_tool.py:377,602-619`；`agent\async_delegation*`；`plugins\kanban\`。 |

### B. 运维场景增强能力

| 编号 | 维度 | 提取结论 | 关键边界 |
|---|---|---|---|
| B01 | CLOOP 闭环执行 | 单次 Agent 可循环调用工具直至答复/预算结束；有 Cron、Kanban 和 Skills。未见统一声明式、可配置的设备变更 SOP/DAG 执行引擎。 | `agent\conversation_loop.py:1646-1802`、`cron\scheduler.py`、`plugins\kanban\`。这是“未找到该抽象”，不是无法扩展。 |
| B02 | HITL 审批 | `approvals.mode=smart` 默认；manual/off、session/always、approval transports 等存在，unattended/cron/single-query 默认 deny。审批只覆盖进入相应 gate 的动作，不是每个 tool/API 操作都必经审批。 | `hermes_cli\config_defaults.py:1645-1705`；`tools\approval.py:1088-1232`；`website\docs\user-guide\security.md:24-64,146-191`。 |
| B03 | ISO 执行隔离 | local（默认）、Docker、SSH、Modal、Daytona、Vercel Sandbox 和插件 terminal environments；Docker 默认仍有网络，插件 host 默认非隔离。 | `hermes_cli\config_defaults.py:284-362,1691-1738`；`tools\terminal_tool_backends.py:52-63`。 |
| B04 | STATE 长流程状态 | SQLite 会话/消息、Cron `jobs.json`、Kanban SQLite、turn lease、可选 workspace checkpoint 各有持久化；子 Agent background 任务不是跨进程 durable workflow。 | `hermes_state.py:181,464-471`；`cron\jobs.py:1-20,95`；`agent\turn_facade_lease.py:1-60`。 |
| B05 | PROTO 协议/工具适配 | Shell/SSH、HTTP/MCP、Web APIs、消息平台通道已实现；未找到 core SNMP/NETCONF/YANG/OpenConfig 适配器。可接外部 MCP 或自定义 plugin/tool，但实际设备契约另待确认。 | `tools\environments\ssh.py`、`website\docs\user-guide\features\mcp.md:311-353`；有限关键词搜索 `tools\`、`plugins\`、`gateway\`。 |

### C. NMC 相关维度（仅盘点，不推定需求）

| 编号 | 维度 | 本地底座证据 | 不能据此推断 |
|---|---|---|---|
| C01 | 网络设备异构兼容/第三方纳管 | 有 SSH、shell、HTTP/MCP、platform plugin 等通用接入点；有限源码检索未发现专用 NMC/SNMP/NETCONF/YANG 抽象。 | 不能推断目标设备、命令集、厂商、拓扑或只读/写入要求。 |
| C02 | 变更安全/合规 | 有危险命令人工审批、硬拒绝规则、Gateway 用户授权、文件路径 denylist、可选文件快照。 | 文件 snapshot 不回滚远端设备；没有由本证据证明的事务、审计签名链、设备级分批或前后验证机制。 |
| C03 | 主动运维闭环 | Agent loop、Cron、tool loops、Kanban/子 Agent 支持自动化组合。 | 未证实具备特定 NMC 的“感知→决策→执行→验证→回退”业务闭环或确定性 SOP。 |
| C04 | All-in-One 硬件+平台 | Windows/Linux/macOS/Docker/Nix 等部署能力；另有 Web/Desktop/Gateway。 | 无目标板型、CPU/RAM/磁盘/系统库或 firmware 接口证据；不判定能否嵌入式部署。 |
| C05 | 自然语言交互/低运维 | 多 provider LLM、prompt/context files、skills、多语言与多个前端入口。 | NMC 产品术语、根因知识与交互规则并未作为已确认需求/现成功能。可扩展不等于已具备。 |

源码树中有 44 个 `skills\**\SKILL.md` 和 141 个 `optional-skills\**\SKILL.md`；这只是文件计数，不代表每个 profile 均已安装/启用。通用 Skill 可以依赖 MCP、脚本或已启用工具；指令文档本身不会带来缺失的模型、工具、网络权限或第三方包。

Dashboard 另有 self-hosted OIDC auth plugin (`plugins\dashboard_auth\self_hosted\__init__.py:80,320`)；本次对 `hermes_cli\`/`gateway\`/`plugins\` Python 源码的定向搜索未找到 LDAP、SAML 或 ITSM 适配器，不据此断言外部插件/服务无法接入。

### D. 新增维度

| 编号 | 维度 | 处理 |
|---|---|---|
| D01 | 未定义 | 不擅自新增评分维度；如用户确定评估范围后再补。 |

## 3. 细化子表（模板 §B.1–§B.14）

| 子维度 | `[已证实]` 的当前能力 | 关闭/变更入口与影响边界 | `[推断/待验证]` 或 `[需测量]` |
|---|---|---|---|
| B.1 部署方式 | Python 源码/PM、Docker/Compose；Tier 1 含 Windows/Linux/WSL2/macOS Apple Silicon，Nix/Termux best effort。 | 更换安装/服务形态不替换 Agent loop。 | 没有目标 NMC 镜像、K8s 发行包或目标架构构建证据；需目标设备验证。 |
| B.2 依赖数量 | 45 条 direct 声明、355 条锁定包；多数 direct pin，带环境 markers、extras、PM 安装 profile。 | `platform_toolsets` 不会移除 Python 依赖；选 PM profile/extras 才影响安装集合。`allow_lazy_installs` 是包安装策略。 | actual venv/容器内包数与磁盘占用需要按目标 profile 安装后采集。 |
| B.3 数据库 | `state.db`/FTS5/WAL；`database.journal_mode` 调整 SQLite journal。 | 该键只调 SQLite journal，不是 PostgreSQL/MySQL backend selector。 | 未发现 SessionDB 配置级 backend adapter；更换会话库需检查/修改多个 state facade/siblings 和迁移。 |
| B.4 Redis | 无 Redis core 必需直接依赖。 | 插件可自带依赖，不由 core manifest 计数涵盖。 | 缓存/队列替代或分布式 Redis 共享会话未验证。 |
| B.5 向量库 | core 会话搜索依赖 FTS5；memory provider 可自有召回存储。 | `memory.provider` 一次选一个外部 provider；它不替换 `SessionDB`。 | 无 core vector DB backend；外部 provider 的索引/体积/网络成本需单独审计。 |
| B.6 企业集成 | DingTalk、Feishu、WeCom、Teams、Email 等 platform manifests；Gateway allowlist/管理员授权；OAuth/外部 Provider auth。 | 平台按 provider credentials/config 启用；一般插件需显式 `plugins.enabled`。 | 未找到 LDAP/SAML/ITSM 的 core integration；外部 MCP/plugin 可实现，但不是内置。 |
| B.7 云集成 | Provider profiles 含 Bedrock/Vertex/Azure Foundry 等；terminal 有 Modal/Daytona/Vercel 相关 extras/backend。 | 使用服务需相应 extra/credential/账户，配置选择才激活；别把 manifest 存在视为服务已连通。 | 不测多云部署、单次 API 成本或厂商 SLA。 |
| B.8 资源占用 | 源码有 session cache、memory trim、工具 schema cache、SQLite WAL 等控制点。 | 开关影响范围见 §4；通常不等于卸载依赖/模块。 | 空闲 RAM、单/10 并发 RSS、CPU、DB/日志/附件/镜像大小均未测。 |
| B.9 冷启动/性能 | 首次 Agent build 等待 MCP discovery；工具/MCP/skills/provider metadata 有缓存或后台加载。 | `agent.mcp_discovery_timeout` 只界定首轮等待；超过会在适用入口后续刷新，不是关闭 MCP。 | 未测冷启动/首 token/首次 provider handshake/沙箱启动；任何“提速”须对照基线。 |
| B.10 部署复杂度 | PM 管理依赖；Gateway health、doctor、升级备份、日志和服务管理可见。 | `monitoring.gateway_health_export`/OTLP 是可选运维面，不是自动开启。 | TLS 终止、密钥管理、证书轮换、托管配置和管理员运维流程需按部署方式验证。 |
| B.11 运维复杂度 | `hermes doctor/update/logs`、DB repair、session/Cron maintenance、多 profile 支持。 | `sessions.auto_prune=false`、`auto_archive=false` 默认不自动删除/归档 ended sessions。 | 长期状态增长、并发恢复、备份恢复时长需要数据规模测试。 |
| B.12 可靠性 | retry/fallback、turn lease、预执行 transcript 持久化、Cron 执行状态和 SQLite 修复存在。 | `checkpoints.enabled=false` 默认；其 checkpoint 是工作目录文件快照，不等于 durable Agent run checkpoint/replay。 | 外部副作用幂等/补偿、设备变更回滚、跨进程 background delegation 恢复没有通用机制证据。 |
| B.13 安全 | approvals、hardline/deny、secret scopes、protected paths、MCP env filters、profile isolation 都有代码。 | `approvals.mode=off` 只关闭 recoverable prompts；hardline 和 `approvals.deny` 仍在。`HERMES_WRITE_SAFE_ROOT` 只管 `write_file`/`patch`。 | 不是针对目标设备的完整权限矩阵/OS capability sandbox；需授权模型和网络边界需求后评审。 |
| B.14 人工审批 | CLI/Gateway 等可接审批提示；危险命令可 smart/manual/off，unattended default deny；MCP reload/destructive slash 有确认设置。 | `.mode=off`/`/yolo` 有意改变提示行为；文件 hard-deny 与 command hardline 不同于普通审批。 | 没有按设备/网点分批审批、审批人接管设备命令等 NMC 语义证据。 |

## 4. 能力加载链、开关与定制入口

### 4.1 Agent turn 与工具调用实际链

`run_agent.AIAgent` → `agent_init.init_agent` → `_load_tools`（为 profile 发现 plugin、固定本 session 的 selection）→ `model_tools.get_tool_definitions`（工具集并集/差集）→ `ToolRegistry.get_definitions`（`check_fn` 过滤并生成 schemas）→ `agent.valid_tool_names` → turn loop `validate_tool_calls` → `tool_executor`/`model_tools.handle_function_call` → middleware/hooks/approval → `registry.dispatch` → backend → tool result/persist。  
证据：`run_agent.py:214-280`；`agent\agent_init.py:994-1021,2056-2174`；`model_tools.py:151,227-302,335-352,524-552,902-976`；`tools\registry.py:592-720,1034-1110`；`agent\turn_tool_round.py:72-145`；`agent\turn_tool_validation.py:97-160`。

同一 turn 的 API error/retry/fallback 是内层循环（`agent\conversation_loop.py:1577-1620,1767-1775`），分类与退避/恢复分别在 `agent\turn_api_error.py:1-35,135-182`、`agent\turn_recovery.py:1-20`；这不是产品可配置的 graph workflow engine。

**关键区别**：工具集主要决定“此 Agent session 给模型看哪些 schema”，正常 loop 还用 `agent.valid_tool_names` 验证模型返回。它不是卸载/删除工具模块，也不是操作系统权限边界；`model_tools.py:151` 在正常 import 时触发内置工具发现，`registry.dispatch(name, ...)` 是注册 handler 的执行入口。任何另一个直调工具的通道需检查自己的会话 grant/hosted policy。

### 4.2 运行时开关与作用域矩阵

| 能力/默认触发 | 开关/配置及**实际作用域** | 仍会存在什么 | 可用扩展/需要代码的边界 |
|---|---|---|---|
| Turn loop：每个 Agent 对话调用 `run_conversation`；`max_iterations` 构造默认 `sys.maxsize`，loop 同时看 `iteration_budget`。 | `AIAgent` 的 `max_iterations`、`agent.max_turns`/`run_budget_seconds` 等预算限制执行时长/次数；无单独的全局“关闭 loop”配置。 | provider client、prompt、turn phases 仍是 core。 | 覆盖 loop semantics/新增执行图需要改 `agent\conversation_loop.py`、`turn_*.py`/门面；对外也可独立调用其它 Agent API。 |
| 内置工具：normal import 加载会注册的工具模块；调用时才按 session 选 schema。 | `platform_toolsets.<platform>` 定义可组合工具集；`agent.disabled_toolsets` 在解析最后执行；部分 `*_DEFAULT_OFF_TOOLSETS` 初始不勾选；`hermes tools` 保存。 | 不卸载源码、模块、core direct deps；本地 terminal/MCP 与其他工具的副作用也不会随 `file` 工具关闭而消失。 | 新通用 core tool 通常涉及 `tools\...` + `toolsets.py`；本地/业务能力优先可通过 plugin/MCP/skill。 |
| `check_fn` gated 工具：建 schema 时测 credentials/reachability。 | 工具自身 `check_fn`/`requires_env`；约 30 秒缓存并按 profile scope；`hermes tools enable` 会 invalidate。它回答 reachability/opt-in，不应代替 per-session surface 选择。 | 已安装 provider SDK/依赖不因 False verdict 被卸载。 | 增加 backend 常通过 plugin/registry；每个 check_fn 的实际设备/credential 条件须行为测试。 |
| Plugins：入口 import/发现时扫描 manifest，再按 `plugins.enabled`/`disabled` gate 后调用 `register(ctx)`。 | `plugins.enabled` allow-list、`plugins.disabled` deny-list（disabled wins）；`plugins.isolation: in_process` 默认，`host` 为 profile 独立进程，`host.launcher` 默认空。 | disable 一般阻止该 plugin load/registrations，但不会清除已下载源码/环境依赖；host 进程分离不自动提供 OS sandbox。Bundled plugins 仍进程内。 | 公开 `PluginContext`/ABC 可改工具、hook、backend；宿主 API 不够才需新增通用 seam，改核心行为则需源码。 |
| Skills：prompt 首次构建插入索引；`skill_view` 按需加载全文。 | `skills.auto_load=[]` 控制每个新 session 固定全文的名单；`skills.project_discovery=True` 仅允许扫描 trusted project roots（默认 trust list 空）；`external_dirs` 加额外目录；`skills.write_approval` 默认 false；`inline_shell` 默认 false。 | 关闭 project discovery 不关闭 profile/bundled/external skills；关闭 `skills` toolset 不等同清除所有 prompt/context 文件。 | 自建 `SKILL.md` 承载流程/知识；可带 `scripts/` 和外部 MCP 前置条件。执行能力仍受现有工具与 OS 权限限制。 |
| Skill self-maintenance：Curator 在 inactivity 条件下评估 agent-created skills。 | `curator.enabled=true` 默认，周期/空闲时长/归档年龄可配；`consolidate=false` 默认关闭 LLM 合并；`skills.guard_agent_created=false`、`write_approval=false` 默认。 | 关闭 curator 不删除已存技能；技能列表和使用仍可用。Hub install 扫描与 project skill 扫描有自己的规则。 | 写入审批、归档/rollback/ledger 等配置可调；改 curator policy 要改 `agent\curator.py`/技能工具。 |
| Models/Provider：每个 Agent init/runtime provider resolution；provider profiles 首次 lookup lazy-discover。 | `model`、`providers`、`fallback_providers`、credential pool、`auxiliary.<task>`；`ProviderProfile` 的 `auth_type/api_mode/base_url/fetch_models` 等决定 route。 | 禁用某个 toolset 不会关闭 LLM 提供商；切辅助模型仅作用于对应 side task。 | 自定义 Provider 用 `plugins\model-providers\<name>` 或用户 plugin；自定义未知传输模式需注册/改 `agent\transports`。 |
| MCP：按 `mcp_servers` 启动 stdio/连接 HTTP，登记工具；首次 Agent build 有 discovery wait。 | 单 server `enabled`/工具白名单、`/reload-mcp`；`mcp.auto_reload_on_config_change=true`、`discovery_concurrency=4`。 | 安装的 MCP extra 与服务进程文件不会被 toolset off 自动卸载；工具集合更新可能重建 prompt prefix。 | 外部业务系统适合独立 MCP server；涉及 authorization/data schema 要另行验证。`prompts/list_changed`、`resources/list_changed` 目前收到但未处理。 |
| Terminal/sandbox：Agent 有 `terminal` 默认 local；执行后端选择决定实际 OS/远端容器。 | `terminal.backend`；Docker `docker_network=true` 默认，`false` 仅对 Docker 容器禁网络；另有 mounts、容器资源、HOME/env passthrough。 | 禁 `terminal` toolset 不阻止别的网络能力；local backend 依旧具有宿主进程所在账号权限。Docker mounts 可扩大 host 可见范围。 | `TerminalEnvironmentProvider` plugin 扩后端；全局进程/网络隔离需要部署级策略而非只调工具列表。 |
| 文件操作：`read_file/write_file/patch/search_files` 在 `file` toolset。 | `agent.disabled_toolsets: [file]` 只移除该工具面；`file_read_max_chars` 限制单次读取；`HERMES_WRITE_SAFE_ROOT` 限 write_file/patch；`checkpoints.enabled=false` 默认。 | read/write 工具代码仍已注册；terminal shell 可访问文件；文件 checkpoint 关闭不影响 SessionDB。 | 文件 deny/审批路径在 `agent\file_safety.py`/`tools\file_tools.py`；检查点在 `tools\checkpoint_manager.py`。 |
| Built-in memory：通常在 Agent init 读取 profile memory，工具面单独由 `memory` toolset 控制。 | `memory.memory_enabled=true`、`user_profile_enabled=true`、`provider=""`（builtin）、`write_approval=false`；`skip_memory` 是调用方参数。 | 单独禁 `memory` toolset 不保证跳过 memory prompt/provider init；`skip_memory=True` 也不必然禁内置 store（toolset 请求时仍创建）。 | 外部 `MemoryProvider` 一次选一个；改 memory backend 用 provider 插件，不替换 SessionDB。 |
| Context compression：Agent init 创建 compressor/context engine；over-threshold 自动压缩。 | `compression.enabled=true`；阈值/尾部/重试可配；`context.engine="compressor"` 默认，可改为已安装 engine 名。`compression.enabled=false` 禁止所有自动 compaction（包括 native path）。 | 模块仍加载、依赖仍保留；会话可变得更长、遇到 provider context limit；不能据关闭推断稳定性提高。 | `ContextEngine` plugin 改压缩策略；whole loop/prompt-cache invariants 仍由 core 控制。 |
| 会话 DB/维护：session 读写 SQLite/WAL/FTS；Gateway/CLI/cron 运行维护。 | `database.journal_mode` 可切 WAL/delete；`sessions.auto_prune=false`、`auto_archive=false` 默认；retention_days=90 只在维护开启时影响期限。 | `state.db` 与消息模式仍是 core SQLite；memory/provider、Cron jobs DB 不与其混为一体。 | 存储 schema/API/事务语义定制需改 `hermes_state*.py`、读写消费者与迁移。 |
| Cron：Gateway/Desktop 后台 ticker 读取 profile `jobs.json`。 | `cron.provider=""` 默认 builtin 60s ticker；`cron.allow_agent_scheduling=false` 仅拦截 Cron job 内 agent 再创建 job；`cron.max_parallel_jobs` 限制每 profile 并发。 | 隐藏 `cronjob` 工具不等于停掉已有 jobs；服务端 scheduler 仍按已存 jobs 运行。 | `CronScheduler` provider 只决定“何时 fire”；执行/交付仍由 core。Provider 接口标 experimental。 |
| Delegation/Kanban：`delegate_task` 可 spawn child；Kanban dispatcher 是独立任务队列。 | delegation toolset、`max_concurrent_children=10`、`max_spawn_depth=1`、`orchestrator_enabled=true`；嵌套默认因 depth 1 仍不生效。kanban 默认 off。 | 禁 orchestrator 不禁平层 delegate；background delegate 进程内存态，restart 不保留。 | `DelegationProvider` 可调整任务路由；持久任务用 Cron/Kanban 的既有模型。 |
| Gateway/platforms：启动 adapter，按 platform config/凭据收发及授权。 | 逐平台配置凭据、allowlist/DM policy、toolset；没有单一开关等价于“保留 Gateway 服务但禁所有 adapter”。 | 已配置后台实例、platform extras 不一定随 `agent.disabled_toolsets` 停止。 | `BasePlatformAdapter`/`register_platform` 是新增通道入口；NMC 协议不是消息 platform adapter。 |
| Network/security：工具各有局部网络 guard。 | `security.allow_private_urls=false` 约束若干 web/vision/media/cloud/CDP 路径；`security.website_blocklist.enabled=false` 默认；`terminal.docker_network=false` 只禁 Docker 网络。 | 默认 terminal local；MCP HTTP、plugin Python、普通模型 API、local shell 不被上述单一 key 全局拦截。 | NMC 全局 allowlist/隔离应在部署网络层或明确实现的集成 seam 验证。 |
| Observability/telemetry：文件日志、内部指标/hooks。 | `telemetry.shared_metrics.enabled=false/send=false`；`monitoring.gateway_health_export.enabled=false` 且 OTLP exporter extra optional。 | 禁上报不关本地 logging/内部事件；启用 collect 与 send 的范围不同。 | 可用 OTLP/Gateway health 或独立 plugin；发往外部的字段/同意策略仍需按配置确认。 |

### 4.2.1 上表开关与实际消费者证据

| 行 | 关键代码/行号（用于沿调用链复核） |
|---|---|
| Turn loop / model budget | `run_agent.py:214-280`；`agent\conversation_loop.py:1646-1802`；`agent\agent_init.py:2056-2194`。 |
| Toolset selection / schema / dispatch | `toolsets.py:11-35,69-112,447-475`；`hermes_cli\tools_config.py:69-115,572-647`；`model_tools.py:151,227-302,335-352,524-552,902-976`；`tools\registry.py:1034-1110`；`agent\turn_tool_round.py:72-145`。 |
| Plugin enable/load/isolation | `hermes_cli\config_defaults.py:1691-1738`；`hermes_cli\plugins_discovery.py:112-123,269-323`；`hermes_cli\plugins.py:393,525-560`；`hermes_cli\plugins_loader.py:559-690`；`hermes_cli\plugin_isolation.py:100-161`。 |
| Plugin/Skill 本地自建入口 | 一般 user plugin 可在 `$HERMES_HOME/plugins/<name>/` 用 `plugin.yaml` + `__init__.py::register(ctx)`；支持 pip entry point。`./.hermes/plugins/` 可发现但不会由处理会话的 Gateway 加载。用户 skill 放 profile `skills/` 或 `skills.external_dirs`；项目 `.hermes/skills`/`.agents/skills` 要先 trust。 | `website\docs\user-guide\features\plugins.md:145-178`；`website\docs\user-guide\features\skills.md:410-430,467-494`；`website\docs\developer-guide\plugins\index.md:323-353`。 |
| Skills/Curator | `hermes_cli\config_defaults.py:1431-1507`；`agent\skill_utils.py:557-591`；`agent\prompt_builder.py:1303-1461`；`tools\skills_tool.py:769-801`；`tools\skill_manager_tool.py:675-710`。 |
| Provider/transport/auxiliary | `hermes_cli\config_defaults.py:22-25,1311-1373`；`providers\base.py:142-238`；`providers\__init__.py:141-219`；`hermes_cli\runtime_provider.py:128-178,1061`；`agent\auxiliary_client.py`。 |
| MCP | `hermes_cli\config_defaults.py:515-562`；`tools\mcp_tool.py:1-80`；`website\docs\user-guide\features\mcp.md:747-817`。 |
| Terminal/file/checkpoints | `hermes_cli\config_defaults.py:284-362,482-500`；`tools\terminal_tool_backends.py:52-63`；`tools\file_tools.py:636-720,890-956,1444-1471`；`agent\file_safety.py:320-463`；`agent\agent_init.py:1080-1095`。 |
| Memory/context/compression | `hermes_cli\config_defaults.py:563-625,1280-1309`；`agent\agent_init.py:1155-1218,1372-1428,1849-1895`；`agent\memory_provider.py:85-140`；`agent\memory_manager.py:199-290,351-373`；`agent\native_compaction.py:110-132`。 |
| Session/Cron/delegation/Kanban | `hermes_cli\config_defaults.py:1311-1373,1778-1838,2249-2298`；`hermes_state.py:181,464-471`；`cron\jobs.py:1-20,95`；`cron\scheduler_provider.py:136,387-438`；`cron\scheduler.py:2501-2507`；`tools\delegate_tool.py:377,602-619`。 |
| Gateway/platform | `gateway\run.py:2842-2854,3392`；`gateway\run_startup.py:1019-1022`；`gateway\platforms\api_server.py:62-65,1034-1052,1555-1575`；`website\docs\developer-guide\adding-platform-adapters.md:15-41`。 |
| Security/network/observability | `hermes_cli\config_defaults.py:1645-1705,1747-1776,2050-2076,2310-2316`；`tools\approval.py:1088-1232`；`hermes_logging.py:357,623-705`；`agent\monitoring\gateway_health_export.py:279-336`。 |

**Cron/Memory 文档差异**：当前执行构造 `cron\scheduler.py:2507` 明确传 `skip_memory=False`，`agent\agent_init.py:1173-1215` 也按正常分支初始化 built-in/external memory；但 `cron\AGENTS.md:107` 仍写 Cron `skip_memory=True`。本表以调用代码为当前实现证据，将该指引视为可能滞后的文档；无运行测试，因此外部 provider 是否在每种 Cron 路径初始化仍应做行为验证。

### 4.3 不可混淆的几个开关

1. **`platform_toolsets` 不等同源码裁剪。** 有效 resolver 在 `hermes_cli\tools_config.py:572-647`；`agent.disabled_toolsets` 最后生效。示例文件明确 top-level `toolsets` deprecated/ignored，但 resolver 仍有历史兼容路径，故不得批量删旧配置。`model_tools.py:151`/`tools\registry.py:112-195` 显示正常启动会扫描并导入可注册的内置工具；关掉模型 schema 不代表代码、direct dependencies 不加载。
2. **工具暴露与工具执行要分开看。** Session init 构造 `valid_tool_names`；turn loop 会校验返回名。registry dispatch 是 handler 执行入口，其他 RPC/plugin/internal caller 不能只凭 `platform_toolsets` 推断权限边界。
3. **Memory、skills、compressor、filesystem checkpoints 各自是不同系统。** 禁 memory toolset 不等同关闭 `memory.memory_enabled`/provider；禁自动压缩不等同卸载 compressor；关闭文件 checkpoint 不等同不持久化 session。
4. **插件 `host` 是进程隔离，不自动是安全沙箱。** 默认 `in_process`；host 默认为普通 Python 子进程，只有运维提供 launcher 才可能额外套系统沙箱；bundled plugin 仍进程内运行。`hermes_cli\config_defaults.py:1691-1738`、`hermes_cli\plugins_loader.py:559-690`。
5. **默认/样例必须沿实际 consumer 核对。** `hermes_cli.config.load_config()` 合并 `DEFAULT_CONFIG`，Gateway `_load_gateway_config()` 用 `load_user_config_effective()` 且不合并默认值。`hermes_cli\config.py:1978-1989`、`gateway\run.py:2842-2854`、`hermes_cli\config_effective.py:1-17,52-82`。`cli-config.yaml.example` 不能单独证明运行时生效。

## 4.4 安全、网络与文件边界

| 边界 | 源码/默认行为 | 实际范围与风险限制 |
|---|---|---|
| Terminal 命令审批 | `approvals.mode="smart"`；`manual` 提示、`off` 只关闭普通审批层。`approvals.deny` 与固定 hardline 在 yolo/off 前执行。 | 对受这些 guard 管理的危险 shell 命令生效；不等同通用 ACL/设备级变更审批。`tools\approval.py:1088-1232`；`tools\approval_floors.py:1-25,98-122`。 |
| File read | `read_file_tool` 先做 Windows NT/device namespace、特殊设备、Hermes credential/internal denylist，再做文档抽取/二进制检查。 | `.env`、`auth.json`、MCP token/browser profile/vault 等阻止工具直接读取；security 文档指出 terminal shell 可绕过 read_file 的保护。`tools\file_tools.py:636-720`；`agent\file_safety.py:388-463`。 |
| File write | `write_file_tool` 检查敏感路径、instruction files、必要审批、sandbox mirror、stale overwrite；`patch` 有同类路径门控。`HERMES_WRITE_SAFE_ROOT` 是可选路径 allow-root。 | `HERMES_WRITE_SAFE_ROOT` 和 write deny 仅覆盖 write_file/patch；没有它不等于禁止 shell 写入。硬拒绝文件不可由聊天审批覆盖。`tools\file_tools.py:890-956,986-1080`；`agent\file_safety.py:320-360`。 |
| Secret scoping | `.env`/`auth.json` 等经 `agent.secret_scope` 按 HERMES_HOME/profile 读取；多 profile scope fail-closed。 | 仅有正确进入 scope 的代码具备 profile 隔离；任何新增 thread/process/env bridge 需按 source scope 处理。`agent\secret_scope.py:121-159,244-278,421-448`。 |
| Docker/network | Docker 默认 `docker_network=true`；false 才对容器运行 `--network=none`。local terminal 默认未隔离。 | 只作用于对应容器；MCP HTTP、plugin、API/model requests、local backend 不受其整体限制。`hermes_cli\config_defaults.py:284-362`；`tools\terminal_tool.py:4-10`。 |
| Web/private URL | `security.allow_private_urls=false` 对若干远端/CDP/web/vision/media path 禁止私网目标；local/sidecar browser 路径另有语义。 | 不是整个 Hermes 的 egress firewall；其他 providers/tools/插件需各自检查。`hermes_cli\config_defaults.py:1747-1767`；`website\docs\user-guide\security.md:830-851`。 |
| MCP stdio credentials | MCP stdio 不继承整个 shell env，仅配置 env + safe baseline；另有 trust/tool filters。 | config 可显式传给 server 的变量仍是 operator 决策；不覆盖 server 自己的数据/动作策略。`website\docs\user-guide\features\mcp.md:802-815`。 |

## 5. 证据章节索引（模板 A/B/C）

| 编号 | 证据覆盖/入口 | 可验证引用 |
|---|---|---|
| A01 | 技术栈/目录/Agent 入口 | `pyproject.toml:5-18,592-615`；`run_agent.py:214-280`；`website\docs\developer-guide\architecture.md:51-109` |
| A02 | License/CI/生态 | `LICENSE`、`.github\workflows\`、`tests\`；在线贡献者/发布统计未核 |
| A03 | Python deps/extras | `pyproject.toml:19-69,224-540`、`uv.lock`、`pm\` |
| A04 | Plugin/Skill/MCP/Provider | `hermes_cli\plugins_discovery.py:112-123,269-323`；`hermes_cli\plugins_loader.py:559-690`；`providers\base.py:142-238`；`tools\registry.py:845-900` |
| A05 | State/runtime/config/HTTP | `hermes_state.py:181,464-471,797-815`；`hermes_cli\config_defaults.py`；`gateway\platforms\api_server.py:1-5,1034-1052` |
| A06 | retry/persistence/lease/checkpoints | `agent\turn_tool_round.py:72-175`；`agent\turn_facade_lease.py:1-60`；`tools\checkpoint_manager.py:1-45` |
| A07 | logging/metrics/hooks | `hermes_logging.py:357-385,623-705`；`agent\monitoring\gateway_health_export.py:279-336`；`hermes_cli\observability\shared_metrics.py:1-40,212` |
| A08 | caches/compaction/resource knobs | `tools\registry.py:256-336,477-574`；`model_tools.py:227-302`；`hermes_cli\config_defaults.py:563-625` |
| A09 | approval, files, secrets, isolation | `tools\approval.py:1088-1232`；`agent\file_safety.py:320-463`；`tools\file_tools.py:636-720,890-956`；`hermes_cli\plugins_loader.py:673-690` |
| A10 | Provider/API/Channels/MCP | `providers\README.md:1-75`；`gateway\platforms\api_server.py:62-65,1034-1052,1555-1575`；`website\docs\user-guide\features\mcp.md:311-353` |
| A11 | Delegation/Kanban | `tools\delegate_tool.py:377,602-619`；`hermes_cli\config_defaults.py:1311-1373`；`plugins\kanban\` |
| B01–B05 | turn/Cron/state/security/protocols | `agent\conversation_loop.py:1646-1802`；`cron\scheduler_provider.py:136,387-438`；`hermes_state.py:464-471`；`tools\approval.py:1088-1232`；协议搜索范围见 §0 |
| C01–C05 | NMC 接入/审计/闭环/设备/交互 | 本表 §2 C 组；通用接入证据如上；没有设备型需求/硬件数据，均不评分 |
| D01 | 新增维度 | 未定义，不新增。 |

## 6. 弱项/成本输入（不替用户做决策）

本模板原 §5 要求评分门槛、补强计划和人周估算；目前没有确认的 NMC 需求、设备规模、安全模型和部署规格，因此**不计算低于门槛维度、不估人周、不判定“不可补”或替换底座**。可供后续验证的事实类别：

| 待决策/待测量项 | 当前能确认的技术事实 | 仍缺的输入 |
|---|---|---|
| 可关闭能力与安装裁剪 | 多数能力有 toolset/config gate；optional extras/PM profiles 按依赖组安装；builtin tool modules 会在正常 registry discovery 时导入。 | 用户决定保留面，再对目标安装 profile 做包清点/冷启动/RSS/模型 schema token 比较。 |
| 设备交互/授权 | 通用 SSH、shell、HTTP/MCP、plugin 接口可用；审批/secret/file guards 是分散边界。 | 设备只读范围、认证方式、网络拓扑、命令动作风险和审计要求。 |
| 嵌入式/边缘 | 项目有 Python runtime、多个服务与可选 extras；有部分 ARM64 平台支持。 | NMC CPU/架构、RAM/ROM、基础系统、网络与可用外部服务。 |
| 长流程恢复/闭环 | SessionDB 持久化对话；Cron jobs 持久化 schedule；delegate background 不跨重启；filesystem checkpoints 可选。 | 哪些业务操作需恢复、幂等/补偿/回滚定义与外部系统事务能力。 |

**裁剪原则记录**：本表没有建议删除源码。工具集开关、插件开关、Skills/Memory/Compression/telemetry 配置和 PM extras 的代码作用范围分别列于 §4；开关能降低模型 schema、阻止某插件 load 或减少某后台任务，不足以推断整进程 RSS、CPU、包体积或维护成本会下降多少。

