# POV_Display_PCB

# 📋 สรุปโปรเจกต์ POV Display 4 แถว LED

เอกสารสำหรับนำเสนออาจารย์ — สรุปการตัดสินใจดีไซน์ทั้งหมด
**สถานะปัจจุบัน:** PCB ออกแบบเสร็จ → กำลังสั่ง components เพื่อประกอบและทดสอบ

---

## 🎯 1. ภาพรวมโปรเจกต์

**ชื่อโปรเจกต์:** Rotating LED Display (POV - Persistence of Vision)

**อ้างอิงหลัก:**
- Hackaday RD40: Rotating LED Display by lhm0
- Royer Converter from Mikrocontroller.net

**ความต่างจากต้นฉบับ:**
| รายการ | RD40 ต้นฉบับ | โปรเจกต์ของเรา |
|---|---|---|
| จำนวนแถว LED | 2 แถว | **4 แถว** |
| LED ต่อแถว | 20 ตัว | 16 ตัว |
| LED รวม | 40 ตัว | **64 ตัว** |
| MCU | Arduino Nano + ESP-01s | **Arduino Nano เดียว** (ATmega328P 5V) |
| Image storage | ESP-01s WiFi | **PROGMEM (32KB Flash)** |
| ขนาด rotor | 120mm (CD size) | 120mm (เท่าเดิม) |
| Royer driver | IRFZ44N MOSFET ×2 | **BC337 BJT ×2** (cost-optimized) |
| Resonance cap | 100nF | **33nF** (matched กับ L1 จริง) |
| PCB strategy | 1 board เดียว | **3 boards แยก** (modular design) |
| LED board layers | 2 layers | **4 layers** (ground/power planes) |

---

## 🏗️ 2. System Architecture — 3 บอร์ดแยก

```
┌─────────────── STATIONARY BASE (ฐานคงที่) ──────────────────┐
│                                                              │
│  ┌─ Power_and_MotorV2 (110×110mm, 2-layer) ──────────────┐ │
│  │                                                         │ │
│  │  J1 DC Jack 12V                                        │ │
│  │   │                                                    │ │
│  │   ├─→ U1 LM317 + RV1 (1kΩ pot) ──→ M1 DC Motor       │ │
│  │   │   [Motor speed control]                           │ │
│  │   │                                                    │ │
│  │   └─→ Royer Driver:                                   │ │
│  │       Q1/Q2 BC337 + L5 330µH + C2 33nF + R1/R2/R3    │ │
│  │       [Self-oscillating @ 120 kHz]                    │ │
│  └──────────────┼────────────────────────────────────────┘ │
│                 │ wires                                      │
│  ┌──────────────▼── Royer_coil (98mm OD ring, 2-layer) ──┐ │
│  │   L1 Primary spiral + L2 Coupling spiral             │ │
│  │   (PCB traces 1.5mm, 1064 segments)                   │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────╫───────────────────────────────────┘
                          ║ 120 kHz wireless
                          ║ (air gap 3-5mm, k ≈ 0.4)
                          ║
┌─────────────────── ROTATING ROTOR (ส่วนหมุน) ────────────────┐
│                                                              │
│  ┌── LED_DisplayV2 (120mm dia, 4-layer) ──────────────────┐ │
│  │   Stack: F.Cu / In1(GND) / In2(+5V) / B.Cu, 1.6mm    │ │
│  │                                                        │ │
│  │   [L1 Secondary coil] → D1-D4 (1N5818 bridge)        │ │
│  │      ↓                                                │ │
│  │   C1, C2 (220µF/25V smoothing) → V_raw ~7V           │ │
│  │      ↓                                                │ │
│  │   U10 AMS1117-5.0 → +5V plane (In2.Cu)              │ │
│  │      ↓                                                │ │
│  │   A2 Arduino Nano + 8×TPIC6C595 + 64×LED + A3144     │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔧 3. หลักการทำงาน

### 3.1 Persistence of Vision (POV)

LED จำนวนน้อยหมุนเร็วๆ + เปิด/ปิดตามจังหวะ → ตามนุษย์เห็นเป็น "ภาพเต็ม"

ที่ 2000 RPM:
- 33 รอบ/วินาที = 33 FPS
- 240 พิกเซลต่อรอบ
- 125 µs ต่อพิกเซล
- **133 Hz effective refresh** (4 rows × 33 rotations) → ไม่กระพริบ

### 3.2 Wireless Power Transfer

ใช้ **Royer Converter** ส่งไฟไร้สายจากฐานคงที่ → บอร์ดที่หมุน
- หลีกเลี่ยง slip ring (สึกหรอ, มี noise)
- ใช้ PCB coil 2 ตัว coupled กันผ่านอากาศ

---

## ⚡ 4. Royer Converter (Wireless Power Source)

### 4.1 หลักการ Resonant Royer

- **Self-oscillating LC tank** ที่ความถี่ resonance
- ใช้ Zero Voltage Switching (ZVS) → ประสิทธิภาพ ~70%
- สร้าง sinewave 120 kHz → secondary coil รับสนามแม่เหล็ก

### 4.2 สเปก Royer (ที่ใช้จริง)

| Parameter | ค่า |
|---|---|
| ความถี่ทำงาน | 120 kHz |
| แรงดัน input | 12V DC adapter |
| กำลังที่ต้องส่ง | ~4-5W |
| **Driver transistor** | **BC337 BJT × 2** (Q1, Q2) |
| **Resonant cap (C2)** | **33nF Polypropylene (MKP, 630V)** — TDK B32672L6333K000 |
| **Supply choke (L5)** | **330µH** — Abracon AIUR-06-331K (Isat 1.62A) |
| Base bias (R1) | 2.2kΩ — Yageo CFR-25JB-52-2K2 |
| Primary coil (L1) | PCB spiral ~53µH (จาก formula 1/(2π√LC) ที่ 120 kHz กับ C=33nF) |

**หมายเหตุ — ทำไม BC337 แทน IRFZ44N?**
- ราคาถูกกว่า ~10×
- ไม่ต้องการ gate drive circuit (Vbe vs Vgs threshold)
- เพียงพอสำหรับ 4-5W Royer output (Ic_max BC337 = 800mA > peak current 660mA)
- ง่ายต่อการ debug

---

## 🔌 5. การกระจายไฟ (Power Distribution)

```
12V DC Adapter
     │
     ▼
