# POV_Display_PCB

# 📋 สรุปโปรเจกต์ POV Display 4 แถว LED

เอกสารสำหรับนำเสนออาจารย์ — สรุปการตัดสินใจดีไซน์ทั้งหมด

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
| MCU | Arduino Nano + ESP-01s | **Arduino Nano เดียว** |
| Image storage | ESP-01s WiFi | **PROGMEM (32KB Flash)** |
| ขนาด rotor | 120mm (CD size) | 120mm (เท่าเดิม) |

---

## 🔧 2. หลักการทำงาน

### 2.1 Persistence of Vision (POV)

LED จำนวนน้อยหมุนเร็วๆ + เปิด/ปิดตามจังหวะ → ตามนุษย์เห็นเป็น "ภาพเต็ม"

ที่ 2000 RPM:
- 33 รอบ/วินาที = 33 FPS
- 240 พิกเซลต่อรอบ
- 125 µs ต่อพิกเซล

### 2.2 Wireless Power Transfer

ใช้ **Royer Converter** ส่งไฟไร้สายจากฐานคงที่ → บอร์ดที่หมุน
- หลีกเลี่ยง slip ring (สึกหรอ, มี noise)
- ใช้ PCB coil 2 ตัว coupled กันผ่านอากาศ

---

## ⚡ 3. Royer Converter (Wireless Power Source)

### 3.1 หลักการ Resonant Royer

- **Self-oscillating LC tank** ที่ความถี่ resonance
- ใช้ Zero Voltage Switching (ZVS) → ประสิทธิภาพ 90%+
- สร้าง sinewave 120 kHz → secondary coil รับสนามแม่เหล็ก

### 3.2 สเปก Royer

| Parameter | ค่า |
|---|---|
| ความถี่ทำงาน | 120 kHz |
| แรงดัน input | 12V DC adapter |
| กำลังที่ต้องส่ง | ~4-5W |
| MOSFET | IRFZ44N × 2 |
| Resonant cap | 100nF polypropylene (film) |
| Input choke | 47-100µH |

---

## 🔌 4. การกระจายไฟ (Power Distribution)

```
12V DC Adapter
     │
     ▼
Royer Converter (120 kHz)
     │
     ▼
Primary PCB Coil (ฐาน)
     ║
     ║ wireless transfer (air gap 3-5mm)
     ║
     ▼
Secondary PCB Coil (rotor)
     │
     ▼
Bridge Rectifier (4× SS14 Schottky)
     │
     ▼
C1 = 220µF + C2 = 1µF ceramic (smoothing)
     │
     ▼ V_raw ~7V
     │
     ├──→ Arduino Nano VIN (regulator internal → 5V)
     │
     └──→ AMS1117-5.0 → +5V rail
              │
              ▼
         C3 = 22µF tantalum + C4 = 100nF ceramic
              │
              ├──→ LED 64 ตัว
              ├──→ TPIC6C595 × 8 ตัว
              └──→ A3144 Hall sensor + pull-up 20k
```

---

## 💡 5. การออกแบบ LED Layout

### 5.1 ตำแหน่งของ 4 แถว

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

### 5.2 หลักการเลื่อนรัศมี

ในแต่ละคู่ของแถว:
- **เลื่อนรัศมี 1mm** → เติม "ช่องว่างเชิงรัศมี" ระหว่าง LED
- ทำให้ภาพต่อเนื่อง ไม่มีจุดมืด

---

## 🖥️ 6. ระบบ Control

### 6.1 Arduino Nano (ATmega328P, 5V logic)

**Pin Assignment:**
| Pin | หน้าที่ |
|---|---|
| D2 (INT0) | Hall sensor input |
| D10 | RCK (latch) ของ shift register |
| D11 (MOSI) | SER_IN ของ shift register |
| D13 (SCK) | SRCK ของ shift register |
| D9 (PWM) | G_N (brightness control) |
| VIN | V_raw 7V จาก rectifier |

### 6.2 Shift Register Cascade

- **TPIC6C595 × 8 ตัว** = 64 outputs
- ต่อ cascade ผ่าน SER_OUT → SER_IN
- SPI 16 MHz: ส่ง 64 bit ใน 4 µs

### 6.3 Image Storage (PROGMEM)

- **ขนาดต่อภาพ:** 960 bytes (240 มุม × 32 bit, เก็บแค่ 1 คู่)
- **Flash 32KB ของ Arduino:** เก็บได้ ~25 ภาพ
- ภาพ static, compile ใหม่เมื่อต้องการเปลี่ยน

---

## 🛒 7. BOM (Bill of Materials)

### 7.1 Power Section

| Designator | Component | Spec | Qty |
|---|---|---|---|
| D1-D4 | SS14 Schottky | 1A 40V SMA | 4 |
| C1 | Electrolytic | 220µF/16V | 1 |
| C2 | Ceramic | 1µF/25V X7R 0805 | 1 |
| U10 | AMS1117-5.0 | SOT-223 | 1 |
| C11 | Tantalum | 22µF/10V 1206 | 1 |
| C12 | Ceramic | 100nF/25V X7R 0603 | 1 |

