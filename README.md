# Executive Summary

The **Pressure-Insole V4** is a compact wearable PCB that reads 16 force-sensitive resistors (FSRs) via two 16:1 analog multiplexers into an ESP32-S3 microcontroller. It is powered by a 2-cell (2S) Li-ion pack stepped down to 3.3 V by a TPS62162 buck regulator. The board is designed for a flexible insole: it omits a bulky USB-C and instead uses six exposed ENIG-plated pogo pads for programming. This README documents the V4 goals, hardware, wiring, PCB features, assembly hints, programming steps, BOM, and design evolution from V1 through V4.

## Project Overview, Goals, and Intended Use

- **Goal:** Build a self-contained smart insole that samples pressure from 16 FSRs and transmits data via Wi‑Fi/BLE.
- **Use Case:** The V4 PCB is meant for a wearable foot insole. It must be small, flexible, and robust, with minimal external connectors. Programming is done off-board via pogo pins.
- **Major Components:** The design centers on an **ESP32-S3-WROOM-1** module (dual-core LX7 MCU with Wi-Fi/BLE), two **CD74HC4067** 16:1 analog multiplexers, 16 FSR sensors (wired one per mux channel), a **2S Li-ion** battery, and a **TPS62162** 3.3 V/1 A buck converter.

## Hardware Summary

- **MCU:** ESP32-S3-WROOM-1 (PCB antenna version). Dual-core 32-bit Xtensa LX7, 240 MHz, Wi‑Fi 802.11b/g/n, Bluetooth 5.0. 36 GPIOs, ADCs, native USB D+/D– (on GPIO19/20).
- **Sensors:** 16 FSRs, arranged as voltage dividers. Each FSR leg goes from 3.3 V → MUX1 → FSR → MUX2 → resistor → GND.
- **Muxes:** 2× TI CD74HC4067 (16:1 analog mux). Operates at 3.3 V supply, typical on-resistance \~60 Ω, ensures bidirectional analog switching. Both MUXs share the same 4 select lines from the ESP32. The “enable” pins (E) are tied LOW (always enabled).
- **Resistors:** Each channel has a fixed divider: **390 Ω + 360 Ω** in series (\~750 Ω total). These standard values bias the FSR to ground. (Exact values can be tuned later.)
- **Power Regulator:** TI **TPS62162** synchronous buck. Input 3–17 V, fixed 3.3 V output, 1 A max. Chosen for small 2×2 mm WSON package and \~17 μA quiescent current (DCS-Control™).
- **Battery:** Two flexible Li-ion cells in series (2S, \~7.4 V nominal, 8.4 V max). This provides power to the TPS62162. A 2S battery pack requires an appropriate protection/BMS and charger (open item).
- **LEDs:** A green (or white) status LED on a GPIO for user feedback (blinking on Wi-Fi connect, etc.), plus optional red LED for power indicator.
- **Programming Pads:** Six copper pads (no USB port on PCB): GND, 3.3V, USB\_D– (GPIO19), USB\_D+ (GPIO20), GPIO0 (BOOT), and EN (reset). These pads are plated with ENIG for durability and sized for pogo-pin contact.

## PCB Features

- **Module & ICs:** The ESP32 module footprint (18×25.5 mm) sits at one end, two 24-pin VQFN footprints for the CD74HC4067 muxes in the middle, and power components at the opposite end.
- **No USB-C:** To save space and reduce flex stress, there is no USB connector. Programming is done via external pogo pin adapter.
- **Pogo Pads:** Six exposed ENIG copper pads along the board edge. These should be large, solder-mask–defined pads without vias (vias in pads can trap solder or cause misalignment). ENIG plating is recommended for corrosion resistance and wear.
- **Antenna:** The ESP32’s onboard PCB antenna (on module) should be oriented away from metal parts and kept near the board edge. Keep copper pours and ground areas away from the antenna region. (Exact antenna clearance TBD.)
- **Solder Mask:** Use Solder Mask Opening (SMO) larger than copper pad for pogo contacts to avoid mask residue; ensure a slight border for alignment.
- **Footprints:** Follow datasheet reference footprints. For example, use TI’s recommended land pattern for TPS62162 (8-pin WSON) and CD74HC4067 (24-pin VQFN/SSOP). The ESP32-WROOM footprint is standardized (18.0×25.5 mm, shown in ESP datasheet).
- **Ground Planes:** Use a solid ground plane. Keep high-speed traces (if any) short; in our design only digital/analog switching at moderate speed, so place decoupling caps close to power pins.

