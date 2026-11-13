# Hermes 改动边界分析

> **对象**：`D:\project\Agent\hermes-agent` 作为 NMC Agent 二次开发底座  
> **快照**：本地 `main`，`a84025ade22d2a4085f575d4353072e5b702635d`（2026-11-11），工作区干净；Git 显示落后 `origin/main` 632 个提交。  
> **问题校正**：模板原句倾向问“哪些改动变得不可能”。本地证据不足以把多数能力判为绝对不可能；本报告改分为 **配置可改 / 扩展可改 / 需要源码改动 / 证据不足 / 需要测量**。源码可修改不代表工作量低，也不等于已确认业务需求。  
> **方法**：完整读取 `template\改动边界提取模板.md`；检查当前源码、依赖声明、配置消费者、主要调用链及已有能力手册。已有 XMind/对齐项不作为已确认需求，本报告不据其作业务判断；不作法律结论，不读取或披露密钥。

## 前置结论

Hermes 不是将某个外部 Agent 框架作为一个可配置替换的黑盒依赖；它是由本仓库 Python Agent loop、注册/dispatch、SQLite session state、Gateway 与多种客户端共同组成的运行底座。Provider、tools/toolsets、Skills、plugins、MCP、memory/context engine、terminal environment 等**局部子系统可通过配置或扩展替换/关闭**；Agent turn loop 和 SessionDB 没有发现全局配置级替代接口，但可维护源码分支或在外围实现独立服务，不应宣称“不能改”。`AIAgent` 与主 turn loop 见 `run_agent.py:214-280`、`agent\conversation_loop.py:1646-1802`；存储见 `hermes_state.py:181,464-471`。

```text
产品入口 (CLI/TUI/Desktop/API/ACP/Gateway)
  └─ Hermes in-repo AIAgent + turn loop                ← 没有全引擎配置替换器
      ├─ ProviderProfile / API transport                ← provider 和 transport 可扩展
      ├─ Tool Registry / Toolsets                       ← session schema 面可配置
      │   ├─ Native plugin / skill                      ← 插件与指令扩展
      │   ├─ MCP client → external MCP server            ← 外部协议边界
      │   └─ terminal/browser/file/media backends        ← 多后端；权限各异
      ├─ ContextEngine / MemoryProvider                  ← 局部治理后端可换
      └─ SessionDB (SQLite/WAL/FTS5)                     ← 当前 in-repo 主会话存储
         + Cron jobs.json / Kanban SQLite / workspace checkpoint（各自不同状态）
```

### 耦合与依赖层

| 层 | 当前事实 | 分类/限制 |
|---|---|---|
| 入口与 Agent 核心 | `hermes-agent` 自身提供 `AIAgent`，循环位于本仓库 sibling modules；CLI/Gateway/ACP/API 复用或接入此核心。 | **进程级/源码级耦合**：在本程序内改变执行循环需改源码；对外可以另开服务/进程，不是架构上不可能。 |
| 必需 Python 依赖 | `pyproject.toml` 声明 Python `>=3.11,<3.15`、45 条 direct dependency 条目；`uv.lock` 有 355 个 package 条目。Core 依赖有版本 marker；多数 direct pin，另有 extras 和 PM profile。 | **安装/构建耦合**：不能凭 toolset 开关减少 direct install closure；具体环境安装哪些包由 extras/平台/PM profile 决定。 |
| UI workspace | Web、Desktop、TUI 使用单独 pnpm workspace/锁文件。 | 若仅部署 headless Gateway/CLI，不代表必须构建每个前端；是否由具体安装包携带 UI 需查看该 build profile。 |
| DB/状态 | core session DB 直接使用 `sqlite3`、`state.db`、WAL/FTS；`database.journal_mode` 仅控制 SQLite journal。 | SessionDB 后端不是已发现的配置插件；替换要改 facade/siblings、schema、迁移及所有读写消费者。 |
| 外部服务 | LLM、聊天平台、HTTP MCP、云 sandbox、部分 Provider 都是网络/凭据依赖。 | 对应 feature 可因未配置凭据而 gate 掉；并非每项都要安装专属 SDK，也不能将静态目录视为 endpoint 已连通。 |
| 遥测 | `nemo-relay` 出现在 core 依赖；共享指标 `enabled/send` 默认关闭，Gateway OTLP export 和其 OpenTelemetry extra 也默认关闭/可选。 | 不能把“依赖在 lock 中”理解为数据已上传；开启行为由 consent 与 monitoring config 分别控制。 |

