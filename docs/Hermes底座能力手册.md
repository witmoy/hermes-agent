# Hermes 底座能力手册

> **用途**：盘点 `hermes-agent` 当前已有的通用 Agent 能力、扩展入口、配置位置及明确限制；只描述仓库可见能力，不推导下游产品需求。  
> **证据口径**：源码根目录为 `D:\project\Agent\hermes-agent`。以下路径均相对此根目录，冒号后的数字为文件行号；功能描述以源码和仓库内文档交叉确认。  
> **快照**：本地 `main`，提交 `a84025ade2`（2026-11-11），工作区干净；本地 Git 状态显示落后 `origin/main` 632 个提交。因此这是该本地快照的能力清单，不保证覆盖更新版本。

## 1. 总览

Hermes 是一个**多入口、共享 Agent 核心、可插拔能力**的运行底座，不只是单一聊天 CLI。CLI、消息网关、ACP、批处理和 API 等入口复用 `AIAgent`；核心负责提示词构造、模型解析、工具分派、重试、上下文压缩及会话持久化。源码结构图和数据流见 `website\docs\developer-guide\architecture.md:11-49,142-177,194-268`；核心类入口见 `run_agent.py:214`。

| 已有能力 | 当前可确认的范围 |
|---|---|
| 对话与工具调用 | 提示词组装、多个模型 API 传输模式、工具调用循环、取消/重试、上下文压缩和提示词缓存。 |
| 工具与执行环境 | 70+ 已登记工具、约 28 个工具集（文档所列数量）；工具按工具集和可用性检查纳入会话。终端支持本地、Docker、SSH、Daytona、Modal、Vercel Sandbox，另可由插件提供后端。 |
| 会话与记忆 | SQLite 会话存储、FTS5 全文检索、会话谱系；另有跨会话记忆及可替换记忆后端。 |
| 自动化与协作 | Cron 定时 Agent 任务、子 Agent 委派、批处理/轨迹生成、生命周期钩子。 |
| 多模态与集成 | 浏览器、文件/图像、语音与 TTS/STT、MCP 外部工具、消息平台等能力；具体后端依配置、凭据和平台而异。 |

上表的工具数、会话存储和网关能力依据 `website\docs\developer-guide\architecture.md:218-250`；功能类别见 `website\docs\user-guide\features\overview.md:15-57` 和 `README.md:24-30`。计数是文档快照值：MCP/插件工具在运行时可增减，不应视作固定 API 清单。

## 2. 运行入口与集成面

- **面向使用者**：交互 CLI/TUI、消息网关、Hermes Desktop 与 Web Dashboard。一个网关可承载多个会话/配置档。`README.md:24-29`；入口与数据流：`website\docs\developer-guide\architecture.md:144-177`。
- **供外部程序集成**：ACP（stdio/JSON-RPC）、TUI Gateway JSON-RPC、OpenAI 兼容 HTTP API；也可在 Python 进程中直接使用 `run_agent.AIAgent`。API 路由与选择建议见 `website\docs\developer-guide\programmatic-integration.md:21-40,174-199,268-274`。
- **模型接入**：共享 provider 解析层与 `ProviderProfile` 注册表；模型服务可通过模型提供者插件扩展，运行时由配置、认证和 API 模式决定实际路由。见 `providers\README.md:1-25,29-55,66-75`。

## 3. 扩展机制

优先区分扩展对象，不要把每种能力都当作同一种“插件”：

| 扩展入口 | 适合承载 | 事实与边界 |
|---|---|---|
| **Skills** | 可按需加载的流程、领域知识、步骤和参考资料 | 使用渐进式披露；技能内容以 `SKILL.md` 为主。项目级技能默认不会未经信任就加载，且项目技能会经过扫描。见 `website\docs\user-guide\features\skills.md:172-192,467-514`。 |
| **Native plugins** | 新工具、钩子、斜杠命令/CLI 命令、平台适配器及各类后端 | 通过 `plugin.yaml`、`register(ctx)` 和 `PluginContext` 注册；能力清单见 `website\docs\user-guide\features\plugins.md:103-131`。 |
| **MCP** | 接入外部进程或远端服务提供的工具 | 配置 `mcp_servers` 后发现工具并注册到通常规工具表；支持 stdio 与 HTTP。可按服务器过滤工具。见 `website\docs\user-guide\features\mcp.md:29-56,311-353,747-817`。 |
| **Model provider profile** | 增加或覆盖推理模型提供商、模型发现及请求差异 | 通过 `ProviderProfile`/提供者注册表扩展，不必为每家服务复制整套运行时。见 `providers\README.md:20-25,59-75`。 |
| **Core tool / toolset** | 确需进入 Hermes 核心工具面的通用工具 | 内置工具通常在工具模块注册，并在 `toolsets.py` 纳入核心或命名工具集；新增内置工具涉及这两个扩展点。见 `website\docs\developer-guide\adding-tools.md:27-35,96-142`。 |

插件发现位置包括仓库内置、用户级 `~/.hermes/plugins/`、项目级 `.hermes/plugins/`、Python entry point，以及 Nix 配置；**项目级插件不由处理会话的 Gateway 加载**。常规第三方插件默认不执行，需启用；见 `website\docs\user-guide\features\plugins.md:145-178`。Native 插件的公开导入清单由 `hermes_cli\data\plugin_api.json` 维护，`hermes plugins validate` 检查私有导入；没有统一 `PLUGIN_API_VERSION`，兼容约定是对已文档化行为作增量演进。见 `website\docs\developer-guide\plugins\public-api.md:7-18`、`website\docs\developer-guide\plugins\index.md:235-254`。

