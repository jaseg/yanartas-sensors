# Review notes and board split (November 2026)

Branch `claude/review-split-pcbs`. One commit per topic:

| Commit | What |
|---|---|
| `cf9d0ef` | Vendored QFN footprints regenerated with kicad-footprint-generator (sources in `lib-src/`) |
| `be5c5f7` | Vendored libraries registered in `fp-lib-table`/`sym-lib-table`; symbols point to the new footprint names |
| `0d42cf1` | `EPCOS_B82723A`: holes enlarged (pins are 0.7 mm square) |
| `912a24a` | Wrong LCSC numbers fixed, U13 footprint fixed |
| `64542f1` | Split into `power/` and `sensing/` projects with an FH12 FFC interconnect |

The combined project in the repository root has not been changed and is kept for reference. You can delete it once you are happy with the split.

## 1. Board split

### Projects

- `power/yanartas-pillowbox-power.kicad_{sch,pcb,pro}`: the processing and power board.
- `sensing/yanartas-pillowbox-sensing.kicad_{sch,pcb,pro}`: the sensing board.
- Both use the shared libraries in the repository root through `${KIPRJMOD}/../` in their own `fp-lib-table` and `sym-lib-table`.

### Which parts went where

**Power board.** Everything that was placed inside the board outline. I also moved these CM5 support parts there, because they were still outside the outline:

- TP1–TP21 (CM5 GPIO, ETH_SYNC and USB2 test points)
- D15/R89 (LED_nPWR), D16/R136 (M.2 LED), D17/R138 (LED_nACT)
- R87, R88, R90, R91, R137 (wireless-disable, EEPROM_nWP, nRPIBOOT and PWR_BUT straps)
- U14, C104, R135 (microSD power switch)

**Sensing board.** Everything else.

### How the files were split

**Schematics.** I deleted the other board's symbols. Wires, labels, junctions, no-connect flags, bus items and netclass flags were deleted only where they became dangling. Wires that also served parts that stay were shortened instead of deleted.

I checked the result with a kicad-cli netlist export. For every pin that remains, the net is identical to the original combined schematic.

**PCBs.** I only deleted the other board's footprints and moved nothing. The board outline and the Dwgs.User rectangle stay on the power board. The sensing board has no outline yet. Every footprint still links to its schematic symbol.

### Interconnect J18 (power) ↔ J19 (sensing)

The link uses FH12S-30S-0.5SH(55) (LCSC C3169607), the same connector as J1, J2 and J4.

Pin numbers below are on the **power board** connector. The sensing-board pin is 31 − k:

| Power pin | Net | Sensing pin | | Power pin | Net | Sensing pin |
|---|---|---|---|---|---|---|
| 1 | GND | 30 | | 16 | GND | 15 |
| 2 | +5V | 29 | | 17 | HOST_TX | 14 |
| 3 | +5V | 28 | | 18 | HOST_RX | 13 |
| 4 | GND | 27 | | 19 | GND | 12 |
| 5 | +3V3D | 26 | | 20 | HOST_nCS | 11 |
| 6 | GND | 25 | | 21 | HOST_HOPI | 10 |
| 7 | PMIC_ENABLE | 24 | | 22 | GND | 9 |
| 8 | ~OUT_SECURE | 23 | | 23 | HOST_SCK | 8 |
| 9 | GND | 22 | | 24 | HOST_HIPO | 7 |
| 10 | ~OUT_ALARM | 21 | | 25 | GND | 6 |
| 11 | ~OUT_ACT | 20 | | 26 | +3V3D | 5 |
| 12 | GND | 19 | | 27 | GND | 4 |
| 13 | HOST_ALM | 18 | | 28 | +5V | 3 |
| 14 | HOST_ALMn | 17 | | 29 | +5V | 2 |
| 15 | GND | 16 | | 30 | GND | 1 |

The mounting pins (MP) are on GND.

**Cable assumption.** The pinout assumes a same-side-contact ("type A") FFC, with both connectors on the top side facing each other. In that arrangement a conductor on power pin k lands on sensing pin 31 − k.

- I checked the mapping net by net on both netlists.
- All supply and GND pins are mirror-symmetric (pin k and pin 31 − k carry the same supply). A reversed (type B) cable or a different board arrangement would therefore swap only signals, never supplies.
- If you stack or fold the boards differently, re-check which cable type gives pin k → 31 − k.

**Current capacity.**

- +5V uses 4 pins (about 2 A at 0.5 A per pin). The sensing board's +5V feeds U7 and, through J5, a USB host (see 2.1).
- +3V3D uses 2 pins.
- I added PWR_FLAGs where a supply now arrives over the cable: +3V3D on the power board, and +5V and GND on the sensing board.

### Still to do in KiCad

1. In each project, run **Update PCB from Schematic**. This brings in:
   - J18/J19;
   - the parts that were missing from the PCB because their libraries weren't registered (U2, U3, U4, U5, U7, U9, U10, U12);
   - the J4 pinout change from commit 8d78104, which never reached the PCB.