### 7.2 Logic Section

| Designator | Component | Spec | Qty |
|---|---|---|---|
| A1 | Arduino Nano | ATmega328P, 5V | 1 |
| U2-U9 | TPIC6C595 | DIP-16 | 8 |
| U1 | A3144 | Hall sensor 5V | 1 |
| R1 | Resistor | 20k pull-up | 1 |
| R2-R9 | Resistor | 10k pull-up for CLR_N | 1 (shared) |
| Cdec | Ceramic | 100nF × 10 (decoupling) | 10 |

### 7.3 LED Section

| Designator | Component | Spec | Qty |
|---|---|---|---|
| LED1-LED64 | LED 0805 | สีตามต้องการ | 64 |
| RN1-RN8 | Resistor Network | 470Ω × 8 | 8 |

### 7.4 Wireless Power

| Component | Spec |
|---|---|
| L1 Primary | PCB bifilar coil, 17 turns, 90/30mm |
| L2 Coupling | PCB coil, 6 turns, 28/10mm |
| L3 Secondary | PCB coil, 14 turns, 90/30mm |
| Royer MOSFET | IRFZ44N × 2 |
| Resonant cap | 100nF film polypropylene |

---

## 🎨 8. PCB Design Decisions

### 8.1 Board Specifications

| Parameter | ค่า |
|---|---|
| ขนาด | 120mm diameter (CD size) |
| Thickness | 1.6mm FR4 |
| Layers | 2 layers, 1oz copper |
| Mounting | CD motor spindle ตรงกลาง |

### 8.2 Critical Design Rules

| Net Class | Trace Width |
|---|---|
| Power (+5V, V_raw, GND) | 0.8-1.0mm |
| Signal (SPI, GPIO) | 0.2mm |
| LED traces | 0.3-0.4mm |

### 8.3 Ground Pour & Stitching

- **Top + Bottom ground pour** บนพื้นที่ว่างทั้งหมด
- **Stitching vias** ทุก 10-15mm เชื่อม top/bottom GND
- พิเศษ: vias 4-6 ตัวใกล้ AMS1117 (heat dissipation)

---

## 🔬 9. การคำนวณสำคัญ

### 9.1 Power Budget

| Load | Average Current | Power @ 5V |
|---|---|---|
| LED 64 ตัว (duty 30%) | 300 mA | 1.5W |
| TPIC6C595 × 8 | 25 mA | 0.13W |
| Hall sensor | 10 mA | 0.05W |
| Arduino Nano | 40 mA | 0.32W (Vin=8V) |
| **รวมที่ฝั่งหมุน** | **~375 mA** | **~2.0W** |
| **+ Coupling loss 30%** | - | **~2.6W** |
| **Royer ต้องส่ง** | - | **~4W** |

### 9.2 Timing ที่ 2000 RPM

- 30 ms/รอบ
- 240 พิกเซล/รอบ → 125 µs/พิกเซล
- SPI 64 bit: ส่งใน 4 µs (3.2% ของ timing budget)
- เหลือเวลา 121 µs สำหรับการคำนวณอื่น ✓

### 9.3 Refresh Rate

- 4× scan ต่อรอบ × 33 rotations/sec = **133 Hz effective refresh**
- ไม่มีการกระพริบที่ตาเห็น

---

## ⚠️ 10. ความท้าทาย & การแก้ปัญหา

### 10.1 EMI/Noise จาก Royer

**ปัญหา:** Royer 120 kHz สร้างสนามแม่เหล็กแรง

**การแก้:**
- Ground plane ทั้ง 2 ชั้น
- Decoupling cap (100nF) ใกล้ทุก IC
- Power-on reset RC สำหรับ shift register
- A3144 อยู่ห่างจาก secondary coil อย่างน้อย 10mm

### 10.2 Mass Balance

**ปัญหา:** บอร์ดต้องสมดุลที่ความเร็ว 2000 RPM

**การแก้:**
- Component placement สมมาตรกับแกน 0°-180°
- รูยึด M2 รอบขอบ 4-8 รู สำหรับ balance weight
- LED 4 แถว 4 ทิศ → balance อัตโนมัติ

### 10.3 Heat Dissipation

**ปัญหา:** AMS1117 drop 2V × ~400mA = 0.8-1.5W

**การแก้:**
- Copper pour 2×2cm ใต้ AMS1117
- Stitching vias เชื่อมไป bottom GND plane
- Operating temp คาดการณ์: ~80°C (ใต้ขีดจำกัด 125°C)

---

## 📅 11. Progress & Next Steps

