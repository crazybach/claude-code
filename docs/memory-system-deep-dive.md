# Claude Code Memory System Deep Dive

This document explains how memory works in this codebase beyond the top-level architecture.

## 1) What “memory” is in this repo

Claude Code memory is a **file-backed persistence layer** rooted in `src/memdir/`, with runtime guidance injected into the system prompt and optional retrieval of relevant memory files per turn.

At a high level:

1. Resolve whether memory is enabled and where memory lives.
2. Inject memory behavior/instructions into the system prompt.
3. Persist memory as Markdown files (+ `MEMORY.md` index files).
4. Optionally auto-extract and save memories in the background.
5. Retrieve only relevant memories for a user query and attach them with freshness caveats.

---

## 2) Enablement and path resolution

`isAutoMemoryEnabled()` defines the gate and priority order:

- `CLAUDE_CODE_DISABLE_AUTO_MEMORY` env override,
- `CLAUDE_CODE_SIMPLE` (`--bare`) disables,
- remote mode without persistent memory directory disables,
- `autoMemoryEnabled` in settings,
- default enabled.

Memory directory resolution (`getAutoMemPath`) supports:

- explicit env override for Cowork,
- trusted settings override,
- otherwise computed project-scoped path under config home.

The path validator rejects unsafe roots / traversal-prone forms (relative paths, near-root, UNC, null bytes, etc.).

---

## 3) Memory schema and taxonomy

Memory files are Markdown with frontmatter, and memories are constrained to four types:

- `user`
- `feedback`
- `project`
- `reference`

The prompt guidance explicitly says **do not save derivable project state** (architecture/code patterns/git history), and includes “when to access”, staleness caveats, and “verify before recommending from memory” behavior.

---

## 4) Prompt-time injection behavior

Memory prompt generation is centralized in `loadMemoryPrompt()`:

- returns `null` when auto memory is off,
- emits single-directory memory guidance for standard mode,
- emits combined private+team prompt when TEAMMEM is enabled,
- emits Kairos-specific append-only daily-log guidance in assistant mode.

In the normal system prompt path (`constants/prompts.ts`), memory is loaded as a cached dynamic section (`systemPromptSection('memory', ...)`).

In SDK/custom-prompt scenarios, QueryEngine can still inject “memory mechanics” if the caller explicitly opts in via memory path override.

---

## 5) Write path: how memory gets persisted

There are two write mechanisms:

### A. Main agent direct writes
The assistant writes Markdown memory files and updates `MEMORY.md` pointers following prompt rules.

### B. Background extraction agent (`extractMemories`)
A forked sub-agent periodically scans recent conversation deltas and writes memories when the main agent didn’t.

Important behavior:

- Mutual exclusion: if direct memory writes already happened, extractor skips.
- Throttled execution (feature-flag configurable cadence).
- Pre-injects memory manifest to avoid wasting turns on directory listing.
- Bounded turns (`maxTurns: 5`) to prevent rabbit holes.

---

## 6) Read/recall path: relevant-memory retrieval

The retrieval flow lives in `findRelevantMemories.ts` + `utils/attachments.ts`:

1. Scan candidate memory files (`scanMemoryFiles`) excluding `MEMORY.md`.
2. Parse only frontmatter/header slices for efficiency.
3. Use a side Sonnet call with JSON schema output to select up to 5 relevant files.
4. Read selected files with line/byte caps and attach as `relevant_memories`.
5. Deduplicate against already surfaced and already-read files.

The prefetch runs asynchronously so it doesn’t block the main turn loop; if not ready at collect point, it can be skipped and retried later.

---

## 7) Safety and quality controls

### Size + truncation controls
- `MEMORY.md` entrypoint caps (`MAX_ENTRYPOINT_LINES`, `MAX_ENTRYPOINT_BYTES`) with explicit truncation warning text.
- Relevant-memory attachment reads are also bounded (`MAX_MEMORY_LINES`, `MAX_MEMORY_BYTES`) and include a “read full file with FileReadTool” note when truncated.

