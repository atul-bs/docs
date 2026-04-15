---
title: "Vercel AI SDK Auto-Tracing — Final"
tags:
  - reference
  - instrumentation
  - vercel-ai
  - auto-tracing
  - shipped
  - observability
  - otel
type: reference
status: shipped
date: 2026-04-15
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

> [!summary]
> This is the **single source of truth** for the [[Vercel AI SDK]] instrumentation in `@browserstack/ai-sdk`. It consolidates the implementation plan, architectural decisions, upstream issues, and every fix shipped through PR #961.

---

## 1. Overview

### What This Is

A wrapper that adds [[OpenTelemetry]] tracing to every call made through the [[Vercel AI SDK]] (`ai` npm package) — `generateText`, `streamText`, `generateObject`, `streamObject`, `embed`, `embedMany`, `generateImage`, `rerank`, plus `Agent` and `ToolLoopAgent` classes.

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

### Versions Covered

| `ai` package | AI SDK version | Status |
| --- | --- | --- |
| `ai@4.x` | AI SDK 4 | Best-effort ([[#5. v4 Compatibility]]) |
| `ai@5.x` | AI SDK 5 | ✅ Full support |
| `ai@6.x` | AI SDK 6 | ✅ Full support (current default) |
| `ai@7.x-beta` | AI SDK 7 | ❌ Out of scope (separate plan — will use `registerTelemetryIntegration`) |

> [!note]
> `v4 + v5 + v6` covers ~95% of `ai`-package downloads.

---

## 2. What We Instrument

### Top-Level Functions and Span Names

| User-facing call | Top-level span | Per-LLM-call child span | Tool child spans |
| --- | --- | --- | --- |
| `generateText` | `vercel-ai.generateText` | `vercel-ai.doGenerate` | `vercel-ai.tool.<n>` |
| `streamText` | `vercel-ai.streamText` | `vercel-ai.doStream` | `vercel-ai.tool.<n>` |
| `generateObject` | `vercel-ai.generateObject` | `vercel-ai.doGenerate` | — |
| `streamObject` | `vercel-ai.streamObject` | `vercel-ai.doStream` | — |
| `embed` | `vercel-ai.embed` | — | — |
| `embedMany` | `vercel-ai.embedMany` | — | — |
| `generateImage` | `vercel-ai.generateImage` | — | — |
| `rerank` | `vercel-ai.rerank` | — | — |
| `Agent.generate` / `ToolLoopAgent.generate` | `vercel-ai.agent.generate` | `vercel-ai.doGenerate` (one per loop step) | `vercel-ai.tool.<n>` (one per tool call) |
| `Agent.stream` / `ToolLoopAgent.stream` | `vercel-ai.agent.stream` | `vercel-ai.doStream` (one per loop step) | `vercel-ai.tool.<n>` |

### Why `doGenerate` / `doStream` Exist as Their Own Spans

> [!info]
> In an agent loop, the LLM is called multiple times — decide → execute tool → decide again → execute another tool → final answer. Each of those LLM calls goes through `model.doGenerate()` (or `model.doStream()` for streaming). We patch these methods on the model instance so each individual LLM call produces its own child span — same approach as [[Braintrust]]'s `wrapAISDK`.
>
> For plain (non-agent) `generateText`, you get a redundant `generateText` parent + one `doGenerate` child. That's intentional consistency — matches [[Braintrust]].

### Span Type Classification (SPAN vs GENERATION)

> [!important]
> Parent wrapper spans are classified as **SPAN**; the per-LLM-call child spans are classified as **GENERATION**. This split is what produces a clean "agent = span, LLM call = generation" trace in the UI, and it's what prevents token cost from being double-counted.

**Classification rules:**

| Span | UI Type | `SpanKind` | Carries `gen_ai.*`? | Carries cost? |
| --- | --- | --- | --- | --- |
| `vercel-ai.generateText` | **SPAN** | `INTERNAL` | ❌ | ❌ |
| `vercel-ai.streamText` | **SPAN** | `INTERNAL` | ❌ | ❌ |
| `vercel-ai.generateObject` | **SPAN** | `INTERNAL` | ❌ | ❌ |
| `vercel-ai.streamObject` | **SPAN** | `INTERNAL` | ❌ | ❌ |
| `vercel-ai.agent.generate` | **SPAN** | `INTERNAL` | ❌ | ❌ |
| `vercel-ai.agent.stream` | **SPAN** | `INTERNAL` | ❌ | ❌ |
| `vercel-ai.doGenerate` | **GENERATION** | `CLIENT` | ✅ | ✅ |
| `vercel-ai.doStream` | **GENERATION** | `CLIENT` | ✅ | ✅ |
| `vercel-ai.tool.<n>` | **SPAN** | `INTERNAL` | only `gen_ai.operation.name: 'execute_tool'` | ❌ |
| `vercel-ai.embed` / `embedMany` | **GENERATION** | `CLIENT` | ✅ | ✅ |
| `vercel-ai.generateImage` | **GENERATION** | `CLIENT` | ✅ | ✅ |
| `vercel-ai.rerank` | **GENERATION** | `CLIENT` | ✅ | ✅ |

> [!warning] Backend classification rule
> The backend classifies **any span with `gen_ai.*` attributes as GENERATION**, regardless of `SpanKind`. This is why the wrapper must **strip all `gen_ai.*` attributes from parent wrapper spans** — simply setting `SpanKind.INTERNAL` is not enough. The wrapper keeps only `vercel_ai.*` and `AIOPS_INTERNAL.*` attributes on parent spans.

**Why `embed` / `embedMany` / `generateImage` / `rerank` are exceptions:**
These have no `doGenerate` child span (they're single-call operations). If we stripped `gen_ai.*` from their parent, the LLM metadata would be lost entirely. We keep `gen_ai.*` on the parent specifically because there is no child to carry it.

**Why cost only appears on the GENERATION child:**
The cost engine multiplies token counts by model pricing. If both parent and child carried `gen_ai.usage.*`, cost would be computed twice and aggregated — the trace would show 2× the true cost. Removing `gen_ai.usage.*` from parents fixes this at the source. See [[#D12. Parent Spans as SPAN, Not GENERATION]].

### Span Hierarchy by Scenario

**Plain generateText with tools** (single LLM call, may emit parallel tool calls):

```
vercel-ai.generateText                  ← SPAN
├── vercel-ai.doGenerate                ← GENERATION (LLM decides "call X, Y, Z")
├── vercel-ai.tool.X                    ← SPAN
├── vercel-ai.tool.Y                    ← SPAN
└── vercel-ai.tool.Z                    ← SPAN
```

**Agent with multi-step loop** (3 LLM calls + 2 tool calls):

```
vercel-ai.agent.generate                ← SPAN
├── vercel-ai.doGenerate                ← GENERATION (step 1: "call calculator")
├── vercel-ai.tool.calculator           ← SPAN
├── vercel-ai.doGenerate                ← GENERATION (step 2: "call calculator again")
├── vercel-ai.tool.calculator           ← SPAN
└── vercel-ai.doGenerate                ← GENERATION (step 3: final answer)
```

**Multi-agent with sub-agents nested inside tools:**

```
vercel-ai.agent.generate (coordinator)  ← SPAN
├── vercel-ai.doGenerate                ← GENERATION (coordinator step 1)
├── vercel-ai.tool.askResearcher        ← SPAN
│   └── vercel-ai.agent.generate        ← SPAN (researcher SUB-AGENT)
│       └── vercel-ai.doGenerate        ← GENERATION (sub-agent's LLM call)
├── vercel-ai.doGenerate                ← GENERATION (coordinator step 2)
├── vercel-ai.tool.askWriter            ← SPAN
│   └── vercel-ai.agent.generate        ← SPAN (writer SUB-AGENT)
│       └── vercel-ai.doGenerate        ← GENERATION (sub-agent's LLM call)
└── vercel-ai.doGenerate                ← GENERATION (coordinator final answer)
```

---

## 3. What We Capture Per Span

> [!note] Attribute scope
> The tables below list every attribute the wrapper can produce. Scope rules:
> - `gen_ai.*` attributes live **only** on `doGenerate` / `doStream` children **and** on single-call leaf operations (`embed`, `embedMany`, `generateImage`, `rerank`). They are **NOT** set on `generateText`, `streamText`, `generateObject`, `streamObject`, `agent.generate`, or `agent.stream` parent wrapper spans.
> - `vercel_ai.*` and `AIOPS_INTERNAL.*` attributes live on **all** wrapper spans (parent and child).
> - Cost is computed from `gen_ai.usage.*`, so cost follows the `gen_ai.*` placement — child only for text/object/agent flows, parent for embed/image/rerank.

### Request Attributes

| Attribute | Source | Notes |
| --- | --- | --- |
| `gen_ai.system` | `model.provider` / `providerId` | e.g. `openai.chat`, `anthropic` — child/leaf only |
| `gen_ai.request.model` | `model.modelId` | e.g. `gpt-4.1-mini` — child/leaf only |
| `gen_ai.operation.name` | `chat` / `embeddings` / `image_generation` / `execute_tool` | Tool spans use `execute_tool` (per OTel GenAI semconv) |
| `gen_ai.request.temperature` | `options.temperature` | child/leaf only |
| `gen_ai.request.max_tokens` | `options.maxOutputTokens ?? options.maxTokens` | v4 fallback — child/leaf only |
| `gen_ai.request.top_p` / `top_k` | `options.topP` / `topK` | child/leaf only |
| `gen_ai.request.frequency_penalty` / `presence_penalty` | passed through | child/leaf only |
| `gen_ai.request.seed` | `options.seed` | child/leaf only |
| `gen_ai.request.stop_sequences` | `options.stopSequences` | JSON array — child/leaf only |
| `vercel_ai.tool_choice` | `options.toolChoice` | `auto` / `none` / `required` / `{type: 'tool', toolName}` |
| `vercel_ai.schema_name` | `options.schemaName` | generateObject only |
| `vercel_ai.provider_options` | `options.providerOptions ?? options.providerMetadata` | JSON, truncated at 8KB with warn log |
| `vercel_ai.max_retries` | `options.maxRetries` | |
| `vercel_ai.timeout_ms` | `options.timeout` | |
| `AIOPS_INTERNAL.observation.input` | structured JSON | prompt, messages, system, tools, instructions; truncated if >50KB |

### Response Attributes

| Attribute | Source | Notes |
| --- | --- | --- |
| `gen_ai.usage.input_tokens` | `usage.inputTokens ?? usage.promptTokens` | child/leaf only |
| `gen_ai.usage.output_tokens` | `usage.outputTokens ?? usage.completionTokens` | child/leaf only |
| `gen_ai.usage.total_tokens` | `usage.totalTokens` or sum | child/leaf only. All 3 coerced to finite numbers (see [[#D8. Number Coercion on Token Attributes]]) |
| `vercel_ai.usage.reasoning_tokens` | `usage.outputTokenDetails.reasoningTokens` | child/leaf only |
| `vercel_ai.usage.cached_input_tokens` | `usage.inputTokenDetails.cachedTokens` / `cacheReadTokens` | both field names supported — child/leaf only |
| `vercel_ai.usage.cache_write_tokens` | `usage.inputTokenDetails.cacheCreationTokens` / `cacheWriteTokens` | both field names supported — child/leaf only |
| `vercel_ai.usage.no_cache_tokens` | `usage.inputTokenDetails.noCacheTokens` | [[OpenAI]]-specific — child/leaf only |
| `vercel_ai.usage.text_tokens` | `usage.outputTokenDetails.textTokens` | [[OpenAI]]-specific — child/leaf only |
| `gen_ai.response.finish_reasons` | normalized string from `result.finishReason` | v6 returns `{unified, raw}` object — see [[#D7. `finishReason` Normalization]] — child/leaf only |
| `gen_ai.response.id` | `result.response.id` | child/leaf only |
| `gen_ai.response.model` | `result.response.modelId` | may differ from request model — child/leaf only |
| `vercel_ai.steps_count` | `result.steps.length` | multi-step agent runs — parent span |
| `vercel_ai.time_to_first_token_ms` | measured | streams only — parent span |
| `vercel_ai.gateway.cost` / `gateway.market_cost` | `result.providerMetadata.gateway.*` | when using [[Vercel AI Gateway]] |
| `vercel_ai.response.provider_metadata` | full `providerMetadata` blob | JSON, truncated at 8KB |
| `AIOPS_INTERNAL.observation.output` | structured JSON | text, reasoning, toolCalls, stepsCount, finishReason |

### Tool Span Attributes

| Attribute | Source |
| --- | --- |
| `gen_ai.operation.name` | `execute_tool` (per OTel GenAI semconv) |
| `vercel_ai.tool.name` | tool name from definition |
| `AIOPS_INTERNAL.observation.input` | tool args (JSON) |
| `AIOPS_INTERNAL.observation.output` | tool result (JSON) |

> [!note]
> Tool spans carry **only** `gen_ai.operation.name` from the `gen_ai.*` namespace. No model/usage/request attributes — tools don't call the LLM themselves. The `execute_tool` value keeps the span out of the GENERATION bucket (backend treats only `gen_ai.system` + usage as generation-typing signals in practice).

### `doGenerate` / `doStream` Span Attributes

Same shape as a parent LLM call would have carried pre-split: `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.*`, `gen_ai.response.*`, `OBSERVATION_INPUT`, `OBSERVATION_OUTPUT`. This is where cost is attributed. Allows per-LLM-call drilldown inside an agent loop.

---

## 4. Architectural Decisions

### D1. Import Wrapping Over Prototype Patching

Three approaches were considered:

| Approach | Provider-agnostic | Full SDK data | ESM + CJS | Zero-config |
| --- | --- | --- | --- | --- |
| CJS require cache patching | ✅ | ✅ | CJS only | ✅ |
| Provider prototype patching (`doGenerate`/`doStream`) | ❌ (hardcoded list) | ❌ (lossy) | ✅ | ✅ |
| **Import wrapping (chosen)** | ✅ | ✅ | ✅ | One line |

> [!tip] Why import wrapping won
> - Captures full SDK-level data (prompt, messages, tools, totalUsage, steps_count) — prototype patching loses these
> - Provider-agnostic — no hardcoded provider list
> - Works for both CJS and ESM out of the box
> - Trade-off: user adds one line `Observe.wrapVercelAI(ai)` instead of zero. Same pattern as [[Braintrust]]'s `wrapAISDK(ai)`.

> [!warning] Lesson learned
> The original plan proposed zero-config auto-tracing by patching ESM standalone function exports. ESM bindings are immutable live references — they cannot be replaced from outside the module. This was the single biggest architectural miss in the project. See `references/sdk-instrumentation-lessons.md` §1a.

### D2. Per-LLM-Call `doGenerate` / `doStream` Spans (added later)

When agents loop through multiple tool calls, the user wants to see each LLM round-trip individually — not just an aggregated agent span. [[Braintrust]] does this; we initially didn't.

**Implementation:** `wrapLanguageModel(tracer, model)` patches `model.doGenerate` and `model.doStream` in place using a `Symbol.for('vercel-ai.language-model.wrapped')` marker for idempotency. Detection: model must have BOTH `doGenerate` AND `doStream` methods (rules out EmbeddingModel, ImageModel, RerankingModel automatically).

**Trade-off:** plain (non-agent) `generateText` now produces a parent + one redundant `doGenerate` child. Acceptable for consistency — matches [[Braintrust]]'s behavior.

### D3. Don't Mutate User Tool Objects (root fix)

> [!danger] The problem
> Initial implementation mutated `tool.execute` in place on the agent instance, so the agent's internal `generateText` call would emit tool spans. **Side effect:** if the same tool was reused later in a plain `generateText`, the existing wrap would compose with our new wrap → nested duplicate spans.

**Root fix:** in `createAgentClassWrapper`'s construct trap, build a NEW tools object with wrapped clones, then replace `instance.settings.tools` (the agent's internal storage). The user's original tool objects stay untouched.

Verified by `test-mutation-check.cjs`:

```js
const originalRef = weatherTool.execute;
new ToolLoopAgent({ tools: { weatherTool } });
weatherTool.execute === originalRef  // ✅ true after fix
```

### D4. No Per-Step LLM Spans via `onStepFinish`

Considered injecting `onStepFinish` to emit a span per agent step. Rejected because:

- [[Braintrust]] and [[Google ADK]] don't do it
- The information lives on the parent agent span (`steps_count`, aggregated `usage`) plus the per-`doGenerate` spans
- Adds visual clutter for limited extra value

### D5. `wrapLanguageModel` Skips Embedding/Image/Rerank Models

These have a single API call per user call, not a loop. Their top-level span (`vercel-ai.embed`, `vercel-ai.generateImage`, `vercel-ai.rerank`) already captures everything. A `doGenerate`-style child would be 100% redundant. Plus their model interfaces don't have `doStream` so detection naturally skips them.

### D6. Stream Wrapper Handles `onError` / `onAbort`

> [!warning]
> If a stream errors mid-flight (rate limit, network drop) or is aborted, `onFinish` never fires → span would never end → memory leak + the trace shows a hanging span forever.

**Fix:** patch `onError` and `onAbort` callbacks alongside `onFinish` / `onChunk`. Use a `spanEnded` flag so the span ends exactly once across all callbacks.

### D7. `finishReason` Normalization

In v6, `LanguageModelV2.doGenerate()` and `onFinish` events return `finishReason` as `{ unified: 'stop', raw: 'stop' }` instead of a string. Setting an object as a span attribute renders as `"[object Object]"` in the UI.

**Fix:** central `extractFinishReason()` helper returns `fr.unified ?? fr.raw ?? fr` if the input is a string, otherwise normalizes to a string. Used everywhere we set `gen_ai.response.finish_reasons`.

### D8. Number Coercion on Token Attributes

Some SDK versions/providers occasionally return object-shaped values (e.g., `{}`) for token fields. Doing `inputTokens + outputTokens` on objects produces the JS classic `"[object Object][object Object]"` and gets stored as a span attribute string.

**Fix:** coerce to finite numbers everywhere via `asNumber(v) = typeof v === 'number' && Number.isFinite(v) ? v : undefined`.

### D9. Image Mask Binary Protection

`generateImage` accepts a prompt of shape `{ text, mask? }`. Originally the wrapper had `input.prompt = options.prompt.text ?? options.prompt` — fallback would serialize the entire object including the binary mask. Removed the fallback; if `prompt.text` isn't a string, we capture only `hasMask: true` and omit the body.

### D10. Embed Value Truncation

`embed` and `embedMany` inputs were previously serialized untruncated. Added `truncateEmbedValue` and `truncateEmbedValues` helpers (mirror `truncateMessages`'s 50KB threshold). Bounded in practice by the model's input limit anyway, but defensive against future model versions or unusual inputs.

### D11. Provider Options Truncation Warning

Provider options are JSON-stringified, truncated at 8KB. Now logs `logger.warn` when truncation happens so debugging stale `providerOptions` isn't silent.

### D12. Parent Spans as SPAN, Not GENERATION

> [!danger] The problem
> Initially, every wrapper span carried `gen_ai.*` attributes (system, model, usage, finish_reasons). The trace UI showed each of them as **GENERATION** — including the agent top-level span. Two consequences:
> 1. **Cost double-counting** — both the parent `generateText` and its `doGenerate` child reported `gen_ai.usage.input_tokens` / `output_tokens`. The cost engine multiplied each by model price, so a single LLM call cost was reported as 2×.
> 2. **Confusing trace UI** — `vercel-ai.agent.generate` appeared as a GENERATION even though it wraps an entire multi-step agent loop, making it indistinguishable from the individual LLM calls inside it.

**Root cause:** the backend classifies any span carrying `gen_ai.*` attributes as GENERATION, **regardless of `SpanKind`**. Changing `SpanKind` alone from `CLIENT` to `INTERNAL` was not enough.

**Fix:** strip all `gen_ai.*` attributes from parent wrapper spans (`generateText`, `streamText`, `generateObject`, `streamObject`, `agent.generate`, `agent.stream`). Keep them **only** on `doGenerate` / `doStream` children. Parent spans carry `vercel_ai.*` (tool_choice, provider_options, steps_count, TTFT, gateway cost) and `AIOPS_INTERNAL.*` (input/output JSON).

**Exception:** `embed`, `embedMany`, `generateImage`, `rerank` keep `gen_ai.*` on the parent because they have no `doGenerate` child to carry it — stripping would lose LLM metadata entirely.

**Result:** the trace UI now shows agent runs as a SPAN hierarchy with GENERATION leaves, and cost is attributed exactly once per LLM call at the `doGenerate` / `doStream` child.

### D13. Tool Span Operation Name: `execute_tool`

Tool child spans set `gen_ai.operation.name: 'execute_tool'` rather than a bespoke `'tool'` value. This aligns with the OpenTelemetry GenAI semantic conventions for tool execution spans and keeps the span out of the GENERATION bucket (tool spans don't have `gen_ai.system` / `gen_ai.usage.*` — the only `gen_ai.*` attribute on them is `operation.name`, which is not a generation-typing signal).

### D14. `experimental_telemetry` Suppression

> [!warning] The problem
> The [[Vercel AI SDK]] ships its own built-in OTel telemetry, enabled by passing `experimental_telemetry: { isEnabled: true }` to any call. If a user (a) enables this AND (b) uses `Observe.wrapVercelAI(ai)`, **both paths emit spans for the same call** — duplicate trace tree, duplicate cost, confusing parent-child relationships.

**Fix:** the wrapper forcibly injects `experimental_telemetry: { isEnabled: false }` into the options of every wrapped call (`generateText`, `streamText`, `generateObject`, `streamObject`, and the inner `doGenerate` / `doStream` calls inside agents). This disables the SDK's internal telemetry path so the only span source is ours.

**Trade-off:** a user who deliberately sets `experimental_telemetry: { isEnabled: true }` will have it silently flipped off. Documented behavior — users who want the SDK's telemetry path should not wrap with `Observe.wrapVercelAI`, or use `experimental_telemetry.recordInputs` / `recordOutputs` flags on the wrapper side instead.

---

## 5. v4 Compatibility

> [!info]
> v4 has a different API surface. We support it via fallback field reads — no separate code path.

| Area | v4 | v5+ | Wrapper handles via |
| --- | --- | --- | --- |
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

## 6. Known Issues & Limitations

### Issue #9 — AI SDK v6 Tool Property Name Mismatch (UPSTREAM)

> [!bug] Status: [vercel/ai#12020](https://github.com/vercel/ai/issues/12020) — OPEN
> **Affected:** `ai@6.x` + any `@ai-sdk/*` provider.
> **Affects all schema types** (`jsonSchema()`, `z.object()` Zod v3 and v4) — NOT a Zod-specific issue.

**Errors:**

- [[Anthropic]]: `tools.0.custom.input_schema.type: Field required`
- [[OpenAI]]: `Invalid schema for function: schema must be a JSON Schema of 'type: "object"', got 'type: "None"'`

**Root cause:** The `tool()` helper is an identity function. AI SDK docs and examples define tools with `parameters`, but `prepareToolsAndToolChoice()` reads `tool.inputSchema`. Since the tool object has `parameters` but NOT `inputSchema`, `tool.inputSchema` is `undefined` and `asSchema(undefined)` returns `{"properties":{},"additionalProperties":false}` — missing `type`, properties contents, and `required`. Provider rejects.

**Workaround:** Use `inputSchema` instead of `parameters`:

```js
tool({
  description: 'Get weather',
  inputSchema: jsonSchema({
    type: 'object',
    properties: { city: { type: 'string' } },
    required: ['city']
  }),
  execute: async ({ city }) => ({ city, temp: 22 }),
})
```

**Affected methods:** `generateText` / `streamText` / `generateObject` (with tools), `agent.generate` / `agent.stream`.
**Not affected:** all calls without `tools`, plus `embed` / `embedMany` / `generateImage` / `rerank`.

> [!note]
> **Impact on us:** None — we pass tools through unchanged. The mismatch is in the AI SDK core.

### Limitation: `generateText` Doesn't Loop in v6

Plain `generateText({ tools, maxSteps: 10 })` makes one LLM call, executes whatever tools it asked for, then stops. It does NOT loop back to synthesize a final answer in v6. Use `ToolLoopAgent` for that.

### Limitation: Cost Shows $0 for Embeddings (Google) and Image Generation

- **Google embedding** returns `usage.tokens: null`. Our wrapper correctly omits the field. The cost engine has no token count to multiply by price → $0.
- **DALL-E** returns `usage.imagesGenerated` only. Cost is per-image, not per-token. Backend cost engine is token-based → $0.

> [!note]
> Both are observability-pipeline gaps in the backend, not bugs in our SDK. The data needed to compute cost (image count, size, model) IS captured on the span.

### Limitation: Sub-Agent Spans Display as `Object.execute` in the UI

Span name internally is `vercel-ai.agent.generate`. The trace UI displays it by `code.function` attribute (which is `Object.execute` because the calling tool's `execute: async () => {...}` is on an object literal). UI rendering preference, not a span data issue.

### Limitation: First Child Indentation Quirk in Trace Viewer

Verified end-to-end via REST API: all sibling spans correctly share the same `parentObservationId`. The visual indentation difference is a renderer quirk in the trace viewer, not a real parent-child relationship issue.

### Limitation: Tool Arguments Not Scanned for Secrets

Tool `execute` arguments are captured verbatim via `JSON.stringify(toolArgs)`. If a user passes API keys, tokens, or auth headers as tool arguments, they will appear in the span. A lightweight redaction pass (scanning for `api_key`, `token`, `secret`, `password`, `bearer` patterns) would improve safety. Tracked for a future enhancement.

### Limitation: `doStream` TransformStream Span Can Leak on Abandoned Streams

The `doStream` wrapper relies on `flush()` as a safety net to end the span. If the consumer abandons the stream without consuming or erroring, `flush()` may never fire. The higher-level `onError`/`onAbort` callbacks mitigate at the top-level span, but the inner `doStream` span has no timeout protection. A configurable timeout (e.g., 5 minutes) would resolve this.

### Resolved Issues Found During Testing

Historical record of issues caught and resolved during development. Issue #9 (above) is the only one still active (upstream).

| # | Issue | Status |
| --- | --- | --- |
| 1 | `json: undefined` in messages | Frontend display issue (not us) |
| 2 | `cached_input_tokens` always 0 | Correct provider value when no caching |
| 3 | `stopSequences` stopped early | Test working correctly |
| 4 | Tool schema raw Zod internals | ✅ **Fixed** — `safeSerializeSchema()` |
| 5 | Missing schema in generateObject input | ✅ **Fixed** — captures `options.schema` |
| 6 | Embed `tokens: null` | ✅ **Fixed** — `!= null` check |
| 7 | generateImage missing cost | 🔧 **Improved** — captures full `providerMetadata` |
| 8 | [[OpenAI]] `no_cache_tokens` extra fields | Correct provider value |
| 9 | Tool calls fail | ⬆️ **Upstream bug** (above) |
| 10 | Parent spans showed as GENERATION (cost double-counted) | ✅ **Fixed** — see [[#D12. Parent Spans as SPAN, Not GENERATION]] |
| 11 | Tool spans mis-typed | ✅ **Fixed** — `execute_tool` per OTel GenAI semconv ([[#D13. Tool Span Operation Name: `execute_tool`]]) |
| 12 | Duplicate spans from `experimental_telemetry` | ✅ **Fixed** — automatic suppression ([[#D14. `experimental_telemetry` Suppression]]) |

---

## 7. What We Do NOT Trace

| Excluded | Why |
| --- | --- |
| `experimental_generateSpeech` | Still experimental. Add when stable. |
| `experimental_transcribe` | Still experimental. Add when stable. |
| Provider HTTP calls | Already traced by separate [[OpenAI]] / [[Anthropic]] / [[Bedrock]] instrumentations. Tracing both layers would create duplicate spans. |
| Per-step LLM spans (via `onStepFinish`) | Already covered by per-`doGenerate` spans. Adding step-level spans would duplicate. |
| `headers` / `abortSignal` | Request infrastructure, not observability-relevant. |
| Embedding / image / rerank `doGenerate` spans | Single-call operations — top-level span already captures everything. |
| `providerOptions` individual fields | Captured as a single JSON blob (truncated at 8KB), not exploded into individual attributes. Provider-specific, deeply nested, and change frequently. |

---

## 8. Competitive Position

| Capability | Us | [[Braintrust]] | [[Langfuse]] | [[Arize]] |
| --- | --- | --- | --- | --- |
| Setup | `Observe.wrapVercelAI(ai)` | `wrapAISDK(ai)` | `experimental_telemetry` per call | Same as [[Langfuse]] |
| ESM + CJS | ✅ | ✅ | ✅ | ✅ |
| `embed` / `embedMany` | ✅ | ✅ | ❌ | ❌ |
| `generateImage` | ✅ | ❌ (verified 2026-04-08, `ai-sdk.ts`) | ❌ | ❌ |
| `rerank` | ✅ | ❌ (verified 2026-04-08) | ❌ | ❌ |
| Agent tracing | ✅ | ✅ | ❌ | ❌ |
| Per-tool child spans | ✅ | ✅ | ❌ | ❌ |
| Per-LLM-call (`doGenerate`) spans | ✅ | ✅ | ❌ | ❌ |
| `totalUsage` (multi-step) | ✅ | ✅ | ❌ | ❌ |
| `steps_count` | ✅ | ✅ | ❌ | ❌ |
| TTFT (time to first token) | ✅ | ✅ | ❌ | ❌ |
| OTel-native spans | ✅ | ❌ (proprietary) | ✅ | ✅ |

> [!warning] Verified claims
> All "No" entries for competitors were verified against source or docs on 2026-04-08. [[Braintrust]] `embed`/`embedMany` support was initially claimed as absent — this was **wrong** (they have `wrapEmbed`/`wrapEmbedMany`). Corrected above. See `references/sdk-instrumentation-lessons.md` §11a for the full competitor-claim lesson.

---

## 9. Files

| File | Role |
| --- | --- |
| `src/instrumentations/vercel-ai/wrap-vercel-ai.ts` | The wrapper (single file, ~1600 LOC) |
| `src/client.ts` | `Observe.wrapVercelAI` public API |
| `src/index.ts` | Re-export `wrapVercelAI` |
| `tests/featureTest/auto-tracing/wrap-vercel-ai.test.ts` | 60+ unit tests |
| `tests/llm-providers/auto-tracing-vercel-ai-agent.cjs` | Live integration test (runs against real [[OpenAI]] + local backend) |
| `docs/plans/vercel-ai-final.md` | This document |

---

## 10. Demo Routes (in `demo-application/nodejs`)

The demo app exposes 20 routes for the `vercel_ai` provider. Recommended demo subset (covers every unique span shape):

| Route | What it shows |
| --- | --- |
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
| `POST /agent-run` | `ToolLoopAgent.generate` — alternating doGenerate/tool/doGenerate |
| `POST /agent-stream` | `ToolLoopAgent.stream` |
| `POST /multi-agent` | Coordinator + 2 sub-agents (nested via tool bodies) |
| `POST /nested-agents` | Main agent → tool → sub-agent → tool (deepest nesting) |
| `POST /image-generation` | `generateImage` (DALL-E 3) |
| `POST /telemetry-override` | Verifies `experimental_telemetry: { isEnabled: true }` is suppressed — see [[#D14. `experimental_telemetry` Suppression]] |

---

## 11. Recent Fixes Timeline (PR #961)

In rough order of when they shipped during the PR:

1. **Property name mismatch documented** — Issue #9 raised against upstream; user-facing workaround documented.
2. **`createAgentStreamWrapper` Promise handling** — `ToolLoopAgent.stream()` returns `PromiseLike` in v6; wrapper now awaits.
3. **Per-tool child spans on agents** — added `wrapToolExecutions`-equivalent at the agent construct trap.
4. **AGENT_TOOL_WRAPPED idempotency marker** — interim fix for nested duplicate tool spans.
5. **Root mutation fix** — replaced the marker fix with proper non-mutating clone of `instance.settings.tools`. User tool objects are no longer modified. (See [[#D3. Don't Mutate User Tool Objects (root fix)]])
6. **`wrapLanguageModel` (`doGenerate` / `doStream`)** — per-LLM-call child spans, matches [[Braintrust]] shape. (See [[#D2. Per-LLM-Call `doGenerate` / `doStream` Spans (added later)]])
7. **`finishReason` normalization** — central `extractFinishReason()` handles v6's `{unified, raw}` object shape. (See [[#D7. `finishReason` Normalization]])
8. **Token coercion** — `asNumber` everywhere; eliminates `[object Object][object Object]` from token attrs. (See [[#D8. Number Coercion on Token Attributes]])
9. **`onError` / `onAbort` for streams** — span no longer leaks when stream errors or is aborted before `onFinish`. (See [[#D6. Stream Wrapper Handles `onError` / `onAbort`]])
10. **Image mask binary leak** — fallback removed; only `hasMask: true` is captured if `prompt.text` is missing. (See [[#D9. Image Mask Binary Protection]])
11. **Embed value truncation** — bounded by `MAX_INPUT_MESSAGES_BYTES` (50 KB). (See [[#D10. Embed Value Truncation]])
12. **Provider options truncation warning** — `logger.warn` emitted when serialized options exceed 8KB. (See [[#D11. Provider Options Truncation Warning]])
13. **Parent spans reclassified to SPAN** — removed all `gen_ai.*` attrs from wrapper spans (`generateText`, `streamText`, `generateObject`, `streamObject`, `agent.generate`, `agent.stream`). Only `doGenerate` / `doStream` children retain `gen_ai.*` and therefore cost. Fixes double-counted cost and the confusing "agent = GENERATION" display. (See [[#D12. Parent Spans as SPAN, Not GENERATION]])
14. **Tool spans use `gen_ai.operation.name: 'execute_tool'`** — aligns with OTel GenAI semconv, keeps tool spans out of the GENERATION bucket. (See [[#D13. Tool Span Operation Name: `execute_tool`]])
15. **`experimental_telemetry` suppression** — wrapper injects `experimental_telemetry: { isEnabled: false }` into every wrapped call, preventing duplicate spans from the SDK's built-in telemetry path. (See [[#D14. `experimental_telemetry` Suppression]])

All changes live in `src/instrumentations/vercel-ai/wrap-vercel-ai.ts`. Net diff for PR: ~1600 LOC added (wrapper) + the supporting test files.

---

## 12. Lessons Contributed to SDK Review Skill

> [!abstract]
> This instrumentation's post-mortem produced lessons that are now encoded in the team's `/sdk-review-instrumentation` skill rubric. Key contributions:

| Lesson | Rubric rule | Category |
| --- | --- | --- |
| ESM standalone function exports cannot be patched | MS-1 (Architectural Blocker) | module-system |
| Competitor claims without source links are wrong by default | CC-1, CC-3 (Blocker) | competitor-claims |
| Version matrix with wrong npm→SDK mapping | VM-1 (Warning) | version-matrix |
| `!== undefined` drops `null` token values | RS-2 (Blocker) | runtime-shape |
| `agent.stream()` returns Promise, not sync result | RS-1, AF-8 (Blocker) | runtime-shape |
| Zod schema internals leak via `JSON.stringify` | CS-3 (Blocker) | content-safety |
| `onFinish` callback chaining must isolate user errors | RS-5 (Warning) | runtime-shape |
| `finishReason` may be object, not string | RS-6 (Warning) | runtime-shape |
| `experimental_telemetry` dedup with our wrapping | AF-7 (Warning) | agent-framework |
| Tool object mutation causes nested duplicates | AF-5 (Blocker) | agent-framework |
| Backend types spans by `gen_ai.*` presence, not SpanKind | ST-1 (Blocker) | span-typing |

See `.claude/skills/sdk:review-instrumentation/references/sdk-instrumentation-lessons.md` for the full catalogue.

---

## 13. References

- Upstream bug: [vercel/ai#12020](https://github.com/vercel/ai/issues/12020)
- [[Vercel AI SDK]] docs: https://ai-sdk.dev
- v5 migration guide: https://ai-sdk.dev/docs/migration-guides/migration-guide-5-0
- v6 migration guide: https://ai-sdk.dev/docs/migration-guides/migration-guide-6-0
- [[OpenTelemetry]] GenAI semantic conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/
- [[Braintrust]] `wrapAISDK` source (reference for `doGenerate` span shape): `github.com/braintrustdata/braintrust-sdk/js/src/wrappers/ai-sdk/ai-sdk.ts`
- SDK instrumentation review skill: `.claude/skills/sdk:review-instrumentation/`
- Lessons catalogue: `.claude/skills/sdk:review-instrumentation/references/sdk-instrumentation-lessons.md`
- Demo app: `demo-application/nodejs/providers/vercel_ai.js`

---

## Related Notes

- [[Pinecone Auto-Tracing Instrumentation — Specification]]
- [[Auto-Tracing Architecture]]
- [[OpenTelemetry Instrumentation Patterns]]
- [[RAG Pipeline Observability]]
- [[Vercel AI SDK v7 — registerTelemetryIntegration Plan]]
- [[SDK Instrumentation Lessons]]
