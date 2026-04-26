# Phase 1 report: prism classifier

Prepared: 2026-04-24. Intended reader: a new session agent picking
up the work with no prior conversation context. This document is
the orientation pass; the durable sources of truth it points to
are what you actually build against.

## TL;DR — can a new agent continue?

**Yes.** Phase 1 closed at commit `e090553` (the `prism dev`
smoke surface). The code is at a clean working tree with 198
passing unit tests + 2 passing integration tests (run via
`uv run pytest -m integration`, deselected from `just check`).
The plan doc, 17 ADRs, `.project/README.md` Open Items tables,
and recent commit messages collectively carry enough context
that a fresh agent can pick up Phase 2 without needing this
conversation.

The Phase 1 exit criterion — "every new bookmark → envelope →
embedding → traced" — has been verified end-to-end against the
real default dense model (`BAAI/bge-large-en-v1.5`, 1024-d,
~1.2 GB on first download via fastembed; see ADR 017 for the
swap from BGE-M3 after fastembed 0.8 dropped it). Both
`tests/test_embedding_integration.py` and
`tests/test_serve_integration.py` exercise the real model.

## Project identity

**Prism** is a Python classification service at
`/Users/abendy/projects/dada.stream/components/classifier/`. It
consumes content envelopes (starting with bookmarks from the X
scraper at `~/projects/x-bookmarks-scraper/`) and will produce
multi-faceted classifications: topics, subtopics, freeform tags,
sentiment, intent.

Package name: `prism` (the directory is still `classifier/`, a
historical artifact; do not rename). Python ≥ 3.12, uv for deps,
ruff + basedpyright (standard mode) + pytest. Storage is SQLite
(WAL) with sqlite-vec for embeddings and DuckDB attached
read-only for analytics. OpenTelemetry spans to Phoenix via OTLP
HTTP. Alembic from day one (no runtime `ALTER TABLE`).

The service is the first dada.stream component built to conform
to dada.stream contracts from the outset rather than retrofit.
It doubles as a reference implementation of the
audit/observability and contract-discipline patterns.

## State of the code

### Git history

Commits on `develop` (newest first):

```text
e090553 feat(cli): add prism dev surface for smoke and iteration
0b191ef fix(embedding): swap default model to bge-large-en-v1.5
6a95139 docs: document venv activation and BGE-M3 weight setup
8e071e4 feat(serve): run ingest loop and tracing behind FastAPI lifespan
7f15176 feat(ingest): embed envelope bodies when embedder is supplied
c7a4177 feat(embedding): add vec0 storage and fastembed-backed Embedder
b2b1d62 feat(audit): add OTel spans, OTLP exporter, and canonical log shape
4340c4b feat(audit): persist run records and structured logs
71fcecf feat(ingest): compose pipeline with bookmark-first filter and refresh
5bcc87d feat(ingest): add ingest queue and scraper outbox reader
3485d25 feat(ingest): map scraper rows to ContentEnvelope via ScraperMapper
44ceaad feat(envelope): persist envelopes in SQLite via EnvelopeRepo
a1c7733 feat(envelope): model the content envelope in Pydantic
02b7e15 refactor: rename Python package to prism
8078a8f feat(api): add FastAPI /health endpoint and serve command
5177f80 feat(storage): wire SQLite, sqlite-vec, DuckDB, and Alembic
f137fb5 feat(config): add config loader with example-template pattern
a48278e feat(manifest): model the service manifest in Pydantic
0a4cd28 chore(scaffold): initialize classifier Python package
e926b37 root
```

Commit messages are tight and focused on the work, per the
`commit` skill convention (no slice/phase/plan references in
message bodies). ADR-introducing commits name the ADR numbers
in the trailing paragraph.

### Module inventory

19 source modules under `src/prism/`:

+ **Core domain:** `envelope.py`, `envelope_repo.py`,
  `embedding.py`, `embedding_repo.py`, `manifest.py`.
