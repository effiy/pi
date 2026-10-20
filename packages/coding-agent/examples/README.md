---
title: Pi Coding Agent Examples — SDK, Extensions, and CLI Integration
tags: [pi, coding-agent, examples, sdk, extensions, cli]
category: pi-example
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi Coding Agent Examples

---

## 💡 概述

Example code for the pi-coding-agent SDK, process integration, and extensions.

---

## ⚡ 功能

- SDK examples for programmatic usage via `createAgentSession()`
- CLI integration with typed `RpcClient` for child process control
- Extension examples covering lifecycle events, custom tools, commands, UI, git integration, and more

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

「待補充」

---

## 📁 目录

### [sdk/](sdk/)
Programmatic usage via `createAgentSession()`. Shows how to customize models, prompts, tools, extensions, and session management.

### [extensions/](extensions/)
Example extensions demonstrating:
- Lifecycle event handlers (tool interception, safety gates, context modifications)
- Custom tools (todo lists, questions, subagents, output truncation)
- Commands and keyboard shortcuts
- Custom UI (footers, headers, editors, overlays)
- Git integration (checkpoints, auto-commit)
- System prompt modifications and custom compaction
- External integrations (SSH, file watchers, system theme sync)
- Custom providers (Anthropic with custom streaming, GitLab Duo)

---

## 🔗 资源

- [SDK Examples](sdk/README.md)
- [CLI Integration](../docs/cli-integration.md)
- [Extensions Documentation](../docs/extensions.md)
- [Skills Documentation](../docs/skills.md)

---

## 🚀 快速开始

### CLI Integration

[`rpc-client.ts`](rpc-client.ts) uses the typed `RpcClient` to run Pi in a child process, stream events, and wait for the run to settle.

Build the coding-agent package before running it from a repository checkout:

```bash
node examples/rpc-client.ts "Explain this repository"
```

---

## ⚙️ 技术栈

- TypeScript
- `@earendil-works/pi-coding-agent` SDK
- Node.js child process for RPC integration