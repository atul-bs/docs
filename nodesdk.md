# @browserstack/ai-sdk — Node.js API Reference

## Importable Symbols

```js
import { AISDK, Observe, wrapVercelAI, Tool, Prompt, Evaluate, Media } from '@browserstack/ai-sdk';
```

---

## AISDK

Main client. Entry point for all SDK functionality.

```js
const client = new AISDK({ publicKey, secretKey, environment? });
```

### Core Tracing

| Method | Params | Returns | Description |
|--------|--------|---------|-------------|
| `trace(body)` | `{ name?, userId?, sessionId?, tags?, metadata?, version?, environment? }` | `TraceWrapper` | Create a trace |
| `generation(body)` | `{ name?, model?, input?, usage?, modelParameters?, ... }` | `GenerationWrapper` | Create a top-level generation |
| `span(body)` | `{ name?, input?, output?, metadata?, level?, ... }` | `SpanWrapper` | Create a top-level span |
| `event(body)` | `{ name?, input?, output?, metadata?, level?, ... }` | `EventWrapper` | Create a top-level event |
| `score(body)` | `{ name, value, traceId?, comment?, dataType?, ... }` | `void` | Send a score |

### Lifecycle

| Method | Returns | Description |
|--------|---------|-------------|
| `flush()` | `Promise<void>` | Flush pending data |
| `flushAsync()` | `Promise<void>` | Flush pending data (async) |
| `shutdown()` | `Promise<void>` | Shutdown and flush |
| `shutdownAsync()` | `Promise<void>` | Shutdown and flush (async) |

### Sub-clients (properties)

| Property | Type | Description |
|----------|------|-------------|
| `client.datasets` | `DatasetsClient` | Dataset CRUD |
| `client.datasetRuns` | `DatasetRuns` | Dataset run operations |
| `client.experiments` | `Experiments` | Experiment CRUD |
| `client.experimentRuns` | `ExperimentRuns` | Experiment run operations |
| `client.evaluators` | `Evaluators` | Evaluator CRUD |
| `client.evalsList` | `EvaluatorLists` | Evaluator list CRUD |
| `client.evals` | `EvaluationExecution` | Run evaluations |
| `client.tools` | `Tool` | Tool CRUD |
| `client.prompt` | `Prompt` | Prompt get/create/update |
| `client.evaluate` | `Evaluate` | Namespaced evaluate API |

### Static

| Member | Description |
|--------|-------------|
| `AISDK.observe.init(config?)` | Initialize auto-tracing |

---

## TraceWrapper

Returned by `client.trace()`. Groups related observations.

| Member | Type | Description |
|--------|------|-------------|
| `.id` | `string` | Trace ID |
| `.traceId` | `string` | Same as id |
| `.update(body)` | `→ TraceWrapper` | Update trace metadata/output |
| `.span(body)` | `→ SpanWrapper` | Create child span |
| `.generation(body)` | `→ GenerationWrapper` | Create child generation |
| `.event(body)` | `→ EventWrapper` | Create child event |
| `.score(body)` | `→ string` | Score this trace |
| `.getTraceUrl()` | `→ string` | URL to view in UI |

---

## SpanWrapper

Returned by `trace.span()` or `span.span()`. Tracks a unit of work.

| Member | Type | Description |
|--------|------|-------------|
| `.id` | `string` | Span ID |
| `.traceId` | `string` | Parent trace ID |
| `.parentId` | `string?` | Parent observation ID |
| `.update(body)` | `→ SpanWrapper` | Update metadata |
| `.end(body?)` | `→ SpanWrapper` | End the span with output |
| `.span(body)` | `→ SpanWrapper` | Create nested span |
| `.generation(body)` | `→ GenerationWrapper` | Create child generation |
| `.event(body)` | `→ EventWrapper` | Create child event |
| `.score(body)` | `→ string` | Score this span |
| `.getTraceUrl()` | `→ string` | URL to view in UI |

---

## GenerationWrapper

Returned by `trace.generation()` or `span.generation()`. Tracks an LLM call.

| Member | Type | Description |
|--------|------|-------------|
| `.id` | `string` | Generation ID |
| `.traceId` | `string` | Parent trace ID |
| `.parentId` | `string?` | Parent observation ID |
| `.update(body)` | `→ GenerationWrapper` | Update metadata |
| `.end(body?)` | `→ GenerationWrapper` | End with output/usage |
| `.span(body)` | `→ SpanWrapper` | Create child span |
| `.generation(body)` | `→ GenerationWrapper` | Create nested generation |
| `.event(body)` | `→ EventWrapper` | Create child event |
| `.score(body)` | `→ string` | Score this generation |
| `.getTraceUrl()` | `→ string` | URL to view in UI |

