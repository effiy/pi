---
title: pi-codemode — Model-Written JavaScript in a QuickJS VM Sandbox
tags: [codemode, quickjs, sandbox, wasm, javascript, agent, tool-calling]
category: pi-package
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# pi-codemode — Model-Written JavaScript in a QuickJS VM Sandbox

---

## 💡 概述

Runs model-written JavaScript in a QuickJS VM (compiled to WebAssembly) where the only capability is calling injected tools. Nested tool calls never enter the LLM context; only the script's output and return value do.

Scripts use `tools`, `ALL_TOOLS`, `text`, `image`, `exit`, `store`, and `load`, and may start with a `// @options:` line. The coding agent uses it for its built-in `codemode` tool. It has no pi dependencies and can be used on its own to expose any functions (remote APIs, MCP servers, application services) to model-written scripts.

---

## ⚡ 功能

- QuickJS VM sandboxed in WebAssembly; scripts have no access to timers, `fetch`, `process`, modules, or `WebAssembly`
- Tool injection: scripts call `tools.<name>(args)` with arguments and results making a JSON round trip
- `text()` and `image()` for output, `console.log` support, `exit()` for early termination
- `store(key, value)` and `load(key)` for synchronous JSON persistence across executions
- Script source format with `// @options:` header for `max_output_tokens` and `timeout_ms`
- `renderDeclarations()` generates TypeScript-style declarations for model-facing tool descriptions
- Deterministic runs via `sandbox.start()` with fixed time, seed, and result ordering
- Three sandbox environments: worker thread (Node/Bun), remote Durable Object (Cloudflare), inline (anywhere)
- Integration with pi-agent-core via `AgentTool` wrapping

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

### Sandbox Environments

The package runs in two kinds of environments. Each has its own sandbox class, but they share one API, one script environment, and one result format:

| Environment | Sandbox | Import from | Where scripts run |
| --- | --- | --- | --- |
| Node and Bun | `CodemodeSandbox` | `@earendil-works/pi-codemode` | a worker thread per execution |
| Cloudflare Workers and Durable Objects | `RemoteCodemodeSandbox` | `@earendil-works/pi-codemode/portable` | a Durable Object in a separately deployed Worker |

A third sandbox, `InlineCodemodeSandbox`, runs scripts on the calling thread for other runtimes, or on Cloudflare when blocking the caller is acceptable.

### Script Environment

Inside the script, with every sandbox:
- `tools.<name>(args)` returns a promise. Arguments and results make a JSON round trip. A tool that throws rejects with an `Error` carrying the same message.
- `ALL_TOOLS` lists `{ name, description }` for every tool, with `name` as the identifier.
- `text(value)` appends a text item to `result.output`; values other than strings are JSON-stringified. `console.log/info/warn/error/debug` append text items with `console: true`.
- `image(urlOrItem)` appends an image item. It accepts a base64 `data:` URL, `{ image_url }`, or an image block.
- `exit()` ends the script successfully right away, keeping its output and store writes.
- `globals` passed to the sandbox are called as top-level functions.
- `store(key, value)` and `load(key)` read and write JSON values synchronously.
- Nothing else: no timers, `fetch`, `process`, `require`, modules, or `WebAssembly`.

`timeoutMs: Infinity` disables the deadline. A script that waits on a promise nothing can settle fails right away instead of hanging.

`memoryLimitBytes` caps the VM's heap. Allocations beyond it fail inside the script as `InternalError: out of memory`.

### Store

`store`/`load` let scripts keep values across executions. The sandbox does not persist anything itself: pass the current values as `options.store`, and a successful result reports what the script changed as `result.storeWrites` (`{ set, delete }`). Failed executions report no writes.

`load` returns a copy. Storing `undefined` deletes the key. A value may be at most `MAX_STORE_VALUE_CHARS` (256 Ki) characters of JSON and all values together at most `MAX_STORE_TOTAL_CHARS` (1 Mi); larger writes throw a `RangeError` inside the script.

### Source Format

`parseCodemodeSource()` accepts a script whose first line may be an options line:

```js
// @options: {"max_output_tokens": 2000, "timeout_ms": 30000}
const source = await tools.read({ path: "package.json" });
text(JSON.parse(source).name);
```

Supported fields are `max_output_tokens`, a token budget for the output, and `timeout_ms`, a hard deadline. The sandbox does not act on them; the caller decides. The options line is replaced by an empty line, so line numbers in stack traces still match the input.

### Bundled Node and Bun Hosts

By default `CodemodeSandbox` loads `quickjs-wasi/quickjs.wasm` from the installed package and starts the worker file that sits next to this package's module. Neither exists on disk when the host is bundled, so pass both `wasm` and `workerUrl`.

