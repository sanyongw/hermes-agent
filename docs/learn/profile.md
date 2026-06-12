# Hermes Profile 机制

## 最短定义

Hermes profile 是一套独立的 `HERMES_HOME`，用来隔离同一份 Hermes 程序下的配置、密钥、人格、技能、记忆、会话、日志和 Gateway 状态。

它的核心不是「模型 preset」，而是通过切换 `HERMES_HOME` 来隔离一整套运行状态。

## 目录模型

默认 profile：

```text
~/.hermes/
  config.yaml
  .env
  SOUL.md
  skills/
  sessions/
  logs/
  state.db
  ...
```

命名 profile，例如 `coder`：

```text
~/.hermes/profiles/coder/
  config.yaml
  .env
  SOUL.md
  skills/
  sessions/
  logs/
  state.db
  ...
```

也就是说，profile 是 Hermes 的「用户态根目录」。代码里所有应该按用户/身份隔离的路径都最终通过 `get_hermes_home()` 解析。

## 它实际隔离什么

一个 profile 通常隔离这些东西：

- `config.yaml`：模型、provider、toolsets、gateway、memory、terminal cwd 等设置。
- `.env`：API keys、平台 token、OAuth/credential 相关密钥。
- `SOUL.md`：人格/身份设定。
- `skills/`：该 profile 安装或生成的技能。
- `sessions/` / `state.db`：会话历史、FTS 搜索、session title、rewind/branch 等状态。
- `logs/`：该 profile 自己的 agent/gateway/error 日志。
- Gateway 运行状态和平台绑定。
- Memory provider 的本地状态，取决于 provider 实现是否遵守 `get_hermes_home()`。
- 可选的 profile-local subprocess home：如果 `{HERMES_HOME}/home/` 存在，子进程可以用它来隔离 git identity、SSH、shell dotfiles 等。

## 它不等于什么

Profile 不是这些东西：

- 不是单纯的模型配置。模型只是 profile 配置的一部分。
- 不是 Python virtualenv。Hermes 代码、安装版本、依赖通常是共享的。
- 不是自动创建一个独立 git workspace。工作目录仍由当前 cwd 或 `terminal.cwd` 决定。
- 不是 OS 用户隔离。它是应用层数据隔离，不是安全沙箱。
- 不是 provider profile。代码里还有 `ProviderProfile` 这个概念，指 OpenAI/Anthropic/Nous/OpenRouter 这类模型供应商的元数据，和用户 profile 不是一回事。

## 它怎么生效

实现上，profile 的关键机制是：尽早设置 `HERMES_HOME`。

`hermes_cli/main.py` 启动时会很早预解析 `--profile/-p`，在大量模块 import 之前把 `HERMES_HOME` 设置到目标 profile 目录。这样后续所有 `get_hermes_home()`、`get_config_path()`、`get_env_path()`、日志、session DB 都会自然落到这个 profile 下面。

选择顺序大概是：

```text
hermes -p coder chat
  -> 显式指定 coder，优先级最高

如果没有 -p：
  -> 如果 HERMES_HOME 已经指向 profiles/<name>，就信任它
  -> 否则读取 ~/.hermes/active_profile
  -> 如果没有 active_profile，就用 default: ~/.hermes
```

`active_profile` 是「粘性默认 profile」。执行：

```bash
hermes profile use coder
```

之后，不带 `-p` 的 `hermes chat`、`hermes gateway` 等也会默认进入 `coder` profile。

## 常用命令

查看当前 active profile：

```bash
hermes profile
```

列出所有 profile：

```bash
hermes profile list
```

创建一个名为 `coder` 的 profile：

```bash
hermes profile create coder
```

从当前 profile 克隆配置、密钥、人格和技能：

```bash
hermes profile create work --clone
```

把 `coder` 设成默认 active profile：

```bash
hermes profile use coder
```

只对这次命令使用 `coder`：

```bash
hermes -p coder chat
hermes -p coder gateway start
```

创建 profile 时还可能生成 wrapper alias，例如 `coder` 命令等价于 `hermes -p coder ...`，所以创建后可能会看到：

```text
coder setup
coder chat
coder gateway start
```

## 为什么需要 profile

典型场景：

```text
personal profile
  私人模型 key、私人聊天历史、私人记忆、私人 Telegram bot

work profile
  公司模型 key、公司项目 cwd、公司相关记忆、不同工具权限

client-a profile
  客户 A 的 Gateway、Slack/Telegram 绑定、项目上下文、会话历史

research profile
  更激进的 toolsets、批处理/trajectory 配置、实验模型
```

这样做的好处是：不需要反复改 `.env` 和 `config.yaml`，也不会把不同身份的 memory/session 混在一起。

## 架构位置

可以把 profile 看成 Hermes 的「租户边界」：

```text
Hermes 安装/代码：共享
  /path/to/hermes-agent

Profile A 数据根：隔离
  ~/.hermes/

Profile B 数据根：隔离
  ~/.hermes/profiles/coder/

Profile C 数据根：隔离
  ~/.hermes/profiles/work/
```

入口层、Agent、tools、plugins、gateway 都不应该硬编码 `~/.hermes`，而应该通过 `get_hermes_home()` 找当前 profile 的根目录。

## 易混淆点

Hermes 里有两种 profile 语境：

1. 用户/运行 profile  
   也就是 `hermes profile create/use/list` 这套。它切换 `HERMES_HOME`，隔离配置和状态。

2. `ProviderProfile`  
   这是模型供应商的注册资料，比如 provider 名称、别名、auth_type、env vars、base_url 等。它用于模型/provider 配置 UI 和 runtime provider resolution，不是用户数据隔离。

配置代码里如果出现「provider profiles」，通常指第二种，不是 `hermes profile` 的用户 profile。
