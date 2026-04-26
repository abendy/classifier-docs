# Plan: Post-Phase-2 structural refactor

## Date

2026-04-25

## Why this plan exists

Phase 2 shipped a working classify-on-ingest path. Before
Phase 3 starts, address the structural issues surfaced by
the post-Phase-2 review at
`.project/reviews/2026-04-25-post-phase-2-structural-review.md`
that are cleanup-shaped (duplicates, type-discriminator
integrity, helper extraction, transaction asymmetry,
parameter-group growth). Behavior does not change; tests
remain real-boundary; ADRs are not contradicted.

## Scope

Single refactor slice. Eight items, all internal. The
worker should change file shapes and type structures;
`just check` stays green throughout (no new unit tests
required, but existing tests must continue to pass).

### Items in scope

Each row references the review item it addresses by
section and headline.

| # | Review ref | Headline | What changes |
| --- | --- | --- | --- |
| 1 | §3 "Low-confidence reason vocabulary is defined twice" | `LowConfidenceReason` Literal duplicated across `topic_llm_pick.py` and `classification_repo.py` | Lift the alias to a shared module (`prism.classification_types`); both layers import it. |
| 2 | §6 "TopicPrototypeRepo owns transactions differently from other repos" | `delete_for_topic` and `save_many` don't commit/rollback while every other mutating repo does; `dev_cli.load_topics_cmd` compensates with manual transaction wrapping | Make `TopicPrototypeRepo` mutating methods own commit/rollback like `RunRepo`/`IngestQueueRepo`/`EnvelopeRepo`/`ClassificationRepo`/`EventOutboxRepo`. Remove the manual `commit()/rollback()` in `dev_cli.load_topics_cmd`. |
| 3 | §3 "Vector row-key delimiter is shared through a private repo symbol" | `topic_prototype_repo.py` imports `_ROW_KEY_DELIMITER` from `embedding_repo.py` | Move the delimiter and key-validation helper to a small `prism.vec_keys` module imported by both repos. |
| 4 | §4 "Queue failure handling is repeated across ingest error sites" | The same `queue_repo.mark_failed(...)` shape appears 8 times in `ingest.py` | Extract a private `_fail_queue_item(item, *, queue_repo, error_prefix, exc, now)` helper inside `ingest.py`; replace the 8 call sites. |
| 5 | §11 "LLM defaults are duplicated between config and ingest" | `ingest_once` has `ollama_model="qwen2.5:7b-instruct-q4_K_M"` default duplicating `TopicConfig.ollama_model` (and `confidence_threshold` / `retrieval_top_k` defaults exist similarly) | Drop the LLM-related defaults on `ingest_once`; make them required keyword args (mirroring the `service_version` shape from the prior round). The `_run` test helper picks up the defaults via `setdefault`. |
| 6a | §8 "TopicPickResult has an implicit result-state invariant" | `TopicPickResult` uses three nullable fields whose valid combinations are implicit; `ingest.py` has a defensive `RuntimeError` for the impossible case | Split into two variants: `TopicPicked(pick: TopicPick, chosen_topic_id: str)` and `TopicLowConfidence(reason: LowConfidenceReason, pick: TopicPick \| None)`. `pick_topic` returns `TopicPicked \| TopicLowConfidence`. The `RuntimeError` in `_classify_envelope` becomes structurally unreachable and is removed. |
| 6b | §8 "ClassificationWrite encodes a discriminated union as nullable fields" | `ClassificationWrite` packs confident and low-confidence shapes into one dataclass; `_validate_write` enforces the invariant at runtime | Split into `ConfidentClassification` and `LowConfidenceClassification` dataclasses. `ClassificationRepo.upsert` accepts the union (`ConfidentClassification \| LowConfidenceClassification`). Remove `_validate_write` — the type system enforces the invariant. |
| 7 | §1 "Phase 2 orchestration has outgrown a single process function" / "Classification sub-pipeline is hidden inside ingest"; §2 "Pipeline entry points pass config and collaborators as loose scalars" | `_process` is 152 LOC / 17 params / returns a 12-tuple; `_classify_envelope` is 188 LOC / 15 params and lives in `ingest.py` | Move `_classify_envelope` and the supporting helpers to a new `prism.classify_pipeline` module. `_process` calls one function from that module. Replace `_process`'s 12-tuple return with a named `IngestPassResult` dataclass; group repo collaborators into an `IngestDeps` dataclass that `_process` accepts. |

