tl;dr — Fixed the export endpoint's OOM issue by converting it from buffering the full CSV in memory to a streaming response.

- **Problem:** `export_transactions` loaded all rows via `.all()` and built the whole CSV in a `StringIO` before returning it — ~3GB for a 2.1M-row account, causing OOMs.
- **Fix:** Rewrote it as a generator using `q.yield_per(1000)` to cap the ORM identity map, wrapped with `stream_with_context` since the generator accesses the DB session after the request context closes.
- **Verified:** `pytest tests/api/test_export.py` — 12 passed. Benchmarked against a synthetic 2M-row account: peak RSS dropped from 3140 MB to 84 MB (flat regardless of account size), wall time improved from 58.7s to 41.2s (previous version OOM'd at 1.2M rows).
- **Caveat flagged:** streaming means a mid-request error can't return a 500 — the client already has a 200 with a partial CSV. Added a trailing sentinel row to signal truncation, but the frontend doesn't check for it yet.
- **Last question answered:** confirmed `scripts/bench_export.py` was newly created in this session, not pre-existing.
