---
title: Pi TUI - Terminal UI Framework
tags: [tui, terminal, rendering, components, mouse, images]
category: library
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi TUI

---

## 💡 概述

Minimal terminal UI framework with differential rendering and synchronized output for flicker-free interactive CLI applications.

---

## ⚡ 功能

- **Interchangeable Renderers**: Shared `TUI` interface with main-screen and alternate-screen implementations
- **Differential Rendering**: Updates only changed lines or viewport rows
- **Application-owned Scrolling**: Alternate-screen viewport supports mouse, trackpad, and keyboard navigation
- **Synchronized Output**: Uses CSI 2026 for atomic screen updates (no flicker)
- **Bracketed Paste Mode**: Handles large pastes correctly with markers for >10 line pastes
- **Component-based**: Simple Component interface with render() method
- **Theme Support**: Components accept theme interfaces for customizable styling
- **Built-in Components**: Text, TruncatedText, Input, Editor, Markdown, Loader, SelectList, SettingsList, MouseRegion, Spacer, Image, Box, Container, VStack, HStack, ScrollView
- **Inline Images**: Renders images in terminals that support Kitty or iTerm2 graphics protocols
- **Autocomplete Support**: File paths and slash commands

---

## 📊 指標

「待補充」

---

## 🏛️ 設計

### TUI interface and renderers

- `TuiMainScreen` renders into the main terminal buffer and preserves terminal scrollback.
- `TuiAltScreen` renders a fixed-height viewport in the alternate terminal buffer with application-owned scrolling. When stopped, it restores the main buffer and prints the complete final document.

### Component Interface

All components implement:

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  handleMouse?(event: TuiMouseEvent): TuiMouseEventResult | undefined;
  invalidate(): void;
}
```

### Colors and terminal styles

`Color` is an indexed ANSI color, an sRGB color, or an OKLCH color. Every color converts to sRGB. `styleText()` converts colors to truecolor or 256-color output based on the requested terminal mode. `parseColor()` also accepts OKHSL.

### Rendering modes

- `TuiMainScreen`: First render outputs all lines. Width change or change above viewport clears and fully re-renders. Normal updates move cursor to first changed line, clear to end, render changed lines.
- `TuiAltScreen`: Owns a terminal-height viewport. With `setLayoutRoot()`, `VStack`, `HStack`, and nested `ScrollView` can reserve fixed regions and independently scroll constrained regions. Both renderers wrap updates in synchronized output for atomic, flicker-free rendering.

### Overlays

Overlays render components on top of existing content without replacing it. Support for anchor-based positioning, percentage positioning, absolute positioning, margin, responsive visibility callbacks, and focus management.

### Mouse input

`TuiAltScreen` normalizes SGR mouse input and hit-tests components and overlays. `TuiMainScreen` does not capture mouse input because the terminal owns its scrollback.

### Focusable interface (IME support)

Components that display a text cursor and need IME support should implement the `Focusable` interface. Uses `CURSOR_MARKER` and `renderFakeCursor()` for cursor positioning. Container components with embedded inputs must propagate focus state to children.

### Terminal interface

The TUI works with any object implementing the `Terminal` interface:
- `ProcessTerminal` - Uses `process.stdin/stdout`
- `VirtualTerminal` - For testing (uses `@xterm/headless`)

---

## 📁 目录

「待補充」

---

## 🔗 资源

- Example: `test/chat-simple.ts` for a complete chat interface example
- Debug logging: set `PI_TUI_WRITE_LOG` to capture the raw ANSI stream

---

## 🚀 快速开始

```typescript
import { type TUI, Text, Editor, ProcessTerminal, TuiMainScreen, matchesKey } from "@earendil-works/pi-tui";

// Create terminal
const terminal = new ProcessTerminal();

// Create the default main-screen renderer through the shared TUI interface
const tui: TUI = new TuiMainScreen(terminal);

// Add components
tui.addChild(new Text("Welcome to my app!"));

import { defaultEditorTheme as editorTheme } from './test/test-themes.ts';
const editor = new Editor(tui, editorTheme);
editor.onSubmit = (text) => {
  console.log("Submitted:", text);
  tui.addChild(new Text(`You said: ${text}`));
};
tui.addChild(editor);

// Focus the editor so it receives keyboard input
tui.setFocus(editor);

// In raw mode Ctrl+C doesn't send SIGINT -- intercept it here to allow exit
tui.addInputListener((data) => {
  if (matchesKey(data, 'ctrl+c')) {
    tui.stop();
    process.exit(0);
  }
});

// Start
tui.start();
```

### Run the example

```bash
node test/chat-simple.ts
```

---

## ⚙️ 技术栈

- TypeScript (Node strip-only mode)
- ANSI escape codes (CSI 2026, SGR, OSC 8, OSC 52)
- Kitty and iTerm2 graphics protocols for inline images
- `@xterm/headless` for testing