### ✅ สิ่งที่เสร็จแล้ว
- [x] Schematic หลัก (Arduino + LED + Power + Hall sensor)
- [x] Bridge rectifier + AMS1117 power supply
- [x] LED layout 4 แถว at 45/135/225/315°
- [x] PCB layout เบื้องต้น
- [x] BOM พร้อม part numbers
- [x] Capacitor selection (X7R, tantalum spec)

### 🚧 กำลังทำ
- [ ] Ground pour (top + bottom)
- [ ] Stitching vias
- [ ] DRC check + final routing

### 📋 ขั้นต่อไป
- [ ] PCB Royer driver (ฐานคงที่)
- [ ] PCB coil design (3 coils: primary bifilar, coupling, secondary)
- [ ] Arduino code + Python script สร้าง PROGMEM image
- [ ] Mechanical mounting (CD motor + base)
- [ ] Testing & calibration

---

## 📚 12. Technical References

1. **Hackaday RD40 Rotating LED Display** - lhm0
   - URL: hackaday.io/project/191947-rotating-led-display
2. **Royer Converter Theory** - Mikrocontroller.net
   - URL: www.mikrocontroller.net/articles/Royer_Converter
3. **AMS1117 Datasheet** - Advanced Monolithic Systems
4. **TPIC6C595 Datasheet** - Texas Instruments
5. **IPC-2221** - PCB Trace Current Capacity Standard

---

## 🎓 ภาคผนวก: ตารางการตัดสินใจสำคัญ

| การตัดสินใจ | ทางเลือกอื่น | ที่เลือก | เหตุผล |
|---|---|---|---|
| MCU | ESP32, RP2040 | **Arduino Nano** | ง่าย, รู้จัก, เพียงพอ |
| Power regulator | MP1584, LM7805 | **AMS1117** | noise ต่ำ, ขนาดเล็ก, ใช้ง่าย |
| Hall sensor | AH3503 (3.3V) | **A3144 (5V)** | ตรงกับ logic level Arduino |
| Image storage | SD card, EEPROM | **PROGMEM** | ง่าย, ไม่เพิ่ม component |
| LED layout | 0/90/180/270, 30/150/210/330 | **45/135/225/315** | KiCad-friendly, 4× scan |
| Bridge diode | 1N4007, DB107 | **SS14 Schottky** | Vf ต่ำ, เร็วพอ 120 kHz |
| Input cap C1 | 100µF, 470µF | **220µF + 1µF** | balance ระหว่าง bulk + HF filter |
| Output cap C3 | Ceramic, Electrolytic | **Tantalum 22µF** | stable กับ AMS1117 |

---

## 💬 คำถามที่อาจารย์อาจถาม + คำตอบ

**Q1: ทำไมไม่ใช้ ESP32 ที่ทันสมัยกว่า?**
A: Arduino Nano เพียงพอสำหรับ timing budget (ใช้แค่ 3% ของเวลาต่อพิกเซล) และไม่ต้องการ WiFi/Bluetooth สำหรับ image storage แบบ static

**Q2: ทำไมเลือก Royer Converter แทน buck/boost ปกติ?**
A: Royer เป็น self-oscillating, ไม่ต้องการ MCU/PWM controller, มี ZVS efficiency สูง และเหมาะกับการขับ resonant coil สำหรับ wireless power

**Q3: ทำไม 4 แถวไม่ใช่ 2 แถวเหมือนต้นฉบับ?**
A: เพิ่ม effective scan rate เป็น 4× ทำให้ภาพสว่างขึ้นและกระพริบน้อยลง โดยเฉพาะที่ RPM ต่ำ

**Q4: ทำไม PCB coil ไม่ใช้ wire-wound?**
A: PCB coil ได้ tolerance ดีกว่า, ทำซ้ำได้, ไม่ต้องประกอบ, ราคาเป็นส่วนหนึ่งของ PCB

**Q5: balance ที่ 2000 RPM จะมีปัญหาไหม?**
A: ออกแบบให้สมมาตรกับแกน + มีรูสำหรับน็อตปรับ + LED 4 แถวกระจายช่วย balance อัตโนมัติ

---

อันนี้คือสรุปทั้งหมดที่คุยกันมา ครอบคลุมทุกการตัดสินใจ พร้อมเหตุผล ตัวเลข และเอกสารอ้างอิงที่อาจารย์มักถาม

ถ้าต้องการให้ผมขยายส่วนไหนเพิ่ม หรือทำเป็น slides PowerPoint บอกได้เลยครับ! 👍


ดีมากครับ Royer coil เป็นส่วนที่อาจารย์น่าจะถามรายละเอียดมาก ผมจะเพิ่มให้ละเอียดและครบทุกเรื่องที่คุยมา

---

# 📋 ภาคเพิ่มเติม: Royer Coil System

เอกสารส่วนเสริมสำหรับนำเสนออาจารย์ — โฟกัสที่ระบบ coil ของ Royer Converter

---

## 🔍 1. ทำไม Royer ต้องมี Coil 3 ตัว?