### Cloudflare Workers: Why a Separate Worker

QuickJS runs synchronously, so a script that computes holds its thread until it awaits a tool. In a Durable Object of a separately deployed Worker, the delay is about 50 ms. `RemoteCodemodeSandbox` therefore keeps the tools and call records in the session and sends only the VM to the sandbox Worker. Each tool call travels back to the session over Workers RPC.

### Cloudflare Limits

- **Interrupt budget.** While a script computes without calling a tool, no timer can run, and `Date.now()` does not advance. A script that never yields is limited only by the interrupt budget, not by `timeoutMs`. The default `DEFAULT_INTERRUPT_BUDGET` of 100,000 polls is about 2 seconds of a tight loop on an Apple M-series laptop and about 10 seconds on Cloudflare.
- **Memory.** Each VM may use up to 32 MiB by default. All VMs in one isolate share its 128 MB.
- **Aborts and timeouts.** `timeoutMs` and `signal` end the execution in the caller right away. The remote VM stops at its next tool call.
- **Billing.** Each execution is one Durable Object request. The sandbox object is billed for wall-clock duration while an execution runs, including time spent waiting on tools.

### On the Calling Thread

When blocking the caller is acceptable, run the VM on the caller's thread instead with `InlineCodemodeSandbox`. `createInlineCodemodeSandbox()` from `@earendil-works/pi-codemode/cloudflare` passes the QuickJS module Wrangler compiled. QuickJS's stack guard is reduced to `INLINE_MAX_STACK_SIZE` so it trips before the caller's smaller stack runs out.

### Deterministic Runs

`sandbox.start(code, options)` runs a script so that a host can run it again after a restart and get the same behavior. The VM only runs when the host hands it something, so it is deterministic except for what it reads from its environment:
- **Time:** `Date.now()` reads `startedAt`, then the `now` of the result being processed, never going backwards.
- **Randomness:** `Math.random()` and the VM's other randomness come from `seed`.
- **Timezone:** UTC, whatever the host's.
- **Garbage collection:** `WeakRef` and `FinalizationRegistry` do not exist.
- **Result order:** the host answers each call with a result number (1, 2, ...), and the VM applies results strictly in that order.

The same script, `startedAt`, `seed`, tools, globals, and store, answered with the same results in the same order, makes the same calls with the same arguments and produces the same output on any VM with the same fingerprint.

### Results

`execute()` never rejects for script failures. `result.error.kind` is one of:

| kind | meaning |
| ---- | ------- |
| `script` | the script threw or failed to parse; `stack` points at `codemode.js:<line>` |
| `timeout` | the deadline expired or the interrupt budget ran out; the VM was stopped |
| `aborted` | `options.signal` fired or `close()` was called; the VM was stopped |
| `sandbox` | the VM or its transport failed, for example a wasm trap, a missing worker file, or a failed remote |

`result.output` holds the text and image items in the order the script produced them, also for failed executions. The host keeps all of it until the script ends, so output is limited to `MAX_OUTPUT_CHARS` (16 Mi) characters and `MAX_OUTPUT_ITEMS` (100000) items. `result.calls` lists every tool call with `status: "ok" | "error" | "cancelled"`.

### How It Works (CodemodeSandbox)

Each `execute()` starts a worker thread (about 20 ms including VM creation) that instantiates a fresh QuickJS VM from the compiled wasm module. The VM is a separate wasm instance with its own linear memory. Its only imports are a WASI shim and one host-call entry point.

The worker evaluates a prelude inside the VM that holds the single host bridge in a closure and builds `tools`, `console`, and globals on top of it. The script is compiled as an async function body.

Tool calls are relayed to the host thread as messages; the host runs the tool and posts the JSON result back. The host owns the deadline and the abort signal. When either fires, it sets a shared interrupt flag that the VM polls, then calls `worker.terminate()`.

### Runtime Notes

- Works the same on Node and Bun, including memory limits, interrupts, and a catchable `RangeError` for deep recursion.
- Stack traces use QuickJS frames (`at f (codemode.js:2:31)`), prefixed with `Name: message` like V8.
- QuickJS is an interpreter. Glue code and filtering tool results are fast enough; heavy computation is slower than in V8.

---

## 📁 目录

「待補充」

---

## 🔗 资源

