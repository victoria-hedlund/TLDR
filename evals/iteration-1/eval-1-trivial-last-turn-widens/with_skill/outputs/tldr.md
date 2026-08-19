**TL;DR** — Converted the export endpoint from buffering the whole CSV in memory to a streaming generator; peak memory dropped from 3.1GB to 84MB and it no longer OOMs on large accounts.

**Did**
- Rewrote `export_transactions` in `api/export.py` to stream rows via a generator with `db.query(...).yield_per(1000)` instead of `.all()` + `StringIO`, using `stream_with_context`
- Ran `pytest tests/api/test_export.py` — 12 passed
- Wrote `scripts/bench_export.py` (new file, created this session) and benchmarked against a synthetic 2M-row account

**Found**
- Root cause was double buffering: `.all()` loaded every row as ORM objects, then `out.getvalue()` made a second full copy — ~3GB for a 2.1M-row account
- Benchmark: peak RSS 84MB (was 3140MB), wall time 41.2s (was 58.7s, previously OOM'd around 1.2M rows)
- Streaming means errors mid-response can no longer return a clean 500 — client already got a 200 with partial CSV. Added a trailing sentinel row to signal truncation, but the frontend doesn't check for it yet

**Next**
- Wire up frontend truncation detection for the sentinel row