证据：`pyproject.toml:5-18,19-69,208-222,224-540,585-615`；`hermes_state.py:181,464-471,797-815`；`hermes_state_fts.py:46-59`；`hermes_cli\config_defaults.py:30-38,2050-2076,2310-2316`；`pyproject.toml:461-469`。

## 1. 运行时引擎替换 — 配置不可整体替换，源码可改，绝非“不可能”

| 可换/可控范围 | 边界及证据 |
|---|---|
| Provider/模型 | `model/providers/fallback_providers`、credential pool、`ProviderProfile` 可换模型来源或自定义 provider；用户 provider profile 可覆盖同名 bundled profile。此处换的是 **推理后端/请求协议**，不是换 Agent loop。`providers\README.md:20-25,29-55,59-75`；`providers\base.py:142-238`；`hermes_cli\runtime_provider.py:128-178,1061`。 |
| Context 管理 | `context.engine` 默认 `"compressor"`，可选已安装的 ContextEngine plugin；`compression.enabled` 控自动 compaction。只替换上下文治理，不替换迭代执行、tool dispatch 或持久化。`hermes_cli\config_defaults.py:1280-1294,563-625`；`agent\agent_init.py:1372-1428,1849-1895`。 |
| Memory | Built-in memory 可由 `memory.memory_enabled` / `user_profile_enabled` 管；一项外部 MemoryProvider 可通过 `memory.provider` 选。只替换/扩展 recall、sync 等 provider 能力。`agent\memory_provider.py:85-140`；`agent\memory_manager.py:199-290,351-373`。 |
| Tools/Skills/MCP | 通过 toolsets、plugins、skills、MCP 扩展模型可见能力，但调用循环仍执行 Hermes 的 `tool_calls → validate → dispatch → tool result`。`agent\agent_init.py:994-1021`；`agent\turn_tool_round.py:72-145`；`model_tools.py:524-552,902-976`。 |
| 完整引擎 | 未发现可在 `config.yaml` 中选择另一个 Agent runtime/turn-loop 的公共配置键；`run_conversation` 直接运行仓库实现。 | **需源码改动**：核心 turn phase、消息格式或执行生命周期变化需改 `agent\conversation_loop.py`、`agent\turn_*.py`、`run_agent.py` 及调用它们的入口。也可在产品外层启动另一个 Agent service；两者是实现选择，不是“不可能/法律边界”。 |

**定制入口**：Provider/Profile、ContextEngine、MemoryProvider、plugin hooks/ctx、MCP server 都是逐组件 seam；`plugins\AGENTS.md` 的维护规则也要求插件使用公开 `ctx`/ABC，不能由插件直接改 `run_agent.py`/`gateway/run.py` 等 core 文件。新增通用 seam 本身需要 source change。引用：`plugins\AGENTS.md:5-16`、`website\docs\developer-guide\plugins\public-api.md:10-18`。

## 2. 执行模型范式切换 — 当前默认范式固定于源码，但可做外围/源码扩展

当前 turn 是同步协调的迭代 loop，预算与中断检查后依序完成 preflight、组装请求、API retry、响应解析、tool round 或 final response；每个阶段拆在 `agent/turn_*.py`。`agent\conversation_loop.py:1646-1802`。

| 期望改变的范围 | 当前边界 | 分类 |
|---|---|---|
| 改 model/provider/api_mode、fallback、prompt/personality、toolset、skills/context、单工具行为 | 有 config/插件/工具注册入口。 | **配置可改 / 扩展可改**，但能力只覆盖这些层。 |
| 新增特定 job、后台触发 | 可用 Cron job（JSON 持久化）或 Kanban；这些不等于通用图引擎。 | **扩展/已有子系统可做**；业务流程语义待用户定义。 |
| 把全局 loop 改成 DAG/FSM、改变 tool/message/turn 执行协议、以其他 Agent runtime 替代 | 未发现运行时 engine selector；当前实现直接执行 loop。 | **需要源码改动或旁路独立服务**。没有证据说不可实现；需评估 prompt cache、session persistence、各种 API surface 的协同影响。 |
| 性能/工作流启动成本 | Async 服务、cache、lazy provider/MCP discovery 等存在，但循环/产品资源开销与 NMC 设备无法从源码静态计算。 | **需测量**。 |

