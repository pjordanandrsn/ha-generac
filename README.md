# Generac MobileLink (Anderson fork) — Home Assistant integration

[![HACS Default][hacs-badge]][hacs]
[![Validate][validate-badge]][validate-workflow]

> **✅ Available in HACS** — this integration is in the [HACS][hacs]
> **default** store, so you can install it directly from HACS with no
> custom repository needed. See [Install via HACS](#install-via-hacs).

A Home Assistant custom integration that polls the Generac MobileLink
cloud and exposes whole-home generators (and propane tank monitors) as
HA entities.

This is a downstream **fork** of [`binarydev/ha-generac`][upstream] that
ships the auth rewrite from [PR #267][pr267] by [@sslivins][sslivins]
plus a small handler for the Auth0 Forms consent prompt some
post-migration accounts get on first login.

> **Status:** Maintained as part of the Anderson family wall-display
> kiosk. Community PRs welcome — no SLA. If you want the official
> upstream, use [`binarydev/ha-generac`][upstream] (note: as of writing,
> upstream is still on the legacy cookie-based auth path that breaks
> when Generac retires the old endpoints).

## Why this fork exists

Generac flipped MobileLink authentication to Auth0/DPoP on
**2026-04-21**
([Generac support article][generac-migration]). The released
`binarydev/ha-generac` v0.4.2 still scrapes `MobileLinkClientCookie`
from the legacy session — that path is fragile and will break outright
once Generac retires the legacy endpoint.

[PR #267][pr267] by `@sslivins` rewrites the auth path to Auth0
universal login (email + password) plus DPoP-bound refresh tokens,
persisted in the HA config entry. The user enters credentials once via
the UI and the integration handles refresh forever.

This fork tracks PR #267 with one extra patch (see
[Differences from upstream PR #267](#differences-from-upstream-pr-267))
and is published in the HACS **default** store so other
MobileLink users aren't blocked on the upstream review queue.

## Install via HACS

This integration is in the HACS **default** store, so you can install it
directly — no custom repository needed:

1. In Home Assistant, open **HACS**.
2. Search for **Generac MobileLink (Anderson fork)** and click
   **Download**.
3. Restart Home Assistant.
4. **Settings → Devices & Services → Add Integration → Generac
   MobileLink** and follow [First-time setup](#first-time-setup).

<details>
<summary>Alternative: add as a custom repository</summary>

1. In Home Assistant, open **HACS**.
2. Open the top-right menu and pick **Custom repositories**.
3. Add this repository:
   - Repository: `https://github.com/pjordanandrsn/ha-generac`
   - Category: `Integration`
4. Click **Add**.
5. Find **Generac MobileLink (Anderson fork)** in the HACS integration
   list and click **Download**.
6. Restart Home Assistant.
7. **Settings → Devices & Services → Add Integration → Generac
   MobileLink** and follow [First-time setup](#first-time-setup).
</details>

## Manual install (no HACS)

1. Copy `custom_components/generac/` from this repo into your HA
   `config/custom_components/` directory.
2. Restart Home Assistant.
3. Add the integration via **Settings → Devices & Services → Add
   Integration → Generac MobileLink**.

## First-time setup

Adding the integration requires a Home Assistant **admin** session
(non-admin users cannot add integrations). If your wall display runs as
a non-admin kiosk user, do this from a separate browser session:

1. Open `http://<HA_HOST>:8123` and sign in as an admin.
2. **Settings → Devices & Services → Add Integration**.
3. Search **Generac MobileLink**.
4. Enter your **MyGenerac** email and password (the same credentials
   you use in the MobileLink mobile app).
5. Submit. Within ~30 s, the integration creates entities for each
   generator and tank monitor on the account.

Wrong credentials surface as **"Invalid email or password"** on the
form. Anything else surfaces as **"Unexpected error"** — check the HA
log for the actual exception.

### What entities you get

Per generator (sensor + binary_sensor):

- `sensor.generac_<id>_status` — e.g. `Ready`, `Running`, `Exercising`,
  `Stopped`
- `sensor.generac_<id>_battery_voltage` — typically 12.5–13.5 V on a
  healthy unit
- `sensor.generac_<id>_run_time`, `protection_time`, `last_seen`,
  `connection_time`, `activation_date`
- `sensor.generac_<id>_dealer_email`, `dealer_name`, `dealer_phone`,
  `address`, `serial_number`, `model_number`, `device_ssid`, `panel_id`
- `binary_sensor.generac_<id>_is_connected`, `is_connecting`,
  `has_maintenance_alert`, `has_warning`

Per propane tank monitor: similar set covering capacity, fuel level,
fuel type, orientation, last reading date, battery level.

There's also a `weather` entity per generator location (forecast at the
generator address) and an `image` entity exposing the device thumbnail.

## Known gotchas

- **Auth0 consent prompts.** Some MyGenerac accounts trigger an Auth0
  Forms consent / T&C prompt on first OAuth login. This fork includes a
  `_handle_custom_prompt()` handler that POSTs `action=default` to
  clear them automatically (handles up to 3 chained prompts before
  giving up). If the handler can't clear the prompt, complete the
  pending form interactively in the MobileLink mobile app once — Auth0
  remembers the acknowledgement account-wide for subsequent OAuth flows
  from any client.
- **Refresh token after password change.** Rotating your MyGenerac
  password invalidates the stored refresh token. HA surfaces this as a
  `Reauth` notification — click it and re-enter the new password. No
  reinstall needed.
- **Conservative poll interval.** `iot_class: cloud_polling` with
  `DEFAULT_SCAN_INTERVAL = 900 s` (15 min). The MobileLink cloud
  doesn't push faster than this, and the API is rate-limited per
  account. Don't bump it without monitoring for 429s.
- **`requirements` pin.** `manifest.json` pins
  `dacite==1.9.2` and `cryptography>=41`. The upstream PR's `setup.cfg`
  pins `dacite==1.9.2`; the manifest is the file HA reads at install
  time, so that's the source of truth here.

## Differences from upstream PR #267

On top of `pr3-email-password-auth` HEAD (8c550ca):

1. **`auth.py: _handle_custom_prompt()`** — handles Auth0 Forms prompts
   (T&C / consent / privacy updates) that some MyGenerac accounts get
   to clear once. POSTs `state=…&action=default` directly to
   `/u/custom-prompt/<id>`. Loops up to 3 chained prompts before giving
   up. The React-rendered page has no static `<form>`, but the POST
   endpoint and body convention are stable across Auth0 Forms
   instances.
2. **`auth.py: WARNING-level step= logging`** — emits a step marker at
   each redirect to make first-time login debuggable from the HA log.
3. **`manifest.json: version`** — `0.5.3-anderson-fork` to differentiate
   from upstream releases.

The `_handle_custom_prompt` patch is a candidate to push back to PR
#267 — benefits any user whose account gets a similar prompt.

## Upstreaming progress

Subscribe to [`binarydev/ha-generac` PR #267][pr267]. When it merges
and a release is cut, follow the upstream-tracking steps in this repo's
`UPSTREAM.md` (TODO) to migrate cleanly back to upstream — refresh
tokens are forward-compatible, so you won't need to re-authenticate.

## License

[MIT][license] — preserved verbatim from `binarydev/ha-generac`.

Copyright (c) 2025 binarydev.
Modifications copyright (c) 2026 Anderson family / pjordanandrsn,
based on PR #267 by sslivins.

## Acknowledgments

- [`@binarydev`][binarydev] — original `ha-generac` integration and
  ongoing upstream maintenance.
- [`@sslivins`][sslivins] — auth rewrite ([PR #267][pr267]) that this
  fork is built on.
- [Jeff Terrace][jterrace] — for the [GenMon + Raspberry Pi local
  alternative blog post][genmon-blog] linked from the upstream README.

[upstream]: https://github.com/binarydev/ha-generac
[pr267]: https://github.com/binarydev/ha-generac/pull/267
[binarydev]: https://github.com/binarydev
[sslivins]: https://github.com/sslivins
[jterrace]: https://github.com/jterrace
[genmon-blog]: https://blog.jeffterrace.com/2025/10/free-from-generac-with-genmon.html
[generac-migration]: https://support.generac.com/s/article/Mobile-Link-Migration
[license]: ./LICENSE
[hacs]: https://github.com/hacs/integration
[hacs-badge]: https://img.shields.io/badge/HACS-Default-blue.svg
[validate-workflow]: https://github.com/pjordanandrsn/ha-generac/actions/workflows/validate.yaml
[validate-badge]: https://github.com/pjordanandrsn/ha-generac/actions/workflows/validate.yaml/badge.svg
