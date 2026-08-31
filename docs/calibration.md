# Position estimation and calibration

## The model

There are no limit switches. Position is a single float `g_position` in
`0.0 … 1.0`, kept in flash (`restore_value: true`) so it survives reboots.

Two measured constants drive everything:

| Global | Meaning |
|---|---|
| `g_cas_vysuv_ms` | Full travel time via `sw_open`, ending at position **0.0** |
| `g_cas_zasuv_ms` | Full travel time via `sw_close`, ending at position **1.0** |

Direction convention, used identically by `jazdi_na_doraz` and
`chod_na_poziciu`:

- `g_smer = -1` → moving towards `0.0`, via `sw_open`
- `g_smer = +1` → moving towards `1.0`, via `sw_close`

If the two scripts ever disagree on this, the extrapolation on stop counts in
the wrong direction and the position drifts every time you interrupt a move.

## Why the end reason is decided from time, not current

Both movement scripts finish by turning the outputs off and *then* deciding
whether they reached an end stop. That decision reads elapsed time, never
`id(prud)`:

```cpp
uint32_t elapsed = millis() - id(g_move_start);
bool dosiahnute = elapsed < (uint32_t)(id(max_chod_s).state * 1000);
```

Current collapses to zero the instant the PhotoMOS relays open, so reading it
after the fact is a race — it would report "end stop reached" for every move,
including timeouts.

The same reasoning applies in `chod_na_poziciu`: finishing *earlier* than the
planned duration means the actuator ran into a limit; finishing on time means
the target was reached.

## Interrupted moves are extrapolated, not frozen

Releasing a physical button or hitting stop calls `stop_vsetko`, which adds the
elapsed fraction of the travel to `g_position` before clearing the direction.
Without that, `g_position` would stay at its pre-move value and Home Assistant
would refuse to move the cover "back" — it would think it is already there.

The same fallback runs when a drive-to-endstop times out: the position is
estimated from elapsed time rather than falsely pinned to `0.0` or `1.0`.

## Calibration

The **Kalibrovať (obe strany)** button in HA runs `kalibruj`, three phases:

| Phase | Direction | Measures? | Why |
|---|---|---|---|
| 1/3 | `sw_open` → 0.0 | no | Start position unknown, so the time is meaningless |
| 2/3 | `sw_close` → 1.0 | yes | Starts from a known end stop — guaranteed full travel |
| 3/3 | `sw_open` → 0.0 | yes | Same, and leaves the sofa at the reference position |

This is why `jazdi_na_doraz` takes a `meraj_cas` parameter. Ordinary
open/close/button presses **must** pass `meraj_cas: false`: they can start from
any mid position, and a partial run would overwrite the full-travel time with a
too-short value, permanently skewing every later position command.

Measured times are only accepted when plausible (`> 1 s` and `< 30 s`).

Calibration does **not** run on boot. `g_position` is restored from flash, so a
normal reboot needs no movement. Run it by hand after installing the board, or
after a power cut that interrupted a movement.

## Tuning parameters

All four are `number` entities in HA, stored in flash — no reflash needed.

| Entity | Default | What it does |
|---|---|---|
| `Prah endstopu` | 0.30 A | Below this, the motor is considered stalled |
| `Rozbehová predĺžka` | 800 ms | Threshold ignored for this long after start |
| `Max. čas chodu` | 15 s | Hard timeout for any single movement |
| `Doba potvrdenia dorazu` | 200 ms | Current must stay below the threshold this long |

If a move stops immediately after starting, `Rozbehová predĺžka` is too short.
If it always runs into the timeout, `Prah endstopu` is too low.