- [Cloudflare example](https://github.com/earendil-works/pi/tree/main/packages/codemode/examples/cloudflare)
- [mcp-codemode example](https://github.com/earendil-works/pi/tree/main/packages/agent/examples/mcp-codemode)

---

## 🚀 快速开始

### Usage on Node and Bun

```ts
import { CodemodeSandbox } from "@earendil-works/pi-codemode";

const sandbox = new CodemodeSandbox({
  timeoutMs: 60_000,
  tools: [
    {
      name: "read",
      execute: async (args, { signal }) => {
        const { path } = args as { path: string };
        return await readFile(path, "utf8");
      },
    },
  ],
});

const result = await sandbox.execute(`
  const source = await tools.read({ path: "package.json" });
  text("bytes " + source.length);
  return JSON.parse(source).name;
`);

console.log(result.output); // [{ type: "text", text: "bytes 1234" }]
if (result.ok) console.log(result.value); // "@earendil-works/pi-codemode"
else console.error(result.error.kind, result.error.message);

await sandbox.close();
```

### Declarations for the Model

```ts
renderDeclarations({ tools: sandbox.tools, globals: sandbox.globals });
// declare const tools: {
//   /** Read a file */
//   read(args: {
//     path: string;
//   }): Promise<string>;
// };
```

### Using with pi-agent-core

Wrap `execute()` as an `AgentTool`:

```ts
const codemodeTool: AgentTool = {
  name: "codemode",
  label: "Codemode",
  description: `Run JavaScript that calls tools as \`await tools.<name>(args)\`.\n\n${renderDeclarations({ tools: sandboxTools })}`,
  parameters: Type.Object({ code: Type.String() }),
  execute: async (_toolCallId, { code }, signal) => {
    const sandbox = new CodemodeSandbox({ tools: sandboxTools });
    try {
      const result = await sandbox.execute(code, { signal });
      const content = [...result.output];
      if (result.ok && result.value !== undefined) content.push({ type: "text", text: JSON.stringify(result.value) });
      if (!result.ok) content.push({ type: "text", text: result.error.stack ?? result.error.message });
      return { content, details: undefined, isError: !result.ok };
    } finally {
      await sandbox.close();
    }
  },
};
```

### Deterministic Run

```ts
const run = sandbox.start(code, {
  startedAt: Date.now(),
  seed: 7,
  tools: [{ name: "read", description: "Read a file" }],
  onStarted: (fingerprint) => {},
  onCall: (call) => {
    void readFile(call.args).then((text) => run.deliver(nextResult++, Date.now(), call.id, { ok: true, value: text }));
  },
  onIdle: (seq, issued) => {},
});
const result = await run.result;
```

### Bundled Hosts

```ts
import { CodemodeSandbox, loadQuickJSWasm } from "@earendil-works/pi-codemode";

const sandbox = new CodemodeSandbox({
  tools,
  wasm: loadQuickJSWasm(pathToQuickJSWasm),
  workerUrl: new URL("./codemode-worker.js", import.meta.url),
});
```

### Cloudflare: Deploying the Sandbox Worker

```ts
// sandbox/src/index.ts
export { CodemodeSandboxDurableObject as CodemodeSandbox } from "@earendil-works/pi-codemode/cloudflare";
export default { fetch: () => new Response("Not found", { status: 404 }) };
```

```jsonc
// sandbox/wrangler.jsonc
{
  "name": "pi-codemode-sandbox",
  "main": "src/index.ts",
  "compatibility_date": "2026-04-22",
  "workers_dev": false,
  "preview_urls": false,
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["CodemodeSandbox"] }],
  "limits": { "cpu_ms": 60000 }
}
```

### Cloudflare: Using It from the Agent's Worker

```ts
import type { CodemodeSandboxDurableObject } from "@earendil-works/pi-codemode/cloudflare";
import { RemoteCodemodeSandbox } from "@earendil-works/pi-codemode/portable";

interface Env {
  CODEMODE_SANDBOX: DurableObjectNamespace<CodemodeSandboxDurableObject>;
}

const sandbox = new RemoteCodemodeSandbox({
  tools,
  timeoutMs: 60_000,
  remote: () => env.CODEMODE_SANDBOX.getByName(sessionId),
});
const result = await sandbox.execute(code, { signal, store });
```

### Cloudflare: Inline Sandbox

```ts
import { createInlineCodemodeSandbox } from "@earendil-works/pi-codemode/cloudflare";

const sandbox = createInlineCodemodeSandbox({ tools, timeoutMs: 60_000, interruptBudget: 50_000 });
const result = await sandbox.execute(code);
```

---

## ⚙️ 技术栈

- TypeScript
- QuickJS compiled to WebAssembly (~640 KB)
- Worker threads for Node/Bun isolation
- Cloudflare Workers and Durable Objects for edge deployment
- Zero pi dependencies (standalone package)
- License: MIT