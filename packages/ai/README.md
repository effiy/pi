---
title: pi-ai — Unified LLM API with Provider Collections, Auth, and Token Tracking
tags: [ai, llm, api, provider, models, streaming, tools, typescript]
category: pi-package
created: 2024
updated: 2026-10-20
source: external
type: index
status: stable
lifecycle: active
---

# pi-ai — Unified LLM API with Provider Collections, Auth, and Token Tracking

---

## 💡 概述

Unified LLM API with provider collections, automatic auth resolution, token and cost tracking, and simple context persistence and hand-off to other models mid-session.

**Note**: The chat catalog only includes models that support tool calling (function calling), as this is essential for agentic workflows. Image and classifier catalogs use their operation-specific capabilities.

---

## ⚡ 功能

- Unified LLM API across 30+ providers (OpenAI, Anthropic, Google, Vertex AI, AWS Bedrock, and more)
- Provider collections with `Models` class: register providers, query model catalogs, stream/complete
- Automatic auth resolution: env vars, stored credentials, OAuth flows
- Token and cost tracking with per-model pricing
- Tool calling with TypeBox schema validation and constrained sampling (JSON schema and grammar)
- Streaming with full event lifecycle: text, thinking, tool call deltas
- Image generation and classification APIs on the same provider model
- Cross-provider handoffs (switch models mid-conversation with automatic message transformation)
- System messages with dynamic prompt and tool changes mid-conversation
- Context serialization for persistence and transfer
- Browser support with static provider catalogs
- Faux provider for deterministic testing
- Tree-shakeable: import only the providers you need

---

## 📊 指標

「待補充」

---

## 🏛️ 设计

### Providers and Models

A **provider** is the runtime unit: it owns its model catalog, its auth (API key resolution, OAuth flows), and its stream behavior. A `Models` collection holds providers and routes every request to the provider that owns the model.

Providers internally share **API implementations** (the wire protocols): Anthropic models use `anthropic-messages`, OpenAI uses `openai-responses`, while xAI, Groq, Cerebras, OpenRouter, and most others share `openai-completions`. Mixed-API providers (GitHub Copilot, OpenCode Zen) dispatch per model.

### Provider Factories

For apps that only need specific providers, there is one factory per built-in provider, each a subpath import that pulls only that provider's catalog. Provider factories import their model catalog and a lazy API wrapper. They do not import other providers. With bundler code splitting, SDK implementations (`@anthropic-ai/sdk`, `openai`, `@google/genai`, etc.) stay in lazy chunks loaded on the first request to a model of that API.

### All Built-in Providers

```typescript
import { builtinModels } from '@earendil-works/pi-ai/providers/all';
const models = builtinModels(); // a Models collection with every built-in provider registered
```

`builtinModels()` accepts the same options as `createModels()` (`credentials`, `authContext`); `builtinProviders()` returns the provider array if you want to register them on your own collection.

### Querying Models

Reads are synchronous and return the last-known lists. The unqualified reads `getModels()`/`getModel()`/`getAvailable()` return chat models (`Model<Api>`) usable with `stream()`. The `*OfType` reads return one model type, and `getAllModels()`/`getAllAvailable()` return every type as `AnyModel`.

The model's `type` decides which operation accepts it: chat models stream, `type: "image"` models generate images, and `type: "classifier"` models classify structured state. `type` is optional on chat models, so a model without `type` is a chat model.

Dynamically listed chat models are typed `Model<Api>`. Narrow with the `hasApi()` guard when you need API-specific option typing.

### Static Catalog Reads

For tooling that wants the generated built-in catalog with full literal typing (provider and model IDs auto-complete), independent of any collection, use static functions from `@earendil-works/pi-ai/providers/all` such as `getBuiltinModel()`, `getBuiltinImageModel()`, `getBuiltinClassifierModel()`, etc.

### Dynamic Providers

Providers may have dynamic model lists (a llama.cpp server, a live OpenRouter listing). Reads stay sync; fetching is an explicit async verb via `models.refresh()`. Static built-in providers are no-ops for `refresh()`.

### Auth

Every provider owns its auth: how API keys resolve (stored credentials, environment variables, ambient sources like AWS profiles or gcloud ADC) and, where supported, OAuth login/refresh flows.

#### How Auth Resolves

When you call `models.stream()`, the collection resolves auth through the owning provider and merges it into the request. Explicit per-request values always win.

`getAuth()`, `checkAuth()`, `getAvailable()`, login, and logout accept optional caller cancellation through their existing options or interaction objects and remain unbounded when no signal is supplied.

### Transforming Request Headers

`Models.stream()`, `complete()`, `streamSimple()`, and `completeSimple()` accept a Models-only `transformHeaders` option. It runs once after provider auth, `model.headers`, and explicit `options.headers` have been merged, but before provider dispatch.