J1 DC Jack (Gravitech CON-SOCJ-2155)
     │
     ├─→ U1 LM317 → motor (variable, RV1 1k pot)
     │
     └─→ Royer Driver (BC337 ×2 + LC tank 33nF/53µH)
          │
          ▼
Royer_coil PCB: Primary L1 + Coupling L2
          ║ wireless transfer (air gap 3-5mm)
          ║
          ▼
LED_DisplayV2: Secondary L1 (รับสนามแม่เหล็ก)
          │
          ▼
Bridge Rectifier (4× 1N5818 STM Schottky DO-41 THT)
          │
          ▼
C1, C2 = 220µF/25V (Panasonic EEU-FR1E221B, Low-ESR)
          │
          ▼  V_raw ~7V DC
          │
          ├──→ A2 Arduino Nano VIN (internal regulator → 5V for MCU)
          │
          └──→ U10 AMS1117-5.0 → +5V plane (In2.Cu)
                   │
                   ▼  +5V
                   │  C11 22µF tantalum + C12 100nF + C13 1µF
                   │
                   ├──→ LED 64 ตัว (ผ่าน RN1-RN8 + TPIC6C595)
                   ├──→ TPIC6C595 × 8 ตัว (logic + decoupling)
                   └──→ A3144 Hall sensor + R1 20k pullup
```

---

## 💡 6. การออกแบบ LED Layout

### 6.1 ตำแหน่งของ 4 แถว

**เลือกใช้:** 45° / 135° / 225° / 315° (KiCad-friendly)

```
        แถว B (135°)
        ●●●●●●●●●●●●●●●●
                │
แถว C ──────────●────── แถว A (45°)
(225°)          │
        ●●●●●●●●●●●●●●●●
        แถว D (315°)
