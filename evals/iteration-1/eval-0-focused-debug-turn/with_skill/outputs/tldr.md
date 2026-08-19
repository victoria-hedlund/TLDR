**TL;DR** — The flaky checkout CI failures were a shared module-level `StripeClient` leaking state between tests; fixed with a per-test fixture, 50 consecutive runs pass.

**Did**
- Converted the module-level `client` in `tests/checkout/conftest.py` to a per-test fixture
- Updated 14 call sites across `test_payment.py` and `test_webhook.py` to take `client` as a fixture argument
- Ran `pytest tests/checkout -x -p no:randomly --count 50` — 2100 passed, all green

**Found**
- `StripeClient._pending` (`payments/client.py:88`) accumulated charges across tests since the client was instantiated once at module scope
- `test_declined_card` and `test_retry_succeeds` were colliding over that shared state, not hitting real logic bugs
- Only reproduced in CI because `-p xdist` changes test ordering; local's default ordering happened to avoid the collision

**Next**
- `shipping/rates.py` and `refunds/processor.py` build clients the same module-level way and likely have the same latent bug — suspected, not verified or tested
