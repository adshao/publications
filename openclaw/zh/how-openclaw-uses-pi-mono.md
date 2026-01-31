---
summary: "OpenClaw 与 Pi-mono 的深度集成：能力分层与工程化落地"
read_when:
  - 你要理解 OpenClaw 如何把 Pi-mono 变成生产级代理系统
  - 你希望提炼可复用的工程经验
---

[English](../en/how-openclaw-uses-pi-mono.md) | [中文](./how-openclaw-uses-pi-mono.md)

# OpenClaw 如何使用 Pi-mono

本文从源码实现出发，系统解析 OpenClaw 如何依托 Pi-mono 运行模型、管理会话与工具，再叠加自身的工程层，形成跨渠道、多模型、安全可控的完整能力。

Pi-mono 在 OpenClaw 中并不是一个黑盒，而是一套可组合的底层能力集合。OpenClaw 在这些能力之上增加了通道编排、工具治理、提示词注入、故障恢复和输出整形，最终形成端到端的“可用助手”。

---

## 1. Pi-mono 在 OpenClaw 里的角色定位

在代码层面，Pi-mono 对应的是一组核心包和能力：

- `@mariozechner/pi-ai`
  - 提供模型调用与流式输出能力
- `@mariozechner/pi-coding-agent`
  - 提供会话管理与工具执行框架
- `@mariozechner/pi-agent-core`
  - 提供消息与工具事件的统一类型体系

OpenClaw 并不直接实现一个 LLM runtime，而是用 Pi-mono 提供的这些抽象来搭建“嵌入式代理运行时”，核心入口在：

- `src/agents/pi-embedded-runner/run.ts`
- `src/agents/pi-embedded-runner/run/attempt.ts`

---

## 2. Pi-mono 提供的基础能力

以下能力是 OpenClaw 直接依赖的 Pi-mono 功能。

### 2.1 模型目录与 provider 抽象

OpenClaw 使用 Pi-mono 的模型注册与发现逻辑：

- `discoverModels` 和 `discoverAuthStorage` 来加载 provider 与模型列表
- 默认模型直接从 Pi-mono 内置 catalog 解析
- 当 config 中定义自定义 provider 时，通过内联模型构造进行兼容

实现路径：`src/agents/pi-embedded-runner/model.ts`

这使得 OpenClaw 无需重复实现模型注册逻辑，只需在本地配置补齐即可。

### 2.2 会话管理与持久化

Pi-mono 提供了 SessionManager 和 SettingsManager，OpenClaw 利用它们：

- 维护会话历史
- 控制 compaction 参数
- 保证会话记录一致性

核心调用：

- `SessionManager.open(...)`
- `SettingsManager.create(...)`

实现路径：`src/agents/pi-embedded-runner/run/attempt.ts`

### 2.3 工具执行框架

Pi-mono 提供 `codingTools` 和 ToolDefinition 机制。
OpenClaw 在此基础上扩展自己的工具体系，但仍复用其执行通道与回调接口。

相关路径：

- `@mariozechner/pi-coding-agent` 提供工具执行上下文
- `src/agents/pi-tool-definition-adapter.ts` 将 OpenClaw 工具适配为 Pi 工具

### 2.4 流式输出和事件模型

Pi-mono 提供流式输出与事件驱动接口，OpenClaw 通过订阅事件实现：

- 部分回复流
- 工具执行流
- compaction 过程回调

核心逻辑：

- `streamSimple` 用于流式输出
- `subscribeEmbeddedPiSession` 将 Pi 事件转成 OpenClaw 的输出粒度

实现路径：

- `src/agents/pi-embedded-runner/run/attempt.ts`
- `src/agents/pi-embedded-subscribe.ts`

---

## 3. OpenClaw 如何把 Pi-mono 变成可用系统

Pi-mono 提供基础能力，OpenClaw 在此之上做了大量工程化补强。

### 3.1 会话级串行执行与并发控制

OpenClaw 将每个 session 放进独立队列，保证：

- 同一会话不会并发执行
- 全局运行不会互相抢占

实现路径：

- `src/agents/pi-embedded-runner/lanes.ts`
- `src/agents/pi-embedded-runner/run.ts`

这对多渠道、多会话的稳定性至关重要。

### 3.2 认证与故障切换

OpenClaw 在 Pi-mono 基础上实现：

- 多 auth profile 轮换
- 失败标记与冷却
- 对 rate limit 和超时的识别
- 失败后可触发模型 fallback

实现路径：

- `src/agents/pi-embedded-runner/run.ts`
- `src/agents/model-auth.ts`
- `src/agents/model-fallback.ts`

