---
title: Pi — Minimal, Extensible Agent Harness
tags: [pi, coding-agent, cli, ai, agent, extensible, typescript]
category: pi-package
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi — Minimal, Extensible Agent Harness

---

## 💡 概述

Pi is a minimal, extensible agent harness that you can make your own.

Adapt Pi to your workflows, not the other way around. Customize Pi with [extensions](docs/extensions.md), [skills](docs/skills.md), [prompt templates](docs/prompt-templates.md), and [themes](docs/themes.md). Bundle them as [Pi packages](docs/packages.md) and share via npm or git.

Pi ships with powerful defaults but skips features like sub-agents and plan mode. Ask Pi to build what you want, or install a package that does it your way.

Use Pi [interactively](docs/usage.md), automate it in [print or JSON mode](docs/cli.md), control it over [RPC](docs/rpc.md), or build apps with the [Pi TypeScript SDK](docs/sdk.md). See [OpenClaw](https://github.com/OpenClaw/OpenClaw) for a real-world integration.

---

## ⚡ 功能

- Interactive CLI agent with extensible tool system
- Extensions API: lifecycle hooks, custom tools, commands, UI customization
- Skills system: reusable prompt templates and workflows
- Multiple modes: interactive TUI, print mode, JSON output, RPC server
- TypeScript SDK for programmatic integration
- Themes for visual customization
- Pi packages for sharing extensions via npm or git

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

「待補充」

---

## 📁 目录

- [docs/index.md](docs/index.md) — Full documentation index
- [docs/extensions.md](docs/extensions.md) — Extension API
- [docs/skills.md](docs/skills.md) — Skills system
- [docs/prompt-templates.md](docs/prompt-templates.md) — Prompt templates
- [docs/themes.md](docs/themes.md) — Themes
- [docs/packages.md](docs/packages.md) — Pi packages
- [docs/usage.md](docs/usage.md) — Interactive usage
- [docs/cli.md](docs/cli.md) — CLI modes
- [docs/rpc.md](docs/rpc.md) — RPC integration
- [docs/sdk.md](docs/sdk.md) — TypeScript SDK
- [docs/quickstart.md](docs/quickstart.md) — Quickstart guide

---

## 🔗 资源

- [Discord](https://discord.com/invite/3cU7Bz4UPx)
- [npm](https://www.npmjs.com/package/@earendil-works/pi-coding-agent)
- [OpenClaw](https://github.com/OpenClaw/OpenClaw) — Real-world integration
- [CONTRIBUTING.md](https://github.com/earendil-works/pi/blob/main/CONTRIBUTING.md)
- [AGENTS.md](https://github.com/earendil-works/pi/blob/main/AGENTS.md)
- [pi-share-hf](https://github.com/badlogic/pi-share-hf) — Share OSS sessions on Hugging Face
- [Published Pi development sessions](https://huggingface.co/datasets/badlogicgames/pi-mono)

---

## 🚀 快速开始

Install the command-line interface:

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

On Windows:

```shell
powershell -c "irm https://pi.dev/install.ps1 | iex"
```

The installer pins all dependencies and updates Pi with `pi update`. Alternatively, install directly with npm, which does not pin transitive dependencies:

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

Pi requires Node.js 22.19 or newer. The macOS, Linux, and Windows installers can install it if needed. Pi does not require dependency lifecycle scripts for a normal npm installation.

On macOS and Linux, Nix users can install the latest release with `nix profile add github:earendil-works/pi/stable`.

Start Pi in the directory where you want it to work:

```bash
cd /path/to/project
pi
```

For a built-in AI provider, run `/login` inside Pi to connect a subscription or API key. Then give Pi a task.

See the [documentation](docs/index.md) for full setup and usage instructions.

### Development

Clone the repository, install its dependencies, and run Pi from source:

```bash
git clone https://github.com/earendil-works/pi
cd pi
npm install --ignore-scripts
./pi-test.sh
```

`pi-test.sh` can be called from any directory and preserves the caller's working directory.

Before submitting changes, run:

```bash
npm run check
./test.sh
```

---

## ⚙️ 技术栈

- TypeScript
- Node.js 22.19+
- Terminal UI (TUI)
- Extensible plugin architecture
- RPC client/server
- License: MIT