# 一次对话的数据链路

下面用一个具体例子说明 Hermes 一次对话的数据链路。

假设用户在 TUI 里输入：

```text
帮我查看 README 里怎么启动项目，然后告诉我需要运行什么命令
```

当前 profile 是 `coder`，所以启动时已经解析成：

```text
HERMES_HOME=~/.hermes/profiles/coder
```

这意味着本轮对话读写的配置、密钥、会话 DB、日志、skills、memory 都落在 `~/.hermes/profiles/coder/` 下。

## 1. 用户输入进入前端

TUI 场景下，屏幕由 Node/Ink 负责。用户在 composer 里输入文本并回车。

前端不会直接调用模型，也不会自己执行工具。它通过 stdio JSON-RPC 把请求发给 Python 后端 `tui_gateway/server.py`，方法名类似：

```json
{
  "method": "prompt.submit",
  "params": {
    "session_id": "...",
    "text": "帮我查看 README 里怎么启动项目，然后告诉我需要运行什么命令"
  }
}
```

如果是经典 CLI，入口会在 `cli.py`；如果是 Telegram/Slack/Discord，入口会在 `gateway/run.py`；如果是编辑器 ACP，入口会在 `acp_adapter/server.py`。但这些入口最后都会走同一个核心：

```python
agent.run_conversation(...)
```

## 2. TUI Gateway 找到当前 session

`tui_gateway/server.py` 收到 `prompt.submit` 后，先根据 `session_id` 找到内存里的 session 对象。这个 session 里通常有：

```python
{
    "agent": AIAgent(...),
    "session_key": "...",
    "history": [...],
    "history_lock": ...,
    "running": False,
    "cwd": "...",
    "attached_images": [],
}
```

然后它会做几件事：

```text
检查 session 是否存在
检查 session 是否 busy
记录 running=True
记录 inflight_turn
确保 SessionDB 里有这一行 session
如果 agent 还没创建，就异步构建 agent
启动后台线程真正跑本轮对话
```

这一步的关键是：TUI 的 UI 线程不会被模型调用阻塞。Python 后端会立刻返回：

```json
{"status": "streaming"}
```

后续再通过事件持续推送：

```text
message.start
message.delta
tool.start
tool.complete
message.complete
session.info
```

## 3. 构建或复用 AIAgent

如果当前 session 已经有 `AIAgent`，就复用；否则 `_make_agent()` 会创建一个新的 `AIAgent`。

创建 agent 时会注入这些运行时信息：

```python
AIAgent(
    model=...,
    provider=...,
    base_url=...,
    api_key=...,
    api_mode=...,
    max_iterations=...,
    enabled_toolsets=...,
    platform="tui",
    session_id=session_id,
    session_db=SessionDB(...),
    quiet_mode=True,
    stream_delta_callback=...,
    tool_start_callback=...,
    tool_complete_callback=...,
    clarify_callback=...,
)
```

这些参数来自几个地方：

```text
当前 profile 的 config.yaml
当前 profile 的 .env
命令行参数 / 环境变量
当前 session 状态
TUI gateway 注册的各种 callback
```

例如：

```text
~/.hermes/profiles/coder/config.yaml
~/.hermes/profiles/coder/.env
~/.hermes/profiles/coder/state.db
~/.hermes/profiles/coder/logs/agent.log
```

## 4. Agent 初始化工具列表

`AIAgent` 初始化时会解析 toolsets。默认可能是：

```yaml
toolsets:
  - hermes-cli
```

然后进入工具系统：

```text
toolsets.py
  -> resolve_toolset("hermes-cli")
  -> 得到工具名列表

model_tools.get_tool_definitions(...)
  -> 从 tools.registry 取 schema
  -> 运行 check_fn 过滤不可用工具
  -> 动态改写部分 schema
  -> 返回 OpenAI function/tool schema
```

最终 agent 会得到一组传给模型的工具定义，例如：

```text
read_file
search_files
terminal
web_search
web_extract
patch
todo
memory
delegate_task
session_search
```

模型看到的不是所有源码里的工具，而是「当前 session 的 toolset + 当前环境可用性」过滤后的工具。

## 5. TUI 后端调用 run_conversation

后台线程准备好后，会调用：

```python
result = agent.run_conversation(
    user_message="帮我查看 README 里怎么启动项目，然后告诉我需要运行什么命令",
    conversation_history=session["history"],
    stream_callback=_stream,
    task_id=session["session_key"],
)
```

其中：

```text
user_message
  当前用户输入

conversation_history
  当前 session 历史消息，通常来自内存；resume 时来自 SessionDB

stream_callback
  模型输出文本 delta 时回调，用于发 message.delta 给 TUI

task_id
  工具隔离用，比如 terminal/browser/process/session 绑定
```

## 6. run_conversation 准备上下文

进入 `agent/conversation_loop.py` 后，会先组装这轮要给模型的消息。

内部概念上会形成：

