# Investigation: Feasibility of adding an external 75 Ω word clock input to drive the DAC

## Hand-off Brief

1. **What was explored.** Whether the AES-DAC board can accept an external 75 Ω word clock (BNC) to clock the PCM1789 DAC, for **studio sync (genlock)**.
2. **Where the case stands (Concluded).** For genlock the device is **already synchronous-by-proxy through its AES input**, so a word-clock input is largely redundant; and the chosen DIR9001 receiver **cannot** reclock decoded AES data from an external clock (its external-clock mode disables data output). A real WCLK-in is therefore a board respin, not a tweak.
3. **What's needed next.** Decide whether AES-inherited sync is sufficient (recommended — no hardware change), or commit to a respin (replace receiver, or add ASRC + WCLK-locked PLL + 75 Ω BNC front end).

## Case Info

| Field            | Value                                                                      |
| ---------------- | -------------------------------------------------------------------------- |
| Ticket           | N/A                                                                        |
| Date opened      | 2026-06-22                                                                 |
| Status           | Concluded                                                                  |
| System           | KiCad hardware project `aes-dac`; AES/EBU → balanced +4 dBu XLR converter, 9V powered |
| Evidence sources | `hw/AES-DAC/AES-DAC.kicad_sch`, rendered PDF, LTspice sims, clock-jitter notebook |

## Problem Statement

User intent (exploration): "Examine the circuit design in this repo and investigate the possibility of adding an external 75 Ω word clock input to drive the DAC." Treated as a feasibility question, not a defect.

## Evidence Inventory

| Source   | Status     | Notes     |
| -------- | ---------- | --------- |
| `hw/AES-DAC/AES-DAC.kicad_sch` | Available | 27k-line KiCad schematic; full clock/data chain extracted via subagent |
| `hw/AES-DAC/output/schematic/AES-DAC.pdf` | Available | Rendered schematic (cross-reference) |
| `sim/clock_jitter/clock_jitter.ipynb` | Available, read | Models 100 ps RMS gaussian jitter on a 20 kHz tone @ 44.1 kHz → spectral skirts. Confirms jitter is a known project concern (context, not the stated goal). |
| `hw/sim/*.asc` | Available, not read | LTspice — output filter / MFB stage; not clock-related, out of scope |
| DIR9001 datasheet | Available (web) | CKSEL clock-mode behavior confirmed (see Finding 4) |
| PCM1789 datasheet | Partial | DAC needs SCKI + BCK + LRCK; standard slave. Exact ratios not re-verified — not pivotal to the verdict |

## Investigation Backlog

| # | Path to Explore | Priority | Status | Notes |
| - | --------------- | -------- | ------ | ----- |
| 1 | Confirm DIR9001 clock modes (PLL vs external master clock) from datasheet | High | Open | Determines whether receiver can be reclocked at all |
| 2 | Confirm PCM1789 clock requirements & whether it can be master | High | Open | DAC needs SCKI + BCK + LRCK; WCLK alone is insufficient |
| 3 | Read clock-jitter notebook to confirm motivation | Medium | Open | Establishes the "why" behind external WCLK |
| 4 | Assess ASRC redesign path (e.g. SRC4392 + PLL like CS2000) | Medium | Open | The architecturally correct way to let external WCLK drive the DAC |

## Confirmed Findings

### Finding 1: Signal chain is AES → transformer → DIR9001 receiver → PCM1789 DAC → OPA1612 output

**Evidence:** `hw/AES-DAC/AES-DAC.kicad_sch` — U1 DIR9001 (~line 22964), U3 PCM1789 (~line 21677), T1 PE-65812NLT transformer (~line 3678), U2/U5 OPA1612 (~lines 20042 / 15547).

**Detail:** AES/EBU enters on XLR J1, is transformer-coupled by T1 into DIR9001 RXIN. DAC differential outputs are buffered by OPA1612 opamps to balanced XLR line outputs J2/J3.

### Finding 2: The DAC clock is recovered from the AES stream — there is no local clock

**Evidence:** `hw/AES-DAC/AES-DAC.kicad_sch` — DIR9001 SCKO→PCM1789 SCKI via R6 (27Ω), BCKO→BCK via R10, LRCKO→LRCLK via R12, DOUT→DIN via R13. DIR9001 XTI/XTO (pins 7/8) marked no-connect; no crystal/oscillator anywhere in the schematic. CKSEL documented "pull low for PLL".

**Detail:** The PCM1789 is a pure clock-slave. All four of its inputs (SCKI master clock, BCK bit clock, LRCLK word clock, DIN data) come straight from the DIR9001's PLL-recovered outputs through 27 Ω series resistors. The system is fully synchronous and self-clocked from the AES input.

