# Phase 2 report: prism classifier

Prepared: 2026-04-25. Intended reader: a new session agent
picking up the work with no prior conversation context. This
document is the orientation pass; the durable sources of truth
it points to are what you actually build against.

## TL;DR — can a new agent continue?

**Yes.** Phase 2 closed at commit `3b2eb4d` (synthetic-event
internal flag + typed event vocabulary). The code is at a
clean working tree with 268 passing unit tests and 2 passing
integration tests (run via `uv run pytest -m integration`,
deselected from `just check`). The plan, 22 ADRs, the
post-Phase-2 structural review, and the README's Open Items
tables collectively carry enough context that a fresh agent
can pick up Phase 3 without needing this conversation.

The Phase 2 exit criterion — "every successfully embedded
envelope produces either a `content-classified` event
(confident pick) or a `content-classified-low-confidence`
event (one of four `LowConfidenceReason` values)" — is met
end-to-end. With `ingest.classification_enabled: true` in
config and an Ollama daemon running, `prism dev seed-scraper
&& prism dev ingest-once` produces:

- A row in `classifications` (with topic assignments on
  confident picks, with `low_confidence_reason` on
  low-confidence picks).
- One synthetic `content-ingested` row in `events_outbox`
  (marked `internal=TRUE`) plus one `content-classified` or
  `content-classified-low-confidence` row (marked
  `internal=FALSE`).
- A `runs` row per pick with correlation/causation IDs and
  trace ID.

## Project identity

Unchanged from Phase 1. Prism is a Python classification
service at
`/Users/abendy/projects/dada.stream/components/classifier/`.
Package `prism`, Python ≥ 3.12, uv + ruff + basedpyright
(standard mode) + pytest. Storage is SQLite (WAL) with
sqlite-vec for embeddings and DuckDB attached read-only for
analytics. OpenTelemetry spans to Phoenix via OTLP HTTP.
Alembic from day one.

The service is the first dada.stream-conformant component;
its audit/observability and contract-discipline patterns
double as reference implementations for downstream services.

## What Phase 2 shipped

Nine commits between Phase 1's HEAD `e090553` and Phase 2's
HEAD `3b2eb4d`:

```text
3b2eb4d feat(events): mark synthetic events internal and type the event vocabulary
afecb4c refactor(classifier): consolidate test fixtures and scrub workflow comments
5c424be refactor(classifier): split classify pipeline and lift shared types
7bab187 feat(classifier): wire pick_topic into the ingest hot path
cd1ecbb feat(classifier): add classifications and topic_assignments storage
bc3399d feat(events): add events_outbox table and EventOutboxRepo seam
e09baf0 feat(topics): add Ollama-backed LLM topic pick
2f2e813 feat(topics): retrieve top-k candidates by cosine over prototypes
51c1aab feat(topics): add curated catalog and prototype embedding job
```

The arc reads top-to-bottom as: catalog → retrieval → LLM
pick → storage seams → wire-up → structural cleanup → final
correctness fix on event semantics.

Five new ADRs landed during Phase 2 (018–022):

- **018** — Python full-scan cosine for topic-prototype
  retrieval (vs. sqlite-vec KNN; the cost of full scan at
  ~75-prototype scale is negligible relative to the LLM
  hop).
- **019** — Ollama as the default local LLM backend (vs.
  llama-cpp-python in-process, MLX, vLLM, OpenAI-compatible
  wrappers).
- **020** — Typed columns for the events outbox table (vs.
  JSON-blob; the dispatcher's poll path is indexable
  without JSON path extraction).
- **021** — Two-table normal form for `classifications` +
  `topic_assignments` (vs. JSON-blob; topic-id lookups are
  indexable).
- **022** — `events_outbox` internal flag + typed event
  vocabulary (the synthetic-event publish-loop concern and
  the inline event-name strings, addressed together).

## State of the code

### Module inventory

30 source modules under `src/prism/` (up from 19 at Phase 1
close):