Royer Converter ในงาน wireless power **ไม่ใช่ขดลวดเดียว** แต่ใช้ระบบ **3 coil ทำงานร่วมกัน**

```
┌─── ฝั่งฐาน (อยู่นิ่ง) ──────────────────┐
│                                          │
│   ┌─────────────────┐                    │
│   │ L1: Primary     │ ← coil หลัก       │
│   │     Bifilar     │   เป็น LC tank    │
│   │     17 turns    │   + ส่งกำลัง      │
│   └─────────────────┘                    │
│        ║║                                 │
│        ║║ inductive coupling             │
│        ║║                                 │
│   ┌────╨╨─────────┐                      │
│   │ L2: Coupling  │ ← coil เล็ก         │
│   │     6 turns    │   feedback ไป      │
│   │                │   gate MOSFET     │
│   └─────────────────┘                    │
└─────────────╫───────────────────────────┘
              ║
              ║ AIR GAP (~3-5mm)
              ║ → ส่งกำลังที่นี่
              ║
┌─────────────╫───── ฝั่งหมุน ──────────────┐
│   ┌─────────────────┐                     │
│   │ L3: Secondary   │ ← coil รับ        │
│   │     14 turns    │   อยู่บน rotor    │
│   │                 │   → rectifier     │
│   └─────────────────┘                     │
└──────────────────────────────────────────┘
```

---

## ⚙️ 2. หน้าที่ของแต่ละ Coil (อธิบายละเอียด)

### 2.1 L1 — Primary Bifilar Coil

**หน้าที่ 2 อย่างพร้อมกัน:**

1. **เป็นส่วนหนึ่งของ LC Resonant Tank**
   - ร่วมกับ capacitor 100nF → สั่นที่ 120 kHz
   - กำหนดความถี่ทำงานของระบบทั้งหมด

2. **ส่งสนามแม่เหล็ก AC**
   - ผ่านอากาศไปยัง secondary coil
   - กำลังส่งสูงสุด ~5W

**ทำไมต้อง Bifilar (พันลวด 2 เส้นพร้อมกัน)?**
- ได้ **2 ครึ่งที่ coupling กันสมบูรณ์**
- ทำหน้าที่เป็น **center-tap transformer**
- ครึ่งซ้าย/ขวาทำงานสลับกัน ตอน MOSFET เปิด/ปิด

```
       ●───────●───────●
        │       │       │  ← Center tap → V+ supply
      Half 1  CT     Half 2
        │       │       │
        ●       │       ●
        │   ↓ MOSFETs ↓ │
        Q1     →     Q2
        │               │
        ●━━━━━━━━━━━━━●
              GND
```

### 2.2 L2 — Coupling Coil (Feedback)

**หน้าที่:** เป็น "ตัวฟัง" สนามแม่เหล็กของ L1 แล้วส่งสัญญาณไปเปิด-ปิด MOSFET

**ทำไมต้องมี?**
- Royer เป็น **self-oscillating** ต้องมี positive feedback
- L2 จะ induced voltage จาก L1 → ใช้ขับ gate ของ MOSFET
- ขั้วของ L2 **สำคัญที่สุด** — ต้องตรงข้ามกับ Q1/Q2 ที่กำลัง on

**ขนาดทำไมเล็กกว่า L1 มาก?**
- L2 = ~1/4 ของ L1 (ในแง่ turns)
- ได้ feedback voltage ที่เหมาะ (5-10V) สำหรับ gate
- ถ้าใหญ่เกินไป → gate voltage สูงเกิน → MOSFET พัง

### 2.3 L3 — Secondary Coil (รับพลังงาน)

**หน้าที่:** รับสนามแม่เหล็กจาก L1 ผ่านช่องอากาศ → จ่ายไฟให้บอร์ดที่หมุน

**ขั้นตอน:**
```
สนามแม่เหล็ก AC ที่ L1 ──→ L3 induced AC voltage
                              ↓
                         Bridge Rectifier
                              ↓
                         DC + Smoothing
                              ↓
                         AMS1117 → 5V → LED
```

**อัตราส่วน turns กำหนดแรงดัน output:**
- n₂/n₁ × coupling × V_primary = V_secondary
- ของเรา: 14/17 × 0.4 × 38V ≈ 12V peak → DC 10V → AMS1117 → 5V

---

## 🧮 3. การคำนวณค่าตัวเลข

### 3.1 ความถี่ Resonance

**สมการพื้นฐาน:**
$$f = \frac{1}{2\pi\sqrt{LC}}$$

**กำหนดเป้าหมาย:**
- f = 120 kHz (sweet spot ระหว่างขนาด coil กับ loss)
- C = 100nF (cap film polypropylene หาง่าย)

**คำนวณ L ที่ต้องการ:**
$$L = \frac{1}{(2\pi f)^2 \cdot C} = \frac{1}{(2\pi \times 120{,}000)^2 \times 100 \times 10^{-9}}$$

