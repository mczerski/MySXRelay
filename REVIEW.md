# MySXRelay — design review

Reviewed at commit `ae69c83` (2025-01-14 PCB, 2024-12-25 schematic).
Companion review: [`../bistable-relay/REVIEW.md`](../bistable-relay/REVIEW.md) — the relay
daughter card that plugs into `J3`. Findings **S4** and **P1** are shared between the two boards
and appear in both documents.

Every number quoted here was measured from the design files, not estimated — see
[Verification](#verification) for the recipes.

---

## System as built

34.5 × 34.5 mm, 4-layer, ~1.18 mm total stackup (0.035 Cu / 0.10 core / 0.035 / 0.84 core /
0.035 / 0.10 prepreg / 0.035), fitted to a 40 × 40 mm HM-1551V1WH case.

### Two ground domains

| Domain | Reference | Contents |
|---|---|---|
| Mains-referenced ("hot") | `N_IN` = mains **neutral** | U6 ATmega328PB, U2 BL0942, U7 SPI memory, U3 BP2525 buck, U5 AP2127N-3.3, J1 AVR ISP, **X3 LED/SW header**, and the relay module's `VCC`/`GND` |
| Isolated (SELV bus) | `GND1` | U4 isolator secondary, U9 ST485E, U1 AP2204RA-5.0, Q2 BSS84A, F1/F2, D9/D10, X1 RS-485 |

The barrier is U4 alone (netclasses `EXT` / `EXT_VCC`). Everything else — including the MCU and
the external LED/button header — floats at neutral potential.

### Power

```
L_IN ─ TH1 ─ D5 ─ C5 (2.2µF/400V) ─ U3 BP2525 floating buck ─ L1 330µH ─ +5V ─ U5 ─ +3V3
                                     D6 ES1J freewheel, C7 470µF, R9 2k min-load
                                     R8/D7/C6 self-supply for U3's VCC after startup
+5V also feeds the relay module via J3.4.
X1 bus V+ ─ Q2 (reverse-polarity P-FET) ─ U1 AP2204RA-5.0 ─ +5VD  (isolated side)
```

### Metering

R12 (1 mΩ, 2512) sits in the load's **neutral return** (`N_OUT` → `N_IN`), so the node's own
supply current — which flows `L_IN` → TH1 → … → `N_IN` — is correctly excluded from the
measurement. Voltage divider is 5 × 390 k over R13 510 Ω, tapping `L_IN`.

BL0942 runs in **UART** mode: `SEL` (pin 7) floats and the datasheet's internal pull-down
selects UART at 4800 bps. That is also why R1 (2 k) pulls up its open-drain `TX` — consistent
with commit `6136a74`.

### Card edge (J3)

`J3.1` = `L_IN`, `J3.2` = `L_OUT`, `J3.3` = `REL`, `J3.4` = `+5V`, `J3.5` = `N_IN`.
Two milled slots 1.2 mm wide receive the module's two 1.0 mm tabs. Mains pads and logic pads are
**10.3 mm** apart on the board surface — a genuinely good arrangement.

---

## Findings

### Safety / spacing

#### S1 — Mains-to-RS-485 isolation is *basic*, not reinforced — **2.963 mm**

Measured minimum in-plane clearance between `EXT*` copper and mains-referenced copper:

| Location | Clearance |
|---|---|
| X3 pad 1 (`/MCU/LED`) → `GND1` zone | **2.963 mm** |
| Across U4 (pad 1 `+3V3` ↔ pad 16 `+5VD`, etc.) | **3.000 mm** |

For 230 V rms, OVC II, PD 2, material group IIIa the requirement is 2.5 mm creepage for basic
and **5.0 mm for reinforced**. The RS-485 bus leaves the enclosure and is user-accessible, so
this barrier must be reinforced. Today it clears basic by 0.46 mm and misses reinforced by 2 mm.

Compounding it — **the schematic names a part that does not fit the board**:

- Footprint on the PCB is `Package_SO:SOIC-16_3.9x9.9mm_P1.27mm` — **narrow body**. Measured pad
  pitch across the package is 4.95 mm, outer span 6.9 mm, inner pad gap 3.0 mm.
- Schematic `Value` is `Si8642BB-B-IS1`, which is the Skyworks **wide-body** SOIC-16
  (VDE-certified 8.5 mm creepage, ~10.3 mm lead span). It is physically ~4.4 mm too wide for
  these pads.
- The part actually sourced in `MySXRelay.ods` is TME `PAI142M31` (2Pai Semi, 4-channel, SO16,
  **3 kVrms**). That one fits, but it is a basic-isolation part.

**Fix (respin):** fit a wide-body SOIC-16W reinforced isolator — `Si8642BB-B-IS1` as the
schematic already claims, or `ISO7742DW` — widen the barrier to ≥ 5 mm creepage everywhere
(≥ 8 mm to make the package's rating usable), add a milled slot along the barrier, and add a
`.kicad_dru` rule stating the number so it cannot silently regress.
**Fix (now):** correct U4's `Value` so the schematic stops claiming a part that cannot be built.

> **Credit where due:** the barrier is a *clean channel through the entire stackup*. Zone-to-zone
> overlap between the isolated and mains-referenced domains measures **0.000 mm² on every
> adjacent layer pair**, including across the 0.10 mm prepreg between F.Cu/Gnd.Cu and
> Pwr.Cu/B.Cu. KiCad's clearance DRC is per-layer only and would not have caught a violation
> here — this was done right by hand. Only the in-plane distance is short.

#### S2 — X3 (external LED + button) is at mains potential

`X3` is a 3-pin JST-XH carrying `/MCU/LED`, `/MCU/SW` and `+3V3`, all referenced to `N_IN` =
mains neutral. Neutral is not a safety earth: with a reversed plug, an IT/TT installation, or a
broken neutral, these three wires sit at up to 230 V relative to earth. The only present
mitigation is the `HIGH VOLTAGE !!!` silkscreen zone.

**Fix — pick one:**
- **(a, no respin)** Document that X3 is strictly for wiring sealed inside the same enclosure,
  never a panel-mounted button or LED. Put that on the silkscreen *next to X3*, not only in the
  general HV zone marking.
- **(b, respin)** One optocoupler per signal (PC817-class for SW, one for LED) so X3 lands on
  the `GND1` side.
- **(c, respin)** Move to a 6-channel isolator and carry both signals across the existing
  barrier.

#### S4 — Card-edge `L_IN`/`L_OUT` creepage collapses after assembly *(shared with the module)*

On the module, `J1` pad 1 (A1, F.Cu) and pad 2 (A2, B.Cu) sit at the **same X/Y on opposite
faces** of the 1.0 mm tab, with the pad edge only **0.143 mm** from the tab end. Post-assembly
creepage from `L_IN` to `L_OUT` around the tab end is ≈ 0.14 + 1.0 + 0.14 ≈ **1.3 mm** — before
solder fillets, which on a hand-soldered card edge wick around the end and can plausibly bridge
it.

On this board the same pair is held at **1.200 mm** across the slot, permitted by the
`relay_pads_clearance` exception in `MySXRelay.kicad_dru`. That 1.2 mm is an air gap across a
through-slot, which is fine on a bare board — but the slot is filled by the module once
assembled, and the path above is what remains.

**Fix (respin, both boards together):** set the pads back ≥ 1 mm from the tab end on both faces,
or split A1 and A2 onto two separate tabs the way the logic pads are already separated. Both
`edge_connector.kicad_mod` copies must change in step — see **P1**.

#### S5 — Mains input protection

- **TH1's mains rating is unverified.** `ERF-LX005V2`, sourced in the BOM as TME `LX005-V2`,
  which TME lists under *polymer THT fuses*. It is the only overcurrent device on the SMPS tap
  from `L_IN`. Most polymer PTCs are rated ≤ 60 V and have no defined breaking capacity at
  230 V; if D5, C5 or U3 fails short, TH1 has to interrupt a mains fault. **Check the
  datasheet**; if it is a PTC, replace with a rated mains fuse or a flameproof fusible resistor.
- **No overcurrent protection on the switched load path** (`X2.4` `L_IN` → relay → `L_OUT` →
  `X2.3`). Defensible if you rely on the building's breaker, but then the maximum load current
  needs stating. The weakest link is X2: the footprint is a Phoenix MKDS-1,5-4-5.08 (8 A part),
  while the BOM substitutes 2 × generic `XY301V-2P` — check both its rating *and* whether its
  body fits the Phoenix footprint on a board this tight.
- **No MOV, no X-capacitor.** C5 is **2.2 µF/400 V**; 230 V +10 % rectifies to ~358 V peak,
  leaving only ~12 % margin before any line transient. Add an MOV across L-N ahead of D5 and
  move C5 to 450 V.
- No CM choke or X-cap also means nothing attenuates the buck's conducted emissions. Fine for
  personal use; not CE-ready.
- Terminal block pin order is `N_OUT, N_IN, L_OUT, L_IN` — inputs and outputs interleave.
  Grouping as `[L_IN, N_IN][L_OUT, N_OUT]` would be harder to mis-wire.

### Functional / reliability

#### F1 — `REL` has no pull-down anywhere *(highest-value single fix)*

Net `D5` on this board is exactly `{U6 PD5, J3.3}`; on the module, `Net-(J1-REL)` is exactly
`{J1.3, U1 IN_A}`. No resistor in either. PD5 is high-Z from power-on until firmware configures
it — and during reset, ISP programming, and any bootloader delay. The MIC4427's inputs are
high-impedance with no internal pull.

A floating input can park near the threshold (driver shoot-through) or glitch, and a glitch here
is a **spurious mains switching event** on a relay that then latches. Note the failure is silent:
nothing on this board can detect that the relay moved.

**Fix:** 100 k from `J1.3` to `GND` **on the module** (protects it even standalone) — see the
module review. Adding one here too is cheap insurance. In firmware, make "configure PD5 as
output, drive low" the first instruction after reset.

#### F2 — No crystal; the internal 8 MHz RC clocks both serial links

`XTAL1/PB6` and `XTAL2/PB7` are unconnected. The factory-calibrated internal RC is ±10 % across
the full voltage/temperature range (±1–2 % only at nominal), while async UART needs better than
~±2 % split between both ends. This board self-heats (mains buck + relay) inside a sealed 40 mm
case, so it will not run at nominal.

**Fix:** PB6/PB7 are free — add a crystal on the next respin. Meanwhile calibrate `OSCCAL` at
runtime against the RS-485 master's traffic, or auto-baud on the first frame.

#### F3 — 1 MΩ pull-ups on `/RESET` (R22) and SPI memory `/CS` (R20)

Atmel AVR042 specifies 4.7 k–10 k on `/RESET`. 1 MΩ is ~100× too weak and leaves the reset pin
effectively floating next to a switching mains buck. 1 MΩ on `/CS` is likewise weak enough that
leakage plus capacitive coupling from the adjacent SPI lines can assert it.

**Fix:** 10 k for both. Drop-in, same footprint, no layout change.

#### F4 — The BL0942's UART RX shares the SPI MOSI net

`BL0942_RX` = `{U2.9 RX/SDI, U6 PB3/MOSI, U7.5 MOSI, J1.4 ISP MOSI}`. Every byte clocked to the
memory, and every byte of an ISP programming session, lands in the metering IC's UART receiver.
Framing plus checksum makes an accidental valid write command unlikely but not impossible.

**Fix (now, firmware):** periodically read back and verify the BL0942's configuration registers;
prefer reads over writes.
**Fix (respin):** the ATmega328PB has a **second SPI port on PE0–PE3, and PE0, PE1, PE2 and PE3
are all unconnected in this design.** Move U7 to SPI1 and the two buses separate cleanly with no
added parts.

#### F5 — BL0942 `SEL` (pin 7) and `SCLK_BPS` (pin 8) left floating

Not a defect — the datasheet specifies internal pull-downs, giving UART @ 4800 bps, which is
what the design wants. But relying on an unspecified-value internal pull on a mains-referenced
board next to a 2 MΩ divider is fragile. Add 10 k to `N_IN` on both in a respin, and decide
deliberately whether you want the faster baud option.

#### F6 — RS-485 front end

- **No fail-safe biasing, no termination footprint.** ST485E/SP485 has no true fail-safe
  receiver; on an idle, unbiased bus the receiver output is indeterminate and the USART sees
  framing errors or a break condition. Fine *if* the bus master biases — otherwise add
  680 Ω–1 k bias (at one node only) and a fitable 120 Ω termination.
- **D9/D10 = SMAJ11CA is the wrong TVS.** Bidirectional ±11 V standoff clamping to ~18 V, while
  RS-485 transceiver bus pins are typically ±14 V absolute max — the clamp lets through more
  than the transceiver survives. **SM712** (−7 V / +12 V asymmetric) is the industry-standard
  RS-485 TVS and matches the common-mode range properly.
- The ordering *is* correct as drawn: polyfuse on the connector side, TVS on the transceiver
  side.

#### F7 — Bus supply voltage is undocumented, and Q2 constrains it

Q2 = `BSS84A` with its gate tied directly to `GND1`, so **V_GS = −V_bus**, and BSS84A is rated
**±20 V maximum**. Continuous I_D in SOT-23 is ~130 mA. U1 = AP2204RA-5.0 (V_in max 24 V,
SOT-89 — good call in `d7353fb`): at 20 V in / 5 V out it dissipates ~0.3 W at 20 mA and
approaches 1 W if the ST485E drives a loaded bus. Nothing in the repo states V_bus.

**Fix:** document V_bus and silkscreen it next to X1. If 24 V is ever possible, add a 12–15 V
zener from Q2's gate to source plus a 100 k series gate resistor. If bus current matters,
consider a small buck instead of the LDO.

#### F8 — Latching-relay state is write-only

The module only changes state on a `REL` edge and has no auxiliary contact, so the MCU cannot
read the relay back. This works, but only under a strict firmware rule: **persist the commanded
level to non-volatile memory *before* driving the edge**, and at boot restore that level without
toggling. There is also no way to force a known state at boot without physically cycling the
relay — a deliberate limitation worth documenting.

**You already have the feedback path:** BL0942 current is an indirect contact-state reading.
Cross-checking measured current against the commanded state also catches welded contacts.

#### F10 — Smaller items

- **No ceramic at U5's input.** `+5V` carries only C7, a 470 µF electrolytic. AP2127/MCP1703-class
  LDOs want ≥ 1 µF ceramic close to V_IN. Add a 1 µF 0603.
- **C7 470 µF/10 V on a 5 V rail** is only 2× margin, electrolytic, in a sealed case next to a
  mains buck and a 330 µH inductor. Specify 105 °C long-life, or go to 16 V.
- **BL0942 voltage channel is under-driven.** 5 × 390 k over **510 Ω** = 1:3824 → 85 mV peak at
  230 V. Reference designs use ~1 kΩ at the bottom (≈1:1950, ~165 mV). You are giving up ~6 dB
  of SNR on the voltage channel; check the BL0942 full-scale spec and consider R13 = 1 kΩ.
- **LED currents are ~1.3 mA** (R6 and R19 both 1 k from 3.3 V). Fine for modern
  high-efficiency LEDs, dim for anything else.

### Project hygiene

#### P1 — Nothing records the interface contract with the module *(shared)*

The two boards mate over a 5-pin card edge with real constraints — 1.0 mm module PCB into
1.2 mm slots, pin assignment, a 5 V/~40 mA pulse budget, `REL` edge semantics and pulse width,
mains-spacing assumptions — and none of it is written down. The two `edge_connector.kicad_mod`
copies have **already drifted**: module pad X positions are consistently 0.015 mm off this
board's.

**Fix:** a shared footprint library (git submodule, or a common `kicad/Library.pretty`) plus a
short `INTERFACE.md` committed to **both** repos.

#### P2 — `3dshapes/bistable-relay.step` may be stale, and 7 courtyard errors point at J3

The STEP is dated 2024-11-26, the same day as the module commit `326cecd "move signal pads to
bottom layer"` (16:27) — the commit that changed exactly the mating geometry — and it predates
`64f8d0c` (2024-12-12). Mechanical fit may have been checked against an outdated model.

Relatedly, the 8 remaining DRC errors are all courtyard overlaps, and **7 of them are J3 against
R13–R18 / C12** (the voltage divider). Either the module really does foul those parts, or J3's
courtyard is over-generous. Resolve it in 3D rather than leaving 8 red errors that will hide the
next real one. (The 8th, F1 vs F2, is trivial.)

#### P3 — Stale artifacts on disk

- **`MySXRelay.csv`** — untracked, gitignored, and **wrong**: it still lists
  `U5 = MCP1703Ax-180xxTT` (a 1.8 V part on a 3.3 V rail) and `Q2 = ~`, where the schematic and
  the tracked `MySXRelay.ods` correctly say `AP2127N-3.3` and `BSS84A`. Delete it before someone
  orders from it.
  *The tracked `.ods` BOM is fully current* — supplier links, 55.44 PLN total. That one is in
  good shape.
- **`MySXRelay (copy).ods`** — untracked duplicate, delete.
- **`plots/`** (2024-12-06 gerbers) predates the last three PCB commits (`dd71edb` HV cutouts,
  `3f3a15d`, `ae69c83` vias out of pads). Gitignored, so local-only — but do not fabricate from
  it; regenerate.
- `MySXRelay.bak`, `MySXRelay.kicad_sym` (leftover local symbol lib) are clutter.

#### P4 — Stale netclass configuration in `MySXRelay.kicad_pro`

- `netclass_assignments` references a class named **`Mains_LV` that does not exist** in
  `classes`. The nets assigned to it — `Net-(U2-IN)`, `Net-(U2-IP)`, `Net-(U2-VP)`,
  `Net-(U5-VI)` — silently fall back to `Default` (0.2 mm clearance). Electrically harmless here
  (all are low-voltage with respect to `N_IN`), but the `MAINS*` DRC rule does not cover them and
  the intent is lost.
- Assignments exist for nets that no longer exist: `Net-(U8-ADJ)`, `Net-(U8-BYP)` (there is no
  U8), `Net-(U3-CS)`, `Net-(D5-K)`.
- **`N_OUT` is pattern-assigned to `Default`** yet it is a mains net carrying the full load
  current through R12. It happens to be routed 2.0 mm wide, so it is fine in practice — but it
  belongs in `Mains_HC` so the rules actually protect it.

#### P5 — The `.kicad_dru` relaxations are undocumented

Every exception drops below the 2.48 mm baseline set everywhere else, and a reader cannot tell
an engineered exception from a DRC-silencing hack:

| Rule | Value | Assessment |
|---|---|---|
| `relay_pads_clearance` `L_IN`↔`L_OUT` | 1.2 mm | across a through-slot on the bare board; see **S4** for what it becomes after assembly |
| `lsense_pads_clearance` `L_IN`↔`L_SENSE` | 1.8 mm | fine — `L_SENSE` is behind R18 (390 k), essentially at `L_IN` potential |
| `bp2525_pads_clearance` `CS`↔`DRAIN` | 0.92 mm | forced by the SOT-23-5 0.95 mm pin pitch; unavoidable and datasheet-sanctioned — say so explicitly |
| `input_cap_clearance` `N_IN`↔`DRAIN` | 1.9 mm | functional insulation at 325 V; above the 1.5 mm clearance need, below the 2.5 mm creepage guide |

Note also that `A.NetClass == 'MAINS*'` does match `Mains_HC`/`Mains_HV` (KiCad's `==` is a
case-insensitive wildcard compare) — but writing `'Mains*'` would make that obvious rather than
accidental.

**Fix:** comment each rule with the working voltage, the insulation class claimed, and why the
reduction is acceptable.

---

## What is already right

Worth stating, because a review that lists only problems is misleading:

- **The isolation channel is clean through the entire stackup** — 0.000 mm² of zone overlap on
  every adjacent layer pair, including across the 0.10 mm prepreg. KiCad checks clearance
  per-layer only, so this was done by hand; most 4-layer mains designs get it wrong.
- **The card edge keeps mains and logic 10.3 mm apart**, on both boards and both faces, with the
  tabs separated by a notch.
- **The shunt is in the load's neutral return**, so the node's own supply current is correctly
  excluded from the energy measurement.
- **The module's 1.0 mm stackup is deliberately matched to the 1.2 mm slot** (0.2 mm play), and
  the tab widths match the slot widths with ~0.4–0.5 mm side play.
- **Si8642 channel directions are correct** (2 forward / 2 reverse for a `Si8642`), the unused
  reverse channel's input `B4` is tied to `GND1` rather than left floating, and the ST485E's
  `DE`/`/RE` are correctly ganged.
- **The polyfuse-then-TVS ordering on the RS-485 lines is correct** (F1/F2 on the connector
  side, D9/D10 on the transceiver side).
- **The `.kicad_dru` shows real thought about mains spacing** — the gap is documentation and two
  specific values, not the approach.
- Schematic/PCB parity and connectivity are clean: `schematic_parity` and `unconnected_items`
  are both empty.

---

## Change plan

Ordered by value. **[respin]** needs a new board; everything else is a file edit, firmware, or
BOM change.

### Now — no respin

1. R22 and R20 → **10 k**. *(F3)*
2. D9, D10 → **SM712**. *(F6)*
3. Correct U4's schematic `Value` to the part actually fitted. *(S1)*
4. Verify TH1's mains rating; replace if it is a ≤ 60 V polymer PTC. *(S5)*
5. `MySXRelay.kicad_pro`: define `Mains_LV` (or reassign those nets), move `N_OUT` to
   `Mains_HC`, delete assignments for `Net-(U8-*)`, `Net-(U3-CS)`, `Net-(D5-K)`. *(P4)*
6. Comment every exception in `MySXRelay.kicad_dru`. *(P5)*
7. Delete `MySXRelay.csv` and `MySXRelay (copy).ods`; regenerate `plots/`; resolve or
   deliberately waive the 8 courtyard errors. *(P2, P3)*
8. Write `INTERFACE.md` into both repos; consolidate `edge_connector.kicad_mod` into one shared
   library. *(P1)*
9. Re-export `3dshapes/bistable-relay.step` and re-check the J3 vs R13–R18 fit in 3D. *(P2)*
10. Silkscreen/docs: mark X3 as mains-referenced and not for panel controls; mark the expected
    bus voltage next to X1; state the maximum load current next to X2. *(S2, S5, F7)*

### Firmware

11. Drive PD5 low as the first instruction after reset. *(F1)*
12. Persist the commanded relay level to NVM *before* toggling `REL`; restore without toggling
    at boot; cross-check against BL0942 current and flag disagreement. *(F8)*
13. Calibrate `OSCCAL` at runtime, or auto-baud. *(F2)*
14. Periodically read back and verify BL0942 configuration registers. *(F4)*
15. Enable BOD — the board loses power without warning and holds relay state in NVM.

### Next respin

16. **[respin]** Widen the mains↔RS-485 barrier to ≥ 5 mm creepage; fit a wide-body SOIC-16W
    reinforced isolator; add a milled slot along the barrier. *(S1)*
17. **[respin]** Card-edge pads set back ≥ 1 mm from the tab end, or A1/A2 on separate tabs —
    both footprints together. *(S4)*
18. **[respin]** Optocouple X3's LED and SW, or move to a 6-channel isolator. *(S2)*
19. **[respin]** Move U7 to the second SPI port (PE0–PE3, all currently unused). *(F4)*
20. **[respin]** Crystal on PB6/PB7. *(F2)*
21. **[respin]** MOV across L-N ahead of D5; C5 → 450 V; 1 µF ceramic at U5's input;
    C7 → 16 V/105 °C; 10 k pull-downs on BL0942 `SEL` and `BPS`; Q2 gate zener + 100 k if V_bus
    can exceed 20 V. *(S5, F5, F7, F10)*

---

## Verification

### Re-measuring any spacing claim

KiCad has no "report minimum distance between two net groups" command, but a throwaway probe
rule plus DRC does it exactly, zones included:

```bash
mkdir -p /tmp/probe && cd /tmp/probe
cp ~/Projekty/My/kicad/MySXRelay/MySXRelay.kicad_{pcb,pro} .

cat > MySXRelay.kicad_dru <<'EOF'
(version 1)
(rule probe
  (condition "A.NetClass == 'EXT*' && B.NetClass != 'EXT*'")
  (constraint clearance (min 8mm)))
EOF

kicad-cli pcb drc --format json -o d.json MySXRelay.kicad_pcb
# every violation reports the real distance; the smallest is the answer
```

Two gotchas: **copy the `.kicad_pro` too** — netclass definitions live there, not in the PCB —
and the DRC report honours the locale, so distances print with a comma decimal separator
(`obecnie 2,9630 mm`).

Swap the condition to probe other pairs, e.g. `A.NetName == 'L_IN' && B.NetName == 'L_OUT'`.

### Re-checking the interlayer barrier

KiCad's clearance DRC is **per-layer only** and will never catch a vertical violation. To verify:
render each zone's `filled_polygon` geometry per layer — those coordinates are absolute in the
file, so no footprint transforms are involved, and a nonzero-winding fill handles the keyhole
cuts correctly — then intersect the isolated-domain zones against the mains-referenced zones for
each adjacent layer pair. Today this is 0.000 mm² on all three pairs; it must stay that way
after any replane.

### Connectivity and parity

```bash
kicad-cli sch export netlist --format kicadsexpr -o out.net MySXRelay.kicad_sch
kicad-cli pcb drc --severity-error --severity-warning --format json -o drc.json MySXRelay.kicad_pcb
```

`schematic_parity` and `unconnected_items` are both empty today — keep them that way. Target:
the 8 courtyard errors also go to zero.

### BOM vs schematic

The committed `.ods` is current but the stray `.csv` was not; re-diff after any value change:

```bash
kicad-cli sch export bom --fields 'Reference,Value,Footprint' --group-by Value \
  --format-preset CSV -o fresh.csv MySXRelay.kicad_sch
```

### Bench

- Power-cycle 20× with the relay in each state and confirm no spurious switching. This is the
  test that would have caught **F1**.
- Confirm the relay does not change state during an ISP programming session.
- Hi-pot the mains-to-`GND1` barrier at the level you intend to claim, before and after the
  **S1** respin.

---

## References

- [Skyworks Si864x datasheet](https://www.skyworksinc.com/-/media/Skyworks/SL/documents/public/data-sheets/si864x-datasheet.pdf)
  — package variants and certified creepage
- [Skyworks AN583 — safety and layout for digital isolators](https://www.skyworksinc.com/-/media/SkyWorks/SL/documents/public/application-notes/AN583.pdf)
- [TME PAI142M31](https://www.tme.eu/en/details/pai142m31/optocouplers-and-digital-isolators/2pai-semi/)
  — the part actually sourced: 4-channel, SO16, 3 kVrms
- [BL0942 datasheet v1.06](https://www.belling.com.cn/media/file_object/bel_product/BL0942/datasheet/BL0942_V1.06_en.pdf)
  — `SEL`/`BPS` internal pull-downs, UART vs SPI mode
- IEC 60664-1 / IEC 62368-1 — creepage and clearance tables used above
  (230 V rms, OVC II, PD 2, material group IIIa)
