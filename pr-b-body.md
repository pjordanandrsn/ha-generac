**Title:** Friendly entity names with stable unique IDs, numeric signal/battery sensors, diagnostics redaction fixes (refs #284)

> This contribution originated with @technorat2 in [pjordanandrsn/ha-generac#4](https://github.com/pjordanandrsn/ha-generac/pull/4). Credit for the original implementation belongs to them: the friendly-name approach that keeps technical unique IDs, the numeric signal-strength and device-battery sensors, the exercise duration sensor, the `last_update_success` availability check and the first round of diagnostics fixes are their work. I adapted it to current upstream, fixed an entity-registry regression and a few redaction bugs, and added regression tests so these improvements can land in the main project. @technorat2 agreed on 2026-07-28 to move the work upstream ([comment](https://github.com/pjordanandrsn/ha-generac/pull/4#issuecomment-5108792803)); I'm finishing that migration with full attribution.
>
> @technorat2, I'd welcome your review, especially of anything I changed from your original.

Refs #284. Companion PR (independent, no ordering dependency): authentication and reliability. <!-- replace with link once opened -->

## Problem

#284: entity and friendly names look like `generac_12345678_run_time`. The name doubles as the unique_id suffix, so making it readable naively would re-key every entity in users' registries. Separately:

- signal strength is a string such as `"75%"`, so it can't be graphed;
- the device battery reported by Mobile Link isn't exposed;
- diagnostics labelled IPv6 addresses as `REDACTED_IPV4`, never redacted e-mail addresses (the `parseaddr` check was inverted), and keyed the output by `apparatusId` even though that same field is redacted.

## Behavior changes

- **Friendly names:** entities use `_attr_has_entity_name = True` with a readable label, so Home Assistant shows `<device name> <label>`, e.g. **Generator Status** or **Propane Tank Last Reading**. Newly created entities get device-scoped entity_ids (`sensor.generator_exercise_duration`), which is the `device_entity` form #284 asks for.
- **Stable unique IDs:** every entity class declares an explicit `entity_key`; nothing is inferred from the class name. unique_ids are byte-for-byte unchanged (`<entry_id>_<device_id>_generac_<device_id>_<key>`).
- **Signal strength:** a numeric `%` value with `state_class: measurement`. `"75%"`, `" 75 % "`, `"75"` and `75` all become `75.0`, and missing or malformed values become unknown instead of `"0%"`. Still always created for generators; now also created for other device types (propane monitors) when reported.
- **New sensors, only when the payload carries them:** *Device Battery Level* (`apparatus.properties[].value.batteryLevel`, battery %) and *Exercise Duration* (detail property type 95, minutes).
- **Fuel level** percentages also go through `_safe_float`, so `"45%"` no longer breaks the state.
- **Availability** also follows `coordinator.last_update_success`, and a `None` coordinator payload no longer raises.
- **Diagnostics:** correct `REDACTED_IPV4` / `REDACTED_IPV6`, e-mail addresses become `REDACTED_EMAIL`, devices are keyed `device_0…`, `panelId` / `shortDeviceId` / `deviceId` are added to the redacted keys, and an entry that isn't loaded returns `{"status": "not_loaded"}` instead of raising `KeyError`.
- The debug log on each coordinator update logs the entity_id instead of the full device payload, which contained the address and serial number.

## Compatibility: entity registry inventory

Every existing entity across sensor, binary_sensor, weather and image keeps its unique_id, and therefore its entity_id, history, customizations and dashboard references.

| platform | class | old name (= unique_id suffix source) | new label | unique_id suffix |
|---|---|---|---|---|
| sensor | `StatusSensor` | `generac_<id>_status` | Status | `…_status` unchanged |
| sensor | `DeviceTypeSensor` | `generac_<id>_device_type` | Device Type | `…_device_type` unchanged |
| sensor | `RunTimeSensor` | `generac_<id>_run_time` | Run Time | `…_run_time` unchanged |
| sensor | `ProtectionTimeSensor` | `generac_<id>_protection_time` | Protection Time | `…_protection_time` unchanged |
| sensor | `ActivationDateSensor` | `generac_<id>_activation_date` | Activation Date | `…_activation_date` unchanged |
| sensor | `LastSeenSensor` | `generac_<id>_last_seen` | Last Seen | `…_last_seen` unchanged |
| sensor | `ConnectionTimeSensor` | `generac_<id>_connection_time` | Connection Time | `…_connection_time` unchanged |
| sensor | `BatteryVoltageSensor` | `generac_<id>_battery_voltage` | Battery Voltage | `…_battery_voltage` unchanged |
| sensor | `OutdoorTemperatureSensor` | `generac_<id>_outdoor_temperature` | Outdoor Temperature | `…_outdoor_temperature` unchanged |
| sensor | `SerialNumberSensor` | `generac_<id>_serial_number` | Serial Number | `…_serial_number` unchanged |
| sensor | `ModelNumberSensor` | `generac_<id>_model_number` | Model Number | `…_model_number` unchanged |
| sensor | `DeviceSsidSensor` | `generac_<id>_device_ssid` | Device SSID | `…_device_ssid` unchanged |
| sensor | `StatusLabelSensor` | `generac_<id>_status_label` | Status Label | `…_status_label` unchanged |
| sensor | `StatusTextSensor` | `generac_<id>_status_text` | Status Text | `…_status_text` unchanged |
| sensor | `AddressSensor` | `generac_<id>_address` | Address | `…_address` unchanged |
| sensor | `DealerNameSensor` | `generac_<id>_dealer_name` | Dealer Name | `…_dealer_name` unchanged |
| sensor | `DealerEmailSensor` | `generac_<id>_dealer_email` | Dealer Email | `…_dealer_email` unchanged |
| sensor | `DealerPhoneSensor` | `generac_<id>_dealer_phone` | Dealer Phone | `…_dealer_phone` unchanged |
| sensor | `PanelIDSensor` | `generac_<id>_panel_id` | Panel ID | `…_panel_id` unchanged |
| sensor | `CapacitySensor` | `generac_<id>_capacity` | Capacity | `…_capacity` unchanged |
| sensor | `FuelTypeSensor` | `generac_<id>_fuel_type` | Fuel Type | `…_fuel_type` unchanged |
| sensor | `OrientationSensor` | `generac_<id>_orientation` | Orientation | `…_orientation` unchanged |
| sensor | `BatteryLevelSensor` | `generac_<id>_battery_level` | Battery Level | `…_battery_level` unchanged |
| sensor | `LastReadingDateSensor` | `generac_<id>_last_reading` | Last Reading | `…_last_reading` **pinned** (class-name inference gives `last_reading_date`) |
| sensor | `FuelLevelSensor` | `generac_<id>_fuel_level` | Fuel Level | `…_fuel_level` unchanged |
| sensor | `SignalStrengthSensor` | `generac_<id>_signal_strength` | Signal Strength | `…_signal_strength` unchanged |
| sensor | `ExerciseMinutesSensor` | — | Exercise Duration | `…_exercise_minutes` **new** |
| sensor | `DeviceBatteryLevelSensor` | — | Device Battery Level | `…_device_battery_level` **new** |
| binary_sensor | `GeneracConnectedSensor` | `generac_<id>_is_connected` | Connected | `…_is_connected` unchanged |
| binary_sensor | `GeneracConnectingSensor` | `generac_<id>_is_connecting` | Connecting | `…_is_connecting` unchanged |
| binary_sensor | `GeneracMaintenanceAlertSensor` | `generac_<id>_has_maintenance_alert` | Maintenance Alert | `…_has_maintenance_alert` unchanged |
| binary_sensor | `GeneracWarningSensor` | `generac_<id>_show_warning` | Warning | `…_show_warning` unchanged |
| weather | `WeatherSensor` | `generac_<id>_weather` | Weather | `…_weather` unchanged |
| image | `HeroImageSensor` | `generac_<id>_hero_image` | Hero Image | `…_hero_image` unchanged |

What users will notice:

- Display names change from `generac_<id>_…` to `<device> <label>`. Names set by the user in the UI still take precedence.
- Signal strength changes from a text state (`"75%"`) to a numeric one (`75.0`). Automations comparing it to the string `"75%"` would need updating, and long-term statistics now start recording.
- A generator that doesn't report signal strength now shows *unknown* instead of `"0%"`.
- Existing entity_ids don't change. Only entities created from now on get the new device-scoped entity_id format. That includes fresh installs, so installation docs or examples that hard-code `sensor.generac_<id>_…` apply only to existing installs.

## Regression tests added

`tests/test_entity_registry.py` (new), plus additions to `test_sensor.py`, `test_entity.py` and `test_diagnostics.py` (+21 tests). Existing name assertions were updated to the new labels.

- **Upgrade test:** seeds the registry with every historical unique_id/entity_id for a generator and a propane monitor (39 entities), sets up the integration from dacite-parsed fixture payloads, and asserts:
  - every entity is reused with the same entity_id;
  - each one has a live state, not a restored orphan;
  - no `_2` duplicates appear;
  - only the 3 genuinely new sensors are added, each on the correct device;
  - friendly names, numeric states and new entity_ids are as listed above.

  Run against the original class-name inference, it fails with `Extra items: …_67890_generac_67890_last_reading_date`.
- **Inventory:** each of the 34 entity classes defines its own `entity_key`, with no collisions.
- Numeric percentage parsing (`"75%"`, `" 75 % "`, `"75"`, `75`, `"N/A"`); missing or unexpectedly typed property payloads (`None`, `[]`, list value, scalar value); exercise duration including a malformed value; fuel level `"45%"`.
- Sensor selection for generators vs propane monitors.
- Availability after a failed poll; a `None` coordinator payload.
- Diagnostics: no apparatusId in keys, correct IPv4/IPv6 labels, e-mail redaction (bare and `Name <addr>`), new identifier keys, an entry that isn't loaded.

## Validation (run locally, Python 3.14.6, `homeassistant==2026.7.4`, `pytest-homeassistant-custom-component==0.13.348` from `requirements.txt`)

- `pytest`: **96 passed** (baseline on `main`: 75 passed)
- `pre-commit run --all-files` (check-yaml, end-of-file, trailing-whitespace, flake8, reorder-python-imports, black 23.3.0, prettier): all passed
- `python -m compileall custom_components`: OK; `git diff --check`: clean
- Merged together with the companion PR: 119 passed
- Not run locally: hassfest and HACS validation (CI runs both; this PR doesn't touch the manifest or translations).

## Commits and attribution

1. `Add friendly entity names and numeric signal/battery sensors`: **authored by @technorat2**, their commit 7e3b067 (entity/telemetry half) applied unchanged to current `main`. It deliberately preserves the original, including the issues fixed in commit 2.
2. `Keep entity registry IDs stable; use has_entity_name for friendly names`: explicit keys, `has_entity_name`, keeping the generator signal sensor unconditional, fuel-level parsing, diagnostics fixes and tests (`Co-authored-by: technorat2`).

If you squash-merge, GitHub keeps @technorat2 as a co-author.

## Not carried over from the original PR

- `brand/icon.png` (Generac logo): Home Assistant brand assets conventionally go through home-assistant/brands, and this repo's HACS check runs with `ignore: brands`. Adding a third-party trademark is the maintainer's call, so it's left out.
- `session_cookie` translation label removal: already on `main` (ff0cc61).
- Dropping `device_class: connectivity` from *Connecting*: an unrelated UI change, so it's kept as-is.

## Limitations

- Entity names are plain English labels, not `translation_key`-based. That could be a follow-up if you want the entity names translated.
- Detail property type 95 (exercise duration) and the `batteryLevel` field come from the original contributor's device payloads. There are no public fixtures for them here, so the sensors appear only when the field is present.
- Propane monitors that report both `tuProperties` battery (type 17, existing *Battery Level*) and `apparatus` `batteryLevel` get both sensors. They come from different API fields, but they may describe the same battery.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01K2RdXuPegztHA1uMG2fbtk

---
_Generated by [Claude Code](https://claude.ai/code)_