$$L \approx 17.6 \, \mu H$$

→ **L1 รวม = 18 µH** (จากปลายถึงปลาย)

### 3.2 คำนวณ Turns ของ Primary (L1)

ใช้ **Mohan's Formula** สำหรับ planar spiral coil:

$$L = \frac{\mu_0 \cdot n^2 \cdot d_{avg}}{2} \left[ \ln\left(\frac{2.46}{\rho}\right) + 0.20 \cdot \rho^2 \right]$$

**ค่าที่กำหนด:**
- d_outer = 90 mm
- d_inner = 30 mm (เผื่อพื้นที่ L2)
- d_avg = 60 mm
- ρ = 0.5 (fill ratio)

**คำนวณ:**
$$L = 6.18 \times 10^{-8} \cdot n^2 \, H$$

ต้องการ L = 18 µH:
$$n^2 = \frac{18 \times 10^{-6}}{6.18 \times 10^{-8}} = 291$$

→ **n ≈ 17 turns** (bifilar = trace 2 เส้น × 17 รอบ = 34 trace ทั้งหมด)

### 3.3 ขนาด Trace ของ L1

พื้นที่ใช้งาน: (90-30)/2 = **30 mm รัศมี**

ระยะต่อรอบ: 30/34 = **0.88 mm**

แบ่ง:
- **Trace width: 0.4 mm**
- **Gap: 0.48 mm**

**ตรวจสอบ current capacity:**
- กระแส peak ของ primary: ~750 mA
- Trace 0.4mm × 35µm (1oz) รับได้ ~1.5A ที่ ΔT 10°C ✓

### 3.4 คำนวณ Turns ของ L2 (Coupling)

**กฎทั่วไป:** L2 = L1/4 ในแง่ inductance

**ค่าที่ใช้:**
- d_outer = 28 mm (ใส่ใน inner ของ L1)
- d_inner = 10 mm
- ρ = 0.47

→ **n ≈ 6 turns** (single, ไม่ bifilar)
→ **L2 ≈ 2-3 µH**

**Trace:**
- Width: 0.3mm
- Gap: 0.5mm
- Layer: single (top)

### 3.5 คำนวณ Turns ของ L3 (Secondary)

**เป้าหมาย:** ได้ V_DC ≈ 7-9V (สำหรับ headroom ของ AMS1117)

**สมการ:**
$$V_{secondary,peak} \approx k \cdot \frac{n_2}{n_1} \cdot V_{primary,peak}$$

**ค่าที่ใช้:**
- V_primary,peak = π × 12V = 38V (ที่ resonance)
- k (coupling) = 0.4 (PCB coil + air gap 3-5mm)
- n₁ = 17 turns
- ต้องการ V_secondary,peak = 10V

$$\frac{n_2}{n_1} = \frac{10}{38 \times 0.4} = 0.66$$

→ n₂ ≈ 11 turns (เกือบ minimum)

**บวก margin เผื่อ coupling แย่กว่าคิด:**
→ **n₂ = 14 turns** (ปลอดภัย)

**Trace:** กว้างขึ้น = resistance ต่ำลง
- Width: 0.8mm
- Gap: 1.3mm

---

## 📐 4. PCB Coil Specifications สุดท้าย

| Coil | หน้าที่ | Diameter (Out/In) | Turns | Trace Width | Gap | Layer | Type |
|---|---|---|---|---|---|---|---|
| **L1 Primary** | Power transmit + LC tank | 90/30 mm | 17 × 2 = 34 | 0.4 mm | 0.48 mm | Top+Bottom (via) | **Bifilar** |
| **L2 Coupling** | Feedback to MOSFET gates | 28/10 mm | 6 | 0.3 mm | 0.5 mm | Top only | Single |
| **L3 Secondary** | Power receive | 90/30 mm | 14 | 0.8 mm | 1.3 mm | Top+Bottom (via) | Single |

---

## 🔬 5. หลักการ Coupling & Air Gap

### 5.1 Coupling Coefficient (k)

**ค่า k บอกอะไร:**
- k = 1.0 → coupling สมบูรณ์ (เช่น transformer มี core)
- k = 0.0 → ไม่ coupling เลย
- **PCB coil + air gap 3-5mm:** k ≈ 0.3-0.5

**ปัจจัยที่กระทบ k:**
| ปัจจัย | ผลต่อ k |
|---|---|
| Air gap น้อยลง | k เพิ่ม |
| Coil diameter ใหญ่ขึ้น | k เพิ่ม |
| Coil aligned ตรงกัน | k เพิ่ม |
| Coil shape เหมือนกัน | k เพิ่ม |
| Off-center | k ลดลงมาก |

### 5.2 ทำไม Air Gap 3-5mm?

**ใหญ่กว่านี้ (>5mm):**
- k ลดลงเร็ว
- ต้องเพิ่มกำลัง Royer → ร้อน
- Efficiency ตก

