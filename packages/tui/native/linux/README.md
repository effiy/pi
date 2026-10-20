---
title: Linux Clipboard Helper for Pi TUI
tags: [tui, native, linux, x11, clipboard, xcb]
category: native
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Linux Clipboard Helper

---

## 💡 概述

Provides asynchronous X11 text and image reads using `libxcb.so.1`. Prebuilds support x64 and arm64 on glibc and musl. Reads have bounded waits; if a native operation stalls, the helper remains unavailable until it finishes.

Coding-agent falls back to command-line tools when native reads are unavailable. Wayland reads use `wl-paste`; all Linux writes use the existing command-line or terminal clipboard paths.

---

## ⚡ 功能

「待補充」

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

### Building

Install a C compiler and XCB development headers, then run from the repository root on each supported Linux architecture:

```sh
npm --prefix packages/tui run build:native:linux
```

### Testing

Install the build dependencies plus `pkg-config`, `Xvfb`, and `xclip`, then run from `packages/tui`:

```sh
node --test test/native-clipboard-linux.test.ts
```

Tests use isolated X11 servers, not the desktop clipboard. They skip when dependencies are missing.

---

## ⚙️ 技术栈

- C (libxcb.so.1)
- X11 clipboard protocol
- Prebuilds for x64 and arm64 (glibc and musl)