# ZOWI Alarm v2

A security-themed game for Zowi that turns the robot into a motion/noise alarm system and a patrolling guardian. Developed by BQ (Anita de Prado & Jose Alberca, December 2015). Released under GPL.

## Overview

Zowi Alarm has **4 modes** selected via the back buttons. The robot uses its ultrasonic sensor and noise sensor to detect intruders and trigger an audio-visual alarm.

## Modes

| Mode | Name | How to Select | Description |
|------|------|---------------|-------------|
| 0 | Idle | Default on boot | Zowi waits. After 80 seconds of inactivity, falls asleep. |
| 1 | Alarm System | Press button **A** | Zowi arms a stationary alarm. After a 10-second countdown, it monitors distance and noise. If an intruder is detected, it blasts a piercing siren. |
| 2 | Zowi Guardian | Press button **B** (or A+B) | Same alarm as Mode 1, but Zowi also performs a patrol routine every 8 seconds — looking around, showing angry expressions, and re-checking the baseline distance. |
| 4 | Teleoperation | Serial data received | Zowi listens for serial commands from the ZowiApp. |

## How to Play

1. **Power on Zowi** — it plays a connection melody, shows a "littleUuh" animation, then smiles.
2. **Press a back button** to select a mode.
3. **Mode 1 (Alarm System):** Zowi starts a 10-second countdown (beeping with a shield symbol on its mouth). After arming, it monitors distance and noise. If an intruder is detected (noise ≥ 680 or distance closer than the baseline), it blasts a loud alternating siren (A5 ↔ A7) with a full-brightness alarm symbol on its mouth. Press any button to stop.
4. **Mode 2 (Zowi Guardian):** Same alarm as Mode 1, but every 8 seconds Zowi performs a patrol routine — looking around with angry expressions and playing a cuddly-then-angry sound sequence. After patrolling, it re-calibrates the baseline distance.

## Key Functions

### `ZowiArmingAlarmSystem()`

A 10-second countdown before the alarm activates. During the countdown:
- Zowi shows an `arming_symbol` (shield shape) on its LED mouth
- Plays a short beep (`note_A7`, 50ms) every second
- Clears the mouth between beeps

After the countdown (if no button was pressed):
- Sets `alarmActivated = true`
- Records the current distance as `initDistance` (minus 10 cm for tolerance)
- Resets the patrol timer

### `ZowiGuardian()`

A patrol routine called every 8 seconds in Mode 2. Zowi:
1. Shows `smallSurprise` and plays `S_cuddly`
2. Shows an `angry` mouth with a descending tone
3. Trembles 4 times (moving to `fretfulPos` and back to home)
4. Looks left (`headLeft2`), returns to home
5. Looks right (`headRight2`), returns to home
6. Smiles and plays `S_happy_short`

After patrolling, it re-calibrates `initDistance` (current distance minus 10 cm).

## Alarm Behavior

Both Mode 1 and Mode 2 share the same alarm logic:

1. **Arming:** A 10-second countdown with a shield symbol and a beep every second.
2. **Monitoring:** Zowi continuously checks:
   - `getNoise() >= 680` (loud noise/clap)
   - `getDistance() < initDistance` (something got closer than the baseline)
3. **Trigger:** When either condition is met, Zowi enters an alarm loop:
   - Shows a full-brightness `alarm_symbol` on the LED mouth
   - Plays an alternating high-pitched siren (A5 ↔ A7, 880 Hz ↔ 3520 Hz)
   - Loops until a button is pressed

### Mode 2 — Guardian Patrol

In addition to the alarm, every 8 seconds Zowi performs a patrol routine (`ZowiGuardian`):
- Shows `smallSurprise` and plays `S_cuddly`
- Shows an `angry` mouth with descending tones
- Trembles 4 times
- Looks left, returns to home, looks right
- Smiles and plays `S_happy_short`
- Re-calibrates `initDistance` (current distance minus 10 cm)

## LED Symbols

| Symbol | Pattern | Used In |
|--------|---------|---------|
| Arming shield | `0b00111111100001100001100001111111` | Countdown phase |
| Alarm full | `0b00111111111111111111111111111111` | Intruder detected |

## Pin Mapping

| Function | Pin |
|----------|-----|
| YL servo | 2 |
| YR servo | 3 |
| RL servo | 4 |
| RR servo | 5 |
| Button A | 6 |
| Button B | 7 |
| US Trigger | 8 |
| US Echo | 9 |
| Buzzer | 10 |
| LED data (SER) | 11 |
| LED latch (RCK) | 12 |
| LED clock (CLK) | 13 |
| Noise sensor | A6 |
| Battery | A7 |

## Serial Protocol

Baud rate: **115200**. All frames use `&&` ... `%%`.

| Command | Direction | Description |
|---------|-----------|-------------|
| `S` | App → Zowi | Stop and return to home |
| `R <name>` | App → Zowi | Set Zowi's name (stored in EEPROM[5..15]) |
| `E` | App → Zowi | Request name → responds `&&E <name>%%` |
| `B` | App → Zowi | Request battery → responds `&&B <percent>%%` |
| `I` | App → Zowi | Request program ID → responds `&&I ZOWI_Alarm_v2%%` |
| `A` | Zowi → App | Acknowledge |
| `F` | Zowi → App | Final acknowledge |
