# Post-Phase-2 structural review

Date: 2026-04-25
Scope: components/classifier/ at commit 5f8ee8bb347c7e1d1559bc95c8f4eeb4661db64d
Files scanned: 28 source, 28 test, 10 migration

## Summary

Phase 2 landed with the right high-level boundaries still visible: raw sqlite3 remains confined to repos/application code per ADR 001, the scraper bridge is still named and bounded per ADR 006, and the serve-time lazy imports for fastembed/Ollama mostly follow ADR 015. The main structural pressure is concentrated in `ingest.py`, where polling, embedding, topic retrieval, LLM pick, event emission, queue retry policy, and stats aggregation now meet in one 642-line module. The sharpest risks are not style issues: duplicated discriminator definitions, public outbox events being used as internal causation records, inconsistent transaction ownership in `TopicPrototypeRepo`, and manifest/config defaults that can drift from runtime behavior. Tests are real-boundary oriented, which is good, but they repeat enough migrated-DB and scraper-bridge setup that a small shared test support layer would now reduce drift without weakening that discipline.

## Findings

### 1. Long files / functions / classes

**Phase 2 orchestration has outgrown a single process function** — `components/classifier/src/prism/ingest.py:255`
Severity: medium

`_process` is 152 lines, accepts 17 parameters, tracks 12 counters, owns queue retry handling, saves/refreshed envelopes, embeds bodies, conditionally classifies, and finally returns a 12-position tuple. This is no longer just linear orchestration: the positional return contract and the many collaborating repos make later changes easy to miswire. I searched for existing stats/context helpers and found only `IngestStats`; there is no parallel helper that would be duplicated by an extraction.

Suggested direction: Introduce an ingest pass context object plus an internal stats accumulator, and have `_process` return a named result instead of a positional tuple.

**Classification sub-pipeline is hidden inside ingest** — `components/classifier/src/prism/ingest.py:437`
Severity: medium

`_classify_envelope` is 188 lines and takes 15 keyword-only parameters while performing five distinct operations: synthetic event creation, topic retrieval, LLM pick, classification/event payload construction, and persistence/error marking. The function reads top-to-bottom today, but the payload-building branches duplicate domain concepts that are also represented in `ClassificationWrite`, `TopicPickResult`, and `EmittedEvent`. No existing event-builder or classification-write builder exists elsewhere, so a split would not collide with a current shared helper.

Suggested direction: Move the classification branch into a small `prism.classification_pipeline` or `prism.classify` module with explicit input context and result types; keep ingest responsible for queue iteration only.

**Dev CLI is long but not the main split candidate** — `components/classifier/src/prism/dev_cli.py:1`
Severity: low

`dev_cli.py` is 458 LOC, crossing the source-file soft trigger, and it contains several independent command bodies. The commands are cohesive as a developer surface, but the module now also carries bridge DDL, topic loading, retrieval, LLM pick, and ingest smoke behavior; navigation will get worse if Phase 3 adds inbox/label commands here. Splitting would clarify command ownership once another command family lands, rather than merely redistributing today’s code.

Suggested direction: When Phase 3 adds more commands, group dev commands by domain (`dev_ingest`, `dev_topics`, `dev_scraper_seed`) and keep the Typer registration module thin.

### 2. Argument-count / parameter growth

**Pipeline entry points pass config and collaborators as loose scalars** — `components/classifier/src/prism/ingest.py:74`
Severity: medium

`ingest_once`, `_process`, and `_classify_envelope` have 11, 17, and 15 parameters respectively. Several parameters travel together as stable groups: classifier repos, classification knobs (`confidence_threshold`, `retrieval_top_k`, `ollama_model`, `service_version`), and queue retry state (`item`, `queue_repo`, `now`). This is the same pressure visible in `serve.py:106` and `dev_cli.py:171`, where the same scalar config values are threaded into `ingest_once`.

Suggested direction: Introduce small dataclasses for pass dependencies and classification settings, built at the service/dev boundary from `Config`.

**Run finalization repeats a wide row-shape signature** — `components/classifier/src/prism/audit.py:99`
Severity: low

`RunRepo.record_success`, `RunRepo.record_error`, and `_finalize` carry 10, 11, and 12 arguments, mostly the same run-row finalization fields. ADR 010 justifies the nested exception handling in `operation()`, but it does not require every finalization field to remain a separate keyword across three method signatures. The current shape makes adding a run column touch multiple wide call sites at once.

Suggested direction: Use a private finalization dataclass for the common row fields while preserving the ADR 010 two-layer exception flow.

### 3. Duplicate definitions

**Low-confidence reason vocabulary is defined twice** — `components/classifier/src/prism/classification_repo.py:32` and `components/classifier/src/prism/topic_llm_pick.py:23`
Severity: high