The ordering is:

```text
provider auth headers -> model.headers -> explicit options.headers -> transformHeaders -> Provider.stream*()
```

### Credential Store

Stored credentials (API keys entered interactively, OAuth tokens) live in a `CredentialStore` -- one type-tagged credential per provider. pi-ai ships an in-memory default; apps inject persistent storage.

The contract is small: `read(providerId)`, `list()` for non-secret metadata, `modify(providerId, fn)`, and `delete(providerId)`. OAuth token refresh runs inside `modify`, so concurrent requests and processes cannot double-refresh a rotated token.

### Tools

Tools enable LLMs to interact with external systems. This library uses TypeBox schemas for type-safe tool definitions with automatic validation using TypeBox's built-in validator and value conversion utilities. TypeBox schemas can be serialized and deserialized as plain JSON, making them ideal for distributed systems.

#### Constrained Sampling for Tools

Tools can opt in to provider-side constrained sampling. For JSON-schema tools, `strict: 'prefer'` uses provider-side strict schema enforcement when supported and otherwise falls back to normal tool calling. `strict: 'require'` fails the request when the active provider/model cannot honor it.

Strict JSON-schema constrained sampling is supported for OpenAI, Anthropic, supported Amazon Bedrock Converse models, Mistral, and Gemini 3 tool calls. OpenAI Responses and Chat Completions can also emit grammar-constrained custom tools with OpenAI Lark or regex grammar variants.

#### Streaming Tool Calls with Partial JSON

During streaming, tool call arguments are progressively parsed as they arrive. During `toolcall_delta` events, `arguments` contains the best-effort parse of partial JSON. Fields may be missing or incomplete -- always check for existence before use. At minimum, `arguments` will be an empty object `{}`, never `undefined`. The Google provider does not support function call streaming; instead, you will receive a single `toolcall_delta` event with the full arguments.

#### Validating Tool Arguments

Use `validateToolCall` to validate arguments against the tool's schema before passing them to your tools.

#### Complete Event Reference

Successful generation follows `start -> updates* -> done`. A failure after generation starts follows `start -> updates* -> error`. Request setup may fail before generation starts, in which case the stream contains only `error`.

Every non-terminal event's `partial` is the shared live response-so-far helper. It is intentionally not an event-time snapshot: providers may mutate the same message and content blocks as generation advances.

All streaming events emitted during assistant message generation:

| Event Type | Description | Key Properties |
|------------|-------------|----------------|
| `start` | Stream begins | `partial`: Initial assistant message structure |
| `text_start` | Text block starts | `contentIndex`: Position in content array |
| `text_delta` | Text chunk received | `delta`: New text, `contentIndex`: Position |
| `text_end` | Text block complete | `content`: Full text, `contentIndex`: Position |
| `thinking_start` | Thinking block starts | `contentIndex`: Position in content array |
| `thinking_delta` | Thinking chunk received | `delta`: New text, `contentIndex`: Position |
| `thinking_end` | Thinking block complete | `content`: Full thinking, `contentIndex`: Position |
| `toolcall_start` | Tool call begins | `contentIndex`: Position in content array |
| `toolcall_delta` | Tool arguments streaming | `delta`: JSON chunk, `partial.content[contentIndex].arguments`: Partial parsed args |
| `toolcall_end` | Tool call complete | `toolCall`: Complete, but not schema-validated, tool call with `id`, `name`, `arguments` |
| `done` | Stream complete | `reason`: Stop reason ("stop", "length", "toolUse"), `message`: Final assistant message |
| `error` | Error occurred | `reason`: Error type ("error" or "aborted"), `error`: AssistantMessage with partial content |

Streaming events for different content blocks are not guaranteed to be contiguous. Consumers must use `contentIndex` to associate each delta/end event with its block.

#### Compact Assistant Message Frames

`AssistantMessageFrameEncoder` converts one stream into compact, persistable `AssistantMessageFrame` values. `reduceAssistantMessageFrames()` is the canonical pure reducer. It reconstructs text, thinking, and tool-call arguments, including interleaved blocks identified by `contentIndex`, and rejects malformed sequences. The reducer does not validate tool arguments against a TypeBox schema; call `validateToolCall` before execution.

### Image Input

Models with vision capabilities can process images. Check if a model supports images via the `input` property. If you pass images to a non-vision model, they are silently ignored.

### Image Generation

Image models live in the same `Models` collection and on the same `Provider` as chat models, so one credential per provider covers both. They are typed `ImageModel` with `type: "image"` and are used through `generateImages()`, a one-shot API that waits for the provider response and returns the final `AssistantImages` result.