+ **Storage:** `db.py` (`connect_sqlite`, `attach_duckdb`,
  `migration_head`); Alembic at `migrations/env.py`.
+ **Audit / observability:** `audit.py` (`RunRepo`,
  `operation()`, `OperationContext`), `tracing.py`
  (`configure_tracing`, `install_in_memory_exporter`),
  `logs.py` (`log_event`).
+ **Ingest pipeline:** `ingest.py` (the orchestrator),
  `ingest_queue.py`, `sync_state.py`, `scraper_outbox.py`,
  `scraper_mapper.py` (bridge components — see ADR 006).
+ **HTTP / CLI:** `api.py` (FastAPI app + `/health`),
  `serve.py` (lifespan + background ingest loop), `cli.py`
  (Typer: `manifest`, `status`, `schema`, `config`, `db`,
  `serve`, `dev` subgroup), `dev_cli.py` (`dev` subgroup with
  `seed-scraper`, `ingest-once`, `embed`).
+ **Config:** `config.py` (Pydantic config schema + loader).

21 test files, 7 Alembic migrations (baseline,
content_envelopes, content_envelopes_source_id,
ingest_queue, sync_state, runs, embeddings — head is
`8a27a28168d0`).

### Quality gates

`just check` = `ruff check .` + `basedpyright` + `pytest`.
Current state: ruff clean, basedpyright 0 errors / 0 warnings
under `standard` mode (not strict), 198 passing unit tests,
2 passing integration tests (deselected from `just check` by
`addopts = -m 'not integration'` in `pyproject.toml`; run via
`uv run pytest -m integration`). No `# type: ignore`, no
`# noqa`, no inline ruff disables anywhere in the tree —
this is a standing rule every slice prompt reiterates.

## Canonical reading order for a new agent

Read these in order to reach working context. Skim first; come
back for detail as needed.

1. **`.project/plans/2026-04-18-classification-service-foundation.md`**
   (775 lines, the plan). Load-bearing. Defines phases, event
   shapes, repo surface, roadmap. Phase 1 exits at the current
   commit; Phase 2 starts with topic-head retrieval.
2. **`docs/adr/README.md`** — the 17-ADR index. Read it whole;
   it's a table that maps number → title → status. ADRs 001–017
   are all Accepted.
3. **`.project/README.md`** — the workspace index. The "Open
   Items" section (Deferred work, Known risks, Candidate ADRs,
   Process insights) carries everything that isn't yet in code
   but is worth knowing.
4. **`README.md`** (root) — operator-facing. Prerequisites,
   `just install` / `source .venv/bin/activate`, default-dense-
   model cache one-liner (`bge-large-en@1.5` per ADR 017), the
   `prism dev` development-commands table, Phoenix docker
   invocation, integration-test invocation.
5. **`contracts/INTEGRATION.md`** + **`service.json`** — the
   external contract surface the manifest conforms to.

After that, navigate code by ADR reference — every ADR has a
"References" section pointing at the files and symbols it
pins.

## Architecture at a glance (anchored to ADRs)

**Storage:** raw `sqlite3` in application code, SQLAlchemy only
in Alembic (ADR 001). Pydantic v2 with `extra="forbid"`,
`strict=True`, `populate_by_name` NOT set (ADR 002). Per-field
relaxation via `Annotated[...]` carve-outs (ADR 003).

**Scraper bridge** (ADR 006 — the umbrella). The classifier
reads the scraper's SQLite directly (outbox + tweets + users +
media tables), extracts only `entityType` + `entityId` from
outbox payloads (ADR 004), and advances its poll cursor via
`OutboxBatch.next_cursor` (ADR 005 — avoids the all-malformed-
window starvation). The bridge retires when the scraper emits
dada.stream-conformant `content-ingested` events; ADR 006
enumerates exactly which files/rules go away at that point.