**น้อยกว่านี้ (<3mm):**
- กระทบกันได้ตอนหมุน
- Mechanical tolerance ลำบาก
- Bearing อาจขยับ

**3-5mm = sweet spot** สำหรับ:
- Mechanical safety margin
- k > 0.4 (efficiency ดี)
- ไม่ต้องการ precision machining

---

## ⚠️ 6. ความท้าทาย & การแก้ปัญหา

### 6.1 Polarity ของ Coupling Coil

**ปัญหา:** ถ้า L2 ต่อขั้วผิด → MOSFET ทั้งสองเปิดพร้อมกัน → ลัดวงจร → พังทันที!

**วิธีตรวจสอบ:**
1. เปิดไฟครั้งแรกผ่าน **current-limited supply** (ตั้ง limit 100mA)
2. หรือใส่ resistor 10Ω อนุกรมที่ V+
3. ถ้ากระแสกระโดด → ปิดทันที, สลับขั้ว L2
4. ถ้า oscillate ปกติ → ต่อตรงๆ ได้

### 6.2 Capacitor ต้องเป็น Film เท่านั้น

**ห้าม:** ceramic capacitor

**เหตุผล:**
- Ceramic 100nF Y5V ที่ AC 38V peak → **ค่าลด 50%**
- **DC bias effect** ทำให้ค่าไม่นิ่ง
- Ceramic ร้อนเร็วเมื่อทำงานที่ 120 kHz

**ใช้:** Polypropylene film cap, ≥250V rating

### 6.3 Tuning หลังประกอบ

ค่า L ที่คำนวณได้อาจ **คลาดเคลื่อน ±20%** เพราะ:
- PCB thickness ไม่เท่ากัน
- Copper thickness ผันแปร
- Layer alignment

**วิธี tune:**
1. วัดความถี่จริงด้วย oscilloscope
2. ถ้า f สูงกว่าเป้า → เพิ่ม C
3. ถ้า f ต่ำกว่าเป้า → ลด C

**Tip:** เริ่มด้วย C = 47nF + 100nF (ขนาน) → 147nF → tune ลงได้

### 6.4 Air Gap คงที่

**ปัญหา:** ถ้า rotor กระดิก → coupling กระเพื่อม → output voltage แกว่ง

**การแก้:**
- ใช้ **standoff/spacer** ที่แน่นอน
- Mounting hole ที่ rotor ตรงกับ bearing
- ตรวจ wobble ก่อนทดสอบความเร็วสูง

---

## 🎨 7. PCB Layout Strategy

### 7.1 Stack-up

**ฐานล่าง (Primary side):**
```
Top Layer:    L1 ครึ่งที่ 1 (spiral นอก→ใน) + L2 + Royer driver
              Vias เชื่อมไป Bottom
Bottom Layer: L1 ครึ่งที่ 2 (spiral ใน→นอก) + GND plane
```

**Rotor (Secondary side):**
```
Top Layer:    LED + IC + Arduino (rotor topside ↑)
Bottom Layer: L3 + bridge rectifier + power supply + GND plane
              (ด้านที่หันลงเข้าหา primary)
```

### 7.2 Spiral Pattern

**Square spiral vs Circular spiral:**

| รูปทรง | ข้อดี | ข้อเสีย |
|---|---|---|
| **Square spiral** | Routing ง่าย, KiCad native | Inductance ต่ำกว่า circular ~5% |
| **Circular spiral** | Inductance สูงสุด | Routing ยาก ต้อง custom angle |
| **Octagonal spiral** | Compromise ที่ดี | ค่าระหว่าง 2 แบบ |

**แนะนำ:** Octagonal spiral (8-sided) — KiCad routable ใน 45° mode

### 7.3 Via Connection

L1 bifilar ต้องใช้ vias เพื่อข้ามตัวเอง:

```
Top layer:    เริ่มขอบนอก →  spiral ใน → via ลง Bottom
Bottom layer: via ขึ้น Top → spiral ออก → ขอบนอก (อีกขั้ว)
```

**Via spec:**
- Size: 0.6 mm drill, 1.0 mm pad
- Quantity: ~10 vias per coil (กระจายตามจุด crossing)
- Current rating: 1 via ≈ 1A (สบายสำหรับ 750mA)

---

## 🧪 8. การทดสอบ (Testing Plan)

### 8.1 ทดสอบ Royer (ไม่มี secondary)

**Step 1:** ต่อแค่ Royer + L1 + L2 (ไม่มี L3 rotor)
**Step 2:** ค่อยๆ เพิ่มไฟจาก 0V → 12V ผ่าน current-limited supply
**Step 3:** วัดที่ L1:
- ความถี่ ≈ 120 kHz ✓
- Sinewave shape ✓
- Peak voltage ≈ 30-40V ✓
- กระแสจากแหล่งจ่าย ≈ 200-300mA (no load)

