---
summary: "OpenClaw 如何管理 Memory：从写入到索引与检索的完整机制"
read_when:
  - 你需要系统理解 OpenClaw memory 的内部实现与数据流
  - 你要进行 memory 索引和检索的深度调优
---

[English](../en/how-openclaw-manages-memory.md) | [中文](./how-openclaw-manages-memory.md)

# OpenClaw 如何管理 Memory

为了理解 OpenClaw memory 的内部机制，本文从源码实现出发，系统拆解 OpenClaw memory 的完整链路：
**记忆写入 → 文件发现 → 分块与嵌入 → SQLite 索引 → 混合检索 → 工具回注入**。

---

## 1. 设计边界与关键原则

OpenClaw 的 memory 是一个分层系统，而不是把所有历史塞回上下文。

- **源数据层**：Workspace 中的 Markdown 文件，永远可读且可审计。
- **派生索引层**：SQLite 索引，随时可重建。
- **检索层**：向量与关键词混合检索，返回 snippet 而不是整文件。

关键原则：

1. **可审计**：memory 的源数据是 Markdown。
2. **可恢复**：索引可删除重建，不影响源数据。
3. **可控与可调**：通过 `memorySearch` 配置细粒度调参。

---

## 2. 源数据层与文件发现

### 2.1 默认文件布局

Memory 的源数据是 Markdown 文件，默认布局如下：

- `MEMORY.md` 或 `memory.md`
  - 长期稳定记忆
- `memory/YYYY-MM-DD.md`
  - 日常记录
- `memory/YYYY-MM-DD-<slug>.md`
  - 会话归档

这些文件都在 workspace 根目录下（`agents.defaults.workspace`）。文档中建议使用 `<workspace>` 作为占位符。

### 2.2 文件发现规则

`src/memory/internal.ts` 的 `listMemoryFiles()` 负责扫描。
扫描范围：

- `<workspace>/MEMORY.md` 与 `<workspace>/memory.md`
- `<workspace>/memory/**/*.md`
- `memorySearch.extraPaths` 中配置的额外目录或文件

限制与细节：

- 只索引 `.md` 文件。
- 忽略所有符号链接。
- 额外路径可以是绝对路径或 workspace 相对路径。
- 真实路径去重，避免重复索引。

---

## 3. 记忆写入路径

OpenClaw 的记忆写入方式不是单一入口，而是多个机制协作：

### 3.1 手动写入

用户或模型显式写入 `MEMORY.md` 或 `memory/YYYY-MM-DD.md`，这是最直接也最可靠的方式。

### 3.2 session-memory hook

当用户执行 `/new` 时，如果启用了 `session-memory` hook，会将最近对话保存为独立文件：

- 文件命名：`memory/YYYY-MM-DD-<slug>.md`
- 内容包括：Session 元信息 + 近期对话摘要

实现路径：`src/hooks/bundled/session-memory/handler.ts`
核心行为：

1. 读取 session JSONL，提取 user 和 assistant 文本。
2. 使用 LLM 生成 slug，失败则回退到时间戳。
3. 写入 Markdown 文件。

### 3.3 预压缩 memory flush

当会话接近 compaction，会触发一次 **隐式 agent turn**，提醒模型把持久记忆写入文件。

实现路径：
- `src/auto-reply/reply/memory-flush.ts`
- `src/auto-reply/reply/agent-runner-memory.ts`

核心触发条件：

```
threshold = contextWindow - reserveTokensFloor - softThresholdTokens
```

当 `totalTokens >= threshold`，且当前 compaction 周期尚未 flush，则触发。

关键特性：

- 默认 prompt 中包含 `NO_REPLY`，避免用户看到该回合。
- sandbox workspace 只读或不可用时跳过。
- CLI provider 会跳过。
- 每个 compaction 周期只触发一次。

配置入口：`agents.defaults.compaction.memoryFlush`。

---

## 4. 索引系统架构

核心实现类：`MemoryIndexManager`（`src/memory/manager.ts`）。
它负责索引调度、同步、检索入口、缓存与回退。

### 4.1 索引存储位置

索引是 per agent 的 SQLite 文件：

```
<state-dir>/memory/<agentId>.sqlite
```

`<state-dir>` 由 `OPENCLAW_STATE_DIR` 决定。
若未显式设置，默认路径遵循 state dir 的解析规则，并支持新旧目录选择逻辑。

### 4.2 SQLite Schema

初始化逻辑在 `src/memory/memory-schema.ts`。
关键表：

