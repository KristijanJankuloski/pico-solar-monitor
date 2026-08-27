# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository is currently in the **planning/documentation stage** — it contains no source code yet, only `README.md` (the hardware and measurement design doc) and `LICENSE`. There is no build, lint, or test tooling to run. The `.gitignore` is Python-oriented, implying the firmware (running on the Raspberry Pi Pico) and/or a companion backend service will be written in Python (likely MicroPython/CircuitPython for the Pico itself, and standard CPython for any Raspberry Pi / Home Assistant integration side).

When code is added, update this file with actual build/flash/test commands (e.g. how to deploy MicroPython to the Pico, how to run any host-side backend, how to test sensor calculation logic).

## System architecture

This is a Raspberry Pi Pico-based solar power monitoring system. The physical measurement pipeline, as designed in `README.md`, is:

```
Solar Panels ──┬── DC Voltage Sensor ──┐
               └── ACS758-100A ────────┤
                                       ▼
                              Solar Controller
                                       │
                                       ▼
Battery (48V) ──┬── DC Voltage Sensor ──┐
                └── ACS758-100A ────────┤
                                        ▼
                                  3 kW Inverter
                                        │
                                        ▼
                        230 V AC Output ──┬── ZMPT101B (AC voltage)
                                          └── ACS712-20A (AC current)
                                        │
                                        ▼
                              2× ADS1115 16-bit ADC (I²C)
                                        │
                                        ▼
                              Raspberry Pi Pico
                                        │
                              Wi-Fi / UART
                                        │
                                        ▼
                    Raspberry Pi / Home Assistant (consumer/API)
```

Key domain concepts future code needs to implement correctly:

- **Three measurement domains**, each with its own voltage + current sensor pair feeding into power calculations: solar (DC), battery (DC), inverter output (AC).
- **ADS1115 channel mapping** (as planned): ADC #1 → solar voltage/current, battery voltage/current; ADC #2 → inverter AC voltage/current (2 channels reserved). Both ADCs share the Pico's I²C bus.
- **Power calculations**:
  - DC power (solar, battery): `Power = Voltage × Current`.
  - AC power (inverter): must be computed as `P = Average(V(t) × I(t))` over sampled waveform points, not a simple V×I of instantaneous or RMS readings taken independently — voltage and current must be sampled together.
- **Energy accumulation**: `Wh += Power × Time`, tracked at multiple rollups (today / this month / lifetime total) per domain.
- **Battery state of charge**: primary method is coulomb counting (`Ah = Current × Time`), periodically corrected/resynced against known battery states. Voltage-based SoC estimation is explicitly *not* to be used as the primary method while the battery is actively charging or discharging — it's only valid as a resting-voltage correction input.
- **Current sign convention**: for the battery, positive current = charging, negative = discharging. Power/energy logic must preserve this sign rather than taking absolute values.
- **Sensor selection is configuration-sensitive**: DC voltage sensor ranges depend on actual panel Voc/series-parallel configuration and battery max charge voltage — these aren't hardcoded constants to assume, they're deployment-specific calibration values.

## Safety-sensitive context

This project involves real mains AC (230 V) and high-current DC wiring alongside low-voltage microcontroller electronics. If asked to help with wiring, sensor placement, or circuit design (as opposed to firmware/software logic), keep in mind the constraints already called out in `README.md`: galvanic isolation is required between mains/high-current circuits and the Pico (hence ACS712/ACS758 Hall-effect isolation and ZMPT101B transformer isolation), and high voltage must never connect directly to the Pico's GPIO/ADC inputs.
