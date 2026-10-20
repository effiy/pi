---
title: Chord — Distributed Objects with Hot Reload and Observable State
tags: [chord, distributed, state-management, hot-reload, ipc, websocket, typescript]
category: pi-package
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Chord — Distributed Objects with Hot Reload and Observable State

---

## 💡 概述

Chord builds applications out of **objects** that live in several processes (a server, workers, browser tabs) and can be **hot reloaded** while they run.

- An **object** is state plus code at an address. It is created on demand and lives while someone holds it.
- Everyone gets a **handle**, never the object. Handles work the same locally and across processes, and survive reloads.
- Every object knows its **principal**: an owner and a guest get different objects, with different methods.
- **State** is observable JSON. Remote viewers get a snapshot followed by deltas.
- **Hot reload** swaps code while it runs. Work in flight finishes on the code it started with, state is kept, and incompatible changes migrate at a safe point.

Contracts are runtime values described with [TypeBox](https://github.com/sinclairzx81/typebox) schemas. Hosts exchange JSON frames over links: in memory, a Node IPC channel, or a WebSocket.

---

## ⚡ 功能

- Distributed objects with handles that work identically locally and across processes
- Principal-based access: owner and guest get different objects with different methods
- Observable JSON state with snapshot + deltas for remote viewers
- Hot reload: swap code while running, with state preserved and safe migration
- Runtime contracts defined with TypeBox schemas
- Multiple link transports: in-memory, Node IPC, WebSocket
- Standalone transactional JSON delta tracker via `@earendil-works/chord/delta`
- Replicated state with `replicatedState()` for standalone observable values

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

### Entry Points

| Import | Contents | Runs in |
|--------|----------|---------|
| `@earendil-works/chord` | Contracts (`local`, `remote`, `method`, `opaque`), declarations (`cell`, `derivation`, `listener`, `resource`), hosts (`createHost`, `defineFacet`, `defineRegistry`), links (`memoryLinkPair`, `ipcLink`, `webSocketLink`, `acceptWebSockets`), errors (`ChordError`, `isChordError`, `forbidden`, `invalid`). Also the shared base: `Context`, `JsonValue`, `JsonRepresentation`, `Draft`, `copyJson`, `isJsonValue`, and `replicatedState` with its types. | Node and browsers |
| `@earendil-works/chord/node` | Child host processes: `createHostManager`, `forkDriver`, `childControl`, `processLock`; the facet module loader `nodeLoader`. | Node |
| `@earendil-works/chord/context` | Explicit cancellation and invocation values: `BACKGROUND_CONTEXT`, `withCancel`, `withAbortSignal`, `withoutAbortSignal`, `withContextValue`, `awaitWithContext`. | Node and browsers |
| `@earendil-works/chord/delta` | The transactional JSON tracker, structural diff, immutable apply, and the wire encoder/decoder for operation batches. | Node and browsers |

### Tracking JSON Deltas

Import the standalone transactional tracker from `@earendil-works/chord/delta`:

```ts
import { applyImmutable, track } from "@earendil-works/chord/delta";

const tracker = track({ output: "", count: 0 });
const change = tracker.beginChange();
change.state.output += "done\n";
change.state.count += 1;
const prepared = change.prepare();

tracker.adopt(prepared);
const replica = applyImmutable(prepared.base, prepared.ops);
```

`tracker.value` is always the latest adopted immutable revision. Preparation does not change authority; adoption validates the preparation and swaps the root pointer. Assigned containers are copied by value and unchanged subtrees may be shared between revisions.

The tracker uses trusted immutable ownership rather than defensive copying or freezing. `track(initial)`, `prepareReplace(value)`, `replicatedState(initial)`, and `replace(context, value)` take ownership of alias-free strict-JSON roots without walking them. Callers must not mutate transferred or published data.

Replicated state provides the same model through a callback. `replicatedState` is a standalone observable value, independent of hosts:

```ts
import { replicatedState } from "@earendil-works/chord";

const initial = { output: "", count: 0 };
const status = replicatedState(initial); // transfers ownership of initial
status.change(context, (draft) => {
  draft.output += "done\n";
  draft.count += 1;
});
```

A successful `change()` publishes exactly one atomic revision. If its callback throws, the original value and sequence remain unchanged. Draft handles become unusable when the callback returns. The tracker emits string append and front-truncate operations, array splices and permutations, sets, and deletes; large edit sets may fold into a complete replacement.

`replicatedState(source, { onError })` instead attaches a publication-only state to an authoritative revision source (`ReplicatedStateSource`): it captures a snapshot, then publishes every committed frame in cursor order.

### Public State Subscriptions

`state.subscribe(async (value, context, delivery) => { ... })` serializes callbacks independently for each subscription. Hydration starts immediately when a value is available, and its returned promise must settle before updates start. Synchronous callbacks still run synchronously.

Each subscription retains at most 100 pending complete-value deliveries, excluding the running callback. On overflow, only the newest pending value, context, and delivery metadata are retained; a not-yet-started initial hydration is preserved. Thus public delivery sequences may skip. This is a frame-count policy, not a byte limit.

The returned unsubscribe function immediately discards pending work and prevents new callbacks. It neither aborts nor waits for the running callback. Synchronous throws and promise rejections are observed, reported, and do not stop other subscriptions or subsequent callbacks.

---

## 📁 目录

「待補充」

---

## 🔗 资源

- [Tutorial](docs/tutorial.md): a shared todo list, built up section by section. Every example runs as `test/tutorial.test.ts`.
- [Usage guide](docs/guide.md): API reference for contracts, objects, state, roots, lifetime, plugins, hot reload, hosts, and links.
- [Specification](docs/spec.md): compatibility, lifetime, and reload semantics, plus known limitations and implementation decisions.
- [Delta guide](src/delta/README.md): ownership, lifecycle, operation, and replica contracts of `@earendil-works/chord/delta`.

---

## 🚀 快速开始

```ts
import { Type } from "typebox";
import { cell, createHost, method, remote } from "@earendil-works/chord";

// contract.ts: shared by provider and consumers
export const Counter = remote("demo.counter", {
  version: 1,
  input: Type.Object({ name: Type.String() }),
  variants: {
    default: {
      state: { count: Type.Number() },
      methods: { increment: method(Type.Object({ by: Type.Number() }), Type.Null()) },
    },
  },
});

const host = createHost({ name: "server", principalKey: (p) => p.kind });

host.serve(Counter, ({ name }, _principal, self) => {
  const state = self.state({ count: cell(Type.Number(), 0) });
  return {
    kind: "default",
    count: state.count,
    increment: async ({ by }, ctx) => {
      state.count.change(ctx, (n) => n + by);
      return null;
    },
  };
});

await host.root(async (ctx) => {
  const clicks = ctx.get(Counter, { name: "clicks" }); // a handle; held until this root ends
  clicks.count.subscribe((n) => console.log("count", n));
  await clicks.increment({ by: 2 }, ctx); // logs 0, then 2
});
```

---

## ⚙️ 技术栈

- TypeScript
- TypeBox for schema definitions
- JSON frames over in-memory, IPC, or WebSocket links
- Node.js and browser compatible
- License: MIT