Both modules define the same `LowConfidenceReason = Literal["below-threshold", "out-of-band", "llm-pick-unsure", "llm-output-malformed"]`. The LLM pick layer emits the values, the repo persists them, and `ingest.py:614` branches on one of them by string, so drift would compile but break behavior or storage interpretation. This is a genuine cross-layer domain vocabulary, not layer-local duplication.

Suggested direction: Move the alias and constants to a small shared classification types module imported by both the picker and repo.

**Vector row-key delimiter is shared through a private repo symbol** — `components/classifier/src/prism/topic_prototype_repo.py:19`
Severity: medium

`TopicPrototypeRepo` imports `_ROW_KEY_DELIMITER` from `embedding_repo.py`, despite the delimiter being a vector-key encoding decision shared by two storage tables rather than owned by one repo. This works but creates an implicit repo-to-repo private dependency: changing the embedding repo’s private constant can silently change topic prototype keys. ADR 011’s composite-key rule applies to vec0 storage generally, so the shared definition has a better home.

Suggested direction: Move the delimiter and key validation helpers into a tiny `vector_keys`/`vec_keys` module used by both repos.

### 4. Repeated code patterns

**Queue failure handling is repeated across ingest error sites** — `components/classifier/src/prism/ingest.py:292`
Severity: medium

The same `queue_repo.mark_failed(item.source_event_id, error=..., backoff_seconds=_backoff_seconds(item.attempts + 1), now=now)` shape appears at `ingest.py:292`, `302`, `333`, `374`, `498`, `514`, `549`, and `634`. The repetition is now large enough that retry policy, error prefixing, and counter increments are easy to update inconsistently. There is no existing helper for this shape; `_backoff_seconds` only covers one scalar.

Suggested direction: Add an ingest-local helper that marks an item failed with a reason prefix and computed backoff, returning the chosen retry metadata if callers need it.

**Migrated SQLite test setup repeats in many files** — `components/classifier/tests/test_audit.py:48`, `components/classifier/tests/test_classification_repo.py:32`, `components/classifier/tests/test_topic_prototype_repo.py:26`
Severity: low

The same Alembic `Config("alembic.ini")` + `command.upgrade(..., "head")` + `connect_sqlite(...)` fixture appears across repository and loader tests, with only `load_vec` changing. This does not violate the real-boundary test discipline in `.project/tasks/real-boundary-test-discipline.md`; each test can still get its own real migrated DB. The duplication risk is drift in migration setup, vec loading, and cleanup behavior.

Suggested direction: Put a fixture factory in `tests/conftest.py` that creates a fresh migrated DB per test and accepts `load_vec`.

**Scraper bridge DDL is copied between dev and integration tests** — `components/classifier/src/prism/dev_cli.py:33`, `components/classifier/tests/test_ingest.py:38`, `components/classifier/tests/test_serve_integration.py:32`
Severity: low

The fake scraper schema appears in three places, and the copies are close but independently maintained. Because ADR 006 makes scraper-schema knowledge bridge-scoped, this duplication is tolerable only while the bridge remains small; a future scraper column or table expectation could update one test path but not the others. I did not find a shared scraper fixture/helper that already owns this.

Suggested direction: Centralize test/dev scraper DDL in test support or a bridge fixture helper while keeping production scraper access read-only.

### 5. Boilerplate that hides intent

**Repo cursor/transaction ceremony dominates simple mutations** — `components/classifier/src/prism/events_outbox.py:110`
Severity: low

`EventOutboxRepo.enqueue` spends about half its body on cursor initialization, try/except rollback, commit, and cursor close around a single insert and return-row construction. Similar ceremony appears in `EnvelopeRepo.save` (`envelope_repo.py:16`), `SyncStateRepo.set` (`sync_state.py:27`), `IngestQueueRepo.observe` (`ingest_queue.py:44`), and `ClassificationRepo.upsert` (`classification_repo.py:124`). ADR 001 explains raw sqlite3, so the issue is not the lack of ORM; it is that each repo hand-rolls the same transaction skeleton.

Suggested direction: Consider a tiny internal transaction/cursor helper for write methods only, preserving raw sqlite3 and explicit SQL while making the business operation easier to see.

### 6. Module / layer health

**TopicPrototypeRepo owns transactions differently from other repos** — `components/classifier/src/prism/topic_prototype_repo.py:60`
Severity: high

`delete_for_topic` and `save_many` execute DELETE/INSERT statements without commit/rollback handling, while other mutating repos commit on success and roll back on `sqlite3.Error`. `dev_cli.load_topics_cmd` compensates by wrapping `embed_catalog` in an explicit `classifier_conn.commit()`/`rollback()` at `dev_cli.py:267`, but tests call `repo.save_many(...)` directly and only assert through the same connection. This makes transaction ownership ambiguous and can leave callers thinking prototypes are durably saved when they are still pending.

