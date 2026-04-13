x---
title: "Vercel AI SDK Auto-Tracing — Final"
type: reference
status: shipped
date: 2026-04-14
supersedes:
  - docs/vercel-ai-auto-tracing-overview.md
  - docs/vercel-ai-auto-tracing-overview-v2.md
  - docs/plans/vercel-v4-v5-v6.md
  - docs/plans/auto_tracing_vercel.md
  - docs/plans/auto_tracing_vercel_v2.md
  - docs/plans/vercel-expanded-scope.md
  - docs/plans/vercel-llm-config-audit.md
---

# Vercel AI SDK Auto-Tracing — Final

This is the single source of truth for the Vercel AI SDK instrumentation in `@browserstack/ai-sdk`. It consolidates the implementation plan, the architectural decisions, the upstream issues we've hit, and every fix shipped through PR #961.

---

## 1. Overview

### What this is

A wrapper that adds OpenTelemetry tracing to every call made through the [Vercel AI SDK](https://ai-sdk.dev) (`ai` npm package) — `generateText`, `streamText`, `generateObject`, `streamObject`, `embed`, `embedMany`, `generateImage`, `rerank`, plus `Agent` and `ToolLoopAgent` classes.

### User API

One line, opt-in:

```ts
import { Observe } from '@browserstack/ai-sdk';
import * as ai from 'ai';

Observe.init();
const { generateText, streamText, ToolLoopAgent } = Observe.wrapVercelAI(ai);

const result = await generateText({
  model: openai('gpt-4o'),
  prompt: 'Hello',
});
```

Calling `Observe.wrapVercelAI(ai)` returns a Proxy over the `ai` module. Every wrapped function emits `vercel-ai.*` OTel spans automatically.

### Versions covered

| `ai` package | AI SDK version | Status |
|---|---|---|
| `ai@4.x` | AI SDK 4 | Best-effort (v4 compatibility layer) |
| `ai@5.x` | AI SDK 5 | Full support |
| `ai@6.x` | AI SDK 6 | Full support (current default) |
| `ai@7.x-beta` | AI SDK 7 | Out of scope (separate plan) |

`v4 + v5 + v6` covers ~95% of `ai`-package downloads.

---

## 2. What we instrument

### Top-level functions and span names

| User-facing call | Top-level span | Per-LLM-call child span | Tool child spans |
|---|---|---|---|
| `generateText` | `vercel-ai.generateText` | `vercel-ai.doGenerate` | `vercel-ai.tool.<name>` |
| `streamText` | `vercel-ai.streamText` | `vercel-ai.doStream` | `vercel-ai.tool.<name>` |
| `generateObject` | `vercel-ai.generateObject` | `vercel-ai.doGenerate` | — |
| `streamObject` | `vercel-ai.streamObject` | `vercel-ai.doStream` | — |
| `embed` | `vercel-ai.embed` | — | — |
| `embedMany` | `vercel-ai.embedMany` | — | — |
| `generateImage` | `vercel-ai.generateImage` | — | — |
| `rerank` | `vercel-ai.rerank` | — | — |
| `Agent.generate` / `ToolLoopAgent.generate` | `vercel-ai.agent.generate` | `vercel-ai.doGenerate` (one per loop step) | `vercel-ai.tool.<name>` (one per tool call) |
| `Agent.stream` / `ToolLoopAgent.stream` | `vercel-ai.agent.stream` | `vercel-ai.doStream` (one per loop step) | `vercel-ai.tool.<name>` |

### Why `doGenerate` / `doStream` exist as their own spans

In an agent loop, the LLM is called multiple times — decide → execute tool → decide again → execute another tool → final answer. Each of those LLM calls goes through `model.doGenerate()` (or `model.doStream()` for streaming). We patch these methods on the model instance so each individual LLM call produces its own child span — same approach as Braintrust's `wrapAISDK`.

For plain (non-agent) `generateText`, you get a redundant `generateText` parent + one `doGenerate` child. That's intentional consistency — same as Braintrust.

### Span hierarchy by scenario

**Plain generateText with tools** (single LLM call, may emit parallel tool calls):
```
vercel-ai.generateText
├── vercel-ai.doGenerate              ← LLM decides "call X, Y, Z"
├── vercel-ai.tool.X
├── vercel-ai.tool.Y
└── vercel-ai.tool.Z
```

**Agent with multi-step loop** (3 LLM calls + 2 tool calls):
```
vercel-ai.agent.generate
├── vercel-ai.doGenerate              ← step 1: LLM decides "call calculator"
├── vercel-ai.tool.calculator
├── vercel-ai.doGenerate              ← step 2: LLM decides "call calculator again"
├── vercel-ai.tool.calculator
└── vercel-ai.doGenerate              ← step 3: final answer (finishReason=stop)
```

**Multi-agent with sub-agents nested inside tools**:
```
vercel-ai.agent.generate (coordinator)
├── vercel-ai.doGenerate              ← coordinator step 1
├── vercel-ai.tool.askResearcher
│   └── vercel-ai.agent.generate      ← researcher SUB-AGENT (UI shows as "Object.execute")
│       └── vercel-ai.doGenerate      ← sub-agent's LLM call
├── vercel-ai.doGenerate              ← coordinator step 2
├── vercel-ai.tool.askWriter
│   └── vercel-ai.agent.generate      ← writer SUB-AGENT
│       └── vercel-ai.doGenerate      ← sub-agent's LLM call
└── vercel-ai.doGenerate              ← coordinator final answer
```

---

## 3. What we capture per span

### Request attributes (all spans)

| Attribute | Source | Notes |
|---|---|---|
| `gen_ai.system` | `model.provider` / `providerId` | e.g. `openai.chat`, `anthropic` |
| `gen_ai.request.model` | `model.modelId` | e.g. `gpt-4.1-mini` |
| `gen_ai.operation.name` | `chat` / `embeddings` / `image_generation` / `agent` / `tool` | OTel semantic convention |
| `gen_ai.request.temperature` | `options.temperature` | |
| `gen_ai.request.max_tokens` | `options.maxOutputTokens ?? options.maxTokens` | v4 fallback |
| `gen_ai.request.top_p` / `top_k` | `options.topP` / `topK` | |
| `gen_ai.request.frequency_penalty` / `presence_penalty` | passed through | |
| `gen_ai.request.seed` | `options.seed` | |
| `gen_ai.request.stop_sequences` | `options.stopSequences` | JSON array |
| `vercel_ai.tool_choice` | `options.toolChoice` | `auto` / `none` / `required` / `{type: 'tool', toolName}` |
| `vercel_ai.schema_name` | `options.schemaName` | generateObject only |
| `vercel_ai.provider_options` | `options.providerOptions ?? options.providerMetadata` | JSON, truncated at 8KB with warn log |
| `vercel_ai.max_retries` | `options.maxRetries` | |
| `vercel_ai.timeout_ms` | `options.timeout` | |
| `AIOPS_INTERNAL.observation.input` | structured JSON | prompt, messages, system, tools, instructions; truncated if >50KB |

### Response attributes

| Attribute | Source | Notes |
|---|---|---|
| `gen_ai.usage.input_tokens` | `usage.inputTokens ?? usage.promptTokens` | v4 fallback |
| `gen_ai.usage.output_tokens` | `usage.outputTokens ?? usage.completionTokens` | |
| `gen_ai.usage.total_tokens` | `usage.totalTokens` or sum | All 3 are coerced to finite numbers (see fix #2) |
| `vercel_ai.usage.reasoning_tokens` | `usage.outputTokenDetails.reasoningTokens` | |
| `vercel_ai.usage.cached_input_tokens` | `usage.inputTokenDetails.cachedTokens` / `cacheReadTokens` | both field names supported |
| `vercel_ai.usage.cache_write_tokens` | `usage.inputTokenDetails.cacheCreationTokens` / `cacheWriteTokens` | both field names supported |
| `vercel_ai.usage.no_cache_tokens` | `usage.inputTokenDetails.noCacheTokens` | OpenAI-specific |
| `vercel_ai.usage.text_tokens` | `usage.outputTokenDetails.textTokens` | OpenAI-specific |
| `gen_ai.response.finish_reasons` | normalized string from `result.finishReason` | v6 returns `{unified, raw}` object — we extract `.unified` (see fix) |
| `gen_ai.response.id` | `result.response.id` | |
| `gen_ai.response.model` | `result.response.modelId` | may differ from request model |
| `vercel_ai.steps_count` | `result.steps.length` | multi-step agent runs |
| `vercel_ai.time_to_first_token_ms` | measured | streams only |
| `vercel_ai.gateway.cost` / `gateway.market_cost` | `result.providerMetadata.gateway.*` | when using Vercel AI Gateway |
| `vercel_ai.response.provider_metadata` | full `providerMetadata` blob | JSON, truncated at 8KB |
| `AIOPS_INTERNAL.observation.output` | structured JSON | text, reasoning, toolCalls, stepsCount, finishReason |

### Tool span attributes

| Attribute | Source |
|---|---|
| `gen_ai.operation.name` | `tool` |
| `vercel_ai.tool.name` | tool name from definition |
| `AIOPS_INTERNAL.observation.input` | tool args (JSON) |
| `AIOPS_INTERNAL.observation.output` | tool result (JSON) |

### `doGenerate` / `doStream` span attributes

Same shape as the top-level span (since it's just one LLM call): `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.*`, `gen_ai.response.*`, `OBSERVATION_INPUT`, `OBSERVATION_OUTPUT`. Allows per-LLM-call drilldown.

---

## 4. Architectural decisions

### D1. Import wrapping over prototype patching

Three approaches were considered:

| Approach | Provider-agnostic | Full SDK data | ESM + CJS | Zero-config |
|---|---|---|---|---|
| CJS require cache patching | Yes | Yes | CJS only | Yes |
| Provider prototype patching (`doGenerate`/`doStream`) | No (hardcoded list) | No (lossy) | Yes | Yes |
| **Import wrapping (chosen)** | **Yes** | **Yes** | **Yes** | **One line** |

**Why import wrapping won:**
- Captures full SDK-level data (prompt, messages, tools, totalUsage, steps_count) — prototype patching loses these
- Provider-agnostic — no hardcoded provider list
- Works for both CJS and ESM out of the box
- Trade-off: user adds one line `Observe.wrapVercelAI(ai)` instead of zero. Same pattern as Braintrust's `wrapAISDK(ai)`.

### D2. Per-LLM-call `doGenerate` / `doStream` spans (added later)

When agents loop through multiple tool calls, the user wants to see each LLM round-trip individually — not just an aggregated agent span. Braintrust does this; we initially didn't.

**Implementation:** `wrapLanguageModel(tracer, model)` patches `model.doGenerate` and `model.doStream` in place using a `Symbol.for('vercel-ai.language-model.wrapped')` marker for idempotency. Detection: model must have BOTH `doGenerate` AND `doStream` methods (rules out EmbeddingModel, ImageModel, RerankingModel automatically).

**Trade-off:** plain (non-agent) `generateText` now produces a parent + one redundant `doGenerate` child. Acceptable for consistency — matches Braintrust's behavior.

### D3. Don't mutate user tool objects (root fix)

Initial implementation mutated `tool.execute` in place on the agent instance, so the agent's internal `generateText` call would emit tool spans. **Side effect:** if the same tool was reused later in a plain `generateText`, the existing wrap would compose with our new wrap → nested duplicate spans.

**Root fix:** in `createAgentClassWrapper`'s construct trap, build a NEW tools object with wrapped clones, then replace `instance.settings.tools` (the agent's internal storage). The user's original tool objects stay untouched.

Verified by `test-mutation-check.cjs`:
```js
const originalRef = weatherTool.execute;
new ToolLoopAgent({ tools: { weatherTool } });
weatherTool.execute === originalRef  // ✅ true after fix
```

### D4. No per-step LLM spans via `onStepFinish`

Considered injecting `onStepFinish` to emit a span per agent step. Rejected because:
- Braintrust and Google ADK don't do it
- The information lives on the parent agent span (`steps_count`, aggregated `usage`) plus the per-`doGenerate` spans
- Adds visual clutter for limited extra value

### D5. `wrapLanguageModel` skips embedding/image/rerank models

These have a single API call per user call, not a loop. Their top-level span (`vercel-ai.embed`, `vercel-ai.generateImage`, `vercel-ai.rerank`) already captures everything. A `doGenerate`-style child would be 100% redundant. Plus their model interfaces don't have `doStream` so detection naturally skips them.

### D6. Stream wrapper handles `onError` / `onAbort`

If a stream errors mid-flight (rate limit, network drop) or is aborted, `onFinish` never fires → span would never end → memory leak + the trace shows a hanging span forever.

**Fix:** patch `onError` and `onAbort` callbacks alongside `onFinish` / `onChunk`. Use a `spanEnded` flag so the span ends exactly once across all callbacks.

### D7. `finishReason` normalization

In v6, `LanguageModelV2.doGenerate()` and `onFinish` events return `finishReason` as `{ unified: 'stop', raw: 'stop' }` instead of a string. Setting an object as a span attribute renders as `"[object Object]"` in the UI.

**Fix:** central `extractFinishReason()` helper returns `fr.unified ?? fr.raw ?? fr` if the input is a string, otherwise normalizes to a string. Used everywhere we set `gen_ai.response.finish_reasons`.

### D8. Number coercion on token attributes

Some SDK versions/providers occasionally return object-shaped values (e.g., `{}`) for token fields. Doing `inputTokens + outputTokens` on objects produces the JS classic `"[object Object][object Object]"` and gets stored as a span attribute string. Coerce to finite numbers everywhere via `asNumber(v) = typeof v === 'number' && Number.isFinite(v) ? v : undefined`.

### D9. Image mask binary protection

`generateImage` accepts a prompt of shape `{ text, mask? }`. Originally the wrapper had `input.prompt = options.prompt.text ?? options.prompt` — fallback would serialize the entire object including the binary mask. Removed the fallback; if `prompt.text` isn't a string, we capture only `hasMask: true` and omit the body.

### D10. Embed value truncation

`embed` and `embedMany` inputs were previously serialized untruncated. Added `truncateEmbedValue` and `truncateEmbedValues` helpers (mirror `truncateMessages`'s 50KB threshold). Bounded in practice by the model's input limit anyway, but defensive against future model versions or unusual inputs.

### D11. Provider options truncation warning

Provider options are JSON-stringified, truncated at 8KB. Now logs `logger.warn` when truncation happens so debugging stale `providerOptions` isn't silent.

---

## 5. v4 compatibility

v4 has a different API surface. We support it via fallback field reads.

| Area | v4 | v5+ | Wrapper handles via |
|---|---|---|---|
| Max tokens param | `maxTokens` | `maxOutputTokens` | `options.maxOutputTokens ?? options.maxTokens` |
| Input token usage | `usage.promptTokens` | `usage.inputTokens` | `usage.inputTokens ?? usage.promptTokens` |
| Output token usage | `usage.completionTokens` | `usage.outputTokens` | `usage.outputTokens ?? usage.completionTokens` |
| Provider options | `providerMetadata` | `providerOptions` | `options.providerOptions ?? options.providerMetadata` |
| Embed response | `rawResponse` | `response` | `result.response ?? result.rawResponse` |
| Tool parameters | `parameters` | `inputSchema` | Read both when serializing tool definitions |
| Tool call fields | `args` / `result` | `input` / `output` | Read both when extracting tool calls |

**v4 message structure:** v4 uses `content` as a plain string. v5+ uses a `parts` array. Our input extraction handles both.

**v4 `onFinish` shape:** Different from v5+ (no `totalUsage`). Detect via `event.totalUsage` existence.

**v4 lacks:** `Agent` / `ToolLoopAgent` classes, `rerank`, `inputTokenDetails` / `outputTokenDetails`, `totalUsage` (use `result.usage` for last step only).

---

## 6. Known issues & limitations

### Issue #9 — AI SDK v6 tool property name mismatch (UPSTREAM)

**Status:** [vercel/ai#12020](https://github.com/vercel/ai/issues/12020) — OPEN
**Affected:** `ai@6.x` + any `@ai-sdk/*` provider.
**Affects all schema types** (`jsonSchema()`, `z.object()` Zod v3 and v4) — NOT a Zod-specific issue.

**Errors:**
- Anthropic: `tools.0.custom.input_schema.type: Field required`
- OpenAI: `Invalid schema for function: schema must be a JSON Schema of 'type: "object"', got 'type: "None"'`

**Root cause:** The `tool()` helper is an identity function. AI SDK docs and examples define tools with `parameters`:
```js
tool({ parameters: z.object({ city: z.string() }), execute: ... })
```
But `prepareToolsAndToolChoice()` reads `tool.inputSchema`. Since the tool object has `parameters` but NOT `inputSchema`, `tool.inputSchema` is `undefined` and `asSchema(undefined)` returns `{"properties":{},"additionalProperties":false}` — missing `type`, properties contents, and `required`. Provider rejects.

**Workaround:** Use `inputSchema` instead of `parameters`:
```js
tool({
  description: 'Get weather',
  inputSchema: jsonSchema({ type: 'object', properties: { city: { type: 'string' } }, required: ['city'] }),
  execute: async ({ city }) => ({ city, temp: 22 }),
})
```

**Affected methods:** `generateText` / `streamText` / `generateObject` (with tools), `agent.generate` / `agent.stream`.
**Not affected:** all calls without `tools`, plus `embed` / `embedMany` / `generateImage` / `rerank`.

**Impact on us:** None — we pass tools through unchanged. The mismatch is in the AI SDK core.

### Limitation: `generateText` doesn't loop in v6

Plain `generateText({ tools, maxSteps: 10 })` makes one LLM call, executes whatever tools it asked for, then stops. It does NOT loop back to synthesize a final answer in v6. Use `ToolLoopAgent` for that.

### Limitation: Cost shows $0 for embeddings (Google) and image generation

- **Google embedding** returns `usage.tokens: null`. Our wrapper correctly omits the field. The cost engine has no token count to multiply by price → $0.
- **DALL-E** returns `usage.imagesGenerated` only. Cost is per-image, not per-token. Backend cost engine is token-based → $0.

Both are observability-pipeline gaps in the backend, not bugs in our SDK. The data needed to compute cost (image count, size, model) IS captured on the span.

### Limitation: Sub-agent spans display as `Object.execute` in the UI

Span name internally is `vercel-ai.agent.generate`. The trace UI displays it by `code.function` attribute (which is `Object.execute` because the calling tool's `execute: async () => {...}` is on an object literal). UI rendering preference, not a span data issue.

### Limitation: First child of a generation-type span appears one indent deeper in the UI

Verified end-to-end via REST API: all sibling spans correctly share the same `parentObservationId`. The visual indentation difference is a renderer quirk in the trace viewer, not a real parent-child relationship issue. File against the trace-viewer team if it causes confusion.

### Resolved issues found during testing

Historical record of issues caught and resolved during development. Issue #9 (above) is the only one still active (upstream).

| # | Issue | Status |
|---|---|---|
| 1 | `json: undefined` in messages | Frontend display issue (not us) |
| 2 | `cached_input_tokens` always 0 | Correct provider value when no caching |
| 3 | `stopSequences` stopped early | Test working correctly |
| 4 | Tool schema raw Zod internals | **Fixed** — `safeSerializeSchema()` |
| 5 | Missing schema in generateObject input | **Fixed** — captures `options.schema` |
| 6 | Embed `tokens: null` | **Fixed** — `!= null` check |
| 7 | generateImage missing cost | **Improved** — captures full `providerMetadata` |
| 8 | OpenAI `no_cache_tokens` extra fields | Correct provider value |
| 9 | Tool calls fail | **Upstream bug** (above) |

---

## 7. What we do NOT trace

| Excluded | Why |
|---|---|
| `experimental_generateSpeech` | Still experimental. Add when stable. |
| `experimental_transcribe` | Still experimental. Add when stable. |
| Provider HTTP calls | Already traced by separate OpenAI / Anthropic / Bedrock instrumentations. Tracing both layers would create duplicate spans. |
| Per-step LLM spans (via `onStepFinish`) | Already covered by per-`doGenerate` spans. Adding step-level spans would duplicate. |
| `headers` / `abortSignal` | Request infrastructure, not observability-relevant. |
| Embedding / image / rerank `doGenerate` spans | Single-call operations — top-level span already captures everything. |

---

## 8. Files

| File | Role |
|---|---|
| `src/instrumentations/vercel-ai/wrap-vercel-ai.ts` | The wrapper (single file, ~1500 LOC) |
| `src/client.ts` | `Observe.wrapVercelAI` public API |
| `src/index.ts` | Re-export `wrapVercelAI` |
| `tests/featureTest/auto-tracing/wrap-vercel-ai.test.ts` | 60+ unit tests |
| `tests/llm-providers/auto-tracing-vercel-ai-agent.cjs` | Live integration test (runs against real OpenAI + local backend) |
| `docs/plans/vercel-ai-final.md` | This document |

---

## 9. Demo routes (in `demo-application/nodejs`)

The demo app exposes 19 routes for the `vercel_ai` provider. Recommended demo subset (covers every unique span shape):

| Route | What it shows |
|---|---|
| `POST /generate` | Plain `generateText` → `doGenerate` |
| `POST /stream` | `streamText` → `doStream` |
| `POST /chat-generate` | Same as `/generate` but messages-based |
| `POST /chat-stream` | Same as `/stream` but messages-based |
| `POST /embed` / `POST /batch-embed` | Embedding (single span, no doGenerate child by design) |
| `POST /function-call` | `generateText` + 1 tool |
| `POST /function-call-multi` | `generateText` + 3 parallel tools (one LLM call) |
| `POST /function-call-required` | `toolChoice: 'required'` |
| `POST /system-instruction` | System prompt |
| `POST /structured-output` | `generateObject` |
| `POST /agent-run` | `ToolLoopcan iAgent.generate` — alternating doGenerate/tool/doGenerate |
| `POST /agent-stream` | `ToolLoopAgent.stream` |
| `POST /multi-agent` | Coordinator + 2 sub-agents (nested via tool bodies) |
| `POST /nested-agents` | Main agent → tool → sub-agent → tool (deepest nesting) |
| `POST /image-generation` | `generateImage` (DALL-E 3) |

---

## 10. Recent fixes timeline (PR #961)

In rough order of when they shipped during the PR:

1. **Property name mismatch documented** — Issue #9 raised against upstream; user-facing workaround documented.
2. **`createAgentStreamWrapper` Promise handling** — `ToolLoopAgent.stream()` returns `PromiseLike` in v6; wrapper now awaits.
3. **Per-tool child spans on agents** — added `wrapToolExecutions`-equivalent at the agent construct trap.
4. **AGENT_TOOL_WRAPPED idempotency marker** — interim fix for nested duplicate tool spans.
5. **Root mutation fix** — replaced the marker fix with proper non-mutating clone of `instance.settings.tools`. User tool objects are no longer modified.
6. **`wrapLanguageModel` (`doGenerate` / `doStream`)** — per-LLM-call child spans, matches Braintrust shape.
7. **`finishReason` normalization** — central `extractFinishReason()` handles v6's `{unified, raw}` object shape.
8. **Token coercion** — `asNumber` everywhere; eliminates `[object Object][object Object]` from token attrs.
9. **`onError` / `onAbort` for streams** — span no longer leaks when stream errors or is aborted before `onFinish`.
10. **Image mask binary leak** — fallback removed; only `hasMask: true` is captured if `prompt.text` is missing.
11. **Embed value truncation** — bounded by `MAX_INPUT_MESSAGES_BYTES` (50 KB).
12. **Provider options truncation warning** — `logger.warn` emitted when serialized options exceed 8KB.

All changes live in `src/instrumentations/vercel-ai/wrap-vercel-ai.ts`. Net diff for PR: ~1500 LOC added (wrapper) + the supporting test files.

---

## 11. References

- Upstream bug: [vercel/ai#12020](https://github.com/vercel/ai/issues/12020)
- Vercel AI SDK docs: https://ai-sdk.dev
- v5 migration guide: https://ai-sdk.dev/docs/migration-guides/migration-guide-5-0
- v6 migration guide: https://ai-sdk.dev/docs/migration-guides/migration-guide-6-0
- OTel GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
- Braintrust `wrapAISDK` source (reference for `doGenerate` span shape): `github.com/braintrustdata/braintrust-sdk/js/src/wrappers/ai-sdk/ai-sdk.ts`
- Demo app: `demo-application/nodejs/providers/vercel_ai.js`
