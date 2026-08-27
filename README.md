# Solar Power Monitoring System

A Raspberry Pi Pico-based monitoring system for tracking a solar installation, including:

- ☀️ Solar panel voltage, current, and power
- 🔋 Battery voltage, current, power, and estimated state of charge
- ⚡ Inverter AC voltage, current, power, and energy consumption
- 📊 Total energy production and consumption in Wh/kWh

## System Overview

```
                    ☀️ SOLAR PANELS
                           │
                  ┌────────┴────────┐
                  │                 │
          DC Voltage Sensor    ACS758-100A
                  │                 │
                  └────────┬────────┘
                           │
                           ▼
                    Solar Controller
                           │
                           ▼
                       🔋 BATTERY
                  ┌────────┴────────┐
                  │                 │
          DC Voltage Sensor    ACS758-100A
                  │                 │
                  └────────┬────────┘
                           │
                           ▼
                     3 kW INVERTER
                           │
                    ⚡ 230 V AC Output
                  ┌────────┴────────┐
                  │                 │
              ZMPT101B          ACS712-20A
              AC Voltage        AC Current
                  │                 │
                  └────────┬────────┘
                           ▼
                    ADS1115 ADC(s)
                           │
                           ▼
                   Raspberry Pi Pico
                           │
                    Wi-Fi / UART
                           │
                           ▼
               Raspberry Pi / Home Assistant
```

## Hardware

### Current Sensors

#### Solar Panel Current

**1× ACS758-100A**

Measures the DC current produced by the solar panels.

```
Solar Panels
     │
     ▼
ACS758-100A
     │
     ▼
Solar Controller
```

The sensor is Hall-effect based and can measure DC current.

A ±100 A sensor is larger than necessary for the current solar system but was selected so the same sensor can be used for both solar and battery monitoring.

#### Battery Current

**1× ACS758-100A**

Measures battery charging and discharging current.

```
Battery
   │
   ▼
ACS758-100A
   │
   ▼
Inverter / Solar Controller
```

The 3 kW inverter can theoretically draw approximately:

```
3000 W / 48 V ≈ 62.5 A
```

A ±100 A sensor provides sufficient headroom.

#### Inverter AC Current

**1× ACS712-20A**

Measures the AC current supplied by the inverter.

```
230 V AC
   │
   ▼
ACS712-20A
   │
   ▼
Load
```

For a 3 kW inverter operating at 230 V:

```
3000 W / 230 V ≈ 13 A
```

The 20 A version provides a suitable measurement range.

The ACS712 provides galvanic isolation between the current conductor and the low-voltage measurement electronics.

### Voltage Sensors

#### Solar Panel Voltage

**1× DC voltage sensor**

Measures the DC voltage produced by the solar array.

The required measurement range depends on:

- Panel configuration
- Number of panels connected in series
- Panel open-circuit voltage (Voc)

The sensor range must be greater than the maximum possible PV open-circuit voltage.

Example:

```
Solar Panels
     │
     ├──────────────► Solar Controller
     │
     ▼
DC Voltage Sensor
     │
     ▼
ADS1115
```

> Do not select the voltage sensor until the panel Voc and series/parallel configuration have been confirmed.

#### Battery Voltage

**1× DC voltage sensor**

Measures the voltage of the 48 V battery bank.

The sensor should safely support the maximum battery charging voltage, not just the nominal 48 V.

Recommended range: **0–100 V DC**

#### AC Voltage Measurement — ZMPT101B

**1× ZMPT101B**

Measures the 230 V AC output voltage of the inverter.

```
230 V AC
   │
   ▼
ZMPT101B
   │
   ▼
ADS1115
```

The ZMPT101B uses transformer isolation and provides a low-voltage analog signal suitable for measurement electronics.

### Analog-to-Digital Conversion — ADS1115

**2× ADS1115 16-bit ADC modules**

Each ADS1115 provides four analog channels.

Example channel allocation:

**ADS1115 #1**

| Channel | Sensor |
|---------|--------|
| A0 | Solar voltage |
| A1 | Solar current |
| A2 | Battery voltage |
| A3 | Battery current |

**ADS1115 #2**

| Channel | Sensor |
|---------|--------|
| A0 | Inverter AC voltage |
| A1 | Inverter AC current |
| A2 | Reserved |
| A3 | Reserved |

Both ADS1115 modules can communicate with the Raspberry Pi Pico using I²C.

## Measurements

### Solar Power

```
Solar Voltage × Solar Current = Solar Power
```

Example:

```
180 V × 5 A = 900 W
```

Energy production is calculated over time:

```
Wh = Power × Time
```

The system can track:

- Current solar power
- Solar Wh today
- Solar kWh this month
- Total solar energy produced

### Battery Monitoring

The battery monitor tracks:

- Voltage
- Charging current
- Discharging current
- Charging power
- Discharging power
- Estimated battery state of charge

Battery power:

```
Battery Voltage × Battery Current = Battery Power
```

Example:

```
52 V × 20 A = 1040 W
```

A positive current can represent charging, while a negative current can represent discharging.

#### Battery State of Charge

State of charge should primarily be calculated using current integration (coulomb counting):

```
Ah = Current × Time
```

The system should periodically synchronize or correct the estimated state of charge using known battery states or battery-management information where available.

Voltage alone should not be used as the primary method for estimating battery percentage while the battery is actively charging or discharging.

### Inverter Monitoring

The inverter output monitor measures:

- AC voltage
- AC current
- Real power
- Energy consumption

For accurate AC power measurement, voltage and current should be sampled and combined:

```
P = Average(V(t) × I(t))
```

Energy is then accumulated:

```
Wh += Power × Time
```

The system can report:

```
Inverter Voltage:     230 V
Inverter Current:     4.2 A
Inverter Power:       850 W
Energy Today:         4.3 kWh
```

## Shopping List

| Quantity | Component | Purpose |
|----------|-----------|---------|
| 1 | Raspberry Pi Pico | Main controller |
| 2 | ACS758-100A | Solar and battery DC current |
| 1 | ACS712-20A | Inverter AC current |
| 1 | ZMPT101B | Inverter AC voltage |
| 2 | ADS1115 | 16-bit analog-to-digital conversion |
| 1 | DC voltage sensor | Battery voltage |
| 1 | DC voltage sensor | Solar panel voltage |

## Final Measurement Architecture

```
SOLAR PANELS
├── DC Voltage Sensor
└── ACS758-100A
        │
        ▼
   Solar Power
        │
        ▼
   Solar Controller
        │
        ▼
BATTERY
├── DC Voltage Sensor
└── ACS758-100A
        │
        ▼
Battery Monitoring
        │
        ▼
INVERTER
        │
        ▼
230 V AC OUTPUT
├── ZMPT101B
└── ACS712-20A
        │
        ▼
ADS1115 ADC
        │
        ▼
Raspberry Pi Pico
        │
        ▼
Home Assistant / Raspberry Pi / API
```

## Safety

⚠️ The solar array and inverter contain potentially dangerous DC and AC voltages.

- Never connect high voltage directly to a Raspberry Pi Pico.
- Ensure all voltage measurement circuits are properly rated for the maximum possible voltage.
- Use appropriate fuses and enclosures.
- Keep the low-voltage Pico electronics physically separated from mains wiring.
- Do not prototype exposed 230 V wiring on a breadboard.
- Confirm the exact maximum PV open-circuit voltage before selecting the solar voltage sensor.
