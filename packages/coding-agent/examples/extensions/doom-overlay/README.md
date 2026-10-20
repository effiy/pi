---
title: DOOM Overlay Demo — Real-Time Game Rendering in Pi
tags: [pi, doom, overlay, game, wasm, extension, tui]
category: pi-example
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# DOOM Overlay Demo

---

## 💡 概述

Play DOOM as an overlay in pi. Demonstrates that the overlay system can handle real-time game rendering at 35 FPS.

---

## ⚡ 功能

- Real-time DOOM gameplay at 35 FPS inside a pi overlay
- Half-block character rendering with 24-bit color (foreground/background per character)
- Auto-download of shareware WAD file (~4MB) on first run
- WASD/arrow key controls with run, fire, weapon switching, and map display

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

DOOM runs as WebAssembly compiled from [doomgeneric](https://github.com/ozkl/doomgeneric). Each frame is rendered using half-block characters (upper half block) with 24-bit color, where the top pixel is the foreground color and the bottom pixel is the background color.

The overlay uses:
- `width: "90%"` - 90% of terminal width
- `maxHeight: "80%"` - Maximum 80% of terminal height
- `anchor: "center"` - Centered in terminal

Height is calculated from width to maintain DOOM's 3.2:1 aspect ratio (accounting for half-block rendering).

---

## 📁 目录

「待補充」

---

## 🔗 资源

- [id Software](https://github.com/id-Software/DOOM) for the original DOOM
- [doomgeneric](https://github.com/ozkl/doomgeneric) for the portable DOOM implementation
- [pi-doom](https://github.com/badlogic/pi-doom) for the original pi integration

---

## 🚀 快速开始

```bash
pi --extension ./examples/extensions/doom-overlay
```

Then run:

```
/doom-overlay
```

The shareware WAD file (~4MB) is auto-downloaded on first run.

### Controls

| Action | Keys |
|--------|------|
| Move | WASD or Arrow Keys |
| Run | Shift + WASD |
| Fire | F or Ctrl |
| Use/Open | Space |
| Weapons | 1-7 |
| Map | Tab |
| Menu | Escape |
| Pause/Quit | Q |

---

## ⚙️ 技术栈

- WebAssembly (DOOM compiled from doomgeneric)
- Pi TUI overlay system
- Half-block Unicode character rendering (U+2580)
- 24-bit terminal color