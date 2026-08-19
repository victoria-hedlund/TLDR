## TL;DR

**JWT migration: done and shipped.** Added token issuance/verification (`api/auth/jwt.py`) and a refresh endpoint (`api/auth/refresh.py`, 15-min access / 30-day refresh, rotating). Migrated all 9 handlers off `require_session`. Tests pass (338 passed, 2 skipped). Pushed to `claude/oauth-rgqext`, **PR #212 open, CI green**.

**Needs your decision — security hole found in `api/legacy.py`, not touched.** The old `/v1/login` endpoint has no CSRF protection and an unvalidated `next` redirect param. A malicious page can log a victim into an attacker's account cross-origin, plus it's an open redirect. Pre-existing, not caused by this work — left alone because old mobile clients may still depend on it. Needs a call on whether/how to retire or patch it.

**Rate limiting: prototyped, not wired in.** Built a token-bucket limiter (`middleware/limit.py`, 7 tests passing) but it's not registered in the app. Reason: it currently tracks limits in-process, so across 8 gunicorn workers the real limit is 8x whatever's configured and resets on every deploy. Redis would fix that but adds a hard dependency — request serving breaks if Redis is down. That's a product tradeoff, deliberately left for you.

**Staging token refresh: still flaky, unfixed.** 2 rejected refreshes out of ~40 watched over 6 hours, both logged as `iat` slightly in the future — consistent with clock skew, fixable with ~30s leeway on verification. Caveat: only 2 data points, and the drifting-clock explanation hasn't been confirmed (could be something else producing a future `iat`). No fix applied yet.

**Bottom line:** auth migration is mergeable, but two things are gated on you — the legacy login vulnerability and the rate-limiting storage tradeoff — and the staging refresh issue still needs a confirmed root cause before fixing.
