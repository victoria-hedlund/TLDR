**TL;DR** — JWT migration is done and PR #212 is open with CI green; along the way we found a pre-existing CSRF/open-redirect hole in the legacy login endpoint, a rate limiter that's prototyped but not wired in pending a Redis-vs-in-process call, and a staging token-refresh failure that's suspected (not confirmed) to be clock skew.

**Did**
- Migrated all 9 `require_session` handlers to JWT (`api/auth/jwt.py`), with `billing.py`'s impersonation flow moved to a custom claim instead of a session lookup
- Added `POST /auth/refresh` (`api/auth/refresh.py`) — 15min access tokens, 30-day rotating refresh tokens
- Ran `tests/api/` — 338 passed, 2 skipped; pushed to `claude/oauth-rgqext`, opened PR #212, CI green
- Prototyped a per-IP token-bucket rate limiter (`middleware/limit.py`), unit tests passing (7/7) — left unregistered, not wired into the app
- Checked staging logs for the refresh-failure reports

**Found**
- `api/legacy.py:140` (`/v1/login`) has no CSRF check and an unvalidated `next` redirect param — pre-existing, not introduced by this work, but enables cross-origin login CSRF plus an open redirect; needs its own decision since old mobile clients may still hit it
- No rate limiting existed anywhere in the app before this
- The rate limiter prototype is process-local, so across 8 gunicorn workers the effective limit is really 8x whatever's configured and resets on every deploy; Redis would fix that but adds a hard dependency for serving requests at all
- Staging token refresh failed twice in ~40 watched attempts over 6 hours, both logged as `iat` timestamp ahead of the verifying node's clock — clock skew is the leading theory (matches error text) but unverified: only two data points, root cause not confirmed, and no fix applied yet (`verify_token` currently allows zero leeway)

**Next**
- Decide how to handle the `api/legacy.py` CSRF/open-redirect exposure
- Decide Redis vs in-process before wiring up the rate limiter
- Confirm the clock-skew theory, then apply a ~30s leeway to `verify_token` if confirmed
- PR #212 still needs a reviewer