**ถ้าผิดปกติ:**
- ไม่ oscillate → ตรวจ L2 polarity
- ความถี่ผิด → ปรับ C
- MOSFET ร้อน → ตรวจ ZVS timing

### 8.2 ทดสอบ Coupling (มี L3)

**Step 1:** วาง rotor PCB เหนือ Royer ที่ air gap 3-5mm
**Step 2:** วัดที่ L3 ด้วย oscilloscope:
- AC peak ≈ 10-12V ✓
- ความถี่เดียวกับ L1 ✓
**Step 3:** ต่อ rectifier + dummy load (resistor 10Ω):
- V_DC หลัง rectifier ≈ 7-9V ✓
- Ripple < 200mV ✓

### 8.3 ทดสอบ Full System

**Step 1:** ต่อ rotor ทั้งบอร์ด, **ยังไม่หมุน**
**Step 2:** วัด +5V rail:
- = 5.0V ± 0.1V ✓
- กระแสรวม ≈ 400mA ✓
**Step 3:** ทดสอบ LED ติด/ดับผ่าน Arduino code
**Step 4:** ค่อยหมุนช้าๆ (เริ่มที่ 200 RPM) → เพิ่มความเร็ว
**Step 5:** ที่ 2000 RPM ตรวจ:
- ภาพ POV ปรากฏชัด
- Hall sensor trigger ถูกต้อง
- ไม่มี restart Arduino

---

## 🎓 9. คำถามที่อาจารย์อาจถาม + คำตอบ

### Q1: ทำไมต้องใช้ Resonant Royer ไม่ใช่ Classic Royer?

**A:** Classic Royer พึ่ง core saturation → core ร้อน, loss สูง, ใช้กับ ferrite transformer
Resonant Royer พึ่ง LC tank → ZVS efficiency 90%+, sinewave (low EMI), เหมาะกับ PCB coil + air-core

### Q2: ทำไมไม่ใช้ Qi standard (15W) แทน?

**A:**
- Qi ใช้ controller IC ซับซ้อน (BQ51, MP-Q)
- ต้องการ communication protocol ระหว่าง primary/secondary
- Overkill สำหรับ 4W static power transfer
- Royer ง่ายกว่า, ราคาถูกกว่า 10 เท่า

### Q3: ทำไมไม่ใช้ ferrite core ใส่ใน coil?

**A:**
- Air gap ที่ต้องการ (~5mm) — ferrite core จะ saturate ที่ gap นี้
- PCB coil + air-core: ทำซ้ำได้, ราคาเป็นส่วนของ PCB
- Ferrite จะเพิ่มน้ำหนัก, ขนาด, ความซับซ้อน
- Efficiency ต่างกัน <10% สำหรับ application นี้

### Q4: ปัญหา EMI ระดับไหน? FCC compliance?

**A:**
- Royer 120 kHz อยู่ใน ISM band ที่ controlled
- Sinewave (resonant) → harmonics น้อยกว่า square-wave switch
- Ground pour + shielding → ลด radiated emissions
- สำหรับ academic project: ไม่ต้อง FCC certified
- สำหรับ commercial: ต้องเพิ่ม EMI filter + shielding

### Q5: ทำไมไม่ใช้ slip ring?

**A:**
| Slip Ring | Wireless (Royer) |
|---|---|
| ทำให้สึกหรอ (carbon brush) | ไม่สึกหรอ |
| Noise จากการเสียดสี | ไม่มี contact noise |
| Mechanical wear → ต้องเปลี่ยน | ไม่มี wear |
| ราคาแพง (mini ~500฿+) | ราคาเป็นส่วนของ PCB |
| ความเร็วจำกัด | หมุนได้ไม่จำกัด |
| ต้องการ space ตรงกลาง | ใช้ PCB area เดิม |

### Q6: ค่า k = 0.4 ดีพอหรือยัง?

**A:**
- Industry standard wireless charging: k ≈ 0.5-0.7
- ของเรา 0.4: efficiency loss ~10% เพิ่มเติม
- ยอมรับได้ เพราะ:
  - Total efficiency ยัง > 70%
  - กำลังต้องส่งแค่ ~4W (ไม่ใช่ระดับ 100W)
  - Simplicity > efficiency optimization

### Q7: ทำไมเลือก 120 kHz ไม่ใช่ 200 kHz หรือ 50 kHz?

**A:**
- **<50 kHz:** Coil ใหญ่เกินไป, ไม่พอดี PCB 120mm
- **>200 kHz:** Switching loss สูง, capacitor ESR เริ่มเป็นปัญหา
- **120 kHz = sweet spot:**
  - Coil ขนาดพอดี
  - MOSFET ทั่วไป (IRFZ44N) ทำงานได้ดี
  - Cap film polypropylene หาง่ายที่ค่านี้
  - ตรงกับ RD40 ต้นฉบับ — ผ่านการ verify แล้ว

---

