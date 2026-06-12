# Hermes Agent 系统架构分析

## 总体结论

Hermes Agent 的架构可以概括为：

```text
多入口客户端/平台
  CLI / TUI / Desktop / Dashboard / Gateway / ACP / Batch
        |
        v
统一 Agent 内核：AIAgent
        |
        +-- Provider/Transport 层：OpenAI-compatible / Anthropic / Bedrock / Codex Responses / ACP 等
        +-- Tool 层：toolsets + registry + model_tools + tool executor
        +-- Context 层：system prompt / memory / skills / compression / context engine
        +-- Persistence 层：SessionDB(SQLite + FTS5) / logs / config / profile home
        +-- Extension 层：plugins / model-provider plugins / memory providers / MCP tools
```

它不是一个单纯 CLI 工具，而是一个「多外壳 + 单 Agent 内核 + 动态工具生态 + 持久化会话中枢」的系统。当前代码还保留历史大文件特征，但核心能力已经在逐步模块化：`run_agent.py` 现在更多是兼容门面，真正的初始化、主循环、工具执行分别下沉到 `agent/agent_init.py`、`agent/conversation_loop.py`、`agent/tool_executor.py`。

## 核心分层

### 1. 入口层

`hermes_cli/main.py` 是命令行总入口，负责解析命令并分发到聊天、TUI、Gateway、Dashboard、Desktop、更新、配置等子命令。

关键职责：

- `cmd_chat()` 决定启动经典 CLI 还是 TUI。
- `_launch_tui()` 通过环境变量把模型、provider、toolsets、skills、工作目录等运行时参数传给 Node/Ink TUI 进程。
- `cmd_gateway()`、`cmd_dashboard()`、`cmd_gui()` 等子命令启动不同外壳。

### 2. 客户端/平台外壳层

不同入口负责把外部交互规范化成 Hermes 内部的 session + message + callback 模型。

经典 CLI 在 `cli.py` 中创建 `AIAgent`，把模型、provider、toolsets、session DB、stream 回调、审批/clarify 回调等注入进去。

TUI 是 Node Ink 前端 + Python JSON-RPC 后端。Python 侧 `tui_gateway/server.py` 的 `_make_agent()` 创建 `AIAgent`，`prompt.submit` 接收前端输入，然后在线程里调用 `agent.run_conversation()` 并通过 `message.delta` 等事件回传流式输出。

Gateway 是异步消息平台运行器。各平台 adapter 统一成 `MessageEvent`，`GatewayRunner` 做授权、session 绑定、排队/中断、媒体处理、tool progress 投递，最终仍然调用 `agent.run_conversation()`。

ACP 适配器把 VS Code/Zed/JetBrains 这类编辑器协议映射到同一个 Agent loop，处理 slash command、排队、审批、tool progress 和流式消息，核心调用点也是 `agent.run_conversation()`。

### 3. Agent 内核层

`AIAgent` 是稳定的公共 API，但现在是门面对象。

- 构造函数转发到 `agent.agent_init.init_agent()`。
- 主循环转发到 `agent.conversation_loop.run_conversation()`。
- 工具执行转发到 `agent.tool_executor`。
- 上下文压缩转发到 `agent.conversation_compression`。

初始化阶段会解析 provider/api_mode、模型、fallback、toolsets、session id、memory、checkpoint、context compressor、callbacks 等。`agent_init.py` 会根据 provider/base_url 自动选择 `chat_completions`、`codex_responses`、`anthropic_messages`、`bedrock_converse` 等模式。

主循环在 `agent/conversation_loop.py`。它围绕 `max_iterations` 和共享 `IterationBudget` 运行，每轮准备 API messages，修复消息序列，注入 memory/plugin context，应用 prompt cache，调用模型，解析 finish reason 和 tool calls，执行工具，然后继续下一轮或产出最终响应。

### 4. 模型/Transport 层

模型后端不是直接散落在各入口，而是由 Agent 根据 `api_mode`/provider 走不同 transport/adapter。

代表性模块：

- `agent/transports/chat_completions.py`
- `agent/transports/anthropic.py`
- `agent/transports/bedrock.py`
- `agent/transports/codex.py`
- `agent/anthropic_adapter.py`
- `agent/bedrock_adapter.py`
- `agent/codex_responses_adapter.py`
- `agent/gemini_native_adapter.py`

Provider 资料通过 `providers/__init__.py` 懒发现，导入 `plugins/model-providers/<name>/__init__.py`，这些模块通过 `register_provider()` 注册 profile。用户 provider 插件可以覆盖内置 provider。

这层的设计意图是：上层统一使用 OpenAI-style message/tool call 形状，底层 adapter 负责把不同 provider 的 API 差异折叠回来。

### 5. 工具系统

工具系统有三层：

```text
tools/*.py 自注册
        |
        v
tools.registry.ToolRegistry
        |
        v
model_tools.get_tool_definitions / handle_function_call
        |
        v
AIAgent tool execution
```

`tools/registry.py` 会扫描 `tools/*.py` 中顶层 `registry.register(...)` 的模块并导入，让工具自注册 schema、handler、toolset、check_fn。

`toolsets.py` 定义默认工具集合，`_HERMES_CORE_TOOLS` 是 CLI 和各消息平台共享的基础工具面，包括 web、terminal、file、vision、browser、skills、todo、memory、delegate、cron、send_message 等。

`model_tools.get_tool_definitions()` 根据 enabled/disabled toolsets 生成传给模型的 OpenAI function schema，并做缓存、check_fn 过滤、动态 schema 重写和 tool_search 折叠。