- **Core domain:** `envelope.py`, `envelope_repo.py`,
  `embedding.py`, `embedding_repo.py`, `manifest.py`,
  `vec_keys.py` (Phase 2: shared row-key delimiter for
  sqlite-vec tables), `classification_types.py` (Phase 2:
  cross-layer `LowConfidenceReason` Literal alias).
- **Storage:** `db.py`. Alembic at `migrations/env.py`.
  11 migrations total (new in Phase 2:
  `6dffc3940f1e_topic_prototypes`,
  `3434894db396_events_outbox`,
  `4d82af96f3f8_classifications`,
  `41c92ab62f96_events_outbox_internal` — current head).
- **Audit / observability:** `audit.py`, `tracing.py`,
  `logs.py`. Unchanged in shape since Phase 1; new
  operation names (`embed`, `classify`) thread through.
- **Ingest pipeline:** `ingest.py` (orchestrator, now using
  `IngestDeps` collaborator object + named `IngestPassResult`
  dataclass), `ingest_queue.py` (also exposes `fail_queue_item`
  and `backoff_seconds` as public retry helpers),
  `sync_state.py`, `scraper_outbox.py`, `scraper_mapper.py`.
- **Classify pipeline (Phase 2):** `classify_pipeline.py`
  (the `classify_envelope` orchestrator extracted from
  ingest), `classification_repo.py` (`ClassificationWrite =
  ConfidentClassification | LowConfidenceClassification`
  discriminated union), `events_outbox.py` (with `EventType`
  Literal and `internal` flag).
- **Topic head (Phase 2):** `topic_catalog.py`,
  `topic_loader.py`, `topic_prototype_repo.py`,
  `topic_retrieval.py`, `topic_llm_pick.py` (with
  `TopicPickResult = TopicPicked | TopicLowConfidence`
  discriminated union), `llm_client.py` (the `LlmClient`
  Protocol + `OllamaClient`).
- **HTTP / CLI:** `api.py`, `serve.py` (lifespan now owns
  `httpx.Client` and `OllamaClient` when classification is
  enabled), `cli.py`, `dev_cli.py` (with new `load-topics`,
  `retrieve-topics`, `pick-topic` smoke commands and
  classification-aware `ingest-once`).
- **Config:** `config.py` (new field:
  `ingest.classification_enabled: bool = False`).

28 test files (up from 21). Notable: `tests/conftest.py`
now owns the migrated-DB `conn` and `conn_with_vec`
fixtures used by repository tests (consolidated mid-Phase-2
to remove drift across nine duplicates).

### Quality gates

`just check` clean: ruff, basedpyright 0/0 in standard
mode, 268 unit tests pass, 2 integration tests deselected
by default. No `# type: ignore`, no `# noqa`, no inline
ruff disables anywhere — the standing rule still holds.

Phase 2 doubled the source-module count and grew tests by
~70 cases without loosening the bar.

## Canonical reading order for a new agent

Read these in order.

1. **`.project/plans/2026-04-18-classification-service-foundation.md`**
   — the plan. Phase 2's facet-heads section is realized
   (topics only — subtopic, tags, sentiment, intent are
   still future work). Phase 3 (inbox + labels) starts where
   `_classify_envelope` writes its result; the plan's
   `accept-classification` / `edit-classification` /
   `get-low-confidence` HTTP commands are the next surface.
2. **`docs/adr/README.md`** — 22-ADR index. Read it whole.
   ADRs 020–022 are Phase-2-fresh and most likely to be
   re-proposed by an agent who hasn't seen them.
3. **`.project/README.md`** — the workspace index. Open
   Items tables (Deferred work, Known risks, Candidate ADRs,
   Cross-slice process insights) are durable cross-session
   state.
4. **`.project/reviews/2026-04-25-post-phase-2-structural-review.md`**
   — the review that drove the mid-Phase-2 refactor. Useful
   when something looks unfamiliar; the review's "Out of
   scope / not flagged" section explains decisions to leave
   things alone.
