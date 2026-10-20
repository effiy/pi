---
title: Coding Agent Suite Tests
tags: [coding-agent, testing, harness, faux-provider, ci]
category: testing
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Coding Agent Suite Tests

---

## 💡 概述

Use `test/suite/` for the new harness-based test suite around `AgentSession` and `AgentSessionRuntime`.

---

## ⚡ 功能

Rules:

- Use `test/suite/harness.ts`
- Use the faux provider from `packages/ai/src/providers/faux.ts`
- Do not use real provider APIs, real API keys, network calls, or paid tokens
- Keep these tests CI-safe and deterministic
- Do not use or extend the legacy `test/test-harness.ts` path unless a missing capability forces it

---

## 📊 指標

「待補充」

---

## 🏛️ 設計

「待補充」

---

## 📁 目录

「待補充」

---

## 🔗 资源

「待補充」

---

## 🚀 快速开始

「待補充」

---

## ⚙️ 技术栈

- TypeScript (Node strip-only mode)
- Vitest
- Faux provider from `packages/ai/src/providers/faux.ts`
- `packages/coding-agent/test/suite/harness.ts`