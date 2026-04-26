# Real-boundary test discipline

## Status

Process insight surfaced by slice 8's review rounds. Informs
test-writing rules for any lifespan-managed, thread-crossing,
or connection-handling code going forward.

## The pattern

When a test passes a placeholder (`cast(T, object())`, a
`_FakeConnection` class, a bare `Mock`) where production passes
a real resource, the test isn't verifying the boundary that
matters. The code may pass the test and still crash in
production on the first real call.

Slice 8 shipped a loop that moved `sqlite3.Connection` objects
across the `asyncio.to_thread` boundary. The unit tests used
placeholder objects, so the loop "worked." Production would
have raised `sqlite3.ProgrammingError("SQLite objects created
in a thread can only be used in that same thread")` on the
first real pass. The integration test would have caught it —
but it was deselected by default and not run during review.

Fix: `connect_sqlite` now passes `check_same_thread=False`.
More importantly, a new unit test opens real
`sqlite3.Connection` instances, drives them through the real
`asyncio.to_thread` boundary, and runs `SELECT 1` on them —
the test pins the thread-affinity invariant rather than
accepting placeholder success as proof.

## Rule

For any slice that adds lifespan-managed state, thread-crossing
code, or connection-handling boundaries:

1. At least one unit test must cross the boundary with a *real*
   instance of the resource — not a placeholder.
2. "Real" means: same type, same construction path, same
   mutation surface as production uses. A `sqlite3.Connection`
   from `connect_sqlite`, not a `Mock` returning `None` from
   `.execute`.
3. The test asserts a real boundary-sensitive operation (query,
   read, context-manager enter/exit), not just that the
   resource was received.
4. Don't rely on the integration test to catch it. Integration
   tests get deselected (`-m 'not integration'`); unit tests
   don't.

## Related: multi-step teardown needs per-step raise tests

Slice 8 also needed explicit "what happens if cleanup step N
raises — does step N+1 still run?" tests (round-1 MEDIUM
startup leak, round-2 MEDIUM shutdown leak). A bare sequence of
cleanup calls inside a single `try/finally` is only correct
when none of them can raise. Each cleanup step that *can*
raise deserves its own `try/finally` and at least one
regression test asserting "raise propagates AND later cleanup
still ran."

## Where this applies going forward

- Any new subsystem owned by the FastAPI lifespan (classify
  worker, LLM client, vector index).
- Any new `asyncio.to_thread` call site.
- Any multi-step teardown (acquire → use → release chains with
  more than one release step).
- Any new repo method that wraps a `sqlite3.Connection` method
  that raises on boundary violations.