5. **`README.md`** (root) — operator-facing. Ollama setup
   (`ollama pull qwen2.5:7b-instruct-q4_K_M && ollama
   serve`) is the new prerequisite for classification.
6. **`contracts/INTEGRATION.md`** + **`service.json`** — the
   external contract surface.

After that, navigate code by ADR reference.

## Architecture at a glance — what changed in Phase 2

The Phase 1 architecture (raw sqlite3, Pydantic strict, vec0
storage, audit pattern, scraper bridge, serve lifespan) is
all still in place. Phase 2 added these layers on top:

**Topic head — retrieval + LLM pick** (ADRs 018, 019).

- Catalog of ~75 topics in `topics.yaml`; each topic has a
  description and exemplars. `prism dev load-topics`
  embeds and stores prototypes in a sqlite-vec table
  `topic_prototypes` (synthetic key per ADR 011, upsert via
  delete-then-insert per ADR 012).
- Per envelope: `retrieve_top_k_for_envelope` does a Python
  full-scan cosine over the prototype rows (ADR 018 — KNN
  via sqlite-vec was rejected as marginal at this scale).
- Top-k candidates feed `pick_topic` which calls
  `OllamaClient.pick` (ADR 019). Schema-violating LLM
  responses surface as a `TopicLowConfidence(reason=
  "llm-output-malformed")` outcome rather than crashing
  the run.

**Classify-on-ingest wiring.**

- The lifespan owns `httpx.Client` and `OllamaClient` when
  `cfg.ingest.classification_enabled` is true (matches the
  embed-pattern of "lifespan owns long-lived I/O resources",
  ADR 015 import discipline).
- `_process` calls `classify_envelope(...)` (in
  `prism.classify_pipeline`) after each successful embed.
  The function: mints a synthetic `content-ingested` event
  to anchor causation, retrieves top-k, runs `pick_topic`
  inside `operation("classify", ...)`, builds the
  `ClassificationWrite` + emitted event payload, calls
  `classification_repo.upsert(write)` then
  `event_outbox_repo.enqueue(emitted_event)`.
- Empty retrieval and LLM transport errors → `mark_failed`
  on the queue row; the queue's existing backoff handles
  retry. No circuit breaker.

**Storage seams** (ADRs 020, 021, 022).

- `events_outbox`: typed scalar columns + `payload` JSON +
  `internal BOOLEAN`. Synthetic causation anchors are
  written `internal=TRUE`; the pending partial index
  narrows on `WHERE dispatched_at IS NULL AND internal = 0`
  so a future dispatcher's poll naturally skips internal
  rows.
- `classifications` + `topic_assignments`: two-table normal
  form. The `ClassificationWrite` is a discriminated union
  (`ConfidentClassification | LowConfidenceClassification`)
  so the `low_confidence_reason IS NULL ⟺ confident`
  invariant lives in the type system, not in a runtime
  validator.
- `EventType` Literal in `events_outbox.py` types the wire
  event vocabulary (`content-ingested`, `content-classified`,
  `content-classified-low-confidence`).

**Discriminated unions everywhere.**

The mid-Phase-2 refactor (`5c424be`) replaced two pairs of
nullable-fields-with-runtime-validator with proper
discriminated unions:

- `TopicPickResult = TopicPicked | TopicLowConfidence`
  (was `TopicPickResult` with three nullable fields and a
  defensive `RuntimeError` for the impossible state).
- `ClassificationWrite = ConfidentClassification |
  LowConfidenceClassification` (was a single dataclass with
  `_validate_write` enforcing the invariant at runtime).

Both runtime checks are gone; basedpyright narrows in each
branch.

**Public retry helpers** (`fail_queue_item` and
`backoff_seconds` in `ingest_queue.py`). After a
near-regression where these helpers got duplicated across
`ingest.py` and `classify_pipeline.py` during the refactor,
they live in `ingest_queue.py` as the single source of
truth — both modules import from there.

## Workflow conventions

