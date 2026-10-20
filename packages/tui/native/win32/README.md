---
title: Windows Native Prebuilds for Pi TUI
tags: [tui, native, windows, win32, clipboard, console]
category: native
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Windows Native Prebuilds

---

## 💡 概述

Provides console input setup, modifier-key state, and asynchronous text/image clipboard access. Links to `kernel32` and `user32`; no Node headers are required.

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

On Windows, install Visual Studio's "Desktop development with C++" workload, then run from the repository root to build x64 and arm64:

```sh
npm --prefix packages/tui run build:native:win32
```

For cross-builds or custom toolchains, provide MinGW-compatible compilers:

```sh
PI_TUI_WIN32_TOOLCHAIN=mingw \
CC_X64=/path/to/x86_64-w64-mingw32-gcc \
CC_ARM64=/path/to/aarch64-w64-mingw32-gcc \
npm --prefix packages/tui run build:native:win32
```

### Testing clipboard writes

On a Windows test desktop, run from `packages/tui` in PowerShell:

```powershell
$env:PI_TEST_NATIVE_CLIPBOARD = "1"
node --test test/native-platform.test.ts
```

This opt-in test checks native text writes and reads. **It replaces the system clipboard contents.**

---

## ⚙️ 技术栈

- C (kernel32, user32)
- MinGW-compatible cross-compilation
- Visual Studio C++ toolchain for native builds