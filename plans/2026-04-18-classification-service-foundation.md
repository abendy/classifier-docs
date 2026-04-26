# Plan: Classification Service Foundation

## Date

2026-04-18

## Why this plan exists

Stand up a dedicated classification service that consumes content envelopes from edge services (starting with X bookmarks) and produces multi-faceted classifications. The service is the first component built to conform to the dada.stream contract layer from day one, rather than retrofitting conformance later.

The plan has three goals beyond "classify bookmarks":

1. **Exercise the dada.stream contract layer for real.** `~/projects/dada.stream` is the authoritative architecture and contract repository for this ecosystem and already lists Classifier as a planned component with a sketched manifest (`contracts/service-manifest.md` lines 411–505). The scraper predates dada.stream and doesn't yet conform; the classifier is a clean slate and should.
2. **Make the audit/observability layer load-bearing.** Every LLM call and every indeterminate optimization decision must be traceable, replayable, and measurable. Build it in the classifier as the reference implementation, shape it as if already shared, and propose extracting it into `dada.stream/platform/infrastructure/` once stable.
3. **Apply the contract discipline the scraper missed.** The code review (`~/projects/x-bookmarks-scraper/.project/tasks/2026-03-07-carmack-code-review.md`) showed that manifests and contracts in the scraper were dead weight because no code read them. This plan treats the dada.stream manifest and event envelope as first-class, gives the manifest live consumers (service itself, future intelligence layer, operator CLI) from day one, and avoids inventing parallel shapes.

Classification itself is the first real workload; the service is designed as a general **content-labeling layer** for any dada.stream edge service producing content envelopes.

## Placement in the dada.stream ecosystem

dada.stream's layered model:

```text
┌─────────────────────────────────────────────────────────┐
│  Intelligence    LLM brain, skills, agents              │
├─────────────────────────────────────────────────────────┤
│  Control Plane   Orchestration, topic mgmt, UI/UX       │
├─────────────────────────────────────────────────────────┤
│  Core Services   Classifier, router, search, store      │  ← this service
├──────────┬──────────┬───────────┬───────────────────────┤
│  X Sync  │ YT Sync  │ Email     │  Web / Manual         │
└──────────┴──────────┴───────────┴───────────────────────┘
```

The classifier sits in Core Services. It:

- **Subscribes** to `content-ingested` events from any edge service (X sync first; others later without code change).
- **Reads** the referenced content envelope (from the content store when it exists; directly from the scraper's DB via a thin mapper in v1).
- **Enriches** the envelope's `classification` section with tags, topic assignments, and proposed additional facets (sentiment, intent).
- **Publishes** `content-classified` or `content-classified-low-confidence` events.
- **Exposes** HTTP commands per the service manifest spec — invokable by the control plane and the intelligence layer.

Neither the event bus nor the service registry nor the content store exists yet. The classifier is designed so:

- Event bus = poll the scraper's outbox as a read-only log; each newly observed source event is copied into a classifier-side ingest queue that tracks its own `attempts`, `available_at`, `dispatched_at`, and `status`. The scraper's outbox state is never mutated by the classifier, so its existing retry semantics are preserved; classifier retries live on the classifier side. An emitter abstraction swaps the outbound path to a real bus later.
- Service registry = attempt to register on startup if reachable; log and continue otherwise.
- Content store = mapper reads the scraper's DB directly for v1 and materializes content envelopes on the classifier side.

No runtime coupling to dada.stream specs today; full structural coupling from the first line of code.

## Non-goals for v1

- Agent harness that opens PRs. The substrate (traces, runs, eval) is built; the autonomous loop is v2.
- Cluster discovery. Needs labeled data and stable topics first.
- Automatic hierarchy derivation from clusters. Topics/subtopics are curated in YAML for v1.
- Quality scoring as a distinct facet. Encoded into intent/sentiment for v1.
- Multimodal (image/video) classification. Content envelope leaves room; no models attached.
- Web UI. Revisited only if TUI + notebooks fall short.
- Postgres migration. Schema designed Postgres-shaped; v1 runs SQLite.
- Extraction of the audit layer to a shared package. Done when the scraper adopts it or dada.stream accepts the spec.
- Topic manager, router, content store, intelligence layer — separate dada.stream components, not in scope here.

## Scope for v1

- Python service, `uv`-managed, `ruff` / `pyright` (`standard` mode) / Pydantic v2. No mypy. See `.project/prompt.md` (slice 1) for the full toolchain rationale.
- Publishes a dada.stream-conformant **service manifest** (JSON), registers with the registry if reachable.
- Consumes `content-ingested` envelopes referring to X bookmarks (via scraper outbox poll + envelope mapper).
- Emits `content-classified` and `content-classified-low-confidence` with dada.stream event envelope shape.
- Classification facets: topics (~75, single assignment usually) + subtopics (hierarchical) + freeform tags + sentiment + intent.
- Streaming pipeline runs on the Mac Mini M4 (24 GB); heavier jobs also run on the Mini overnight.
- Storage: SQLite (WAL) + sqlite-vec for vectors + DuckDB attached for analytics.
- **HTTP API** (FastAPI) implementing the manifest's commands.
- **CLI** (small, config-heavy) for operator use — wraps the HTTP API locally plus adds job invocations.
- **Textual TUI** for inbox review.
- **marimo notebooks** for exploration.
- **Audit layer**: OTel spans + Phoenix traces + `runs` table + eval runs, correlation/causation threaded end-to-end.
- Seed labels via the TUI inbox — no upfront labeling bolus.

## Repository

Sibling repo to `x-bookmarks-scraper`, not a monorepo. Conforms to the dada.stream convention: "Each component/service lives in its own repository." When developing the classifier, add the dada.stream directory as an additional working directory for reference.

Service name: **`prism`** (selected from `atlas` / `compass` / `prism` / `signal` / `sieve` — multi-faceted classification IS splitting content into facets; the metaphor is exact). The Python package, CLI entry point, FastAPI title, `service.json` identity, and emitted-event `source` field all use `prism`. The containing directory stays `components/classifier/` so the components tree can group future classifier-flavored services side by side. This plan still says "the classifier" in generic prose below — treat that as shorthand for "this service".

This plan lives in the scraper repo until the sibling repo exists, then moves.

## Compute placement

The Mini (24 GB, always-on) hosts the service. The M5 Max (128 GB) is a burst machine for ad-hoc heavy work when plugged in. Hetzner or similar is the escape valve for workloads exceeding the Mini's overnight budget.

| Workload | Host | Notes |
| --- | --- | --- |
| HTTP API + per-item streaming classification | Mini | <500 ms/item p95, ~8 GB resident |
| Small-LLM facet heads per item | Mini | Qwen 2.5 7B @ 4-bit (~5 GB) or similar |
| Nightly clustering (v2) | Mini (overnight) | Streaming idle → ~14 GB available |
| Small-encoder LoRA fine-tuning | Mini (overnight) or Max | ModernBERT / DeBERTa-v3-base |
| 7B+ LoRA fine-tune | Max or Hetzner | Not in v1 scope |
| DSPy program compilation | Mini (overnight) or Max | |
| Eval harness | Either | Cheap |
| Agent harness (v2) | Max or dev laptop | Uses Claude API, no local burden |

Mini memory budget (24 GB):

- OS + other services reserved: ~8 GB
- Serve mode (always-on): ~8 GB envelope
- Overnight headroom while serve idles: ~14 GB

Jobs are CLI invocations reading the same config as `serve`. Portable to GitHub Actions self-hosted runners, Modal, or a Hetzner box without code change.

### Scheduling

- launchd on the Mini for nightly/off-hours jobs; one plist per job.
- Manual `classifier job <name>` invocations on the Max for ad-hoc.

## Service shape

One codebase, three process entry points, shared DB:

- **`classifier serve`** — always-on. Runs the HTTP API (FastAPI), polls for `content-ingested` events, runs the streaming pipeline, emits events, backs the TUI inbox.
- **`classifier job <name>`** — scheduled or ad-hoc. Heavy work: backfill, eval, fine-tuning, taxonomy validation, clustering (v2), DSPy compilation.
- **`classifier review`** — launches the Textual TUI; connects to the same backing service or the DB.

No second service, no worker pool.

## Classification pipeline

Fan-out from a single embedding pass:

```text
content envelope
  │
  ▼
[ embed once ]  ──────────────▶ stored, keyed by (envelope_id, model_version)
  │
  ├────────▶ [ topic head ]     retrieval + LLM-pick over 75 topic prototypes
  ├────────▶ [ subtopic head ]  LLM picks subtopic(s) within selected topic(s)
  ├────────▶ [ tags head ]      LLM + structured output, freeform tags
  ├────────▶ [ sentiment head ] proposed addition to dada.stream classification section
  └────────▶ [ intent head ]    proposed addition to dada.stream classification section
                  │
                  ▼
           [ reconcile + enrich envelope's classification section ]
                  │
                  ▼
           emit content-classified  (or content-classified-low-confidence)
```

### Embedding

- **Model**: `BAAI/bge-large-en-v1.5` (1024-d, English, ~335M params, 512-token context) via fastembed. Selected as the supported BGE-family member in fastembed 0.8 after `BAAI/bge-m3` was dropped from `TextEmbedding`; storage shape (1024-d) is unchanged. Trade-offs (multilingual loss, sparse-retrieval loss, 8K → 512 context drop) and the named-rejected alternatives are recorded in ADR 017.
- **A/B candidates** once seed labels accumulate: a multilingual + long-context replacement for the lost BGE-M3 properties (e.g. multilingual-e5-large), jina-v3, Qwen3-Embedding-4B.
- **Storage**: `(envelope_id, model_version, vector)` in sqlite-vec. Raw content is the envelope itself; embeddings are derived.
- **Version tracking**: every run record captures `embedding_model_version`.

### Topic head — retrieval + LLM-pick

75 topics is too many for zero-shot LLM per item. Prompt bloats, calibration degrades. Instead:

1. **Offline**: embed each topic's definition + 2–4 exemplars from `topics.yaml`. Store prototype vectors in sqlite-vec.
2. **Per item**: cosine similarity between item embedding and topic prototypes, take top 5 candidates.
3. **LLM pick**: small local LLM receives the envelope's text + top-5 topic definitions, returns one topic ID with reasoning and confidence (Instructor/Outlines structured output).
4. **Unsure is valid**: confidence below threshold → `topicAssignments` empty, emit `content-classified-low-confidence` for inbox triage.

### Subtopic head

- `topics.yaml` defines subtopics per topic.
- Once a topic is chosen, a second (or bundled) LLM call picks applicable subtopic(s) with confidences.
- The shared `TopicAssignment` shape carries a singular optional `subtopicId`. When multiple subtopics apply, emit one `TopicAssignment` per subtopic that shares the parent `topicId` — rather than packing multiple subtopics into one assignment. Consumers aggregating by `topicId` see the full set; consumers pinned to the contract shape read each assignment correctly.

### Tags head — freeform

- `classification.tags: string[]` per the dada.stream content model — freeform.
- LLM emits 2–6 tags via structured output. No hard taxonomy; normalization is light (lowercase, hyphenate, strip punctuation).
- Tags surface genuinely novel signals not fitting the topic tree.

### Sentiment and intent heads (proposed additions)

dada.stream's classification section doesn't define sentiment or intent. v1 proposes adding them as additive, non-breaking fields:

- **`sentiment`**: one of `{positive, negative, neutral, mixed}` with confidence.
- **`intent`**: one of `{learn, reference, entertain, act, unclear}` with confidence. Starter set; refine with usage.

ADR to land in dada.stream before the classifier's first public release: `dada.stream/platform/architecture/adr/XXX-classification-sentiment-intent-fields.md`. Additive schema change per dada.stream event-types.md L408–410.

### Reconcile and emit

Facet outputs collected. Classification section written to the envelope's classification block. Run record persisted. One `content-classified` event emitted (or `-low-confidence` if the topic head was unsure).

## Event contract

### Envelope

Adopt dada.stream's event envelope verbatim (`contracts/event-types.md` L14–26):

```json
{
  "id": "01JX7...",
  "type": "content-classified",
  "source": "classifier",
  "timestamp": "2026-04-18T10:30:15Z",
  "correlationId": "01JX7... (envelope id)",
  "causationId": "01JX7... (content-ingested event id)",
  "payload": { ... },
  "version": "1.0.0"
}
```

`correlationId` is always the content envelope's `id` — stable across every classification, re-classification, and correction of that envelope. `causationId` is the **immediate trigger** ID: the `content-ingested` event ID for first-pass classification, and a fresh command ID minted when an HTTP `reclassify-content` or `edit-classification` call is received (persisted in the run record, also emittable as a command event if/when the bus carries commands). This keeps the causal chain one hop wide so downstream consumers can distinguish a user correction or scheduled re-run from the original ingest-driven classification. Run records and Phoenix traces carry both IDs.

### Consumed: `content-ingested`

Per `contracts/event-types.md` L72–83. Payload:

| Field | Type | Meaning |
| --- | --- | --- |
| `envelopeId` | string | Content envelope ID |
| `contentType` | string | `post`, `video`, etc. |
| `service` | string | Source edge service |
| `sourceId` | string | Original platform ID |

### Emitted: `content-classified`

Per `contracts/event-types.md` L87–102. Payload:

| Field | Type | Meaning |
| --- | --- | --- |
| `envelopeId` | string | Envelope ID |
| `tags` | string[] | Freeform tags |
| `topicAssignments` | TopicAssignment[] | `{topicId, subtopicId?, confidence, matchedTerms?}` |
| `confidence` | number | Aggregate confidence |
| `classifiedBy` | string | `classifier@0.1.0 + bge-large-en@1.5 + qwen2.5-7b@4bit + topics@2026.04.1` |
| `sentiment` (proposed) | `{label, confidence}` | Additive |
| `intent` (proposed) | `{label, confidence}` | Additive |

### Emitted: `content-classified-low-confidence`

Per event-types.md L134–146. Payload:

| Field | Type | Meaning |
| --- | --- | --- |
| `envelopeId` | string | Envelope ID |
| `confidence` | number | Score that fell below threshold |
| `candidateTopics` | TopicAssignment[] | Best guesses |
| `reason` | string | `retrieval-low-similarity`, `llm-pick-unsure`, `no-topic-match`, etc. |

### Versioning

- `version` in every envelope. Consumers pin or track latest.
- Pydantic v2 models are source of truth; JSON Schema generated at build time and committed under `contracts/`.
- A topic/subtopic edit that changes IDs bumps minor; a structural change bumps major. Sentiment/intent addition is a minor bump.

## Service manifest

Adopt dada.stream's JSON manifest format (`contracts/service-manifest.md`). Starting shape, extending the sketch at lines 411–505:

```json
{
  "identity": {
    "name": "classifier",
    "version": "0.1.0",
    "description": "Multi-faceted content classification for any dada.stream content envelope.",
    "language": "python",
    "manifestVersion": "1.0.0"
  },
  "capabilities": [
    {
      "type": "classify",
      "description": "Applies topics, subtopics, freeform tags, sentiment, and intent to content envelopes.",
      "contentTypes": ["post", "video", "article", "email", "note", "bookmark", "audio", "image", "document"]
    }
  ],
  "commands": [
    {
      "name": "classify-content",
      "description": "Classify a single content envelope. Returns the enriched classification section.",
      "parameters": [
        {"name": "envelopeId", "type": "string", "required": true, "description": "Envelope to classify."}
      ],
      "response": {"type": "object", "description": "Classification section written to the envelope."},
      "sideEffects": ["emits content-classified", "writes run record"],
      "idempotent": false
    },
    {
      "name": "reclassify-content",
      "description": "Re-classify an envelope with the current topics/models, useful after taxonomy changes.",
      "parameters": [
        {"name": "envelopeId", "type": "string", "required": true, "description": "Envelope to re-classify."},
        {"name": "reason", "type": "string", "required": false, "description": "Why (logged)."}
      ],
      "response": {"type": "object", "description": "Newly written classification section; the prior one is retained as a run record."},
      "sideEffects": ["emits content-classified", "writes run record"],
      "idempotent": false
    },
    {
      "name": "get-classification",
      "description": "Return the current classification for an envelope.",
      "parameters": [{"name": "envelopeId", "type": "string", "required": true, "description": "Envelope."}],
      "response": {"type": "object", "description": "Current classification section, or 404 if the envelope has not been classified."},
      "sideEffects": [],
      "idempotent": true
    },
    {
      "name": "list-topics",
      "description": "Return the active topic taxonomy with subtopics.",
      "parameters": [],
      "response": {"type": "array", "description": "List of topics, each with id, displayName, definition, and subtopics."},
      "sideEffects": [],
      "idempotent": true
    },
    {
      "name": "get-low-confidence",
      "description": "Return envelopes awaiting inbox review (low-confidence classifications). Each item carries the runId of the specific proposal being shown, which accept/edit must echo back.",
      "parameters": [
        {"name": "limit", "type": "number", "required": false, "description": "Max items.", "default": 50}
      ],
      "response": {"type": "array", "description": "Envelopes whose most recent classification fell below the confidence threshold, each with its runId and proposal hash, ordered for inbox triage."},
      "sideEffects": [],
      "idempotent": true
    },
    {
      "name": "accept-classification",
      "description": "Accept a specific proposed classification (identified by runId) as user-confirmed; promotes it to the labeled set.",
      "parameters": [
        {"name": "envelopeId", "type": "string", "required": true, "description": "Envelope."},
        {"name": "runId", "type": "string", "required": true, "description": "The run whose proposal the reviewer saw. If a newer run has superseded it, the request is rejected as stale — the reviewer is redirected to the fresh proposal."}
      ],
      "response": {"type": "object", "description": "Label record written, including the accepted facets, the runId it was anchored to, and the labeler identity. Repeat calls for the same (envelopeId, runId) return the existing label record unchanged."},
      "sideEffects": ["writes labels record"],
      "idempotent": true
    },
    {
      "name": "edit-classification",
      "description": "Override facets of a specific proposed classification (identified by runId). User correction.",
      "parameters": [
        {"name": "envelopeId", "type": "string", "required": true, "description": "Envelope."},
        {"name": "runId", "type": "string", "required": true, "description": "The run whose proposal the reviewer saw. Stale runIds are rejected; the reviewer is redirected to the latest proposal to re-edit."},
        {"name": "changes", "type": "object", "required": true, "description": "Fields to override."}
      ],
      "response": {"type": "object", "description": "Updated classification section and the label record persisting the correction, anchored to the submitted runId."},
      "sideEffects": ["writes labels record", "emits content-classified"],
      "idempotent": false
    },
    {
      "name": "run-eval",
      "description": "Run the eval harness against the held-out labeled set.",
      "parameters": [
        {"name": "baselineRunId", "type": "string", "required": false, "description": "Optional run to diff against."}
      ],
      "response": {"type": "object", "description": "Eval run ID plus per-facet metric summaries and diffs against the baseline run when provided."},
      "sideEffects": ["writes eval run record"],
      "idempotent": false,
      "async": true
    }
  ],
  "api": {
    "protocol": "http",
    "host": "localhost",
    "port": 3200,
    "basePath": "/api/v1",
    "endpoints": [
      {"command": "classify-content",       "method": "POST", "path": "/classify"},
      {"command": "reclassify-content",     "method": "POST", "path": "/classify/re"},
      {"command": "get-classification",     "method": "GET",  "path": "/classifications/{envelopeId}"},
      {"command": "list-topics",            "method": "GET",  "path": "/topics"},
      {"command": "get-low-confidence",     "method": "GET",  "path": "/inbox/low-confidence"},
      {"command": "accept-classification",  "method": "POST", "path": "/inbox/{envelopeId}/accept"},
      {"command": "edit-classification",    "method": "POST", "path": "/inbox/{envelopeId}/edit"},
      {"command": "run-eval",               "method": "POST", "path": "/eval"}
    ]
  },
  "events": {
    "publishes": [
      {"type": "content-classified", "description": "Classification completed for an envelope."},
      {"type": "content-classified-low-confidence", "description": "Classification was attempted but fell below threshold — candidate for review."}
    ],
    "subscribes": [
      {"type": "content-ingested", "description": "Triggers classification when new content enters the system."}
    ]
  },
  "health": {
    "healthCheck": {"endpoint": "/health", "method": "GET", "interval": 30, "timeout": 5}
  },
  "configuration": {
    "required": [
      {"name": "DATABASE_PATH", "type": "string", "description": "SQLite DB path.", "secret": false},
      {"name": "SCRAPER_DB_PATH", "type": "string", "description": "Path to x-sync SQLite for v1 envelope mapping.", "secret": false}
    ],
    "runtime": [
      {"name": "TOPIC_CONFIDENCE_THRESHOLD", "type": "number", "description": "Below this, emit low-confidence event.", "secret": false, "default": 0.55}
    ]
  },
  "dependencies": {
    "services": [],
    "infrastructure": ["event-bus (optional in v1 — polled outbox instead)"]
  }
}
```

The manifest is loaded at startup, validated against the dada.stream spec, and registered with the service registry if one is reachable. The operator CLI (`classifier status`, `classifier manifest`) reads the same file.

## Contract discipline

Learning from the reviewed dead contracts: build only what has a live reader.

- **Pydantic v2 models** are the single source of truth for event payloads, envelopes, manifest, config, and internal API boundaries.
- **JSON Schema** generated from Pydantic, committed under `contracts/` as reviewable artifacts. PR diffs show contract changes.
- **One validation system.** No parallel completeness/assessment layer.
- **The dada.stream specs are the authoritative shape.** Our Pydantic models conform to them — and where we propose additions (sentiment/intent), we ship them as ADRs in dada.stream first.
- **One Markdown page** (`contracts/INTEGRATION.md`) pointing at the dada.stream specs we conform to, with any service-specific details and version alignment. Not a spec directory.
- Revisit a service registry and capability catalog implementation only when a third dada.stream component exists.

## Audit / observability layer

Foundational concern, not a classifier feature. Built inside the classifier as the reference implementation; the API shaped as if the shared infrastructure package already exists so extraction is a move, not a rewrite. dada.stream's `infrastructure/` folder is currently empty — this work will propose filling it.

### Components

| Element | Purpose |
| --- | --- |
| **Trace** | OTel spans + Phoenix viewer. Every LLM call and facet head wrapped. `correlationId` + `causationId` carried throughout. |
| **Run record** | One row per pipeline execution. `runId` (UUID v7) is the primary key and the stable anchor that downstream inbox actions pin to. Carries envelope ref, input hash, model versions, topic version, outputs, confidences, timing, cost, `trace_id`, `correlation_id`, `causation_id`, `superseded_by` (immediate successor runId for audit / history). The **authoritative "current" pointer per envelope lives on the `ClassificationRepo` row** (`active_run_id`), updated atomically whenever a new run lands. Staleness checks read that single pointer — no chain walking through `superseded_by` — so an old submission resolves directly to the current active run, never to another stale intermediate. Queryable via DuckDB. |
| **Eval run** | A scored pass over the labeled set, tagged by config hash. Metrics + per-item diff vs gold. Diffable across runs. |
| **Proposal record** (v2) | Schema shipped; consumer deferred. For agent-proposed taxonomy/prompt/config changes with eval deltas and merge status. |
| **Replay** | Any run re-runnable from stored inputs; deterministic where models allow. |

### API

Decorator/context-manager style in Python:

```python
with audit.operation(
    "classify",
    envelope_id=envelope.id,
    correlation_id=envelope.id,
    causation_id=ingested_event.id,
    topics_version="2026.04.1",
) as op:
    emb = op.step("embed", model="bge-large-en@1.5")(embed_fn, envelope.content)
    topic = op.llm_step("topic", model="qwen2.5-7b@4bit")(pick_topic, emb.top(5))
    tags = op.llm_step("tags", model=...)(...)
    ...
    event = reconcile_and_enrich(envelope, topic, tags, ...)
    op.emit(event)
```

Every call produces an OTel span, a Phoenix trace entry, a row in `runs`, and an emitted event at commit time. All correlated by `run_id` and the dada.stream `correlationId`.

### Logging

Structured JSON. Field set per `docs/spec/observability.md` (scraper's) + dada.stream's correlation/causation fields: `service`, `correlation_id`, `causation_id`, `operation`, `envelope_id`, `duration_ms`, `error`.

### Extraction path

- v1: `classifier/audit/` with a clean import boundary.
- Propose the spec as `dada.stream/platform/infrastructure/observability.md` once stable. Python implementation first; TypeScript port when the scraper adopts.

## Storage

### Databases

- **SQLite (WAL)** for OLTP: classifications, runs, labels, review events, eval results, ingest queue, emitted event outbox, topic cache, envelope cache (mapped from scraper for v1).
- **sqlite-vec** for vectors: item embeddings, topic prototypes, future multimodal embeddings.
- **DuckDB (attached read-only)** for analytics: cluster stats (v2), label co-occurrence, eval deep-dives.

All three are file-based. No infrastructure.

### Repository pattern

Business logic never touches SQL. Repositories:

- `EnvelopeRepo` — cached content envelopes (mapped from scraper or read from content store later).
- `ClassificationRepo` — one row per envelope_id holding the current classification section and `active_run_id`. The pointer is the single source of truth for "which proposal is live right now"; inbox staleness checks resolve directly to it.
- `RunRepo` — run records.
- `EmbeddingRepo` — item and prototype vectors.
- `LabelRepo` — human-labeled ground truth (from inbox).
- `TopicRepo` — topics/subtopics loaded from YAML, cached for JOINs.
- `IngestQueueRepo` — classifier-side copy of observed source events with per-event `attempts`, `available_at`, `dispatched_at`, and `status`. The classifier never mutates the scraper's outbox; poll cursor is derived from `MAX(source_event_id)` already mirrored here. Retries live on this table.
- `EventOutboxRepo` — emitted events, dispatched_at pattern.
- `ReviewEventRepo` — inbox actions that are not ground-truth labels (reject, defer). Kept separate from `labels` so training/eval stay clean.

Swap to Postgres = new repo implementations, delete SQLite ones.

### Schema discipline

- Postgres-portable types. No SQLite-specific patterns (`ROWID`, implicit coercion).
- Datetimes as ISO-8601 at boundary; `datetime` internally.
- UUID v7 for envelope IDs and event IDs, per dada.stream decisions.
- **Alembic migrations from day one.** No runtime `ALTER TABLE` (lesson from the scraper).

### Migration trigger to Postgres

Any one of: concurrent writers across machines become real, cross-service joins frequent, ~10M+ items. Probably 6–18 months out.

## Envelope mapping (v1 bridge)

The scraper doesn't yet emit dada.stream content envelopes. For v1, a mapper in the classifier reads the scraper's SQLite (read-only) and materializes envelopes on demand:

- `tweets` + `users` + `media` → content envelope with `contentType = post`, `source.service = x-sync`, `content.body = tweet.text`, `content.author = ...`, `content.mediaUrls = ...`, etc. The `bookmarks` table is **not** read by the mapper: its columns (`folder_id`, `bookmarked_at`, `deleted_from_x`) would only surface through `source.sourceData`, and folder → tag/topic/routing translation is downstream-pipeline work (routing service's concern). Explicit bookmark-field preservation is deferred until a consumer demonstrably needs it.
- **The "this tweet was bookmarked" filter is explicit, not implicit.** Verified against the scraper source (`x-bookmarks-scraper/src/lib/db/tweet-write-repo.ts`, `src/temporal/activities/store.ts`): the scraper emits `entityType='tweet'` events for both bookmark-driven syncs AND enrichment/included-tweet paths. A `tweet`-only filter would ingest a large non-bookmark corpus. The ingest loop therefore uses `bookmark.created` as the sole bootstrap signal and treats `record.synced` / `record.enriched` as refresh-only (enqueued only when an envelope already exists for that tweet). See ADR 006 (bridge scope) for the broader framing and ADR 007 (envelope refresh invariants) for the refresh semantics.
- **What is authoritative in the scraper repo:** the live table schema (`schema.ts`) and the outbox table. Real rows, real columns, real sync mechanism.
- **What is not authoritative:** the scraper's `src/contracts/` directory (`routable-content.ts`, `manifest.ts`, `events.ts`, `capabilities.ts`, `planning.ts`, `json-schema.ts`). Per the Carmack reviews (`~/projects/x-bookmarks-scraper/.project/tasks/2026-03-07-carmack-*-review.md` in the scraper repo), these are half-thought-out additions that were never load-bearing — aspirational scaffolding, not shipped contracts. The classifier does not "port" them. The mapping logic is the classifier's concern from scratch; the scraper's live schema is the only reference it needs.
- **The scraper's own plan agrees.** `~/projects/x-bookmarks-scraper/.project/plans/2026-02-13-foundation-architecture-core-runtimes-capabilities.md` lines 7–22 explicitly flag the plan's proposed contracts as "superseded" by dada.stream's specs — `RoutableContent` is replaced by our `ContentEnvelope`, the scraper's `ServiceEventEnvelope` v1 is replaced by dada.stream's event envelope, etc. When the scraper cleanup picks up, the work is "mostly conformance to dada.stream's existing specs, not inventing new ones" (their words). The classifier's mapper targets the dada.stream shape; the scraper's future cleanup will emit envelopes of the same shape, at which point this interim mapper is deleted.
- **Consumer-side decoupling for slice 4:** the scraper's outbox table has row-level columns for event metadata (`id`, `event_type`, `created_at`, etc.) but the entity identifiers live inside the JSON `payload` column (shape currently follows the scraper's pre-dada.stream envelope: `{entityType, entityId, data, provenance, ...}`). The classifier parses the payload **minimally — only `entityType` and `entityId`** — and uses those two fields to route events into its own ingest queue. Everything else in the payload (`data.*`, `provenance.*`, `version`, etc.) is ignored. This gives the classifier a two-field dependency on the scraper's payload shape; the scraper can freely reshape the rest on its path toward dada.stream conformance.
- Scraper outbox provides the `content-ingested`-equivalent signal; classifier wraps it into a proper event envelope for internal causal tracing.

When the scraper is cleaned up to emit dada.stream-conformant `content-ingested` events directly (separate work, likely driven by a scraper-focused agent consuming this repo's `envelope.py` / `service.json` as the target shape), this mapper is deleted and the classifier subscribes to the real event bus.

### Known quirks to revisit

- **Media type-fidelity under URL fallback.** When `media.url` is NULL (common for `video` / `animated_gif` rows in the scraper), the mapper falls back to `preview_image_url` but keeps the original `type` ("video" / "animated_gif"). The resulting `MediaRef` then carries a thumbnail URL paired with a video type — accurate about the signal's origin, inaccurate about what the URL actually points to. Accepted for now because no downstream consumer (embedder, UI) exists to distinguish "video I can play" from "thumbnail I can display." Revisit when the first consumer lands; options at that point: skip fallback rows entirely, re-classify `type` to `"thumbnail"` when falling back, or keep the current behavior and document it at the consumer.

## Topics, subtopics, and tags

### Files

- **`topics.yaml`** — ~75 topics, each with: `id`, `displayName`, `definition` (1–2 sentences), `exemplars[]` (2–4 short snippets), `subtopics[]` (each with `id`, `displayName`, `definition`).
- Both versioned in the classifier repo. `version:` at file head; referenced in event payloads as `classifiedBy`.

Freeform tags aren't curated — they're produced per-item by the tags head.

### Reconciliation with dada.stream

- dada.stream's `TopicAssignment` = `{topicId, subtopicId?, confidence, matchedTerms?}`. Matches our output.
- dada.stream's content model supports tags as `string[]`. Matches.
- Sentiment and intent are **proposed additions** (see below); encoded as tags-with-prefix in the interim if the ADR lands later than first emission.

### Evolution

- Edits via PR. Code review over topic changes.
- Agent harness (v2) proposes PRs with eval-delta justification.
- Cluster discovery (v2) proposes new topics/subtopics; promoted to YAML via PR.

### Prototype management

- On topics.yaml change: a job embeds each topic's definition + exemplars, writes prototype vectors tagged with the topics version.
- Old prototype vectors retained for replay.

## Proposed additions to dada.stream

Filed as ADRs in `dada.stream/platform/architecture/adr/` before being relied on:

1. **Classification sentiment and intent fields**. Additive fields on the `classification` section of the content envelope and on the `content-classified` event payload. Non-breaking per event-types.md schema-evolution rules.
2. **Observability infrastructure spec**. Fills `dada.stream/platform/infrastructure/observability.md`. Covers the trace + run record + eval run + correlation/causation conventions. Ports the classifier's implementation to a language-agnostic spec.
3. **Event bus transport in v1 is polled outbox**. Formalizes the bridging pattern while a real bus doesn't yet exist. Mirrors the scraper's existing outbox model.
4. **Service registry optionality**. Services may register when a registry is reachable; they are not required to depend on one. Enables incremental rollout.

These ADRs live in dada.stream, not this service's repo — the service conforms to dada.stream, not the other way around.

## Seed labeling

No upfront 500-item bolus. The inbox flow produces seed data as you use it.

1. Classifier runs on all existing bookmarks as a backfill job. Every item gets a prediction + confidence; low-confidence items emit `content-classified-low-confidence`.
2. TUI inbox lists low-confidence items first, then everything by age. Each rendered item is pinned to a specific `runId` — the proposal that produced what the reviewer is seeing. The runId (plus a hash of the proposed facets) travels with every accept/edit submission.
3. User actions per item: **accept** (all facets) · **edit** (modify any facet) · **reject** (send back for re-run) · **defer** (skip without judgement).
4. Accept/edit must carry the `runId` the reviewer saw. Staleness is checked by comparing against `ClassificationRepo.active_run_id` for the envelope — a single lookup that always resolves to the live proposal, not to another superseded one. If the submitted runId doesn't match, the submission is rejected as stale and the inbox refreshes with the current proposal; the reviewer re-judges. Accept for an `(envelopeId, runId)` that already has a label is a no-op returning the existing row, which is what keeps `accept-classification` truly idempotent under retry. This also prevents the reviewer's judgement from being silently applied to a proposal they never saw.
5. Only **accept** and **edit** write to `labels` — these are the actions that assert ground-truth facets, so they feed both training and eval. **Reject** triggers a `reclassify-content` and records a review event (not a label) noting which facet was wrong if the user specified one. **Defer** sets a review timestamp on the inbox item and writes nothing else. Mixing reject/defer into `labels` would poison the held-out split and any downstream DSPy/fine-tuning run.

### Eval split

- 20% of labeled items reserved for held-out eval (random per-item, deterministic via hash on envelope ID).
- Never shown as training data to DSPy or fine-tuning jobs.
- Eval run reports per-facet metrics: F1 for topic, F1 for subtopic conditional on topic, Jaccard for tags, accuracy for sentiment/intent.

## HTTP API (via FastAPI)

Primary external interface. One endpoint per manifest command, implemented by the same service that runs the streaming pipeline. Shape matches the manifest's `api.endpoints` block. Health at `/health`. OpenAPI spec auto-generated from Pydantic models.

## CLI

Operator tooling. Small surface, config-heavy, `--json` everywhere for agent consumption. Most subcommands wrap the HTTP API so there's one code path.

```text
classifier serve                         # always-on: HTTP + streaming pipeline
classifier job <name> [--args]           # scheduled or ad-hoc
classifier review                        # launch Textual TUI
classifier eval [--baseline <run_id>]    # run eval, print metrics
classifier topics list|diff|validate     # topic tools
classifier trace <run_id> [--open]       # dump trace or open Phoenix
classifier classify <envelope_id>        # manual classify (wraps HTTP)
classifier status                        # manifest + recent runs summary
classifier manifest                      # emit the JSON manifest
```

Behavior lives in `config.yaml`, not flags.

## TUI

Textual. Two primary screens:

- **Inbox**: list pane + detail pane. Keys: `a` accept, `e` edit, `r` reject, `d` defer, `j/k` navigate, `space` multi-select, `/` search. Batch accept on selection.
- **Eval review**: side-by-side predicted vs gold for mis-classifications, filterable by facet and disagreement magnitude.

## Notebooks (marimo)

Ship under `notebooks/`:

- `eval-deep-dive.py` — confusion matrices, per-topic metrics, disagreement clusters.
- `topic-coverage.py` — topic coverage across corpus; gaps.
- `cluster-inspection.py` (v2).

Not a deploy target.

## Web UI

Not v1. Revisit only if:

- Batching multimodal review becomes painful in TUI (v2+).
- Remote access becomes a real need.

When built, an HTTP frontend layered on the existing FastAPI backend. HTMX or the simplest SPA that works.

## Config

`config.yaml` at repo root, Pydantic-validated on load:

```yaml
service:
  http_port: 3200
  manifest_path: ./service.json
  registry_url: null   # if set, register on startup

pipeline:
  embedding:
    model: bge-large-en
    version: "1.5"
  topic:
    model: qwen2.5-7b-instruct
    quantization: q4_k_m
    confidence_threshold: 0.55
    retrieval_top_k: 5
  tags:
    model: qwen2.5-7b-instruct
    max_tags: 6
  sentiment_intent:
    bundled_with: tags    # one LLM call in v1

ingest:
  source: x-sync-outbox
  scraper_db_path: ../x-bookmarks-scraper/bookmarks.db
  poll_interval_ms: 1000

audit:
  phoenix:
    enabled: true
    local_url: http://localhost:6006
  runs_retention_days: 90

storage:
  sqlite_path: ./data/classifier.db
  sqlite_vec: enabled
  duckdb_attach: true

jobs:
  backfill:
    batch_size: 200
  eval:
    holdout_ratio: 0.2
  dspy_compile:
    max_rounds: 5
```

## Phased implementation

### Phase 0 — Skeleton (days, not weeks)

- Sibling repo initialized. `uv`, `ruff`, `pyright --strict`, Pydantic v2, Typer, Textual, FastAPI, OTel, Phoenix SDK, Alembic.
- `service.json` (manifest) written and validated against dada.stream spec.
- Manifest loader + validator; `classifier manifest` prints it; `classifier status` prints summary.
- SQLite + sqlite-vec + DuckDB wiring; Alembic baseline.
- Health endpoint live.

**Exit**: `classifier status` prints manifest + empty-DB stats. `classifier manifest` emits JSON. HTTP `/health` returns green. No classification yet.

### Phase 1 — Envelope ingest + embedding

- Scraper outbox poll consumer.
- Tweet-record → content envelope mapper (classifier-owned Python, referencing the scraper's live schema for column names; the scraper's TypeScript contract files are not authoritative).
- `EnvelopeRepo` + `EmbeddingRepo`.
- Embed every consumed envelope with the default dense model (`bge-large-en@1.5` per ADR 017); store vectors.
- Audit layer plumbing: OTel, Phoenix, `runs` table, structured logs, correlation/causation threaded.
- `classifier serve` running end-to-end with embed-only.

**Exit**: Mini runs `classifier serve`. Every new bookmark → envelope → embedding → traced.

### Phase 2 — Facet heads + event emission

- Topic head: retrieval + LLM-pick. Prototype embedding job. `topics.yaml` seeded with ~75 topics + subtopics.
- Subtopic head (LLM, conditional on topic).
- Tags head (LLM + structured output).
- Sentiment + intent: bundled LLM call. Proposed dada.stream ADR drafted.
- Reconcile, enrich envelope classification section, emit `content-classified` (or `-low-confidence`).
- Event outbox with dispatched_at + retries.
- HTTP commands `classify-content`, `reclassify-content`, `get-classification`, `list-topics`, `get-low-confidence` implemented.

**Exit**: New bookmarks produce full classifications. `content-classified` events visible in outbox. HTTP API exercisable via `curl` and via `classifier classify <id>`.

### Phase 3 — Inbox + labels

- Textual inbox reading from `get-low-confidence` HTTP endpoint + local DB.
- `accept-classification` and `edit-classification` HTTP commands.
- `labels` table + repo.
- Backfill job over existing bookmarks.
- Eval split enforced on label write.

**Exit**: Seed labels accumulating via normal review flow.

### Phase 4 — Eval harness + nightly jobs

- Held-out eval, per-facet metrics, baseline diff.
- `run-eval` HTTP command.
- launchd plists for nightly eval + backfill on Mini.
- DSPy scaffolding: compilable programs for topic and tags heads. First manual compile pass.
- Phoenix trace links surfaced in `classifier trace`.

**Exit**: Overnight jobs run on Mini. Eval metrics reported. DSPy compile is a runnable command.

### Phase 5 — dada.stream integration and extraction (ongoing)

- Propose `infrastructure/observability.md` as a dada.stream ADR + spec, referencing this implementation.
- Propose `sentiment` and `intent` ADR + spec additions.
- When a real event bus exists: swap the outbox poll for a bus subscriber; same event shapes.
- When a content store exists: read envelopes from it rather than from the scraper DB.
- When the scraper adopts dada.stream contracts: delete the v1 envelope mapper.

### v2 and beyond

- Cluster discovery (HDBSCAN over embeddings, promotion flow).
- Agent harness loop (reads traces, proposes topic/prompt changes, opens PRs, gated by eval-delta).
- Multimodal: image embeddings (SigLIP2 or DINOv3).
- Small-encoder fine-tuning (LoRA) for facets where the labeled set is rich enough.
- Web UI if TUI+notebooks prove insufficient.
- Quality scoring as a distinct facet.
- Postgres migration if/when triggers hit.

## Open decisions

Recorded so they don't rot:

- **Service name**: resolved — `prism`. See "Repository" section.
- **Small LLM for the Mini's hot path**: Qwen 2.5 7B vs Llama 3.1 8B vs Gemma 2 9B — decide after Phase 1 benchmarks on a sample.
- **Embedder A/B**: `bge-large-en@1.5` baseline (per ADR 017, after fastembed 0.8 dropped BGE-M3); evaluate a multilingual + long-context replacement, jina-v3, and Qwen3-Embedding-4B once eval set has ≥100 items.
- **Intent taxonomy starter set**: `{learn, reference, entertain, act, unclear}` — refine during Phase 3 inbox usage.
- **Sentiment+intent as bundled LLM call vs separate heads**: bundled in v1 for latency; split when labels exist.
- **Scraper outbox read mechanism**: read-only SQLite attach (simple, cross-process-safe with WAL) vs HTTP endpoint on the scraper (cleaner). Direct attach first.
- **Registry URL**: null in v1 (no registry yet); wire up when one exists.
- **Prometheus metrics in v1**: skip unless a real dashboard exists.
- **`alembic.ini` gitignore posture**: currently committed — purely conventional (script_location, file template, logging) with `sqlalchemy.url` deliberately blank, since `migrations/env.py` resolves the runtime URL from `config.yaml` (which is gitignored). Revisit if operators ever need env-specific Alembic tuning (custom logging handlers, alternate file templates): split into committed `alembic.example.ini` + gitignored `alembic.ini`, mirroring the `config.example.yaml` / `config.yaml` pattern. Not needed while the file stays identical across environments.

## Success criteria for v1

- Every new X bookmark produces a dada.stream-conformant `content-classified` envelope within 5 seconds p95.
- Every LLM call has a Phoenix trace and a row in `runs`, correlated by `correlationId`.
- Inbox review flow sustains 20+ labels/day comfortably.
- Eval harness runs nightly and reports per-facet metrics trending.
- Service manifest passes dada.stream's spec validation and is loaded by at least one consumer (the operator CLI + a test simulating an intelligence-layer read).
- Contract diffs appear as reviewable JSON Schema changes in PRs when Pydantic models change.
- No dead manifest fields — every field has a live reader.
- Proposed dada.stream ADRs (sentiment/intent, observability spec) drafted and open for review.

## References

- `~/projects/dada.stream/README.md` — ecosystem overview.
- `~/projects/dada.stream/platform/architecture/system-overview.md` — full layered architecture.
- `~/projects/dada.stream/platform/contracts/service-manifest.md` — manifest spec + classifier sketch (L411–505).
- `~/projects/dada.stream/platform/contracts/event-types.md` — event envelope + content-pipeline events.
- `~/projects/dada.stream/platform/contracts/api-conventions.md` — HTTP conventions.
- `~/projects/dada.stream/platform/domains/content-model.md` — content envelope spec.
- `~/projects/dada.stream/platform/domains/topic-model.md` — topics-as-lenses model.
- `~/projects/dada.stream/platform/intelligence/notes.md` — intelligence layer vision.
- `/Users/abendy/projects/x-bookmarks-scraper/.project/plans/2026-02-13-foundation-architecture-core-runtimes-capabilities.md` — earlier ecosystem plan on which the dada.stream ecosystem is based.
- `docs/adr/024-x-datasource-service-boundary.md` — scraper's (pre-dada.stream) event envelope; conformance gap to close separately.
- `~/projects/x-bookmarks-scraper/.project/tasks/2026-03-07-carmack-code-review.md` — dead-contracts lesson.
- `~/projects/x-bookmarks-scraper/.project/tasks/2026-03-07-carmack-project-review.md` — broader architecture review.
- `~/projects/x-bookmarks-scraper/src/lib/db/schema.ts` — the scraper's live table schema; reference for column names the classifier mapper queries. (The `src/contracts/` directory in the scraper is aspirational scaffolding per the Carmack reviews; not a reference.)
