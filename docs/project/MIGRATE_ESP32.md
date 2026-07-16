# Migrating Zowi Source Code to ESP32

This document describes a detailed plan to port the Zowi source code (currently
targeting the **bq Zum** board, an Arduino UNO-compatible board based on the
8-bit **ATmega328P**) to an **ESP32** (32-bit Xtensa LX6 / RISC-V, dual-core,
3.3 V logic, Wi-Fi + Bluetooth).

> **Note:** The precompiled `.hex` binaries in `code/` are AVR machine code and
> can **never** run on the ESP32. This plan is exclusively about recompiling the
> **source code** (`.ino` sketches + the libraries in `arduinolibs/`) for the
> ESP32 target.

---

## Table of Contents

- [1. Goals and Scope](#1-goals-and-scope)
- [2. Summary of AVR-Specific Dependencies Found](#2-summary-of-avr-specific-dependencies-found)
- [3. Toolchain Setup](#3-toolchain-setup)
- [4. Pin Mapping Plan](#4-pin-mapping-plan)
- [5. Library-by-Library Migration Steps](#5-library-by-library-migration-steps)
  - [5.1 Oscillator (`arduinolibs/Oscillator`)](#51-oscillator-arduinolibsoscillator)
  - [5.2 Servo count / LEDC channels](#52-servo-count--ledc-channels)
  - [5.3 EEPROM (`Zowi.cpp`)](#53-eeprom-zowicpp)
  - [5.4 Buttons / EnableInterrupt (`*.ino`)](#54-buttons--enableinterrupt-ino)
  - [5.5 Sound / tone (`Zowi.cpp:_tone`)](#55-sound--tone-zowicpp_tone)
  - [5.6 Battery reader (`BatReader.h/.cpp`)](#56-battery-reader-batreaderhcpp)
  - [5.7 Ultrasonic sensor (`US.cpp`)](#57-ultrasonic-sensor-uscpp)
  - [5.8 LED matrix (`LedMatrix.cpp`)](#58-led-matrix-ledmatrixcpp)
  - [5.9 Zowi core (`Zowi.cpp/.h`)](#59-zowi-core-zowicpph)
  - [5.10 Serial / Bluetooth (`*.ino`, ZowiSerialCommand)](#510-serial--bluetooth-ino-zowiserialcommand)
- [6. Sketch-Level Changes](#6-sketch-level-changes)
- [7. Validation Plan](#7-validation-plan)
- [8. Risks and Open Questions](#8-risks-and-open-questions)
- [9. Suggested Milestones](#9-suggested-milestones)

---

## 1. Goals and Scope

- Compile and run the following sketches on ESP32:
  - `code/base/ZOWI_BASE_v2.ino`
  - `code/factoryZowi/factoryZowi.ino`
  - `code/games/ZOWI_Adivinawi_v2/ZOWI_Adivinawi_v2.ino`
  - `code/games/ZOWI_Alarm_v2/ZOWI_Alarm_v2.ino`
- Port the supporting libraries in `arduinolibs/`:
  - `Zowi`, `Oscillator`, `US`, `LedMatrix`, `BatReader`, `ZowiSerialCommand`,
    `EnableInterrupt`
- Preserve the public API of the `Zowi` class so the sketches remain (mostly)
  unchanged.

Out of scope: redesigning the robot electronics. This plan assumes the same
peripherals (4 servos, HC-SR04 ultrasonic sensor, 74HC595-driven LED matrix,
buzzer, noise sensor, battery divider, 2 buttons) are wired to ESP32 GPIOs.

---

## 2. Summary of AVR-Specific Dependencies Found

| Area | Location | AVR dependency | ESP32 impact |
|------|----------|----------------|--------------|
| Servo control | `Oscillator.h/.cpp`, `Zowi` | `<Servo.h>` (AVR Timer1) | Replace with `ESP32Servo` (LEDC PWM) |
| EEPROM | `Zowi.cpp:28,77` | `<EEPROM.h>` direct read/write | Use ESP32 `EEPROM` emulation (`begin`/`commit`) or `Preferences` |
| Buttons / interrupts | `*.ino` | `<EnableInterrupt.h>`, `enableInterrupt()`, `disableInterrupt()` | Not compatible; use core `attachInterrupt(digitalPinToInterrupt(...))` |
| Buzzer / sound | `Zowi.cpp:709` | `tone()` / `noTone()` | Not implemented in older ESP32 cores; use LEDC tone or `ESP32Servo`'s tone, or a `tone()` shim |
| Analog read (noise) | `Zowi.cpp:549,552` | `analogRead(A6)` 10-bit, 5 V ref | ESP32 ADC is 12-bit, 3.3 V, pins are different; rescale |
| Battery reader | `BatReader.h:21-24` | `analogRead(A7)`, `ANA_REF 5`, `/1024` | Rescale for 12-bit ADC and 3.3 V ref; A7 does not exist |
| Ultrasonic | `US.cpp` | `pulseIn`, `digitalWrite`, `delayMicroseconds` | Portable, but verify 5 V Echo -> use level shifter/divider to 3.3 V |
| LED matrix | `LedMatrix.cpp:70-86` | `asm volatile ("nop")` timing, pins 11/12/13 (SPI) | `nop` compiles but timing differs; remap pins; consider `shiftOut` |
| Pin numbers | all sketches / libs | `2..13`, `A6`, `A7` | Remap to valid ESP32 GPIOs |
| Serial / Bluetooth | `*.ino` | `Serial` at 115200 for BT module | ESP32 has native BT (`BluetoothSerial`); can drop external module |
| PROGMEM / flash tables | `Zowi_mouths.h`, `Zowi_sounds.h`, `Zowi_gestures.h` | possible `PROGMEM` | On ESP32 flash access is transparent; verify no `pgm_read_*` misuse |

---

## 3. Toolchain Setup

1. Install the **ESP32 Arduino core** (via Boards Manager URL
   `https://espressif.github.io/arduino-esp32/package_esp32_index.json`, or via
   PlatformIO `platform = espressif32`).
2. Select a target board (e.g. `ESP32 Dev Module` / `esp32dev`).
3. Install required libraries:
   - `ESP32Servo` (servo + tone replacement)
   - Optionally `Preferences` (bundled with the core) for persistent storage.
4. Recommended: adopt **PlatformIO** with a `env:esp32dev` so the AVR
   (`env:uno`) and ESP32 builds can coexist and be validated separately.

---

## 4. Pin Mapping Plan

Define a single board-config header (e.g. `arduinolibs/Zowi/Zowi_pins.h` or a
per-sketch `#ifdef ESP32` block) so pin numbers are chosen per target. Suggested
ESP32 mapping (adjust to your wiring; avoid strapping pins 0/2/12/15 and
input-only pins 34-39 for outputs):

| Function | AVR pin | Suggested ESP32 GPIO | Notes |
|----------|---------|----------------------|-------|
| Servo YL | 2  | 13 | PWM-capable (LEDC) |
| Servo YR | 3  | 12 | avoid boot strap conflicts; verify |
| Servo RL | 4  | 14 | |
| Servo RR | 5  | 27 | |
| Button 2 | 6  | 32 | input, use `INPUT_PULLUP` if needed |
| Button 3 | 7  | 33 | input |
| US Trigger | 8 | 25 | output |
| US Echo  | 9  | 26 | **input-only tolerant; divide 5V->3.3V** |
| Buzzer   | 10 | 4  | LEDC channel |
| LedMatrix SER | 11 | 23 | |
| LedMatrix RCK | 12 | 5  | |
| LedMatrix CLK | 13 | 18 | |
| Noise sensor | A6 | 35 | ADC1, input-only |
| Battery | A7 | 34 | ADC1, input-only, via divider |

> Keep all servo and ADC pins on ADC1 (GPIO 32-39) if Wi-Fi is used, since ADC2
> conflicts with the Wi-Fi radio.

---

## 5. Library-by-Library Migration Steps

### 5.1 Oscillator (`arduinolibs/Oscillator`)
- Replace `#include <Servo.h>` with `#include <ESP32Servo.h>`.
- `Servo _servo;` -> `ESP32Servo` provides the same `attach/write/detach` API,
  so `Oscillator.cpp` should compile with minimal change.
- In `attach()`, call `ESP32PWM::allocateTimer()` once at init (per ESP32Servo
  docs) to reserve LEDC timers for the 4 servos.
- Verify `refresh()` timing still uses `millis()` (portable).

### 5.2 Servo count / LEDC channels
- ESP32 has 16 LEDC channels; 4 servos + 1 buzzer fit easily.
- Ensure the buzzer tone and servos do not share a timer that causes glitches.

### 5.3 EEPROM (`Zowi.cpp`)
- Option A (minimal change): use the ESP32 `EEPROM` emulation.
  - Add `EEPROM.begin(512);` in `Zowi::init()` before first read.
  - After each `EEPROM.write(...)` in `saveTrimsOnEEPROM()`, call
    `EEPROM.commit();`.
- Option B (recommended long-term): migrate to `Preferences` (NVS) with keys
  for each trim and the robot name; provides wear-leveling and clearer API.
- Keep the EEPROM address map documented in the root `README.md` (trims at
  0..3, name at 5+) if staying with Option A.

### 5.4 Buttons / EnableInterrupt (`*.ino`)
- Remove `#include <EnableInterrupt.h>`.
- Replace `enableInterrupt(pin, isr, RISING)` with
  `attachInterrupt(digitalPinToInterrupt(pin), isr, RISING);`.
- Replace `disableInterrupt(pin)` with `detachInterrupt(digitalPinToInterrupt(pin));`.
- Mark ISRs with `IRAM_ATTR` and keep them short; set only `volatile` flags.
- Configure button pins with `pinMode(pin, INPUT_PULLUP)` (or external pulls)
  and adjust edge (`FALLING`) to match wiring.

### 5.5 Sound / tone (`Zowi.cpp:_tone`)
- Older ESP32 cores lack `tone()`. Choose one:
  - Use `ESP32Servo`'s `tone(pin, freq)` / `noTone(pin)` helpers, or
  - Implement a small `tone()` wrapper using `ledcSetup`/`ledcWriteTone`
    (ledc API) on a dedicated buzzer channel.
- Recent ESP32 Arduino cores (>=2.0.1) do provide `tone()`/`noTone()`; if using
  such a core, no change may be needed. Verify at build time.

### 5.6 Battery reader (`BatReader.h/.cpp`)
- Change `#define BAT_PIN A7` to the mapped GPIO (e.g. 34).
- Change ADC math: ESP32 ADC is 12-bit, so replace `/1024` with `/4095`.
- Change `ANA_REF 5` to `3.3` (or the actual divider-scaled reference), and
  recompute the resistor divider so the max battery voltage maps under 3.3 V.
- Consider `analogSetAttenuation(ADC_11db)` and use ESP-IDF calibration for
  better linearity.

### 5.7 Ultrasonic sensor (`US.cpp`)
- Code is largely portable (`digitalWrite`, `delayMicroseconds`, `pulseIn`).
- **Hardware:** the HC-SR04 Echo outputs 5 V; the ESP32 is **not 5 V-tolerant**.
  Add a voltage divider or level shifter on the Echo line.
- `pulseIn` exists on ESP32 but is less accurate; if readings are noisy,
  reimplement using `pulseInLong` or a hardware timer / RMT.

### 5.8 LED matrix (`LedMatrix.cpp`)
- Remap `SER/CLK/RCK` default pins (11/13/12) to the ESP32 mapping.
- The `asm volatile ("nop")` calls compile on Xtensa but the timing is wildly
  different (ESP32 runs at 240 MHz vs 16 MHz). This is only a small delay for
  the 74HC595 setup/hold; it will still work but consider replacing the manual
  bit-bang loop with `shiftOut()` for clarity, or add explicit
  `delayMicroseconds`-scale timing if the shift register misbehaves.
- `digitalWrite` on ESP32 is slower than AVR, which actually helps satisfy
  74HC595 timing.

### 5.9 Zowi core (`Zowi.cpp/.h`)
- No structural change beyond the includes above and the EEPROM `begin/commit`.
- Verify `analogRead(pinNoiseSensor)` scaling in `getNoise()` (12-bit range) and
  adjust any thresholds used in gesture/mode logic in the sketches.
- Verify the flash-resident tables in `Zowi_mouths.h`, `Zowi_sounds.h`,
  `Zowi_gestures.h` do not rely on `PROGMEM` + `pgm_read_*`; on ESP32 plain
  arrays are directly addressable, so remove any `pgm_read_*` accessors if
  present.

### 5.10 Serial / Bluetooth (`*.ino`, ZowiSerialCommand)
- The original design talks to the ZowiApp over an external BT module on the
  hardware `Serial` (115200). On ESP32 you can either:
  - Keep using an external module on a `HardwareSerial` (e.g. `Serial2`), or
  - Use the ESP32's built-in Bluetooth via `BluetoothSerial` and feed its stream
    into `ZowiSerialCommand`. Ensure `ZowiSerialCommand` accepts a `Stream&`
    (refactor if it hardcodes `Serial`).

---

## 6. Sketch-Level Changes

For each `.ino`:
1. Add a target guard block, e.g.:
   ```cpp
   #if defined(ESP32)
     #define PIN_YL 13
     // ... ESP32 pins
   #else
     #define PIN_YL 2
     // ... AVR pins
   #endif
   ```
2. Swap the interrupt include/API as in 5.4.
3. Confirm `Serial.begin(115200)` and (optionally) `BluetoothSerial` init.
4. Rebuild and address compiler errors iteratively.

Recommended file order: `factoryZowi.ino` (calibration/hardware bring-up) ->
`ZOWI_BASE_v2.ino` -> the two games.

---

## 7. Validation Plan

1. **Build**: compile each sketch for `esp32dev` with no errors/warnings.
2. **Bring-up (factoryZowi)**: verify each subsystem individually:
   - Servos sweep and center correctly (trim calibration).
   - LED matrix displays known mouths.
   - Buzzer plays tones at correct pitch.
   - US sensor returns plausible distances (with level shifting).
   - Battery and noise ADC readings are in expected ranges after rescaling.
   - Both buttons trigger their ISRs.
3. **Persistence**: set trims, power-cycle, confirm they reload from EEPROM/NVS.
4. **Integration (ZOWI_BASE)**: run the 3 standalone mini-games (A, B, A+B).
5. **Comms**: verify command parsing over the chosen BT transport with ZowiApp
   or a serial terminal.
6. Regression: keep the AVR build working (dual-target) to avoid breaking bq Zum.

---

## 8. Risks and Open Questions

- **5 V vs 3.3 V**: every peripheral line (Echo, buttons, LED matrix, servo
  signal, sensors) must be checked; servos are usually fine with 3.3 V logic but
  need adequate current on their power rail.
- **ADC2 vs Wi-Fi**: if Wi-Fi is enabled, keep all analog inputs on ADC1.
- **tone()/Servo timer sharing**: LEDC channel/timer allocation conflicts.
- **BT stack size**: enabling `BluetoothSerial` increases flash/RAM usage and
  affects partition scheme.
- **pulseIn accuracy**: may need RMT-based ultrasonic driver.

---

## 9. Suggested Milestones

1. Toolchain + dual-target build skeleton (PlatformIO envs).
2. Oscillator/Servo port + servo bring-up.
3. EEPROM/Preferences persistence.
4. Buttons/interrupts + LED matrix + buzzer.
5. Sensors (US + battery + noise) with rescaling and level shifting.
6. Full `ZOWI_BASE` standalone modes passing.
7. Bluetooth comms with ZowiApp.
8. Port the two games and final regression.
