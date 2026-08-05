# RF-Based-Intelligent-Soldier-Safety-and-Silent-Communication-System
## 1. Project Overview

Two independent embedded units:

- **Soldier Unit (Transmitter)** — ESP32, reads buttons + MPU6050, sends silent
  tactical messages and emergency fall alerts over RF via an HT12E encoder.
- **Base Station (Receiver)** — Arduino UNO, decodes RF data via HT12D,
  displays messages, drives LEDs, and sounds an active buzzer alarm.

### 2. Troubleshooting: OLED and MPU6050 both not working

The OLED and MPU6050 share **one I2C bus** (GPIO21 = SDA, GPIO22 = SCL). If
**both** stop responding at the same time, the cause is almost always the
shared bus itself, not two coincidentally broken parts. Check in this order:

1. **SDA/SCL not swapped** — OLED SDA and MPU6050 SDA must both land on
   GPIO21; OLED SCL and MPU6050 SCL must both land on GPIO22.
2. **Common ground** — ESP32 GND, OLED GND, and MPU6050 GND must all share
   the same ground rail. A missing ground on just one board can silence
   the entire bus.
3. **`Wire.begin()` pin order** — must be `Wire.begin(I2C_SDA_PIN,
   I2C_SCL_PIN)`, i.e. `Wire.begin(21, 22)` on ESP32 (SDA first, then SCL).
4. **Reseat every jumper** on the breadboard header rows — a very common
   failure point when several wires are crowded into one row.
5. **Run an I2C scanner sketch** (search "Arduino I2C scanner", works as-is
   on ESP32 with `Wire.begin(21, 22)`). Expect to see `0x3C` (OLED) and
   `0x68` (MPU6050). Finding **neither** confirms a bus/wiring/ground issue
   rather than a code or component fault; finding only one means you have
   two separate problems, not one shared cause.
6. **Check MPU6050's AD0 pin** is tied to GND (or left floating) so its
   address stays at `0x68` and doesn't drift/float onto the bus.
7. **Power budget** — try powering both modules from 5V instead of the
   ESP32's 3.3V pin if multiple devices share that rail; an underpowered
   rail can brown out both I2C devices simultaneously.

The firmware itself already treats this defensively: `initializeOLED()`
and `initializeMPU()` each check their own return value and report
failure independently on the self-test screen and over Serial (115200
baud) — but a bus-level short/miswiring will make *both* report failure
together, which is the signature described above.

## 3. Flowchart

### 3.1 Soldier Unit (Transmitter)

```
[Power On]
   │
   ▼
[Initialize OLED, MPU6050, HT12E pins]
   │
   ▼
[Self-Test: OLED / MPU6050 / RF] ──fail──► [Display SYSTEM ERROR]
   │ pass
   ▼
[Display READY screen: ID / STATUS / RF]
   │
   ▼
[Loop] ──► [Read Buttons] ──pressed──► [Send RF code] ─► [Display Sending → Sent] ─┐
   │                                                                                │
   ├──► [Read MPU6050: accel + gyro] ─► [Compute total accel, pitch, roll, gyro]   │
   │           │                                                                    │
   │      impact ≥ threshold?                                                      │
   │           │ yes                                                               │
   │      within fusion window: orientation Δ AND gyro spike?                      │
   │           │ yes                                                               │
   │           ▼                                                                   │
   │   [SOLDIER DOWN: send RF 1111, display EMERGENCY] ◄──────────────────────────┘
   │           │
   └───────────┴──► back to [Loop]
```

### 3.2 Base Station (Receiver)

```
[Power On]
   │
   ▼
[Initialize OLED, HT12D pins, LEDs, Alarm pin]
   │
   ▼
[Display "COMMAND CENTER / Waiting..."]
   │
   ▼
[Loop] ──► [Poll HT12D VT pin]
              │ valid pulse (debounced)
              ▼
        [Read 4-bit data word] ──► [Decode command]
              │
              ▼
   ┌─────────────────────────────────────────────┐
   │  Need Backup / Enemy Spotted / Mission Done  │──► Green LED, display message
   │  Need Medical Help                           │──► Red LED, alarm ON, display
   │  Soldier Down                                │──► Red LED, alarm ON, EMERGENCY screen
   └─────────────────────────────────────────────┘
              │
              ▼
   [Timers expire (non-blocking)] ──► LED/alarm reset, return to Waiting (unless Emergency)
```

---

## 4. Working Explanation

**Silent messaging:** The soldier presses one of four buttons. The ESP32
debounces the input, loads the corresponding 4-bit code onto the HT12E data
lines, and pulses the TE (Transmit Enable) pin LOW for ~250 ms so the HT12E
repeatedly transmits the encoded word through the RF link. The OLED shows
"Sending…" then "Message Sent" before returning to the ready screen — all
using `millis()` timers so the button loop is never blocked.

**Fall detection (sensor fusion, not a timer):** Every 20 ms the ESP32 reads
the MPU6050's accelerometer and gyroscope. It computes total acceleration
magnitude (to catch a hard impact), plus pitch/roll from the accelerometer
and angular velocity from the gyroscope (to catch a sudden orientation
change). A "Soldier Down" event is only declared when a high-acceleration
impact **and** a correlated sudden rotation/orientation change both occur
within a short time window. This avoids false positives from a soldier
diving into cover or standing still (which a simple "wait 5 seconds"
inactivity timer would misinterpret), while still reacting immediately —
there is no waiting period before the alert is sent.

**Receiving and alerting:** The Arduino UNO continuously polls the HT12D's
VT (Valid Transmission) pin. When a debounced, valid pulse is detected, the
4-bit data lines are read and mapped back to a message. Routine messages
light the green LED; medical/emergency messages light the red LED and
pulse an active buzzer module on pin D9 on/off using `digitalWrite()` to
create an attention-grabbing alarm pattern. Since active buzzers have
their own internal oscillator, no PWM tone generation is needed — just
on/off switching.
