# AES-ADC — Draft BOM (candidate parts)

Companion to `aes-adc-architecture.md`. Pre-schematic candidate BOM: orderable MPNs and packages for the active devices, connectors, and key passives. Passive values (attenuator/AAF resistors, decoupling) are finalized at schematic time. Footprints for **reused** parts are taken from the existing `hw/AES-DAC` board so they drop in.

**Disposition:** Reuse = same part as AES-DAC · Swap = replaces an AES-DAC part · New = not on AES-DAC.

---

## Converter & digital interface

| Ref | Part / MPN | Package / Footprint | Function | Disp. | Notes |
| --- | ---------- | ------------------- | -------- | ----- | ----- |
| U1 | **PCM4202IDB** (TI) | SSOP-28 (DB) | Stereo 24-bit/216 kHz audio ADC, 118 dB, diff in, I2S **master** | Swap (← PCM1789) | Confirmed SSOP-28. Tape-reel = `PCM4202IDBR`. |
| U2 | **DIT4192IDB** (TI) | SSOP-28 (DB) — *verify suffix* | AES3/EBU transmitter, single-wire, I2S **slave** | Swap (← DIR9001) | Sibling SRC4192 is SSOP-28; confirm DIT4192 package/orderable at quote. |
| T1 | AES **transmit** transformer — Pulse **PE-65612NL** or Scientific Conversions **SC937-02** | per vendor (verify vs `components:PE-65812NLT` footprint) | 110 Ω AES line-driver isolation | Swap (← PE-65812NLT Rx) | Existing PE-65812NLT is Rx-rated; use a Tx transformer per DIT4192 datasheet. **D4.** |

## Clock generation & word-clock input

| Ref | Part / MPN | Package / Footprint | Function | Disp. | Notes |
| --- | ---------- | ------------------- | -------- | ----- | ----- |
| U3 | **CS2000CP-CZZ** (Cirrus) | MSOP-10 | Fractional-N clock gen: WCLK-slave (Hybrid PLL) / xtal-master (Synth) → MCLK 256·Fs | New | I2C-controlled by MCU. Output 6–75 MHz covers 12.288/24.576 MHz. |
| Y1 | Crystal **24.576 MHz** (e.g. Abracon ABM8) | SMD 3.2×2.5 mm | CS2000 REF_CLK timebase / master-mode reference | New | Confirm value vs CS2000 RefClk range + output-ratio math. **Q (clock).** |
| J1 | **75 Ω PCB BNC** (Amphenol RF 031-series / TE) | BNC SMD/THT | External word-clock input | New | Add 75 Ω term + optional term-disable for loop-through. |
| U4 | **TLV3501AIDBV** (TI) | SOT-23-6 | High-speed comparator — squares WCLK to 3.3 V CMOS → CS2000 CLK_IN | New | Lower added jitter than a 74AHC14. **D7.** AC-couple + mid-rail bias. |
| R/C | 75 Ω term, AC-couple cap, bias divider | 0603/0805 | WCLK input network | New | Values at schematic. |

## MCU & on-board flashing

| Ref | Part / MPN | Package / Footprint | Function | Disp. | Notes |
| --- | ---------- | ------------------- | -------- | ----- | ----- |
| U5 | **RP2040** (Raspberry Pi) | QFN-56 7×7 | Control MCU: CS2000/DIT config, WCLK-lock detect, DIP→rate, LEDs | New | Arduino + PlatformIO cores. **D2.** |
| U6 | **W25Q128JVSIQ** (Winbond) 16 MB | SOIC-8 (208 mil) | QSPI boot flash for RP2040 | New | W25Q16/32 also fine; 128 Mbit common/cheap. |
| Y2 | Crystal **12 MHz** (e.g. Abracon ABM8) | SMD 3.2×2.5 mm | RP2040 system/USB clock | New | Required for reliable USB. |
| J2 | **USB-C receptacle** (GCT USB4085-GF-A or similar) | USB-C SMD | Firmware flash + comms | New | Native RP2040 bootloader → drag-drop UF2. **R8.** |
| SW1 | Tact switch (SMD) — **BOOTSEL** | SMD tact | Hold-at-boot → USB mass storage | New | |
| SW2 | Tact switch (SMD) — **RUN/reset** | SMD tact | RP2040 reset | New | |
| SW3 | **DIP switch** (SW_DIP slide) | `Button_Switch_SMD:SW_DIP_...` (reuse family) | Standalone sample-rate select | New | Same DIP footprint family as AES-DAC SW1. **D6.** |
| D1–D3 | LEDs (lock / rate / power) | `LED_THT:LED_D3.0mm` (reuse) or 0805 | Status indication | New | |

