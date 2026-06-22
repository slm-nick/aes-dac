# AES-ADC — Schematic Capture Plan

This KiCad 8 project is a **hierarchical skeleton**, not a finished schematic. It opens cleanly and gives you the subsystem structure, the inter-sheet nets (as hierarchical labels), and a per-sheet component + connection checklist. **No component symbols are placed yet** — capture is intentionally left to KiCad so symbol/footprint links are real.

Design rationale lives in `../../_bmad-output/planning-artifacts/`:
`aes-adc-architecture.md` · `aes-adc-bom.md` · `aes-adc-firmware.md` · `aes-adc-cost.md`.

## Project structure

```
AES-ADC.kicad_pro        project
AES-ADC.kicad_sch        ROOT — 5 hierarchical sheet symbols + inter-sheet pins
├── power.kicad_sch          rails + supervisor (reuse AES-DAC tree)
├── clocking.kicad_sch       CS2000 + word-clock BNC front end
├── analog-input.kicad_sch   2ch balanced receive / attenuate / AAF
├── digital-aes.kicad_sch    PCM4202 ADC + DIT4192 Tx + transformer
└── mcu.kicad_sch            RP2040 + USB-C + DIP/LEDs
```

Each child sheet already contains a **text note** (component list) and the **hierarchical labels** for nets that cross sheet boundaries. The root has matching **sheet pins**.

## How to start capture

1. Open `AES-ADC.kicad_pro` in KiCad 8. Confirm the 5 sheets navigate from the root.
2. Per sheet, place the symbols from the note, wire to the **existing hierarchical labels**.
3. Reuse AES-DAC symbols/footprints where marked "reuse" (OPA1612, TPS7A4901, APX803, Neutrik XLR, GCT barrel jack, DIP). Copy from `../AES-DAC/AES-DAC.kicad_sch` or the shared `components` footprint lib.
4. For new ICs (PCM4202, DIT4192, CS2000, RP2040, TLV3501, W25Q128) create/import symbols and assign footprints from the BOM packages.
5. Power rails (`+5V`, `+3V3`, `+3V3_CLK`, `GND`) use **global power port symbols** — they are intentionally NOT hierarchical labels.
6. After wiring a child's labels, on the root use **Import Sheet Pins** if you add/rename any.

## Inter-sheet net interface (already placed as hierarchical labels)

| Net | Driver sheet | Consumer(s) | Meaning |
| --- | ------------ | ----------- | ------- |
| `RST_N` | Power | Digital/AES, MCU | Power-on reset (APX803) |
| `MCLK` | Clocking | Digital/AES | Master clock 256·Fs (CS2000 CLK_OUT) |
| `WCLK_MON` | Clocking | MCU | Squared word clock tap → RP2040 PIO freq count |
| `CS2K_UNLK` | Clocking | MCU | CS2000 unlock/lock status |
| `SDA` / `SCL` | MCU | Clocking | I2C to CS2000 |
| `DIT_SCK`/`DIT_MOSI`/`DIT_CS` | MCU | Digital/AES | SPI control to DIT4192 |
| `CONV_RST_N` | MCU | Digital/AES | Gated ADC+DIT reset (clock-first sequencing) |
| `AINL_P`/`AINL_N`/`AINR_P`/`AINR_N` | Analog Input | Digital/AES | Conditioned differential audio → ADC |

Rails distributed globally (not in the table): `+5V`, `+3V3`, `+3V3_CLK`, `GND`.

## Per-sheet connection guide

### power.kicad_sch  (mostly reuse)
- `J` barrel jack DCJ200-10-A → `VIN`; 2× (or 3×) TPS7A4901 → `+5V`, `+3V3`, `+3V3_CLK`.
- APX803S05 → `RST_N`. VBUS-OR diode between `USB_VBUS` (MCU) and the rail (**D8**).

### clocking.kicad_sch
- BNC `J` → 75 Ω term → AC-couple + bias → TLV3501 → `WCLK_CMOS` (also tap to `WCLK_MON`).
- CS2000: `CLK_IN ← WCLK_CMOS`, `REF_CLK ← Y(24.576 MHz)`, `CLK_OUT → MCLK`, I2C `SDA/SCL`, status → `CS2K_UNLK`. Power from `+3V3_CLK`.

### analog-input.kicad_sch
- 2× NC3FAH1 XLR → balanced 0.1 % attenuator (0 dBFS = +24 dBu, ≈ −12.5 dB) → OPA1612 diff/AAF → `AINL_P/N`, `AINR_P/N`. Vcom mid-rail bias from `+5V`.

### digital-aes.kicad_sch  (the core)
- **PCM4202** (I2S master, hardware-strapped M/S, FMT, rate-range): `MCLK` in; `AIN*` to VIN±; outputs **internal** `BCK`/`LRCK`/`SDATA`.
- **DIT4192** (I2S slave): `MCLK` in; `BCK`/`LRCK`/`SDATA` from ADC; control `DIT_SCK/MOSI/CS`; `CONV_RST_N`; `TX± →` AES Tx transformer → NC3MAH XLR (110 Ω). Single-wire.
- `BCK`/`LRCK`/`SDATA` stay on this sheet (local labels), not in the inter-sheet table.

### mcu.kicad_sch
- RP2040 + 12 MHz xtal + W25Q128 + USB-C + BOOTSEL/RUN + rate DIP + LEDs.
- I2C → `SDA/SCL`; SPI → `DIT_*`; reads `WCLK_MON`, `CS2K_UNLK`; drives `CONV_RST_N`. `USB_VBUS` → Power.

## Not done / still open (carried from planning docs)
- Symbols + footprints for the new ICs; exact pin assignments per datasheet.
- DIT4192 control-port type & channel-status map; CS2000 RefClkDiv/Ratio math.
- AES Tx transformer footprint (swap from Rx PE-65812NLT); 75 Ω BNC MPN.
- Decoupling/values, PCB layout (new `AES-ADC.kicad_pcb` once schematic is wired & annotated).
