---
summary: "How OpenClaw Manages Memory: from writes to indexing and retrieval"
read_when:
  - You need a systematic understanding of OpenClaw memory internals and data flow
  - You want to tune memory indexing and retrieval in depth
---

[English](./how-openclaw-manages-memory.md) | [中文](../zh/how-openclaw-manages-memory.md)

# How OpenClaw Manages Memory

To understand OpenClaw memory internals, this article starts from the source code and breaks down the full OpenClaw memory pipeline:
**memory writes -> file discovery -> chunking and embeddings -> SQLite index -> hybrid retrieval -> tool re-injection**.

---

## 1. Design boundaries and key principles

OpenClaw memory is a layered system, not "dump all history back into context."

- **Source data layer**: Markdown files in the workspace, always readable and auditable.
- **Derived index layer**: SQLite index, always rebuildable.
- **Retrieval layer**: Hybrid vector + keyword search, returning snippets instead of whole files.

Key principles:

1. **Auditable**: the source of truth is Markdown.
2. **Recoverable**: indexes can be deleted and rebuilt without touching source data.
3. **Controllable and tunable**: `memorySearch` exposes fine-grained tuning.

---

## 2. Source data layer and file discovery

### 2.1 Default file layout

Memory source data lives in Markdown files with the default layout:

- `MEMORY.md` or `memory.md`
  - Long-term stable memory
- `memory/YYYY-MM-DD.md`
  - Daily notes
- `memory/YYYY-MM-DD-<slug>.md`
  - Session archives

These files live at the workspace root (`agents.defaults.workspace`). Docs use `<workspace>` as a placeholder.

### 2.2 File discovery rules

`listMemoryFiles()` in `src/memory/internal.ts` scans:

- `<workspace>/MEMORY.md` and `<workspace>/memory.md`
- `<workspace>/memory/**/*.md`
- extra directories or files configured in `memorySearch.extraPaths`

Limits and details:

- Only `.md` files are indexed.
- All symlinks are ignored.
- Extra paths can be absolute or workspace-relative.
- Real paths are de-duplicated to avoid double indexing.

---

## 3. Memory write paths

OpenClaw memory writes are not a single entry point, but multiple cooperating mechanisms:

### 3.1 Manual writes

Users or models explicitly write to `MEMORY.md` or `memory/YYYY-MM-DD.md`. This is the most direct and reliable path.

### 3.2 session-memory hook

When a user runs `/new`, the `session-memory` hook (if enabled) saves recent conversation to a standalone file:

- File name: `memory/YYYY-MM-DD-<slug>.md`
- Content: session metadata + recent conversation summary

Implementation: `src/hooks/bundled/session-memory/handler.ts`

Core behavior:

1. Read session JSONL and extract user + assistant text.
2. Use an LLM to generate a slug; fall back to a timestamp on failure.
3. Write the Markdown file.

### 3.3 Pre-compaction memory flush

When a session approaches compaction, an **implicit agent turn** is triggered to prompt the model to write durable memory to files.

Implementation:
- `src/auto-reply/reply/memory-flush.ts`
- `src/auto-reply/reply/agent-runner-memory.ts`

Core trigger condition:

```
threshold = contextWindow - reserveTokensFloor - softThresholdTokens
```

When `totalTokens >= threshold` and the current compaction cycle has not flushed, the flush runs.

Key traits:

- Default prompt includes `NO_REPLY` to keep it hidden from users.
- Skips when the sandbox workspace is read-only or unavailable.
- Skips for the CLI provider.
- Runs at most once per compaction cycle.

Config entry: `agents.defaults.compaction.memoryFlush`.

---

## 4. Index system architecture

Core class: `MemoryIndexManager` (`src/memory/manager.ts`).
It owns indexing orchestration, sync, retrieval entry points, caches, and fallbacks.

### 4.1 Index storage location

Indexes are per-agent SQLite files:

```
<state-dir>/memory/<agentId>.sqlite
```

`<state-dir>` is determined by `OPENCLAW_STATE_DIR`.
If unset, default state-dir resolution rules apply, including legacy path selection.

### 4.2 SQLite schema

Initialization lives in `src/memory/memory-schema.ts`.
Key tables:

- `files`: file metadata (hash, mtime, size, source)
- `chunks`: chunk text, line ranges, embedding JSON
- `embedding_cache`: embedding cache (provider model key hash)
- `chunks_vec`: sqlite-vec vector index (optional)
- `chunks_fts`: FTS5 keyword index (optional)