**Envelope lifecycle** (ADR 007). `EnvelopeRepo.save` on first
observation; `EnvelopeRepo.refresh_by_source_id` on
later events for the same `source_id`. Refresh preserves
`identity.id` and `system.created_at`; everything else takes
the new envelope's values. Refresh returns the merged envelope
(ADR 014) so callers thread the preserved id into downstream
keys (embeddings, runs rows).

**Audit / observability:**

+ `operation()` context manager wraps every unit of work
  (ADR 010 — two-layer exception handling with identity-based
  dedup on the inner/outer raise). Writes a `pending` run row
  on enter, transitions to `success`/`error` on exit, opens and
  closes an OTel span around the same boundary.
+ Terminal log events (`ingest.done`, `embed.done`, etc.) fire
  *after* the `with` block closes, with `trace_id` captured
  inside the block and passed explicitly (ADR 009) — prevents
  a success-shaped log line if `record_success` itself raises.
+ `configure_tracing` swaps OTel providers by writing the
  private `_TRACER_PROVIDER` global directly; the public setter
  is one-shot (ADR 008). Previous provider is shut down to
  prevent batch-worker thread leaks.

**Embedding** (ADRs 011, 012, 013, 014).

+ `vec0` virtual table with a synthetic TEXT PK encoding
  `(envelope_id, model_version)` via ASCII 0x1F delimiter; the
  two components are also stored as auxiliary `+` columns for
  filter queries (ADR 011). Input validation rejects 0x1F in
  either component.
+ Upsert via DELETE-then-INSERT inside a single transaction —
  `vec0` rejects `INSERT OR REPLACE` (ADR 012).
+ Per-envelope operation spans pass three kwargs
  (`envelope_id`, `correlation_id=envelope_id`,
  `causation_id=parent_run_id`) — plan's event contract
  distinguishes correlation (cross-stage content thread) from
  causation (immediate trigger) (ADR 013).
+ Mutating repo methods (`refresh_by_source_id`) return the
  canonical post-mutation state so downstream keys anchor to
  the stable id (ADR 014).

**Serve orchestration** (ADRs 015, 016). `prism serve` attaches
a FastAPI lifespan that:

+ Calls `configure_tracing(cfg.audit)` once.
+ Opens `scraper_conn` (load_vec=False) + `classifier_conn`
  (load_vec=cfg.storage.sqlite_vec).
+ Constructs an `Embedder` via `create_default_embedder()` when
  `cfg.ingest.embedding_enabled` is true — import lazy inside
  the lifespan body so `import prism.api` stays cheap for tests
  (ADR 015).
+ Launches an `asyncio.Task` running `ingest_loop`; each tick
  offloads `ingest_once` via `asyncio.to_thread`, sleeps
  `poll_interval_ms`, doubles backoff to 30s on pass-level
  error, races `shutdown.wait()` against the sleep for prompt
  cooperative shutdown.
+ The loop owns no span itself (ADR 016) — `ingest_once` and
  the embed phase own their own spans; a loop-lifetime span
  would never close in Phoenix.
+ Teardown has a two-level `try/finally`: outer closes
  connections on any startup failure; inner ensures close +
  `serve.stop` log run even if the task raises on shutdown.

**SQLite thread boundary.** `connect_sqlite` passes
`check_same_thread=False` so connections can move across the
`asyncio.to_thread` boundary under the single-task-loop +
single-worker-thread invariant. This came out of slice 8's
HIGH review finding and is pinned by
`test_ingest_loop_connection_usable_on_worker_thread`. See
`.project/tasks/real-boundary-test-discipline.md` for the
discipline this enforces.

## Workflow conventions

### Slice prompt → worker → review → commit → ADR

Work is sliced via a prompt at `.project/prompt.md`. Each
prompt has: Context (prior work + vetted inputs + what-this-is
/ what-this-is-NOT), Problem, Fix (file-by-file), Scope (what
to modify / not modify), Acceptance cases, What NOT to do,
Quality bar, References. A worker agent implements against
the prompt, writes `.project/worker-handoff.md` on completion.
A reviewer agent critiques against the plan + ADRs + prompt,
writes `.project/review-handoff.md`; rounds iterate until the
reviewer signs off.

