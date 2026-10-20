---
title: Pi Extension Examples — Lifecycle, Tools, Commands, UI, Git, Providers
tags: [pi, coding-agent, extensions, examples, tools, ui, lifecycle]
category: pi-example
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi Extension Examples

---

## 💡 概述

Example extensions for pi-coding-agent.

---

## ⚡ 功能

### Lifecycle and Safety
- Permission gates for dangerous bash commands
- Project trust event handling
- Protected path write blocking
- Destructive action confirmation
- Dirty repo guard

### Custom Tools
- Todo list tool with state persistence and rendering
- Minimal custom tool examples
- UI-driven user questions (select, questionnaire)
- Tool overrides with logging/access control
- Dynamic tool registration at runtime
- Structured output with terminate support
- Custom built-in tool rendering
- SSH delegation of tools to remote machines
- Subagent delegation with isolated context windows

### Commands and UI
- Named presets for model/thinking/tools
- Plan mode with read-only exploration and step tracking
- Interactive tool enable/disable
- Context handoff to new sessions
- Q&A extraction from responses
- Turn progress in status bar
- GitHub issue autocomplete
- Custom widgets, editors, footers, headers
- DOOM game overlay at 35 FPS
- Desktop notifications and title bar animations
- Modal vim-like editor, rainbow text effects

### Git Integration
- Git stash checkpoints at each turn
- Auto-commit on exit

### System Prompt and Compaction
- Dynamic system prompt modifications
- Claude rules folder scanning
- Custom compaction and trigger

### Custom Providers
- Anthropic provider with OAuth and custom streaming
- GitLab Duo provider via pi-ai proxy
- Jev classifier router for virtual model switching

### Resources and Messages
- Dynamic resource discovery (skills, prompts, themes)
- Custom message and entry renderers
- Debug provider stream capture
- Inter-extension event bus

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

「待補充」

---

## 📁 目录

### Lifecycle and Safety
| Extension | Description |
|-----------|-------------|
| `permission-gate.ts` | Prompts for confirmation before dangerous bash commands (rm -rf, sudo, etc.) |
| `project-trust.ts` | Demonstrates the `project_trust` event for user/global and CLI extensions |
| `protected-paths.ts` | Blocks writes to protected paths (.env, .git/, node_modules/) |
| `confirm-destructive.ts` | Confirms before destructive session actions (clear, switch, fork) |
| `dirty-repo-guard.ts` | Prevents session changes with uncommitted git changes |
| `sandbox/` | OS-level sandboxing using `@anthropic-ai/sandbox-runtime` with per-project config |
| `gondolin/` | Route built-in tools and `!` commands into a Gondolin micro-VM |

### Custom Tools
| Extension | Description |
|-----------|-------------|
| `todo.ts` | Todo list tool + `/todos` command with custom rendering and state persistence |
| `hello.ts` | Minimal custom tool example |
| `question.ts` | Demonstrates `ctx.ui.select()` for asking the user questions with custom UI |
| `questionnaire.ts` | Multi-question input with tab bar navigation between questions |
| `tool-override.ts` | Override built-in tools (e.g., add logging/access control to `read`) |
| `dynamic-tools.ts` | Register tools after startup and at runtime via command |
| `structured-output.ts` | Final structured-output tool that returns `terminate: true` |
| `built-in-tool-renderer.ts` | Custom compact rendering for built-in tools |
| `minimal-mode.ts` | Override built-in tool rendering for minimal display |
| `truncated-tool.ts` | Wraps ripgrep with proper output truncation (50KB/2000 lines) |
| `ssh.ts` | Delegate all tools to a remote machine via SSH using pluggable operations |
| `subagent/` | Delegate tasks to specialized subagents with isolated context windows |