---

## EventWrapper

Returned by `trace.event()`. Point-in-time observation.

| Member | Type | Description |
|--------|------|-------------|
| `.id` | `string` | Event ID |
| `.traceId` | `string` | Parent trace ID |
| `.parentId` | `string?` | Parent observation ID |
| `.span(body)` | `→ SpanWrapper` | Create child span |
| `.generation(body)` | `→ GenerationWrapper` | Create child generation |
| `.event(body)` | `→ EventWrapper` | Create child event |
| `.score(body)` | `→ string` | Score this event |
| `.getTraceUrl()` | `→ string` | URL to view in UI |

---

## DatasetsClient

Access via `client.datasets`.

| Method | Params | Description |
|--------|--------|-------------|
| `.create(name, description?, metadata?)` | `string, string?, Record?` | Create dataset |
| `.createItems(params, options?)` | `CreateItemsRequest, CreateItemsOptions?` | Add items to dataset |
| `.getTags(datasetName)` | `string` | Get dataset tags |
| `.list(page?, limit?, name?)` | `number?, number?, string?` | List datasets |

---

## DatasetRuns

Access via `client.datasetRuns`.

| Method | Params | Description |
|--------|--------|-------------|
| `.create(datasetName, options?)` | `string, { name?, description?, metadata?, tag? }` | Create mutable run |
| `.create(params)` | `{ name, promptName, promptVersion?, description?, metadata? }` | Create immutable run with prompt |
| `.createItems(datasetName, datasetRunId, items)` | `string, string, CreateDatasetRunItemRequest[]` | Add items to run |
| `.getRunByTag(datasetName, tagName)` | `string, string` | Get run by tag |
| `.getRunTags(datasetName, datasetRunId, page?, limit?)` | `string, string, number?, number?` | Get run tags |
| `.list(datasetName, page?, limit?, name?, type?)` | `string, number?, number?, string?, string?` | List runs |
| `.listItems(datasetName, datasetRunId, page?, limit?)` | `string, string, number?, number?` | List run items |

---

## Experiments

Access via `client.experiments`.

| Method | Params | Description |
|--------|--------|-------------|
| `.create(data)` | `{ name, datasetRunTagId, evaluatorListId, description?, concurrency? }` | Create with tag |
| `.create(data)` | `{ name, promptId, datasetId, evaluatorListId, description?, concurrency? }` | Create with prompt |
| `.delete(experimentId)` | `string` | Delete experiment |
| `.find(experimentId)` | `string` | Find by ID |
| `.list(limit?, page?)` | `number?, number?` | List experiments |

---

## ExperimentRuns

Access via `client.experimentRuns`.

| Method | Params | Description |
|--------|--------|-------------|
| `.create(experimentId, llmColumnName?, concurrency?, runConfig?)` | `string, string?, number?, Record?` | Create run |
| `.delete(experimentRunId)` | `string` | Delete run |
| `.find(experimentRunId)` | `string` | Find by ID |
| `.list(experimentId?, page?, limit?)` | `string?, number?, number?` | List runs |
| `.subscribe(experimentRunId, timeout?, pollInterval?)` | `string, number?, number?` | Poll until complete |
| `.update(experimentRunId, updateData)` | `string, UpdateExperimentRunRequest` | Update run |

---

## Evaluators

Access via `client.evaluators`.

| Method | Params | Description |
|--------|--------|-------------|
| `.create(options)` | `CreateEvaluatorOptions` | Create evaluator |
| `.get(evaluatorId)` | `string` | Get by ID |
| `.list(options?)` | `ListEvaluatorsOptions?` | List evaluators |
| `.listVersions(evaluatorId, options)` | `string, ListVersionsOptions` | List versions |
| `.setLabels(evaluatorId, labels)` | `string, string[]` | Set labels |
| `.update(evaluatorId, options)` | `string, UpdateEvaluatorOptions` | Update evaluator |

---

## EvaluatorLists

Access via `client.evalsList`.

| Method | Params | Description |
|--------|--------|-------------|
| `.create(requestData)` | `CreateEvaluatorListRequest` | Create list |
| `.delete(evaluatorListId)` | `string` | Delete list |
| `.get(evaluatorListId)` | `string` | Get by ID |
| `.list(limit?, page?, orderBy?)` | `number?, number?, { column?, order? }` | List all |

---

## EvaluationExecution

Access via `client.evals`.

| Method | Params | Description |
|--------|--------|-------------|
| `.evaluate(requestData)` | `{ evaluators: [{ metricName, params? }], data: { input?, output?, context?, expectedOutput? }, provider?, model?, concurrency? }` | Run evaluation |

---

## Prompt