真正执行工具时走 `model_tools.handle_function_call()`，它会做参数类型矫正、tool_search bridge 权限收敛、pre/post hook/middleware，然后调 registry dispatch。

工具执行本身在 Agent 内部分为顺序和并发路径。`AIAgent._execute_tool_calls()` 会判断一批 tool calls 是否可并发，读类工具和无路径冲突的文件类工具可以并行，否则走顺序执行。

### 6. 会话与持久化

`SessionDB` 是系统的会话中枢，使用 SQLite，带 schema reconcile、WAL fallback、FTS5、trigram FTS、软删除/rewind、session title、handoff、Telegram topic binding 等。

架构上，session DB 不只是聊天历史存储，还承担了这些职责：

- 恢复/继续会话
- 跨入口 session 浏览
- `session_search` / recall
- prompt cache 前缀稳定性辅助
- undo / rewind / branch
- gateway 平台绑定和 handoff
- dashboard session 管理

消息写入支持 multimodal content JSON 编码、tool_calls、reasoning、codex reasoning items、platform message id 等字段。

### 7. 配置与 profile

配置主源是 `~/.hermes/config.yaml`，默认值在 `hermes_cli/config.py::DEFAULT_CONFIG`，包含 model、toolsets、agent、terminal、memory、gateway、plugins、cron 等。

`load_config()` 会深合并用户配置、展开 env、做缓存，并支持 profile 切换时按 config path 分隔缓存。

一个重要架构细节：配置加载路径不完全统一。CLI 用自己的 `load_cli_config()`，很多子命令用 `hermes_cli.config.load_config()`，Gateway 部分路径会直接读 YAML/raw runtime config。新增配置时必须确认入口路径，否则容易出现「CLI 生效但 Gateway 不生效」这类问题。

### 8. 插件系统

插件层分几套：

- 通用插件：`hermes_cli/plugins.py`
- model-provider 插件：`providers/__init__.py` + `plugins/model-providers/`
- memory provider 插件：`plugins/memory/`
- context engine / image_gen / TTS / browser / web search 等 provider registry
- MCP 动态工具：通过工具注册系统并入 toolsets

通用插件由 `PluginManager` 扫描 bundled/user/project/entry point 插件，加载模块并调用 `register(ctx)`。插件可注册 tools、hooks、middleware、commands、context engine、platform、skills、auxiliary tasks 等。

这个插件层是 Hermes 的主要扩展边界。原则上新增能力应该优先走插件和 registry，而不是改 `run_agent.py`、`cli.py`、`gateway/run.py` 这些核心文件。

## 关键运行链路

普通 CLI/TUI/Gateway 一轮对话大致是：

```text
用户输入
  -> 入口层解析 slash command / session / cwd / media / platform context
  -> 创建或复用 AIAgent
  -> AIAgent.run_conversation()
  -> 构造 system prompt + history + ephemeral context
  -> 生成 API messages + tool schemas
  -> 调 provider transport
  -> 若模型返回 tool_calls：
       -> tool executor
       -> model_tools.handle_function_call()
       -> tools.registry.dispatch()
       -> tool result 追加到 messages
       -> 下一轮 API call
     否则：
       -> final_response
  -> 持久化到 SessionDB
  -> 外壳层把结果/stream/tool progress 发回用户
```

## 架构优点

- 统一内核复用充分：CLI、TUI、Gateway、ACP、Batch 都复用 `AIAgent.run_conversation()`，避免每个入口重写 agent loop。
- 工具扩展边界清晰：工具自注册、toolset 控制暴露面、registry 执行，插件/MCP 可以自然并入。
- 多 provider 适配能力强：通过 provider profile + transport/adapter 隔离 API 差异，上层尽量维持统一 message/tool call 模型。
- 会话能力强：SQLite + FTS5 让历史搜索、resume、undo、branch、dashboard 浏览和 gateway continuity 有共同基础。
- 面向长运行进程做了工程处理：tool schema 缓存、check_fn TTL、config cache、thread/session context、interrupt、gateway queue/drain 都是长生命周期 agent 必需的机制。

## 主要风险点

- 历史大文件仍是复杂度中心：`cli.py`、`gateway/run.py`、`tui_gateway/server.py`、`run_agent.py` 仍非常大，入口逻辑、业务策略、错误恢复、平台细节交织。
- 配置路径不完全一致：CLI、Gateway、TUI、web dashboard 的配置读取路径不同，新增配置或改默认值时容易漏入口。
- 全局状态较多：环境变量、ContextVar、thread-local callback、global plugin manager、registry generation、session key 等共同参与运行。
- 工具权限面大：默认核心工具包含 terminal/file/browser/delegate/send_message 等高权限能力，安全主要依赖 toolsets、approval、platform policy、check_fn、middleware 和 prompt 约束。
- 插件发现有多套机制：通用插件、model-provider、memory-provider、MCP 动态工具各自发现，扩展能力强，但定位「为什么某插件没加载」会比较分散。
- 不同外壳重复了一些会话/中断/排队语义：CLI、TUI、Gateway、ACP 都要处理 busy、interrupt、slash command、history mutation，这些语义没有完全集中到一个公共 session runtime。

## 后续演进建议

如果后续做架构演进，优先考虑三件事：

1. 把 CLI/TUI/Gateway/ACP 共有的 turn orchestration 抽成一个 `SessionRuntime` 或 `TurnRunner`，统一 busy、interrupt、history_version、callbacks、approval context 清理。
2. 把配置读取做成明确的 typed runtime config 快照，减少入口各自读 YAML/env 的差异。
3. 继续把 `gateway/run.py` 和 `tui_gateway/server.py` 中的平台无关逻辑下沉到共享模块，只把协议适配留在外壳层。
