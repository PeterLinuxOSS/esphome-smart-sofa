# Home Assistant integration

## Entities

Names below are the Slovak defaults; Home Assistant prefixes them with
`friendly_name`.

### Control

| Entity | Domain | Notes |
|---|---|---|
| `Gauč` | `cover` | `device_class: awning`, full 0–100 % position support |
| `Zámok tlačidiel` | `switch` | ON = the physical buttons do nothing; HA still works |
| `Kalibrovať (obe strany)` | `button` | Runs the three-phase calibration |

### State

| Entity | Domain | Notes |
|---|---|---|
| `Pozícia` | `sensor` | 0–100 %, same value as the cover |
| `Gauč sa hýbe` | `binary_sensor` | `device_class: moving` |
| `Prúd aktuátora` | `sensor` | A, throttled to 1 s with a 0.01 delta |
| `Napájacie napätie` | `sensor` | V |
| `Príkon` | `sensor` | W |

### Diagnostic and config

`Zmeraný čas vysuv` / `Zmeraný čas zasuv` (the calibration results),
`WiFi signál`, `Uptime`, and the four tuning numbers described in
[`calibration.md`](calibration.md).

## Position semantics

Home Assistant's convention is `1.0 = open`, `0.0 = closed`, and calibration
defines the retracted end as the `0.0` reference. So in the cover:

- `open_action` drives with `smer_open: false` (towards 1.0)
- `close_action` drives with `smer_open: true` (towards 0.0)

The internal `smer_open` flag names the **output relay**, not the HA direction.
It is the single most confusing thing in this project — check it before
"fixing" a direction bug.

## The reported position is an estimate

It comes from elapsed time, not an encoder. It is accurate enough for scenes
and sliders, but:

- It drifts slightly on every interrupted movement.
- After a power cut *during* a movement, the stored value is stale — recalibrate.
- Reaching either end stop resets the drift to zero, so anything that
  occasionally drives fully open or fully closed is self-correcting.

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

## Note on renaming

Entity IDs are derived from `device_name` and the entity `name:`. Changing
either renames every entity of that board and silently breaks any automation,
group, dashboard card or voice alias referring to the old ID. Decide on the
names before the first flash.
