---
title: pi-agent-core — Stateful Agent with Tool Execution and Event Streaming
tags: [agent, ai, llm, tool-calling, streaming, events, typescript]
category: pi-package
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# pi-agent-core — Stateful Agent with Tool Execution and Event Streaming

---

## 💡 概述

Stateful agent with tool execution and event streaming. Built on `@earendil-works/pi-ai`.

---

## ⚡ 功能

- Stateful agent with tool execution, event streaming, and steering/follow-up queues
- AgentMessage type with declaration merging for custom message types
- Flexible tool system with parallel/sequential execution modes
- `beforeToolCall` / `afterToolCall` hooks for tool interception
- `prepareRequest` and `finishTurn` lifecycle hooks
- Steering and follow-up message queues for interrupting or queueing work mid-run
- Session-based model lookup, thinking budgets, and dynamic API key resolution
- Low-level `agentLoop()` / `agentLoopContinue()` API for direct control
- Proxy streaming support for browser apps
- Integration with `@earendil-works/pi-mcp` for MCP tools and `@earendil-works/pi-codemode` for model-written scripts

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

### AgentMessage vs LLM Message

The agent works with `AgentMessage`, a flexible type that can include:
- Standard LLM messages (`user`, `assistant`, `toolResult`)
- Custom app-specific message types via declaration merging

LLMs only understand `user`, `assistant`, and `toolResult`. The `convertToLlm` function bridges this gap by filtering and transforming messages before each LLM call.

