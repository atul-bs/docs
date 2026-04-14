# SDK Instrumentation Review Skill

Automated first-pass review for instrumentation code and plans across all SDK wrappers. Reviews by **pattern** (module-system, version matrix, content safety, span dedup, etc.) — not by specific provider or framework.

## How to use

```bash
/sdk-review-instrumentation          # Review current branch diff (default)
/sdk-review-instrumentation code     # Same as above
/sdk-review-instrumentation plan <path>  # Review a plan document before coding
```

Report is written to `.claude/sdk-review-report.md`.

---

## File tree

```
.claude/skills/sdk:review-instrumentation/
├── SKILL.md                              ← Entry point (dispatch logic, report format)
├── README.md                             ← You are here
└── references/
    ├── sdk-instrumentation-lessons.md    ← Lessons catalogue (human-readable, all SDKs)
    │
    ├── cross-cutting/                    ← Always loaded (every review)
    │   ├── module-system.md              ←   ESM/CJS patching, auto vs manual tracing
    │   ├── version-matrix.md             ←   Multi-version support, field renames
    │   ├── runtime-shape.md              ←   Sync/async, streams, null handling, idempotency
    │   ├── content-safety.md             ←   Binary, truncation, schemas, secrets
    │   ├── competitor-claims.md          ←   Evidence requirements for platform comparisons
    │   ├── otel-hygiene.md               ←   Span naming, attributes, AIOPS_INTERNAL prefix
    │   └── span-deduplication.md         ←   Cross-framework interference, skip logic
    │
    ├── lang-nodejs.md                    ← Loaded for testops-nodejs-wrapper/ changes
    ├── lang-python.md                    ← Loaded for testops-python-wrapper/ changes
    ├── lang-java.md                      ← Loaded for testops-java-wrapper/ changes
    ├── lang-claude-plugin.md             ← Loaded for testops-claude-plugin/ changes
    │
    ├── category-llm-provider.md          ← Loaded when target is an LLM provider
    ├── category-agent-framework.md       ← Loaded when target is an agent framework
    ├── category-vector-db.md             ← Loaded when target is a vector database
    ├── category-retrieval-rerank.md      ← Loaded when target is rerank / retrieval pipeline
    └── category-mcp-tooling.md           ← Loaded when target is MCP server/client
```

---

## How the two-axis rubric works

Every review loads **three layers** of rubric files:

```
┌─────────────────────────────────┐
│  1. Cross-cutting (always)      │  7 files — module system, versions, runtime,
│                                 │  content safety, competitors, OTel, dedup
├─────────────────────────────────┤
│  2. Language (by SDK path)      │  1 file — nodejs OR python OR java OR plugin
├─────────────────────────────────┤
│  3. Category (by target)        │  1+ files — llm-provider, agent-framework,
│                                 │  vector-db, retrieval-rerank, mcp-tooling
└─────────────────────────────────┘
```

Example: a Pinecone Node.js PR loads `cross-cutting/*` + `lang-nodejs.md` + `category-vector-db.md`.
A LangChain Python PR loads `cross-cutting/*` + `lang-python.md` + `category-agent-framework.md`.
A mixed PR loads both relevant category files.

---

## Key principles

- **Pattern-based, not provider-specific.** Rules say "streaming APIs must classify sync vs Promise" — not "framework X's `.stream()` returns a Promise." Applies to any provider or framework, including ones that don't exist yet.
- **Vector DBs: capture text chunks + dimensions, never raw vectors.** Embedding vectors are large float arrays with no debugging value. Capture `{ dimensions: N }` for the vector, and the text chunk that was embedded (with truncation). Never serialize `number[]` / `List[float]` into a span.
- **Competitor claims require evidence.** Any "Platform X does not support Y" must link to source + date. The rubric tracks 12+ platforms but asserts nothing about them.
- **Systemic issues downgraded.** If a violation's root cause is in unchanged code (e.g., a shared constant), it's flagged as a Nit for awareness, not a PR blocker.

---

## How to extend

### Add a new rule to an existing rubric

1. Open the relevant `references/*.md` file
2. Add a new H3 section:

```markdown
### XX-N: Short rule name

**Rule:** What must be true.
**Origin:** Where this lesson came from.
**Detection cue:** How to spot the violation (plan-mode and/or code-mode).
**Severity:** Blocker / Warning / Nit
**Suggested fix:** What to do instead.
```

3. Use the next available number for the prefix (e.g., if last rule is `CS-7`, use `CS-8`)

### Add a new target category

1. Create `references/category-<name>.md` with rules following the same format
2. Add the category to the dispatch table in `SKILL.md` Phase 2

### Add a new language

1. Create `references/lang-<name>.md`
2. Add the language to the dispatch table in `SKILL.md` Phase 1

### Add a new competitor to the claims rubric

1. Open `references/cross-cutting/competitor-claims.md`
2. Add the platform name to the bullet list under "Tracked Platforms"

---

## Rule count summary

| Rubric | Rules | Key concerns |
|--------|-------|-------------|
| cross-cutting/module-system | 8 | ESM patching, CJS ordering, auto vs manual tracing, OTel vs custom |
| cross-cutting/version-matrix | 6 | Field renames, version detection, forward compat |
| cross-cutting/runtime-shape | 7 | Sync/async, null handling, callbacks, idempotency |
| cross-cutting/content-safety | 7 | Binary, truncation, schemas, secrets, serialisation |
| cross-cutting/competitor-claims | 4 | Evidence linking, verification dates |
| cross-cutting/otel-hygiene | 6 | Span naming, attributes, usage fields |
| cross-cutting/span-deduplication | 6 | Cross-framework, HTTP filtering, reparenting |
| lang-nodejs | 13 | InstrumentationBase, rollup, _isPatched, test layout |
| lang-python | 12 | ImportError guards, bundle config, bare except, async |
| lang-java | 7 | ThreadLocal, ByteBuddy, volatile |
| lang-claude-plugin | 9 | Hook scripts, secrets, base64, state races |
| category-llm-provider | 8 | Request/response attrs, streaming, tools, multimodal |
| category-agent-framework | 8 | Hierarchy, aggregated usage, state-graph, dedup |
| category-vector-db | 8 | No raw vectors, text chunks + dimensions only, batch storm |
| category-retrieval-rerank | 4 | Document truncation, scores, pipeline linking |
| category-mcp-tooling | 4 | Arg redaction, transport, schema safety |
| **Total** | **~117** | |

---

## Related skills

| Skill | Relationship |
|-------|-------------|
| `/sdk-add-llm-instrumentation` | Authoring guide for Node instrumentations — checklist items mirrored as review rules here |
| `/sdk-py-add-instrumentation` | Same for Python |
| `/sdk-check-provider` | Pre-implementation provider check — run before authoring, review after |
| `/code-review` | Generic code review — this skill is SDK-specific and runs separately |
