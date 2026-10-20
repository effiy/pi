---
title: Pi Durable - Durable Agent Harness
tags: [durable, agent, harness, storage, persistence, conversation]
category: library
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi Durable

---

## 💡 概述

A durable agent harness. Conversations, model turns, tool calls, and your own state are committed to storage before anything is shown. If the process dies mid-turn, reopening the storage picks the work up where it stopped.

Built on [`@earendil-works/pi-ai`](../ai/README.md) for model access and `@earendil-works/chord` for document state.

---

## ⚡ 功能

- **Harness**: one open storage plus the machinery that runs agents on it. All changes go through one line of atomic commits, and nothing is shown before its commit is stored.
- **Conversation**: a transcript. `root()` creates the root conversation on first use; you can create more and fork them. A `Conversation` handle holds no state; compare handles by `id`.
- **Entry**: one immutable transcript record, such as a user message (`pi.user`), a model response (`pi.assistant`), a tool result (`pi.tool-result`), a system prompt change (`pi.system`), a reset (`pi.reset`), or your own kind. The model sees the entries from the newest reset onward.
- **Commit**: an atomic write. `conversation.commit((tx) => ...)` can append entries, edit documents, and create tasks together; either all of it is stored or none of it.
- **Document**: typed JSON state stored next to the transcript and changed in commits. Built-in ones hold each conversation's agent choices (`pi.agent`), provider-facing session identity (`pi.provider`), running generation and tools (`pi.live`), queued submissions (`pi.inbox`), and spend (`pi.usage`).
- **Task**: a durable state machine that saves a checkpoint at every step, so a restarted process continues from the last one.
- **Submission**: something you hand to a conversation, either user input or an entry to write, which you can wait for.
- **Extension**: a named bundle of tools, system prompt sections, hooks, wrappers, and tasks.
- **Registry**: the extensions this process installed. It can change while the Harness runs; new work uses the new state.
- **Agent**: what a conversation runs with: model, thinking level, selected extensions, tools, instructions, and working directory.

One answered input, as entries and tasks:

```
submit(input) -> pi.user
  pi.generation -> pi.system (only if the prompt or tools changed), pi.assistant (tool calls)
    pi.tool x n -> pi.tool-result x n
  pi.generation -> pi.assistant (answer) -> submission done
```

---

## 📊 指標

「待補充」

---

## 🏛️ 設計

### Concepts

- **Harness**: one open storage plus the machinery that runs agents on it.
- **Conversation**: a transcript; `root()` creates the root conversation on first use.
- **Entry**: one immutable transcript record (user message, model response, tool result, system prompt change, reset, or custom).
- **Commit**: an atomic write that appends entries, edits documents, and creates tasks together.
- **Document**: typed JSON state stored next to the transcript.
- **Task**: a durable state machine that saves a checkpoint at every step.
- **Submission**: input or entry to a conversation that can be waited for.
- **Turn and run**: a turn is one model response and its tool calls; a run is the turns from an input to its final answer.
- **Extension**: a named bundle of tools, system prompt sections, hooks, wrappers, and tasks.
- **Registry**: the extensions this process installed; conversations store extension names.
- **Agent**: model, thinking level, selected extensions, tools, instructions, and working directory per conversation.

### Storage backends

| Backend | Import | Notes |
|---|---|---|
| Memory | `MemoryStorage` from the package root | Nothing is persisted. |
| SQLite | `openNodeSqliteStorage(file)` from `/storage/sqlite/node` | One database file. WAL mode with `synchronous = NORMAL`. |
| JSONL | `openNodeJsonlStorage(directory, context)` from `/storage/jsonl/node` | Append-only files. Keeps whole history in memory. |
| Cloudflare Durable Object | `openDurableObjectSqliteStorage(ctx.storage)` from `/storage/sqlite/cloudflare` | SQLite storage of one Durable Object. |

### Hooks

- **Generation:** `beforeRequest`, `afterResponse`, `onYield`, `afterTools`
- **Tools:** `beforeTool`, `afterTool`

### Settings

Run policy shared by every conversation: extensions, stream timeout, retry, compaction, progress intervals, tool execution mode, context and document retention, followUpMode.

### Code Mode

`@earendil-works/pi-durable/codemode` adds a `codemode` tool: the model writes JavaScript that calls the conversation's tools as `await tools.<name>(args)`, in a QuickJS sandbox (`@earendil-works/pi-codemode`). Each call is a nested tool call, and only the script's output reaches the model. The script survives a restart.

### Agent Events (Experimental)

