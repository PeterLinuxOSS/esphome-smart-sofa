# Position estimation and calibration

## The model

There are no limit switches. Position is a single float `g_position` in
`0.0 … 1.0`, kept in flash (`restore_value: true`) so it survives reboots.

Two measured constants drive everything:

| Global | Meaning |
|---|---|
| `g_extend_ms` | Full travel time via `sw_open`, ending at position **0.0** |
| `g_retract_ms` | Full travel time via `sw_close`, ending at position **1.0** |

Direction convention, used identically by `drive_to_endstop` and
`move_to_position`:

- `g_direction = -1` → moving towards `0.0`, via `sw_open`
- `g_direction = +1` → moving towards `1.0`, via `sw_close`

If the two scripts ever disagree on this, the extrapolation on stop counts in
the wrong direction and the position drifts every time you interrupt a move.

## Why the end reason is decided from time, not current

Both movement scripts finish by turning the outputs off and *then* deciding
whether they reached an end stop. That decision reads elapsed time, never
`id(current_a)`:

```cpp
uint32_t elapsed = millis() - id(g_move_start);
bool reached = elapsed < (uint32_t)(id(max_runtime_s).state * 1000);
```

Current collapses to zero the instant the PhotoMOS relays open, so reading it
after the fact is a race — it would report "end stop reached" for every move,
including timeouts.

The same reasoning applies in `move_to_position`: finishing *earlier* than the
planned duration means the actuator ran into a limit; finishing on time means
the target was reached.

## Interrupted moves are extrapolated, not frozen

Releasing a physical button or hitting stop calls `stop_all`, which adds the
elapsed fraction of the travel to `g_position` before clearing the direction.
Without that, `g_position` would stay at its pre-move value and Home Assistant
would refuse to move the cover "back" — it would think it is already there.

The same fallback runs when a drive-to-endstop times out: the position is
estimated from elapsed time rather than falsely pinned to `0.0` or `1.0`.

## Calibration

The **Calibrate (both ends)** button in HA runs `calibrate`, three phases:

| Phase | Direction | Measures? | Why |
|---|---|---|---|
| 1/3 | `sw_open` → 0.0 | no | Start position unknown, so the time is meaningless |
| 2/3 | `sw_close` → 1.0 | yes | Starts from a known end stop — guaranteed full travel |
| 3/3 | `sw_open` → 0.0 | yes | Same, and leaves the sofa at the reference position |

This is why `drive_to_endstop` takes a `measure` parameter. Ordinary
open/close/button presses **must** pass `measure: false`: they can start from
any mid position, and a partial run would overwrite the full-travel time with a
too-short value, permanently skewing every later position command.

Measured times are only accepted when plausible (`> 1 s` and `< 30 s`).

Calibration does **not** run on boot. `g_position` is restored from flash, so a
normal reboot needs no movement. Run it by hand after installing the board, or
after a power cut that interrupted a movement.

### Upgrading across a rename

ESPHome keys each `restore_value` global in NVS by a **hash of its id**, so
renaming one silently starts a fresh slot: the stored value is still in flash
but nothing looks for it, and the global comes up at `initial_value`. Nothing
warns about this — the config validates and the build succeeds.

That is what the v1 → v2 rename of `g_cas_vysuv_ms` → `g_extend_ms` did: every
board came back with the seed times instead of its measured ones. **Recalibrate
after upgrading across it.** Changing `extend_time_ms` and reflashing does not
help — by then the new slot holds a value, so `initial_value` is ignored.

The same applies to the tuning `number` entities, and it is easy to miss there:
if a stored value happens to equal the default, the reset is invisible. Check
the ones you deliberately tuned away from the default.

## Tuning parameters

All four are `number` entities in HA, stored in flash — no reflash needed.

| Entity | Default | What it does |
|---|---|---|
| `Endstop threshold` | 0.30 A | Below this, the motor is considered stalled |
| `Startup grace period` | 800 ms | Threshold ignored for this long after start |
| `Max runtime` | 15 s | Hard timeout for any single movement |
| `Endstop confirmation time` | 200 ms | Current must stay below the threshold this long |

If a move stops immediately after starting, `Startup grace period` is too short.
If it always runs into the timeout, `Endstop threshold` is too low.
