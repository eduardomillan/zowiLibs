# zowiLibs vs OttoDIYLib: Library Comparison

This document compares the **Zowi Arduino libraries** (`arduinolibs/` in this
repository, by bq) with the official **OttoDIYLib**
(`github.com/OttoDIY/OttoDIYLib`). It focuses on **similarities and differences**
to inform an eventual Zowi → Otto-compatible port.

> TL;DR: The two libraries share a **common ancestor** (Obijuan's `Oscillator`
> and the same biped-robot design). Their motion/sound/gesture APIs are almost
> **byte-for-byte identical**. The main differences are the **LED mouth driver**
> (Zowi 74HC595 5×6 vs Otto MAX7219 8×8), **ESP32 support**, **serial command
> handling**, and a few **extra Otto helpers**.

---

## Table of Contents

- [1. Origin & Lineage](#1-origin--lineage)
- [2. High-Level Similarities](#2-high-level-similarities)
- [3. File / Module Structure](#3-file--module-structure)
- [4. Main Class API Comparison](#4-main-class-api-comparison)
- [5. Initialization Signature Difference](#5-initialization-signature-difference)
- [6. Sounds: Identical](#6-sounds-identical)
- [7. Gestures: Identical Values, Different Prefix](#7-gestures-identical-values-different-prefix)
- [8. Mouth / LED Matrix: The Biggest Difference](#8-mouth--led-matrix-the-biggest-difference)
- [9. Servo / Oscillator Engine](#9-servo--oscillator-engine)
- [10. Board / Architecture Support](#10-board--architecture-support)
- [11. Serial / Communication](#11-serial--communication)
- [12. Sensors & Battery](#12-sensors--battery)
- [13. Otto-Only Features](#13-otto-only-features)
- [14. Zowi-Only Features](#14-zowi-only-features)
- [15. Porting Implications](#15-porting-implications)
- [16. Summary Table](#16-summary-table)

---

## 1. Origin & Lineage

Both libraries descend from the same open-source biped robot work:

- **Obijuan (Juan González-Gómez)** authored the `Oscillator` servo engine used
  **verbatim** by both projects.
- bq's **Zowi** and the community **Otto DIY** were developed in the same era
  from the same concept (4-servo biped, LED mouth, buzzer, ultrasonic sensor).
- As a result, the class methods, movement algorithms, sound tables, and gesture
  definitions are **the same code** with cosmetic renaming (`Zowi` → `Otto`).

This makes the two libraries **highly interoperable** at the API level.

---

## 2. High-Level Similarities

Both libraries provide:

- A single main class (`Zowi` / `Otto`) wrapping a 4-servo biped.
- The Obijuan `Oscillator` for smooth sinusoidal servo motion.
- Identical **movement** methods: `walk, turn, jump, bend, shakeLeg`.
- Identical **dance** methods: `moonwalker, crusaito, flapping, swing,
  tiptoeSwing, jitter, updown, ascendingTurn`.
- Identical **low-level motion**: `_moveServos`, `oscillateServos`, `_execute`,
  `home`, `getRestState`, `setRestState`.
- Identical **calibration**: `setTrims`, `saveTrimsOnEEPROM` (trims stored in
  EEPROM addresses 0–3).
- Identical **sound** API: `_tone`, `bendTones`, `sing` + the same `S_*` sound
  IDs and note-frequency table.
- Identical **gesture** API: `playGesture(int)` + the same 13 gesture IDs.
- Identical **mouth** API surface: `putMouth`, `putAnimationMouth`, `clearMouth`.
- Same motion **constants**: `FORWARD/BACKWARD/LEFT/RIGHT/SMALL/MEDIUM/BIG`.

---

## 3. File / Module Structure

| Purpose | zowiLibs | OttoDIYLib |
|---------|----------|------------|
| Main class | `Zowi/Zowi.h` + `Zowi.cpp` | `src/Otto.h` + `Otto.cpp` |
| Sounds | `Zowi/Zowi_sounds.h` | `src/Otto_sounds.h` |
| Gestures | `Zowi/Zowi_gestures.h` | `src/Otto_gestures.h` |
| Mouths | `Zowi/Zowi_mouths.h` | `src/Otto_mouths.h` |
| LED matrix driver | `LedMatrix/` (separate lib, 74HC595) | `src/Otto_matrix.*` (MAX7219) |
| Servo engine | `Oscillator/` (separate lib) | `Oscillator.*` (bundled in src) |
| Ultrasonic | `US/` (separate lib) | bundled/inline in Otto |
| Battery | `BatReader/` (separate lib) | not a dedicated class |
| Serial parser | `ZowiSerialCommand/` (separate lib) | `SerialCommand` (bundled) |
| Button interrupts | `EnableInterrupt/` (bundled dep) | left to the sketch |

Key structural difference: **zowiLibs ships several independent libraries**
(each in its own folder: `Zowi`, `Oscillator`, `US`, `LedMatrix`, `BatReader`,
`ZowiSerialCommand`, `EnableInterrupt`). **OttoDIYLib bundles everything under
one `src/`** as a single Arduino library.

---

## 4. Main Class API Comparison

Comparing `Zowi.h` and `Otto.h`, the public method sets are almost identical.

**Common to both (identical signatures):**

```
attachServos(), detachServos()
setTrims(YL,YR,RL,RR), saveTrimsOnEEPROM()
_moveServos(time, target[]), oscillateServos(A,O,T,phase,cycle)
home(), getRestState(), setRestState(state)
jump, walk, turn, bend, shakeLeg
updown, swing, tiptoeSwing, jitter, ascendingTurn
moonwalker, crusaito, flapping
putMouth, putAnimationMouth, clearMouth
_tone, bendTones, sing
playGesture
```

**Only in Otto:**

```
_moveSingle(position, servo_number)      // move one servo
initMATRIX(DIN, CS, CLK, rotate)         // MAX7219 setup
matrixIntensity(intensity)               // MAX7219 brightness
setLed(X, Y, value)                      // 8x8 pixel control
writeText(s, scrollspeed)                // scrolling text on matrix
enableServoLimit(speed), disableServoLimit()  // servo speed limiter
```

**Only in Zowi:**

```
getDistance()        // ultrasonic (US lib) exposed on the class
getNoise()           // noise sensor via analogRead
getBatteryLevel()    // battery % (BatReader)
getBatteryVoltage()  // battery voltage (BatReader)
```

> Note: Otto can still read sensors, but in Otto sketches these are typically
> done inline/with helper libs rather than as methods on the `Otto` class.
> Zowi promotes sensors/battery to first-class class methods.

---

## 5. Initialization Signature Difference

The `init()` signatures differ, reflecting how each library treats peripherals:

**Zowi** (buzzer, noise sensor, and ultrasonic pins are all parameters):

```cpp
void init(int YL, int YR, int RL, int RR,
          bool load_calibration = true,
          int NoiseSensor = PIN_NoiseSensor,
          int Buzzer = PIN_Buzzer,
          int USTrigger = PIN_Trigger,
          int USEcho = PIN_Echo);
```

**Otto** (only buzzer is a parameter):

```cpp
void init(int YL, int YR, int RL, int RR,
          bool load_calibration, int Buzzer);
```

The 4 servo pins + calibration flag are common. Zowi bakes in more of the Zowi
hardware (US + noise sensor) at init time; Otto keeps `init()` leaner.

---

## 6. Sounds: Identical

`Zowi_sounds.h` and `Otto_sounds.h` are effectively the same file. Both define
the same note-frequency table (`note_C0` … `note_D8`) and the **same 19 sound
IDs with identical values**:

```
S_connection=0  S_disconnection=1  S_buttonPushed=2
S_mode1=3       S_mode2=4          S_mode3=5
S_surprise=6    S_OhOoh=7          S_OhOoh2=8
S_cuddly=9      S_sleeping=10      S_happy=11
S_superHappy=12 S_happy_short=13   S_sad=14
S_confused=15   S_fart1=16         S_fart2=17   S_fart3=18
```

`sing(int songName)`, `_tone(...)`, and `bendTones(...)` behave the same.

---

## 7. Gestures: Identical Values, Different Prefix

The gesture IDs are **numerically identical**; only the macro prefix differs
(`Zowi*` vs `Otto*`):

| ID | Zowi | Otto |
|----|------|------|
| 0 | `ZowiHappy` | `OttoHappy` |
| 1 | `ZowiSuperHappy` | `OttoSuperHappy` |
| 2 | `ZowiSad` | `OttoSad` |
| 3 | `ZowiSleeping` | `OttoSleeping` |
| 4 | `ZowiFart` | `OttoFart` |
| 5 | `ZowiConfused` | `OttoConfused` |
| 6 | `ZowiLove` | `OttoLove` |
| 7 | `ZowiAngry` | `OttoAngry` |
| 8 | `ZowiFretful` | `OttoFretful` |
| 9 | `ZowiMagic` | `OttoMagic` |
| 10 | `ZowiWave` | `OttoWave` |
| 11 | `ZowiVictory` | `OttoVictory` |
| 12 | `ZowiFail` | `OttoFail` |

Because the underlying integers match, `playGesture()` is cross-compatible: a
simple `#define OttoHappy ZowiHappy` alias layer would suffice.

---

## 8. Mouth / LED Matrix: The Biggest Difference

This is the **most significant hardware/software difference**.

| Aspect | Zowi | Otto |
|--------|------|------|
| Display | 5×6 LED matrix | 8×8 LED matrix |
| Driver IC | **74HC595** shift register | **MAX7219** |
| Library | `LedMatrix` (bit-bang SER/CLK/RCK) | `Otto_matrix` (SPI-like MAX7219) |
| Storage | `unsigned long memory` (30 bits) | 8×8 buffer |
| Extra features | none | brightness, `setLed(x,y)`, scrolling `writeText()` |
| Init | via `LedMatrix` constructor pins | `initMATRIX(DIN, CS, CLK, rotate)` |

Both expose `putMouth`, `putAnimationMouth`, `clearMouth`, but the **bitmap
encoding differs** (5×6 vs 8×8), so mouth graphics are **not** directly
interchangeable. Otto additionally supports **text scrolling** and per-pixel
control that Zowi's 74HC595 driver does not.

---

## 9. Servo / Oscillator Engine

- Both use **Obijuan's `Oscillator`** class with the same API
  (`SetA/SetO/SetT/SetPh/SetTrim/SetPosition/refresh/attach/detach`).
- Motion algorithms (`_moveServos`, `oscillateServos`, `_execute`) are the same.
- **Otto adds a servo speed limiter** (`enableServoLimit` /
  `SERVO_LIMIT_DEFAULT = 240` deg/s) and `_moveSingle()`, which Zowi lacks.
- Servo indexing convention is the same order: `{YL, YR, RL, RR}` (Otto docs
  label them LeftLeg/RightLeg/LeftFoot/RightFoot — same physical mapping).

---

## 10. Board / Architecture Support

| | zowiLibs | OttoDIYLib |
|--|----------|------------|
| AVR (UNO/Nano/bq Zum) | ✅ (original target) | ✅ |
| ESP8266 | ❌ | ✅ |
| ESP32 | ❌ (needs porting) | ✅ (`#ifdef ARDUINO_ARCH_ESP32` → `ESP32Servo.h`) |

Otto.h already switches between `<Servo.h>` and `<ESP32Servo.h>` at compile
time; zowiLibs hardcodes `<Servo.h>` and depends on the AVR-only
`EnableInterrupt` library, so it needs the migration described in
`MIGRATE_ESP32.md` to run on ESP32.

---

## 11. Serial / Communication

| | zowiLibs | OttoDIYLib |
|--|----------|------------|
| Parser | `ZowiSerialCommand` (fork of Steven Cogswell's `SerialCommand`) | `SerialCommand` (same lineage) |
| Protocol | ZowiApp: single-letter cmds `S,L,T,M,H,K,C,G,R,E,D,N,B,I`, framed `&&...%%` | Otto app: its own command set |
| Bluetooth | External module on hardware `Serial` | External module (AVR) or **native BLE on ESP32** |

Both serial parsers share the same origin, but the **application-level
protocols differ** (ZowiApp vs Otto app), which matters for app compatibility.

---

## 12. Sensors & Battery

| | zowiLibs | OttoDIYLib |
|--|----------|------------|
| Ultrasonic | dedicated `US` lib + `getDistance()` method | inline / helper, sketch-driven |
| Noise sensor | `getNoise()` method (`analogRead`) | sketch-driven `analogRead` |
| Battery | dedicated `BatReader` lib + `getBatteryLevel/Voltage()` | no dedicated class |

Zowi exposes richer **on-class** sensor/battery helpers; Otto keeps the core
class focused on motion + mouth and defers sensors to the sketch/examples.

---

## 13. Otto-Only Features

- `_moveSingle()` — move a single servo.
- Full **MAX7219 8×8 matrix** support: `initMATRIX`, `matrixIntensity`,
  `setLed`, `writeText` (scrolling text).
- **Servo speed limiter** (`enableServoLimit` / `disableServoLimit`).
- **ESP8266 / ESP32** architecture support out of the box.
- Larger, actively-maintained example set and Arduino Library Manager listing.

---

## 14. Zowi-Only Features

- On-class **battery** monitoring (`BatReader`: `getBatteryLevel`,
  `getBatteryVoltage`).
- On-class **noise sensor** (`getNoise`) and **ultrasonic** (`getDistance`).
- **ZowiApp protocol** integration and the specific `&&...%%` framing.
- Modular per-feature library layout (each peripheral is its own library).
- `EnableInterrupt`-based button handling packaged with the project.

---

## 15. Porting Implications

Given the near-identical core, making Zowi Otto-compatible is mostly a matter of
**thin adaptation layers** rather than a rewrite:

1. **API alias layer** — `#define`/wrapper mapping `Otto*` names to `Zowi*`
   (gestures, sounds) and an `Otto` facade delegating to the Zowi motion core.
   Movement/dance/sound methods already match 1:1.
2. **init() reconciliation** — adapt Otto's `init(...,Buzzer)` onto Zowi's
   richer `init()` (default the extra Zowi pins).
3. **Mouth abstraction** — the real work: bridge Zowi's 74HC595 5×6 driver and
   Otto's MAX7219 8×8 API (`IMouth` interface + best-effort bitmap mapping).
4. **ESP32 support** — adopt Otto's `#ifdef ARDUINO_ARCH_ESP32` pattern and the
   changes in `MIGRATE_ESP32.md` (ESP32Servo, EEPROM emulation, interrupts,
   tone, ADC rescale).
5. **Protocol layer** — keep ZowiApp parsing; optionally add Otto app / BLE.

The shared `Oscillator` and identical motion/sound/gesture tables mean
**movement, sound, and gesture behavior transfer with essentially no changes**.

---

## 16. Summary Table

| Dimension | zowiLibs | OttoDIYLib | Verdict |
|-----------|----------|------------|---------|
| Servo engine | Obijuan `Oscillator` | Obijuan `Oscillator` | **Same** |
| Movements/dances | full set | full set | **Identical API** |
| Sound IDs & notes | `S_*` 0–18 | `S_*` 0–18 | **Identical** |
| Gesture IDs | `Zowi*` 0–12 | `Otto*` 0–12 | **Identical values** |
| Calibration/EEPROM | trims 0–3 | trims 0–3 | **Same** |
| Mouth display | 74HC595 5×6 | MAX7219 8×8 | **Different** |
| Single-servo / limiter | ✗ | ✓ | Otto extra |
| Sensors/battery on class | ✓ | ✗ (sketch) | Zowi extra |
| Architectures | AVR only | AVR + ESP8266 + ESP32 | Otto broader |
| Serial protocol | ZowiApp `&&...%%` | Otto app | Different |
| Packaging | multiple libs | single `src/` | Different layout |

---

## 17. References

- Zowi libraries: `arduinolibs/` in this repository.
- OttoDIYLib: `github.com/OttoDIY/OttoDIYLib`
- Oscillator author: Juan González-Gómez (Obijuan).
- Companion docs: `docs/project/MIGRATE_ESP32.md`,
  `docs/project/ZOWI_VS_OTTO_VS_ESP32.md`.