### Items NOT in scope

Each row notes why it's deferred and what triggers
revisiting.

| Review ref | Headline | Why deferred |
| --- | --- | --- |
| §6 "Events outbox publishes a type the manifest says is subscribed" | Synthetic `content-ingested` events live in `events_outbox`, which the future dispatcher will publish | This is a **fix**, not a refactor. The right shape is an additive `events_outbox` schema change (e.g. `internal BOOLEAN DEFAULT FALSE`) the dispatcher filters on. Will be written as a separate prompt after this refactor lands so file references are post-refactor accurate. |
| §1 "Dev CLI is long but not the main split candidate" | `dev_cli.py` 458 LOC | Reviewer flagged "split when Phase 3 adds more commands"; the split would be premature today. |
| §6 "Public manifest and runtime API have drifted apart" | `service.json` declares 8 endpoints, FastAPI exposes 1 | Phase 3 will materialize the missing endpoints. The manifest's aspirational stance is intentional; the user-facing risk is small while the project has no real consumers. |
| §2 "Run finalization repeats a wide row-shape signature" | `RunRepo.record_success` / `record_error` / `_finalize` 10–12 args | ADR 010's two-layer exception flow is explicitly carved out; the parameter width is a cost of that design. Defer until ADR 010 itself is revisited. |
| §4 "Migrated SQLite test setup repeats in many files" | Alembic + `connect_sqlite` fixture duplicated across 3+ test files | Real concern (drift risk on migration setup) but not urgent. Defer to a test-support pass; consolidate into `conftest.py` when adding the next test file would make it the 6th copy. |
| §4 "Scraper bridge DDL is copied between dev and integration tests" | Bridge schema in 3 places | ADR 006 bounds bridge knowledge; copies are tolerable while the bridge stays small. |
| §5 "Repo cursor/transaction ceremony dominates simple mutations" | Cursor open/try/finally pattern across 5+ repos | Touches every repo; deserves its own dedicated pass with an explicit helper API. |
| §7 "Event names are call-site strings rather than typed vocabulary" | `"content-classified"` etc. inline strings | Tied to the synthetic-events fix; bundle with that prompt rather than this refactor. |
| §7 "Operation names are untyped strings across audit, ingest, and queries" | `"ingest"` / `"embed"` / `"classify"` strings | Smaller risk than event names; defer until Phase 3 adds new audited operations. |
| §9 "Phase 2 classification tests repeat the same setup cluster" | Test setup repetition in `test_ingest.py` | Add scenario helper when next test would be the 6th repeat. |
| §9 "Large test files now mix old ingest coverage with Phase 2 classify coverage" | `test_ingest.py` 1,091 LOC | Splitting now risks losing helper context that the items above will rely on. Reconsider after this refactor settles. |
| §10 "Phase/slice comments have started to leak into stable modules" | Slice/phase wording in module docstrings | Quick fix worth doing as a separate small pass; pulling it into this refactor would muddle the diff. |
| §11 "Classification enablement affects serve, dev CLI, and ingest through separate checks" | `classification_enabled` flag scattered | Not a pressing structural issue today; revisit when a second classifier backend or partial facet enablement materializes. |
| §12 "Classification persistence return value is constructed, not read as canonical state" | `ClassificationRepo.upsert` returns the input shape | ADR 014 nuance; the test contract is enforced via comparison to a manual fetch. Defer to either an ADR refresh or a behavior change motivated by a real consumer need. |

## Item-by-item shape

### #1 — Lift `LowConfidenceReason`

Create `./src/prism/classification_types.py`:

```python
"""Cross-layer type aliases for classification.

Owns the LowConfidenceReason vocabulary because both the
LLM-pick layer (which emits values) and the storage layer
(which persists them) read the alias. Keeping it in either
of those modules creates a layer-direction problem.
"""

from __future__ import annotations

from typing import Literal

LowConfidenceReason = Literal[
    "below-threshold",
    "out-of-band",
    "llm-pick-unsure",
    "llm-output-malformed",
]
```

Update both `topic_llm_pick.py` and `classification_repo.py`
to import from `classification_types`. Delete the local
`Literal` definitions.