## Analog input stage (2 ch)

| Ref | Part / MPN | Package / Footprint | Function | Disp. | Notes |
| --- | ---------- | ------------------- | -------- | ----- | ----- |
| U7, U8 | **OPA1612AIDR** (TI) | `Package_SO:SOIC-8_3.9x4.9mm` (reuse) | Diff receiver + anti-alias + ADC driver (1 dual/ch) | Reuse part, new circuit | Same opamp as AES-DAC output stage. |
| R(att) | 0.1 % thin-film resistors | 0603/0805 | Balanced attenuator: **0 dBFS = +24 dBu** (≈ −12.5 dB) + diff-amp set | New | Precision/matched for CMRR & gain accuracy. **D1/R7.** |
| C(aaf) | C0G/NP0 caps | 0603/0805 | Anti-alias filtering | New | |
| J3, J4 | **Neutrik NC3FAH1** female XLR | `Connector_Audio:Jack_XLR_Neutrik_NC3FAH1-0_Horizontal` (reuse) | Balanced L / R analog inputs | Reuse footprint | Was 1× on AES-DAC; now 2× input. |
| J5 | **Neutrik NC3MAH** male XLR | `Connector_Audio:Jack_XLR_Neutrik_NC3MAH-0_Horizontal` (reuse) | AES3 output | Reuse footprint | |

## Power & supervisor

| Ref | Part / MPN | Package / Footprint | Function | Disp. | Notes |
| --- | ---------- | ------------------- | -------- | ----- | ----- |
| U9 | **TPS7A4901** (TI) | `HVSSOP-8-1EP_3x3mm` (reuse) | +5 V analog LDO | Reuse | |
| U10 | **TPS7A4901** (TI) | HVSSOP-8-1EP (reuse) | +3.3 V digital LDO | Reuse | |
| U11 | **TPS7A4901** *(optional)* or low-noise LDO (TPS7A2033 / ADM7150) | HVSSOP-8 / SOT-23 | Dedicated clean rail for CS2000 + crystal | New (opt.) | Clock-domain noise → AES jitter. §6. |
| U12 | **APX803S05-29SA-7** | `Package_TO_SOT_SMD:SOT-23` (reuse) | Reset supervisor | Reuse | |
| J6 | **GCT DCJ200-10-A-K1-K** barrel jack | `components:BarrelJack_GCT_DCJ200-10-A_Horizontal` (reuse) | 9 V input (center-negative) | Reuse | |
| D4 | Schottky / ideal-OR (e.g. SOD-123) | SOD-123 | USB VBUS ↔ 9 V rail OR-ing / isolation | New | **D8** — prevent back-feed when flashing on bench. |
| — | Bulk + decoupling caps, ferrite beads | 0805 / electrolytic (reuse families) | Power conditioning | Mix | Per-rail decoupling at schematic. |

---

## Removed from AES-DAC

- **DIR9001** (AES receiver, TSSOP-28) and its PLL loop-filter network.
- **PCM1789** (DAC, TSSOP-24) and the DAC output-buffer network.

## Items to finalize before ordering

1. **DIT4192 package/orderable suffix** — confirm on TI (expected SSOP-28 DB).
2. **AES Tx transformer** — pick a transmit-rated part per the DIT4192 datasheet (PE-65612NL / SC937-02 candidates) and lay out its footprint.
3. **CS2000 REF_CLK crystal value** — confirm 24.576 MHz (or alternative) is within the CS2000 RefClk range and yields the needed 256·Fs outputs across 44.1–96 kHz.
4. **BNC MPN** — select an orderable 75 Ω PCB BNC and its footprint.
5. **Clean clock LDO (U11)** — decide whether it's warranted vs filtering off the +3.3 V rail.
6. **Precision-resistor tolerances** for the attenuator (CMRR/gain) — 0.1 % vs 0.05 %.