## Power Architecture and BMS/Charger Notes

- **2S Battery:** Two cells in series (6.0–8.4 V). The pack must include a 2S protection/BMS IC to handle balance charging, cell over/under-voltage, and possibly current limiting. Common ICs include TI BQ76920 or BQ76940 (open design decision). If using parallel cells instead (1S2P), a buck‑boost could be used instead of buck, but 2S was chosen for simplicity of wiring.
- **Buck Converter (TPS62162):** Steps the 6–8.4 V down to 3.3 V. The module’s VIN connects to the battery positive, and PGND/AGND to pack negative. Enable pin (EN) is tied high via 10 kΩ to VIN. An input capacitor (e.g. 10 µF) and output caps (two 10 µF + one 100 nF) are placed per TI’s datasheet recommendations to ensure stability. Layout should keep VIN, SW, PGND, and VOUT clusters short (see TI guidelines).
- **Battery Charger:** Not included on-board. A separate charger (e.g. USB- or wireless-charging module for 2S Li-ion) is required. This can be a dedicated 2S charger IC or external charging board. The design should ensure the load is disconnected or managed during charging (typically via the BMS).

## Detailed Connections and Pin Mapping

### ESP32-S3-WROOM-1 Pinout

| **ESP32 Signal** | **Net** | **Description** | **Connection(s)** |
| --- | --- | --- | --- |
| **3.3V** (pin 2)                            | 3V3     | Regulated 3.3 V rail          | Powers ESP32 VCC and all 3.3V nets (MUX VCC, LEDs, pull-ups)                  |
| **GND** (pins 5,7, etc)                     | GND     | System ground                 | Return for battery (-), regulator PGND, component GNDs                        |
| **EN** (pin 3)                              | EN      | Reset/enable (active low)     | Pulled up to 3.3V via 10 kΩ; also routed to pogo pad                          |
| **GPIO0** (pin 27)                          | BOOT    | Boot mode select              | Pulled up to 3.3V via 10 kΩ; also to pogo pad (drive low to enter bootloader) |
| **GPIO19** (pin 13)                         | USB\_D- | USB D– for native USB         | To pogo pad (for USB programming via S3’s native USB), alternate ADC2\_CH8    |
| **GPIO20** (pin 14)                         | USB\_D+ | USB D+ for native USB         | To pogo pad, alternate ADC2\_CH9                                              |
| **GPIO4** (pin 32)                          | S0      | MUX address bit S0            | Connect to CD74HC4067 S0 on both MUX1 and MUX2                                |
| **GPIO5** (pin 33)                          | S1      | MUX address bit S1            | → CD74HC4067 S1 (both MUXes)                                                  |
| **GPIO6** (pin 34)                          | S2      | MUX address bit S2            | → CD74HC4067 S2                                                               |
| **GPIO7** (pin 35)                          | S3      | MUX address bit S3            | → CD74HC4067 S3                                                               |
| **GPIO8** (pin 12)                          | ADC\_IN | ADC1\_CH7 (ADC input)         | Connected to the common output (COM) of MUX2 (through 750Ω to GND)            |
| **GPIO21** (pin 37)                         | STATUS  | Digital output for status LED | Drives an LED (e.g. to GND through 1–2.2 kΩ)                                  |

### Wiring (Nets and Components)