Unchanged from Phase 1. Slice prompt → worker → review →
commit → ADR. The `commit` skill enforces conventional
commit format with no slice/phase/plan references in the
body. The `adr` skill promotes candidate ADRs into the
commit that ships their implementation.

A few patterns surfaced in Phase 2 worth carrying forward:

- **Defensive coding for impossible cases is a recurring
  worker reflex.** Multiple slices proposed runtime
  guards (`RuntimeError` for an unreachable
  `TopicPickResult` branch, `_validate_write` for
  combinations the type system should prevent,
  `_require_service_version`). Each was flagged in review
  and removed in favor of stronger types or required-no-
  default kwargs.
- **Helpers extracted in one slice tend to get duplicated
  when a later slice moves code to a new module.** The
  `_fail_queue_item` and `_backoff_seconds` duplication in
  the structural refactor was the canonical example.
  Future prompts should explicitly direct: when extracting
  helpers, lift them to a leaf module both consumers can
  import from, not into the consumer-most module.
- **Single-leading-underscore on cross-module API.** The
  refactor briefly shipped `_CLASSIFY_FAILED` imported
  across modules. Convention says single-underscore is
  module-private; cross-module API drops the underscore.

These are flagged in the post-Phase-2 review (§"Anything
else") and resolved in commits `5c424be` / follow-up
rounds.

## Open items → `.project/README.md`

The README's Open Items tables are canonical:

- **Deferred work** (3 rows): config-driven embedder
  selection, body-less envelope title/summary fallback,
  runtime-independent `prism serve` (the `ingest.source`
  switch). Triggers and rationale unchanged from Phase 1.
- **Known risks accepted for now** (6 rows): same as
  Phase 1.
- **Candidate ADRs** (2 rows): optional-dependency
  injection shape (now 2-of-3 consumers; promote when
  reconcile/route adopts), `asyncio.wait_for` cooperative-
  shutdown idiom (zero new loops added in Phase 2; trigger
  unchanged).
- **Cross-slice process insights** (4 rows): unchanged.

The post-Phase-2 review surfaced a longer list of items
deferred to later passes (see review §"Out of scope / not
flagged" and the refactor plan's "Items NOT in scope"
table) — `service.json` declaring 8 endpoints when only
`/health` is live, repo cursor-ceremony refactor across
all repos, `dev_cli.py` / `test_ingest.py` splits, etc.
None blocks Phase 3.

## Phase 3 entry

Per the plan (§Phase 3 — Inbox + labels), the next slice
materializes the operator review flow on top of the
classifications + low-confidence events that Phase 2
produces:

- **Textual TUI inbox** reading from a new
  `get-low-confidence` HTTP endpoint + local DB.
- **`accept-classification` and `edit-classification` HTTP
  commands.**
- **`labels` table + repo.**
- **Backfill job** over existing bookmarks.
- **Eval split** enforced on label write (20% held out,
  deterministic via hash on envelope ID per the plan).

Decisions Phase 3 will need to make:

- `labels` table shape — the plan §Storage names it but
  doesn't pin columns. Likely mirrors classification fields
  - `accepted_run_id` / `accepted_at` / `labeler` fields.
- Inbox staleness check shape — the plan pins this
  conceptually (`active_run_id` on `ClassificationRepo` is
  the single source of truth), but the HTTP-level
  enforcement is uncoded.
- TUI architecture — Textual is named in the plan but not
  yet a dependency. First TUI slice introduces the package
  - a thin first screen; subsequent slices grow it.
- A real dispatcher for `events_outbox`. With the
  `internal` flag in place (ADR 022), a dispatcher slice
  is unblocked. Whether it lands in Phase 3 or later
  depends on whether Phase 3's needs (inbox queries) reach
  past `events_outbox` into the eventual bus.

Phase 4 (eval harness + nightly jobs) and Phase 5
(dada.stream integration / extraction) are unchanged from
the plan.

## Known dragons

Things a new agent will hit that aren't obvious from docs:

1. **The scraper is a separate repo.** Bridge is at
   `~/projects/x-bookmarks-scraper/`; classifier reads its
   SQLite DB at `../x-bookmarks-scraper/bookmarks.db`. ADR
   006 bounds the bridge.
2. **`populate_by_name=True` is NOT set on Pydantic
   models** (ADR 002). Wire format is camelCase only. A
   helpful PR that flips this is wrong.
3. **Vec0 quirks** (ADRs 011, 012). Synthetic TEXT PK +
   delete-then-insert upsert. The shared `vec_keys` module
   owns the row-key encoding; both `embedding_repo` and
   `topic_prototype_repo` use it.
4. **`refresh_by_source_id` returns the merged envelope**
   (ADR 014). The input `envelope.identity.id` is the
   mapper's transient UUID; the preserved `identity.id`
   lives in the return value.
5. **The serve loop intentionally owns no span** (ADR 016).
6. **Lazy heavyweight imports** (ADR 015). `fastembed`,
   `httpx`, `OllamaClient` import inside lifespan or
   function bodies, not at module top. New code that adds
   heavyweight deps should follow this pattern.
7. **Synthetic `content-ingested` events live in
   `events_outbox` with `internal=TRUE`** (ADR 022). A
   future dispatcher MUST filter on `internal=FALSE` (or
   use the partial pending index that already conditions
   on `internal = 0`) to avoid publishing internal
   causation anchors as if they were edge-service events.
8. **Discriminated unions are how the classifier
   represents low-confidence vs confident outcomes.** Don't
   collapse `TopicPicked | TopicLowConfidence` back into a
   single dataclass with nullable fields — the previous
   shape required a runtime validator that's been deleted.
   Same for `ConfidentClassification |
   LowConfidenceClassification`.
9. **`fail_queue_item` and `backoff_seconds` live in
   `ingest_queue.py`.** Both `ingest.py` and
   `classify_pipeline.py` import them. Don't reproduce
   them in a third module.
10. **`tests/conftest.py` owns `conn` (no vec) and
    `conn_with_vec` (vec loaded) fixtures.** Repository
    tests use these; tests that need elaborate scraper-
    bridge or lifespan setup keep their fixtures local.
11. **Ollama is required for classify on the hot path.**
    Without a running daemon at `cfg.pipeline.topic.ollama_url`,
    classify-enabled ingest fails on every envelope. Set
    `ingest.classification_enabled: false` to bypass.
12. **`config.yaml` is gitignored**; `config.example.yaml`
    is committed. New defaults go in the example.

## Gaps this handoff doesn't cover

- **Phase 3+ schemas.** `labels`, `review_events`, eval
  result tables are sketched in the plan but not in
  migrations. Phase 3 slices will flesh these out.
- **Subtopic, tags, sentiment, intent heads.** Phase 2
  shipped only the topic head. The other facet heads are
  still future work; the plan's §Phase 2 description
  promised them but the realized scope was topics only,
  per the conversational scope adjustments during Phase 2.
- **Real event bus.** The `events_outbox` is durable but
  has no consumer. The dispatcher slice is unblocked by
  ADR 022's `internal` flag but not yet scheduled.
- **dada.stream specs in `~/projects/dada.stream/platform/`
  beyond manifest + envelope conformance.** The classifier
  doesn't yet emit to a real bus or read from a real
  content store; both are downstream-component work.
- **Operational runbooks.** No production deployment yet;
  launchd plists referenced in the plan don't exist
  in-repo.

## Verdict

Phase 2 is fully landed. A new agent should be able to
read this report, then the plan, then the ADR index, and
be productive in Phase 3 within a single session. The
slice-prompt template at `.project/prompt.md` (currently
holds Phase 2's last prompt — overwrite when drafting the
next slice; do not append) shows the shape a new Phase 3
slice prompt should take.

If the new agent feels blocked, the most common cause will
be re-proposing something an ADR already rejected — skim
`docs/adr/` before arguing with the existing shape.