### #2 — `TopicPrototypeRepo` transaction discipline

`TopicPrototypeRepo.delete_for_topic` and `save_many`
adopt the same shape as other mutating repos:

- Wrap in try/except `sqlite3.Error`: rollback + raise.
- Commit on success at the end of the method body.
- Cursor closed in finally.

In `dev_cli.load_topics_cmd`, remove the manual
`classifier_conn.commit()` / `rollback()` calls; the repo
owns the transaction.

Tests for `TopicPrototypeRepo` may need light updates if
they currently rely on observing pre-commit state through
the same connection (unlikely given existing tests assert
post-`save_many` results).

### #3 — Vec key delimiter shared module

Create `./src/prism/vec_keys.py`:

```python
"""Composite-key encoding for sqlite-vec storage tables (ADR 011).

The delimiter is the ASCII unit separator (``\x1f``) per ADR
011 — never appears in any well-formed identifier the
classifier owns, so collisions are structurally impossible.
"""

from __future__ import annotations

ROW_KEY_DELIMITER = "\x1f"


def encode(*parts: str) -> str:
    for p in parts:
        if ROW_KEY_DELIMITER in p:
            raise ValueError(
                f"row-key part contains delimiter: {p!r}"
            )
    return ROW_KEY_DELIMITER.join(parts)
```

`embedding_repo.py` and `topic_prototype_repo.py` both
import `ROW_KEY_DELIMITER` from `vec_keys`. Remove the
leading-underscore private definitions and the cross-repo
private import. The stored row-key bytes are unchanged
(still `\x1f`-separated per ADR 011).

### #4 — `_fail_queue_item` helper

Add to `ingest.py`:

```python
def _fail_queue_item(
    item: IngestQueueItem,
    *,
    queue_repo: IngestQueueRepo,
    error_prefix: str,
    exc: Exception | str,
    now: datetime | None,
) -> None:
    """Mark an ingest queue item failed with the standard backoff."""
    error = exc if isinstance(exc, str) else f"{error_prefix}: {exc!r}"
    queue_repo.mark_failed(
        item.source_event_id,
        error=error,
        backoff_seconds=_backoff_seconds(item.attempts + 1),
        now=now,
    )
```

Replace the 8 call sites at lines 292, 302, 333, 374,
498, 514, 549, 634 (line numbers pre-refactor) with calls
to this helper.

### #5 — Drop `ingest_once` LLM defaults

Remove the defaults on `ingest_once`:

```python
# before
confidence_threshold: float = 0.55,
retrieval_top_k: int = 5,
ollama_model: str = "qwen2.5:7b-instruct-q4_K_M",

# after
confidence_threshold: float,
retrieval_top_k: int,
ollama_model: str,
```

`_process`'s parameters likewise lose any defaults.
`serve.py` and `dev_cli.py` already supply these
explicitly. The `_run` test helper gets one
`setdefault(...)` line per parameter so existing
non-classify tests still pass.

### #6a — Split `TopicPickResult`

In `topic_llm_pick.py` (or `classification_types.py` if
co-located makes sense):

```python
@dataclass(frozen=True)
class TopicPicked:
    pick: TopicPick
    chosen_topic_id: str


@dataclass(frozen=True)
class TopicLowConfidence:
    reason: LowConfidenceReason
    pick: TopicPick | None


TopicPickResult = TopicPicked | TopicLowConfidence
```

`pick_topic` returns the union. Each existing branch
constructs the matching variant. `_classify_envelope`'s
`if pick_result.chosen_topic_id is not None and
pick_result.pick is not None` becomes
`if isinstance(pick_result, TopicPicked):`. The defensive
`RuntimeError` is deleted.

Test updates: `tests/test_topic_llm_pick.py` adapts the
construction patterns; assertions adapt to the variants.

### #6b — Split `ClassificationWrite`

In `classification_repo.py`:

```python
@dataclass(frozen=True)
class ConfidentClassification:
    envelope_id: str
    active_run_id: str
    tags: list[str] | None
    topic_assignments: list[TopicAssignment]
    confidence: float
    classified_by: str
    classified_at: datetime


@dataclass(frozen=True)
class LowConfidenceClassification:
    envelope_id: str
    active_run_id: str
    reason: LowConfidenceReason
    classified_by: str
    classified_at: datetime


ClassificationWrite = ConfidentClassification | LowConfidenceClassification
```

