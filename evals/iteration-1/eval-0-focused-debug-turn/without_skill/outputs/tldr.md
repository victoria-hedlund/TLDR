## /tldr

**Task:** Investigate flaky checkout tests in CI.

**Root cause:** `tests/checkout/conftest.py` instantiated `StripeClient` once at module level, so all tests in the package shared a single client instance. Its `self._pending` dict accumulated charges across tests, causing state to leak between them. This only surfaced in CI because `-p xdist` changes test ordering; the default local ordering happened to avoid the collision.

**Failing tests:**
- `test_declined_card` — read a leftover charge's status as `'pending'` instead of `'declined'`
- `test_retry_succeeds` — `KeyError: 'ch_test_991'` after a different test had already popped that charge

**Fix:** Converted `client` in `conftest.py` from a module-level singleton to a per-test pytest fixture, and updated the 14 call sites in `test_payment.py` and `test_webhook.py` to take it as a fixture argument.

**Verification:** `pytest tests/checkout -x -p no:randomly --count 50` → 2100 passed, 50/50 consecutive runs green.

**Follow-up (not done):** `shipping/rates.py` and `refunds/processor.py` build their clients the same module-level way and likely have the same latent bug, but this wasn't verified.
