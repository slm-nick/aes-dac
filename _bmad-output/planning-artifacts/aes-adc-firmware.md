# AES-ADC — RP2040 Firmware Design

Companion to `aes-adc-architecture.md` and `aes-adc-bom.md`. Scope: what the RP2040 is responsible for, the interfaces it drives, the slave↔master control logic, and the build/flash workflow. This is a design sketch — exact register values are flagged for datasheet confirmation.

---

## 1. Role (and explicit non-role)

The MCU is **a configuration and supervision controller, not in the audio path.** No audio samples ever touch it; it never gates jitter-critical clocks in software. Its jobs:

1. Bring up and configure the **CS2000** clock generator (slave vs. master ratio).
2. Configure the **DIT4192** transmitter (software mode, single-wire, AES **channel-status** — declared sample rate, professional flag, etc.).
3. Detect **word-clock presence + frequency**, and drive the slave↔master state machine.
4. Read the **rate-select DIP** for standalone (master) operation.
5. Drive **status LEDs** (external lock / internal master / fault).
6. Enforce **clock-first reset sequencing** so the ADC/DIT only run once a valid MCLK exists.

**Not the MCU's job:** the **PCM4202 ADC** is hardware-strapped (M/S, format, rate-range pins) — no register interface, so firmware does not configure it. The MCU only ensures its MCLK is valid before releasing its reset.

---

## 2. Interfaces / pin map (proposal)

| RP2040 | Peripheral | Direction | Purpose |
| ------ | ---------- | --------- | ------- |
| GP4/GP5 | **I2C0** SDA/SCL → CS2000 | bidir | CS2000 register config + lock/unlock status (addr ~0x4E, verify AD0 strap) |
| GP2/GP3/GP1 | **SPI0** SCK/TX/CSn → DIT4192 control port | out | DIT4192 software-mode config + channel status (*verify DIT4192 port = SPI-like*) |
| GP6 | **WCLK monitor** (tap after comparator) | in | PIO frequency counter — presence + exact Fs |
| GP7 | CS2000 **UNLOCK**/lock pin (optional) | in | Hardware lock indication (back-up to I2C status) |
| GP10–GP11 | **Rate DIP** (2 bits) | in | Standalone master rate: 00=44.1 01=48 10=88.2 11=96 kHz |
| GP12 | DIP: force-master / auto | in | Optional override |
| GP13–GP15 | **LEDs** | out | EXT-LOCK (grn) / INT-MASTER (amber) / FAULT (red) |
| GP20 | **DIT4192 / ADC reset** gate | out | Hold converter+Tx in reset until MCLK valid |
| QSPI | W25Q128 flash | — | Boot |
| USB DP/DM | USB-C | — | Flash (UF2) + optional serial console |
| RUN, QSPI_CSn | RUN + BOOTSEL buttons | in | Reset / enter bootloader (hardware, not firmware GPIO) |

Crystal: 12 MHz on XIN/XOUT (USB-accurate clock).

---

## 3. Clock logic — the core state machine

```
            power-up / APX803 reset
                     │
                     ▼
              [ INIT ]  configure CS2000 + DIT4192, hold converter reset
                     │  (MCLK not yet trusted)
                     ▼
        ┌───── WCLK present & in-range? ─────┐
       yes                                   no
        ▼                                    ▼
  [ SLAVE / EXT-LOCK ]                 [ MASTER / INTERNAL ]
  CS2000 Hybrid-PLL, MCLK=256·WCLK     CS2000 Freq-Synth from xtal
  Fs := measured WCLK                  Fs := DIP selection
  DIT CS bits := Fs                    DIT CS bits := Fs
  release converter reset              release converter reset
  EXT-LOCK LED                         INT-MASTER LED
        │                                    │
        └───────── on change / unlock ───────┘
                     │
              [ FAULT ] CS2000 unlock, WCLK out-of-range, or sequencing error
                  → re-assert converter reset, FAULT LED, retry
```

- **Slave path (WCLK present):** CS2000 in **Hybrid PLL Mode** dynamically locks MCLK to `CLK_IN` (the word clock); firmware programs the multiply ratio so **MCLK = 256·Fs**. Firmware *measures* the WCLK frequency (PIO) to (a) validate it's a real audio rate and (b) write the matching sample-rate code into the DIT4192 channel status.
- **Master path (WCLK absent):** CS2000 reverts to **Frequency Synthesizer Mode** off the 24.576 MHz crystal; firmware programs the ratio for the **DIP-selected** rate and writes the corresponding channel-status code.
- **Automatic handover:** the CS2000's own CLK_IN-present detection performs the glitchless hardware switch; firmware tracks it (via I2C status / UNLOCK pin) to update LEDs and channel status. Firmware does **not** mux clocks itself.

