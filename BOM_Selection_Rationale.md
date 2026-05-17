# 📋 BOM Selection Rationale — POV Display Project

> เอกสารสรุปการเลือก components สำหรับ POV Display 4-row 64-LED Project
> เนื้อหา: เหตุผลการเลือกแต่ละชิ้น พร้อมการคำนวณและการตรวจสอบ

**Project:** Rotating LED Display (Persistence of Vision)
**Boards:** 3 บอร์ด
1. **Power_and_MotorV2** — Base station (Motor controller + Royer driver)
2. **Royer_coil** — Coil ring PCB (Primary + Coupling)
3. **LED_DisplayV2** — Rotor (Secondary coil + Rectifier + Arduino + LED matrix)

---

## 🎯 System Architecture

```
┌─────────────── STATIONARY BASE ──────────────────────┐
│                                                       │
│  Power_and_MotorV2 (110×110mm, 2-layer)              │
│  ┌─ J1 DC Jack 12V                                   │
│  ├─ U1 LM317 + RV1 1k pot → M1 Motor                 │
│  └─ Royer Driver: Q1/Q2 BC337 + L5 + C2 + R1/R2/R3   │
│                              │                        │
│  Royer_coil PCB (98mm OD ring, 2-layer)              │
│  L1 Primary + L2 Coupling spiral                      │
└──────────────────────────╫───────────────────────────┘
                          ║ 120 kHz wireless (air gap 3-5mm)
                          ║
┌────────────── ROTATING ROTOR ───────────────────────┐
│  LED_DisplayV2 (120mm dia, 4-layer)                  │
│  ┌─ L1 Secondary coil → D1-D4 bridge → C1,C2 220µF  │
│  ├─ U10 AMS1117-5.0 → +5V plane                     │
│  └─ A1 Arduino Nano + 8×TPIC6C595 + 64 LEDs + Hall  │
└──────────────────────────────────────────────────────┘
```

---

## 📦 Bill of Materials (Final Selection)

## 1️⃣ Power_and_MotorV2 (Base Station)

### Capacitors

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **C1, C6** | 100µF | **EEU-FM1E101** (Panasonic FM series) | Aluminum electrolytic 25V, low ESR, smoothing on power input/output rails |
| **C2** ⭐ | **33nF** | **B32672L6333K000** (TDK MKP) | **Royer resonance cap** — design value 33nF (not 100nF per README) to set 120kHz frequency with primary coil L1. **MKP polypropylene** required for high-frequency switching (low ESR, no DC bias), **630V rating** safe for resonant peak voltage |
| **C3, C4, C5** | 100nF | **168104J50A-F** (Cornell Dubilier MKT) | Film cap 100nF/50V — bypass on D1 output (C3), +12V rail (C5), and snubber/filter (C4). **Polyester film** chosen over ceramic because: no DC bias effect (ceramic loses 25% at 12V), better noise filtering for motor + Royer circuit |

**Why C2 = 33nF instead of 100nF (per README)?**
- Design uses **BC337 BJT** Royer driver (not IRFZ44N MOSFET per RD40 reference)
- Topology requires different LC tank for 120kHz @ given primary coil inductance
- Formula: f = 1/(2π√LC) → with C=33nF and target 120kHz → L₁ ≈ 53µH

### Diodes

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **D1** | SS14 | **SS14** (onsemi) | Schottky 1A/40V SMA SMD — fast recovery, low Vf for input protection/filtering |
| **D2** | 1N4001 ⚠️ | **1N4004-T** (Lumimax) | ⚠️ BOM shows 1N4004 (400V) instead of 1N4001 (50V) — both work, 4004 has higher voltage margin. Used for back-EMF clamping/protection |

### Inductors

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **L5** | 330µH | **AIUR-06-331K** (Abracon) | **Royer choke** between +12V and primary center-tap. Specs verified: **Isat 1.62A, I_rated 1.32A, DCR 0.385Ω** — gives 4× safety margin on 333mA average Royer current. Drum core, 9.4×4.4mm fits L_vertical12 footprint at 5.08mm pitch |

