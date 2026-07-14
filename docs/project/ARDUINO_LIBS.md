# Arduino Libraries for Zowi

This document describes every Arduino library in the Zowi ecosystem and how to use them in your own sketches.

## Table of Contents

- [Installation](#installation)
- [Library Overview](#library-overview)
- [1. Zowi — Main Robot Library](#1-zowi--main-robot-library)
- [2. Oscillator — Servo Motion Engine](#2-oscillator--servo-motion-engine)
- [3. LedMatrix — LED Mouth Display](#3-ledmatrix--led-mouth-display)
- [4. US — Ultrasonic Distance Sensor](#4-us--ultrasonic-distance-sensor)
- [5. BatReader — Battery Monitor](#5-batreader--battery-monitor)
- [6. ZowiSerialCommand — Serial Command Parser](#6-zowiserialcommand--serial-command-parser)
- [7. EnableInterrupt — Pin Interrupts](#7-enableinterrupt--pin-interrupts)
- [Typical Sketch Structure](#typical-sketch-structure)

---

## Installation

Copy each library folder into your Arduino libraries directory:

- **Windows:** `My Documents\Arduino\libraries\`
- **macOS:** `Documents/Arduino/libraries/`
- **Linux:** `libraries/` inside your sketchbook folder

After copying, restart the Arduino IDE. You should see them under **Sketch → Include Library**.

---

## Library Overview

| Library | Role | Dependency of |
|---|---|---|
| **Zowi** | Top-level robot API (motion, gestures, sounds, mouth, sensors) | — |
| **Oscillator** | Sinusoidal servo motion engine | Zowi |
| **LedMatrix** | 5x6 LED matrix via 74HC595 shift registers | Zowi |
| **US** | HC-SR04 ultrasonic distance sensor | Zowi |
| **BatReader** | Battery voltage/percentage reader | Zowi |
| **ZowiSerialCommand** | Serial command parser (standalone) | — |
| **EnableInterrupt** | Pin-change/external interrupts (standalone) | — |

---

## 1. Zowi — Main Robot Library

This is the top-level library. It depends on `Oscillator`, `LedMatrix`, `US`, `BatReader`, `Servo`, and `EEPROM`.

### Initialization

```cpp
#include <Zowi.h>

Zowi zowi;

void setup() {
    // Pin order: YL, YR, RL, RR (left/right hip, left/right foot)
    zowi.init(2, 3, 4, 5);
    // Optional: pass false to skip loading EEPROM calibration
    // zowi.init(2, 3, 4, 5, false);
}
```

### Motion

All motion methods accept `steps` (how many repetitions), `T` (period in ms), and direction/height where applicable.

```cpp
zowi.walk(4, 1000, FORWARD);       // Walk 4 steps forward
zowi.turn(4, 2000, LEFT);          // Turn left
zowi.jump(1, 2000);                // Jump once
zowi.bend(1, 1400, LEFT);          // Bend left
zowi.shakeLeg(1, 2000, RIGHT);    // Shake right leg
zowi.updown(1, 1000, 20);         // Bounce up and down
zowi.swing(1, 1000, 20);          // Swing side to side
zowi.tiptoeSwing(1, 900, 20);     // Tiptoe swing
zowi.jitter(1, 500, 20);          // Fast jitter
zowi.ascendingTurn(1, 900, 20);   // Jitter while turning
zowi.moonwalker(1, 900, 20, LEFT); // Moonwalk
zowi.crusaito(1, 900, 20, FORWARD); // Crusaito walk
zowi.flapping(1, 1000, 20, FORWARD); // Flapping
```

### Sensors

```cpp
float dist = zowi.getDistance();   // cm (999 if no echo)
int noise = zowi.getNoise();       // analog noise level
```

### Battery

```cpp
double level = zowi.getBatteryLevel();   // 0–100 %
double volts = zowi.getBatteryVoltage(); // volts
```

### LED Mouth

```cpp
#include "Zowi_mouths.h"

zowi.putMouth(smile);              // predefined expression
zowi.putMouth(heart_code, false);  // raw 30-bit pattern
zowi.putAnimationMouth(wave, 0);  // single frame of animation
zowi.clearMouth();
```

Available mouth indices: `zero`–`nine`, `smile`, `happyOpen`, `happyClosed`, `heart`, `bigSurprise`, `smallSurprise`, `tongueOut`, `vamp1`, `vamp2`, `lineMouth`, `confused`, `diagonal`, `sad`, `sadOpen`, `sadClosed`, `okMouth`, `xMouth`, `interrogation`, `thunder`, `culito`, `angry`.

### Sounds

```cpp
zowi.sing(S_connection);    // connection melody
zowi.sing(S_happy);         // happy jingle
zowi.sing(S_sad);           // sad tune
zowi.sing(S_fart1);         // (yes, really)
```

Full song list: `S_connection`, `S_disconnection`, `S_buttonPushed`, `S_mode1`–`S_mode3`, `S_surprise`, `S_OhOoh`, `S_OhOoh2`, `S_cuddly`, `S_sleeping`, `S_happy`, `S_superHappy`, `S_happy_short`, `S_sad`, `S_confused`, `S_fart1`–`S_fart3`.

### Gestures

```cpp
zowi.playGesture(ZowiHappy);       // happy dance
zowi.playGesture(ZowiSad);         // sad slump
zowi.playGesture(ZowiLove);        // heart eyes
zowi.playGesture(ZowiAngry);       // stomp
zowi.playGesture(ZowiVictory);     // victory pose
```

Full list: `ZowiHappy`, `ZowiSuperHappy`, `ZowiSad`, `ZowiSleeping`, `ZowiFart`, `ZowiConfused`, `ZowiLove`, `ZowiAngry`, `ZowiFretful`, `ZowiMagic`, `ZowiWave`, `ZowiVictory`, `ZowiFail`.

### Calibration

```cpp
zowi.setTrims(0, 0, 0, 0);        // YL, YR, RL, RR trim offsets
zowi.saveTrimsOnEEPROM();         // persist to EEPROM[0..3]
```

### Low-Level Control

```cpp
int targets[4] = {90, 90, 90, 90};
zowi._moveServos(1000, targets);  // linear move over 1s

int A[4] = {30, 30, 30, 30};      // amplitudes
int O[4] = {90, 90, 90, 90};      // offsets
int T = 2000;                      // period (ms)
double phase[4] = {0, PI/2, PI, 3*PI/2};
zowi.oscillateServos(A, O, T, phase, 2.0); // 2 cycles
```

---

## 2. Oscillator — Servo Motion Engine

Generates smooth sinusoidal motion for each servo. Zowi uses 4 instances internally (one per leg joint).

```cpp
#include <Oscillator.h>

Oscillator osc;

void setup() {
    osc.attach(9);              // servo on pin 9
    osc.SetA(45);               // amplitude ±45°
    osc.SetO(90);               // offset (center) 90°
    osc.SetT(2000);             // period 2000 ms
    osc.SetPh(0);               // initial phase 0 rad
}

void loop() {
    osc.refresh();              // call every loop — updates servo at 30ms intervals
}
```

**Methods:**

| Method | Description |
|--------|-------------|
| `attach(pin, rev=false)` | Attach servo, optionally reverse direction |
| `detach()` | Detach servo |
| `SetA(A)` | Amplitude in degrees |
| `SetO(O)` | Offset (center position) in degrees |
| `SetPh(Ph)` | Initial phase in radians |
| `SetT(T)` | Period in ms |
| `SetTrim(trim)` | Calibration trim offset |
| `getTrim()` | Current trim value |
| `SetPosition(pos)` | Immediate servo position |
| `Stop()` | Hold current position |
| `Play()` | Resume oscillation |
| `Reset()` | Reset phase to 0 |
| `refresh()` | Call in loop — computes and writes next position |

---

## 3. LedMatrix — LED Mouth Display

Controls a 5x6 LED matrix via three 74HC595 shift registers (data=11, clock=13, latch=12 by default).

```cpp
#include <LedMatrix.h>

LedMatrix matrix;                    // default pins: 11, 13, 12
// LedMatrix matrix(11, 13, 12);    // explicit SER, CLK, RCK

matrix.writeFull(0b000001000010000100001000001111); // custom pattern
matrix.setLed(1, 1);                // row 1, col 1 (1-indexed)
matrix.unsetLed(3, 4);              // turn off row 3, col 4
matrix.clearMatrix();
matrix.setEntireMatrix();

bool on = matrix.readLed(2, 3);     // check if LED is on
unsigned long val = matrix.readFull(); // read current 30-bit pattern
```

The 30-bit value maps row-by-row, left-to-right, top-to-bottom. Bit 29 = row 1, col 1; Bit 0 = row 5, col 6.

---

## 4. US — Ultrasonic Distance Sensor

Interface for HC-SR04 (or compatible) ultrasonic sensors.

```cpp
#include <US.h>

US us;
us.init(8, 9);          // trigger pin, echo pin
// or: US us(8, 9);

float cm = us.read();   // distance in cm, 999 if no echo
```

---

## 5. BatReader — Battery Monitor

Reads battery voltage from analog pin A7. Voltage range: 3.25V (0%) to 4.2V (100%).

```cpp
#include <BatReader.h>

BatReader bat;

double volts = bat.readBatVoltage();   // raw voltage
double percent = bat.readBatPercent(); // 0–100 %
```

---

## 6. ZowiSerialCommand — Serial Command Parser

Parses ASCII commands from the serial port. Commands are space-delimited and terminated with `\r`. Up to 14 commands can be registered.

```cpp
#include <ZowiSerialCommand.h>

ZowiSerialCommand sc;

void handleMove() {
    char *arg = sc.next();   // first argument
    // ...
}

void handleStop() {
    // ...
}

void handleUnknown() {
    // no matching command
}

void setup() {
    Serial.begin(9600);
    sc.addCommand("MOVE",  handleMove);
    sc.addCommand("STOP",  handleStop);
    sc.addDefaultHandler(handleUnknown);
}

void loop() {
    sc.readSerial();  // call frequently
}
```

---

## 7. EnableInterrupt — Pin Interrupts

Enables interrupts on any digital pin. Supports both external interrupts and pin-change interrupts. Works on ATmega328P (Uno), ATmega2560 (Mega), ATmega32U4 (Leonardo), and ATmega1284.

```cpp
#include <EnableInterrupt.h>

void buttonPressed() {
    // handle interrupt
}

void setup() {
    pinMode(7, INPUT_PULLUP);
    enableInterrupt(7, buttonPressed, FALLING);
    // modes: CHANGE, FALLING, RISING
}

void loop() {
    // main code
}
```

For high-speed mode (no function-call overhead in ISR):

```cpp
#define NEEDFORSPEED
#include <EnableInterrupt.h>
```

---

## Typical Sketch Structure

```cpp
#include <Zowi.h>
#include <ZowiSerialCommand.h>

Zowi zowi;
ZowiSerialCommand sc;

void handleWalk() {
    char *arg = sc.next();
    int steps = arg ? atoi(arg) : 4;
    zowi.walk(steps);
}

void handleGesture() {
    char *arg = sc.next();
    if (arg) zowi.playGesture(atoi(arg));
}

void setup() {
    Serial.begin(9600);
    zowi.init(2, 3, 4, 5);          // YL, YR, RL, RR

    sc.addCommand("WALK",    handleWalk);
    sc.addCommand("GESTURE", handleGesture);
}

void loop() {
    sc.readSerial();

    // Autonomous behavior
    if (zowi.getDistance() < 15) {
        zowi.turn(4, 1000, LEFT);
    }
}
```
