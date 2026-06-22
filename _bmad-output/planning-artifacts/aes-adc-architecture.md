# AES-ADC — Architecture Design

**Analog → AES converter (the reverse of AES-DAC).** Converts 2 channels of balanced analog line audio (+4 dBu) into a single 110 Ω AES/EBU output, clock-slaved to an external word clock so its output matches the receiving equipment.

| Field | Value |
| ----- | ----- |
| Date | 2026-06-22 |
| Author | Nick |
| Status | Draft architecture (pre-schematic) |
| Sibling project | `hw/AES-DAC` (AES → analog), reused where possible |
| Direction | Analog in → ADC → AES transmitter → 110 Ω XLR out |

---

## 1. Requirements

| # | Requirement | Source |
| - | ----------- | ------ |
| R1 | 2 channels (stereo) balanced analog input, +4 dBu nominal pro line level, XLR | User + sibling product convention |
| R2 | Single AES3/EBU output, 110 Ω, transformer-coupled, XLR | User |
| R3 | Sample rates 44.1 / 48 / 88.2 / 96 kHz, 24-bit | User decision |
| R4 | Clock: **slave to external 75 Ω word clock** when present (match receiving gear); **internal crystal master** fallback — **standalone operation is a required mode** | User decision |
| R5 | Share elements with AES-DAC where sensible; component swaps acceptable | User |
| R6 | 9 V external supply (guitar-pedal style), as per sibling | Sibling convention |
| R7 | **Full-scale alignment: 0 dBFS = +24 dBu** (SMPTE); +4 dBu nominal → −20 dBFS | User decision |
| R8 | On-board **MCU**, Arduino / PlatformIO-compatible, with **firmware flashing built into the board** (no external programmer) | User decision |

---

## 2. System block diagram

```
                +4dBu bal.                                                   I2S                       AES3
  XLR L ──► [attenuator + diff receiver] ──┐                          ┌──────────────┐         ┌──────────┐
            (OPA1612, anti-alias)          ├──► [ PCM4202 ADC ]──DATA──► DIT4192 AES  ├──TX±──► │ 110Ω xfmr│──► XLR AES out
  XLR R ──► [attenuator + diff receiver] ──┘     (I2S master)   ─BCK/LRCK►  Transmitter        └──────────┘
            (OPA1612, anti-alias)                    ▲    ▲          (I2S slave)
                                                     │    │ MCLK         ▲ MCLK
                                                     │    └──────┬───────┘
                                                  BCK/LRCK       │
                                                            ┌────┴─────────┐
   75Ω BNC WCLK in ─► [term + comparator]──CLK_IN──────────► CS2000-CP PLL │──CLK_OUT (MCLK = 256·Fs)
                       (squaring to CMOS)                    │  clock gen  │
                                                  XTAL ──────► REF_CLK     │
                                              (24.576 MHz timebase)        │
                                                            └────┬─────────┘
                                                            I2C  │ (config / mode / lock)
                                                       ┌─────────┴─────────┐
                                       USB-C ──────────►   RP2040 MCU       │── lock / rate LEDs
                                  (flash + comms)          (Arduino/PIO)    │── rate-select DIP switch
                                  BOOTSEL + RUN btns ──────►                │── DIT4192 channel-status (I2C)
                                                       └───────────────────┘
            Power: 9V ─► TPS7A49 LDOs ─► +5V analog, +3.3V digital, (+3.3V clean clock rail);  USB VBUS OR-ed for flashing
```

---

## 3. Signal chain detail