Access via `client.prompt` or `new Prompt(publicKey, secretKey)`.

| Method | Params | Returns | Description |
|--------|--------|---------|-------------|
| `.get(name, version?, options?)` | `string, number?, { type?: 'text', label?, cacheTtlSeconds?, fallback?, fetchTimeoutMs? }` | `Promise<TextPromptClient>` | Get text prompt |
| `.get(name, version?, options?)` | `string, number?, { type: 'chat', label?, cacheTtlSeconds?, fallback?, fetchTimeoutMs? }` | `Promise<ChatPromptClient>` | Get chat prompt |
| `.create(body)` | `ExtendedCreateTextPromptBody \| ExtendedCreateChatPromptBody \| ...` | `Promise<PromptClient>` | Create prompt |
| `.update(body)` | `{ name, version, newLabels }` | `Promise<any>` | Update labels |

### TextPromptClient

Returned by `prompt.get()` with `type: 'text'`.

| Member | Type | Description |
|--------|------|-------------|
| `.name` | `string` | Prompt name |
| `.version` | `number` | Version number |
| `.prompt` | `string` | Raw prompt text |
| `.labels` | `string[]` | Labels |
| `.tags` | `string[]` | Tags |
| `.config` | `Record` | Config |
| `.type` | `"text"` | Always "text" |
| `.isFallback` | `boolean` | Using fallback? |
| `.commitMessage` | `string?` | Commit message |
| `.promptResponse` | `any` | Raw response |
| `.tools` | `ToolList` | Attached tools |
| `.compile(variables?)` | `→ string` | Compile with variables |
| `.getLangchainPrompt()` | `→ string` | Get LangChain format |

### ChatPromptClient

Returned by `prompt.get()` with `type: 'chat'`.

Same as TextPromptClient except:
- `.prompt` → `ChatMessage[]`
- `.compile(variables?)` → `ChatMessage[]`
- `.normalizePrompt` → `any`

### ToolList (on prompt.tools)

| Member | Type | Description |
|--------|------|-------------|
| `.length` | `number` | Number of tools |
| `.compile(options?)` | `→ CompiledTool[]` | Compile tools for provider |

---

## Tool

Access via `client.tools` or `new Tool(publicKey?, secretKey?)`.

| Method | Params | Returns | Description |
|--------|--------|---------|-------------|
| `.create(options)` | `{ name, description, parameters?, strict?, sampleOutput?, isRunnable?, labels?, commitMessage? }` | `Promise<ToolInstance>` | Create tool |
| `.get(name, provider?, options?)` | `string, string?, { version?, label?, cacheTtlSeconds? }` | `Promise<ProviderToolResult>` | Get tool |
| `.list(limit?, cursor?, label?)` | `number?, string?, string?` | `Promise<ListToolsResponse>` | List tools |
| `.update(options)` | `UpdateToolOptions` | `Promise<ToolInstance>` | Update tool |

### ToolInstance

Returned by `tool.create()` and `tool.update()`.

| Member | Type | Description |
|--------|------|-------------|
| `.id` | `string` | Tool ID |
| `.name` | `string` | Name |
| `.description` | `string?` | Description |
| `.parameters` | `ToolParameters` | JSON schema params |
| `.version` | `number` | Version |
| `.labels` | `string[]` | Labels |
| `.commitMessage` | `string?` | Commit message |
| `.createdAt` | `string` | Created timestamp |
| `.updatedAt` | `string` | Updated timestamp |
| `.isRunnable` | `boolean?` | Is runnable |
| `.sampleOutput` | `Record?` | Sample output |
| `.compile(options?)` | `→ CompiledTool` | Compile for provider |
| `.toData()` | `→ LlmTool` | Raw data |
| `.toString()` | `→ string` | String representation |

### ProviderToolResult

Returned by `tool.get()`.

| Method | Returns | Description |
|--------|---------|-------------|
| `.compile(options?)` | `CompiledTool` | Compile for provider |
| `.toData()` | `Record` | Raw data |

---

## Evaluate

Access via `client.evaluate` or `new Evaluate(publicKey, secretKey)`.

Namespaced API with 7 sub-objects:

### evaluate.dataset

| Method | Params |
|--------|--------|
| `.create(name, description?, metadata?)` | `string, string?, Record?` |
| `.list(page?, limit?, name?)` | `number?, number?, string?` |
| `.getTags(datasetName)` | `string` |
| `.createItems(params, options?)` | `CreateItemsRequest, CreateItemsOptions?` |

### evaluate.datasetRun

