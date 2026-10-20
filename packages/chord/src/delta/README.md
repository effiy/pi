---
title: Chord Delta — Immutable JSON Revisions and Exact Operation Batches
tags: [chord, delta, json, immutable, operations, replication, tracker]
category: pi-package
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Chord Delta — Immutable JSON Revisions and Exact Operation Batches

---

## 💡 概述

Chord Delta produces immutable JSON revisions and exact operation batches for ordered replicas. Import it from `@earendil-works/chord/delta`.

Immutability is an ownership contract. Nothing is frozen or defensively copied, so an illegal mutation is not detected. It silently corrupts state.

---

## ⚡ 功能

- Immutable JSON revisions with structural sharing between revisions
- Exact operation batches: string append/truncate, array splice/permute, set, delete, replace
- Draft-based mutation model with preparation and adoption lifecycle
- Immutable replay (`applyImmutable`) for in-process replicas
- Mutable replay (`apply`) for detached replicas
- Wire encoder/decoder for operation batches with path interning
- Trusted ownership model (no freezing, no defensive copying)
- No-op detection and deduplication

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

### Basic Usage

```ts
import { applyImmutable, applyImmutableBatches, track } from "@earendil-works/chord/delta";

const initial = { output: "", entries: [] as { id: number }[] };
const tracker = track(initial); // `initial` is transferred: never mutate it again

const change = tracker.beginChange();
change.state.output += "done\n"; // the draft is mutable only while the change is open
change.state.entries.push({ id: 1 }); // placed values are cloned; the caller keeps its object
const prepared = change.prepare(); // draft handles are unusable from here on

// tracker.value, prepared.base, prepared.value, and prepared.ops (including paths
// and payloads) are immutable by contract.
tracker.adopt(prepared); // tracker.value === prepared.value

// Shares containers with prepared.base and the op payloads: never mutate it either.
const replica = applyImmutable(prepared.base, prepared.ops);
```

### Mutation Rights

| Value | May it be mutated? |
| --- | --- |
| Root passed to `track()`, `prepareReplace()`, `replicatedState()`, or `replace()` | No. Ownership moved to the tracker. |
| `change.state` and handles read from it | Yes, only while that change is open. After `prepare()`, `abort()`, or another adoption, every use throws. |
| External value after assigning or inserting it into a draft | Yes. The draft stored a validated clone. |
| `tracker.value`, `prepared.base`, retained older revisions | No. |
| `prepared.value`, `prepared.ops`, op tuples, paths, permutations, payloads | No. Payloads may be the same objects as parts of `prepared.value`. |
| `applyImmutable()` / `applyImmutableBatches()` inputs and result | No. The result shares containers with both inputs. |
| Values from replicated state and cells | No. In-process consumers may share the provider's containers. |
| Mutable replica passed to `apply()` | Only through `apply()`, with batches it owns exclusively. |
| Batch passed to `apply()` | Consumed. `apply()` adopts payload containers into the replica. |

### Replaying Batches

Immutable replay is the preferred in-process path. One batch can fan out to any number of immutable replicas:

```ts
let replica = tracker.value; // shared with authority; never mutated
// For each adopted batch, in order:
replica = applyImmutable(replica, prepared.ops); // replica must equal prepared.base
```

When only the final result of an ordered backlog is needed, replay its batches without concatenating their operations:

```ts
replica = applyImmutableBatches(
  replica,
  queuedFrames.map((frame) => frame.ops),
);
```

Use separate `applyImmutable()` calls when every intermediate revision is published or retained.

Mutable replay needs a detached starting root and a detached copy of every batch for every replica. Never apply one in-memory batch to two mutable replicas:

```ts
import { apply } from "@earendil-works/chord/delta";

const detach = <T>(value: T): T => JSON.parse(JSON.stringify(value)) as T;

let replica = detach(tracker.value); // exclusively owned starting root
// For each adopted batch, in order:
replica = apply(replica, detach(prepared.ops)); // fresh copy for this replica only
// Only apply() may change `replica`; do not edit it in application code.
```

### Do and Don't

- Do build a fresh root and then transfer it; validate untrusted roots at your ingestion boundary.
- Do keep structural sharing between revisions, for example `replace(ctx, { ...state.value, changed })`.
- Do clone or serialize before handing any published value to code that may mutate it.
- Do settle every change with `prepare()` or `abort()`.
- Don't reuse one container at two places in a transferred root.
- Don't mutate anything the tracker owns, published, or received from a batch.
- Don't keep draft handles past their change or place a settled handle.
- Don't infer meaning from op shape. Only the resulting value is contractual.

### Roots and Placements

Full roots are trusted. `track()` and `prepareReplace()` take ownership in O(1) and never walk the tree. Roots must be alias-free, acyclic strict JSON: dense plain arrays, plain or null-prototype objects with enumerable own data properties, strings, booleans, finite numbers, and `null`. Violations have unspecified behavior.