```python
messages = [
    {"role": "user", "content": "之前的用户消息"},
    {"role": "assistant", "content": "之前的回答"},
    {"role": "user", "content": "帮我查看 README 里怎么启动项目，然后告诉我需要运行什么命令"},
]
```

构造 API 请求前，还会额外做一批处理：

```text
恢复或构建 system prompt
注入 SOUL.md / AGENTS.md / 项目规则
注入当前 platform 提示
注入 memory prefetch 结果
注入 plugin pre_llm_call 上下文
修复坏掉的 tool_call arguments
修复 role alternation
清理 provider 不支持的字段
应用 prompt caching 标记
估算 token 数
检查是否需要压缩上下文
```

最终发给模型的 `api_messages` 形状类似：

```python
[
    {
        "role": "system",
        "content": "...Hermes system prompt + 当前项目规则 + 工具使用规范..."
    },
    {
        "role": "user",
        "content": "之前的用户消息"
    },
    {
        "role": "assistant",
        "content": "之前的回答"
    },
    {
        "role": "user",
        "content": "帮我查看 README 里怎么启动项目，然后告诉我需要运行什么命令\n\n<可能追加的 memory/plugin context>"
    }
]
```

很多 ephemeral context 只加到 API 请求里，不直接写回历史。这样可以减少污染 session 历史，也保持 system prompt 前缀稳定，方便 prompt cache 命中。

## 7. 第一次模型调用

Agent 根据 provider/api_mode 选择 transport。

```text
OpenAI-compatible provider
  -> chat_completions transport

Anthropic
  -> anthropic_messages adapter

Bedrock
  -> bedrock_converse adapter

OpenAI Codex / GPT-5 Responses
  -> codex_responses adapter
```

请求大致包含：

```python
client.chat.completions.create(
    model=model,
    messages=api_messages,
    tools=agent.tools,
)
```

模型看到用户问题和可用工具后，很可能不会直接回答，而是返回 tool call：

```json
{
  "role": "assistant",
  "tool_calls": [
    {
      "id": "call_1",
      "type": "function",
      "function": {
        "name": "read_file",
        "arguments": "{\"path\":\"README.md\"}"
      }
    }
  ]
}
```

## 8. Agent 执行工具调用

`conversation_loop` 发现 assistant message 有 `tool_calls`，于是走：

```text
AIAgent._execute_tool_calls(...)
  -> agent.tool_executor
  -> AIAgent._invoke_tool(...)
  -> model_tools.handle_function_call(...)
  -> tools.registry.dispatch(...)
  -> read_file handler
```

工具执行前后会触发 callback 和 hook：

```text
tool.start
pre_tool_call hooks
tool handler
post_tool_call hooks
tool.complete
```

TUI 会收到类似事件：

```json
{
  "event": "tool.start",
  "payload": {
    "name": "read_file",
    "args": {"path": "README.md"}
  }
}
```

工具返回结果会被包装成 tool role message，追加到 `messages`：

```python
{
    "role": "tool",
    "tool_call_id": "call_1",
    "name": "read_file",
    "content": "...README.md 的内容或摘要..."
}
```

## 9. 第二次模型调用

现在 messages 变成：

```python
[
    {"role": "user", "content": "帮我查看 README..."},
    {
        "role": "assistant",
        "tool_calls": [
            {"function": {"name": "read_file", "arguments": "{\"path\":\"README.md\"}"}}
        ]
    },
    {
        "role": "tool",
        "tool_call_id": "call_1",
        "content": "...README.md 内容..."
    }
]
```

Agent 再次构造 API 请求，把工具结果带给模型。

这次模型可能继续调用工具，比如：

```json
{
  "function": {
    "name": "search_files",
    "arguments": "{\"query\":\"npm run dev|uv pip install|hermes\"}"
  }
}
```

也可能已经足够回答，就返回普通文本：

```text
README 里的启动方式是：

1. 安装依赖...
2. 运行 ...
```

如果模型继续调工具，Hermes 就继续：

```text
模型调用 -> tool_calls -> 执行工具 -> 追加 tool result -> 再调用模型
```

直到出现最终文本响应、用户中断、达到 `max_iterations`、预算耗尽或发生不可恢复错误。

## 10. 流式输出回到 TUI

如果 provider 支持 streaming，模型生成最终回答时，每个 delta 会经过：

```text
agent stream_delta_callback
  -> tui_gateway _stream()
  -> _emit("message.delta", ...)
  -> Node/Ink 前端渲染
```

例如后端发送：

```json
{
  "event": "message.delta",
  "payload": {
    "text": "README 里的启动方式是："
  }
}
```

TUI 前端把这些 delta 拼成屏幕上的 assistant 消息。

如果同时有 reasoning/tool progress，也会分别走：

```text
reasoning_callback -> reasoning.delta / thinking UI
tool_start_callback -> tool.start
tool_complete_callback -> tool.complete
status_callback -> status.update
```