**Note:** Initial concern about saturation current was incorrect — verified from Digikey datasheet that AIUR-06-331K has Isat=1.62A (not 280mA as initially assumed)

### Transistors

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **Q1, Q2** | BC337 | **BC33725TFR** (onsemi) | NPN BJT Royer driver pair. Chosen over IRFZ44N MOSFET (RD40 reference) for: lower cost, simpler drive (no gate drive circuit needed), adequate for 4-5W Royer output, Ic max 800mA covers peak current |

### Resistors

| Ref | Value | Tolerance | Part Number | Reasoning |
|---|---|---|---|---|
| **R1** | 2.2kΩ | ±5% | **CFR-25JB-52-2K2** (Yageo) | **Royer base bias** — provides startup current to Q1 (BC337). I = 11.3V/2.2kΩ = 5.1mA, P = 57mW < ¼W rating |
| **R2** ⭐ | 270Ω | **±1%** | **MFR-25FBF52-270R** (Yageo) | **LM317 R_top** voltage divider — sets motor voltage range. Precision 1% chosen because directly affects V_motor accuracy (±1% gives ±1.4% V_out, vs ±5% gives ±4%) |
| **R3** | 100Ω | ±5% | **CFR-25JB-52-100R** (Yageo) | **LM317 ADJ filter** — RC decoupling with C4 (100Ω × 100nF = 10µs time constant). Low current (~50µA from ADJ pin), 5% adequate |

### Potentiometer & Regulator

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **RV1** | 1kΩ | **P160KN-0QD15B1K** (TT Electronics/BI) | Motor speed control pot. With R2=270Ω: V_motor = 1.25V (off) to 5.88V (max). Linear taper for predictable speed adjustment |
| **U1** | LM317 | **LM317AT/NOPB** (Texas Instruments) | Variable LDO regulator TO-220 vertical mount. Drives motor with adjustable voltage. TI brand for reliability + datasheet support |

### Connectors & Switch (Pending)

| Ref | Value | Status | Notes |
|---|---|---|---|
| **J1** | Barrel_Jack_Switch_Pin3Ring | 🟡 ใช้ Gravitech CON-SOCJ-2155 (custom footprint) | DC Jack 2.1mm with switch contact |
| **SW1** | SW_DPDT_x2 | ❌ Not yet selected | Recommend: **C&K JS202011AQN** (Digikey CKN12025-ND) — matches footprint SW_CK_JS202011AQN_DPDT_Angled |

---

## 2️⃣ LED_DisplayV2 (Rotor)

### Capacitors

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **C1, C2** ⭐ | 220µF | **EEU-FR1E221B** (Panasonic FR, **25V**) | Rectifier output smoothing. **25V chosen over 16V** (EEU-FR1C221B) for safer margin against V_raw transients (peak ~12V, surge could reach 15V). Low-ESR 105°C 10,000h life — premium for reliable operation under Royer noise |
| **C3-C10** | 100nF (×8) | **CL10E104KC8VPNC** (Samsung MLCC) | TPIC6C595 + AMS1117 + Arduino decoupling. 0603 SMD X7R 25V. Acceptable DC bias at 5V (~10% loss → effective ~90nF), suitable for HF decoupling role |
| **C11** | 22µF tantalum | (TBD — keep as designed) | AMS1117 output stabilization (required per datasheet for stable operation) |
| **C12** | 100nF | (TBD — SMD 0603) | Additional HF bypass |
| **C13** | 1µF | (TBD — SMD 0805) | Mid-frequency bypass |

### Diodes

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **D1-D4** | 1N5818 ×4 | **1N5818** (STMicroelectronics) | **Bridge rectifier** for secondary coil output. Schottky low Vf (0.55V max), fast recovery (<10ns) for Royer 120kHz operation. 1A/30V package DO-41 horizontal THT |
| **D5-D68** (64 LEDs) | LED | **XL-257UWC-L** (Xinglight) | **64 Ultra White Cool LEDs** for POV display. Rectangular 3×2mm THT, lead pitch 2.54mm, fits LED_Rectangular_W3.0mm_H2.0mm footprint. Vf 3.3V, If 20mA |

