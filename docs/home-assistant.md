# Home Assistant integration

## Entities

Every name below is a substitution, so the dashboard can be in any language
without forking — see [Renaming](#renaming) before changing one. Home Assistant
prefixes them all with `friendly_name`.

### Control

| Entity | Substitution | Domain | Notes |
|---|---|---|---|
| `Sofa` | `name_cover` | `cover` | `device_class: awning`, full 0–100 % position support |
| `Button lock` | `name_button_lock` | `switch` | ON = the physical buttons do nothing; HA still works |
| `Calibrate (both ends)` | `name_calibrate` | `button` | Runs the three-phase calibration |

### State

| Entity | Substitution | Domain | Notes |
|---|---|---|---|
| `Position` | `name_position` | `sensor` | 0–100 %, same value as the cover |
| `Sofa moving` | `name_moving` | `binary_sensor` | `device_class: moving` |
| `Actuator current` | `name_current` | `sensor` | A, throttled to 1 s with a 0.01 delta |
| `Supply voltage` | `name_voltage` | `sensor` | V |
| `Power` | `name_power` | `sensor` | W |
| `Button up` / `Button down` | `name_btn_up` / `name_btn_down` | `binary_sensor` | The hand controller's two inputs |

### Diagnostic and config

`Measured extend time` / `Measured retract time` (`name_extend_time`,
`name_retract_time` — the calibration results), `WiFi signal`
(`name_wifi_signal`), `Uptime` (`name_uptime`), and the four tuning numbers
described in [`calibration.md`](calibration.md)
(`name_endstop_thr`, `name_startup_grace`, `name_max_runtime`,
`name_endstop_debounce`).

## Device info

Without a `project:` block a board shows up as a bare **esp32 by Espressif**.
The package sets one, and Home Assistant renders it like this:

| Field | Comes from | Example |
|---|---|---|
| Manufacturer | `project_name`, before the dot | `PeterLinuxOSS` |
| Model | `project_name`, after the dot | `Smart Sofa` |
| Firmware | `project_version` + ESPHome version | `2.1.0 (ESPHome 2026.9.0-dev)` |

`project_name` must contain **exactly one dot** — ESPHome rejects the config
otherwise. Both are substitutions, so a fork can put its own name there:

```yaml
substitutions:
  project_name:    "YourName.Sofa Controller"
  project_version: "1.0.0"
```

This is metadata only. It never touches `unique_id`, so unlike the entity names
it is safe to change on a running installation.

## Position semantics

Home Assistant's convention is `1.0 = open`, `0.0 = closed`, and calibration
defines the retracted end as the `0.0` reference. So in the cover:

- `open_action` drives with `towards_zero: false` (towards 1.0)
- `close_action` drives with `towards_zero: true` (towards 0.0)

The internal `towards_zero` flag names the **output relay**, not the HA direction.
It is the single most confusing thing in this project — check it before
"fixing" a direction bug.

## The reported position is an estimate

It comes from elapsed time, not an encoder. It is accurate enough for scenes
and sliders, but:

- It drifts slightly on every interrupted movement.
- After a power cut *during* a movement, the stored value is stale — drive to
  either end and it re-references itself.
- Reaching either end stop resets the drift to zero, so anything that
  occasionally drives fully open or fully closed is self-correcting. Commanding
  0 % or 100 % counts: both go to the physical end stop, not to a timer.

Two diagnostic sensors show whether the timing the estimate rests on has
settled: **Last travel sample** is the raw measurement that went in most
recently, **Travel samples** is how many have been accepted. A stored travel
time that barely moves while the count keeps rising is one you can trust.

## Grouping several units

One piece of furniture may hold more than one actuator. Group them with a
`cover` group helper so scenes and voice commands address the furniture rather
than the motors:

```yaml
cover:
  - platform: group
    name: Corner sofa
    entities:
      - cover.smart_sofa1_gauc
      - cover.smart_sofa2_gauc
      - cover.smart_sofa4_gauc
```

The group forwards `set_cover_position`, so a single call moves all of them.

## Renaming

Set the names **before the first flash**, in whatever language you want:

```yaml
substitutions:
  name_cover:    "Canapé"
  name_position: "Position"
  # …
```

Changing one afterwards is not a rename. ESPHome derives each entity's
`unique_id` from `device_name` and the entity's `name:`, so a changed name
registers a **new** entity — the old one stays behind as an orphan, and every
automation, group, dashboard card and voice alias still points at the dead ID.
An explicit `id:` does not help; it is internal to ESPHome and never reaches
Home Assistant.

If you must rename on a live system, expect to fix up the references by hand,
or rename the entity in Home Assistant's UI instead and leave the YAML alone.
