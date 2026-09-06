# esphome-smart-sofa

ESPHome firmware and open hardware for motorising a recliner sofa: a custom
ESP32 board wired in **parallel** with the original hand controller, exposing
the sofa to Home Assistant as a `cover` with a real 0–100 % position slider —
**without adding any limit switches**.

Firmware here, [board on OSHWLab](https://oshwlab.com/rigopeter11/project_vgotgxfv).

Position is estimated from travel time, and the end stops are detected from the
actuator's current draw measured by an INA226. The original buttons keep
working, and pressing them updates the position estimate too.

<!-- TODO: add photo of the assembled board in the sofa -->

## Why

Off-the-shelf sofa actuators give you two wires and two buttons. There is no
position feedback, so a naive ESPHome config can only do "up" and "down". This
project adds:

- **Position** — a `cover` with a slider, so scenes can set 35 %, 45 %, …
- **End stop detection from current** — when the actuator hits its mechanical
  limit the motor stalls and the current collapses; that is the end stop.
- **Self-calibration** — one button in HA drives both directions and measures
  the real full-travel times. No hard-coded constants to maintain.
- **Physical buttons preserved** — opto-isolated, and a HA-controllable lock.
- **Interlock** — hardware-adjacent software interlock, never both directions on.

## Repository layout

| Path | What |
|---|---|
| `packages/smart-sofa.yaml` | The whole firmware, as an ESPHome package |
| `example/smart-sofa.yaml` | A minimal device config that includes it |
| `docs/hardware.md` | Board, wiring and pinout |
| `docs/calibration.md` | How calibration and position estimation work |
| `docs/home-assistant.md` | Entities exposed, and how to use them |

The PCB is published on OSHWLab — schematic, board and a JLCPCB order button:
**https://oshwlab.com/rigopeter11/project_vgotgxfv**

## Usage

Create one small config per sofa and pull in the package:

```yaml
substitutions:
  device_name:   smart-sofa1
  friendly_name: Smart-Sofa1
  pin_btn_open:  "GPIO25"
  pin_btn_close: "GPIO26"

packages:
  sofa: github://PeterLinuxOSS/esphome-smart-sofa/packages/smart-sofa.yaml@v3.1.0

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "${friendly_name} Fallback Hotspot"
    password: !secret ap_password

captive_portal:
```

Pin the package to a **tag**, not `@main` — otherwise a commit here silently
changes the firmware of every board on its next build.

`wifi:`, `api:` and `ota:` deliberately stay in the device config: ESPHome
cannot resolve `!secret` from inside a remote package.

### Substitutions

| Name | Default | Meaning |
|---|---|---|
| `device_name` | `smart-sofa` | ESPHome node name |
| `friendly_name` | `Smart Sofa` | Prefix for entity names in HA |
| `pin_btn_open` | `GPIO25` | Button IN1, through PC817C |
| `pin_btn_close` | `GPIO26` | Button IN2, through PC817C |
| `pin_out_open` | `GPIO17` | PhotoMOS OUT1 |
| `pin_out_close` | `GPIO16` | PhotoMOS OUT2 |
| `pin_scl` / `pin_sda` | `GPIO21` / `GPIO22` | I2C to the INA226 |
| `ina_address` | `0x40` | INA226 address |
| `shunt_ohm` | `0.02` | Shunt resistor value |
| `max_current` | `4.0` | INA226 full-scale current, in A |
| `extend_time_ms` | `9000` | Seed full-extend time (calibration overwrites it) |
| `retract_time_ms` | `10000` | Seed full-retract time (calibration overwrites it) |
| `project_name` | `PeterLinuxOSS.Smart Sofa` | Device info in HA; split on the dot into manufacturer and model |
| `project_version` | package version | Shown as `x (ESPHome y)` |

The two travel times are only used on a **first** boot. They are stored in flash
with `restore_value`, so once calibration has run the substitutions no longer
matter.

Every entity name is a substitution too — `name_cover`, `name_position`,
`name_current`, `name_calibrate` and so on, listed in
[`docs/home-assistant.md`](docs/home-assistant.md). The defaults are English;
override them to put the dashboard in another language without forking:

```yaml
substitutions:
  name_cover:    "Canapé"
  name_current:  "Courant"
```

**Set them before the first flash.** ESPHome derives each entity's `unique_id`
from the name, so changing one later registers a *new* entity rather than
renaming the old one.

## Safety

This drives mains-adjacent motor wiring and a piece of furniture that moves
with enough force to trap fingers. The board switches the **low-voltage
actuator side** through PhotoMOS relays, not mains. Read `docs/hardware.md`
before building one, keep the original controller's power supply, and do not
remove the actuator's own thermal protection.

Provided as-is, with no warranty. See `LICENSE`.

## Status

Running on four boards in three pieces of furniture since mid-2026.