Suggested direction: Make `TopicPrototypeRepo` follow the same repo-owned transaction discipline as the other mutating repos, or explicitly document and rename it as a unit-of-work participant.

**Public manifest and runtime API have drifted apart** — `components/classifier/service.json:104` and `components/classifier/src/prism/api.py:78`
Severity: medium

`service.json` declares eight command endpoints (`/classify`, `/classify/re`, `/classifications/{envelopeId}`, `/topics`, inbox endpoints, and `/eval`), while the FastAPI runtime currently exposes only `/health`. Some of this is planned Phase 3+ surface, but the manifest reads like an active contract and is validated by `manifest.py` rather than marked as aspirational. This can mislead integration consumers before those endpoints exist.

Suggested direction: Either mark future commands as planned/non-runtime in the manifest contract, or keep `service.json` limited to endpoints actually served in this phase.

**Events outbox publishes a type the manifest says is subscribed** — `components/classifier/src/prism/ingest.py:482` and `components/classifier/service.json:120`
Severity: high

`_classify_envelope` writes a synthetic `content-ingested` row into `events_outbox` and uses its id as the causation id for classification. But `service.json` lists `content-ingested` under `subscribes`, not `publishes`, and ADR 020 frames `events_outbox` as classifier-emitted events for a future dispatcher. On retries after retrieval/LLM failure, this can create externally dispatchable `content-ingested` rows without a corresponding classification event.

Suggested direction: Keep internal causation anchors out of the public emitted-events table, or explicitly ADR/manifest-gate `content-ingested` as a classifier-published synthetic event with idempotency rules.

### 7. Magic strings / inline literals

**Event names are call-site strings rather than typed vocabulary** — `components/classifier/src/prism/ingest.py:483`
Severity: medium

The emitted event types `content-ingested`, `content-classified`, and `content-classified-low-confidence` appear inline in `ingest.py`, tests, `service.json`, and comments in `config.example.yaml`. These strings act as cross-service contract discriminators, not incidental log labels. The current `EmittedEvent.new(event_type: str, ...)` accepts any string, so a typo would be caught only by tests that happen to inspect that branch.

Suggested direction: Define event-type constants or a `Literal` alias near `events_outbox.py`, and use them from ingest and tests.

**Operation names are untyped strings across audit, ingest, and queries** — `components/classifier/src/prism/ingest.py:101`
Severity: low

`operation("ingest")`, `operation("embed")`, and `operation("classify")` are meaningful audit discriminators, and tests/dev queries look them up as strings (`test_serve_integration.py:134`, `dev_cli.py:451`). The string-at-call-site form is readable, but the names are now storage keys and observability dimensions. This is a smaller risk than event names because the vocabulary is internal.

Suggested direction: Use module constants for operation names once Phase 3 adds more audited operations.

### 8. Type aliases / discriminator integrity

**ClassificationWrite encodes a discriminated union as nullable fields** — `components/classifier/src/prism/classification_repo.py:40`
Severity: medium

`ClassificationWrite` uses `confidence`, `topic_assignments`, and `low_confidence_reason` combinations to represent two branches, then `_validate_write` enforces the invariant at runtime. The module docstring explicitly calls out the discriminator, and tests cover invalid combinations, so this is controlled today. The risk is structural: callers can construct invalid states freely until repo write time, and `ingest.py` has to mirror the invariant manually when building the confident vs. low-confidence branches.

Suggested direction: Represent confident and low-confidence writes as two dataclasses or a discriminated Pydantic union, with the repo accepting the union.

**TopicPickResult has an implicit result-state invariant** — `components/classifier/src/prism/topic_llm_pick.py:51`
Severity: medium

`TopicPickResult` combines `pick`, `chosen_topic_id`, and `low_confidence_reason` as nullable fields. Valid states are implicit: confident picks require `pick` and `chosen_topic_id` with no reason; low-confidence paths require a reason and may or may not have a pick. `ingest.py:625` has a defensive `RuntimeError` for an impossible combination, which is a sign the type does not fully encode its own states.

Suggested direction: Split picker results into explicit `TopicPicked` and `TopicLowConfidence` variants that share the low-confidence reason alias.

### 9. Test structure

**Phase 2 classification tests repeat the same setup cluster** — `components/classifier/tests/test_ingest.py:686`
Severity: low

The classification section of `test_ingest.py` repeatedly seeds a bookmark, user, tweet, topic prototype, embedder, LLM client, and the same `confidence_threshold`/`retrieval_top_k`/`ollama_model`/`service_version` arguments. Some repetition is healthy because these are integration-style tests, but the common setup is now more prominent than the behavior under test in several cases. The local helpers `_seed_topic_prototype`, `_pick_payload`, `_StubBackend`, and `_StubLlmClient` are a good start; the next reusable unit is the full "classifiable tweet" scenario.

