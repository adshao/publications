---
summary: "OpenClaw and Pi-mono deep integration: layered capabilities and engineering delivery"
read_when:
  - You want to understand how OpenClaw turns Pi-mono into a production-grade agent system
  - You want to distill reusable engineering lessons
---

[English](./how-openclaw-uses-pi-mono.md) | [中文](../zh/how-openclaw-uses-pi-mono.md)

# How OpenClaw Uses Pi-mono

This article starts from the source code and explains how OpenClaw relies on Pi-mono to run models, manage sessions, and execute tools. OpenClaw then layers its own engineering stack on top to deliver a complete, secure, multi-channel, multi-model system.

Pi-mono is not a black box inside OpenClaw. It is a composable set of low-level capabilities. OpenClaw adds channel orchestration, tool governance, prompt injection, failure recovery, and output shaping on top, forming an end-to-end "usable assistant."

---

## 1. Pi-mono's role inside OpenClaw

At the code level, Pi-mono maps to a set of core packages and capabilities:

- `@mariozechner/pi-ai`
  - Model calls and streaming output
- `@mariozechner/pi-coding-agent`
  - Session management and tool execution framework
- `@mariozechner/pi-agent-core`
  - Unified message and tool event types

OpenClaw does not implement its own LLM runtime. It uses Pi-mono's abstractions to build an "embedded agent runtime." Core entry points:

- `src/agents/pi-embedded-runner/run.ts`
- `src/agents/pi-embedded-runner/run/attempt.ts`

---

## 2. Baseline capabilities from Pi-mono

The following are Pi-mono capabilities that OpenClaw depends on directly.

### 2.1 Model catalog and provider abstraction

OpenClaw uses Pi-mono's model registration and discovery:

- `discoverModels` and `discoverAuthStorage` to load providers and model lists
- Default models come from Pi-mono's built-in catalog
- Custom providers in config are handled via inline model construction

Implementation: `src/agents/pi-embedded-runner/model.ts`

This lets OpenClaw avoid re-implementing model registration and focus on local config.

### 2.2 Session management and persistence

Pi-mono provides SessionManager and SettingsManager, which OpenClaw uses to:

- Maintain session history
- Control compaction parameters
- Keep session records consistent

Core calls:

- `SessionManager.open(...)`
- `SettingsManager.create(...)`

Implementation: `src/agents/pi-embedded-runner/run/attempt.ts`

### 2.3 Tool execution framework

Pi-mono provides `codingTools` and the ToolDefinition mechanism.
OpenClaw extends the tool system while reusing Pi-mono's execution channels and callbacks.

Relevant paths:

- `@mariozechner/pi-coding-agent` provides the tool execution context
- `src/agents/pi-tool-definition-adapter.ts` adapts OpenClaw tools into Pi tools

### 2.4 Streaming output and event model

Pi-mono provides streaming output and event-driven interfaces. OpenClaw subscribes to events to implement:

- Partial reply streaming
- Tool execution streaming
- Compaction callbacks

Core logic:

- `streamSimple` for streaming output
- `subscribeEmbeddedPiSession` to convert Pi events into OpenClaw output granularity

Implementation:

- `src/agents/pi-embedded-runner/run/attempt.ts`
- `src/agents/pi-embedded-subscribe.ts`

---

## 3. How OpenClaw turns Pi-mono into a usable system

Pi-mono provides the foundation. OpenClaw adds significant engineering reinforcements on top.

### 3.1 Session-level serial execution and concurrency control

OpenClaw puts each session into a dedicated queue to ensure:

- No concurrent execution within a session
- Global runs do not starve each other

Implementation:

- `src/agents/pi-embedded-runner/lanes.ts`
- `src/agents/pi-embedded-runner/run.ts`

This is essential for stability across multiple channels and sessions.

### 3.2 Auth and failover

OpenClaw extends Pi-mono with:

- Multiple auth profile rotation
- Failure marking and cooldown
- Rate limit and timeout detection
- Model fallback on failure

Implementation:

- `src/agents/pi-embedded-runner/run.ts`
- `src/agents/model-auth.ts`
- `src/agents/model-fallback.ts`

### 3.3 System prompts and context injection