- `files`：文件元数据（hash、mtime、size、source）
- `chunks`：chunk 文本、行号范围、embedding JSON
- `embedding_cache`：embedding 缓存（provider model key hash）
- `chunks_vec`：sqlite-vec 向量索引（可选）
- `chunks_fts`：FTS5 关键词索引（可选）

### 4.3 向量扩展与降级策略

- 如果 `store.vector.enabled` 为 true，会尝试加载 sqlite-vec 扩展。
- 加载失败时自动降级为内存相似度计算。
- 向量维度变化会触发向量表重建。

---

## 5. 分块与标识体系

### 5.1 文件 hash

每个文件读取后计算 SHA-256 hash，用于判断是否需要重新索引。
若 hash 未变化，则跳过该文件。

### 5.2 chunk 规则

默认 chunking 参数：

- `tokens = 400`
- `overlap = 80`

实现逻辑：

- 按行分割并记录行号区间。
- 以 `tokens * 4` 字符估算 chunk 大小。
- 保留 overlap 形成滑窗。

### 5.3 chunk ID 生成

每个 chunk 的唯一 ID 由以下字段 hash 得到：

```
source + path + startLine + endLine + chunkHash + model
```

这保证不同来源、不同模型或行号变动都会生成新 chunk ID。

---

## 6. Embedding 生成与缓存

### 6.1 Provider 选择逻辑

`resolveMemorySearchConfig()` 会把未显式配置的 provider 解析为 `auto`。
`auto` 的顺序逻辑在 `createEmbeddingProvider()`：

1. 本地模型可用且路径存在
2. OpenAI 可解析出 API key
3. Gemini 可解析出 API key
4. 否则 memory search 不可用

### 6.2 OpenAI 与 Gemini 客户端构建

- OpenAI 使用 `POST /embeddings`
- Gemini 使用 `embedContent` 与 `batchEmbedContents`

两者都支持 `memorySearch.remote` 的 `baseUrl` 和 `headers`。

### 6.3 Local provider

Local provider 使用 `node-llama-cpp`。
OpenClaw 只负责调用 `resolveModelFile()` 和 `createEmbeddingContext()`，模型解析与缓存由该库处理。

### 6.4 批处理与容错

索引时默认使用 batch：

- OpenAI 和 Gemini 优先走 batch API。
- 单次 batch 最多 50k 请求。
- OpenAI 使用 `/files` + `/batches`，completion window 为 `24h`。
- Gemini 使用 upload endpoint 并调用 `asyncBatchEmbedContent`。

容错机制：

- batch 超时会自动重试一次。
- 连续失败达到上限会禁用 batch，并回退到普通 embedding。
- 部分 provider 报错会强制禁用 batch。

### 6.5 Embedding 缓存

`embedding_cache` 以 `provider + model + providerKey + hash` 作为键：

- providerKey 会包含 baseUrl、model 和 headers 指纹，避免跨端点污染。
- 缓存会按 `updated_at` 进行淘汰。
- full reindex 时会尝试从旧索引中 seed 缓存。

---

## 7. 同步与新鲜度策略

索引同步是异步执行，不阻塞搜索请求。
核心机制：

### 7.1 触发源

- **session start**：触发 warm sync
- **search 时**：如果 dirty 则后台同步
- **watcher**：文件变更触发
- **interval**：定时触发

### 7.2 watcher 细节

- 使用 chokidar
- 默认 debounce 1500ms
- 监听 `MEMORY.md`、`memory/` 和 extraPaths

### 7.3 session transcripts 索引

当开启 session 索引：

- 监听 JSONL 文件变动
- 以新增字节数或行数作为阈值
- 只抽取 user 和 assistant 文本
- 文本会被压缩为单行

索引文件路径形式：

```
<state-dir>/agents/<agentId>/sessions/*.jsonl
```

### 7.4 异步一致性

`memory_search` 不会等待 sync 完成，结果可能略有滞后。
这是一种延迟优先的设计选择。

---

## 8. 检索机制与混合评分

### 8.1 向量检索

向量检索路径：

- 若 sqlite-vec 可用：使用 `vec_distance_cosine`
- 否则：载入所有 chunk embedding 并计算 cosine similarity

最终分数计算：

```
score = 1 - cosine_distance
```

### 8.2 关键词检索

若 FTS5 可用：

- 使用 BM25 排序
- 只提取字母数字和下划线 token
- 多 token 采用 AND 组合

BM25 rank 被转换成：

```
textScore = 1 / (1 + rank)
```

### 8.3 混合合并

混合检索的合并公式：

```
finalScore = vectorWeight * vectorScore + textWeight * textScore
```

默认参数：

- `vectorWeight = 0.7`
- `textWeight = 0.3`
- `candidateMultiplier = 4`
- 候选池最多 200

