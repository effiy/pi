---
title: pi-codemode on Cloudflare — Example with Remote Sandbox Worker
tags: [codemode, cloudflare, workers, durable-objects, example, sandbox]
category: pi-example
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# pi-codemode on Cloudflare

---

## 💡 概述

This example has two Workers:

- **`sandbox/`** deploys `CodemodeSandboxDurableObject` as its own Worker, `pi-codemode-sandbox`. Scripts run there, in a separate isolate.
- **`agent/`** has a `Session` Durable Object that owns the tools. It runs scripts through `RemoteCodemodeSandbox`, which keeps the tools and results in `Session` while the sandbox Worker runs the VM.

The [README of the package](../../README.md#usage-on-cloudflare-workers) explains why the sandbox must be a separate Worker, and which limits apply.

---

## ⚡ 功能

「待補充」

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

「待補充」

---

## 📁 目录

「待補充」

---

## 🔗 资源

- [Package README](../../README.md)

---

## 🚀 快速开始

Install `@earendil-works/pi-codemode` and `wrangler` in each directory, then start both Workers in one process:

```sh
npx wrangler dev -c agent/wrangler.jsonc -c sandbox/wrangler.jsonc
curl -X POST http://localhost:8787 --data 'const n = await tools.count({ add: 2 }); text("n=" + n); return n'
```

### Deploy

Deploy the sandbox first, because the agent's binding refers to it by name:

```sh
(cd sandbox && npx wrangler deploy)
(cd agent && npx wrangler deploy)
```

---

## ⚙️ 技术栈

- Cloudflare Workers
- Cloudflare Durable Objects
- `@earendil-works/pi-codemode`
- Wrangler CLI