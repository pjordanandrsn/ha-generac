**Title:** Force one token refresh after an API 401, bound HTTP timeouts, and keep reauth reachable

> This contribution originated with @technorat2 in [pjordanandrsn/ha-generac#4](https://github.com/pjordanandrsn/ha-generac/pull/4). Credit for the original implementation belongs to them: the forced refresh after a 401, the bounded timeouts, the timeout retry and the credential replacement on reauth are their work. I adapted it to current upstream, fixed a defect that blocked reauthentication, and added regression tests so these improvements can land in the main project. @technorat2 agreed on 2026-07-28 to move the work upstream ([comment](https://github.com/pjordanandrsn/ha-generac/pull/4#issuecomment-5108792803)); I'm finishing that migration with full attribution.
>
> @technorat2, I'd welcome your review, especially of anything I changed from your original.

Companion PR (independent, no ordering dependency): entity naming and telemetry, refs #284. <!-- replace with link once opened -->

## Problem

- A `401` from the Mobile Link resource API currently raises `SessionExpiredException` straight away. Entities stay unavailable until the next poll, even when a single token refresh would have worked.
- No API or token request sets a timeout, so a stalled connection holds a poll for aiohttp's 5-minute default.
- Reauth/reconfigure merge new credentials into `entry.data` (`{**entry.data, **entry_data}`). Entries created by the retired cookie/password flows therefore keep `password` and `session_cookie` indefinitely.

## Behavior changes

- **401 → one forced refresh → one retry.** `GeneracAuth.force_refresh(rejected_token)` mints a new access token. If a concurrent caller has already replaced the rejected token, it reuses that one instead of refreshing again, so concurrent callers can't spend the refresh token twice. A second `401` raises `SessionExpiredException` (→ `UpdateFailed`).
- **Correct exception semantics** (fix beyond the original patch): the forced refresh runs outside the transport `try/except`. A revoked refresh token (`invalid_grant`, 400/401/403) therefore reaches the coordinator as `InvalidCredentialsException` → `ConfigEntryAuthFailed` → reauth. In the original patch it was wrapped as `IOError` and treated as transient, so reauth never started. A token-endpoint outage keeps its own type (`UpdateFailed`).
- **Bounded timeouts:** 45 s total, 10 s connect, 30 s read. Applied to API GETs, token refresh, and the isolated login session, which covers the whole universal-login flow and code exchange.
- **One transport retry:** a timeout or `aiohttp.ClientConnectionError` on a GET is retried once after 1 s, then raised as `IOError`. The transport retry and the 401 refresh retry each have their own single budget, so a request makes at most 3 attempts.
- **Coordinator:** `OSError` (timeouts, connection failures, the `IOError` above) becomes a logged warning plus `UpdateFailed` instead of an "unexpected error" with a traceback, and the coordinator is marked offline as in the other failure branches.
- **Credential replacement:** reauth/reconfigure drop `password` and `session_cookie` and keep any other keys.
- The resource server still uses `Authorization: Bearer`; nothing moves to DPoP (see the existing probe note in `api.py`).

## Compatibility

- No config entry version bump or migration. Entry data keeps the same three keys (`username`, `refresh_token`, `dpop_pem`); only stale legacy keys are removed, and only on reauth/reconfigure.
- No changes to entities, the manifest, versions, CI or translations.

## Regression tests added

`tests/test_api.py`, `tests/test_auth.py`, `tests/test_coordinator.py`, `tests/test_config_flow.py` (+23 tests). All network I/O is mocked; nothing contacts Mobile Link or Auth0.

- valid token, no refresh; expired token refreshed (with timeout)
- 401 → forced refresh → success (new Bearer on the retry)
- persistent 401 after refresh → `SessionExpiredException`
- revoked refresh token during the forced refresh → `InvalidCredentialsException`, explicitly *not* `OSError`
- **end to end:** API 401 → real `GeneracAuth`/`GeneracApiClient` → 403 `invalid_grant` → `ConfigEntryAuthFailed`. This test fails on the original patch with `OSError: … InvalidCredentialsException: revoked`.
- token-endpoint 5xx during the forced refresh keeps `RuntimeError`
- timeout then success; repeated timeout; connection error then success; repeated connection error; timeout + 401 + success (separate budgets)
- concurrent callers on expiry share one refresh; concurrent forced refreshes for the same rejected token share one refresh (verified to fail without the dedupe check); refresh-token rotation is persisted exactly once
- failed refresh (403 and 500): neither the exception nor the logs contain the refresh token or the DPoP key
- login session has the bounded timeout
- reauth and reconfigure replace stale `password` / `session_cookie`

## Validation (run locally, Python 3.14.6, `homeassistant==2026.7.4`, `pytest-homeassistant-custom-component==0.13.348` from `requirements.txt`)

- `pytest`: **98 passed** (baseline on `main`: 75 passed). The 2 extra warnings are another instance of the existing `GeneracFlowHandler.async_remove was never awaited` warning, triggered by the new config-flow tests.
- `pre-commit run --all-files` (check-yaml, end-of-file, trailing-whitespace, flake8, reorder-python-imports, black 23.3.0, prettier): all passed
- `python -m compileall custom_components`: OK; `git diff --check`: clean
- Merged together with the companion PR: 119 passed
- Not run locally: hassfest and HACS validation (CI runs both; this PR doesn't touch the manifest).

## Commits and attribution

1. `Retry once after an API 401 and bound Mobile Link HTTP timeouts`: **authored by @technorat2**, ported from their commits 7e3b067 and 40ae00b and adapted to current `main`. It deliberately preserves the original logic, including the defect fixed in commit 2.
2. `Keep reauth reachable when the forced refresh is rejected`: the exception-semantics fix, refresh dedupe, connection-error retry and tests (`Co-authored-by: technorat2`).

If you squash-merge, GitHub keeps @technorat2 as a co-author.

## Not carried over from the original PR

- `async_set_unique_id(email.casefold())`: this changes config-entry identity for new entries only, which would make the duplicate check inconsistent with existing entries. It's out of scope here.
- Email `.strip()` on user setup and removal of the `session_cookie` translation labels: already on `main` (#310, ff0cc61).
- Version bump to 0.5.6: release metadata, left to the maintainer.

## Limitations

- Exercised only against mocks. The retry and timeout values (45/10/30 s, 1 s retry delay) are the original contribution's, not measured against Mobile Link.
- A persistent 401 after a successful refresh stays `UpdateFailed` rather than reauth: the refresh token still works, so prompting for a password would not help.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01K2RdXuPegztHA1uMG2fbtk

---
_Generated by [Claude Code](https://claude.ai/code)_