2. Draw the sensing board outline.
3. The logo footprint libraries listed in `fp-lib-table` are not in the repository: PTF-Logo, PTF-Logo-Mask, nlnet-logo, ngi-zero-commons-logo, BMFTR-Logo, ashen-logo and ashen-logo-text. Please add them.
4. The 3D model folders `CM5IO.3dshapes/` and `MDT420M02001.3dshapes/` are referenced but not in the repository either. The paths now point to the repository root.

## 2. Schematic review

### 2.1 Worth fixing

1. **J5 VBUS is tied directly to +5V** (sensing board, standalone USB-C).
   - If USB is plugged in while the 12–24 V input is present, the REC30K output and the USB host's VBUS are shorted together.
   - Without the 24 V input, the USB host would try to power the CM5 through J5 and the FFC.
   - Fix: add an ideal-diode power mux (for example TPS2116 or LM66200) or a VBUS-sense-only connection.
2. **C101/C108/C109/C110 are rated 25 V** (470 µF polymer on the REC30K input), but the input is 24 V nominal. Use at least 35 V, preferably 50 V, in the same 10 mm / 5 mm pitch case.
3. **C103 (Bob Smith termination cap to Earth) is 1 nF 50 V 0603.** It needs to be a 2 kV part, for example 1206 or 1812 2 kV C0G or X7R.
4. **PMIC_ENABLE is on STM32 PA5, a TT pin (3.6 V tolerant, not 5 V tolerant).** The CM5 pulls PMIC_Enable up to 5 V internally through 100 kΩ (CM5 datasheet, pin 99). Either:
   - move it to an FT pin used in open-drain mode, or
   - drive it with a small N-MOSFET.
5. **The microSD supply (U14 IN) comes from +3V3D.** After the split, +3V3D is generated on the sensing board (U7) and arrives over the FFC, so the CM5 cannot boot from SD without the sensing board attached. Feed U14 from the CM5's own 3.3 V output instead (RPI_3V3, pins 84/86, 600 mA total), as Raspberry Pi's IO boards do.
6. **LED_nPWR (D15) and LED_nACT (D17) drive LEDs directly, with the anodes on +3V3D.**
   - The CM5 datasheet says LED_nPWR "needs to be buffered".
   - +3V3D stays up while the CM5 is off, which back-feeds the CM5 GPIOs through the LEDs.
   - Fix: use RPI_3V3 for the anodes, or add a buffer.
7. **FFC ordering.** FH12S-30S-0.5SH(55) is the bottom-contact type with the strengthened flip-lock actuator. For the documented pin k ↔ 31 − k mapping, order a 30-way, 0.5 mm, same-side-contact (type A) FFC.

### 2.2 Checked and fine, though ERC complains

- **M.2 J7 PCIe.** KiCad's `Bus_M.2_Socket_M` names pins 41/43 "PERn0/PERp0" from the host side, which is the opposite of the M.2 spec's module-side naming. That is the source of the pin-type ERC errors.
  - The wiring here is CM5 RX_P/RX_N → pins 41/43 and TX_P/TX_N → pins 47/49, with P/N swapped on both pairs.
  - This is identical to Raspberry Pi's official M.2 HAT+ (which notes "P/N swapped for better routing"). PCIe receivers support lane polarity inversion.
  - REFCLK is not swapped.
  - The SSD carries its own TX AC-coupling caps, which the CM5 datasheet requires on the RX side.
- **U5 JTAGSEL_N tied to GND.** This permanently enables the JTAG pins, which matches the STM32 configuring the FPGA over JTAG.
- **U5 VCCIO1 at 1.2 V** (R54 fitted, R55 = 3.3 V option, DNP). This looks intentional.
- **CM5 VBAT (pin 76) on +3V3D.** This is within the 2.5–3.5 V spec.

### 2.3 Clean-up (ERC noise)

- The netclass `diffpair` is used by 144 netclass flags but not defined in the project settings. Define it, with the impedance you want, in `sensing/*.kicad_pro`.
- There are 131 `net_not_bus_member` warnings: the labels on the bus entries aren't members of the bus. You can either give the buses a name or alias, or delete them.
- `lib_symbol_mismatch` on U4, U7, U10, U12, U16, U17, U18 and U19: run "Update symbols from library".
- U1 (AP3441): the exposed pad is a hidden power-input pin named `PAD`, which creates a global net "PAD" merged with GND. Make the pin passive in the symbol.
- Unconnected U5 pins without no-connect flags: 14, 15, 22, 29, 30, 31, 32, 33.
- Net label `PCIE _3V3` contains a space.
- There is a 1.27 mm stub wire with an unconnected end on the power sheet (at 589.28, 431.8).
- Power-input pins without a driver: U5 VCCIO1, U8 IN, U11 ±Vin, J7 3.3V and a few GND/+3V3 symbols. These were already flagged in the original; add PWR_FLAGs where appropriate.
- TR1 (Ethernet magnetics) sits on the power board, while the earlier plan put the magnetics at the far end of the FFC. Please confirm which is intended.

## 3. Footprints

### 3.1 Regenerated with kicad-footprint-generator

The three vendored libraries (`HighSpeed_Interface`, `Gowin_FPGA`, `Power_Management`) were replaced with footprints generated by kicad-footprint-generator. The YAML is in `lib-src/`, and STEP models are included.

