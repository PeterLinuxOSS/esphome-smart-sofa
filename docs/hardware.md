# Hardware

## Overview

A small ESP32 board that sits **in parallel** with the sofa's original hand
controller. It never replaces it — the buttons keep their direct function, and
the ESP32 reads them through opto-isolators while driving the same two actuator
lines through PhotoMOS relays.

```
                 ┌──────────────── original hand controller ───────────────┐
                 │  IN1 (up)                                    IN2 (down) │
                 └────┬────────────────────────────────────────────┬───────┘
                      │                                            │
                 U2 PC817C                                    U4 PC817C
                      │ SIG1                                       │ SIG2
                      ▼                                            ▼
                   GPIO25 ◄──────── ESP32-WROOM-32-N4 ────────► GPIO26
                                     │            │
                        GPIO17 ──────┘            └────── GPIO16
                        R10 470R                          R11 470R
                          │                                  │
                     U5 AQY212GH                        U6 AQY212GH
                          │ OUT1                             │ OUT2
                          ▼                                  ▼
                  ┌──────────────── linear actuator ──────────────────┐

                        INA226 + 20 mΩ shunt on the actuator supply
                              I2C 0x40 → GPIO21 (SCL) / GPIO22 (SDA)
```

## Bill of materials

| Ref | Part | Notes |
|---|---|---|
| U1 | ESP32-WROOM-32-N4 | 4 MB flash, ESP-IDF framework |
| U2, U4 | PC817C | Opto-isolators, read the original buttons |
| U5, U6 | AQY212GH | PhotoMOS relays, drive the actuator lines |
| U3 | INA226 | Current/voltage monitor, I2C address `0x40` |
| Rs | 20 mΩ shunt | In the actuator supply line |
| R10, R11 | 470 Ω | PhotoMOS LED current limit |

<!-- TODO: complete BOM with values, footprints and an LCSC/Mouser column,
     exported from the EasyEDA project -->

## Pinout

| Signal | Default GPIO | Direction | Goes to |
|---|---|---|---|
| SIG1 | `GPIO25` | in, active low | U2 PC817C ← button IN1 |
| SIG2 | `GPIO26` | in, active low | U4 PC817C ← button IN2 |
| SOUT1 | `GPIO17` | out | R10 → U5 → OUT1 |
| SOUT2 | `GPIO16` | out | R11 → U6 → OUT2 |
| SCL | `GPIO21` | I2C | INA226 |
| SDA | `GPIO22` | I2C | INA226 |

All of these are substitutions, so a board with different routing only needs
its device config changed. On one of the author's four boards IN2 was rerouted
away from IO26, which is exactly why the button pins are parametrised.

## Board files

The board was designed in EasyEDA and is published on OSHWLab:

**https://oshwlab.com/rigopeter11/project_vgotgxfv**

From there you can open the schematic and PCB in the EasyEDA editor, clone the
project, or order the board directly through JLCPCB.

<!-- Worth adding here when there is time: a schematic PDF and a photo of the
     assembled board, so the repo is readable without an EasyEDA account. Also
     the fab settings used (layers, thickness, copper weight, surface finish)
     so the next order is reproducible, and a hardware licence stated separately
     from the firmware's MIT -- CERN-OHL-S or CC BY-SA are the usual picks. -->

## Choosing the current threshold

The end stop is detected as "current below `Endstop threshold` for at least
`Endstop confirmation time`". To find the right threshold, watch **Actuator current**
in Home Assistant during a full travel:

- Running under load, expect a steady draw well above the threshold.
- At the mechanical limit the value collapses toward zero.

Set the threshold roughly halfway between the two, and leave
`Startup grace period` long enough to cover the inrush and the initial slack —
during that window the threshold is ignored, otherwise every start would look
like an end stop.