**不可忽略的兼容约束**：源码设计要求会话 prompt prefix 稳定；工具集、memory 等 system-prompt 状态一般在 session 建立/边界变更，compression 是允许的历史重写路径。对 loop/prompt 的改动要验证各 provider 的序列化与 prompt cache，而不是只改一个 API call。见 `AGENTS.md:11-30`、`website\docs\developer-guide\context-compression-and-caching.md`、`agent\turn_tool_round.py:127-175`。

## 3. 状态持久化语义定制 — 局部可换，全局 SessionDB 需要源码验证

| 状态域 | 存储/生命周期 | 可配置/扩展 | 尚不能据证据宣称 |
|---|---|---|---|
| Conversation/session | `HERMES_HOME\state.db` SQLite；WAL/FTS；profile scoped；turn 有 durable lease。 | journal_mode、retention/prune/archive 配置；通过 Gateway/SessionDB API 读写。 | 没有发现 SessionDB 整体的 external DB/StorageProvider 配置接口；不能由 memory provider 推断 SessionDB 可换。 |
| Tool-call turn | Loop 会尝试在执行副作用前保存 assistant tool-call block，恢复时据持久化对话处理。 | 修改 persistence/serialization 需改 turn/session code。 | 这不是远端设备操作日志，也不等于外部系统 side effect 可 replay/rollback。 |
| Workspace 文件 | CheckpointManager 用共享 Git shadow store；默认 `checkpoints.enabled=false`，可首个 `write_file/patch` 时按 turn 快照并 `/rollback`。 | 配置快照数/容量/保留。 | 它不是可恢复整个 Agent 运行的 checkpoint，也不撤销已发消息、设备操作或其它外部 side effect。 |
| Memory | Built-in `MEMORY.md`/`USER.md` + 最多一个 external MemoryProvider。 | provider 可扩展/替换。 | memory provider 并非 state.db backend，也不改变 conversation session lineage。 |
| Cron/Kanban | Cron `jobs.json` 及执行记录；Kanban 自己的 SQLite board。 | Cron SchedulerProvider 可替换“何时 fire”；Kanban 是独立可选子系统。 | SchedulerProvider 接口标 experimental；不能把它解读成 SessionDB 或执行引擎可替换。 |
| Delegation | 子 Agent 独立上下文；background delegation process-local。 | 可调 route/toolset/concurrency/timeout；如需跨重启工作，现有指导指向 Cron 或 notify background process。 | 不持久化到 state.db 的通用工作流 checkpoint。 |

证据：`hermes_state.py:181,464-471,797-815`；`hermes_state_fts.py:46-59`；`agent\turn_facade_lease.py:1-60`；`agent\turn_tool_round.py:127-175`；`hermes_cli\config_defaults.py:30-38,482-500,2249-2298`；`tools\checkpoint_manager.py:1-45`；`cron\jobs.py:1-20,95`；`cron\scheduler_provider.py:1-5,136,387-438`；`cron\AGENTS.md` background delegation 约定。

**默认状态维护边界**：`sessions.auto_prune=false`、`auto_archive=false`，retention_days=90 是 prune/保留期值而非自动删除开关。Cron toolset off 不等于暂停已存 jobs；`cron.allow_agent_scheduling=false` 只限制 Cron Agent 再排 Cron，不是 scheduler 总开关。源码没有发现“全局禁 Cron runner”的对应 key。

## 4. 部署形态与资源约束 — 目标平台数据不足，不判硬性不可部署

1. **已证实支持**：Windows 10/11、Linux/WSL2、macOS Apple Silicon、Docker Tier 1；Nix 与 Termux Tier 2 best effort。Python/Node 服务/前端均为项目组成。`website\docs\getting-started\platform-support.md:13-50`；`website\docs\user-guide\windows-native.md:8-26`。
2. **裁剪事实**：core 依赖作用域本来是“所有 session 用的包”，Provider/web/media/audio/cloud 等有 extras 和 PM lazy install；关一个 model toolset 不会缩小已安装 venv，且 `firecrawl-anydoc` 为内置 read_file 文档提取而列为 core dependency。`pyproject.toml:19-69,224-540`。
3. **运行容器事实**：终端默认 backend 是 `local`；切 `docker/modal/daytona/vercel_sandbox/ssh` 才改变命令执行位置。Docker 默认开启网络及文件/credential 配置会影响实际隔离。`hermes_cli\config_defaults.py:284-362`；`tools\terminal_tool_backends.py:52-63`。
4. **未证实**：NMC CPU/RAM/磁盘/目标 libc/固件接口与在线/离线要求；本地没有对 target build、镜像、空闲内存、并发 RSS、冷启动做测量。故不能将“含多个依赖/仓库较大”直接升级成“不可嵌入”结论，也不能宣称关闭 feature 会有某个性能收益。