## 4. 配置与状态入口

- 默认用户状态目录包含 `config.yaml`、`.env`、OAuth `auth.json`、`SOUL.md`、记忆、技能、Cron、会话及日志；具体目录结构：`website\docs\user-guide\configuration.md:19-31`。
- 非机密设置放 `config.yaml`，API key/token/password 放 `.env`；优先级为命令行参数、配置文件、环境变量/`.env`、内置默认值。`hermes config get/set/edit/check` 是主要管理入口，且 `config set` 会根据设置名将值写入对应文件。见 `website\docs\user-guide\configuration.md:34-67`。
- 配置样例：`cli-config.yaml.example`（含数据库、运行时、插件、模型、终端、浏览器、压缩、记忆、技能、Gateway 等章节）；这是样例而非运行时配置。完整选项文档在 `website\docs\user-guide\configuration.md`，环境变量索引在 `website\docs\reference\environment-variables.md`。
- 路径通过 `HERMES_HOME`/配置档解析，而非所有平台固定使用 `~/.hermes`；代码入口 `hermes_constants.py:206-213`。Windows 原生默认数据位置为 `%LOCALAPPDATA%\hermes`，WSL 使用 `~/.hermes`：`website\docs\user-guide\windows-native.md:8-26`。

## 5. 支持范围与明确限制

- **安装/平台支持等级**：Tier 1 为 macOS Apple Silicon、Windows 10/11、Linux/WSL2、Docker；Nix 与 Android/Termux 为 Tier 2 best effort。Windows 原生运行受支持，不要求 WSL。见 `website\docs\getting-started\platform-support.md:13-50`、`website\docs\user-guide\windows-native.md:8-15`。具体可选后端、安装包和架构仍有单独限制，以支持矩阵为准。
- **插件隔离不是默认安全沙箱**：第三方 Python 插件默认在 Hermes 进程内运行；可设置 `plugins.isolation: host` 将其移到每个配置档的独立 plugin-host 进程，但 bundled plugins 仍在进程内。可配置外部 launcher；不能据此推断默认已有 OS 级沙箱。见 `website\docs\user-guide\features\plugins.md:894-920`。
- **MCP 能力有边界**：发现默认并发上限为 4；会话的工具面通常固定，新增凭据/后端后需要重新加载、开新会话或压缩上下文才能刷新。当前会接收 prompt/resource 列表变更通知，但文档明确说尚未处理；见 `website\docs\user-guide\features\mcp.md:749-776,786`。
- **“可扩展”不等于任意内部模块均为稳定接口**：插件应使用登记的公开 API 与 `ctx`，而非依赖内部导入路径；插件实现/类别也有各自发现规则。API 清单见 `website\docs\developer-guide\plugins\public-api.md:10-18`。
- **凭据与外部依赖决定实际可用性**：存在某项工具/插件代码，不代表当前机器已配置 API 凭据、平台 token、外部服务或相应运行环境；注册层的 `check_fn`/enabled 状态会控制能力是否暴露。见 `website\docs\developer-guide\adding-tools.md:100-118`、`website\docs\user-guide\features\plugins.md:173-178`。

## 6. 已实现、实验性与未确认项

1. **可视为本快照已有**：源码中可定位到核心 Agent、工具注册表、配置/状态管理、插件加载器和 provider 注册机制；仓库文档还给出对应运行入口及配置方式。本手册的“已有能力”限于此证据范围。
2. **明确标记实验性**：MCP Events 在功能文档中标为 experimental（`website\docs\user-guide\features\mcp.md:1035-1039`）；不要把它与稳定的 stdio/HTTP 工具接入等同。
3. **规划/不确定**：没有找到可作为统一产品路线图的本地清单；未标为已实现的功能不据名称、样例或规划文档推断为可用。尤其该本地分支落后 `origin/main` 632 个提交，需在更新源码后重新核对当前状态。

## 7. 源码导航（首选证据点）

| 主题 | 本地文件 |
|---|---|
| Agent 门面与循环 | `run_agent.py:214`；循环实现见 `agent\conversation_loop.py`、`agent\turn_*.py`；架构说明 `website\docs\developer-guide\agent-loop.md` |
| 工具注册、工具集 | `tools\registry.py:592,845`；`toolsets.py:11,70`；工具分派门面 `model_tools.py` |
| 会话/状态 | `hermes_state.py:464`（`SessionDB` 门面）；`hermes_constants.py:206`（配置档 home 解析） |
| 插件 | `hermes_cli\plugins.py:266,393`（管理器/发现加载）；`hermes_cli\plugins_context.py:36`（插件上下文）；公开 API 清单 `hermes_cli\data\plugin_api.json` |
| Provider | `providers\base.py:142`（`ProviderProfile`）；`providers\README.md`；内置 provider 插件位于 `plugins\model-providers\` |
| 平台与终端后端 | 消息适配器位于 `plugins\platforms\`；终端实现位于 `tools\environments\` |
| 用户配置与说明 | `cli-config.yaml.example`；`website\docs\user-guide\configuration.md`；`website\docs\reference\environment-variables.md` |

