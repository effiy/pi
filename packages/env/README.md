---
title: Pi Env - Remote Execution Environments
tags: [env, execution, ssh, remote, daemon, durable]
category: library
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi Env

---

## 💡 概述

Remote execution environments for [Pi Durable](../durable): an agent's tools run on another machine, usually over SSH, while the Durable worker, its storage and credentials stay local.

- `pi-env` (`daemon/`): a small Rust program that runs on the remote machine. It speaks a framed protocol on stdin and stdout ([docs/protocol.md](docs/protocol.md)) and performs file operations and commands there.
- `RemoteExecutionEnv`: a Durable `ExecutionEnv` that talks to the daemon through a `Connection`. Its results match `NodeExecutionEnv` running on the remote machine; only error messages may differ ([docs/semantics.md](docs/semantics.md)).

---

## ⚡ 功能

Supported remote systems: Linux, macOS, Android (Termux) and Windows, on x86-64 and arm64. On Windows, string commands run through Git Bash as `NodeExecutionEnv` runs them there; argv commands run directly.

- Lazy connection: nothing happens until the first operation, which detects, deploys and connects.
- `env.listFiles()`: lists the tree below a directory, as `fd --hidden --exclude .git` does.
- `nodePlatform()`: gives the remote system in Node's names (`win32`, `darwin`, `x64`, `arm64`).
- `openTerminal()`: runs a program in a pseudo-terminal of the daemon for interactive terminals.
- `localDaemon()`: the daemon for this machine, for terminals without SSH.

---

## 📊 指標

「待補充」

---

## 🏛️ 設計

The package ships the daemon for every supported remote system in `bin/`. `ssh` runs with `BatchMode`, strict host-key checking against the application's own known-hosts file under a fixed alias, no forwarding of any kind, no shared connections or configured commands, and without forwarding the local locale. On Windows, detection and deployment go through PowerShell, and the daemon starts through the server's default shell.

For local sessions, `LocalExecutionEnv` is `NodeExecutionEnv` plus `listFiles` through a daemon on the same machine.

---

## 📁 目录

「待補充」

---

## 🔗 资源

- [docs/protocol.md](docs/protocol.md): framed protocol specification
- [docs/semantics.md](docs/semantics.md): execution semantics

---

## 🚀 快速开始

```ts
import { acceptHostKey, connectSsh, RemoteExecutionEnv, scanHostKey, sshConnection } from "@earendil-works/pi-env";

const target = { host: "gpu-box", knownHostsFile: "/data/ssh/known_hosts", hostKeyAlias: "pi-env-gpu" };
// Once: show the host's key fingerprint to the owner, who compares it out of band and accepts it.
const { lines, fingerprints } = await scanHostKey(target);
await acceptHostKey(target, lines);

// Detects the remote system, deploys the daemon if missing, and returns a connection.
const { connection } = await connectSsh(target);
const env = new RemoteExecutionEnv({ connection, id: "pi-env:gpu", cwd: "/home/me/project" });

// Or lazily: nothing happens until the first operation.
const lazy = sshConnection(target);
const lazyEnv = new RemoteExecutionEnv({ connection: lazy.connection, id: "pi-env:gpu", cwd: "/home/me/project" });
```

### Local daemon

```ts
const local = new Connection({ command: [localDaemon()] });
const env = new LocalExecutionEnv({ cwd: "/home/me/project", connection: local });
const listing = await env.listFiles(".", { exclude: ["node_modules"] }, context);
```

---

## ⚙️ 技术栈

- Rust (daemon)
- TypeScript (client)
- SSH with strict host-key checking
- Cross-platform: Linux, macOS, Android (Termux), Windows (x86-64, arm64)

### Development

```bash
npm run build:daemon    # Build the daemon with Cargo
```

Tests talk to `daemon/target/debug/pi-env` over a pipe, or to the binary named by `PI_ENV_DAEMON`. Tests run Durable's env conformance suite against `RemoteExecutionEnv`.