### Message Flow

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                    (optional)                           (required)
```

1. **transformContext**: Prune old messages, inject external context
2. **convertToLlm**: Filter out UI-only messages, convert custom types to LLM format

### Event Flow

The agent emits events for UI updates. Understanding the event sequence helps build responsive interfaces.

#### prompt() Event Sequence

When you call `prompt("Hello")`:

```
prompt("Hello")
├─ agent_start
├─ turn_start
├─ message_start   { message: userMessage }      // Your prompt
├─ message_end     { message: userMessage }
├─ message_start   { message: assistantMessage } // LLM starts responding
├─ message_update  { message: partial... }       // Streaming chunks
├─ message_update  { message: partial... }
├─ message_end     { message: assistantMessage } // Complete response
├─ turn_end        { message, toolResults: [] }
└─ agent_end       { messages: [...] }
```

#### With Tool Calls

If the assistant calls tools, the loop continues:

```
prompt("Read config.json")
├─ agent_start
├─ turn_start
├─ message_start/end  { userMessage }
├─ message_start      { assistantMessage with toolCall }
├─ message_update...
├─ message_end        { assistantMessage }
├─ tool_execution_start  { toolCallId, toolName, args }
├─ tool_execution_update { partialResult }           // If tool streams
├─ tool_execution_end    { toolCallId, result }
├─ message_start/end  { toolResultMessage }
├─ turn_end           { message, toolResults: [toolResult] }
│
├─ turn_start                                        // Next turn
├─ message_start      { assistantMessage }           // LLM responds to tool result
├─ message_update...
├─ message_end
├─ turn_end
└─ agent_end
```

Tool execution mode is configurable:

- `parallel` (default): preflight tool calls sequentially, execute allowed tools concurrently, emit `tool_execution_end` as soon as each tool is finalized, then emit toolResult messages and `turn_end.toolResults` in assistant source order
- `sequential`: execute tool calls one by one, matching the historical behavior

The `beforeToolCall` hook runs after `tool_execution_start` and validated argument parsing. It can block execution and attach `terminate: true` to the blocked result. The `afterToolCall` hook runs after tool execution finishes and before `tool_execution_end` and final tool result message events are emitted.

Tools, blocked `beforeToolCall` results, and `afterToolCall` overrides can return `terminate: true` to hint that the automatic follow-up LLM call should be skipped. The loop only stops early when every finalized tool result in that batch sets `terminate: true`. Mixed batches continue normally.

### Request preparation and turn finalization

`prepareRequest` runs immediately before every conversational provider request, including the first. Use it to install canonical persisted context after pending input has been emitted. `prepareRequest` does not poll queues.

`finishTurn` runs after the assistant and all tool results are finalized, but before `turn_end`. It runs for normal, error, and aborted responses. Returning `undefined` preserves normal scheduling. `{ action: "end" }` stops immediately after `turn_end`, before polling steering or follow-up queues or preparing another request. On a normal response, `{ action: "continue" }` ensures one next provider request.

Each provider turn follows this lifecycle:

```text
selected input events
→ prepareRequest
→ provider response
→ tool results
→ finishTurn
→ turn_end
→ existing continuation scheduling or agent_end
```

### continue() and queued input

`continue()` retains its existing queue behavior. Empty and system-only transcripts reject without consuming queues. A non-assistant tail continues from existing context: steering is polled at startup, while follow-up input waits until the response naturally stops. An assistant tail cannot be sent directly, so `continue()` falls back to one queued steering batch, then one queued follow-up batch.

### Event Types

| Event | Description |
|-------|-------------|
| `agent_start` | Agent begins processing |
| `agent_end` | Final event for the run. Awaited subscribers for this event still count toward settlement |
| `turn_start` | New turn begins (one LLM call + tool executions) |
| `turn_end` | Turn completes with assistant message and tool results |
| `message_start` | Any message begins (user, assistant, toolResult) |
| `message_update` | **Assistant only.** Includes `assistantMessageEvent` with delta |
| `message_end` | Message completes |
| `tool_execution_start` | Tool begins |
| `tool_execution_update` | Tool streams progress |
| `tool_execution_end` | Tool completes |

### Agent Options

```typescript
const agent = new Agent({
  initialState: {
    systemPrompt: string,
    model: Model<any>,
    thinkingLevel: "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max",
    tools: AgentTool<any>[],
    messages: AgentMessage[],
  },
  convertToLlm: (messages) => messages.filter(...),
  transformContext: async (messages, signal) => pruneOldMessages(messages),
  steeringMode: "one-at-a-time",
  followUpMode: "one-at-a-time",
  streamFn: models.streamSimple.bind(models),
  sessionId: "session-123",
  getApiKey: async (provider) => refreshToken(),
  toolExecution: "parallel",
  beforeToolCall: async ({ toolCall, args, context }) => {
    if (toolCall.name === "bash") {
      return { block: true, reason: "bash is disabled", terminate: true };
    }
  },
  afterToolCall: async ({ toolCall, result, isError, context }) => {
    if (toolCall.name === "notify_done" && !isError) {
      return { terminate: true };
    }
    if (!isError) {
      return { details: { ...result.details, audited: true } };
    }
  },
  prepareRequest: async ({ context }, signal) => {
    return { context: { ...context, messages: await loadCanonicalMessages(signal) } };
  },
  finishTurn: async ({ message, toolResults }, signal) => {
    return shouldContinue(message, toolResults) ? { action: "continue" } : undefined;
  },
  thinkingBudgets: {
    minimal: 128,
    low: 512,
    medium: 1024,
    high: 2048,
  },
});
```

### Agent State

```typescript
interface AgentState {
  model: Model<any>;
  thinkingLevel: ThinkingLevel;
  tools: AgentTool<any>[];
  messages: AgentMessage[];
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
```

The transcript owns the system prompt and tool declarations: the leading system message is the prompt, later system messages patch it (see `SystemMessage` in pi-ai). `agent.state.systemPrompt` is read-only and replays the transcript. `agent.state.tools` is the executable loadout; before every request the loop diffs it against the tools the transcript declares and, if they differ, announces the change in a system message.

### Steering and Follow-up

Steering messages let you interrupt the agent while tools are running. Follow-up messages let you queue work after the agent would otherwise stop.

When steering messages are detected after a turn completes:
1. All tool calls from the current assistant message have already finished
2. Steering messages are injected
3. The LLM responds on the next turn

Follow-up messages are checked only when there are no more tool calls and no steering messages. If any are queued, they are injected and another turn runs.

### Custom Message Types

Extend `AgentMessage` via declaration merging:

```typescript
declare module "@earendil-works/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string; timestamp: number };
  }
}
```

### Tools

Define tools using `AgentTool`. Tools can specify `executionMode: "sequential"` to force the entire batch to run one at a time, or `"parallel"` to allow concurrent execution. Throw an error when a tool fails — do not return error messages as content. Return `terminate: true` to hint that the agent should stop after the current tool batch.

`@earendil-works/pi-mcp` connects to MCP servers and `@earendil-works/pi-codemode` runs model-written JavaScript that calls tools. [examples/mcp-codemode](examples/mcp-codemode) wraps both as `AgentTool`s.

### Low-Level API

For direct control without the Agent class:

```typescript
import { agentLoop, agentLoopContinue } from "@earendil-works/pi-agent-core";