Draft placements are copied, and that copy walk also validates. Placements come from property writes, index writes, `push`, `unshift`, `splice`, `fill`, and `copyWithin`. A non-strict value throws `TypeError` before the draft changes. Rejected values include: cycles, accessors, symbol keys, class instances, sparse or non-plain arrays, functions, bigint, `NaN`, infinities, and `undefined`. Null-prototype objects are accepted and preserved.

`undefined` rules:
- `draft.obj.key = undefined` deletes `key`.
- `arr[i] = undefined`, `push(undefined)`, `unshift`, `splice`, and `fill` with `undefined` throw.
- `undefined` nested anywhere inside a placed object or array throws.

Arrays stay dense. Writing past the next index and deleting an element throw. Growing `length` inserts `null`; shrinking it removes elements.

### Lifecycle

`tracker.value` is the latest adopted revision. `beginChange()` opens an overlay draft over it and never modifies it. A draft may stay open across `await`. `prepare()` materializes the candidate and ops and does not change authority. `adopt()` checks the preparation and swaps the root pointer.

Several changes may be open or prepared from one revision. Adopting one makes all others stale: open drafts become unusable and their `prepare()` throws, and prepared competitors are rejected by `adopt()`. `adopt()` also rejects foreign, aborted, and already consumed preparations. `Change.abort()` or `Prepared.abort()` after `prepare()` prevents adoption. The candidate stays readable.

No-op batches:
- Empty `ops` means `prepared.value === prepared.base`.
- Equality ignores key order and prototype.
- Structural array edits may emit a nonempty exact batch even when the result is deeply equal.
- Adopting a no-op still advances `tracker.revision` and stales competitors. Replicated state does not publish no-ops.

`prepareReplace(value)` is a whole-root operation, not a diff. If `value` is deeply equal to the current root, `ops` is empty and the current root is kept. Otherwise `ops` is `[["r", value]]` and `prepared.value === value`.

Array mutators: `push`, `pop`, `shift`, `unshift`, `splice`, `reverse`, `sort`, `fill`, and `copyWithin`. Held handles follow elements through reindexing.

### Operations

Paths contain object keys and non-negative integer array indices.

| Tuple | Meaning |
| --- | --- |
| `["r", value]` | Replace the complete value. |
| `["s", path, value]` | Set an object property or array element. |
| `["d", path]` | Delete an object property or remove an array element. |
| `["a", path, text]` | Append to a string. |
| `["t", path, count]` | Remove `count` UTF-16 code units from a string's front. |
| `["p", path, index, remove, items]` | Splice an array. |
| `["m", path, permutation]` | Reorder an array: `new[i] = old[permutation[i]]`. |

Batches are exact but not canonical. The same change may use different tuples, and large edit sets may fold into a region splice, an ancestor `s`, or `r`.

Reserved keys: the tracker never emits `__proto__`, `constructor`, or `prototype` as a path segment. A mutation at or below such a key is folded into a set of the nearest safe ancestor, or `r` at the root. `apply()`, `applyImmutable()`, `applyImmutableBatches()`, and `decoder()` reject those segments with `UnsafePathError`.

`encoder()` interns repeated paths into `WireOp` tuples; `decoder()` validates and restores `Op` tuples. Use one encoder/decoder pair per ordered state stream. Path IDs span batches and an `r` resets both dictionaries.

### Limits and Footguns

- No freezing. Mutating any immutable value corrupts state silently.
- In-process sharing: local handles, cell subscribers, and `replicatedState` subscribers receive the provider's containers. A consumer mutation corrupts authority and makes remote replicas diverge.
- A change inside a large flat array copies that array's pointer storage for the new revision.
- Placement validation costs time and memory on bulk inserts.
- Draft reads go through proxies; slower than raw object reads.
- External proxies as placements, argument coercion callbacks that mutate the draft, non-primitive array indices, and non-numeric comparator results are out of contract.
- Object identity is not replicated. Each path is an independent value placement.
- One tracker is one revision sequence. Delivery order and persistence belong to the surrounding protocol.

---

## 📁 目录

「待補充」

---

## 🔗 资源

- [Chord main README](../../README.md)

---

## 🚀 快速开始

```ts
import { applyImmutable, track } from "@earendil-works/chord/delta";

const initial = { output: "", entries: [] as { id: number }[] };
const tracker = track(initial);
const change = tracker.beginChange();
change.state.output += "done\n";
change.state.entries.push({ id: 1 });
const prepared = change.prepare();
tracker.adopt(prepared);
const replica = applyImmutable(prepared.base, prepared.ops);
```

---

## ⚙️ 技术栈

- TypeScript
- Trusted immutable ownership model (no proxies, no freezing for published values)
- Node.js and browser compatible
- Part of `@earendil-works/chord`