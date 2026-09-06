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

## How the travel times are learned

The board measures itself. Every run that is *provably* a full travel updates
the stored time; there is exactly one place in the firmware that writes
`g_extend_ms` / `g_retract_ms`, at the end of `drive_to_endstop`.

A run counts as a sample only when all of these hold:

- it set off from the **confirmed opposite end stop** (`g_start_endstop`),
- it **arrived** at the target end stop, without hitting `Max runtime`,
- nothing interrupted it.

`g_at_endstop` carries that confirmation. Only a detected end stop sets it, and
it is deliberately **not** restored across reboots — after a restart the board
has not seen an end stop yet, so the first move teaches nothing.

### What the sample actually measures

The timestamp used is `g_below_thr_since`, the moment the current first dropped
below the threshold — not the moment the debounce confirmed it. That removes
`Endstop confirmation time` from the measurement **exactly**, rather than
approximately.

What remains is the group delay of the median filter on `current_a`: a window of
5 at 100 ms crosses roughly two intervals after the real edge, so a constant
`FILTER_LAG_MS = 200` is subtracted. **If you change either the window size or
the update interval, change that constant with them.**

Before this, the stored time included both delays — about 400 ms of detection
lag baked into a 9 s travel. Every partial move then overshot by that fraction.

### Averaging, and rejecting nonsense

```
stored += 0.3 * (sample - stored)      // only if sample is within ±15 % of stored
```

A sample further than 15 % from the stored value is logged and thrown away. This
is what keeps a false end stop mid-travel — a current dip, a supply sag, a
frozen INA226 reading — from being written in as a full travel. Real-world
scatter between runs is 1–4 %, so the band is wide enough never to fire on an
honest measurement.

When a direction's seeded flag is `false` the stored value is still the seed from
the device file, or calibration dropped it. Either way it was never measured, so
there is nothing to range-check against and the first clean sample in that
direction is taken at face value.

The flag is **per direction** (`g_seeded_extend`, `g_seeded_retract`). It used to
be one shared flag, and that quietly broke calibration: the flag was cleared
once, so only whichever direction was measured first got a clean value while the
other kept averaging into the very number calibration was there to discard.

### The 0 % and 100 % commands do not run on the clock

`move_to_position` hands both extremes to `drive_to_endstop`. Planning them from
the stored time would stop the moment the plan says so, so a travel *longer*
than the stored one could never be observed and the learned value could only
ever drift downwards. Driving to the real end stop also re-references
`g_position`, which is where accumulated estimation drift gets cleaned up.

### Watching it settle

Two diagnostic sensors exist for exactly this: **Last travel sample** (the raw
measurement that went in) and **Travel samples** (how many have been accepted).
A stored time that barely moves while the count keeps climbing is a settled one.
That is the difference between a number you can trust and a number that appeared
out of nowhere.

## The Calibrate button

**Calibrate (both ends)** is not a separate way of measuring. It clears both
seeded flags and the sample counter, seeks the reference end stop, then forces
**three round trips** — six clean full travels — and lets the ordinary learning
above digest them. One code path, no second way to write the times.

Use it after installing a board, or when the mechanics changed so much that
normal runs are being rejected as out of range. Clearing the flags is what lets
it escape that: the old value is precisely what would reject the new, correct
samples.

The counter is reset as well, so **Travel samples** answers "how many samples
back do the current times go" rather than counting up forever across
calibrations. Three samples per direction: the first at face value, the next two
averaged in at alpha 0.3.

While it runs, `g_calibrating` makes the hand controller and Home Assistant's
open/close/position commands do nothing. Releasing a physical button, or the
cover's stop action, still aborts it — nobody sitting on the sofa at 02:00 is
trapped by a calibration run.

Calibration does **not** run on boot. `g_position` is restored from flash, so a
normal reboot needs no movement.

## Pushing into an end stop does nothing

If the sofa is already at an end stop and you ask for that same end stop again,
`drive_to_endstop` returns immediately without energising the actuator. It
cannot move past its own internal limit switch anyway, and without the early
return the relay was held on for grace + debounce — about a second of nothing,
every press.

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
