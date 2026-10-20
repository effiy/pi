---
title: Darwin Native Prebuilds for Pi TUI
tags: [tui, native, darwin, macos, appkit, clipboard]
category: native
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Darwin Native Prebuilds

---

## 💡 概述

Provides modifier-key state and asynchronous text, image, and file-path clipboard access using AppKit.

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

On macOS, run from the repository root:

```sh
npm --prefix packages/tui run build:native:darwin
```

The build uses Apple clang and the macOS SDK found through `xcrun`. Either Intel or Apple Silicon can build both targets: arm64 (macOS 11+) and x64 (macOS 10.15+).

Cross-building requires a Darwin toolchain with a macOS SDK and Mach-O linker, such as osxcross:

```sh
CC=/path/to/osxcross/clang SDKROOT=/path/to/MacOSX.sdk \
  npm --prefix packages/tui run build:native:darwin
```

The SDK must be obtained and used under Apple's license. Plain clang or Zig alone is not sufficient.

---

## ⚙️ 技术栈

- C / Objective-C (AppKit)
- Apple clang + macOS SDK (via xcrun)
- Cross-compilation via osxcross