```

**เหตุผล:**
- ✅ KiCad routing สนับสนุน 45° native
- ✅ Effective scan rate = 4× ต่อรอบ
- ✅ Mirror symmetric → balance สมบูรณ์
- ✅ พื้นที่ว่างที่ 0/90/180/270° สำหรับวางอุปกรณ์

### 6.2 LED ที่ใช้

| รายการ | ค่า |
|---|---|
| **Part Number** | **XL-257UWC-L** (Xinglight) |
| Color | **Ultra White Cool** (8000-10000K) |
| Package | Rectangular THT 3.0×2.0×4.0mm |
| Lead pitch | 2.54mm |
| Vf | ~3.3V |
| If | 20mA continuous, 30mA peak |
| **Quantity** | **64 ตัว** (4 rows × 16) |

### 6.3 LED Current Limiting

ที่ +5V supply กับ Vf=3.3V (White LED):
- $R = \frac{5 - 3.3}{0.017A} = 100Ω$ ที่ I = 17mA
- **เลือก Bourns 4609X-101-101LF** (8 resistors × 100Ω bussed, SIP9)

⚠️ **หมายเหตุ:** ค่า 470Ω ใน RD40 reference ใช้สำหรับ Red LED (Vf=2V) → ของเราใช้ White LED ต้องลด R ลงเป็น 100Ω เพื่อให้ I=17mA สว่างพอสำหรับ POV

### 6.4 หลักการเลื่อนรัศมี

ในแต่ละคู่ของแถว:
- **เลื่อนรัศมี 1mm** → เติม "ช่องว่างเชิงรัศมี" ระหว่าง LED
- ทำให้ภาพต่อเนื่อง ไม่มีจุดมืด

---

## 🖥️ 7. ระบบ Control

### 7.1 Arduino Nano (ATmega328P, 5V logic)

⚠️ **สำคัญ:** ใช้ **Arduino Nano v3.x** (ATmega328P 5V) — **ไม่ใช่** Nano ESP32 (3.3V)

**Pin Assignment:**
| Pin | หน้าที่ |
|---|---|
| D2 (INT0) | Hall sensor (A3144) input |
| D10 | RCK (latch) ของ shift register |
| D11 (MOSI) | SER_IN ของ shift register |
| D13 (SCK) | SRCK ของ shift register |
| D9 (PWM) | G_N (brightness control) |
| VIN | V_raw 7V จาก rectifier |

### 7.2 Shift Register Cascade

- **TPIC6C595 × 8 ตัว** (Texas Instruments DIP-16) = 64 outputs
- ต่อ cascade ผ่าน SER_OUT → SER_IN
- Open-drain DMOS sink 100mA/channel → ขับ LED ได้ตรงๆ
- SPI 16 MHz: ส่ง 64 bit ใน 4 µs

### 7.3 Image Storage (PROGMEM)

- **ขนาดต่อภาพ:** 960 bytes (240 มุม × 32 bit, เก็บแค่ 1 คู่)
- **Flash 32KB ของ Arduino:** เก็บได้ ~25 ภาพ
- ภาพ static, compile ใหม่เมื่อต้องการเปลี่ยน

---

## 🛒 8. BOM (Bill of Materials) — Final Selection

### 8.1 Power_and_MotorV2 Board

| Designator | Value | Part Number | Manufacturer |
|---|---|---|---|
| C1, C6 | 100µF/25V | EEU-FM1E101 | Panasonic |
| **C2** | **33nF/630V MKP** | **B32672L6333K000** | **TDK** |
| C3, C4, C5 | 100nF film | 168104J50A-F | Cornell Dubilier |
| D1 | SS14 SMA | SS14 | onsemi |
| D2 | 1N4004 (or 1N4001) | 1N4004-T | Lumimax |
| L5 | 330µH | AIUR-06-331K | Abracon |
| Q1, Q2 | BC337 | BC33725TFR | onsemi |
| R1 | 2.2kΩ 5% ¼W | CFR-25JB-52-2K2 | Yageo |
| **R2** | **270Ω 1% ¼W** | **MFR-25FBF52-270R** | Yageo |
| R3 | 100Ω 5% ¼W | CFR-25JB-52-100R | Yageo |
| RV1 | 1kΩ pot | P160KN-0QD15B1K | TT Electronics/BI |
| U1 | LM317 | LM317AT/NOPB | Texas Instruments |
| J1 | DC Jack | CON-SOCJ-2155 | Gravitech (Thailand) |
| SW1 | DPDT toggle | JS202011AQN | C&K *(pending)* |
| M1 | DC Motor | (Already have) | - |

### 8.2 Royer_coil Board

| Designator | Value | Note |
|---|---|---|
| L1, L2 | Coil spiral | PCB traces (1.5mm width, 1064 segments) |
| Terminals | 2× THT pads | @ (120.5, 137.5) and (179.5, 137.5) |

### 8.3 LED_DisplayV2 Board (Rotor)

| Designator | Value | Part Number | Manufacturer |
|---|---|---|---|
| **C1, C2** | **220µF/25V** Low-ESR | **EEU-FR1E221B** | **Panasonic FR** |
| C3-C10 | 100nF 0603 X7R | CL10E104KC8VPNC | Samsung |
| C11 | 22µF/10V tantalum 3528 | *(TBD)* | - |
| C12 | 100nF 0603 | *(TBD)* | - |
| C13 | 1µF 0805 | *(TBD)* | - |
| **D1-D4** | **1N5818 Schottky** | **1N5818** | **STMicroelectronics** |
| **D5-D68** | **LED White ×64** | **XL-257UWC-L** | **Xinglight** |
| R1 | 20kΩ 1% (Hall pullup) | MFR-25FRF52-20K | Yageo |
| **RN1-RN8** | **100Ω SIP9 bussed** | **4609X-101-101LF** | **Bourns** |
| U1 | A3144 Hall sensor | *(TBD — genuine Allegro)* | - |
| U3-U10 | TPIC6C595N ×8 | TPIC6C595N | Texas Instruments |
| **U10** | **AMS1117-5.0** | **AMS1117-5.0** | **EVVO** (Digikey) |
| A1 | Arduino Nano v3.x | *(TBD — ATmega328P, NOT ESP32!)* | - |

---

## 🎨 9. PCB Design Decisions

### 9.1 Board Specifications

| Board | ขนาด | Layers | Status |
|---|---|---|---|
| **LED_DisplayV2** | 120mm dia circle | **4-layer** (F/GND/+5V/B) | ✅ Routed, pending mounting holes |
| **Power_and_MotorV2** | 110×110mm rect | 2-layer | ✅ Gerber exported |
| **Royer_coil** | 98mm OD ring (58mm inner) | 2-layer | ✅ Gerber exported |

### 9.2 LED_DisplayV2 4-Layer Stack

```
F.Cu      35µm (1oz)  ← Signal + components
prepreg   0.1mm FR4
In1.Cu    35µm        ← GND plane (full pour)
core      1.24mm FR4
In2.Cu    35µm        ← +5V plane (full pour)
prepreg   0.1mm FR4
B.Cu      35µm (1oz)  ← Signal + GND fill
```

**ประโยชน์ของ 4-layer:**
- Internal GND + +5V planes → noise immunity ดี (สำคัญ Royer EMI 120kHz)
- Signal traces ที่ outer layers reference internal planes → controlled impedance
- ลด IR drop ของ +5V (current loop กว้าง)
- Heat distribution ผ่าน internal copper plane

### 9.3 Critical Design Rules

| Net Class | Trace Width |
|---|---|
| Power (V_raw, GND) | 1-2mm |
| +5V routing | 1mm |
| Signal (SPI, GPIO, LED cathode) | 0.25mm |
| Coil traces (Royer_coil) | 1.5mm |

### 9.4 Ground Pour & Stitching

- **GND pour ทั้ง 4 layers** ของ LED_DisplayV2 (รวม F.Cu/B.Cu outer)
- **Stitching vias** ทุก 10-15mm
- พิเศษ: thermal vias 4-6 ตัวใกล้ U10 AMS1117 (heat dissipation)
- **Keepout zones** บน In1.Cu + In2.Cu ใต้ secondary coil L1 (กัน eddy current)

---

## 🔬 10. การคำนวณสำคัญ

### 10.1 Power Budget

| Load | Average Current | Power @ 5V |
|---|---|---|
| LED 64 ตัว (duty 30%, 17mA each) | 326 mA | 1.6W |
| TPIC6C595 × 8 | 25 mA | 0.13W |
| Hall sensor A3144 | 10 mA | 0.05W |
| Arduino Nano | 40 mA | 0.32W (Vin=7V) |
| **รวมที่ฝั่งหมุน** | **~400 mA** | **~2.1W** |
| **+ Coupling loss 30%** | - | **~2.7W** |
| **Royer ต้องส่ง** | - | **~4W** |

### 10.2 Resonance Frequency

**สมการ:** $f = \frac{1}{2\pi\sqrt{LC}}$

ค่าออกแบบ:
- C2 = **33nF** (TDK B32672L6333K000)
- ต้องการ f = **120 kHz**
- → L1 = $\frac{1}{(2\pi \times 120000)^2 \times 33 \times 10^{-9}}$ = **~53 µH**

### 10.3 Timing ที่ 2000 RPM

- 30 ms/รอบ
- 240 พิกเซล/รอบ → 125 µs/พิกเซล
- SPI 64 bit: ส่งใน 4 µs (3.2% ของ timing budget)
- เหลือเวลา 121 µs สำหรับการคำนวณอื่น ✓

### 10.4 Refresh Rate

- 4× scan ต่อรอบ × 33 rotations/sec = **133 Hz effective refresh**
- ไม่มีการกระพริบที่ตาเห็น

### 10.5 AMS1117 Thermal

- V_drop = 7V - 5V = 2V
- I_load = 400mA
- P_diss = 2V × 0.4A = **0.8W**
- SOT-223 θJA ≈ 50°C/W (no copper pour) → 40°C rise
- ที่ ambient 25°C → junction 65°C ✓ (max 125°C)
- มี In1.Cu GND plane เป็น heat sink → ดียิ่งขึ้น

---

## ⚠️ 11. ความท้าทาย & การแก้ปัญหา

### 11.1 EMI/Noise จาก Royer

**ปัญหา:** Royer 120 kHz สร้างสนามแม่เหล็กแรง

**การแก้:**
- 4-layer stack ของ LED_DisplayV2 → internal GND + +5V planes แยก signal
- Ground pour ทั้ง F.Cu + B.Cu
- Decoupling cap (100nF) ใกล้ทุก IC (8 ตัว สำหรับ TPIC6C595)
- A3144 อยู่ห่างจาก secondary coil อย่างน้อย 10mm

### 11.2 Mass Balance

**ปัญหา:** บอร์ดต้องสมดุลที่ความเร็ว 2000 RPM

**การแก้:**
- LED 4 แถว 4 ทิศ → balance อัตโนมัติ
- Component placement สมมาตรกับแกน 0°-180° (ยังต้องปรับ — heavy caps C1, C2 ปัจจุบันอยู่ครึ่งล่าง)
- รูยึด M2 รอบขอบ (4-8 รู) — **ยังต้องเพิ่ม** เป็น next step

### 11.3 Heat Dissipation

**ปัญหา:** AMS1117 drop 2V × ~400mA = 0.8W

**การแก้:**
- 4-layer stack → internal +5V plane = heat spreader
- Tab (Pin 2) ต่อ +5V plane ผ่าน thermal vias
- Operating temp คาดการณ์: ~65°C (ต่ำกว่าขีดจำกัด 125°C มาก)

### 11.4 Eddy Current ใน Inner Planes

**ปัญหา:** Royer 120kHz coupling ผ่าน secondary coil — ถ้า +5V plane (In2.Cu) อยู่ใต้ coil → eddy current loss

**การแก้:**
- **Keepout zone** บน In1.Cu + In2.Cu ใต้ secondary coil
- Ground/Power pour หลีกเลี่ยงพื้นที่ใต้ spiral

---

## 📅 12. Progress & Status

### ✅ สิ่งที่เสร็จแล้ว

#### Schematic (100%)
- [x] Power_and_MotorV2 — Motor controller + Royer driver
- [x] Royer_coil — Primary + Coupling coils
- [x] LED_DisplayV2 — Bridge rectifier + AMS1117 + Arduino + LED matrix
- [x] LED layout 4 แถว at 45/135/225/315° (16 LEDs/row)
- [x] Component selection — เลือก part จริงครบทุกตัว (BOM_Selection_Rationale.md)

#### PCB Layout
- [x] **Royer_coil PCB** — Spiral traces complete, gerber exported ✓
- [x] **Power_and_MotorV2** — Fully routed, gerber exported ✓
- [x] **LED_DisplayV2** — 4-layer routed + ground/power pours + 58 vias ✓
- [x] Net classes (Default 0.2mm, 1mm power, 2mm heavy)
- [x] Edge cuts: 120mm dia, 15mm spindle hole

#### BOM
- [x] Power_and_MotorV2 — ครบ (ยกเว้น SW1)
- [x] LED_DisplayV2 — ครบ (ยกเว้น Arduino, A3144, C11/C12/C13)

### 🚧 กำลังทำ / ค้าง

- [ ] **LED_DisplayV2 mounting holes** — ต้องเพิ่ม M2 holes 4-8 รูสำหรับ balance weight
- [ ] **L1 Secondary coil ของ LED_DisplayV2** — copy/paste pattern จาก Royer_coil มาวาง ที่ rotor side
- [ ] **เลือก:** A1 Arduino Nano, SW1 DPDT, U1 A3144, C11/C12/C13 caps
- [ ] DRC check ครั้งสุดท้าย

### 📋 ขั้นต่อไป

- [ ] สั่ง PCB (3 บอร์ด) จาก JLCPCB / PCBWay
- [ ] สั่ง components จาก Digikey + Gravitech + LCSC
- [ ] ประกอบและบัดกรี
- [ ] Test bench:
  - [ ] ทดสอบ Royer (Power_and_MotorV2 + Royer_coil) เฉพาะ — วัด 120 kHz oscillation
  - [ ] ทดสอบ Wireless transfer (เพิ่ม LED_DisplayV2 secondary coil) — วัด V_DC
  - [ ] ทดสอบ Full system ไม่หมุน — ดู LED ติด
  - [ ] ทดสอบ Full system หมุน 200 RPM → 2000 RPM
- [ ] Arduino code + Python script สร้าง PROGMEM image
- [ ] Mechanical mounting (CD motor + base + spacer)
- [ ] Calibration และ tuning Royer frequency

---

## 📚 13. Technical References

1. **Hackaday RD40 Rotating LED Display** - lhm0
   - URL: hackaday.io/project/191947-rotating-led-display
2. **Royer Converter Theory** - Mikrocontroller.net
   - URL: www.mikrocontroller.net/articles/Royer_Converter
3. **AMS1117 Datasheet** - Advanced Monolithic Systems
4. **TPIC6C595 Datasheet** - Texas Instruments
5. **LM317 Datasheet** - Texas Instruments
6. **BC337 Datasheet** - onsemi
7. **AIUR-06-331K Datasheet** - Abracon
8. **IPC-2221** - PCB Trace Current Capacity Standard
9. **Mohan's Formula** - Planar Spiral Inductance (IEEE JSSC 1999)

**Project-Specific Documents:**
- [BOM_Selection_Rationale.md](BOM_Selection_Rationale.md) — เหตุผลการเลือก components แต่ละชิ้น

---

## 🎓 14. ภาคผนวก: ตารางการตัดสินใจสำคัญ

| การตัดสินใจ | ทางเลือกอื่น | ที่เลือก | เหตุผล |
|---|---|---|---|
| MCU | ESP32, RP2040 | **Arduino Nano ATmega328P** | ง่าย, รู้จัก, 5V logic ตรง TPIC6C595 |
| Power regulator | MP1584, LM7805 | **AMS1117-5.0** | noise ต่ำ, ขนาดเล็ก, ใช้ง่าย |
| **Royer driver** | IRFZ44N MOSFET | **BC337 BJT** | ราคาถูก, ง่าย, พอสำหรับ 4-5W |
| **Resonance cap** | 100nF | **33nF MKP** | ตรงกับ L1 จริง, target 120kHz |
| Hall sensor | AH3503 (3.3V) | **A3144 (5V)** | ตรงกับ logic level Arduino |
| Image storage | SD card, EEPROM | **PROGMEM** | ง่าย, ไม่เพิ่ม component |
| LED layout | 0/90/180/270, 30/150/210/330 | **45/135/225/315** | KiCad-friendly, 4× scan |
| **LED color** | Red, Blue, RGB | **White XL-257UWC-L** | flexible, แสดงทุกภาพได้ |
| **LED current R** | 470Ω (per RD40) | **100Ω 4609X-101-101LF** | White LED Vf=3.3V → ต้องลด R |
| **Bridge diode** | SS14 SMA, 1N4007 | **1N5818 STM DO-41 THT** | Schottky low Vf, THT ง่ายบัดกรี |
| Input cap C1 | 100µF, 470µF | **220µF/25V Panasonic FR** | balance bulk + voltage margin |
| Output cap C3 | Ceramic, Electrolytic | **Tantalum 22µF** | stable กับ AMS1117 |
| **PCB strategy** | บอร์ดเดียว | **3 บอร์ดแยก** | modular, ลด complexity |
| **LED board layers** | 2 layers | **4 layers** | GND/+5V planes → noise + thermal |

---

## 💬 15. คำถามที่อาจารย์อาจถาม + คำตอบ

**Q1: ทำไมไม่ใช้ ESP32 ที่ทันสมัยกว่า?**
A: Arduino Nano ATmega328P เพียงพอสำหรับ timing budget (ใช้แค่ 3.2% ของเวลาต่อพิกเซล) และไม่ต้องการ WiFi/Bluetooth สำหรับ image storage แบบ static. ESP32 Nano ใช้ 3.3V logic ซึ่งจะ marginal กับ TPIC6C595 V_IH=2V threshold ในสภาพ EMI noise

**Q2: ทำไมเลือก Royer Converter แทน buck/boost ปกติ?**
A: Royer เป็น self-oscillating, ไม่ต้องการ MCU/PWM controller, มี ZVS efficiency สูง และเหมาะกับการขับ resonant coil สำหรับ wireless power

**Q3: ทำไม 4 แถวไม่ใช่ 2 แถวเหมือนต้นฉบับ?**
A: เพิ่ม effective scan rate เป็น 4× ทำให้ภาพสว่างขึ้นและกระพริบน้อยลง โดยเฉพาะที่ RPM ต่ำ

**Q4: ทำไม PCB coil ไม่ใช้ wire-wound?**
A: PCB coil ได้ tolerance ดีกว่า, ทำซ้ำได้, ไม่ต้องประกอบ, ราคาเป็นส่วนหนึ่งของ PCB

**Q5: balance ที่ 2000 RPM จะมีปัญหาไหม?**
A: ออกแบบให้สมมาตรกับแกน + มีรูสำหรับน็อตปรับ + LED 4 แถวกระจายช่วย balance อัตโนมัติ — กำลังเพิ่ม mounting holes รอบขอบสำหรับ balance weight

**Q6: ทำไม Royer ใช้ BC337 BJT ไม่ใช้ MOSFET?**
A: BC337 ราคาถูกกว่า 10 เท่า, ไม่ต้องการ gate drive circuit complex (Vbe = 0.7V vs Vgs = 4-10V), Ic_max = 800mA เพียงพอสำหรับ peak current 660mA ของ Royer 4-5W. MOSFET ดีกว่าสำหรับ power สูง >20W

**Q7: ทำไม resonance cap = 33nF ไม่ใช่ 100nF ตามต้นฉบับ?**
A: เป็นผลจาก design iteration. ค่า L ของ primary coil ที่เราออกแบบจริงคือ ~53µH (ไม่ใช่ 17µH ของต้นฉบับ). ตามสมการ f = 1/(2π√LC) ที่ 120 kHz → ต้องใช้ C = 33nF เพื่อ match กับ L ของเรา

**Q8: ทำไม LED ใช้ White ไม่ใช้สีอื่น?**
A: White LED ครอบคลุมทุก use case (สามารถแสดงสีใดๆ ผ่าน firmware ก็ได้ในแง่ visual perception). Red อาจสว่างกว่าและ Vf ต่ำกว่า แต่จำกัด aesthetic options

**Q9: ทำไมใช้ 4-layer PCB เฉพาะ LED board?**
A: LED board มี:
- Royer EMI noise ที่ต้องป้องกัน (4-layer มี GND plane shielding)
- กระแสรวม 400mA+ ที่ +5V — internal plane ช่วย IR drop
- Thermal management สำหรับ AMS1117 + TPIC6C595 × 8
- Power และ Royer board ไม่มี complexity เท่า → 2-layer พอ

**Q10: 3 บอร์ดแทน 1 บอร์ดจะวุ่นวายไหม?**
A: Benefits ของการแยก:
- Royer_coil สามารถปรับ-ทดสอบ-ปรับ ซ้ำได้โดยไม่ต้องทำใหม่ทั้งบอร์ด
- Power_and_Motor + Royer_coil เป็นชุด base, แยกจาก rotor ที่ต้อง balance
- รูปร่างต่างกัน (square vs ring vs circle) → ใช้ board space ดีกว่า
- Cost: 3 บอร์ดเล็กถูกกว่า 1 บอร์ดใหญ่ที่ JLCPCB

---

*Last updated: 2026-05-18*
*Component selection finalized — pending order + assembly + test*
