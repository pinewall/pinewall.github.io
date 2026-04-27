---
layout: default
title: Nanobot 开源 AI 智能体基本原理
---

# Nanobot 开源 AI 智能体基本原理

Nanobot 是一个轻量级**开源 AI 智能体（AI Agent）**项目，它可以连接到飞书、Telegram 等即时通讯渠道，提供带工具调用和长期记忆能力的 AI 助手服务。整个项目仅约 **4000 行 Python 代码**，设计简洁清晰，易于理解和二次开发。

项目地址：[https://github.com/HKUDS/nanobot](https://github.com/HKUDS/nanobot)

## 什么是 Nanobot

Nanobot 是一个**自托管 AI 网关代理**，它的核心定位：

- 🤖 **带工具使用的 AI 助手**：支持读写文件、执行命令、搜索、网页抓取等工具
- 🧠 **两层记忆架构**：支持长期记忆的自动压缩，解决上下文窗口限制
- 🔌 **多渠道支持**：可连接飞书、Telegram 等 IM 平台，随时随地对话
- 🔄 **ReAct 执行循环**：标准的思考-行动-观察循环，支持多轮工具调用
- 🔑 **多提供商支持**：支持 30+ LLM 提供商（Anthropic、OpenAI、OpenRouter、Ollama 等）
- 🧩 **极简架构**：整个项目约 4000 行代码，易于阅读和修改

## 整体架构

Nanobot 采用**分层模块化设计**，架构清晰：

```
入口层 (CLI)
  ↓
配置层 (Pydantic  schema)
  ↓
消息总线 (async queue)
  ↓
核心引擎 (AgentLoop + ReAct)
  ↓
渠道层 (飞书/Telegram/...)
  ↓
用户端
```

完整的模块结构：

```
nanobot/
├── cli/commands.py        # CLI 入口，三个核心命令
├── config/
│   ├── schema.py          # Pydantic 配置定义
│   └── loader.py          # 配置加载
├── providers/
│   ├── registry.py       # Provider 注册表
│   └── ...               # 各提供商适配
├── agent/
│   ├── loop.py           # 主循环 (584 行)
│   ├── runner.py         # ReAct 执行器 (232 行)
│   ├── context.py        # Prompt 构建 (200 行)
│   ├── memory.py         # 记忆系统 (核心)
│   ├── skills.py         # 技能系统
│   └── tools/            # 工具实现
│       ├── registry.py
│       ├── filesystem.py
│       ├── shell.py
│       ├── web.py
│       └── ...
├── session/
│   └── manager.py        # 会话管理
├── bus/
│   ├── queue.py          # 消息总线 (44 行)
│   └── events.py         # 事件数据结构
├── channels/
│   ├── base.py           # 渠道基类
│   ├── manager.py        # 渠道管理 (264 行)
│   ├── feishu.py         # 飞书适配
│   └── telegram.py       # Telegram 适配
├── cron/                 # 定时任务
└── heartbeat/            # 心跳服务
```

## 核心入口命令

在 `pyproject.toml` 中注册了三个入口命令：

| 命令 | 用途 |
|------|------|
| `nanobot gateway` | 启动网关模式（连接飞书/Telegram） |
| `nanobot agent` | 本地交互调试，不走外部渠道 |
| `nanobot onboard` | 初始化工作区和配置 |

启动 `gateway` 时，会创建所有核心组件：

```python
gateway()
  ├── MessageBus         (消息总线，异步队列)
  ├── LLMProvider        (LLM 提供商适配器)
  ├── SessionManager     (会话管理器)
  ├── CronService        (定时任务服务)
  ├── AgentLoop          (核心引擎主循环)
  ├── ChannelManager     (渠道管理器)
  └── HeartbeatService   (心跳服务)
```

## 核心引擎：ReAct 循环

这是整个系统的心脏，处理流程如下：

```
用户消息 → AgentLoop.run()
    │
    ├── 斜杠命令？ → CommandRouter 处理
    │
    ▼
_process_message()
    │
    ├── SessionManager.get_or_create()   # 获取或新建会话
    ├── MemoryConsolidator.maybe_consolidate()  # 检查是否需要压缩记忆
    ├── ContextBuilder.build_messages()  # 构建完整 prompt
    │   ├── 系统提示 = 身份 + 引导文件 + 记忆 + 技能
    │   ├── 历史消息 = 只包含未压缩的最近对话
    │   └── 当前用户消息 + 运行时上下文
    │
    ▼
_run_agent_loop()  → AgentRunner.run()
    │
    ├── FOR 最多 40 次迭代:
    │   │
    │   ├─ provider.chat_with_retry()    # 调用 LLM
    │   │
    │   ├─ 如果有工具调用:
    │   │   ├─ ToolRegistry.execute()    # 执行工具
    │   │   ├─ 添加工具结果到消息列表
    │   │   └─ 继续循环
    │   │
    │   └─ 如果没有工具调用:
    │       └─ 返回最终回复
    │
    ├── 保存本轮消息到 Session
    └── 构造 OutboundMessage → 发回消息总线 → 渠道发送给用户
```

**核心特点**：
- 支持**并发工具执行**：多个工具调用可以并行运行
- 最大迭代次数默认为 40，防止无限循环
- 完全符合标准 ReAct 模式：Thinking → Acting → Observing → Repeat

## 工具系统

Nanobot 内置了多种工具，LLM 可以按需使用：

| 工具 | 用途 |
|------|------|
| `read_file` | 读取本地文件 |
| `write_file` | 写入本地文件 |
| `edit_file` | 编辑现有文件 |
| `list_dir` | 列出目录内容 |
| `exec` | 执行 Shell 命令 |
| `web_search` | 网络搜索 |
| `web_fetch` | 抓取网页内容 |
| `message` | 发送消息到渠道 |
| `spawn` | 启动子代理 |
| `cron` | 定时任务管理 |

每个工具都继承 `Tool` 基类，实现 `execute()` 方法，并以 OpenAI function calling 格式暴露给 LLM。

## 记忆系统（核心设计）

Nanobot 采用**两层记忆架构**，灵感来源于人脑：

| 层级 | 存储文件 | 用途 | 特点 |
|------|----------|------|------|
| **长期记忆** | `memory/MEMORY.md` | 存储关键事实、偏好、项目上下文 | LLM 可读，持续更新 |
| **历史日志** | `memory/HISTORY.md` | 对话摘要时间线 | grep 可搜索，append-only |

### 记忆压缩流程

当会话 token 数超过阈值时，自动触发压缩：

```
1. 估算当前 prompt token 数
2. 如果超过阈值 (context_window - max_tokens - buffer):
   a. 在 user turn 边界选择切割点（保持对话合法性）
   b. 调用 LLM 压缩旧对话 → 生成新的长期记忆和历史条目
   c. 写入 MEMORY.md 和 HISTORY.md
   d. 更新 session.last_consolidated 指针
3. 下次构建上下文时：
   - MEMORY.md 注入到 system prompt
   - 只把未压缩的消息作为历史
   - 总 token 数回到限制以内
```

### 压缩决策逻辑

```python
budget = context_window_tokens - max_completion_tokens - safety_buffer
# 默认: 65536 - 8192 - 1024 = 56320 tokens

if estimated_tokens > budget:
    触发压缩
```

### 关键设计亮点

**1. 在 user turn 边界切割**
```python
# 只在 user 消息处切割
for idx in ...:
    if message.role == "user":
        选择此处为边界
```
- 避免切割 `assistant → tool → tool_result` 完整链
- 保证压缩后消息列表仍然合法

**2. 渐进式压缩**
- 每次只压缩超出预算的部分
- 单次最多压缩 5 轮，避免耗时过长

**3. 容错降级**
- LLM 压缩失败连续 3 次 → 直接原始归档
- 不阻塞主流程，保证可用性

**4. Append-only 设计**
- Session 消息从不删除，只移动 `last_consolidated` 指针
- 对 LLM 缓存友好，不破坏已缓存的前缀
- 数据完整，可随时重建

**5. 后台异步压缩**
- 消息处理完成后在后台触发再次检查压缩
- 不阻塞用户交互，提升体验

## 消息总线设计

消息总线是渠道和核心引擎之间的**解耦层**，只用 44 行代码实现：

```
渠道A ──→ 入队队列 ──→ AgentLoop
渠道B ──↗                │
                         ▼
渠道A ←── 出队队列 ←── AgentLoop
渠道B ←──↗
```

只有两个 `asyncio.Queue`：
- `inbound`: 渠道 → Agent
- `outbound`: Agent → 渠道

## 配置与提供商自动匹配

配置使用 Pydantic 定义，结构清晰：

```python
Config
├── agents: AgentsConfig          # 模型参数（temperature, max_tokens...）
├── channels: ChannelsConfig      # 渠道配置
├── providers: ProvidersConfig    # API 密钥（30+ 提供商）
├── gateway: GatewayConfig        # 网关端口、心跳
└── tools: ToolsConfig            # 工具配置（web, exec, MCP...）
```

提供商自动匹配逻辑：
1. 如果配置显式指定 `provider` → 直接使用
2. 否则，模型名前缀匹配（`anthropic/claude-3` → Anthropic）
3. 否则，关键词匹配（包含 `claude` → Anthropic）
4. fallback 到本地（Ollama）或网关（OpenRouter）

**添加新 provider 只需两步**：
1. 在 `registry.py` 添加一个 `ProviderSpec`
2. 在 `schema.py` 的 `ProvidersConfig` 添加一个字段

## 完整数据流转示例

以 Telegram 渠道为例：

```
1. Telegram 推送消息 → TelegramChannel._on_message()
2. 构造 InboundMessage → MessageBus.publish_inbound()
3. AgentLoop.run() 异步消费消息
4. _dispatch() 获取会话锁
5. _process_message():
   a. SessionManager 获取会话
   b. MemoryConsolidator 检查压缩
   c. ContextBuilder 构建完整 prompt（注入记忆）
   d. _run_agent_loop() ReAct 循环
   e. 如有工具调用 → 执行工具 → 结果回传给 LLM
   f. LLM 返回最终回复
6. 构造 OutboundMessage → MessageBus.publish_outbound()
7. ChannelManager._dispatch_outbound() 消费
8. TelegramChannel.send() 发送回复给用户
9. 后台任务 → 再次检查记忆压缩
```

## 技能系统

技能系统从 `skills/` 目录加载：

- 每个技能一个目录，包含 `SKILL.md`
- 支持 `always-on` （总是注入上下文）和按需加载
- 技能内容作为系统提示的一部分交给 LLM
- 类似于可插拔的能力模块

## 代码规模

整个项目非常精简，约 4000 行 Python：

| 模块 | 行数 | 职责 |
|------|------|------|
| `agent/loop.py` | 584 | 核心 ReAct 循环 |
| `agent/runner.py` | 232 | 纯粹的工具执行循环 |
| `agent/context.py` | 200 | Prompt 构建 |
| `channels/manager.py` | 264 | 渠道管理 |
| `providers/registry.py` | 355 | Provider 注册 |
| `config/schema.py` | 262 | 配置定义 |
| `cli/commands.py` | 1233 | CLI 命令处理 |
| `bus/queue.py` | 44 | 消息总线 |

## 设计特点总结

1. **简单胜于复杂**：没有过度工程，核心逻辑清晰易懂
2. **模块化解耦**：消息总线解耦渠道和核心引擎
3. **可靠的记忆管理**：两层架构 + 自动压缩解决上下文窗口问题
4. **渐进式压缩**：保持响应性，避免单次压缩时间过长
5. **完整的容错降级**：压缩失败不影响服务可用性
6. **多提供商支持**： easily 切换不同 LLM 服务

## 部署使用

```bash
# 安装
pip install nanobot-ai

# 初始化工作区
nanobot onboard

# 编辑配置 ~/.nanobot/config.yaml 添加 API 密钥

# 启动网关
nanobot gateway
```

配置完成后，你就拥有了一个自托管、带长期记忆和工具能力的个人 AI 助手，可以在飞书/Telegram 随时访问。

## 总结

Nanobot 是一个**设计优雅、代码精简的开源 AI Agent 项目**。它展示了如何用几千行代码构建一个完整的可用 AI 智能体，包含了 ReAct 循环、工具调用、自动记忆压缩等现代 AI Agent 核心技术。如果你想学习 AI Agent 原理，或者需要一个自托管的个人 AI 助手，Nanobot 是一个很好的起点。

项目源码：[https://github.com/HKUDS/nanobot](https://github.com/HKUDS/nanobot)