## 5. 核心引擎/上游演进 — 具备 fork 与配置边界，维护成本待估

- Python direct dependencies 大部分显式 pin，项目要求 pin 改动更新 `uv.lock`；PM 管理安装环境，extras 有 platform gates。**版本可通过依赖/构建工作流调整**，但不意味着任意上/下游版本兼容。`pyproject.toml:19-39,585-635`；`pm\`；`uv.lock`。
- Native plugin 兼容约定是 documented behavior 增量演进，不使用一个全局 `PLUGIN_API_VERSION`；公开 import 列表由 `hermes_cli\data\plugin_api.json`/validator 约束，内部 import paths 不是稳定接口。`website\docs\developer-guide\plugins\index.md:235-254`；`website\docs\developer-guide\plugins\public-api.md:10-18`。
- 代码在本地是完整仓库，可技术上维护分支/补丁；上游未来 API/dependency 变化可能增加 merge/test 负担，但本任务未分析完整 git 历史、上游 release notes 或 fork 维护人力，**没有成本数字**。
- 当前本地分支已落后 origin tracking 632 commits，因此本报告不能代表 HEAD/线上 latest；没有 fetch、checkout 或 source update。

## 6. 扩展框架替换 — 多个局部 seam 可替换，不是一个统一总插件

| 改动目的 | 可走配置/扩展的范围 | 可能需要 core 源码的范围 |
|---|---|---|
| 新行业知识、步骤、文案 | 新建/安装 `SKILL.md`；按需要列 prerequisite MCP/tools。项目技能先 trust，并扫描；optional skills 不默认安装。 | 若需要 prompt assembly 新字段/非现有技能加载生命周期，改 prompt/skills subsystem。 |
| 新工具/外部 SaaS/设备协议 | 外部 MCP server；native plugin `ctx.register_tool`；可接 terminal/provider interface。自建业务外能力可通过上述接口进入，不必先改 Agent loop。 | 暴露能力缺少 hook/ctx seam、要做全局政策或更改核心请求/授权/数据模型时改 generic core interface；不能从插件 API 自行替换 turn loop。 |
| 新 LLM provider | `ProviderProfile` + model-provider plugin 或兼容自定义 endpoint。 | 新 wire protocol、客户端凭据生命周期、特殊 tool serialization 如无法由已有 transport/provider hooks 表达，改 provider/runtime/transport。 |
| 新聊天平台 | `BasePlatformAdapter`/`ctx.register_platform`，按 adapter contract 实现 intake/auth/delivery/config。 | 新的跨 surface session semantics、身份/授权/route 机制超出 adapter hook 时改 Gateway generic seam。 |
| 新记忆/压缩/图片/视频/终端/调度后端 | 各自 provider ABC/registry。| 不同 provider 类之间不是一个可互换的大接口；改变生命周期、持久化格式/调用 contract 需 source。 |
| UI / REST / IDE client | 使用已有 API server、ACP 或 TUI Gateway JSON-RPC。 | 新产品 UI/控制面仍需自有客户端与认证/session 集成；不能从 plugin 自动产生整套产品体验。 |

Native plugin 通过 `plugin.yaml`/`register(ctx)`；一般插件由 `PluginManager` 按配置和来源扫描/加载。General plugin 的 enable/disable 控制 import/registration，和 toolset disable 不同；但下载依赖不会自动从环境卸载。`hermes_cli\plugins.py:393,525-560`；`hermes_cli\plugins_discovery.py:112-123,269-323`；`hermes_cli\plugins_loader.py:559-690`；`hermes_cli\plugins_context.py:36`；`tools\registry.py:592-620`。

**隔离边界**：默认 `plugins.isolation: in_process`，Python 插件与 Hermes 共享进程权限；`host` 会把非 bundled plugin 放进 per-profile 普通子进程，经代理/pipe 接口调用。`host.launcher: []` 默认没有 sandbox runner，Bundled plugin 仍进程内；不能把这个进程边界宣传为容器/权限沙箱。`hermes_cli\plugin_isolation.py:100-161`；`hermes_cli\plugins_loader.py:559-690`；`hermes_cli\config_defaults.py:1691-1738`。

## 7. 投入沉淀/所有权边界（不作法律结论）

技术上可以选择配置、独立 plugin/skill/MCP repo，或维护 Hermes source branch；三者对应不同的 core diff/升级耦合。仓库自身 `pyproject.toml` 声明 MIT、`LICENSE` 有许可文本；本文件**不解释授权适用范围、衍生作品权属、商标、分发或法定义务**，需由用户另行确认。不能从“可 fork”推断后续维护成本低，也不能从“需要 patch core”推断投入归谁所有。

## 可改 vs 需改源码 vs 待测量

| 层面 | 配置可改 | 扩展可改 | 需要 core 源码 / 额外核实 |
|---|---|---|---|
| Model/Agent 输入 | 模型/provider/fallback/auxiliary、toolset、prompt/personality/context files。 | provider plugin、skill、MCP、hooks。 | 替换 turn engine、系统消息协议、预算/执行语义。 |
| Tool surface | per-platform toolsets、`agent.disabled_toolsets`、每项工具 availability gate。 | 插件 tools/MCP toolset。 | 全局授权/执行器语义改变；Toolset off 不会卸载 source/deps。 |
| Memory/context | 内存注入开关/provider 名、compression 阈值/开关、context engine 名。 | MemoryProvider/ContextEngine plugin。 | SessionDB schema/storage backend；修改核心 prompt-cache 不变量。 |
| Security/network | approvals、private URL、Docker network/mount、write-safe-root、MCP filters。 | plugin pre-tool hook/外部网络策略/MCP server。 | 全局端到端权限模型、对所有执行通路一致的策略；目标部署还需 OS/network 控制。 |
| Session/automation | session retention、Cron provider/tuning、delegation limits。 | Cron scheduler provider、Kanban/skills/cron jobs。 | 可恢复任意 long-run workflow、设备操作事务/回滚；现有 DB/任务契约要扩充。 |
| 部署/性能 | profile、backend、extra/PM profile、日志/观测开关。 | 插件/远端 MCP/sandbox backend。 | 若目标硬件不满足 runtime/dependency，要经构建和实测再决定移植/裁剪。 |

## 关键开关的真实作用范围

| 开关 | 实际改变什么 | 明确不改变什么 | 证据 |
|---|---|---|---|
| `platform_toolsets.<platform>` / `agent.disabled_toolsets` | 解析每个 Agent session 的 schemas；normal loop 再用 `valid_tool_names` 校验模型 tool calls。 | 不卸载已发现 builtin 模块/依赖，也不提供 OS 级权限限制。 | `hermes_cli\tools_config.py:572-647`；`model_tools.py:151,335-352,524-552`；`agent\agent_init.py:994-1021`；`agent\turn_tool_round.py:80-130`。 |
| `plugins.enabled/disabled` | 在 PluginManager load 前 gate manifest；禁用通常阻止相应 plugin import/register。 | 不自动删除 plugin 目录、Python 环境已安装包，也不改变 bundled plugin 的 host policy。 | `hermes_cli\plugins_discovery.py:112-123,269-323`；`hermes_cli\plugins_loader.py:559-690`；`hermes_cli\config_defaults.py:1691-1738`。 |
| `plugins.isolation` | 将非 bundled Python plugin 的 ctx 执行放入 profile host 进程；host `launcher` 可由运维配置。 | 默认 `in_process`；host 默认不是容器/sandbox；不隔离 bundled plugins。 | `hermes_cli\plugin_isolation.py:100-161`；`hermes_cli\plugins_loader.py:673-690`。 |
| `skills.project_discovery` / `trusted_project_dirs` / `auto_load` | 控项目技能扫描/信任与哪些 Skill 在新 prompt 中完整预载。 | `project_discovery=false` 不关闭 profile/bundled/external Skill；`auto_load=[]` 不关闭按需 `skill_view`。 | `hermes_cli\config_defaults.py:1431-1479`；`agent\skill_utils.py:557-591`；`agent\prompt_builder.py:1303-1461`。 |
| `memory.*` vs `agent.disabled_toolsets: [memory]` | `memory_enabled`、`user_profile_enabled`、`memory.provider` 控 memory 初始化/后端；memory toolset 控 session 向模型暴露 memory 工具。 | 单独隐藏 toolset 不等同关闭 prompt memory/provider。 | `hermes_cli\config_defaults.py:1294-1310`；`agent\agent_init.py:1155-1218`；`tools\memory_tool.py:307-309`。 |
| `compression.enabled` / `context.engine` | 关闭自动 compact 或替换上下文 engine。 | 不卸载压缩代码/SDK，也不关闭 SessionDB；长上下文会把成本/overflow 风险转移到 provider。 | `hermes_cli\config_defaults.py:563-625,1280-1294`；`agent\agent_init.py:1372-1428,1849-1895`；`agent\native_compaction.py:110-132`。 |
| `checkpoints.enabled` | 控 `write_file`/`patch` 等工作区快照。 | 不控制对话持久化或外部系统事务/回滚。 | `hermes_cli\config_defaults.py:482-500`；`agent\agent_init.py:1080-1095`；`tools\checkpoint_manager.py:1-45`。 |
| `cron.allow_agent_scheduling` / `cron.provider` | 前者防 Cron Agent 内再 schedule；后者选触发 scheduler provider，缺失/不可用回退 builtin ticker。 | 不以 `cronjob` 工具集关闭已有 schedule 的后台触发。 | `hermes_cli\config_defaults.py:1778-1838`；`cron\scheduler.py:418-430,2507`；`cron\scheduler_provider.py:387-438`。 |
| `delegation.orchestrator_enabled` / `max_spawn_depth` | 限制子 Agent 是否继续派生孙 Agent与最大深度。 | 不关闭平层 `delegate_task` 本身；关闭 toolset 才移除其模型 schema。 | `hermes_cli\config_defaults.py:1311-1373`；`tools\delegate_tool.py:242-259,602-619`。 |
| `terminal.backend` / `terminal.docker_network` | 更换执行 backend；后者只控制 Docker 命令网络。 | local 默认不隔离；Docker 网络 key 不限制 MCP/provider/plugin 等其它流量。 | `hermes_cli\config_defaults.py:284-362`；`tools\terminal_tool_backends.py:52-63`。 |
| `approvals.mode` / `approvals.deny` | 普通危险命令审批策略；deny/hardline 是更早的拒绝层。 | `mode=off`/YOLO 不绕过 hardline、用户 deny、受保护文件硬拒绝。 | `hermes_cli\config_defaults.py:1645-1705`；`tools\approval.py:1088-1232`；`tools\approval_floors.py:1-25,98-122`。 |
| `telemetry.shared_metrics` / `monitoring.gateway_health_export` | 分别控制本地匿名聚合指标 collect/send 与 Gateway 运维诊断 OTLP。 | 二者不是同一功能开关；关闭上报不关本地日志。 | `hermes_cli\config_defaults.py:2050-2076,2310-2316`；`hermes_cli\observability\shared_metrics_consent.py:1-20`；`agent\monitoring\gateway_health_export.py:279-336`。 |

**调用链差异提示**：Gateway `_load_gateway_config()` 使用不合并 `DEFAULT_CONFIG` 的 effective-user loader，CLI `load_config()` 则合并默认值。因此一个新增/未显式保存的 key 不能仅凭 `DEFAULT_CONFIG` 或 `cli-config.yaml.example` 断言每个 surface 都读到相同值。证据：`gateway\run.py:2842-2854`；`hermes_cli\config_effective.py:1-17,52-82`；`hermes_cli\config.py:1978-1989`。

**Cron/Memory 文档差异**：当前 `cron\scheduler.py:2507` 构造 Agent 时传 `skip_memory=False`；`agent\agent_init.py:1173-1218` 会进入正常 memory 初始化路径。`cron\AGENTS.md:107` 则写 `skip_memory=True`。源码比这条指引更能说明当前调用参数，但每种 Cron profile/provider 的运行行为仍未测试。

**“不可改”结论**：本次没有将模板中的任何一项标成“架构上绝对不可能”。证据支持的是“没有配置级整引擎/SessionDB 替换 seam”、“局部 provider/tool/API 可以扩展”、“剩余行为改 core 需源代码工作”及“资源/成本未测量”。

## 摘要

Hermes 的改动边界由自有 `AIAgent` loop、tool registry 与 SQLite SessionDB 等 core，以及围绕它们的插件/配置接口共同决定；常见能力可以从配置或扩展层改变，但不会因此自动裁掉包、源码或宿主权限。真正涉及 Agent loop、session storage、跨入口授权/状态语义时，需要改 core 或建设外围服务；NMC 协议需求、设备规模、安全策略与嵌入式目标仍待用户确认/测量，因此不做选型建议或实现承诺。

