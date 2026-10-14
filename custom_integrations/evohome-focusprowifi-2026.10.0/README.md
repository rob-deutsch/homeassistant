# Evohome FocusProWifi for Home Assistant 2026.10.0

This custom integration adds the schedule-free FocusProWifiRetail thermostat described in [Home Assistant core PR #94805](https://github.com/home-assistant/core/pull/94805). It is based on the evohome integration shipped in **Home Assistant Core 2026.10.0**, with its original `evohome-async==2.1.0` dependency. The custom integration version is `2026.10.0.1`.

## Installation

1. Back up your Home Assistant configuration and any existing `custom_components/evohome` directory. Keep the old component backup **outside** `custom_components` so Home Assistant cannot load it as another integration.
2. Copy this package's entire `custom_components/evohome` directory into your Home Assistant configuration directory. The resulting path should be `/config/custom_components/evohome/manifest.json` on a typical HA OS/container installation. Replace the old component directory after backing it up; do not mix old and new files. Do not put the outer package folder inside `custom_components`.
3. Keep your existing `evohome:` YAML configuration, credentials, `location_idx`, and scan interval. No new credentials, YAML options, or HACS setup are needed.
4. Check Home Assistant's configuration, then restart Home Assistant Core. An expected startup warning says you are using the custom evohome integration.
5. Confirm that the existing thermostat climate entity is present. It should expose Off/Heat, current temperature, target temperature, and a 0.5°C target step. Test setting a temperature and turning heating off/on, then verify the change on the physical thermostat.

The controller's existing unique ID and location-based name are retained from the original FocusPro branch. This allows HA's existing entity registry and automations to keep using the same entity. If you previously ran stock evohome, its separate zone entity may remain as an unavailable registry entry; the new component creates one active climate entity for this thermostat.

To roll back, stop HA, replace the new `custom_components/evohome` directory with your backup (or remove it to use the built-in integration), and restart. Do not delete `.storage/evohome` or other HA state as part of installation or rollback.

## Behavior

- A single combined climate entity is used for a FocusProWifiRetail system with one zone and no FollowSchedule capability. Other systems retain the stock 2026.10.0 behavior.
- Heating on/off changes the controller's Heat/Off mode; it does not simulate off by setting a frost temperature.
- Temperature changes send permanent holds, matching the original FocusPro branch. They never request a nonexistent schedule.
- `climate.set_temperature`, `climate.set_hvac_mode`, `climate.turn_on`, `climate.turn_off`, and `homeassistant.update_entity` work with the combined entity.
- Targeted `evohome.set_system_mode` works. `evohome.set_zone_override` supports permanent holds when `duration` is omitted; its setpoint is limited to the thermostat's range.
- Timed overrides, including duration 0 (“until next switchpoint”), and `evohome.clear_zone_override` return a validation error because this thermostat cannot follow a schedule.
- Reset-to-schedule buttons are omitted for this thermostat. Live controller and zone status remain available in the entity's `status` attribute, including temperature, setpoint, and faults.
- Reported temperature is unknown when its sensor is unavailable. A failed cloud refresh marks the entity unavailable.

This package supports the **heating-only, schedule-free** device in the PR: minimum 4.5°C, maximum 32°C, 0.5°C resolution, identical system and zone IDs. Cooling and fan controls are not implemented. Compatibility was tested on Core 2026.10.0; other Core versions require separate validation.

## Validation and source changes

The PR's published configuration/status fields were reproduced in synthetic fixtures with fake IDs and account details. The original 2026.10.0 integration reproduced a missing-schedule failure. The patched integration passed **154 tests and 144 existing snapshots**, including climate actions, permanent holds, controller services, unsupported schedule actions, sensor/cloud unavailability, supported-mode filtering, and custom integration discovery/loading.

Ruff checks/formatting, Pylint, and Mypy passed for the changed implementation files. The pre-commit wrapper could not run in the source archive because it has no Git metadata; the relevant tools were run directly. Vendor authentication and physical hardware were not tested; API operations were mocked.

`implementation.patch` contains the changes and tests relative to Home Assistant's `2026.10.0` tag. To reproduce in a separate HA core 2026.10.0 development checkout, apply that patch, install its documented development dependencies, generate evohome translations, and run:

```sh
export EVOHOME_PACKAGE_ROOT=/absolute/path/to/evohome-focusprowifi-2026.10.0
python3 -m script.translations develop --integration evohome
uv run --no-sync pytest -q --timeout=60 tests/components/evohome
```

Home Assistant source and the custom integration are licensed under Apache 2.0; see `LICENSE.md`.