const context: AgentContext = {
  messages: [{ role: "system", content: "You are helpful.", timestamp: Date.now() }],
  tools: [],
};

const config: AgentLoopConfig = {
  model: getModel("openai", "gpt-4o"),
  convertToLlm: (msgs) => msgs.filter(m => ["user", "assistant", "toolResult"].includes(m.role)),
  toolExecution: "parallel",
  beforeToolCall: async ({ toolCall, args, context }) => undefined,
  afterToolCall: async ({ toolCall, result, isError, context }) => undefined,
};

const userMessage = { role: "user", content: "Hello", timestamp: Date.now() };
const streamFn = models.streamSimple.bind(models);
for await (const event of agentLoop([userMessage], context, config, undefined, streamFn)) {
  console.log(event.type);
}

for await (const event of agentLoopContinue(context, config, undefined, streamFn)) {
  console.log(event.type);
}
```

These low-level streams are observational. They preserve event order, but they do not wait for your async event handling to settle before later producer phases continue. If you need message processing to act as a barrier before tool preflight, use the `Agent` class instead of raw `agentLoop()` or `agentLoopContinue()`.

---

## 📁 目录

「待補充」

---

## 🔗 资源

「待補充」

---

## 🚀 快速开始

```bash
npm install @earendil-works/pi-agent-core
```

```typescript
import { Agent } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-6");
if (!model) throw new Error("Model not found");

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model,
  },
  streamFn: models.streamSimple.bind(models),
});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await agent.prompt("Hello!");
```

### Prompting

```typescript
// Text prompt
await agent.prompt("Hello");

// With images
await agent.prompt("What's in this image?", [
  { type: "image", data: base64Data, mimeType: "image/jpeg" }
]);

// AgentMessage directly
await agent.prompt({ role: "user", content: "Hello", timestamp: Date.now() });

// Continue existing non-assistant input
await agent.continue();
```

### State Management

```typescript
agent.state.model = getModel("openai", "gpt-4o");
agent.state.thinkingLevel = "medium";
agent.state.tools = [myTool];
agent.toolExecution = "sequential";
agent.beforeToolCall = async ({ toolCall }) => undefined;
agent.afterToolCall = async ({ toolCall, result }) => undefined;
agent.prepareRequest = async ({ context }) => ({
  context: { ...context, messages: await loadCanonicalMessages() },
});
agent.finishTurn = async () => undefined;
agent.state.messages = newMessages;
agent.state.messages.push(message);
const nextQueuedMessages = agent.peekQueuedMessages();
agent.reset();
```

### Control

```typescript
agent.abort();
await agent.waitForIdle();
```

### Events

```typescript
const unsubscribe = agent.subscribe(async (event, signal) => {
  if (event.type === "agent_end") {
    await flushSessionState(signal);
  }
});
unsubscribe();
```

### Proxy Usage

For browser apps that proxy through a backend:

```typescript
import { Agent, streamProxy } from "@earendil-works/pi-agent-core";

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: "...",
      proxyUrl: "https://your-server.com",
    }),
});
```

---

## ⚙️ 技术栈

- TypeScript
- Built on `@earendil-works/pi-ai` for model abstraction
- Integrates with `@earendil-works/pi-mcp` (MCP servers) and `@earendil-works/pi-codemode` (QuickJS sandbox)
- License: MIT