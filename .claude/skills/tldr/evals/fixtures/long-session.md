# Session transcript

**User:** we need to move off session cookies to JWT before the mobile launch. can you scope it and start

**Claude:** Let me map the current auth surface first.

[Ran: `grep -rn "session\[" api/ --include=*.py | wc -l`]
```
47
```

[Ran: `grep -rln "require_session" api/`]
```
api/handlers/account.py
api/handlers/billing.py
api/handlers/orders.py
api/handlers/profile.py
api/handlers/settings.py
api/handlers/team.py
api/handlers/webhooks.py
api/handlers/admin.py
api/handlers/reports.py
api/legacy.py
```

Ten files use the decorator. Nine of them are normal handler modules; `api/legacy.py` is different and I'll come back to it.

I'll do this in three steps: add JWT issuance alongside the existing cookie path, migrate handlers one at a time, then remove the cookie path once nothing references it.

[Created api/auth/jwt.py]

Implemented `issue_token`, `verify_token`, and a `require_token` decorator matching `require_session`'s signature so handler changes are one-line.

[Edited 9 handler files]

Migrated all nine. The only non-mechanical one was `billing.py`, which read `session['impersonating_admin_id']` for the support-impersonation flow — that had to become a custom claim rather than a lookup.

[Ran: `pytest tests/api/ -q`]
```
338 passed, 2 skipped in 22.1s
```

[Created api/auth/refresh.py]

Added `POST /auth/refresh`. Access tokens are 15 minutes, refresh tokens 30 days, rotating on use.

I pushed this to `claude/oauth-rgqext` and opened PR #212. CI is green on it.

Now, `api/legacy.py`. This is the one that concerns me:

[Read: api/legacy.py]
```python
@app.route("/v1/login", methods=["POST"])
def legacy_login():                            # line 140
    u = User.by_email(request.form["email"])
    if u and u.check_password(request.form["password"]):
        session["uid"] = u.id
        return redirect(request.form.get("next", "/"))
```

This endpoint has no CSRF token check at all, and it honours an unvalidated `next` parameter. Both predate the JWT work — this is existing exposure, not something I introduced. It's a form-POST login, so a malicious page can submit to it cross-origin and log a victim into an attacker-controlled account, and the `next` param makes it an open redirect on top. I did not change this file; it needs its own decision because there may be old mobile clients still hitting it.

**User:** ok also can you look at rate limiting while you're in there, we're getting hammered on /search

**Claude:** Sure. Let me see what's there now.

[Ran: `grep -rn "ratelimit\|rate_limit" api/ middleware/`]
```
(no output)
```

Nothing at all — there's no rate limiting anywhere in the app.

[Created middleware/limit.py]

I prototyped a token bucket: per-IP buckets, configurable rate and burst, sliding refill. It's about 60 lines and unit tested in isolation.

[Ran: `pytest tests/middleware/test_limit.py`]
```
7 passed in 0.4s
```

But I have not wired it into the app, and I don't think I should without a decision from you. The prototype keeps buckets in a process-local dict, which means each of our 8 gunicorn workers enforces its own independent limit — so the effective limit is 8x whatever you configure, and it resets on every deploy. The alternative is backing it with Redis, which makes the limit accurate and durable but adds a hard dependency on Redis being up for requests to be served at all.

That tradeoff is a product call more than a technical one, so I've left the middleware unregistered pending your answer.

**User:** got it. what about the token refresh thing, did staging ever settle down

**Claude:** Not entirely. Staging has failed refresh twice today out of roughly 40 attempts I watched.

[Ran: `kubectl logs -n staging deploy/api --since=6h | grep -i "refresh"`]
```
2026-08-19T09:14:22Z WARN  token refresh rejected: iat 3.2s in future
2026-08-19T11:47:08Z WARN  token refresh rejected: iat 2.8s in future
```

Both rejections are clock skew — the token's issued-at timestamp is slightly ahead of the verifying node's clock, and my `verify_token` currently allows zero leeway. The standard fix is a small `leeway` on the claim validation, typically 30 seconds.

I want to be careful about how confident I sound here: two data points over six hours is thin, and I haven't confirmed that the staging nodes actually have drifting clocks rather than something else producing a future `iat`. It's the most likely explanation and it matches the error text exactly, but I haven't proven it and I haven't applied the fix.