| Method | Params |
|--------|--------|
| `.create(datasetName, options?)` | `string, { name?, description?, metadata?, tag? }` |
| `.create(params)` | `{ name, promptName, promptVersion?, ... }` |
| `.list(datasetName, page?, limit?, name?, type?)` | `string, ...` |
| `.listItems(datasetName, datasetRunId, page?, limit?)` | `string, string, ...` |
| `.createItems(datasetName, datasetRunId, items)` | `string, string, array` |
| `.getRunByTag(datasetName, tagName)` | `string, string` |
| `.getRunTags(datasetName, datasetRunId, page?, limit?)` | `string, string, ...` |

### evaluate.evaluator

| Method | Params |
|--------|--------|
| `.create(options)` | `CreateEvaluatorOptions` |
| `.get(evaluatorId)` | `string` |
| `.list(options?)` | `ListEvaluatorsOptions?` |
| `.update(evaluatorId, options)` | `string, UpdateEvaluatorOptions` |
| `.listVersions(evaluatorId, options)` | `string, ListVersionsOptions` |
| `.setLabels(evaluatorId, labels)` | `string, string[]` |

### evaluate.evaluatorList

| Method | Params |
|--------|--------|
| `.create(requestData)` | `CreateEvaluatorListRequest` |
| `.list(limit?, page?, orderBy?)` | `number?, number?, { column?, order? }` |
| `.get(evaluatorListId)` | `string` |
| `.delete(evaluatorListId)` | `string` |

### evaluate.experiment

| Method | Params |
|--------|--------|
| `.create(data)` | `CreateExperimentWithTagRequest` or `CreateExperimentWithPromptRequest` |
| `.list(limit?, page?)` | `number?, number?` |
| `.find(experimentId)` | `string` |
| `.delete(experimentId)` | `string` |

### evaluate.experimentRun

| Method | Params |
|--------|--------|
| `.create(experimentId, llmColumnName?, concurrency?, runConfig?)` | `string, ...` |
| `.list(experimentId?, page?, limit?)` | `string?, ...` |
| `.find(experimentRunId)` | `string` |
| `.delete(experimentRunId)` | `string` |
| `.update(experimentRunId, updateData)` | `string, UpdateExperimentRunRequest` |
| `.subscribe(experimentRunId, timeout?, pollInterval?)` | `string, number?, number?` |

### evaluate.evaluationExecution

| Method | Params |
|--------|--------|
| `.evaluate(requestData)` | `EvaluationExecutionRequest` |

---

## Media

```js
const media = new Media({ contentType, base64DataUri?, contentBytes?, filePath?, unsignedUrl? });
```

| Member | Type | Description |
|--------|------|-------------|
| `.contentLength` | `number?` | Byte size |
| `.contentSha256Hash` | `string?` | SHA-256 hash |
| `.obj` | `object?` | Source object |
| `.unsignedUrl` | `string?` | Unsigned URL |
| `.toJSON()` | `→ string` | Serialize |
| `Media.parseReferenceString(str)` | `→ { mediaId, source, contentType }` | Static: parse ref |

---

## Observe

Auto-tracing and OpenTelemetry context.

| Member | Type | Description |
|--------|------|-------------|
| `.init(config?)` | `→ Promise<void>` | Initialize auto-tracing |
| `.setAttribute(key, value)` | `→ void` | Set trace attribute |
| `.TraceAttribute.SESSION_ID` | `"session.id"` | Session ID key |
| `.TraceAttribute.USER_ID` | `"user.id"` | User ID key |
| `.TraceAttribute.TRACE_INPUT` | `"AIOPS_INTERNAL.trace.input"` | Trace input key |
| `.TraceAttribute.TRACE_OUTPUT` | `"AIOPS_INTERNAL.trace.output"` | Trace output key |
| `.propagation.inject(ctx, carrier, setter?)` | `→ void` | Inject context for distributed tracing |
| `.propagation.extract(ctx, carrier, getter?)` | `→ Context` | Extract context from carrier |
| `.context.active()` | `→ Context` | Get active context |
| `.context.with(ctx, fn)` | `→ T` | Run fn in context |
| `.trace.setSpan(ctx, span)` | `→ Context` | Set span on context |
| `.defaultTextMapGetter` | `object` | Default getter for extract |
| `.defaultTextMapSetter` | `object` | Default setter for inject |

---

## wrapVercelAI

```js
const ai = wrapVercelAI(aiModule);
```

Wraps Vercel AI SDK module for auto-tracing. Returns the same module with instrumentation.

---

## Enums (not directly importable, used in params)

### ExperimentRunStatus
`PENDING` | `RUNNING` | `COMPLETED` | `FAILED` | `CANCELLED`

### EvaluatorParamDataType
`string` | `integer` | `float` | `boolean` | `string[]` | `integer[]` | `float[]` | `boolean[]` | `object`
