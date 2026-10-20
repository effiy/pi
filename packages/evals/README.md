---
title: Pi Evals - Behavioral Evals for Coding Agent
tags: [evals, coding-agent, vitest-evals, documentation, docker, ci]
category: testing
created: 2025
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# Pi Evals

---

## 💡 概述

Behavioral evals for Pi's coding agent, built with `vitest-evals`.

---

## ⚡ 功能

### Eval file conventions

Eval definitions are flat under `evals/`:

- `*.docs.eval.ts` is a documentation-lift eval. `eval:docs` runs each case in isolated `without_docs` and `with_docs` containers and reports lift.
- Other `*.eval.ts` files are host evals. `eval:host` runs them with Vitest on this machine. They are ordinary vitest-evals suites, not paired comparisons.

### Runner code layout

- `cli.ts` orchestrates a comparison
- `docker.ts` builds the two images, discovers cases, and runs one isolated arm
- `plan.ts` expands cases into `(case, variant, repetition)` tasks
- `report.ts` reads Vitest JSON, pairs arms, and computes lift
- `harness.ts` is the vitest-evals adapter

### Documentation variants

`without_docs` omits the coding-agent `README.md`, `CHANGELOG.md`, `docs/`, and `examples/`, then removes the Pi documentation-routing section from the default system prompt.

`with_docs` includes those files and uses the unchanged default prompt.

---

## 📊 指標

Results are reported as paired comparisons with pass-rate lift. A pair contributes to pass-rate lift only when both arms produce exactly one score. Missing, duplicate, skipped, pending, unscored, or errored arms block the pair.

The report flags no lift, negative deltas, saturated controls or treatments, and observed flakiness.

---

## 🏛️ 設計

### Documentation comparison runner

1. Mounts the repository ephemerally for a Docker build, packs the current workspace packages, then creates separate `without_docs` and `with_docs` images from the staged runtime.
2. Discovers the selected cases in both images and requires identical cohorts.
3. Plans every `(case, variant, model, runNumber)` arm before execution.
4. Runs each arm in a fresh container. A failed or missing arm is recorded and the planned cohort continues.
5. Reads native Vitest JSON through `@vitest-evals/core/node` when a report exists.
6. Pairs exact arms and writes the comparison report. Blocked pairs withhold headline lift; the process exits nonzero.

Repetition order alternates by run number to reduce order bias.

### Security

Documentation evals allow only `read`, `write`, `edit`, `grep`, `find`, and `ls` by default. They do not expose shell or web-search tools. Each run receives a new home, agent directory, workspace, session directory, and container filesystem.

### Writing an eval

```ts
import { describeEval, StructuredOutputJudge } from "vitest-evals";
import { createPiDocumentationEvalHarness } from "../src/harness.ts";

const harness = createPiDocumentationEvalHarness();
const judge = StructuredOutputJudge({ expected: { ok: true }, match: "strict", allowExtras: false });

describeEval("Target workflow", { harness, judges: [judge], judgeThreshold: null }, (it) => {
  it("completes the task", async ({ run }) => {
    await run("Complete the target task.");
  });
});
```

Use `judgeThreshold: null` for comparative scoring. A low score is data, not an infrastructure failure. Reserve Vitest assertions for broken suite invariants.

---

## 📁 目录

- `evals/`: eval definitions and fixtures
- `src/`: runner code (cli, docker, plan, report, harness)
- `docker/`: image build files

---

## 🔗 资源

「待補充」

---

## 🚀 快速开始

Host evals and documentation-lift evals need `PI_PROVIDER` and `PI_MODEL`.

```bash
PI_PROVIDER=openai-codex PI_MODEL=gpt-5.6-sol npm run eval -w packages/evals
```

Host only:

```bash
PI_PROVIDER=openai-codex PI_MODEL=gpt-5.6-sol npm run eval:host -w packages/evals
```

One host suite:

```bash
PI_PROVIDER=openai-codex PI_MODEL=gpt-5.6-sol \
  npm run eval:host -w packages/evals -- evals/documentation-audit.eval.ts
```

Documentation comparisons:

```bash
npm run eval:docs -w packages/evals -- \
  --provider openai-codex \
  --model gpt-5.6-sol
```

With repetitions:

```bash
npm run eval:docs -w packages/evals -- \
  evals/extensions.docs.eval.ts \
  --runs-per-variant 5
```

### Results location

Each invocation creates an ignored `.eval/<timestamp>_<id>/` directory containing `protocol.json`, `expected-runs.json`, `observations.jsonl`, task results, variant sessions, and `report.json`/`report.txt`.

---

## ⚙️ 技术栈

- TypeScript
- vitest-evals
- Docker (for documentation comparisons)
- `@vitest-evals/core/node` for JSON report reading