# Zowi vs Otto vs ESP32: Plan to Make Zowi an Otto-Compatible ESP32 Robot

This document is a **planning document only**. It describes the strategy to
convert a Zowi robot (originally built on the **bq Zum** board, an Arduino
UNO-compatible board based on the 8-bit **ATmega328P**) into an **Otto-compatible
robot running on an ESP32**, reusing the original Zowi body.

> No development is started by this document. It records goals, findings,
> architecture and milestones so implementation can begin later.

---

## Table of Contents

- [0. Investigation Findings (baseline)](#0-investigation-findings-baseline)
- [0.1 Ecosystem Compatibility Matrix (AVR vs ESP32)](#01-ecosystem-compatibility-matrix-avr-vs-esp32)
- [1. Goals & Scope (confirmed)](#1-goals--scope-confirmed)
- [2. Target Architecture (layered, dual-stack)](#2-target-architecture-layered-dual-stack)
- [3. Prerequisite: ESP32 Port of the Zowi Core](#3-prerequisite-esp32-port-of-the-zowi-core)
- [4. Multi-Board Support (ESP32 DevKit + Arduino Nano ESP32)](#4-multi-board-support-esp32-devkit--arduino-nano-esp32)
  - [4.1 Proposed pin maps (to be finalized during implementation)](#41-proposed-pin-maps-to-be-finalized-during-implementation)
- [5. Otto-Compatibility Layer](#5-otto-compatibility-layer)
- [6. Dual Protocol / App Support](#6-dual-protocol--app-support)
- [7. Modular HP Robots Accessories (total scope)](#7-modular-hp-robots-accessories-total-scope)
- [8. Deliverables (when implementation starts)](#8-deliverables-when-implementation-starts)
- [9. Validation Plan](#9-validation-plan)
- [10. Milestones](#10-milestones)
- [11. Risks & Open Questions](#11-risks--open-questions)
- [12. References](#12-references)

---

## 0. Investigation Findings (baseline)

- **hprobots.com is ESP32-based.** HP Robots is the official commercial
  evolution of the open-source Otto DIY project. Its current platform (the
  "Ottoky" IoT robot, the web-blocks coding editor, and the Creator kits) runs
  on **ESP32** with WiFi + BLE. Their contest/docs explicitly state advanced
  users can use "any **ESP32-compatible coding platform**, including Thonny IDE
  for MicroPython or the Arduino IDE".
- **Otto DIY has had ESP32 support since 2019** (`OttoDIY/DIY/Otto_ESP32`), and
  the official `OttoDIYLib` lists **ESP32** among compatible hardware. Community
  ports exist (e.g. `rhansenne/OttoDIY_ESP32`) featuring native BLE app control,
  OTA updates, and multiple interaction modes.
- **Zowi and Otto share a common ancestry.** Both use the Obijuan
  `Oscillator.h` servo algorithm and the same **4-servo biped** architecture.
  Their public APIs are almost identical:

  | Concept | Zowi | Otto |
  |---------|------|------|
  | Init | `zowi.init(YL,YR,RL,RR,cal,...)` | `Otto.init(LeftLeg,RightLeg,LeftFoot,RightFoot,cal,Buzzer)` |
  | Movements | `walk/turn/jump/bend/shakeLeg` | identical |
  | Dances | `moonwalker/crusaito/flapping/swing/tiptoeSwing/jitter/updown/ascendingTurn` | identical |
  | Sounds | `sing()`, `_tone()`, `bendTones()` | identical |
  | Gestures | `playGesture(int)` | `playGesture(OttoHappy...)` |
  | Oscillator | Obijuan `Oscillator` | Obijuan `Oscillator` |

- **Zowi-specific differences to reconcile:**
  - **Mouth:** Zowi uses a **5×6 LED matrix driven by a 74HC595 shift register**
    (`LedMatrix` lib, pins SER/CLK/RCK). Otto typically uses an **8×8 matrix with
    a MAX7219**.
  - **App protocol:** Zowi uses a **proprietary ZowiApp serial protocol** with
    single-letter commands (`S,L,T,M,H,K,C,G,R,E,D,N,B,I`) framed as `&&...%%`,
    parsed by `ZowiSerialCommand`. Otto uses its own Bluetooth command set.

---

## 0.1 Ecosystem Compatibility Matrix (AVR vs ESP32)

A common source of confusion is treating "Otto / HP Robots" as a single thing.
It is actually **three distinct layers**, and board support differs per layer.
This is important because it determines what a converted Zowi can and cannot do
on a given microcontroller.

### The three layers

1. **Otto DIY open-source library & desktop tools** — the classic maker path.
2. **Otto Blockly (desktop app)** — offline visual programming, open source.
3. **HP Robots commercial ecosystem (hprobots.com)** — the new "HP Otto"
   product: web-blocks editor, mobile app, MicroPython, WiFi/IoT/AI, RGB, and
   modular expansions.

### Compatibility per layer

| Layer / feature | Arduino UNO / Nano (ATmega328) | ESP32 |
|-----------------|-------------------------------|-------|
| `OttoDIYLib` (Arduino IDE) | ✅ Supported (avr) | ✅ Supported (esp32) |
| Otto Blockly desktop (board selector: Nano, Uno, Leonardo, Micro, Mega, ZUM BT-328, Nano Every, MKR WiFi, ESP8266/ESP32) | ✅ Supported | ✅ Supported |
| HP Robots **web** code editor (hprobots.com/otto-robot/code) | ❌ Not supported | ✅ ESP32-only |
| HP Robots **mobile app** + Bluetooth control | ❌ Not possible on classic AVR | ✅ Native BLE |
| **MicroPython** (Thonny) | ❌ Not supported | ✅ Supported |
| **WiFi / IoT / AI / cloud** | ❌ No radio on ATmega328 | ✅ Built-in |
| RGB / modular >10 expansions | ⚠️ Limited by pins/memory | ✅ Designed for it |

### Key finding

- **The open-source Otto DIY project still fully supports Arduino UNO and Nano**
  (via `OttoDIYLib` — architectures `avr`, `esp8266`, `esp32` — and the desktop
  Otto Blockly board selector).
- **The current HP Robots commercial ecosystem (hprobots.com) is ESP32-only.**
  Its differentiating features (mobile app + Bluetooth, web-blocks editor,
  MicroPython, WiFi/IoT/AI, rich modularity) are physically impossible on a
  classic ATmega328 (no radio, limited flash/RAM). Otto DIY's own site frames
  this as two paths: *"Otto DIY"* (Arduino IDE only, **no** Bluetooth/App) vs
  *"HP Otto"* (Arduino IDE **and** MicroPython, Bluetooth + App + WiFi).
- So HP Robots did **not** port its full cloud/app stack down to AVR; it
  **adopted the open-source Otto project and built an ESP32-based premium layer
  on top**.

### Implication for the Zowi → Otto conversion

This **reinforces the ESP32 decision**:

- Targeting **UNO/Nano** would only give compatibility at the **library / desktop
  Blockly** level — never with the HP Robots app, web editor, or IoT ecosystem.
- Achieving the project's **"total ecosystem" goal** (code + app + web blocks +
  modules) **requires ESP32**. AVR is therefore a non-goal for this project,
  except as an optional secondary build target for the pure motion API.

---

## 1. Goals & Scope (confirmed)

- **Full dual compatibility:** keep the original **ZowiApp protocol** working
  **and** expose the **Otto API + Otto app/BLE** control.
- **Total ecosystem target:** code API + app/blocks control + modular HP Robots
  accessories.
- **Hardware:** reuse the **original Zowi body** (4 servos, HC-SR04 ultrasonic
  sensor, 74HC595 5×6 mouth, buzzer, noise sensor, battery divider, 2 buttons)
  with a **new ESP32 board** replacing the bq Zum.
- **Board support:** the design must support **both**:
  - **ESP32 DevKit** (38-pin / 30-pin WROOM dev board), and
  - **Arduino Nano ESP32** (drop-in on standard Nano shields / Otto-style I/O).

Out of scope for now: writing production code. This is planning only.

---

## 2. Target Architecture (layered, dual-stack)

```
+-------------------------------------------------------------+
|  Sketches: ZowiApp-compat sketch  |  Otto-compat sketch     |
+-------------------------------------------------------------+
|  Protocol layer:  ZowiSerialCommand  |  Otto app/BLE cmds    |
+-------------------------------------------------------------+
|  Unified motion API  (Zowi API  <->  Otto API alias shim)   |
+-------------------------------------------------------------+
|  HAL (ESP32): ESP32Servo | LEDC tone | EEPROM/NVS | ADC12 |  |
|               74HC595 mouth driver | US | interrupts        |
+-------------------------------------------------------------+
|  Board config: multi-board pin map (DevKit + Nano ESP32)    |
+-------------------------------------------------------------+
```

Key idea: **one shared motion/oscillator core** for the Zowi body, with **two
thin API facades** (`Zowi.*` and `Otto.*`) and **two protocol front-ends**
(ZowiApp serial + Otto BLE/app), all on a **board-agnostic HAL** that supports
both ESP32 DevKit and Arduino Nano ESP32.

---

## 3. Prerequisite: ESP32 Port of the Zowi Core

This depends on the companion document `docs/project/MIGRATE_ESP32.md`. Execute
that migration first. Summary of required changes:

- `<Servo.h>` → **ESP32Servo** (LEDC PWM), allocate LEDC timers for 4 (+ arms)
  servos.
- `<EEPROM.h>` → ESP32 EEPROM emulation (`begin`/`commit`) or **Preferences**
  (NVS) for trims + robot name.
- `<EnableInterrupt.h>` → core `attachInterrupt(digitalPinToInterrupt(...))`
  with `IRAM_ATTR` ISRs.
- `tone()`/`noTone()` → LEDC tone (or core `tone()` on recent ESP32 Arduino
  cores) for the buzzer.
- ADC rescale: 10-bit/5V (AVR) → **12-bit/3.3V** (ESP32) for noise + battery;
  recompute battery divider.
- HC-SR04 Echo is 5V → **level shift / voltage divider** to 3.3V.
- 74HC595 mouth: remap SER/CLK/RCK pins; the `asm nop` timing still works but
  consider `shiftOut()`.

This is **Milestone A**.

---

## 4. Multi-Board Support (ESP32 DevKit + Arduino Nano ESP32)

Provide a single board-config header that selects the pin map at compile time
based on the Arduino target macro, so the same firmware builds for both boards.

```
#if defined(ARDUINO_NANO_ESP32)
    // Arduino Nano ESP32 pin map (drop-in on Nano/Otto shields)
#elif defined(ARDUINO_ESP32_DEV) || defined(CONFIG_IDF_TARGET_ESP32)
    // ESP32 DevKit (WROOM) pin map
#else
    #error "Unsupported board: add a pin map"
#endif
```

Guidelines that apply to **both** boards:
- Keep all analog inputs (noise, battery) on **ADC1** (safe with WiFi/BLE on).
- Avoid boot-strapping pins for critical outputs where possible.
- Servo signal pins must be LEDC/PWM-capable.

### 4.1 Proposed pin maps (to be finalized during implementation)

| Function | AVR (bq Zum) | ESP32 DevKit (suggested) | Arduino Nano ESP32 (suggested) |
|----------|--------------|--------------------------|--------------------------------|
| Servo YL / LeftLeg  | 2  | 13 | D2 |
| Servo YR / RightLeg | 3  | 12 | D3 |
| Servo RL / LeftFoot | 4  | 14 | D4 |
| Servo RR / RightFoot| 5  | 27 | D5 |
| Button A | 6  | 32 | D6 |
| Button B | 7  | 33 | D7 |
| US Trigger | 8 | 25 | D8 |
| US Echo (÷ to 3.3V) | 9 | 26 | D9 |
| Buzzer | 10 | 4 | D13 (Otto default) |
| Mouth SER | 11 | 23 | D11 |
| Mouth RCK | 12 | 5  | D12 |
| Mouth CLK | 13 | 18 | D10 |
| Noise sensor | A6 | 35 (ADC1) | A6 |
| Battery | A7 | 34 (ADC1) | A7 |
| (Optional) Arm L/R | – | spare LEDC pins | spare D pins |

> The Nano ESP32 mapping aims to stay compatible with standard Nano/Otto I/O
> shields; the DevKit mapping targets a bare WROOM dev board on a breadboard/PCB.

---

## 5. Otto-Compatibility Layer

- **Reference `OttoDIYLib`** for exact signatures and gesture/sound enums; map
  them onto the ported Zowi core (do not blindly fork).
- **`Otto` facade class** (e.g. `arduinolibs/OttoCompat/Otto.h`):
  - Otto-style `init(LeftLeg,RightLeg,LeftFoot,RightFoot,cal,Buzzer)` delegating
    to the Zowi motion core.
  - Alias all movement/dance methods (already 1:1 with Zowi).
  - Map Otto **gesture enums** (`OttoHappy, OttoSad, OttoVictory, OttoAngry,
    OttoSleeping, OttoFretful, OttoLove, OttoConfused, OttoFart, OttoWave,
    OttoMagic, OttoFail, OttoSuperHappy`) to Zowi integer `playGesture()` cases
    via a lookup table; **document gaps** where sets differ.
  - Map Otto **sound enums** (`S_connection ... S_fart3`) to Zowi `sing()`
    indices.
- **Servo index reconciliation:** unify Zowi `{YL,YR,RL,RR}` with Otto
  `{LeftLeg,RightLeg,LeftFoot,RightFoot}` in one place.
- **Mouth abstraction (`IMouth` interface):**
  - `Mouth74HC595` — Zowi 5×6 backend (existing `LedMatrix`).
  - `MouthMAX7219` — optional Otto 8×8 backend (only if Otto electronics are
    later fitted).
  - Provide best-effort **8×8 → 5×6 bitmap mapping** so Otto mouth animations
    still render on the Zowi mouth (approximated).

---

## 6. Dual Protocol / App Support

- **Keep ZowiApp protocol:** port `ZowiSerialCommand` to ESP32 (plain C++,
  mostly portable) and preserve the `&&...%%` framing + single-letter handlers so
  the original ZowiApp keeps working over BLE-serial.
- **Add Otto app / BLE:** implement Otto's Bluetooth command set using the
  ESP32's **native BLE / BluetoothSerial** (no external HC-06 module needed).
  Use `OttoDIY` app protocol and `rhansenne/OttoDIY_ESP32` (BLE app + modes) as
  references.
- **Runtime/compile selection:** a flag
  (`PROTOCOL_ZOWI` / `PROTOCOL_OTTO` / `PROTOCOL_BOTH`) and/or a boot-time mode
  (button combo) to choose the active stack and avoid transport contention.
- **HP Robots web-blocks / MicroPython:** document how to flash the ESP32 on both
  boards; note HP Robots' block editor targets ESP32. Treat **full block-editor
  integration as a stretch goal** (may require their firmware/handshake —
  investigate separately).

---

## 7. Modular HP Robots Accessories (total scope)

- Reserve spare ESP32 GPIOs and an **I²C / Grove-style header** for HP
  Robots-style expansions (arms, extra sensors).
- Exploit the ESP32's extra PWM channels/memory to expose optional **arm
  servos** behind the facade (Otto ESP32 supports arms).
- Maintain a **pin-budget table** per board reserving pins for future modules;
  keep analog on ADC1 when WiFi/BLE is active.

---

## 8. Deliverables (when implementation starts)

- `docs/project/ZOWI_VS_OTTO_VS_ESP32.md` — this plan (persisted).
- `arduinolibs/OttoCompat/` — `Otto.h/.cpp` facade + gesture/sound/mouth maps.
- `arduinolibs/ZowiHAL/Zowi_pins_esp32.h` — multi-board pin config
  (DevKit + Nano ESP32).
- Mouth driver abstraction (`IMouth`, `Mouth74HC595`, optional `MouthMAX7219`).
- Example sketches: `examples/ZowiOtto_esp32/` (Otto API), a ZowiApp-compat
  sketch, and a `BOTH` demo.
- Wiring/README updates for the ESP32↔Zowi-body harness and level-shifting, for
  both boards.

---

## 9. Validation Plan

1. **Core bring-up (Milestone A):** servos, 74HC595 mouth, buzzer, US, ADC,
   buttons on ESP32 — verified on **both** DevKit and Nano ESP32.
2. **Otto API:** compile & run selected `OttoDIYLib` examples (`Otto_allmoves`,
   `Otto_avoid`, `Otto_APP`) against the facade.
3. **Zowi protocol regression:** original ZowiApp connects and drives via
   `&&...%%`.
4. **Otto app:** Otto BLE app connects and controls modes.
5. **Dual mode:** switch stacks at boot with no transport contention.
6. **Persistence:** trims survive power-cycle (EEPROM/NVS).
7. **Accessory smoke test:** attach one extra servo/sensor via the reserved
   header.

---

## 10. Milestones

- **A.** ESP32 port of the Zowi core (per `MIGRATE_ESP32.md`), building for both
  boards.
- **B.** Otto facade + gesture/sound/mouth mapping; Otto examples pass.
- **C.** Mouth abstraction (74HC595 ⇄ optional MAX7219).
- **D.** Dual protocol: ported ZowiApp + Otto BLE app.
- **E.** Modular accessory support + per-board pin budget.
- **F.** Docs, wiring, and stretch: HP Robots web-blocks / MicroPython.

---

## 11. Risks & Open Questions

- **Transport contention** between ZowiApp serial and Otto BLE → needs mode
  switching.
- **Mouth resolution mismatch** (5×6 vs 8×8) → some Otto faces render
  approximated.
- **Gesture/sound set differences** → a few Otto gestures may be unmapped
  (document).
- **HP Robots block editor** may require proprietary firmware/handshake →
  confirm feasibility before committing to full block compatibility.
- **3.3 V vs 5 V** on US Echo, buttons, 74HC595, noise/battery ADC → hardware
  adaptation required.
- **Board differences:** ESP32 DevKit (bare WROOM) vs Arduino Nano ESP32
  (Nano-shield friendly) differ in pin availability, USB flashing behavior, and
  power; the pin maps and wiring docs must cover both.

---

## 12. References

- Otto DIY libraries: `github.com/OttoDIY/OttoDIYLib`
- Otto ESP32 sketch: `github.com/OttoDIY/DIY/tree/master/Otto_ESP32`
- Community ESP32 BLE port: `github.com/rhansenne/OttoDIY_ESP32`
- ESP32Servo: `github.com/madhephaestus/ESP32Servo`
- HP Robots platform: `hprobots.com`
- Companion migration plan: `docs/project/MIGRATE_ESP32.md`