`ClassificationRepo.upsert` accepts the union; branches
on `isinstance(write, ConfidentClassification)`. Delete
`_validate_write`. Delete the `tags` and `topic_assignments`
fields from `LowConfidenceClassification` (they're always
None / empty by construction).

Test updates: `tests/test_classification_repo.py` constructs
the appropriate variant; the parameterized invalid-combos
test becomes structurally unreachable and is deleted (the
type system enforces the invariant).

### #7 — Extract classify pipeline; replace tuple return

Create `./src/prism/classify_pipeline.py`:

- Move `_classify_envelope` and its supporting types
  (`_ClassifyOutcome`, `_ClassifyFailed`, `_CLASSIFY_FAILED`)
  to this module.
- The function becomes public: `classify_envelope(...)`.
- The classify-specific imports (`pick_topic`,
  `retrieve_top_k_for_envelope`, `TopicAssignment`,
  `EmittedEvent`, `ClassificationRepo`,
  `EventOutboxRepo`, `TopicPrototypeRepo`) move with it.

In `ingest.py`:

- Replace the inline classify branch with one call to
  `classify_envelope(...)`.
- Group the repo collaborators into:

```python
@dataclass(frozen=True)
class IngestDeps:
    queue_repo: IngestQueueRepo
    envelope_repo: EnvelopeRepo
    embedding_repo: EmbeddingRepo
    classification_repo: ClassificationRepo
    event_outbox_repo: EventOutboxRepo
    topic_repo: TopicPrototypeRepo
    run_repo: RunRepo
    mapper: ScraperMapper
```

  Constructed in `ingest_once`; passed into `_process`
  as one argument.

- Replace the 12-tuple return from `_process` with:

```python
@dataclass(frozen=True)
class IngestPassResult:
    observed: int
    skipped_already_saved: int
    processed: int
    refreshed: int
    mapper_returned_none: int
    failed: int
    malformed: int
    embedded: int
    embed_skipped_no_body: int
    embed_failed: int
    classified_confident: int
    classified_low_confidence: int
    classify_skipped_no_embedding: int
    classify_failed: int
```

  `ingest_once` builds `IngestStats` from this.

`_process`'s parameter count drops from 17 to ~5 (the
deps object plus the few non-collaborator params:
`process_limit`, `parent_run_id`, `now`, classify
settings).

## Acceptance bar

- `just check` exits 0 with zero errors and zero warnings.
- Test count unchanged (or one fewer if the
  `ClassificationWrite` invalid-combos parameterized test
  is removed — the type system now prevents the cases).
- `git diff --stat` against the slice baseline shows
  changes in: `src/prism/classification_types.py` (new),
  `src/prism/vec_keys.py` (new),
  `src/prism/classify_pipeline.py` (new),
  `src/prism/topic_llm_pick.py`,
  `src/prism/classification_repo.py`,
  `src/prism/topic_prototype_repo.py`,
  `src/prism/embedding_repo.py`,
  `src/prism/ingest.py`,
  `src/prism/dev_cli.py`,
  `src/prism/serve.py` (lifespan kwarg call),
  `tests/test_topic_llm_pick.py`,
  `tests/test_classification_repo.py`,
  `tests/test_ingest.py`. No ADR changes.
- Behavior unchanged: `prism dev seed-scraper && prism dev
  ingest-once` produces the same classified envelopes,
  same events, same row counts as before the refactor.

## What this slice does NOT do

- Does not change ANY ADR.
- Does not change the wire format of any emitted event.
- Does not change any migration. (`vec_keys` rename is
  a Python-side rename only — the column values stored
  do not change because the delimiter string `"::"` is
  unchanged.)
- Does not address the synthetic-content-ingested-events
  concern. That fix lands in a follow-up prompt with
  post-refactor file references.
- Does not split `dev_cli.py` or `test_ingest.py`.
- Does not address the cursor/transaction ceremony or
  the manifest/runtime endpoint mismatch.

## After this slice

Once the refactor lands and is committed, the next
prompt addresses the synthetic-content-ingested-events
correctness concern (review §6 second entry). With the
classify pipeline now in `classify_pipeline.py`, the file
references in that prompt will be stable.