OpenClaw builds system prompts before the Pi-mono session and injects:

- Workspace files and bootstrap content
- Skills instructions
- Channel capability notes
- Tool lists and tool summaries
- Memory recall rules

Implementation:

- `src/agents/pi-embedded-runner/system-prompt.ts`
- `src/agents/system-prompt.ts`
- `src/agents/bootstrap-files.ts`

This turns a baseline Pi-mono agent into an "environment-aware agent."

### 3.4 Tool governance and safety control

OpenClaw adds multi-layer policies on top of Pi tools:

- Global tool allowlist and denylist
- Per-agent tool policies
- Per-group tool policies
- Per-sandbox tool policies

It also supports:

- Tool schema normalization to avoid provider rejections
- Provider-specific parameter fixes
- apply_patch restricted to specific models

Implementation:

- `src/agents/pi-tools.ts`
- `src/agents/pi-tools.policy.ts`
- `src/agents/pi-tools.schema.ts`

### 3.5 Transcript health repair

To handle provider differences, OpenClaw repairs transcripts at the session level, without relying on user input:

- Role order fixes
- Tool call and tool result pairing fixes
- Google turn ordering fixes
- OpenAI reasoning block downgrades
- Gemini schema keyword cleanup

Implementation:

- `src/agents/pi-embedded-runner/google.ts`
- `src/agents/pi-embedded-helpers/openai.ts`
- `src/agents/transcript-policy.ts`

### 3.6 Making streaming output usable

Pi-mono streams raw tokens, but OpenClaw further processes them:

- Block chunker to avoid breaking fenced code
- Partial reply de-duplication
- Reasoning block isolated output
- Tool output aggregation and formatting

Implementation:

- `src/agents/pi-embedded-subscribe.ts`
- `src/agents/pi-embedded-block-chunker.ts`
- `src/agents/pi-embedded-utils.ts`

---

## 4. Capability split between Pi-mono and OpenClaw

In short:

- Pi-mono solves
  - Provider abstraction
  - Session runtime
  - Streaming and tool calls

- OpenClaw solves
  - Multi-channel access and routing
  - Tool safety policy
  - Prompt orchestration and context governance
  - Transcript quality repair
  - Stability strategies in real-world conditions

This layered model lets OpenClaw track Pi-mono provider updates without re-writing business logic.

---

## 5. Final capability stack

Together, OpenClaw delivers a complete capability chain:

1. **Multi-provider model selection and dynamic fallback**
2. **Persistent sessions and compaction management**
3. **Strong tool ecosystem**
4. **Consistent output and action behavior across channels**
5. **Highly reliable streaming replies and de-duplication**
6. **Auditable, replayable transcript processing**

This makes OpenClaw a truly usable multi-channel agent system, not just a thin API wrapper.

---

## 6. Engineering takeaways

From the OpenClaw + Pi-mono integration, you can extract these reusable practices:

### 6.1 Base runtime and business layer must be separated

OpenClaw treats Pi-mono as the "model runtime layer" and itself as the "interaction layer."
This avoids stuffing business logic into provider adapters and reduces long-term maintenance cost.

### 6.2 Tool governance needs multiple layers

Real-world setups must distinguish:

- Global policies
- Per-agent policies
- Per-group policies
- Per-sandbox policies

Otherwise tool permissions are easily over-granted or incorrectly blocked.

### 6.3 Streaming output needs post-processing

Raw model streams should not go directly to users. You need:

- Chunking rules
- Fenced code protection
- Reasoning filtering
- Duplicate suppression

These are business-layer responsibilities.

### 6.4 Transcript repair is mandatory

Provider format limitations and historical anomalies are unavoidable.
OpenClaw chooses to repair at the session layer, which greatly improves stability.

### 6.5 Error classification and fallback are production requirements

A single call is not enough:

- You must detect rate limits
- You must detect auth failures
- You must switch profiles or models after failures

This is the key jump from demo to system.

---

## 7. Related docs

- [Model providers](https://docs.openclaw.ai/concepts/model-providers)
- [OAuth](https://docs.openclaw.ai/concepts/oauth)
- [Architecture](https://docs.openclaw.ai/concepts/architecture)
- [Debugging](https://docs.openclaw.ai/debugging)