`generateImages()` accepts only `ImageModel` values. If an upstream model supports both chat and image generation, the catalog contains separate entries with the same provider and ID. Failures never reject; they return an `AssistantImages` with `stopReason: "error"`.

A provider declares image support with the `images` option of `createProvider()`: a map from `model.api` to an implementation with `generateImages()`. Image models go into the same `models` list as chat models.

### Classification

Classifier models consume structured JSON state, and optionally images, and answer one or more typed questions. They do not use chat or image-generation APIs.

Available built-in classifiers:

| Provider | Model IDs | Auth |
| --- | --- | --- |
| `typesafe` | `jev-latest` | `TYPESAFE_API_KEY` |
| `openrouter` | `typesafe/jev-1.13`, `~typesafe/jev-latest` | `OPENROUTER_API_KEY` or OpenRouter OAuth |
| `cloudflare-workers-ai` | `typesafe/jev`, `@cf/cloudflare/clef`, `@cf/cloudflare/clef-flash` | `CLOUDFLARE_API_KEY` and `CLOUDFLARE_ACCOUNT_ID` |
| `vercel-ai-gateway` | `typesafe-ai/jev` | `AI_GATEWAY_API_KEY` |
| `opencode` | `jev-1.13`, `jev-1.13-free` | `OPENCODE_API_KEY` |
| `openai` | `gpt-6-luna` | `OPENAI_API_KEY` |

`ClassifierOptions.temperature` divides the answer logits by the given value before they are normalized. `ClassifierContext.images` holds image blocks judged together with the state.

#### OpenAI Decisions

The `openai-decisions` API calls OpenAI's Decisions API (`POST /v1/decisions`). GPT-6 Luna is its only model. The endpoint needs an OpenAI API key; Sign in with ChatGPT credentials are rejected.

#### Chat models on llama.cpp

llama.cpp also serves decision models natively through `/v1/systemone`. The `llama-cpp-classify` API turns a chat model served by llama.cpp's `llama-server` into a classifier. Each question becomes one chat prompt. Raw label probabilities are usually overconfident; pass `temperature` above 1 to soften them.

### Thinking/Reasoning

Many models support thinking/reasoning capabilities. Check if a model supports reasoning via the `reasoning` property. If you pass reasoning options to a non-reasoning model, they are silently ignored.

#### Unified Interface (streamSimple/completeSimple)

Use the simplified reasoning option: `reasoning: 'minimal' | 'low' | 'medium' | 'high' | 'xhigh' | 'max'`.

`xhigh` and `max` are model-specific, opt-in levels. Use `getSupportedThinkingLevels(model)` to determine whether a concrete model exposes either level.

#### Provider-Specific Options (stream/complete)

`models.stream()`/`complete()` accept the owning API's full option set. Use `hasApi()` to narrow a dynamically looked-up model to its API for full option typing.

### Stop Reasons

Every `AssistantMessage` includes a `stopReason` field:

- `"pending"` - Only present in partial messages when we do not know what the stop reason will be
- `"stop"` - This is the final message the model will produce this turn
- `"length"` - Output hit the maximum token limit
- `"toolUse"` - Model is calling tools and expects tool results
- `"error"` - An error occurred during generation
- `"aborted"` - Request was cancelled via abort signal

### Error Handling

Request failures after a stream is returned never throw: when a request ends with an error (including aborts and tool call validation errors), the streaming API emits an error event and the final message carries the details. Setup failures may emit `error` without `start`; failures after generation begins emit `start`, any observed updates, then `error`. Direct API `streamSimple()` calls throw synchronously when request auth is missing.

#### Aborting Requests

The abort signal allows you to cancel in-progress requests. Aborted requests have `stopReason === 'aborted'`. Aborted messages can be added to the conversation context and continued in subsequent requests.

#### Debugging Provider Payloads

Use the `onPayload` callback to inspect the request payload sent to the provider. Use `onProviderStreamEvent` to inspect provider-specific fields that Pi does not include in `AssistantMessage`.

### Custom Providers

#### createProvider()

`createProvider()` builds a provider from parts: identity, auth, a model list, and an API implementation (`api` for chat models, `images` for image generation, `classifiers` for classification; at least one is required).

For standard key providers, `envApiKeyAuth(displayName, envVars)` gives the standard behavior (stored credential wins, then the first set env var).

Mixed-API providers pass a map keyed by `model.api`; each model dispatches to its API's implementation.

Provider-wide endpoint or request transformations belong in the provider's API implementation: wrap the `ProviderStreams` you pass as `api`.

Dynamic model lists use `fetchModels`, which can return models of every type. `Models.refresh()` refreshes every configured dynamic provider.

Custom models can carry `headers` (e.g. proxies behind bot detection) and `compat` flags.