### Commands and UI
| Extension | Description |
|-----------|-------------|
| `preset.ts` | Named presets for model, thinking level, tools, and instructions |
| `plan-mode/` | Claude Code-style plan mode for read-only exploration |
| `tools.ts` | Interactive `/tools` command to enable/disable tools |
| `handoff.ts` | Transfer context to a new focused session via `/handoff <goal>` |
| `qna.ts` | Extracts questions from last response into editor |
| `status-line.ts` | Shows turn progress in footer via `ctx.ui.setStatus()` |
| `github-issue-autocomplete.ts` | Adds `#1234` issue completions |
| `widget-placement.ts` | Shows widgets above and below the editor |
| `hidden-thinking-label.ts` | Customizes the collapsed thinking label |
| `working-indicator.ts` | Customizes the streaming working indicator |
| `model-status.ts` | Shows model changes in status bar |
| `snake.ts` | Snake game with custom UI, keyboard handling, and session persistence |
| `tic-tac-toe.ts` | Tic-tac-toe vs the agent with sequential tool execution |
| `send-user-message.ts` | Demonstrates `pi.sendUserMessage()` |
| `timed-confirm.ts` | Demonstrates AbortSignal for auto-dismissing dialogs |
| `rpc-demo.ts` | Exercises all RPC-supported extension UI methods |
| `modal-editor.ts` | Custom vim-like modal editor |
| `rainbow-editor.ts` | Animated rainbow text effect via custom editor |
| `notify.ts` | Desktop notifications via OSC 777 |
| `titlebar-spinner.ts` | Braille spinner animation in terminal title |
| `summarize.ts` | Summarize conversation with GPT-5.2 |
| `custom-footer.ts` | Custom footer with git branch and token stats |
| `custom-header.ts` | Custom header via `ctx.ui.setHeader()` |
| `overlay-test.ts` | Test overlay compositing with inline text inputs and edge cases |
| `overlay-qa-tests.ts` | Comprehensive overlay QA tests |
| `doom-overlay/` | DOOM game running as an overlay at 35 FPS |
| `shutdown-command.ts` | Adds `/quit` command demonstrating `ctx.shutdown()` |
| `reload-runtime.ts` | Adds `/reload-runtime` and `reload_runtime` tool |
| `interactive-shell.ts` | Run interactive commands (vim, htop) with full terminal |
| `inline-bash.ts` | Expands `!{command}` patterns in prompts |
| `input-transform-streaming.ts` | Skips expensive input preprocessing for mid-stream steering |

### Git Integration
| Extension | Description |
|-----------|-------------|
| `git-checkpoint.ts` | Creates git stash checkpoints at each turn |
| `auto-commit-on-exit.ts` | Auto-commits on exit using last assistant message |

### System Prompt and Compaction
| Extension | Description |
|-----------|-------------|
| `pirate.ts` | Demonstrates `systemPromptAppend` to dynamically modify system prompt |
| `claude-rules.ts` | Scans `.claude/rules/` folder and lists rules in system prompt |
| `custom-compaction.ts` | Custom compaction that summarizes entire conversation |
| `trigger-compact.ts` | Triggers compaction when context usage exceeds 100k tokens |

### System Integration
| Extension | Description |
|-----------|-------------|
| `mac-system-theme.ts` | Syncs pi theme with macOS dark/light mode |

### Resources
| Extension | Description |
|-----------|-------------|
| `dynamic-resources/` | Loads skills, prompts, and themes using `resources_discover` |

### Messages and Communication
| Extension | Description |
|-----------|-------------|
| `message-renderer.ts` | Custom message rendering with colors and expandable details |
| `entry-renderer.ts` | TUI-only session entry rendering |
| `debug-provider.ts` | Toggle raw provider stream capture |
| `event-bus.ts` | Inter-extension communication via `pi.events` |

### Session Metadata
| Extension | Description |
|-----------|-------------|
| `session-name.ts` | Name sessions for the session selector |
| `bookmark.ts` | Bookmark entries with labels for `/tree` navigation |

### Custom Providers
| Extension | Description |
|-----------|-------------|
| `custom-provider-anthropic/` | Custom Anthropic provider with OAuth support and custom streaming |
| `custom-provider-gitlab-duo/` | GitLab Duo provider using pi-ai's built-in Anthropic/OpenAI streaming |
| `jev-router.ts` | Virtual model via `registerVirtualModel` that plans on Codex Sol/Terra and switches to Luna after first edit |

### External Dependencies
| Extension | Description |
|-----------|-------------|
| `with-deps/` | Extension with its own package.json and dependencies |
| `file-trigger.ts` | Watches a trigger file and injects contents into conversation |

---

## 🔗 资源

- [Extensions Documentation](../../docs/extensions.md)

---

## 🚀 快速开始

```bash
# Load an extension with --extension flag
pi --extension examples/extensions/permission-gate.ts

# Or copy to extensions directory for auto-discovery
cp permission-gate.ts ~/.pi/agent/extensions/
```

### Extension Template

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // Subscribe to lifecycle events
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  // Register custom tools
  pi.registerTool({
    name: "greet",
    label: "Greeting",
    description: "Generate a greeting",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // Register commands
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify("Hello!", "info");
    },
  });
}
```

### Key Patterns

**Use StringEnum for string parameters** (required for Google API compatibility):

```typescript
import { StringEnum } from "@earendil-works/pi-ai";

// Good
action: StringEnum(["list", "add"] as const)

// Bad - doesn't work with Google
action: Type.Union([Type.Literal("list"), Type.Literal("add")])
```

**State persistence via details:**

```typescript
// Store state in tool result details for proper forking support
return {
  content: [{ type: "text", text: "Done" }],
  details: { todos: [...todos], nextId },  // Persisted in session
};

// Reconstruct on session events
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "message" && entry.message.toolName === "my_tool") {
      const details = entry.message.details;
      // Reconstruct state from details
    }
  }
});
```

---

## ⚙️ 技术栈

- TypeScript with TypeBox schemas
- `@earendil-works/pi-coding-agent` ExtensionAPI
- `@earendil-works/pi-ai` for model interaction and schema validation
- Node.js TUI