## 📚 10. References เพิ่มเติม

1. **Mohan's Formula for PCB Spiral Inductors**
   - Mohan, S. S., et al. "Simple accurate expressions for planar spiral inductances." IEEE J. Solid-State Circuits, 1999.

2. **Royer Converter Original Patent**
   - US Patent 2783384 (Royer, 1957)

3. **Resonant Royer (Baxandall)**
   - Baxandall, P. J. "Transistor sine-wave LC oscillators." Proc. IEEE, 1959.

4. **Wireless Power Transfer Theory**
   - Kurs, A., et al. "Wireless power transfer via strongly coupled magnetic resonances." Science, 2007.

5. **RD40 Project Documentation**
   - lhm0, Hackaday RD40, 2023

---

## ✅ สรุป Royer Coil System

ระบบ Royer coil ของเราใช้แนวคิด **3 coils ทำงานร่วมกัน:**
- **L1 Bifilar Primary** (17 turns × 2): สร้าง resonance + ส่งกำลัง
- **L2 Coupling** (6 turns): feedback ขับ MOSFET
- **L3 Secondary** (14 turns): รับกำลังบน rotor

ออกแบบทั้งหมดเป็น **PCB spiral coils** ผ่านการคำนวณตามทฤษฎี:
- Mohan's formula → กำหนด turns
- Resonance equation → กำหนด LC tank
- Coupling theory → กำหนด air gap & alignment

ทำงานที่ **120 kHz, 4-5W output, efficiency ~70%, k ≈ 0.4** — เหมาะกับ rotor 120mm ที่หมุน 2000 RPM

---

📖 คำอธิบายประกอบการดู Animation
ลองกด Play แล้วดู animation ทำงาน — ผมจะอธิบายสิ่งที่เกิดขึ้นในแต่ละ step ให้
ขั้นที่ 1-4: ครึ่งคลื่นแรก (Positive Half-Cycle)
Step 1: Q1 เปิด — MOSFET ตัวซ้าย "เปิดสวิตช์" ทำให้กระแสไหลจาก 12V → choke → L1a (ครึ่งซ้ายของ primary coil) → Q1 → GND
Step 2: สนามแม่เหล็กขยายตัว — เมื่อกระแสไหลผ่าน coil → สร้างสนามแม่เหล็ก B ตามกฎ Ampere ขยายออกจาก coil
Step 3: L3 รับสนามแม่เหล็ก — สนาม B พุ่งผ่าน air gap → ตัด L3 บน rotor → ตามกฎ Faraday เกิด induced EMF ขั้วบวก
Step 4: D1+D4 conduct — แรงดัน AC ครึ่งคลื่นบวก → D1 และ D4 forward biased → กระแสไหลไป V+ output
ขั้นที่ 5-8: ครึ่งคลื่นที่สอง (Negative Half-Cycle)
Step 5: Q2 เปิด, Q1 ปิด — LC tank "ring" ทำให้ MOSFET สลับกันโดยอัตโนมัติ (self-oscillation)
Step 6: B field กลับทิศ — กระแสไหลผ่าน L1b ในทิศตรงข้าม → สนาม B กลับทิศด้วย
Step 7: L3 induced AC ขั้วลบ — สนาม B ทิศใหม่ตัด L3 → induced EMF กลับขั้ว
Step 8: D2+D3 conduct — แม้ AC กลับขั้ว แต่ bridge rectifier ฉลาด — เลือก D2 และ D3 แทน → กระแสยังไหลไป V+ ทิศเดียวกับครึ่งคลื่นแรก!
🔑 จุดสำคัญที่ต้องเข้าใจ
1. Self-Oscillation: Royer ไม่ต้องมี clock หรือ microcontroller — LC tank สั่นเองที่ 120 kHz เพราะ resonance
2. Wireless ผ่าน Magnetic Field: ไม่มีสายไฟข้าม air gap — แค่สนามแม่เหล็กที่สั่นเปลี่ยนทิศ 240,000 ครั้งต่อวินาที (120 kHz × 2)
3. Full-Wave Rectification: Bridge rectifier ใช้ครึ่งคลื่นทั้งบวกและลบของ AC → จ่ายไฟ DC ออกมาทุกครึ่งคลื่น (efficiency สูง)
4. LED ติดต่อเนื่อง: ความถี่ 120 kHz เร็วมาก → cap หลัง rectifier กรองให้เป็น DC นิ่ง → LED ติดต่อเนื่องไม่กระพริบ
💡 เปรียบเทียบให้เห็นภาพ
ลองนึกถึง:

Royer = หัวใจที่เต้นเอง ไม่ต้องมีสมองสั่ง — pace maker built-in
Air gap = ทางเดินของแสง ผ่านอากาศได้ แม้บอร์ดหมุน
Bridge rectifier = วาล์วน้ำ บังคับให้ไฟไหลทิศเดียว ไม่ว่า input จะกลับขั้วยังไง