**Note on LED count:** Excel lists 63 references (D4-D66) but design has 64 LEDs (D5-D68 in 4 rows of 16). Likely transcription error in BOM — verify before ordering.

### Resistors

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **R1** | 20kΩ ±1% | **MFR-25FRF52-20K** (Yageo) | Hall sensor (A3144) pullup. 20k chosen for low current draw (5V/20k = 250µA) while keeping rise time < 1µs (matches A3144 datasheet recommendation). 1% precision overkill but cheap |
| **RN1-RN8** | 100Ω | **4609X-101-101LF** (Bourns) | **LED current limit network** — 9-pin SIP bussed (8 resistors + common). 100Ω chosen for White LEDs at +5V: I = (5-3.3)/100 = 17mA per LED — bright enough for POV at 30% duty cycle, within TPIC6C595 sink rating |

**Why 100Ω instead of 470Ω (per README)?**
- README spec assumed Red LEDs (Vf 2V), but design uses White LEDs (Vf 3.3V)
- 470Ω with White LED → only 3.6mA = too dim for POV at 2000 RPM
- 100Ω with White LED → 17mA = visible during rotation

### ICs

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **U3-U10** | TPIC6C595N ×8 | **TPIC6C595N** (Texas Instruments) | **64-bit shift register cascade** (8 chips × 8 outputs). Open-drain DMOS outputs sink 100mA/channel, perfect for direct LED current sinking. DIP-16 THT |
| **U1** | A3144 | (TBD — Allegro genuine) | Hall effect sensor for rotation index trigger. Custom footprint, 3-pin |
| **U10** | AMS1117-5.0 | **EVVO AMS1117-5.0** (verified from Digikey) | LDO regulator V_raw 7V → +5V. SOT-223 with tab=Vout. 1A capability, 1.2V dropout adequate for 7V input |
| **A1** | Arduino Nano v3.x | ❌ Not yet selected | Use genuine **Arduino Nano** (ATmega328P, 5V) or compatible. **Avoid Nano ESP32** (3.3V logic) |

---

## 3️⃣ Royer_coil (Coil PCB)

| Ref | Value | Part Number | Reasoning |
|---|---|---|---|
| **L1 Primary** | (PCB spiral) | **On PCB** | Designed in PCB traces — 1.5mm wide spiral on F.Cu + B.Cu with vias |
| **L2 Coupling** | (PCB spiral) | **On PCB** | Inner coil for feedback to Q1/Q2 BC337 gates |
| **L3, L4, L5** | (Coil sections) | **On PCB** | Various coil sections |

**Connection terminals:** 4 footprints on Royer_coil PCB
- 2× Pin_2mm (test points / inner taps)
- 2× THT pads at (120.5, 137.5) and (179.5, 137.5) — main coil terminals

---

## 🔴 Items Still Pending Selection