### 3.3 系统提示词与上下文注入

OpenClaw 在 Pi-mono session 前构建系统提示词，注入：

- Workspace 文件与 bootstrap 内容
- Skills 指令
- Channel 能力说明
- Tool 列表与工具摘要
- Memory recall 规则

实现路径：

- `src/agents/pi-embedded-runner/system-prompt.ts`
- `src/agents/system-prompt.ts`
- `src/agents/bootstrap-files.ts`

这使得 Pi-mono 的基础代理变成“理解当前环境的代理”。

### 3.4 工具治理与安全控制

OpenClaw 在 Pi 工具体系之上叠加多级策略：

- global 级工具 allowlist 和 denylist
- agent 级工具策略
- group 级工具策略
- sandbox 级工具策略

还支持：

- 工具 schema 归一化，避免 provider 拒绝
- 特定 provider 的参数修正
- apply_patch 限制只对特定模型开放

实现路径：

- `src/agents/pi-tools.ts`
- `src/agents/pi-tools.policy.ts`
- `src/agents/pi-tools.schema.ts`

### 3.5 Transcript 健康修复

OpenClaw 处理 provider 差异时，不依赖用户输入，而是在 session 级别自动修复：

- 角色顺序修正
- tool call 和 tool result 配对修复
- Google turn ordering 修正
- OpenAI reasoning block 降级
- Gemini 不支持 schema keyword 的清洗

实现路径：

- `src/agents/pi-embedded-runner/google.ts`
- `src/agents/pi-embedded-helpers/openai.ts`
- `src/agents/transcript-policy.ts`

### 3.6 流式输出的可用化

Pi-mono 提供流式 token，但 OpenClaw进一步处理：

- block chunker 避免打断 fenced code
- partial reply 去重
- reasoning block 独立输出
- tool output 聚合与格式化

实现路径：

- `src/agents/pi-embedded-subscribe.ts`
- `src/agents/pi-embedded-block-chunker.ts`
- `src/agents/pi-embedded-utils.ts`

---

## 4. Pi-mono 与 OpenClaw 的能力分工图

简单总结角色分工：

- Pi-mono 解决
  - provider 抽象
  - session runtime
  - streaming 与工具调用

- OpenClaw 解决
  - 多渠道接入和路由
  - 工具安全策略
  - 提示词编排和上下文治理
  - transcript 质量修复
  - 真实场景下的稳定性策略

这种“基础 runtime + 业务增强层”的分层模式，使得 OpenClaw 可以跟随 Pi-mono 更新 provider 能力，而不必重写业务逻辑。

---

## 5. 最终形成的完整能力

当两者结合后，OpenClaw 具备以下完整能力链路：

1. **多 provider 模型选择与动态 fallback**
2. **持久会话与 compaction 管理**
3. **强工具生态**
4. **跨渠道一致的输出与动作行为**
5. **高可靠的流式回复与去重机制**
6. **可审计、可重放的 transcript 处理**

这让 OpenClaw 成为真正可用的跨渠道代理系统，而不是一个简单的 API 封装。

---

## 6. 工程经验总结

从 OpenClaw 和 Pi-mono 的集成中，可以提炼出以下可借鉴实践：

### 6.1 基础 runtime 与业务增强必须分层

OpenClaw 将 Pi-mono 视为“模型运行层”，将自身视为“交互层”。
这种分层避免了把业务逻辑塞进 provider 适配器，降低长期维护成本。

### 6.2 工具治理需要多级策略

真实场景中必须区分：

- 全局策略
- 单 agent 策略
- group 策略
- sandbox 策略

否则工具权限会被误放大或误封闭。

### 6.3 流式输出必须二次加工

模型的 raw stream 不能直接对外输出，需要：

- chunking 规则
- fenced code 保护
- reasoning 过滤
- duplicate suppression

这些都属于业务层必须补上的“输出整形”。

### 6.4 transcript 修复是必须组件

不同 provider 的格式限制与历史异常不可避免。
OpenClaw 选择在 session 层做统一修复，这使系统稳定性大幅提升。

### 6.5 错误分类与 fallback 是生产必需品

只做一次调用是不够的：

- 需要识别 rate limit
- 需要识别 auth failure
- 需要从失败中切换 profile 或模型

这一层是“从 demo 到系统”的关键跃迁。

---

## 7. 相关文档

- [Model providers](https://docs.openclaw.ai/concepts/model-providers)
- [OAuth](https://docs.openclaw.ai/concepts/oauth)
- [Architecture](https://docs.openclaw.ai/concepts/architecture)
- [Debugging](https://docs.openclaw.ai/debugging)