Some OpenAI-compatible servers do not understand the `developer` role used for reasoning-capable models. For those providers, set `compat.supportsDeveloperRole` to `false`. If the server also does not support `reasoning_effort`, set `compat.supportsReasoningEffort` to `false` too.

Use model-level `thinkingLevelMap` to describe model-specific thinking controls. Keys are pi thinking levels (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`). Missing standard levels through `high` use provider defaults; `xhigh` and `max` are opt-in and require a non-null map entry.

#### Calling API Implementations Directly

The API implementations are importable on their own. Each module exports exactly `stream` and `streamSimple` with that API's full option typing. Direct calls bypass provider auth and context normalization -- pass `apiKey` explicitly and wrap the context in `normalizeContext()`.

Built-in API implementations live under `./api/<api-id>`:

| API id | Options type |
|--------|--------------|
| `anthropic-messages` | `AnthropicOptions` |
| `openai-completions` | `OpenAICompletionsOptions` |
| `openai-responses` | `OpenAIResponsesOptions` |
| `openai-codex-responses` | `OpenAICodexResponsesOptions` |
| `azure-openai-responses` | `AzureOpenAIResponsesOptions` |
| `google-generative-ai` | `GoogleOptions` |
| `google-vertex` | `GoogleVertexOptions` |
| `mistral-conversations` | `MistralOptions` |
| `bedrock-converse-stream` | `BedrockOptions` |

#### OpenAI Compatibility Settings

The `openai-completions` API is implemented by many providers with minor differences. By default, the library auto-detects compatibility settings based on `baseUrl` for a small set of known OpenAI-compatible providers. For custom proxies or unknown endpoints, you can override these settings via the `compat` field.

Key compat fields include: `supportsStore`, `supportsDeveloperRole`, `supportsReasoningEffort`, `supportsUsageInStreaming`, `supportsStrictMode`, `supportsOpenAIGrammarTools`, `supportsMidConvoSystemMessages`, `supportsMidConvoToolAdditions`, `sendSessionAffinityHeaders`, `sessionAffinityFormat`, `maxTokensField`, `requiresToolResultName`, `requiresAssistantAfterToolResult`, `requiresThinkingAsText`, `requiresReasoningContentOnAssistantMessages`, `thinkingFormat`, `chatTemplateKwargs`, `chatTemplateArgs`, `thinkingTokenBudgetField`, `cacheControlFormat`, `openRouterRouting`, `vercelGatewayRouting`.

OpenAI Responses compat fields: `supportsDeveloperRole`, `sessionAffinityFormat`, `supportsLongCacheRetention`, `supportsStrictMode`, `supportsOpenAIGrammarTools`.

### Faux Provider for Tests

`fauxProvider()` builds an in-memory provider with scripted responses for tests and demos. Responses are consumed from a queue in request start order. Use `faux.setResponses([...])` to replace the remaining queue and `faux.appendResponses([...])` to add more responses. By default, each streamed chunk is emitted on its own microtask. Set `tokensPerSecond` to pace chunk delivery in real time.

### Cross-Provider Handoffs

The library supports seamless handoffs between different LLM providers within the same conversation. When messages from one provider are sent to a different provider, the library automatically transforms them for compatibility:

- **User and tool result messages** are passed through unchanged
- **Assistant messages from the same provider/API** are preserved as-is
- **Assistant messages from different providers** have their thinking blocks converted to text with `<thinking>` tags
- **Tool calls and regular text** are preserved unchanged

All providers can handle messages from other providers -- text, tool calls and results (including images), thinking blocks (transformed to tagged text), and aborted messages with partial content.

### System Messages

`Context.systemPrompt` and `Context.tools` are shorthand for a leading system message. The transcript can also carry system messages later in the conversation to change the prompt or the tool set without rewriting the history:

```typescript
interface SystemMessage {
  role: "system";
  content: string | TextContent[];
  sections?: Record<string, string | null>;
  toolsAdded?: Tool[];
  toolsRemoved?: ToolReference[];
  timestamp: number;
}
```

Models that accept system messages mid-conversation receive each later system message in place. Every other model receives `collapseSystemMessages(transcript)`: the replayed prompt and current tools as the leading system message, with later system messages dropped. Anthropic models that also set `supportsMidConvoToolChanges` send tool changes as native blocks.

### Context Serialization

The `Context` object can be easily serialized and deserialized using standard JSON methods. Models are plain serializable data too -- no functions or implementations attached. If the context contains images (encoded as base64), those will also be serialized.

### Browser Usage

The library supports browser environments. The core entrypoint and provider factories are side-effect free and bundle cleanly. Environment variables are not available in browsers, so pass API keys explicitly -- or inject a `CredentialStore` (e.g. localStorage-backed) and let provider auth resolve from stored credentials.

> **Security Warning**: Exposing API keys in frontend code is dangerous. For production applications, use a backend proxy that keeps your API keys secure.

Browser compatibility notes:
- Amazon Bedrock is not supported in browser environments
- OAuth login flows are Node-only (lazy-loaded behind bundler-opaque imports)
- Use a server-side proxy if you need Bedrock or OAuth-based auth from a web app

### Bundling and Tree Shaking

For small bundles and low-overhead unbundled scripts, import the model runtime and only the providers you need from `@earendil-works/pi-ai/models`.

Rules:
- `@earendil-works/pi-ai/models` exports the model runtime without TypeBox, built-in catalogs, or SDK implementations
- `@earendil-works/pi-ai` is the core entrypoint and does not import built-in catalogs, real provider factories, or SDK implementations, but it eagerly imports TypeBox and schema validation
- `@earendil-works/pi-ai/providers/<provider>` imports that provider's catalog and lazy API wrapper only
- `@earendil-works/pi-ai/providers/all` imports every built-in provider factory and all catalogs
- With code splitting, provider SDKs stay in lazy chunks and load on first request
- Importing `@earendil-works/pi-ai/api/<api-id>` directly loads that API implementation and its SDK immediately

### Provider-Scoped Environment Overrides

Pass `env` in stream options to scope provider configuration to a request. Values in `env` are used before process environment variables for provider auth and configuration.

### OAuth Providers

Several providers support OAuth authentication instead of static API keys:
- **Anthropic** (Claude Pro/Max subscription)
- **OpenAI** (Sign in with ChatGPT)
- **OpenAI Codex (legacy)** (ChatGPT Plus/Pro subscription)
- **GitHub Copilot** (Copilot subscription)
- **OpenRouter** (OAuth PKCE that mints a user-controlled API key)

Each of these providers carries an `OAuthAuth` on `provider.auth.oauth` with three operations: `login`, `refresh`, and `toAuth`. Refresh is automatic: `models.getAuth(providerId)` and request paths refresh expired tokens under a credential-store lock.

#### Vertex AI

Vertex AI models support either a Google Cloud API key or Application Default Credentials (ADC). Its provider-owned API-key login flow can configure either method.

#### CLI Login

```bash
npx @earendil-works/pi-ai login              # interactive provider selection
npx @earendil-works/pi-ai login anthropic    # login to specific provider
npx @earendil-works/pi-ai list               # list available providers
```

#### Programmatic OAuth

Built-in login and refresh flows are private provider implementations. Use provider-owned `OAuthAuth`, which composes with `CredentialStore` and gets locked auto-refresh through `Models`.

Provider notes:
- **OpenAI Codex (legacy)**: Superseded by Sign in with ChatGPT on the OpenAI provider. Requires ChatGPT Plus or Pro subscription. The library automatically handles session-based prompt caching.
- **Azure OpenAI (Responses)**: Uses the Responses API only. Deployment names are treated as model IDs by default, override with `azureDeploymentName` or `AZURE_OPENAI_DEPLOYMENT_NAME_MAP`.
- **GitHub Copilot**: If you get "The requested model is not supported" error, enable the model manually in VS Code.

### Migrating from the Old Global API

Older versions exposed a global API. That surface lives unchanged on the **compat entrypoint**:

```typescript
// Before
import { getModel, complete } from '@earendil-works/pi-ai';

// After (verbatim behavior, one import-path change)
import { getModel, complete } from '@earendil-works/pi-ai/compat';
```

Compat will be removed in a future release; migrate to `createModels()` + provider factories:

| Old | New |
|-----|-----|
| `getModel('openai', 'gpt-4o-mini')` | `models.getModel('openai', 'gpt-4o-mini')` or `getBuiltinModel()` |
| `getModels('anthropic')` / `getProviders()` | `models.getModels('anthropic')` / `models.getProviders()` |
| `stream(model, ctx, opts)` | `models.stream(model, ctx, opts)` |
| `registerApiProvider({ api, stream, streamSimple })` | `createProvider({ id, auth, models, api })` |
| `getEnvApiKey('openai')` | `await models.getAuth(model.provider)` |
| `streamAnthropic(model, ctx, opts)` | `stream` from `@earendil-works/pi-ai/api/anthropic-messages` |
| `registerFauxProvider()` | `fauxProvider()` + `models.setProvider()` |
| `getImageModel('openrouter', id)` / `generateImages(model, ctx, { apiKey })` | `models.getModelOfType('image', 'openrouter', id)` / `models.generateImages(model, ctx)` |

---

## 📁 目录

### Supported Providers

- **OpenAI** (including the Decisions classifier API)
- **Ant Ling**
- **Azure OpenAI (Responses)**
- **OpenAI Codex (legacy)** (ChatGPT Plus/Pro subscription, requires OAuth)
- **Radius** (API key or OAuth, with a dynamically refreshed gateway catalog)
- **TypeSafe** (System One classifier API)
- **DeepSeek**
- **NVIDIA NIM**
- **Anthropic**
- **Google**
- **Vertex AI** (Gemini via Vertex AI)
- **Mistral**
- **Groq**
- **Cerebras**
- **Cloudflare AI Gateway**
- **Cloudflare Workers AI**
- **xAI**
- **OpenRouter**
- **Vercel AI Gateway**
- **ZAI Coding Plan (Global)** (with separate China provider)
- **MiniMax** (with separate China provider)
- **Together AI**
- **Baseten**
- **Hugging Face**
- **Moonshot AI** (with separate China provider)
- **GitHub Copilot** (requires OAuth)
- **Amazon Bedrock**
- **OpenCode Zen**
- **OpenCode Go**
- **Fireworks** (uses OpenAI- and Anthropic-compatible APIs)
- **Kimi For Coding** (Moonshot AI subscription endpoint)
- **Meta** (Model API, uses OpenAI Responses-compatible API)
- **Qwen Token Plan** (separate Individual and existing catalogs)
- **Xiaomi MiMo** (defaults to API billing endpoint)
- **Any OpenAI-compatible API**: Ollama, vLLM, LM Studio, etc.

### Environment Variables

| Provider | Environment Variable(s) |
|----------|------------------------|
| OpenAI | `OPENAI_API_KEY` |
| Ant Ling | `ANT_LING_API_KEY` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_BASE_URL` or `AZURE_OPENAI_RESOURCE_NAME` |
| Anthropic | `ANTHROPIC_API_KEY` or `ANTHROPIC_OAUTH_TOKEN` |
| Radius | `RADIUS_API_KEY` |
| TypeSafe | `TYPESAFE_API_KEY` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| NVIDIA NIM | `NVIDIA_API_KEY` |
| Google | `GEMINI_API_KEY` |
| Vertex AI | `GOOGLE_CLOUD_API_KEY` or `GOOGLE_CLOUD_PROJECT` + `GOOGLE_CLOUD_LOCATION` + ADC |
| Mistral | `MISTRAL_API_KEY` |
| Groq | `GROQ_API_KEY` |
| Cerebras | `CEREBRAS_API_KEY` |
| Cloudflare AI Gateway | `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID` + `CLOUDFLARE_GATEWAY_ID` |
| Cloudflare Workers AI | `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID` |
| xAI | `XAI_API_KEY` |
| Fireworks | `FIREWORKS_API_KEY` |
| Together AI | `TOGETHER_API_KEY` |
| Baseten | `BASETEN_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` |
| ZAI Coding Plan (Global) | `ZAI_API_KEY` |
| ZAI Coding Plan (China) | `ZAI_CODING_CN_API_KEY` |
| MiniMax (Global) | `MINIMAX_API_KEY` |
| MiniMax (China) | `MINIMAX_CN_API_KEY` |
| Moonshot AI / Moonshot AI (China) | `MOONSHOT_API_KEY` |
| Hugging Face | `HF_TOKEN` |
| OpenCode Zen / OpenCode Go | `OPENCODE_API_KEY` |
| Kimi For Coding | `KIMI_API_KEY` |
| Meta | `META_API_KEY` |
| Qwen Token Plan | `QWEN_TOKEN_PLAN_API_KEY` |
| Qwen Token Plan (China) | `QWEN_TOKEN_PLAN_CN_API_KEY` |
| Xiaomi MiMo (API billing) | `XIAOMI_API_KEY` |
| Xiaomi MiMo Token Plan (China) | `XIAOMI_TOKEN_PLAN_CN_API_KEY` |
| Xiaomi MiMo Token Plan (Amsterdam) | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` |
| Xiaomi MiMo Token Plan (Singapore) | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN` |

Amazon Bedrock resolves ambient AWS credentials. Vertex AI resolves either an explicit key or gcloud Application Default Credentials.

---

## 🔗 资源

- [Official docs: Vertex AI Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials)
- [OpenAI Decisions API](https://developers.openai.com/api/docs/guides/decisions)

---

## 🚀 快速开始

```bash
npm install @earendil-works/pi-ai
```

TypeBox exports are re-exported from `@earendil-works/pi-ai`: `Type`, `Static`, and `TSchema`.

```typescript
import { Type, type Context, type Tool } from '@earendil-works/pi-ai';
import { builtinModels } from '@earendil-works/pi-ai/providers/all';

const models = builtinModels();
const model = models.getModel('openai', 'gpt-4o-mini')!;

const tools: Tool[] = [{
  name: 'get_time',
  description: 'Get the current time',
  parameters: Type.Object({
    timezone: Type.Optional(Type.String({ description: 'Optional timezone' }))
  })
}];

const context: Context = {
  systemPrompt: 'You are a helpful assistant.',
  messages: [{ role: 'user', content: 'What time is it?', timestamp: Date.now() }],
  tools
};

// Streaming with all event types
const s = models.stream(model, context);
for await (const event of s) {
  switch (event.type) {
    case 'start': break;
    case 'text_delta': process.stdout.write(event.delta); break;
    case 'toolcall_end': console.log(`Tool called: ${event.toolCall.name}`); break;
    case 'done': console.log(`Finished: ${event.reason}`); break;
    case 'error': console.error(`Error: ${event.error.errorMessage}`); break;
  }
}

const finalMessage = await s.result();
context.messages.push(finalMessage);

// Handle tool calls
const toolCalls = finalMessage.content.filter(b => b.type === 'toolCall');
// ... add tool results to context

// Continue if there were tool calls
if (toolCalls.length > 0) {
  const continuation = await models.complete(model, context);
  context.messages.push(continuation);
}

console.log(`Total tokens: ${finalMessage.usage.input} in, ${finalMessage.usage.output} out`);
console.log(`Cost: $${finalMessage.usage.cost.total.toFixed(4)}`);

// Option 2: Get complete response without streaming
const response = await models.complete(model, context);
for (const block of response.content) {
  if (block.type === 'text') console.log(block.text);
  else if (block.type === 'toolCall') console.log(`Tool: ${block.name}(${JSON.stringify(block.arguments)})`);
}
```

### Provider Factories (selective imports)

```typescript
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';
import { openaiProvider } from '@earendil-works/pi-ai/providers/openai';
import { openrouterProvider } from '@earendil-works/pi-ai/providers/openrouter';

const models = createModels();
models.setProvider(anthropicProvider());
models.setProvider(openrouterProvider());
```

### Thinking/Reasoning

```typescript
const model = models.getModel('anthropic', 'claude-sonnet-4-5')!;
const response = await models.completeSimple(model, {
  messages: [{ role: 'user', content: 'Solve: 2x + 5 = 13', timestamp: Date.now() }]
}, {
  reasoning: 'medium'
});
```

### Image Input

```typescript
const model = models.getModel('openai', 'gpt-4o-mini')!;
if (model.input.includes('image')) {
  const imageBuffer = readFileSync('image.png');
  const base64Image = imageBuffer.toString('base64');
  const response = await models.complete(model, {
    messages: [{
      role: 'user',
      content: [
        { type: 'text', text: 'What is in this image?' },
        { type: 'image', data: base64Image, mimeType: 'image/png' }
      ],
      timestamp: Date.now()
    }]
  });
}
```

### Image Generation

```typescript
const model = models.getModelOfType('image', 'openrouter', 'google/gemini-2.5-flash-image')!;
const result = await models.generateImages(model, {
  input: [{ type: 'text', text: 'Generate a red circle on a plain white background.' }]
});
```

### Classification

```typescript
const model = models.getModelOfType('classifier', 'typesafe', 'jev-latest')!;
const result = await models.classify(model, {
  state: { message: 'The change works perfectly, thanks.' },
  questions: {
    category: {
      type: 'choice',
      instructions: 'Classify the message.',
      criteria: { approval: 'The user approves', correction: 'The user requests a correction' }
    },
    satisfaction: {
      type: 'score',
      instructions: 'Score user satisfaction.',
      criteria: ['dissatisfied', 'neutral', 'satisfied']
    },
    approved: {
      type: 'bool',
      instructions: 'Does the user approve?',
      criteria: { true: 'Approval', false: 'No approval' }
    }
  }
});
console.log(result.answers);
```

### Custom Providers (Ollama example)

```typescript
import { createModels, createProvider, envApiKeyAuth, type Model } from '@earendil-works/pi-ai';
import { openAICompletionsApi } from '@earendil-works/pi-ai/api/openai-completions.lazy';

const ollamaModel: Model<'openai-completions'> = {
  id: 'llama-3.1-8b',
  name: 'Llama 3.1 8B (Ollama)',
  api: 'openai-completions',
  provider: 'ollama',
  baseUrl: 'http://localhost:11434/v1',
  reasoning: false,
  input: ['text'],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 32000
};

const ollama = createProvider({
  id: 'ollama',
  name: 'Ollama',
  baseUrl: 'http://localhost:11434/v1',
  auth: { apiKey: { name: 'Ollama', resolve: async () => ({ auth: {} }) } },
  models: [ollamaModel],
  api: openAICompletionsApi(),
});

const models = createModels();
models.setProvider(ollama);
await models.complete(models.getModel('ollama', 'llama-3.1-8b')!, context);
```

### Faux Provider for Tests

```typescript
import {
  createModels,
  fauxAssistantMessage,
  fauxProvider,
  fauxText,
  fauxThinking,
  fauxToolCall,
} from '@earendil-works/pi-ai';

const faux = fauxProvider({ tokensPerSecond: 50 });
const models = createModels();
models.setProvider(faux.provider);
const model = faux.getModel();

faux.setResponses([
  fauxAssistantMessage([
    fauxThinking('Need to inspect package metadata first.'),
    fauxToolCall('echo', { text: 'package.json' })
  ], { stopReason: 'toolUse' })
]);

const first = await models.complete(model, context, {
  sessionId: 'session-1',
  cacheRetention: 'short'
});
```

### Cross-Provider Handoffs

```typescript
const models = createModels();
models.setProvider(anthropicProvider());
models.setProvider(openaiProvider());
models.setProvider(googleProvider());

const context: Context = { messages: [] };

// Start with Claude
const claude = models.getModel('anthropic', 'claude-sonnet-4-5')!;
context.messages.push({ role: 'user', content: 'What is 25 * 18?', timestamp: Date.now() });
context.messages.push(await models.completeSimple(claude, context, { reasoning: 'medium' }));

// Switch to GPT-5 - it will see Claude's thinking as <thinking> tagged text
const gpt5 = models.getModel('openai', 'gpt-5-mini')!;
context.messages.push({ role: 'user', content: 'Is that calculation correct?', timestamp: Date.now() });
context.messages.push(await models.complete(gpt5, context));

// Switch to Gemini
const gemini = models.getModel('google', 'gemini-2.5-flash')!;
context.messages.push({ role: 'user', content: 'What was the original question?', timestamp: Date.now() });
const geminiResponse = await models.complete(gemini, context);
```

### Context Serialization

```typescript
const context: Context = {
  systemPrompt: 'You are a helpful assistant.',
  messages: [{ role: 'user', content: 'What is TypeScript?', timestamp: Date.now() }]
};

const response = await models.complete(model, context);
context.messages.push(response);
const serialized = JSON.stringify(context);
localStorage.setItem('conversation', serialized);

// Later: deserialize and continue
const restored: Context = JSON.parse(localStorage.getItem('conversation')!);
restored.messages.push({ role: 'user', content: 'Tell me more', timestamp: Date.now() });
const continuation = await models.complete(newModel, restored);
```

### Browser Usage

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel('anthropic', 'claude-3-5-haiku-20241022')!;
const response = await models.complete(model, {
  messages: [{ role: 'user', content: 'Hello!', timestamp: Date.now() }]
}, {
  apiKey: 'your-api-key'
});
```

### OAuth Login

```typescript
import { createModels } from '@earendil-works/pi-ai';
import { anthropicProvider } from '@earendil-works/pi-ai/providers/anthropic';

const models = createModels({ credentials: myStore });
models.setProvider(anthropicProvider());

await models.login('anthropic', 'oauth', {
  prompt: async (p) => { return await askUser(p.message); },
  notify: (event) => { console.log(event.message); },
});

const model = models.getModel('anthropic', 'claude-sonnet-4-5')!;
await models.complete(model, context);

await models.logout('anthropic');
```

---

## ⚙️ 技术栈

- TypeScript with TypeBox schemas
- Multiple SDK backends: `@anthropic-ai/sdk`, `openai`, `@google/genai`, AWS SDK, etc.
- Lazy-loaded provider SDKs for tree shaking
- Node.js and browser environments
- License: MIT

## Development

### Adding a New Provider

1. **Core Types** (`src/types.ts`): Add API identifier, provider name, options type
2. **API Implementation** (`src/api/<api-id>.ts`): Create stream/streamSimple with message conversion and response parsing
3. **Model Generation** (`scripts/generate-models.ts`): Fetch and parse models, map to Model/ImageModel/ClassifierModel
4. **Provider Factory** (`src/providers/<id>.ts`): Wire catalog + auth + lazy API wrapper; register in `all.ts`
5. **Tests** (`test/`): Cover streaming, tokens, abort, empty, context-overflow, image-limits, unicode, tool-call-without-result, image-tool-result, total-tokens, cross-provider-handoff, providers
6. **Coding Agent Integration** (`../coding-agent/`): Update model-resolver, CLI args, README
7. **Documentation**: Update this README (Supported Providers, Environment Variables)
8. **Changelog**: Add entry to `packages/ai/CHANGELOG.md`