After sign-off:

1. The user (or this assistant) commits the work using the
   `commit` skill (conventional commits, README/ADR/gitignore
   checks, concise body, no workflow scaffolding in the
   message).
2. `/adr slice` evaluates the commit for ADR candidates. The
   bar is "named rejected alternative, cross-cutting, a future
   agent would re-propose without the ADR." High-priority
   candidates get promoted via the `adr` skill and amended
   into the same commit (ADRs ship with the code that embodies
   them).
3. Residual items — deferred work, accepted risks, unpromoted
   ADR candidates, process insights — land in
   `.project/README.md` Open Items tables (durable across
   session clears) or task files under `.project/tasks/`.

Both `commit` and `adr` skills are defined at
`~/projects/dada.stream/.claude/skills/` and are invoked by
slash command.

### Rules worth restating

+ `.project/` files are **not committed** (see the folder's own
  README). They're workspace artifacts; the durable
  architectural record lives in `docs/adr/`, git history, and
  source.
+ Worker slice prompts say `.project/` is read-only to the
  worker. The user (and this assistant when not in worker mode)
  write there freely.
+ Commit messages describe the work, not the process. No
  "slice N", "phase M", "per the plan", "per review" — the
  commit log is durable; the workflow scaffolding rots. See
  `memory/feedback_commit_messages.md` in the assistant's
  cross-conversation memory.
+ Amend commits are OK for immediate follow-ups (e.g., adding
  the just-written ADRs to the commit of the code they
  describe). The `commit` skill's body explicitly permits
  this.
+ Quality bar is absolute: `just check` clean, zero
  basedpyright warnings, no inline ignores.

## Open items → `.project/README.md`

Rather than duplicate here, the Open Items tables in
`.project/README.md` are canonical:

+ **Deferred work** (8 items) — config-driven embedder
  selection (→ `.project/tasks/multi-model-embedding-config.md`),
  content-hash re-embed skip, embed batching, public
  `get_by_source_id`, `test_serve.py` split,
  `prism trace` CLI, empty-body fallback, first real-model
  integration run.
+ **Known risks accepted for now** (6 items) — `TestClient`
  lifespan behavior, `check_same_thread=False` invariant,
  provider swap on every lifespan enter, default thread-pool
  size, error-log rate limiting, integration-test deadline.
+ **Candidate ADRs (low priority)** — optional-dep injection
  shape, `asyncio.wait_for(shutdown)` idiom. Neither at
  promote-now bar.
+ **Cross-slice process insights** — real-boundary test
  discipline (→ `.project/tasks/real-boundary-test-discipline.md`),
  per-step raise tests on multi-step teardown,
  prompt/plan/ADR contradiction escalation, scope-vs-acceptance
  tiebreaking.

## Next slice: Phase 2 entry

Per the plan (`§Phase 2 — Facet heads`), the first slice is the
**topic head retrieval + LLM-pick**. Shape:

+ Load ~75 topic prototypes from `topics.yaml` (not yet
  authored). Each topic has a name, optional subtopic list, a
  seed description for the prototype embedding.
+ Embed prototypes once at startup (or on-demand with caching);
  store in sqlite-vec as a second `vec0` table or via the
  existing `embeddings` table with a model-version tag.
+ Per envelope: cosine-similarity retrieval over prototypes,
  take top-k (config: `retrieval_top_k`, currently 5),
  feed candidates + body excerpt to the LLM for final pick.
+ Below `confidence_threshold` (currently 0.55) → emit
  `content-classified-low-confidence` event; above → emit
  `content-classified` event. Event emission lands alongside
  because Phase 2 also introduces the event outbox.

Decisions the next slice will need to make:

+ Where topic prototypes live (migration-managed rows vs. YAML
  loaded at startup). Plan implies YAML versioned; migration-
  managed would be easier to audit in `runs`.
+ Which LLM client and how lazy it loads. The lazy-import ADR
  (015) applies.