### 8.4 返回结果

每条结果包含：

- path
- startLine, endLine
- snippet（最多 700 字符）
- score
- source（memory 或 sessions）

---

## 9. 工具层与系统提示注入

### 9.1 memory 工具（运行时）

当 memory search 启用时，OpenClaw 会暴露两个 memory 工具；工具定义位于
`src/agents/tools/memory-tool.ts`，并在策略允许时被加入运行时工具列表：

- `memory_search`
  - 参数：`query`，可选 `maxResults`、`minScore`
  - 通过 `getMemorySearchManager(...)` 对 `MEMORY.md`、`memory/**/*.md`
    以及可选的会话转录（`memorySearch.sources` 启用时）做语义检索
  - 返回 top snippets，包含 `path` 与行号区间，并带上 provider/model 元数据
- `memory_get`
  - 参数：`path`，可选 `from`、`lines`
  - 通过 `MemorySearchManager.readFile(...)` 只读取需要的行
  - 允许路径限制在 `MEMORY.md`、`memory/**/*.md` 与配置的 `memorySearch.extraPaths`

这两个工具仅在 `resolveMemorySearchConfig(...)` 返回 enabled 配置时可用。

### 9.2 system prompt 注入规则

当 `memory_search` 或 `memory_get` 可用时，`buildAgentSystemPrompt(...)` 会插入 **Memory Recall**
提示，要求模型在处理历史问题时：

1. 先执行 `memory_search`
2. 再用 `memory_get` 拉取必要行，避免上下文膨胀

实现路径：`src/agents/system-prompt.ts`。

### 9.3 Project Context 与 memory 工具的分工

`MEMORY.md` 或 `memory.md` 可能作为 bootstrap 文件被注入到 **Project Context**
（超过上限会被截断）。

但注意：

- `memory/` 目录不会被注入进 Project Context
- `memory_search` / `memory_get` 才是持久记忆的标准检索路径，尤其适合长历史或被截断场景

---

## 10. CLI 与运维入口

OpenClaw 同时提供 CLI：

- `openclaw memory status`
- `openclaw memory status --deep`
- `openclaw memory index`
- `openclaw memory search "..."`

`status --deep` 会探测：

- sqlite-vec 是否可用
- embeddings 是否可用

参考：
- [CLI memory](https://docs.openclaw.ai/cli/memory)
- [Plugins](https://docs.openclaw.ai/plugin)

---

## 11. 调优指南

### 11.1 准确率与召回

- 增大 `chunking.tokens` 提高语义连贯性，但降低精确定位。
- 调整 `query.minScore` 控制召回阈值。
- 提高 `query.hybrid.textWeight` 改善对代码符号和 ID 的命中。

### 11.2 性能与成本

- batch API 适合大规模回填索引。
- `cache.maxEntries` 可降低重复 embedding 成本。
- 向量扩展不可用时，检索会变慢，建议优先修复 sqlite-vec。

### 11.3 Session 索引

- 对快速滚动的 session，可以提高 `sync.sessions.deltaBytes` 或 `deltaMessages`。
- 如果更关注近期对话内容，可启用 `sources: ["memory", "sessions"]`。

---

## 12. 常见故障路径与排查

1. **memory_search 无结果**
   - 检查索引是否 dirty
   - 运行 `openclaw memory status --deep --index`

2. **向量检索不可用**
   - sqlite-vec 加载失败会自动降级
   - 查看 `status --deep` 的 vector 状态

3. **embedding 报错**
   - 确认 API key 解析路径
   - 检查 provider 是否被 fallback

4. **memory flush 未执行**
   - workspace 可能处于只读或无权限
   - compaction 周期内已执行过一次

---

## 13. 总结

OpenClaw memory 是一个可审计、可重建、可调优的工程化记忆系统：

1. Markdown 是唯一事实来源
2. SQLite 索引提供可重建的检索层
3. 向量与关键词混合检索保证语义与精确度
4. 自动 flush 和 hook 机制保证重要记忆不丢失

这套设计非常适合个人或团队的长期智能助手场景，兼顾可控性与可扩展性。

---

## 延伸阅读

- [Memory](https://docs.openclaw.ai/concepts/memory)
- [Agent workspace](https://docs.openclaw.ai/concepts/agent-workspace)
- [Session management compaction](https://docs.openclaw.ai/reference/session-management-compaction)
- [Hooks](https://docs.openclaw.ai/hooks)
- [Plugins](https://docs.openclaw.ai/plugin)
- [CLI memory](https://docs.openclaw.ai/cli/memory)