For consumers that want coding-agent style events instead of structural state, `watchEvents()` delivers `message_start`, `message_update`, `tool_execution_start`, etc.

---

## 📁 目录

「待補充」

---

## 🔗 资源

- [`docs/spec.md`](https://github.com/earendil-works/pi/blob/main/packages/durable/docs/spec.md): the normative specification
- Benchmarks: `npm run bench:storage`, `npm run bench:storage:memory`, `npm run bench:tool-output`, `npm run bench:nested-tools`

---

## 🚀 快速开始

```bash
npm install @earendil-works/pi-durable @earendil-works/pi-ai @earendil-works/chord
```

```typescript
import { BACKGROUND_CONTEXT } from "@earendil-works/chord/context";
import { createModels } from "@earendil-works/pi-ai/models";
import { openaiProvider } from "@earendil-works/pi-ai/providers/openai";
import { AssistantEntry, createRegistry, Harness, MemoryStorage } from "@earendil-works/pi-durable";

const context = BACKGROUND_CONTEXT;

const models = createModels();
models.setProvider(openaiProvider()); // reads OPENAI_API_KEY

const harness = await Harness.open(new MemoryStorage(), { models, registry: createRegistry() }, context);
const root = await harness.root(context, { agent: { model: { provider: "openai", modelId: "gpt-6-sol" } } });

const submission = await root.submit({ type: "input", content: "What is the capital of France?" }, context);
const settled = await submission.wait(context);
if (settled.status === "done" && settled.type === "input") {
    const answer = await root.commit((tx) => tx.entry(AssistantEntry, settled.answer), context);
    console.log(answer?.model?.[0]);
}
await harness.close(context);
```

### Persist and Resume

```typescript
import { openNodeSqliteStorage } from "@earendil-works/pi-durable/storage/sqlite/node";

const harness = await Harness.open(await openNodeSqliteStorage("./session.sqlite"), { models, registry }, context);
const root = await harness.root(context); // the same root as last time
harness.resume(); // continue any run the last process left unfinished
```

### Examples

Runnable examples live in [`test/examples`](test/examples). Run one from this package directory with:

```bash
node --conditions=source --experimental-strip-types test/examples/14-chat.ts
```

| Example | Shows |
|---|---|
| [14-chat](test/examples/14-chat.ts) | One question and answer |
| [16-real-model](test/examples/16-real-model.ts) | Streaming an answer from OpenAI |
| [17-coding-tools](test/examples/17-coding-tools.ts) | A tool-using turn on JSONL storage |
| [18-print](test/examples/18-print.ts) | Print mode: submit a prompt, print the answer |
| [19-json](test/examples/19-json.ts) | JSON mode: agent events or raw view operations |
| [20-inbox](test/examples/20-inbox.ts) | Steers, follow-ups, writes, and withdrawal while busy |
| [21-late-join](test/examples/21-late-join.ts) | Attaching a view and an event stream mid-run |
| [22-subagent-foreground](test/examples/22-subagent-foreground.ts) | A replay-safe subagent tool |
| [23-subagent-background](test/examples/23-subagent-background.ts) | Persistent subagents |
| [24-child-tasks](test/examples/24-child-tasks.ts) | A checkout that owns and waits for payments |
| [25-compaction](test/examples/25-compaction.ts) | Long chat compacted in background, manually, and after overflow |
| [26-coding-agent](test/examples/26-coding-agent.ts) | CodingTools, live settings, per-conversation environment |
| [27-plan-mode](test/examples/27-plan-mode.ts) | Read-only plan mode as an extension |
| [28-reviewer](test/examples/28-reviewer.ts) | Reviewer conversation with its own model and tools |
| [29-sandbox-per-conversation](test/examples/29-sandbox-per-conversation.ts) | Environment per conversation |
| [30-tool-override](test/examples/30-tool-override.ts) | Same-name bash and a wrapper |
| [31-reload-and-restart](test/examples/31-reload-and-restart.ts) | Reloading an extension mid-call |
| [00](test/examples/00-conversation.ts)-[13](test/examples/13-recovery.ts) | Layers: sessions, documents, forks, watches, Harness, agent config, reload, extensions, tasks, recovery |

---

## ⚙️ 技术栈

- TypeScript (Node strip-only mode)
- `@earendil-works/pi-ai` for model access
- `@earendil-works/chord` for document state
- `@earendil-works/pi-codemode` for QuickJS sandbox
- SQLite, JSONL, or Cloudflare Durable Objects for storage
- TypeBox for tool argument validation