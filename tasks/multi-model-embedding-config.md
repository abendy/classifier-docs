# Multi-model embedding config threading

## Status

Deferred. Triggered when a second embedder enters (plan's A/B
candidates: jina-v3, Qwen3-Embedding-4B).

## Current state

- `src/prism/embedding.py` hard-codes `MODEL_VERSION =
  "bge-m3@1.5"` and `EMBEDDING_DIMENSION = 1024`.
- `create_bge_m3_embedder()` takes no args.
- `config.yaml` has `pipeline.embedding.model` and `.version`
  but no code reads them.
- `src/prism/ingest.py` writes `MODEL_VERSION` as the tag on
  every `embeddings` row.
- Constant and yaml stay in sync by convention only — nothing
  catches drift.

## What changes when a second embedder lands

- Factory becomes config-driven:
  `create_embedder(cfg.pipeline.embedding) -> Embedder`
  dispatches on `model`. Each factory branch does its own lazy
  import per ADR 015.
- `MODEL_VERSION` stops being a module constant; threaded from
  config through `ingest_once` to `EmbeddingRepo.save`. Repo
  signature doesn't change (`model_version` is already a kwarg).
- `EMBEDDING_DIMENSION` likely becomes a per-model property
  reported by the `Embedder` instance. Tests that assert
  `(1024,)` shape today need to read the Embedder's declared
  dim or be parametrised.
- Schema: no change. ADR 011's synthetic composite key already
  supports A/B coexistence per envelope.
- New ADR: "config-driven embedder selection; model_version
  threading." Decision points: where the dispatch lives (factory
  module vs. config-annotation registry), how tests exercise the
  off-path model (stub with a different version tag).

## Non-goals at that time

- Runtime hot-swap. Restart serve to change embedder.
- Per-request embedder selection. The serve loop embeds with one
  configured model per process; A/B is across re-embeds over
  time, not across concurrent requests.
- Removing the constant in a no-op slice. Land it with the
  second embedder so there's a real second caller exercising
  the new path.

## Files likely to touch

- `src/prism/embedding.py` — factory + constant removal +
  second backend.
- `src/prism/ingest.py` — `MODEL_VERSION` import replaced with
  config read (threaded through `ingest_once`).
- `src/prism/serve.py` — passes config section to the factory
  instead of calling `create_bge_m3_embedder()` directly.
- `src/prism/config.py` — possibly widen `EmbeddingConfig` with
  per-backend fields (e.g., quantization, max_length).
- `tests/test_embedding.py`, `tests/test_ingest.py`,
  `tests/test_serve.py` — parametrise or add second-model cases.
- New ADR under `docs/adr/`.