### Finding 3: No ASRC, no BNC/coax, no spare clock input, only 3.3V/5V rails

**Evidence:** `hw/AES-DAC/AES-DAC.kicad_sch` — no sample-rate-converter IC present; connectors are XLR (J1/J2/J3) + barrel jack (J6) + test points (TP3/4/5/7, J4/J5); rails are +3.3V and +5V from two TPS7A49 LDOs (U6/U7), 9V input. No ±15V analog rail.

**Detail:** There is no architectural provision for an external clock: no BNC, no 75 Ω termination, no clock-squaring/comparator stage, and no FIFO/ASRC to absorb a rate difference between the AES data and an external clock.

### Finding 4: The DIR9001's external-clock mode disables data decoding (no reclock path)

**Evidence:** TI/Burr-Brown DIR9001 datasheet — CKSEL pin selects the output clock source. `CKSEL = L`: PLL recovery, decodes biphase data, outputs SCKO/BCKO/LRCKO (current design). `CKSEL = H`: outputs the XTI source clock to peripherals, but **"recovered clock and decoded data is NOT output."** (Burr-Brown DIR9001 datasheet, clock-source-selection section.)

**Detail:** The two modes are mutually exclusive. The DIR9001 cannot simultaneously decode the AES audio and clock its outputs from an external master clock. There is no "external master clock reclocks the recovered data" mode (unlike e.g. CS8416 OMCK). Therefore the existing receiver cannot be made to drive the DAC from an external word clock while still passing AES audio.

## Deduced Conclusions

### Deduction 1: Word clock alone cannot clock the PCM1789

**Based on:** Findings 2 and 3.

**Reasoning:** A word clock provides only the sample-rate reference (Fs = LRCLK). The PCM1789 additionally requires a system/master clock (SCKI, typically 128–512×Fs) and a bit clock (BCK). An external WCLK input would still need a clock generator/PLL to synthesize SCKI and BCK locked to it.

**Conclusion:** "External WCLK to drive the DAC" cannot be a simple wire-in; it requires added clock-generation hardware at minimum.

### Deduction 2: Without an ASRC, an external clock asynchronous to the AES source will glitch

**Based on:** Findings 2 and 3.

**Reasoning:** The audio samples arriving over AES are produced at the upstream source's sample rate. If the DAC consumes them at a rate set by an independent external word clock, the two rates drift; with no FIFO/ASRC to absorb the difference, the result is periodic sample slips (clicks/dropouts).

**Conclusion:** An external WCLK can only "drive the DAC" cleanly if either (a) the AES source is itself slaved to that same word clock (genlock), or (b) an ASRC is added to decouple the DAC clock domain from the AES data.

### Deduction 3: For the genlock goal, a word-clock input on this DAC is largely redundant

**Based on:** Findings 2, 4, and the stated goal (studio sync / genlock).

**Reasoning:** Genlock means every device — including the console/source that produces this board's AES feed — is locked to the house word clock. The AES stream arriving here is therefore *already* at the house-clock rate, the DIR9001 recovers exactly that rate, and the DAC plays out aligned to it. The device is genlocked **by proxy through its AES input**. A separate WCLK input cannot improve sample-rate sync (the rate is already correct) and cannot reclock the data (Finding 4). The other classic genlock benefit — sample *phase* alignment — is moot for an output-only DAC: there is no capture to align, and the output phase is already deterministic relative to the AES source plus fixed DAC group delay.

**Conclusion:** For pure studio sync, no hardware change is warranted; rely on AES-inherited sync. A physical WCLK-in only becomes meaningful if the requirement is reclocking/jitter-rejection or the AES source is *not* itself genlocked — and either of those is a respin (Deduction 2 / Finding 4).

## Hypothesized Paths

### Hypothesis 1: Jitter is a project concern (context for any future reclock work)

**Status:** Confirmed (as context), not the driver of this request.

**Theory:** `sim/clock_jitter/clock_jitter.ipynb` exists to quantify how clock jitter degrades the DAC output.

**Resolution:** Confirmed the notebook models 100 ps RMS gaussian jitter on a 20 kHz tone at 44.1 kHz and shows the resulting spectral skirts — so jitter is demonstrably on the designer's radar. However, the user's stated goal for this investigation is genlock, not jitter reduction. Noted as the motivation that *would* justify the ASRC-reclock respin if priorities change.

## Missing Evidence