### Freshness controls
- Memory age helpers convert mtime into human-readable staleness (“N days ago”).
- For older memories, explicit freshness warning text is injected to push verification against current code.

### Directory existence and reliability
- Memory directories are ensured via `ensureMemoryDirExists` before prompting model to write.
- Errors are logged but prompt generation continues (write errors surface when tool actually runs).

---

## 8) Team memory mode

When TEAMMEM is enabled, memory is split into:

- **private** memory (user-specific),
- **team** memory (`.../memory/team`) synced with backend API.

Sync service (`services/teamMemorySync`) characteristics:

- repo-scoped GET/PUT sync with checksums/ETags,
- delta uploads using per-entry checksums,
- server-wins pull semantics per key,
- deletions are not propagated (server entries can restore on next pull),
- guarded by OAuth requirements and secret scanning before push.

---

## 9) Assistant/Kairos mode nuance

In KAIROS mode, memory writes switch to append-only **daily logs** (date path pattern), and a later process distills logs into `MEMORY.md` + topic files. This is different from standard mode’s immediate index maintenance.

---

## 10) Practical mental model

Think of memory as three cooperating layers:

1. **Policy layer**: prompt instructions define what should/shouldn’t be stored and when to trust/verify.
2. **Storage layer**: Markdown files in structured directories (+ optional team sync).
3. **Recall layer**: lightweight scan + model ranker + bounded attachment injection with freshness guardrails.

This design keeps memory persistent and useful while minimizing stale or over-broad recall.

---

## 11) Is there a local DB / index?

Short answer: **no dedicated local database/vector index** for memory retrieval in this path.

What exists locally:

- memory persisted as Markdown files on disk,
- light frontmatter/header scan (`scanMemoryFiles`) over `.md` files,
- in-memory dedup/state guards (`alreadySurfaced`, `readFileState`) during a turn.

What is LLM-driven:

- selecting which scanned memories are relevant (`selectRelevantMemories`) is delegated to a side model call with a strict JSON schema output.

So the architecture is: **file system + lightweight local scan + model-based rank/select**, not “send every memory file to the main model each turn.”

---

## 12) Concrete, code-based processing example

Assume:

- user prompt: “Why did we stop mocking DB in tests?”
- memory directory has:
  - `feedback_testing.md` (description: “integration tests must hit real DB”),
  - `reference_dashboards.md`,
  - `project_release_freeze.md`.

### Step-by-step

1. **System prompt includes memory policy/instructions**
   - Memory section is assembled via `loadMemoryPrompt()` and inserted as `systemPromptSection('memory', ...)`.

2. **At turn start, memory prefetch begins in parallel**
   - `queryLoop()` starts `startRelevantMemoryPrefetch(...)` once per turn.

3. **Local scan of candidate files**
   - `findRelevantMemories(...)` calls `scanMemoryFiles(...)`.
   - It recursively lists `.md` files, excludes `MEMORY.md`, reads only header/frontmatter slices, and builds a manifest.

4. **LLM side-query selects up to 5 likely files**
   - `selectRelevantMemories(...)` sends `Query + manifest` to a side Sonnet call.
   - Output must match JSON schema `{ selected_memories: string[] }`.

5. **Selected files are read with hard limits**
   - `readMemoriesForSurfacing(...)` loads chosen files with line/byte truncation rules.
   - If truncated, Claude gets a note pointing to FileReadTool for full content.

6. **Attachment injection (non-blocking)**
   - On collection point, if prefetch is already settled, `query.ts` injects `relevant_memories` attachments.
   - If not settled yet, it skips without waiting and may consume next iteration.

7. **Main model sees only selected memory snippets, not the whole corpus**
   - In this example, likely `feedback_testing.md` is attached;
   - unrelated memory files remain unsent for this turn.

8. **Freshness warning guards stale claims**
   - If selected memory is old, header includes staleness warning text to verify against current code.

This gives a practical behavior: memory retrieval is selective and bounded, with local file scanning plus model ranking, rather than brute-force full-memory context stuffing.
