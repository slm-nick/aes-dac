# AES-ADC — Cost Roll-up (budgetary estimate)

Companion to `aes-adc-bom.md`. **These are rough budgetary estimates** (USD, ~2025–26 distributor pricing, Digi-Key/Mouser-class), **not quotes**. Use for go/no-go and pricing direction; re-cost against live distributor carts before ordering. Two columns: **Proto** (qty ~1–10, single-piece pricing) and **Qty 100** (small production).

Excludes (listed separately in §3): PCB assembly labor, enclosure, NRE/tooling, test/programming labor, shipping, tariffs.

---

## 1. BOM cost by subsystem

| Item | MPN | Proto ea. | Qty-100 ea. |
| ---- | --- | --------: | ----------: |
| **Converter & digital** | | | |
| Stereo ADC | PCM4202IDB | 9.00 | 7.00 |
| AES transmitter | DIT4192IDB | 7.00 | 5.50 |
| AES Tx transformer | PE-65612NL / SC937-02 | 5.00 | 4.00 |
| **Clock + word-clock input** | | | |
| Clock generator | CS2000CP-CZZ | 5.00 | 4.00 |
| RefClk crystal 24.576 MHz | — | 0.50 | 0.30 |
| 75 Ω PCB BNC | Amphenol/TE | 3.00 | 2.50 |
| WCLK comparator | TLV3501AIDBV | 3.00 | 2.30 |
| Term/bias passives | — | 0.20 | 0.15 |
| **MCU & flashing** | | | |
| MCU | RP2040 | 1.00 | 0.70 |
| QSPI flash 16 MB | W25Q128 | 1.00 | 0.80 |
| 12 MHz crystal | — | 0.40 | 0.25 |
| USB-C receptacle | GCT USB4085 | 0.60 | 0.40 |
| Tact switches ×2 (BOOTSEL/RUN) | — | 0.20 | 0.15 |
| Rate DIP switch | — | 0.70 | 0.50 |
| Status LEDs ×3 | — | 0.30 | 0.20 |
| **Analog input (2 ch)** | | | |
| Opamps ×2 | OPA1612AIDR | 7.00 | 6.00 |
| Precision 0.1 % resistors (attenuator/diff) | — | 1.50 | 1.00 |
| C0G/NP0 AAF caps | — | 0.50 | 0.40 |
| **Power & supervisor** | | | |
| LDOs ×2 (+5 V / +3.3 V) | TPS7A4901 | 3.60 | 3.00 |
| Clean clock LDO (optional) | TPS7A4901 | 1.80 | 1.50 |
| Reset supervisor | APX803S05 | 0.40 | 0.30 |
| Barrel jack 9 V | DCJ200-10-A | 0.80 | 0.60 |
| USB VBUS OR diode | SOD-123 | 0.15 | 0.10 |
| **Connectors & bulk passives** | | | |
| XLR ×3 (2× NC3FAH1 in, 1× NC3MAH out) | Neutrik | 7.50 | 6.00 |
| Bulk + decoupling caps, ferrites, misc R/C | — | 3.50 | 2.50 |
| **BOM subtotal (parts)** | | **≈ 62.65** | **≈ 49.65** |

> Rounded: **~$63 / board proto**, **~$50 / board at qty 100**.

## 2. Board total (parts + bare PCB)

| | Proto (~qty 10) | Qty 100 |
| - | --------------: | ------: |
| BOM parts | 63 | 50 |
| Bare PCB (4-layer, ~AES-DAC size) | 8 | 4 |
| **Board total** | **≈ $71** | **≈ $54** |

(4-layer assumed — the AES-DAC gerbers show In1/In2 inner copper layers; same stackup carries over.)

## 3. Costs not in the board total

| Item | Rough range | Note |
| ---- | ----------- | ---- |
| PCB assembly (turnkey contract) | ~$10–25/board @ qty 100; higher at proto | QFN-56 + SSOP + MSOP + USB-C; SMT-heavy, one transformer/THT |
| Enclosure (pedal-style diecast, e.g. Hammond) | ~$8–15 | + drilling/labeling for XLR/BNC/USB/DIP |
| NRE / tooling | one-time | Stencil ~$20–40; assembly setup; firmware dev time |
| Test + programming labor | per unit | RP2040 flash is drag-drop UF2 (cheap); functional/audio test is the real cost |
| Shipping, tariffs | varies | |

## 4. Where the money is — and the clock-feature delta

- **Biggest BOM line items:** XLR connectors (~$7.5), the two OPA1612 (~$7), PCM4202 ADC (~$9), DIT4192 (~$7), AES transformer (~$5). Connectors + opamps + the two converters are >60 % of parts cost.
- **Cost of the word-clock + MCU capability** (CS2000 + RefClk xtal + BNC + comparator + RP2040 + flash + 12 MHz xtal + USB-C + buttons + DIP + LEDs) ≈ **$15.7 proto / $12.5 qty-100**. That is the price of "slave to house clock + standalone master + field-flashable firmware" versus a fixed-clock design.
- **Cheap trims if needed:** drop the optional clean-clock LDO (−$1.5–1.8); smaller flash W25Q16 (−$0.5); 74AHC14 instead of TLV3501 comparator (−~$2, at some jitter cost, **D7**).

## 5. Positioning

At **~$50–55/board parts+PCB at qty 100** (before assembly/enclosure), the design stays consistent with the project's "affordable vs. high-end studio gear" goal from the README. A reasonable loaded cost-of-goods target (parts + PCB + assembly + enclosure) lands roughly **$80–110/unit at qty 100**, leaving room for a sane margin well under typical commercial AES converter pricing.

## 6. Confidence / refresh triggers

- Estimates assume parts in stock at list-ish pricing; **CS2000, PCM4202, and the AES transformer are the volatile lines** (specialty audio/clock parts) — confirm stock + price first.
- Re-cost once the **DIT4192 orderable suffix** and **Tx transformer MPN** are locked (BOM §"Items to finalize").