| Board | Ref | Value | Recommendation |
|---|---|---|---|
| LED_DisplayV2 | **A1** | Arduino Nano | **Genuine Arduino Nano v3.x** (ATmega328P, 5V logic) — avoid ESP32 variant. ~$5-25 depending on source |
| Power_and_MotorV2 | **SW1** | DPDT switch | **C&K JS202011AQN** (Digikey #CKN12025-ND) — matches footprint exactly. ~$5 |
| LED_DisplayV2 | **U1** | A3144 Hall sensor | Allegro A3144 from genuine source (avoid AliExpress) — through-hole TO-92 |
| LED_DisplayV2 | **C11, C12, C13** | Various | C11 = 22µF/10V tantalum 3528; C12 = 100nF 0603; C13 = 1µF 0805 |

---

## 📐 Key Design Decisions Summary

### Deviations from README/RD40 Reference

| Item | RD40 Reference | Our Design | Reason |
|---|---|---|---|
| Royer driver | IRFZ44N MOSFET ×2 | **BC337 BJT ×2** | Lower cost, simpler drive, adequate for 4-5W |
| Resonance cap C2 | 100nF | **33nF** | Match BC337 driver characteristics + actual L1 inductance |
| LED color | (assumed Red) | **White XL-257UWC-L** | User preference for white color |
| LED current resistor | 470Ω | **100Ω** | Compensate for White LED higher Vf (3.3V vs 2V) |
| Bridge rectifier | SS14 SMA | **1N5818 DO-41 THT** | Easier hand assembly (THT) |
| Bulk caps | 220µF/16V | **220µF/25V** | Safer voltage margin |
| Arduino | (Nano v3.x) | **Nano v3.x ATmega328P** | Confirmed — not ESP32 variant |

### Critical Design Constraints

1. **Mass Balance** (2000 RPM rotor) — heavy components (C1, C2 220µF, Arduino) should be placed symmetrically
2. **EMI Shielding** — 4-layer stack with internal GND (In1.Cu) + +5V plane (In2.Cu) ✓
3. **Coil Keepout** — Inner planes must be cleared under secondary coil L1 on rotor (no eddy current loss)
4. **Edge Clearance** — Outermost LEDs at r=57mm vs board edge r=60mm → 3mm clearance (tight)

---

## 📊 Precision Tolerance Strategy

| Type | Tolerance | Used For |
|---|---|---|
| **±1% Metal Film** | High precision | R2 (LM317 feedback — affects V_motor), R1 LED board (Hall pullup) |
| **±5% Carbon Film** | Standard | R1 (Royer base bias), R3 (LM317 filter), other general resistors |
| **±10% Film Cap** | Standard | C2 MKP (Royer resonance — within range tolerable) |
| **±20% Aluminum Electrolytic** | Standard | C1, C2 (bulk smoothing — value not critical), C1, C6 power board |
| **±10% MLCC X7R** | Standard | Decoupling caps (DC bias acceptable) |

---

## 🛒 Supplier Strategy

### Primary: **DigiKey** ⭐
- Most components from DigiKey for genuine parts + reliable shipping
- Especially critical for: regulators (LM317), shift registers (TPIC6C595N), Schottky diodes (1N5818, SS14)

### Secondary: DigiKey Marketplace
- LEDs (XL-257UWC-L, 1N5818, 1N4004, TPIC6C595N)
- Generally lower-cost reliable sources

### Local Thailand
- J1 DC Jack: Gravitech CON-SOCJ-2155 (footprint designed for this)
- Mounting hardware / mechanical parts

---

## 📋 Pre-Order Checklist

- [ ] Verify A1 Arduino Nano — confirm ATmega328P version (NOT ESP32)
- [ ] Select SW1 DPDT switch — C&K JS202011AQN recommended
- [ ] Verify LED count in BOM (Excel shows 63 vs design 64)
- [ ] Source C11 (22µF tantalum 3528), C12 (100nF 0603), C13 (1µF 0805)
- [ ] Source U1 A3144 Hall sensor (genuine Allegro)
- [ ] Verify D2 1N4001 vs BOM 1N4004 — choose 1N4004 for higher margin
- [ ] Verify footprint match for L_vertical12 with AIUR-06 (5.08mm pitch)
- [ ] Verify all custom footprints (my_footprints library):
  - GRAVITECH_CON-SOCJ-2155
  - MKP10_630V
  - L_vertical12
  - Pin_2mm
  - coil_large
  - motor
  - A3144

---

## 🎓 Lessons & Notes for Documentation

1. **Always verify Digikey datasheet** before quoting specs (initial AIUR-06 saturation current was misjudged)
2. **README values may be outdated** — actual schematic values (e.g., C2 = 33nF not 100nF) reflect design evolution
3. **Footprint mismatch is the #1 issue** when substituting parts — always check pin count, pitch, body size
4. **DC bias effect** on MLCC ceramics is significant at low voltage (12V → 25% loss on 25V X7R)
5. **Film caps** preferred over ceramic for motor/Royer circuits (no DC bias, lower ESR)

---

*เอกสารนี้สรุปจากการ review PCB design + การคุย component selection กับ Claude*
*Updated: 2026-05-18*