### 4.3 Vector extension and fallback

- If `store.vector.enabled` is true, OpenClaw tries to load the sqlite-vec extension.
- If loading fails, it falls back to in-memory similarity computation.
- Vector dimension changes trigger vector table rebuilds.

---

## 5. Chunking and identifiers

### 5.1 File hash

Each file is read and hashed with SHA-256 to decide whether reindexing is needed.
If the hash is unchanged, the file is skipped.

### 5.2 Chunk rules

Default chunking parameters:

- `tokens = 400`
- `overlap = 80`

Implementation logic:

- Split by lines and record line ranges.
- Estimate chunk size as `tokens * 4` characters.
- Keep overlap with a sliding window.

### 5.3 Chunk ID generation

Each chunk's unique ID hashes:

```
source + path + startLine + endLine + chunkHash + model
```

This ensures new chunk IDs when source, model, or line positions change.

---

## 6. Embedding generation and cache

### 6.1 Provider selection logic

`resolveMemorySearchConfig()` resolves unspecified providers to `auto`.
The `auto` order in `createEmbeddingProvider()` is:

1. Local model available and path exists
2. OpenAI API key available
3. Gemini API key available
4. Otherwise memory search is disabled

### 6.2 OpenAI and Gemini client setup

- OpenAI uses `POST /embeddings`
- Gemini uses `embedContent` and `batchEmbedContents`

Both respect `memorySearch.remote` `baseUrl` and `headers`.

### 6.3 Local provider

The local provider uses `node-llama-cpp`.
OpenClaw only calls `resolveModelFile()` and `createEmbeddingContext()`; model resolution and caching are handled by the library.

### 6.4 Batching and resilience

Indexing uses batches by default:

- OpenAI and Gemini prefer batch APIs.
- Max batch size is 50k requests.
- OpenAI uses `/files` + `/batches` with a 24h completion window.
- Gemini uses the upload endpoint and calls `asyncBatchEmbedContent`.

Resilience:

- Batch timeouts retry once.
- Repeated failures disable batch and fall back to standard embeddings.
- Some provider errors force batch to disable.

### 6.5 Embedding cache

`embedding_cache` keys on `provider + model + providerKey + hash`:

- providerKey includes baseUrl, model, and header fingerprints to avoid cross-endpoint pollution.
- Cache eviction is ordered by `updated_at`.
- Full reindex attempts to seed cache from previous indexes.

---

## 7. Sync and freshness strategy

Index sync runs asynchronously and does not block search requests.
Core mechanisms:

### 7.1 Triggers

- **Session start**: warm sync
- **On search**: background sync when dirty
- **Watcher**: file change trigger
- **Interval**: periodic trigger

### 7.2 Watcher details

- Uses chokidar
- Default debounce is 1500ms
- Watches `MEMORY.md`, `memory/`, and extraPaths

### 7.3 Session transcript indexing

When session indexing is enabled:

- Watches JSONL file changes
- Uses added bytes or line count as thresholds
- Extracts user + assistant text only
- Compresses text to a single line

Indexed file path form:

```
<state-dir>/agents/<agentId>/sessions/*.jsonl
```

### 7.4 Asynchronous consistency

`memory_search` does not wait for sync to complete, so results can be slightly stale.
This is a latency-first design choice.

---

## 8. Retrieval and hybrid scoring

### 8.1 Vector retrieval

Vector retrieval path:

- If sqlite-vec is available: use `vec_distance_cosine`
- Otherwise: load all chunk embeddings and compute cosine similarity

Final score:

```
score = 1 - cosine_distance
```

### 8.2 Keyword retrieval

If FTS5 is available:

- Rank by BM25
- Tokens include only alphanumerics and underscores
- Multiple tokens are combined with AND

BM25 rank converts to:

```
textScore = 1 / (1 + rank)
```

### 8.3 Hybrid merge

Hybrid merge formula:

```
finalScore = vectorWeight * vectorScore + textWeight * textScore
```

Default parameters:

- `vectorWeight = 0.7`
- `textWeight = 0.3`
- `candidateMultiplier = 4`
- Max candidate pool 200

### 8.4 Returned results

Each result includes:

- path
- startLine, endLine
- snippet (max 700 chars)
- score
- source (memory or sessions)

---

## 9. Tool layer and system prompt injection

### 9.1 Memory tools (runtime)

OpenClaw exposes two memory tools when memory search is enabled; the tool definitions live in
`src/agents/tools/memory-tool.ts` and are included in the runtime tool list when allowed by policy:

- `memory_search`
  - Parameters: `query`, optional `maxResults`, optional `minScore`
  - Uses `getMemorySearchManager(...)` to run semantic search over `MEMORY.md`, `memory/**/*.md`,
    and optional session transcripts (when enabled in `memorySearch.sources`).
  - Returns top snippets with `path` and line ranges, plus provider/model metadata.
- `memory_get`
  - Parameters: `path`, optional `from`, optional `lines`
  - Uses `MemorySearchManager.readFile(...)` to fetch only the needed lines.
  - Allowed paths are constrained to `MEMORY.md`, `memory/**/*.md`, and configured `memorySearch.extraPaths`.

These tools are only available when `resolveMemorySearchConfig(...)` returns enabled config.

### 9.2 System prompt guidance (model behavior)

When `memory_search` or `memory_get` is available, `buildAgentSystemPrompt(...)` injects a **Memory Recall**
section that instructs the model to:

1. Run `memory_search` before answering about past work, decisions, dates, people, preferences, or todos.
2. Follow up with `memory_get` to pull only the relevant lines.

Implementation: `src/agents/system-prompt.ts`.

### 9.3 Project Context vs. memory tools

`MEMORY.md` or `memory.md` may also appear in **Project Context** because bootstrap files are injected
into the system prompt (with truncation if they exceed the max chars limit).

However:

- The `memory/` directory is **not** injected into Project Context.
- `memory_search`/`memory_get` are the canonical retrieval path for durable memory, especially for
  long history or when Project Context was truncated.

---

## 10. CLI and ops entry points

OpenClaw also provides CLI commands:

- `openclaw memory status`
- `openclaw memory status --deep`
- `openclaw memory index`
- `openclaw memory search "..."`

`status --deep` probes:

- sqlite-vec availability
- embeddings availability

References:
- [CLI memory](https://docs.openclaw.ai/cli/memory)
- [Plugins](https://docs.openclaw.ai/plugin)

---

## 11. Tuning guide

### 11.1 Precision vs recall

- Increase `chunking.tokens` for better semantic coherence, but worse pinpoint accuracy.
- Tune `query.minScore` to control recall threshold.
- Increase `query.hybrid.textWeight` to improve hits for code symbols and IDs.

### 11.2 Performance and cost

- Batch APIs are good for large backfills.
- `cache.maxEntries` reduces repeated embedding cost.
- When the vector extension is unavailable, search slows down; prioritize fixing sqlite-vec.

### 11.3 Session indexing

- For fast-moving sessions, increase `sync.sessions.deltaBytes` or `deltaMessages`.
- If you care more about recent dialog, enable `sources: ["memory", "sessions"]`.

---

## 12. Common failure paths and debugging

1. **No results from memory_search**
   - Check whether the index is dirty
   - Run `openclaw memory status --deep --index`

2. **Vector search unavailable**
   - sqlite-vec load failure falls back automatically
   - Check vector status in `status --deep`

3. **Embedding errors**
   - Confirm API key resolution paths
   - Check whether the provider was fallen back

4. **Memory flush not executed**
   - Workspace may be read-only or inaccessible
   - A flush already ran in the current compaction cycle

---

## 13. Summary

OpenClaw memory is an auditable, rebuildable, and tunable engineering memory system:

1. Markdown is the only source of truth
2. SQLite indexes provide a rebuildable retrieval layer
3. Hybrid vector + keyword retrieval balances semantics and precision
4. Automated flush and hooks keep important memory from being lost

This design is ideal for long-term personal or team assistant scenarios, balancing control and extensibility.

---

## Further reading

- [Memory](https://docs.openclaw.ai/concepts/memory)
- [Agent workspace](https://docs.openclaw.ai/concepts/agent-workspace)
- [Session management compaction](https://docs.openclaw.ai/reference/session-management-compaction)
- [Hooks](https://docs.openclaw.ai/hooks)
- [Plugins](https://docs.openclaw.ai/plugin)
- [CLI memory](https://docs.openclaw.ai/cli/memory)
