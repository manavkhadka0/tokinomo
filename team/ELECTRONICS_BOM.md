# Electronics BOM — Tokinomo (per unit)

Parts for **one** device. Sourcing is for Nepal / Kathmandu:
- **Local** = available in Kathmandu electronics shops or Daraz.
- **Import** = order from AliExpress / India (Robu) — carry spares, longer lead time.

> **Prototype phase:** buy enough for **5 units + spares** (mmWave, ESP32-S3,
> DAC, and flash are the risky/import items — order extras of these first).

---

## Compute

| # | Item | Suggested part | Qty | Purpose | Sourcing |
|---|---|---|---|---|---|
| 1 | Main MCU | **ESP32-S3 DevKit (N16R8 — 16 MB flash, 8 MB PSRAM)** | 1 | Wi-Fi, audio, motor + sensor control | Import (some local stock; verify S3 variant) |
| 2 | External SPI flash *(optional)* | 16–32 MB NOR flash + LittleFS | 0–1 | Extra audio storage **only if** onboard 16 MB isn't enough | Import |

> **Note:** the N16R8 already has **16 MB onboard flash** — likely enough for
> one clip and room for several more. Add the external chip only if the clip
> library grows. This can save cost per unit.

## Presence detection

| # | Item | Suggested part | Qty | Purpose | Sourcing |
|---|---|---|---|---|---|
| 3 | mmWave sensor | **HLK-LD2410** (24 GHz, presence + distance, UART) | 1 | Detect shoppers incl. stationary → dwell time | Import |

## Audio

| # | Item | Suggested part | Qty | Purpose | Sourcing |
|---|---|---|---|---|---|
| 4 | I²S DAC + amp | **MAX98357A** | 1 | Decode + amplify audio from the ESP32 | Import |
| 5 | Speaker | 4 Ω / 8 Ω, 3 W | 1 | Sound output | Local |

## Motion

| # | Item | Suggested part | Qty | Purpose | Sourcing |
|---|---|---|---|---|---|
| 6 | Servo | **MG90S** (metal gear) or MG996R for more torque | 1 | Move / grip the product | Local |
| 7 | Bulk capacitor | 1000 µF electrolytic | 1 | Absorb servo current spikes on 5 V rail | Local |

## Light effect (optional, cheap)

| # | Item | Suggested part | Qty | Purpose | Sourcing |
|---|---|---|---|---|---|
| 8 | Addressable LED | WS2812B (few pixels or a short strip) | 1 | Light-up attention effect | Local |

## Power

| # | Item | Suggested part | Qty | Purpose | Sourcing |
|---|---|---|---|---|---|
| 9 | Battery | 18650 Li-ion, **2S** (2×, optionally 2S2P for runtime) | 2–4 | Main power | Local |
| 10 | BMS | 2S protection board (balance + protect) | 1 | Battery safety | Local |
| 11 | Charger | CC/CV 2S (8.4 V) charging module | 1 | Charge pack from AC | Local / Import |
| 12 | AC adapter | 12 V, 2 A wall adapter | 1 | Mains input | Local |
| 13 | Buck regulator | MP1584 or LM2596 → 5 V | 1 | Clean 5 V rail | Local |
| 14 | Battery holder | 18650 holder (2S) | 1 | Hold cells | Local |

## Build / misc

| # | Item | Qty | Purpose | Sourcing |
|---|---|---|---|---|
| 15 | Matrix / perfboard | 1 | Prototype circuit (pre-PCB) | Local |
| 16 | Jumper wires + headers | set | Wiring | Local |
| 17 | Connectors (JST etc.) | set | Battery / servo / speaker leads | Local |
| 18 | Resistors, caps, LEDs | set | Support components | Local |
| 19 | Switch / power button | 1 | On/off | Local |

---

## Import shortlist (order first, with spares)
These are the long-lead / not-reliably-local items — get them moving on day 1:

1. **ESP32-S3 N16R8** boards
2. **HLK-LD2410** mmWave sensors
3. **MAX98357A** I²S audio amps
4. (Optional) external **SPI flash** chips

## Locally available (buy as needed)
Servos, speakers, 18650 cells + holders, BMS, chargers, AC adapters, buck
converters, perfboard, wires, connectors, passives, LEDs, switches.

---

## Cost note
Fill in per-unit BOM cost once quotes are in — **mmWave (LD2410)** and the
**battery pack** are the two biggest cost variables. Total these across the 5
prototype units to inform the go/no-go and the price you quote Xtreme per unit
(base = 1 clip; +clips = higher tier).
