---
title: Pi Telemetry - Vendor-Neutral Telemetry Contracts
tags: [telemetry, spans, attributes, events, observability, open-telemetry]
category: library
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi Telemetry

---

## 💡 概述

Vendor-neutral telemetry contracts and typed schema utilities for pi packages.

This package provides:

- an explicit, callback-based `TelemetryContext` / `TelemetrySpan` contract;
- a shared `NOOP_TELEMETRY_CONTEXT`;
- a reference `InMemoryTelemetryContext` implementation;
- serializable schema definitions with inferred TypeScript types;
- no exporter, global current-span state, or dependency on a telemetry backend.

Applications can use the in-memory reference or provide an adapter for OpenTelemetry, Sentry, logs, or another backend. Pi packages pass telemetry contexts explicitly and define their domain schemas separately.

---

## ⚡ 功能

### Telemetry concepts

| Concept | Meaning |
|---|---|
| **Span** | A timed record of one operation |
| **Parent and child spans** | Operations can contain smaller operations, forming a tree |
| **Attribute** | A named fact attached to a span |
| **Event** | A named occurrence at a point during a span, no duration |
| **Status** | The operation's outcome: `ok` or `error` |
| **Context** | A handle identifying where new work belongs in the span tree |

### Core API

- `TelemetryContext`: starts a span around a callback
- `TelemetrySpan`: records attributes, events, and status; also acts as a child context
- `SpanOptions`: span name and optional start attributes
- `NOOP_TELEMETRY_CONTEXT`: shared passive context for disabled telemetry
- `InMemoryTelemetryContext`: reference adapter with deterministic process-local recording

### Typed schemas

- `defineTelemetrySchema()`: typed identity helper for serializable schema data
- `createTypedSpanStarter()`: binds a parent context to one or more schema vocabularies
- Attribute types: `string`, `number`, `boolean`, and their arrays
- Attribute metadata: `values` (closed set), `elementValues`, `examples`, `sensitive`, `cardinality`
- Start and completion attributes with compile-time validation

### Package integration

- `@earendil-works/pi-telemetry` owns the vendor-neutral contract, no-op/in-memory contexts, schema utilities, and adapter conformance suite
- `@earendil-works/pi-ai` accepts and propagates `telemetryContext` in provider request options
- `@earendil-works/pi-agent-core` owns and exports the pi AI-request and harness schemas

---

## 📊 指標

「待補充」

---

## 🏛️ 設計

### Adapter contract

An adapter implements `TelemetryContext` and must:
- create a child span and invoke the callback synchronously, exactly once;
- preserve the callback's returned value and rejection value;
- keep the native span open until a returned promise settles;
- treat normal completion as `ok` and throws/rejections as errors unless an explicit status was set;
- make repeated `setStatus()` calls last-write-wins;
- merge `setAttributes()` calls, with later defined values replacing earlier values and `undefined` ignored;
- make recording methods synchronous, passive, and non-throwing;
- ignore calls made after settlement;
- ignore a failed recording call atomically, suppress backend failures.

### No-op context

- invokes callbacks synchronously;
- preserves returned values and asynchronous rejections;
- uses one shared frozen inert span, including for nested spans;
- does not inspect or retain names, attributes, events, or statuses.

### Schema metadata

Supported attribute types: `string`, `number`, `boolean`, `string[]`, `number[]`, `boolean[]`.

Parent metadata: `{ kind: 'any' }`, `{ kind: 'root_or_external' }`, `{ kind: 'spans', spans: [...] }`.

### Security

Telemetry is process-local diagnostics, not durable application state. Do not persist a TelemetryContext, TelemetrySpan, or backend-native trace object. Attribute values are limited to primitive scalars and arrays. The package does not use AsyncLocalStorage.

---

## 📁 目录

「待補充」

---

## 🔗 资源

「待補充」

---

## 🚀 快速开始

```bash
npm install @earendil-works/pi-telemetry
```

```typescript
import { NOOP_TELEMETRY_CONTEXT, type TelemetryContext } from '@earendil-works/pi-telemetry';

async function loadAccount(
  accountId: string,
  telemetryContext: TelemetryContext = NOOP_TELEMETRY_CONTEXT,
) {
  return telemetryContext.startSpan(
    { name: 'example.account.load', attributes: { 'example.account.id': accountId } },
    async (span) => {
      const account = await readAccount(accountId);
      span.setAttributes({ 'example.account.found': account !== undefined });
      return account;
    },
  );
}
```

### Typed span starter

```typescript
import { createTypedSpanStarter, defineTelemetrySchema } from '@earendil-works/pi-telemetry';

export const EXAMPLE_TELEMETRY_SCHEMA = defineTelemetrySchema({
  version: 1,
  spans: {
    'example.read': {
      description: 'Read one resource',
      parents: { kind: 'any' },
      startAttributes: {
        'example.resource': { type: 'string', required: true, values: ['account', 'project'] },
      },
      endAttributes: {
        'example.item_count': { type: 'number', description: 'Number of returned items' },
      },
    },
  },
} as const);

const startSpan = createTypedSpanStarter(telemetryContext, [EXAMPLE_TELEMETRY_SCHEMA]);
```

### Adapter conformance

```typescript
import { createTelemetryAdapterConformance, type TelemetryAdapterFixture } from '@earendil-works/pi-telemetry/testing';

const conformance = createTelemetryAdapterConformance(async () => {
  const adapter = createMyTelemetryAdapter();
  return {
    context: adapter.context,
    getSpans: async () => adapter.normalizedSpans(),
    async [Symbol.asyncDispose]() { await adapter.close(); },
  } satisfies TelemetryAdapterFixture;
});
```

---

## ⚙️ 技术栈

- TypeScript (runtime-neutral: Node.js, Bun, browsers, workers)
- No async local storage or ambient context APIs
- No external telemetry backend dependency