Suggested direction: Add a local scenario helper in `test_ingest.py` that seeds a classifiable tweet and runs ingest with overridable LLM payload/settings.

**Large test files now mix old ingest coverage with Phase 2 classify coverage** — `components/classifier/tests/test_ingest.py:1`
Severity: low

`test_ingest.py` is 1,091 LOC, more than double the test-file soft trigger, and now covers polling, bridge refresh, logging/tracing, embedding, classification persistence, low-confidence events, and retry behavior. The file still has useful local helpers, but navigation is becoming the cost: failures in unrelated ingest phases land in the same long file. Splitting by pipeline phase would clarify test ownership without hiding setup behind mocks.

Suggested direction: Split into real-boundary files such as `test_ingest_polling.py`, `test_ingest_embedding.py`, and `test_ingest_classification.py`, sharing only fixture helpers.

### 10. Comment / docstring quality

**Phase/slice comments have started to leak into stable modules** — `components/classifier/src/prism/events_outbox.py:4`
Severity: low

Several comments describe implementation slices or future work rather than durable system behavior: `events_outbox.py:4` references a "future dispatcher (separate slice)", `classification_repo.py:8` says envelope composition is "in a later slice", and `topic_retrieval.py:5` says the LLM slice consumes candidates downstream. These are not harmful, but they age faster than ADR references and can make stable modules read like project notes. Comments that cite accepted ADRs are more durable and should stay.

Suggested direction: Convert slice/phase comments into behavior-oriented wording, keeping ADR references where the rationale matters.

### 11. Configuration coupling

**LLM defaults are duplicated between config and ingest** — `components/classifier/src/prism/config.py:37` and `components/classifier/src/prism/ingest.py:80`
Severity: medium

`TopicConfig.ollama_model` defaults to `qwen2.5:7b-instruct-q4_K_M`, `config.example.yaml:41` repeats it, and `ingest_once` has the same default at `ingest.py:82`. The confidence threshold default is similarly present in `ingest_once` and examples/service manifest even though normal serve/dev paths pass config values explicitly. This creates a direct-call path where runtime behavior can drift from configured behavior.

Suggested direction: Keep operational defaults in `Config` only; require `ingest_once` callers to pass a classification settings object when classification is enabled.

**Classification enablement affects serve, dev CLI, and ingest through separate checks** — `components/classifier/src/prism/config.py:61`
Severity: low

`classification_enabled` controls whether `serve.py:170` constructs an Ollama client, whether `dev_cli.py:160` constructs one, and indirectly whether `ingest_once` classifies by receiving `llm_client is not None`. The flag’s behavior is still simple, but its effect is already spread across two boot paths plus the ingest sentinel. A future second classification backend or partial facet enablement would amplify this coupling.

Suggested direction: Build a single classifier dependency/settings object at boot time and pass `None` or that object to ingest, rather than scattering flag checks.

### 12. Anything else

**Classification persistence return value is constructed, not read as canonical state** — `components/classifier/src/prism/classification_repo.py:176`
Severity: low

`ClassificationRepo.upsert` returns a `StoredClassification`, but it builds that object from the input write after committing rather than reading back from the tables. That mostly matches reality today, but ADR 014’s canonical-post-mutation convention is strongest when callers receive the actual stored state, especially after `ON CONFLICT` updates and JSON serialization. Tests compare the return to a manually fetched value, which helps, but the method itself does not enforce that relationship.

Suggested direction: Either fetch the stored classification after commit before returning, or document that the method returns the canonicalized input because no DB-side defaults/triggers participate.

## Out of scope / not flagged

- Raw sqlite3 and explicit cursor closing are not flagged as architectural violations; ADR 001 accepts that baseline.
- `audit.operation()` is long, but its nested two-layer exception handling matches ADR 010 closely; I flagged only the parameter-width pressure around finalization.
- `serve.lifespan()` is long but mostly linear resource orchestration, and its fastembed/Ollama imports follow ADR 015.
- `ScraperMapper.fetch_envelope()` is bridge-specific and about 64 lines, but ADR 006 explicitly bounds scraper schema/mapping knowledge there.
- `api._check_health()` is just over the function soft trigger, but it is linear probe code; splitting it today would mostly add indirection.
- The known config-driven embedder-selection drift around `MODEL_VERSION` is already recorded in `.project/README.md`, so I did not re-flag it as new.
- Per-test real database creation is not itself a problem; the finding is about repeated fixture mechanics, not about replacing real boundaries with mocks.