| Gap | Impact | How to Obtain |
| --- | ------ | ------------- |
| DIR9001 datasheet clock-mode details | Confirms whether the receiver can run from an external master clock at all | TI DIR9001 datasheet |
| PCM1789 clock/master-mode details | Confirms required clock ratios and whether DAC can be master | TI PCM1789 datasheet |
| User's actual goal (sync vs. jitter) | Determines which redesign path is in scope | Ask user |

## Conclusion

**Confidence:** High on architecture and on the DIR9001 mode limitation (both Confirmed from schematic + datasheet). The genlock-redundancy conclusion is Deduced from those facts.

**Verdict for the stated goal (studio sync / genlock):** A 75 Ω word-clock input is **not worth adding** to this board as-is, for two compounding reasons:

1. **It's redundant for sync.** In a genlocked studio the AES source already follows the house clock, so this DAC — which slaves to its AES input — is already sample-rate-synchronous with the house clock. Phase alignment, the other genlock benefit, is meaningless for an output-only DAC.
2. **The current chipset can't use it anyway.** The DIR9001 cannot reclock decoded AES data from an external clock (Finding 4), so a WCLK input cannot be wired into the existing receiver to "drive the DAC." And a word clock alone (Fs only) cannot clock the PCM1789, which also needs SCKI and BCK (Deduction 1).

A genuine, working word-clock input is therefore a **board respin**, justified only if (a) you must interoperate with a source that is *not* itself genlocked, or (b) the real aim is jitter rejection (the notebook's concern). Design sketches for both respin paths are below.

## Recommended Next Steps

### Fix direction

**Recommended (no hardware change): rely on AES-inherited sync.** Confirm the upstream console's AES output is locked to the house word clock. If it is, this DAC is already genlocked through AES and needs nothing. This is the right answer for the stated goal.

**If a physical WCLK-in is still required — two respin options:**

- **Option B1 — Receiver swap (lighter respin).** Replace the DIR9001 with an AES receiver that supports running its decode from a supplied master clock that is frequency-locked to the input (verify the exact mode against the datasheet — candidates: CS8416, AK4118). Feed it a master clock derived from the external word clock. Still only works if the AES source is genlocked to that same word clock (no ASRC = no rate decoupling). Effort: new receiver + clock-gen + BNC front end; moderate.

- **Option B2 — ASRC reclock (robust respin, also fixes jitter).** Insert an asynchronous sample-rate converter so the DAC clock domain is fully decoupled from the AES source. Works whether or not the source is genlocked. Effort: high.
  - **AES front end** → **SRC4392** (integrates AES Rx + ASRC; can re-time to a new master clock).
  - **Word-clock-locked clock generator** (e.g. CS2000-CP) takes the 75 Ω WCLK reference and synthesizes a low-jitter MCLK at 256/512×Fs for both the SRC4392 output port and the PCM1789 SCKI.
  - **PCM1789** then runs entirely off the WCLK-locked clock; the ASRC absorbs any rate offset against the AES data.

- **75 Ω BNC word-clock front end (needed for B1 or B2):** BNC connector → 75 Ω termination to GND → AC-couple → bias to mid-rail (Vcc/2) → high-speed Schmitt/comparator (e.g. 74AHC14, or a fast comparator such as TLV3501/ADCMP600) to square the signal to clean 3.3 V CMOS → into the clock generator/PLL. Add this only as part of B1/B2 — feeding raw WCLK anywhere on the current board does nothing useful.

### Diagnostic

- Verify the console's AES output clock source (house clock vs. internal) — this single fact decides whether *any* hardware change is needed.
- If pursuing a respin, confirm exact clock-mode behavior in the DIR9001-replacement and PCM1789 datasheets (master-clock ratios, master/slave roles).

## Reproduction Plan

Verification plan (exploration case): With a genlocked source feeding the existing board, measure the DAC LRCLK against the house word clock — they will be frequency-locked already, demonstrating the redundancy conclusion. To validate a B2 respin, inject an AES stream whose source is *not* locked to the WCLK reference and confirm glitch-free output (the ASRC absorbing the offset) versus the current board (which would only track the AES rate).

## Side Findings

- The board runs the OPA1612 output stage single-supply from +5 V with a Vcom bias node, and has only +3.3 V / +5 V rails — relevant because a B1/B2 respin adds a clock generator/ASRC that may need a clean dedicated rail and careful clock-domain grounding.
- `sim/clock_jitter/clock_jitter.ipynb` already quantifies jitter sensitivity; if priorities shift from genlock to sound quality, Option B2 is the path that directly addresses it.
- The DIR9001 also offers an automatic PLL↔XTI fallback (switches on its ERROR flag), but in XTI fallback no data is decoded — it is a loss-of-lock mute behavior, not a reclock feature.