---

## 4. Device configuration detail

### 4.1 CS2000 (I2C) — *register values to confirm against datasheet*
- **RefClkDiv** — divide the 24.576 MHz crystal into the CS2000's supported internal reference window.
- **Ratio (32-bit, fixed-point)** — set output/input multiple. Slave: ratio = 256 (MCLK/WCLK). Master: ratio chosen with RefClkDiv/R-value so MCLK = 256·Fs for the selected rate (12.288 MHz @48 k … 24.576 MHz @96 k). *Compute in the datasheet's 12.20/20.12 format.*
- **AuxOutSrc / device config** — route synthesized clock to CLK_OUT (→ MCLK), enable output.
- **Auto source select** — enable dynamic-ratio-when-CLK_IN-present, static-ratio-when-absent (the master/slave fallback).
- **Status polled:** PLL lock/unlock, CLK_IN present.

### 4.2 DIT4192 (control port) — *port type + map to confirm*
- Put in **software/control mode**; **single-wire** AES3 output (D9).
- Serial port as **slave** to the ADC (BCK/LRCK from PCM4202), MCLK from CS2000.
- **Channel status block:** professional/consumer = professional; sample-rate field = current Fs; copy/emphasis defaults; data validity. Re-written whenever Fs changes (slave-measured or DIP).

### 4.3 PCM4202 (no firmware) 
- Hardware-strapped: master serial port, I2S/LJ format, high-rate vs base-rate range pin set for 44.1–96 kHz. Firmware only sequences its reset.

---

## 5. Sequencing & safety

1. After reset, **configure CS2000 first**; wait for lock/valid MCLK.
2. Only then **release the ADC/DIT reset** (GP20) — prevents the converter and transmitter from running on an absent/garbage clock.
3. On **WCLK loss while in slave**, the CS2000 auto-falls-back to master; firmware updates channel status to the DIP rate (or holds last) and updates LEDs — avoid emitting AES tagged with the wrong Fs.
4. On **CS2000 unlock**, enter FAULT, re-assert converter reset, retry bring-up.

---

## 6. Toolchain & flashing

- **Core:** Arduino-Pico (earlephilhower) under **PlatformIO** — gives `Wire`, `SPI`, and **PIO** access; or the bare-metal pico-sdk if preferred.
- **Flashing (no external programmer, R8):** hold **BOOTSEL**, tap **RUN** → board enumerates as `RPI-RP2` mass storage → drag-drop the `.uf2`. Or `pio run -t upload` (picotool/USB). SWD pads optional for debug.
- Skeleton `platformio.ini`:
  ```ini
  [env:rp2040]
  platform = raspberrypi
  board = pico
  framework = arduino
  upload_protocol = picotool   ; or mbed/uf2
  lib_deps =                   ; Wire/SPI are built-in
  ```

### Pseudocode — main supervision loop
```cpp
setup() {
  resetConverters(HOLD);
  cs2000_init();            // RefClkDiv, ratio, auto-source-select, enable
  dit4192_init();           // software mode, single-wire, slave port
}
loop() {
  float wclk = pio_measure_wclk_hz();        // 0 if absent
  bool slave = cs2000_clkin_present() && in_audio_range(wclk);
  uint32_t fs = slave ? nearest_rate(wclk) : dip_rate();
  if (fs != last_fs || slave != last_slave) {
     cs2000_set_mode(slave, fs);             // ratio for slave vs master
     wait_lock(CS2000);
     dit4192_set_channel_status(fs, /*pro=*/true);
     resetConverters(RELEASE);
     leds(slave ? EXT_LOCK : INT_MASTER);
  }
  if (cs2000_unlocked()) fault_recover();
  sleep_ms(20);
}
```

---

## 7. Open items / to confirm

1. **DIT4192 control-port type** (SPI vs other) and exact channel-status register map.
2. **CS2000 register math** — RefClkDiv + Ratio fixed-point values for each Fs; confirm 24.576 MHz RefClk is in range and that 256·Fs lands within CS2000 (6–75 MHz) and DIT4192 MCLK (≤25 MHz) limits at 96 k (24.576 MHz — OK).
3. **CS2000 I2C address / AD0 strap.**
4. **WCLK measurement gate time** vs lock-acquisition latency (UX: how fast EXT-LOCK asserts).
5. **PCM4202 strapping table** — finalize the hardware mode pins (so firmware scope stays "no ADC config").
6. **Channel-status policy on WCLK loss** — hold last Fs vs follow DIP (clicks/mute behavior).