| Part | Package | Source |
|---|---|---|
| PI3HDX12311 | XEA32 X1-QFN-32 2.85×4.5, land 3.21/4.86 outer, pads 0.45×0.20, EP 1.75×3.4 | Diodes datasheet. Pins 14/30 are joined to the EP by a strip in the vendor land pattern; see `lib-src/README.md` |
| PI3WVR41310 | ZL52 TQFN-52 3.5×9, pads 0.6×0.2 at 3.5/9.0 centre-to-centre, EP 2.0×7.5 | Diodes datasheet |
| TMUXHS4212 | RKS0020A VQFN-20 2.5×4.5, pads 0.6×0.24, EP 1×3 | TI datasheet |
| GW1NZ-LV1QN48 | QN48 6×6, pads 0.85×0.2 at 5.9 centre-to-centre, EP 4.3 | Gowin UG |
| TMI7003C | QFN-20 3×3 P0.4, EP 1.7 | TMI datasheet |
| TMUXHS4446 | RET0040A | stock `Package_DFN_QFN:Texas_WQFN-40-1EP_3x6mm_P0.4mm_EP1.7x4.5mm` |

The vendored symbols match the datasheets' pin tables (PI3HDX12311, PI3WVR41310, TMUXHS4212, TMUXHS4446, GW1NZ-LV1QN48, TMI7003C). The only difference: some secondary pin names on the GW1NZ (MSPI/SSPI functions) are missing, which is cosmetic.

### 3.2 `footprints.pretty`, checked against manufacturer drawings

| Footprint | Result |
|---|---|
| CJIANG_FXL0420 | OK (CJiang land pattern: a=2.2, b=5.2, c=2.5 → 1.5×2.5 pads at ±1.85) |
| CONN-SMD_MDT420M02001 | OK (Amphenol drawing: 0.30×1.55 pads at 0.5 mm pitch, rows ±3.775, key M pins 59–66 removed, mount pads 1.2×2.75 at ±10.35, NPTH Ø1.1/Ø1.6) |
| TDK_ACM7060 | Geometry OK (9.0/4.0/4.5/1.5). The pad numbers differ from TDK's (TDK: 1 TL, 2 BL, 3 BR, 4 TR, windings 1–4/2–3), but they are consistent with KiCad's `L_Ferrite_Coupled` (windings 1–2/3–4), so the board is correct. Do not "fix" one side only |
| EPCOS_B82723A | Pitch (15×25), numbering and windings OK. **Drill was 1.0 mm for 0.7 mm square pins (0.99 mm diagonal) → changed to 1.3 mm** |
| RECOM_REC30K-2405SZ | OK (RECOM REC30K Rev. 1-2025: pin positions, Ø1.0 pins, Ø1.4 holes) |
| SINHOO_SMTSO20XX / 25XX | OK (holes 3.8 / 4.3 against 3.73 / 4.22 +0.08 mm, pads 7.0 against 6.2 mm minimum) |
| Raspberry-Pi-5-Compute-Module-A/B | OK, identical to the stock Hirose DF40C-100DS-0.4V (1.5 mm stack, matching the 1.5 mm SMTSO2515 standoffs J10–J13) |
| U1 AP3441 (stock DFN-8 2×2 EP1.05×1.75) | Acceptable (Diodes suggests EP 1.0×1.8, pads 0.35×0.5) |

## 4. MPN / LCSC check

### Fixed in `912a24a`

| Ref | Was | Problem | Now |
|---|---|---|---|
| R3–R6, R17–R20 (91 Ω) | C4147 | is 91 kΩ | C270574 |
| R141–R143 (470 Ω 0402) | C23179 | 0603 part | C25117 |
| R36 (18k2) | C25762 | is 18 k | C25859 |
| R34 (22k1) | C25768 | is 22 k | C43473 |
| U13 OPA4354 | SOIC-14 footprint | C527494 is TSSOP-14 | footprint → TSSOP-14 |
| U4, U12 | none | | C35471229 TMUXHS4446RETR |

I also cleaned up some fields:

- U1 had a duplicate `LCSC\`` field; removed.
- U6 had a duplicate `LSCS` field; removed.
- U8's `LSCS` field (TI original C408972) was renamed to `LCSC_Alt`. U8's `LCSC` field is the SiChange XTLV74333 clone C51953291.

### Stock risks (JLC stock, early November 2026)

- **STM32U575RGT6 (C5270980):** 9 pcs.
- **PI3HDX12311 (C49454029):** 5 pcs.
- **TMUXHS4446 (C35471229):** 41 pcs, and 2 are needed per board.
- **PI3WVR41310:** no stocked LCSC/JLC part (C18184981 had 0 stock and no longer resolves). You'd need to consign it.
- **REC30K-2405SZ and B82723A2402N001:** not available on LCSC. Both are THT, so hand-solder or consign them.
- **Reasonable stock:** TMUXHS4212 (882) and GW1NZ (684).
- **Low on LCSC but fine at JLC:** C1523, C23733, C45783, C114218.