### 3.1 Analog input stage (NEW — replaces the DAC output buffer)
- **Balanced receive + scale.** Each channel: a balanced/differential input stage built around **OPA1612** (reused part) that receives the +4 dBu balanced signal and scales it to the ADC's full-scale input. Alignment is fixed at **0 dBFS = +24 dBu** (R7): +24 dBu ≈ 12.3 Vrms (≈ 17.4 Vpk) must map to the PCM4202 full scale (≈ 2.8 Vrms differential), i.e. roughly **−12 to −13 dB of attenuation**, set by a precision resistive balanced attenuator **ahead of the opamp** so peaks never exceed the +5 V rail. +4 dBu nominal then sits at −20 dBFS (20 dB headroom). See Decision D1.
- **Anti-alias filter.** Low-order analog AAF ahead of the delta-sigma ADC (the PCM4202's decimation filter does the heavy lifting; the analog filter only needs to suppress energy near the modulator rate). Implementable with the same OPA1612 stage.
- **ADC differential drive.** PCM4202 has differential inputs; the OPA1612 stage drives VIN+/VIN− around the ADC's common-mode reference.

### 3.2 ADC — PCM4202 (NEW — replaces PCM1789 DAC)
- 24-bit, up to 216 kHz, **118 dB** dynamic range, differential inputs, **master or slave** serial port, I2S/LJ/RJ formats. Burr-Brown/TI — same family as the existing BOM.
- Configured as **I2S master**: it takes MCLK from the CS2000 and generates BCK + LRCK for the DIT4192. (Alternative: DIT is master — see D3.)
- Comfortably covers R3 (44.1–96 kHz) with margin.

### 3.3 AES transmitter — DIT4192 (NEW — replaces DIR9001 receiver)
- 192 kHz / 24-bit AES3/EBU **transmitter**. Requires a master clock at MCLK (≤ 25 MHz); supports master or slave serial port; hardware (pin) or software (I2C) control.
- Configured as **I2S slave** to the ADC, fed the same MCLK as the ADC from the CS2000.
- Drives the AES line via a transmit transformer (Decision D4).
- **Channel-status bits** (declared sample rate, professional/consumer, etc.) set in software mode via the MCU for correct downstream interpretation (Decision D5).

### 3.4 AES output transformer + XLR (REUSE w/ verification)
- AES3 line-driver transformer to XLR, 110 Ω. The sibling board's **PE-65812NLT is a receive-side part** — verify Tx suitability or swap to a transmit-rated transformer (Decision D4).

---

## 4. Clocking architecture (the crux)

This is the part that genuinely needs the word clock, and it maps cleanly onto **one chip — the Cirrus CS2000-CP**:

| WCLK present? | CS2000 mode | MCLK source | Role |
| ------------- | ----------- | ----------- | ---- |
| Yes (locked) | **Hybrid PLL Mode** — auto-derives ratio from `CLK_IN` | Locked to external word clock | **Slave** (matches receiving gear) — R4 primary |
| No / lost | **Frequency Synthesizer Mode** — static ratio from `REF_CLK` | Local crystal timebase | **Master** (standalone fallback) — R4 fallback |

- **`CLK_IN` ← external word clock** (after the 75 Ω BNC front end). In Hybrid mode the CS2000 dynamically tracks the WCLK frequency, so the same configuration handles all of 44.1–96 kHz without per-rate tuning.
- **`REF_CLK` ← local crystal** (e.g. 24.576 MHz) — the low-jitter timebase. Its quality directly sets PLL output jitter, so a good crystal + clean rail matter (ties to the jitter concern from `sim/clock_jitter`).
- **`CLK_OUT` → MCLK** for both ADC and DIT, at **256·Fs** (12.288 MHz @ 48 k; 24.576 MHz @ 96 k — within the DIT4192's 25 MHz MCLK ceiling). CS2000 output range 6–75 MHz covers this.
- The CS2000's **automatic source selection** performs the slave↔master handover on word-clock loss — this is exactly the R4 behavior, in hardware, with no glitch-prone external clock mux.
- **Master-mode multi-rate caveat:** in Frequency Synthesizer (standalone master) mode the ratio is static, so standalone defaults to one rate family unless the MCU reprograms it. See Decision D2.

> Contrast with the parked AES-DAC investigation: there, an external word clock was *redundant* because the DAC inherited sync through its AES input and the DIR9001 couldn't reclock. Here the device *originates* the clock, so a word-clock input is the correct and necessary way to align the output to the receiving equipment.

---

## 5. Part selection

| Function | AES-DAC (existing) | AES-ADC (this design) | Disposition | Rationale |
| -------- | ------------------ | --------------------- | ----------- | --------- |
| Digital interface IC | DIR9001 (receiver) | **DIT4192** (transmitter) | **Swap** | Direction reversed; TI family, 192k/24-bit, slave-able |
| Converter | PCM1789 (DAC) | **PCM4202** (ADC) | **Swap** | 118 dB pro ADC, differential in, master/slave, same family |
| Clock generator | — (none; recovered from AES) | **CS2000-CP** + crystal | **New** | WCLK-slave + master fallback in one chip |
| WCLK front end | — | BNC + 75 Ω term + comparator (e.g. 74AHC14 / TLV3501) | **New** | Square the 75 Ω word clock to CMOS for CLK_IN |
| Control MCU | — (SW1 DIP only) | **RP2040** + QSPI flash + 12 MHz xtal | **New** | Arduino/PlatformIO core; config CS2000/DIT, rate select, lock LEDs |
| MCU flashing | — | **USB-C** + BOOTSEL & RUN buttons | **New** | RP2040 native USB bootloader — drag-drop UF2, no external programmer (R8) |
| Analog stage | OPA1612 (DAC output buffer) | **OPA1612** (input diff receiver + AAF) | **Reuse part, new circuit** | Same opamp, repurposed as input conditioning |
| Regulators | 2× TPS7A49 | 2× TPS7A49 (+ maybe a 3rd for clock) | **Reuse** | +5 V analog / +3.3 V digital; consider clean clock rail |
| Supervisor | APX803 | APX803 | **Reuse** | Reset |
| AES transformer | PE-65812NLT (Rx) | Tx-rated transformer (verify/swap) | **Verify/Swap** | Tx vs Rx turns ratio (D4) |
| Connectors | XLR ×3, barrel jack | XLR ×3, barrel jack, **+ BNC** | **Reuse + add BNC** | Add 75 Ω word-clock input |
| Power input | 9 V barrel | 9 V barrel | **Reuse** | Same supply concept |

---

## 6. Power architecture

- Keep the **9 V → TPS7A49 → +5 V analog / +3.3 V digital** spine from the sibling board.
- PCM4202 (5 V analog + 3.3 V digital), DIT4192 (3.3 V), CS2000 (3.3 V), RP2040 (3.3 V) all fit these rails.
- **Recommended addition:** a dedicated low-noise rail (or RC/ferrite-isolated branch) for the CS2000 + crystal, since clock-domain noise directly degrades AES jitter.
- **USB VBUS:** the RP2040 USB-C is for flashing/comms; keep 9 V as the primary supply and OR-in VBUS through a diode/ideal-OR (or isolate VBUS entirely and self-power the MCU rail from the board) so the board can be flashed on the bench and run from 9 V in service without back-feeding.
- Watch total budget and LDO dropout/thermals if a third regulator is added.

---

## 7. Key design decisions & open risks

| ID | Decision / Risk | Options | Recommendation |
| -- | --------------- | ------- | -------------- |
| **D1** | **RESOLVED — Input headroom / alignment.** 0 dBFS = +24 dBu (R7); pro peaks exceed the +5 V rail. | (a) Passive balanced attenuator before OPA1612 on +5 V; (b) add bipolar rail for THD margin. | **(a)** — ~−12.5 dB passive attenuator sets 0 dBFS = +24 dBu, +4 dBu → −20 dBFS. Keep (b) as a fallback only if low-level THD/noise proves insufficient at schematic/proto stage. |
| **D2** | **RESOLVED — MCU included.** CS2000-CP is I2C-only; channel-status best in software; standalone multi-rate master needed (R4). | RP2040 / ESP32-S3 / ATmega32U4. | **RP2040** — Arduino + PlatformIO support, native USB bootloader for on-board flashing (R8), ample I2C/GPIO. ESP32-S3 is the alternative if Wi-Fi/BT is ever wanted. |
| **D3** | **I2S master role.** Who generates BCK/LRCK. | ADC master / DIT master. | ADC (PCM4202) master, DIT slave — common, keeps converter timing authoritative. |
| **D4** | **AES output transformer.** PE-65812NLT is Rx-rated. | Verify Tx use vs swap to a transmit line-driver transformer per DIT4192 datasheet. | Swap to a DIT4192-recommended Tx transformer unless datasheet confirms PE-65812 Tx use. |
| **D5** | **AES channel status.** Declared Fs, pro/consumer, emphasis. | Hardware defaults vs software (MCU) per-rate. | Software via the RP2040 so channel status tracks the actual sample rate. |
| **D6** | **RESOLVED — Standalone multi-rate master.** Standalone is required (R4); CS2000 synth ratio is static per rate. | Single fixed master rate vs MCU-reprogrammed per selector. | **MCU-selected** master rate via a **DIP switch** (mirrors the existing board's SW1); RP2040 reads the DIP and reprograms the CS2000 ratio when no WCLK is detected. |
| **D9** | **RESOLVED — AES wiring mode.** Single-wire AES3 only; receiver expects no dual-wire/legacy (Q4). | Single-wire / dual-wire. | **Single-wire** — fits 44.1–96 kHz comfortably; simplifies DIT4192 config and the output path. |
| **D7** | **WCLK input electrical.** 75 Ω word clock squaring. | 74AHC14 Schmitt (cheap) vs fast comparator TLV3501/ADCMP600 (lower added jitter). | Comparator for lower jitter; AC-couple + bias, 75 Ω term to GND, optional loop-through/term-disable. |
| **D8** | **NEW — USB VBUS handling.** USB-C present for flashing while 9 V powers the board. | Diode/ideal-OR VBUS with the board rail vs isolate VBUS (data-only) and self-power MCU rail. | Isolate or OR VBUS so the board flashes on the bench and runs from 9 V in service without back-feeding; decide at power-tree design. |

---

## 8. BOM delta vs AES-DAC (summary)

- **Remove:** DIR9001, PCM1789 (and the AES-Rx loop-filter parts), DAC output-buffer network.
- **Add:** PCM4202, DIT4192, CS2000-CP + crystal, WCLK comparator/buffer, BNC connector + 75 Ω term, **RP2040 + QSPI flash + 12 MHz xtal + USB-C + BOOTSEL/RUN buttons**, rate-selector + lock LEDs, input attenuator + anti-alias network, (optional) clock-rail LDO, (optional) bipolar input rail.
- **Keep:** OPA1612 (re-purposed), TPS7A49 ×2, APX803, XLR ×3, 9 V barrel, general power/decoupling topology.
- **Verify/Swap:** AES transformer (Rx→Tx).

---

## 9. Open questions for next pass

1. ~~Input alignment~~ — **Resolved: 0 dBFS = +24 dBu (R7).**
2. ~~Standalone master~~ — **Resolved: standalone is a required mode (R4); master fallback retained.**
3. ~~MCU choice~~ — **Resolved: RP2040 with USB-C on-board flashing (R8).**
4. ~~Single vs dual-wire AES~~ — **Resolved: single-wire AES3 only (D9).**
5. **Form factor / enclosure:** default assumption — **reuse the AES-DAC board outline/connector layout as the starting point**, adding panel cut-outs for the BNC and USB-C. Confirm at layout time.
6. ~~Rate selector UX~~ — **Resolved: DIP switch (D6), mirroring SW1.**

## 10. Recommended next steps

1. Lock the part list (confirm PCM4202 / DIT4192 / CS2000-CP / RP2040, choose the Tx transformer, pick the WCLK comparator).
2. Settle the D8 USB VBUS strategy and confirm the §9.5 enclosure assumption.
3. Proceed to schematic capture (new KiCad project, e.g. `hw/AES-ADC`), reusing AES-DAC power/connector sheets as a starting point.