+ How `model_version` threads through — this is the first real
  test of the "hand-maintained `MODEL_VERSION` constant" risk
  (see `.project/tasks/multi-model-embedding-config.md`).
+ Event outbox table shape and emission ordering vs. run row
  commit.

None of the above is in code yet. The plan and topics-model
spec at `~/projects/dada.stream/platform/domains/topic-model.md`
are the canonical inputs.

## Known dragons

Things a new agent will hit that aren't obvious from docs:

1. **The scraper is a separate repo at
   `~/projects/x-bookmarks-scraper/`.** Its SQLite DB lives at
   `../x-bookmarks-scraper/bookmarks.db` (relative to this
   component). The classifier reads it directly; it must not
   write. Scraper state column names are the authoritative
   reference — not the scraper's `src/contracts/` directory
   (per the Carmack review at `~/projects/x-bookmarks-scraper/.project/tasks/2026-03-07-carmack-code-review.md`).
2. **Pydantic model config explicitly does NOT set
   `populate_by_name=True`** (ADR 002). The wire format accepts
   only the aliased camelCase key. A helpful-looking PR that
   flips this is wrong and will break contract conformance.
3. **Basedpyright is `standard`, not `strict`.** Some residual
   `reportUnknownMemberType` / `reportUnknownVariableType` are
   set to `"none"` in `pyproject.toml` because third-party
   typing holes aren't worth fighting. Don't "upgrade" to
   strict without a plan for the typing-hole triage.
4. **Many vec0 quirks are pinned in ADRs 011 + 012.** The
   synthetic composite key and DELETE-then-INSERT upsert look
   over-engineered if you don't know why — the ADRs explain
   what sqlite-vec refuses to do.
5. **`refresh_by_source_id` returns the merged envelope**
   (ADR 014). The input `envelope.identity.id` is the mapper's
   transient UUID; the preserved `identity.id` lives in the
   return value. Slice 7b almost shipped a bug around this.
6. **The serve loop intentionally owns no span** (ADR 016).
   Don't "helpfully" wrap it in one.
7. **Integration tests are deselected by default.** `addopts =
   -m 'not integration'` in `pyproject.toml`. Run explicitly
   with `uv run pytest -m integration`. The default dense model
   (`BAAI/bge-large-en-v1.5`) is ~1.2 GB on first download via
   fastembed.
8. **Phoenix is optional** — set `audit.phoenix.enabled: false`
   in `config.yaml` to skip OTLP export. `runs` rows still
   populate; only the Phoenix push is suppressed. README
   documents the `docker run` one-liner.
9. **`config.yaml` is gitignored**; `config.example.yaml` is
   committed. Operators copy and edit. If you change a default
   every env should pick up, edit the `.example.yaml` — never
   commit `config.yaml` itself.

## Gaps this handoff doesn't cover

+ **Phase 2+ detail beyond the plan.** The plan lists facet
  heads, inbox, eval, jobs at a conceptual level. The concrete
  schemas for `classifications`, `labels`, `events_outbox`,
  `review_events` are sketched in the plan but not realized in
  code or migrations. Expect slice prompts to flesh these out.
+ **dada.stream spec docs beyond `contracts/INTEGRATION.md`.**
  The full contract set lives at
  `~/projects/dada.stream/platform/` and
  `~/projects/dada.stream/contracts/`. This repo's conformance
  today is manifest + envelope model; classification events
  are not yet wired.
+ **Operational runbooks.** No production deployment yet. The
  plan references launchd plists on a Mac Mini as the target
  runtime for Phase 4; nothing exists in-repo.

## Verdict

A new agent should be able to read this handoff, then the plan,
then the ADR index, and be productive within a single session.
The slice-prompt template in `.project/prompt.md` (still
carrying slice 8's content at handoff time) shows the shape a
new Phase 2 slice prompt should take.

If the new agent feels blocked, the most common cause will be
re-proposing something an ADR already rejected — skim
`docs/adr/` before arguing with the existing shape.
