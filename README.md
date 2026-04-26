# Project Workspace Index

This folder holds in-progress task plans, reviews, ideas, and internal issue notes.
Files here are intentionally not committed.

## Plans & Tasks

| Date | Type | File | Description | Status |
| --- | --- | --- | --- | --- |
| 2026-04-25 | Report | [phase-2-report](./reports/2026-04-25-phase-2-report.md) | Phase 2 orientation pass for a fresh-session agent: state of code at HEAD `3b2eb4d`, what shipped (9 commits, ADRs 018–022), architecture-changes anchored to ADRs, Phase 3 entry points, known dragons | Complete |
| 2026-04-25 | Plan | [post-phase-2-structural-refactor](./plans/2026-04-25-post-phase-2-structural-refactor.md) | Behavior-preserving cleanup addressing eight items from the post-Phase-2 review: shared types, transaction discipline, helper extraction, type-discriminator splits, classify-pipeline extraction | Complete |
| 2026-04-25 | Review | [post-phase-2-structural-review](./reviews/2026-04-25-post-phase-2-structural-review.md) | Structural pass after Phase 2: long units, duplicate definitions, repeated patterns, layer health, type-discriminator integrity, config coupling | Complete |
| 2026-04-24 | Report | [phase-1-report](./reports/2026-04-24-phase-1-report.md) | Phase 1 orientation pass for a fresh-session agent: state of code, canonical reading order, architecture anchored to ADRs, known dragons | Complete |
| 2026-04-20 | Task | [multi-model-embedding-config](./tasks/multi-model-embedding-config.md) | Thread `config.pipeline.embedding.*` through the embedder factory when a second model lands | 🔴 Deferred |
| 2026-04-20 | Task | [real-boundary-test-discipline](./tasks/real-boundary-test-discipline.md) | Process insight: unit tests must cross real resource boundaries, not placeholders | 🟢 Approved |
| 2026-04-18 | Plan | [classification-service-foundation](./plans/2026-04-18-classification-service-foundation.md) | Standalone Python classification service with audit layer, service manifest, and multi-faceted classification pipeline | 🟡 In Progress |

## Open Items

Durable cross-session record of things not yet in code but worth
carrying forward. One-liners where the trigger is obvious; linked
task files where the item has cross-cutting implications.

### Deferred work

`When` is the date the item entered the deferred state. `Trigger`
is the event/condition that should bring it back into scope.

| Item | Why | When | Trigger |
| --- | --- | --- | --- |
| Config-driven embedder selection → [task](./tasks/multi-model-embedding-config.md) | `MODEL_VERSION` constant and `config.pipeline.embedding.*` drift silently today | 2026-04-20 | Second embedder (jina-v3, Qwen3-Embedding-4B, multilingual+long-context) lands |
| Title/summary fallback when `content.body` is empty | Body-less envelopes get no vector today; search/ranking can't retrieve them at all | 2026-04-24 | Product/ranking requires embedded representation for body-less items |
| Runtime-independent `prism serve` (`ingest.source` switch: `scraper-bridge` / `none` / `fixture`, making `scraper_db_path` only apply to `scraper-bridge`) | The switch's value is making the scraper bridge one adapter among others; designing the abstraction against a single implementation now would bake in the wrong shape | 2026-04-25 | Second ingest adapter materializes (real event bus or fixture source) during/after Phase 2 |

### Known risks accepted for now

| Risk | What would change our mind |
| --- | --- |
| `TestClient(app)` without `with` skips the lifespan — future Starlette change or a test using `with` form could start running it and hit a non-existent scraper DB path | Starlette default changes, or `test_api.py` adopts the `with` form |
| `check_same_thread=False` in `connect_sqlite` is safe only under the single-task-loop + single-worker-thread invariant | Any code ever shares a connection across concurrent tasks |
| `configure_tracing(cfg.audit)` called on every lifespan enter swaps the OTel provider — could surprise a dev REPL that set up an in-memory exporter before booting via `TestClient(app, lifespan="on")` | Test/dev flow grows a pattern that collides |
| `asyncio.to_thread` uses the default executor (min 32, cpu+4) — fine for the single-task loop today | Future loop fans out parallel `to_thread` calls |
| Pass-level error log has no rate limiting (cap is the 30 s backoff) | Operators see the log flood during a real extended outage |
| `test_serve_integration.py` deadline is hard-coded at 120 s | Cold-cache runs on slow CI start timing out |

### Candidate ADRs (surfaced, not yet promoted)

| Candidate | Priority | Promote when |
| --- | --- | --- |
| Optional-dependency injection shape for per-stage kwargs (`embedder: Embedder \| None = None`, etc.) | Low | Classify/route/reconcile each adopt the shape and an agent argues for "required dep" |
| `asyncio.wait_for(shutdown.wait(), timeout=backoff)` cooperative-shutdown idiom | Low | Phase 2 adds three more loops without converging on this shape |

### Cross-slice process insights

| Insight | Where it lives |
| --- | --- |
| Stub-placeholder tests miss real-boundary failures | [task](./tasks/real-boundary-test-discipline.md) |
| Multi-step teardown needs explicit "step N raises, does step N+1 still run?" tests | Same task file (related section) |
| Prompt/plan/ADR contradictions need escalation, not silent implementation — raise before typing | Commit `7f15176`'s handoff insight 1 |
| Prompt internal contradictions (scope vs. acceptance) — tiebreak to the stricter reading, surface the divergence | Commit `8e071e4`'s handoff "Divergences from the plan" |

## Status legend

- 🔵 **Planned** — in queue, not started
- 🟡 **In Progress** — actively being worked on
- 🟢 **Approved** — approved, merged, or active as a standing rule
- 🔴 **Deferred** — parked, not doing now
- (no icon) **Complete** / **Implemented, partial**