## 11. 写入会话历史

本轮完成后，`run_conversation()` 返回：

```python
{
    "final_response": "README 里的启动方式是：...",
    "messages": [
        {"role": "user", "content": "..."},
        {"role": "assistant", "tool_calls": [...]},
        {"role": "tool", "content": "..."},
        {"role": "assistant", "content": "README 里的启动方式是：..."}
    ],
    "api_calls": 2,
}
```

TUI gateway 拿到结果后，会更新内存 session：

```python
session["history"] = result["messages"]
session["history_version"] += 1
session["running"] = False
```

同时 Agent 内部也会把消息 flush 到 `SessionDB`，也就是当前 profile 的 SQLite：

```text
~/.hermes/profiles/coder/state.db
```

消息表里会保存：

```text
role
content
tool_name
tool_calls
tool_call_id
reasoning
reasoning_content
finish_reason
token_count
platform_message_id
timestamp
```

这就是为什么后续可以：

```text
/resume
/session list
/session search
/undo
/branch
dashboard 查看历史
跨入口恢复同一 session
```

## 12. 记忆和技能后处理

对话结束后，Agent 还可能触发后处理：

```text
memory sync
plugin on_session_end hooks
background review
skill curator / skill nudge
usage/cost tracking
trajectory 保存
context compression 状态更新
```

这些不是每轮都一定发生，取决于配置和 provider/tool 使用情况。

例如 memory provider 开启时，本轮对话可能会同步到外部 memory backend 或本地 memory 状态。插件也可以通过 hook 观察本轮结果。

## 13. 最终响应完成

最后 TUI gateway 发出完成事件：

```json
{
  "event": "message.complete",
  "payload": {
    "text": "README 里的启动方式是：..."
  }
}
```

前端停止 loading，session 回到 idle 状态，可以接受下一条消息。

## 压缩成一张图

```text
用户输入
  |
  v
TUI/CLI/Gateway/ACP 入口
  |
  | 解析 session、profile、cwd、slash command、附件
  v
AIAgent.run_conversation()
  |
  | 构造 messages
  | 构造 system prompt
  | 注入 memory / skills / project rules / plugin context
  | 加载 tool schemas
  v
Provider transport
  |
  | 第一次 LLM call
  v
模型返回 tool_calls?
  |
  +-- 是:
  |     v
  |   tool_executor
  |     v
  |   model_tools.handle_function_call()
  |     v
  |   tools.registry.dispatch()
  |     v
  |   工具结果追加为 role=tool
  |     v
  |   回到 Provider transport 再调模型
  |
  +-- 否:
        v
      final assistant response
        |
        v
SessionDB 写入 state.db
        |
        v
入口层回传 message.delta / message.complete
        |
        v
用户看到回答
```

## Gateway 场景的差异

如果用户从 Telegram 发消息，前半段不同：

```text
Telegram update
  -> TelegramPlatformAdapter
  -> MessageEvent
  -> GatewayRunner._handle_message()
  -> 找到/创建 SessionEntry
  -> 组装 platform context
  -> 创建 AIAgent(platform="telegram")
  -> agent.run_conversation()
```

中间的 Agent/tool/model/session DB 链路基本一样。

差异主要在入口和回传：

```text
TUI 回传：JSON-RPC event -> Ink UI
Gateway 回传：adapter.send() -> Telegram/Slack/Discord API
CLI 回传：prompt_toolkit/Rich 输出
ACP 回传：ACP session_update -> 编辑器 UI
```

## Profile 在这一轮里影响哪里

如果当前是 `coder` profile，那么这次对话至少影响这些路径：

```text
读取：
  ~/.hermes/profiles/coder/config.yaml
  ~/.hermes/profiles/coder/.env
  ~/.hermes/profiles/coder/SOUL.md
  ~/.hermes/profiles/coder/skills/
  当前项目 AGENTS.md

写入：
  ~/.hermes/profiles/coder/state.db
  ~/.hermes/profiles/coder/logs/agent.log
  ~/.hermes/profiles/coder/logs/errors.log
  ~/.hermes/profiles/coder/sessions/
  memory/plugin/provider 自己的 profile-local 状态
```

所以同样一句话，在不同 profile 下可能得到不同结果，因为：

```text
模型不同
API key 不同
toolsets 不同
SOUL.md 不同
skills 不同
memory 不同
历史会话不同
gateway/platform 配置不同
```

## 关键理解

一次 Hermes 对话不是「用户文本直接发给模型」。它更像一个带状态机的循环：

```text
用户意图
  -> 上下文组装
  -> 模型决策
  -> 工具执行
  -> 工具结果回灌
  -> 模型再决策
  -> 最终响应
  -> 会话/记忆/日志持久化
```

`AIAgent.run_conversation()` 是这个循环的中心；各入口只负责把不同来源的用户消息规范化进来，并把同一个 Agent loop 的事件翻译回各自的 UI 或平台。