- **3V3 (3.3 V rail):** TPS62162 VOUT → 3V3 net → ESP32 VCC, CD74HC4067 VCC pins, status LED resistor(s). Place 100 nF decouplers on the 3V3 rail at each IC.
- **GND:** Common ground. Battery pack negative → ground plane. Tie PGND/AGND of regulator and all component GNDs here.
- **MUX1 (CD74HC4067 #1):**
  - VCC to 3.3V; GND to GND; E (pin 6) to GND (active low always on).
  - S0–S3 to ESP32 GPIO4–7 respectively.
  - SIG common (pin 1, labeled “COM”): tied to 3.3V (this mux supplies 3.3V to the selected channel).
  - I0–I15 (pins 9–16,19–23) each connects to one FSR, the other side of which goes to the corresponding MUX2 channel.
- **MUX2 (CD74HC4067 #2):**
  - VCC to 3.3V; GND to GND; E to GND.
  - S0–S3 also to ESP32 GPIO4–7 (same lines as MUX1 for shared addressing).
  - SIG common (COM) pin: connects through the 750 Ω network to ground and to ESP32 ADC (GPIO8/ADC1\_CH7).
  - I0–I15: each one connects to the same FSR as MUX1’s I0–I15 respectively (i.e. FSRn is between MUX1.In and MUX2.In).
- **FSR Wiring (for channel n):** MUX1.In\_n → FSR\_n (sensor) → MUX2.In\_n. The other end of FSR\_n goes to GND through 390Ω + 360Ω. Thus the voltage at MUX2.COM = (3.3V \* R\_FSR) / (750Ω + R\_FSR).
- **Resistor Network:** Each FSR line has a series 390Ω and 360Ω (total \~750Ω) to GND. These can be implemented as two discrete 0603 or 0805 resistors per channel (common easy values).
- **Pogo Pads:**
  - Pad1 = GND (ties to ground plane).
  - Pad2 = 3.3V (connect to 3V3 net).
  - Pad3 = USB\_D– (ESP32 GPIO19/pin13).
  - Pad4 = USB\_D+ (ESP32 GPIO20/pin14).
  - Pad5 = GPIO0 (boot) (pin27).
  - Pad6 = EN (reset) (pin3).

> *See “Connections and Pin Mapping” tables below for a concise wiring list.*

## PCB Assembly Notes

- **Footprints:** Use manufacturer-recommended land patterns. For example, the CD74HC4067 can use TI’s [application note layout](https://www.ti.com/lit/pdf/sccy157) or standard 24-pin SSOP (or 24-pin 5.5×3.5 mm VQFN if space-critical). The TPS62162 uses TI’s 2×2 mm WSON footprint. The ESP32 module footprint follows Espressif’s datasheet (18×25.5 mm).
- **Pogo Pad Preparation:** Pads should be solder-mask–defined, with a clear copper area (solder-mask openings) and plated (prefer ENIG). No vias in the pad area. Optionally, a small solder bump can be applied (by including pad in solder-paste stencil) to improve contact reliability.
- **Decoupling:** Place 0.1 µF (100 nF) ceramic capacitors right next to each IC’s VCC pin (ESP32, both MUXes, regulator VIN/VOUT). Add 1–10 µF bulk ceramics on VIN and VOUT of the regulator as per datasheet. Place an EN pull-up (10 kΩ) next to ESP32 EN pin.
- **Antenna Clearance:** Keep the area around the ESP32 module’s antenna free of ground pours or copper. A 3–5 mm clearance zone around the antenna quarter-wave trace is recommended. Avoid components above the antenna.
- **No Solder Thief Needed:** As there are no high-speed differential lines or ethernet, no special solder-thief or impedance control is needed, beyond normal short routing.

## KiCad Design Hints

- **Symbol Library:** Use or create symbols for the CD74HC4067 (24-pin analog mux) and define nets for S0–S3, E, COM, I0–I15. Espressif provides a Kicad library for ESP32 modules, or use a generic module footprint and assign pins manually using the datasheet.
- **Net Labels:** Name nets clearly (e.g. `S0`, `MUX1_COM`, `MUX2_COM`, `FSR0`, etc.). This helps in seeing the connections for each sensor.
- **Hierarchical Sheets:** Consider one sheet for power (battery, regulator), one for MCU, and one or more for sensors/muxes.
- **Ground and 3V3 Planes:** Define a solid GND zone and 3.3V zone. Use polygons for better thermal dissipation. Ensure vias in these zones connect appropriately.
- **Reset/Boot Circuit:** Use the design from Espressif’s reference: a 10 kΩ pull-up on EN and GPIO0, with optional push-button footprints if manual reset/programming is desired. We skip buttons in final design, but you can still include footprints for debugging.
- **Modularity:** If uncertain about resistor values or battery config, leave footprints for alternative parts (e.g. 0 Ω jumpers to parallel two 1S booster boost circuits). Mark these as “open design decisions” for future revision.

## Testing and Programming Procedure (Pogo Jig)

1. **Assemble Jig:** Build a small PCB or fixture that wires USB-C to USB signals and supplies power:
   ```
   pgsql
   ```
   
   ```
          USB-C (to PC) ──> 5V/3.3V ──> pogo pad 2 (3.3V)
                         USB D+ ──> pogo pad 4
                         USB D- ──> pogo pad 3
                         GND   ──> pogo pad 1
        Also include two control lines: one to pad5 (GPIO0) and one to pad6 (EN).

   ```
2. **Attach Pogo Connectors:** Align pogo pins with the 6 exposed pads. Ensure correct orientation. Pad1 (GND) often has a distinctive shape.
3. **Power-Up:** Plug USB-C into the jig. It will provide regulated 3.3 V to the board (via pogo) and USB D+/D– for data. Ensure the 3.3 V pad reads \~3.3 V with a multimeter.
4. **Enter Bootloader (Upload Mode):** Pull **GPIO0 (pad5)** LOW and pulse **EN (pad6)** LOW (or toggle to GND) to reset the ESP32. This enters the ROM bootloader. Keep USB connected.
5. **Flash Firmware:** On your PC, use esptool or ESP-IDF to upload firmware. For example, with ESP-IDF:
   ```
   bash
   ```
   
   ```
   idf.py -p /dev/ttyUSB0 flash monitor

   ```
   or with esptool (adjust COM port and baud):
   ```
   css
   ```
   
   ```
   esptool.py --chip esp32s3 --port COM3 write_flash 0x1000 firmware.bin

   ```
6. **Exit Bootloader:** After flashing, reset the board (pull EN low momentarily) to start running the new code. Release GPIO0 to let it boot normally.

```c
// Example: Arduino-style ADC read on GPIO8 (ADC1_CH7)
// and print FSR voltage (assuming 12-bit ADC, Vref=3.3V)
#include <Arduino.h>
void setup() {
  Serial.begin(115200);
  analogReadResolution(12);   // ESP32 ADC is 12-bit by default
  pinMode(21, OUTPUT);        // status LED pin
}
void loop() {
  int raw = analogRead(8);    // Read ADC from pin 12 (GPIO8)
  float voltage = raw * (3.3 / 4095.0);
  Serial.print("FSR1 Voltage = ");
  Serial.println(voltage, 3);
  digitalWrite(21, (raw > 2000) ? HIGH : LOW); // light LED if pressure high
  delay(500);
}

```

## BOM (Bill of Materials)

| **Part** | **Value / Type** | **Footprint** | **Qty** | **Notes / Supplier Links** |
| --- | --- | --- | ---: | --- |
| **ESP32-S3-WROOM-1**                                 | Wi-Fi/BLE MCU module   | 18×25.5 mm QFN  | 1   | Espressif module                             |
| **CD74HC4067** (×2)                                  | 16:1 analog MUX        | 24-pin SSOP/VQFN | 2   | TI CD74HC4067                                |
| **TPS62162**                                         | 3.3 V 1 A buck conv.   | 2×2 mm WSON      | 1   | TI TPS62162                                  |
| **Resistor, 390 Ω**                                  | 1% SMD (0805)          | 0805             | 16  | FSR divider R1                               |
| **Resistor, 360 Ω**                                  | 1% SMD (0805)          | 0805             | 16  | FSR divider R2                               |
| **Capacitor, 100 nF**                                | 0.1 µF ceramic         | 0603/0805        | 6   | Decoupling (ESP32 VDD, MUXes, regin)         |
| **Capacitor, 10 µF**                                 | X5R MLCC               | 0805             | 2   | Input (10 µF) and output (10 µF) of buck     |
| **Capacitor, 1 µF**                                  | NP0/X5R MLCC           | 0603             | 2   | Additional decoupling                        |
| **LED (Green/White)**                                | 3 mm/0603 SMT diffused | 0603             | 1   | Status indicator                             |
| **Resistor, 2.2 kΩ**                                 | 1% SMD (0603)          | 0603             | 1   | LED current-limiter (adjust color as needed) |
| **Push Button (option)**                             | Tactile (optional)     | SMD or TH        | 0–2 | (For EN/GPIO0, if used for debug)            |
| **Battery Connector**                                | FPC/SMT (2-wire)       | (see note)       | 1   | Connector for 2S battery (to be chosen)      |
| **Battery Protection IC**                            | 2S Li-ion protection   | (QFN, TBD)       | 1\* | e.g. TI BQ76920 (open design)                |
| **Test Pads (ENIG)**                                 | Copper pads            | Custom           | 6   | GND, 3.3V, D-, D+, GPIO0, EN                 |
| **Standoff/Mount (optional)**                        | Nylon/Post             | –                | –   | For securing board, if needed                |

*\* Footprints and exact parts for battery connector/Protection IC depend on final mechanical and safety design (open item).*

For supplier links and datasheets, see the cited references. For example, Espressif’s [ESP32-S3-WROOM-1 datasheet] provides module details, TI’s [CD74HC4067 datasheet] covers the MUX specs, and TI’s [TPS62162 product page] lists regulator parameters.

## Pin Mapping

| **Signal** | **Board Net** | **ESP32 Pin # (Name)** | **Notes** |
| --- | --- | --- | --- |
| 3V3 (VCC)                                  | 3.3V        | Pin 2           | Regulator output → powers ESP32 VDD, MUX VCC   |
| GND                                        | GND         | Pins 5,7…       | Battery –, regulator PGND, etc.                |
| EN (Reset)                                 | EN          | Pin 3           | Active-low reset; pulled up to 3.3V            |
| GPIO0 (BOOT)                               | BOOT        | Pin 27          | Low at boot = serial download mode             |
| USB D–                                     | USB\_D-     | Pin 13 (GPIO19) | Native USB D– (to pogo)                        |
| USB D+                                     | USB\_D+     | Pin 14 (GPIO20) | Native USB D+ (to pogo)                        |
| MUX S0                                     | S0          | Pin 32 (GPIO4)  | Address bit 0 (to both MUX S0)                 |
| MUX S1                                     | S1          | Pin 33 (GPIO5)  | Address bit 1 (to both MUX S1)                 |
| MUX S2                                     | S2          | Pin 34 (GPIO6)  | Address bit 2 (to both MUX S2)                 |
| MUX S3                                     | S3          | Pin 35 (GPIO7)  | Address bit 3 (to both MUX S3)                 |
| MUX1 COM                                   | 3.3V (Vref) | (not MCU)       | Tied to 3.3V (MUX1 supplies sensor power)      |
| MUX2 COM                                   | ADC\_IN     | Pin 12 (GPIO8)  | Connect to ADC1\_CH7; goes to resistor network |
| Status LED                                 | LED\_OUT    | Pin 37 (GPIO21) | Drives an LED to GND via resistor              |
| (Misc. GPIOs)                              | –           | –               | Many GPIOs remain free for future use          |

*ESP32 pin numbers refer to the WROOM module pin labels as per the datasheet.*

## Connection List

| **Net Name** | **Source** | **Destination(s)** |
| --- | --- | --- |
| **3.3V**                         | TPS62162 VOUT           | ESP32 VCC, MUX1 VCC, MUX2 VCC, LED resistors, pull-ups on EN/GPIO0            |
| **GND**                          | Battery– (pack return)  | TPS62162 PGND, ESP32 GNDs, MUX1 GND, MUX2 GND, resistor network, LED cathodes |
| **S0**                           | ESP32 GPIO4 (pin32)     | CD74HC4067 #1 S0, CD74HC4067 #2 S0                                            |
| **S1**                           | ESP32 GPIO5 (pin33)     | CD74HC4067 #1 S1, CD74HC4067 #2 S1                                            |
| **S2**                           | ESP32 GPIO6 (pin34)     | CD74HC4067 #1 S2, CD74HC4067 #2 S2                                            |
| **S3**                           | ESP32 GPIO7 (pin35)     | CD74HC4067 #1 S3, CD74HC4067 #2 S3                                            |
| **MUX1.I0–I15**                  | FSR0–FSR15              | MUX2.I0–I15 (the other ends of each FSR)                                      |
| **MUX1.COM**                     | 3.3V                    | Supplies Vref to selected FSR through MUX1                                    |
| **MUX2.COM**                     | Resistor net            | Connects through 750Ω to GND and to ESP32 ADC (pin12)                         |
| **ResNet (FSR)**                 | MUX2.COM                | FSRs & resistors: Each FSR to two series resistors to GND                     |
| **GPIO0**                        | ESP32 pin27 (pulled-up) | Pogo pad (to force BOOT when held low)                                        |
| **EN**                           | ESP32 pin3 (pulled-up)  | Pogo pad (ground to reset)                                                    |
| **USB D-**                       | ESP32 GPIO19/pin13      | Pogo pad (for USB programmer D–)                                              |
| **USB D+**                       | ESP32 GPIO20/pin14      | Pogo pad (for USB programmer D+)                                              |

## Design Evolution (V1 → V4)

The project progressed through several versions: V1 was a simple breadboard prototype with a single MCU and a few sensors; V2 introduced an ESP32 and a single multiplexer; V3 moved to a custom PCB with USB-C and basic buck conversion; and V4 (this design) finalizes a wearable-friendly layout. The timeline below summarizes major milestones:

```
2023-01V1 –Proof-of-concept onbreadboard (fewFSRs, no wireless)2023-06V2 – ESP32 devkit,16-channel MUX viaCD74HC4067 added2024-02V3 – First PCB (withUSB-C, power reg);still bulky prototype2024-09V4 – Current designflex insole PCB,pogo-padprogramming, 2Sbattery, native USBInsole Project V1–V4 Evolution
```

*Figure: Project version timeline (mermaid).*

Each revision addressed specific issues: increasing sensor count, adding wireless comms, refining power management, and finally mechanical integration (removing USB jack, adding pogo pads) for the insole form factor.

## Troubleshooting and Measurement Checklist

- **Power Rails:** Verify 3.3 V output from TPS62162 (no load and with load). Check \~3.3 V at ESP32 VCC and MUX VCC.
- **Ground:** Ensure solid ground continuity between battery –, regulator PGND, and ESP32 GND.
- **Boot Mode:** Check pull-up resistors on EN and GPIO0 (should read 3.3 V at reset).
- **Regulator Layout:** If output is unstable or overheats, re-check input/output capacitors and inductor orientation per TPS62162 datasheet.
- **Mux Functionality:** With the board powered, manually switch S0–S3 and measure continuity: the 3.3 V from MUX1.COM should appear at one FSR input, and the resistor network from the same FSR should connect to MUX2.COM.
- **ADC Reading:** Use a multimeter or temporary ADC code to read known voltages: e.g., disconnect an FSR and measure \~0V, tie MUX2.COM to 3.3 V and verify ADC \~4095 (12-bit).
- **Pogo Pads:** Use an ohmmeter to confirm each pad to its net: e.g. Pad1↔GND, Pad5↔GPIO0 pin27, etc.
- **LEDs:** Apply 3.3 V to the LED resistor net and ensure LEDs light correctly.
- **Antenna:** For RF sanity, verify no ground copper under antenna and that antenna region is free per Espressif guidance.

## Next Steps and Open Decisions

- **FSR Resistor Tuning:** The 390+360 Ω network was estimated. Fine-tune these based on actual FSR resistance range and desired sensitivity. Possibly measure FSR curves and adjust divider for best ADC span.
- **ADC Calibration:** Implement an ADC calibration procedure (ESP32-S3 ADC is nominally 12-bit with Vref \~1100 mV). Use either one-shot or continuous ADC driver with calibration to improve accuracy.
- **Battery Charger/BMS:** Select an appropriate 2S charging IC or module. Options include linear chargers for 2S or switching charger; ensure inclusion of a proper protection IC (e.g. TI BQ76920 series) on the PCB or pack. Mark this as an external module if not on-board.
- **Antenna Optimization:** If custom geometry needed, consider the choice between ESP32-S3-WROOM-1 (PCB antenna) vs WROOM-1U (u.FL) for flexibility. V4 used the PCB antenna version; future versions might require tuning or a matching network.
- **Firmware Features:** Add Wi‑Fi/BLE stack, packet protocol, and sensor fusion code. Possibly implement power-saving modes (light-sleep) to reduce battery drain.
- **Enclosure/Packaging:** Design a physical holder or encapsulation for the insole electronics to protect from moisture and mechanical stress.

## Assembly and Test Checklist (Concise)

- **Component Inspection:** Verify part values and orientation (IC pins, resistor values 390/360Ω). Check footprints match parts.
- **Footprint Verification:** Confirm SOT23-5 (if used for Reg), QFN/SOIC pinouts.
- **Before Soldering:** Inspect PCB silk for pin1 markings, net labels. Ensure no solder-mask shorts on antenna area.
- **Populate & Inspect:** Solder ESP32 module first (if reflow, align using guides). Then place MUXes and regulator. Hand-solder passives (caps, resistors).
- **No Shorts:** Check for solder bridges, especially on fine-pitch parts (ESP32, MUX).
- **Continuity:** Verify basic continuity: VCC to reg-out, GND plane continuous, no shorts to 3.3V.
- **Power-Up Smoke Test:** Power the board via the pogo jig or regulated supply (limit current to \~200 mA). Check 3.3V rail. No excessive heat or smoke.
- **Programming:** Connect pogo jig, enter bootloader, flash a “blink LED” test sketch. Confirm LED blinks.
- **Sensor Readings:** Connect or simulate FSRs (e.g. a fixed resistor) and read ADC values. Verify changing resistor changes ADC linearly.
- **Range Test:** With actual FSRs on, apply known weights or use a reference resistor to verify sensor linearity and no channel saturation.

**Congratulations!** Following this guide should allow assembly and use of the V4 insole PCB. For more details on each component, consult the cited datasheets and TI application notes. Good luck with your project!
