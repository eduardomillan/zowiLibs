# ZOWI Adivinawi v2

A multi-game sketch for Zowi that turns the robot into an interactive fortune-teller, dice, and game partner. Developed by BQ (Anita de Prado & Jose Alberca, December 2015). Released under GPL.

## Overview

Adivinawi is a state-machine-based game with **5 modes** selected via Zowi's back buttons. The robot listens for noise (claps or taps) to trigger actions, and communicates through its LED mouth, sounds, and body movements.

## Modes

| Mode | Name | How to Select | Description |
|------|------|---------------|-------------|
| 0 | Idle | Default on boot | Zowi waits. After 80 seconds of inactivity, falls asleep (dream animation + snoring sounds). |
| 1 | Zowi Adivino Adivinawi | Press button **A** | Fortune teller. Zowi answers yes/no questions with a magic animation, then shows `okMouth` (yes) or `xMouth` (no). |
| 2 | Zowi Dado Dice | Press button **B** | Dice roller. Zowi rolls a virtual die (1–6) and shows the result on its LED mouth. |
| 3 | Rock Paper Scissors | Press **A + B** | Play rock-paper-scissors against Zowi. The robot randomly picks one and displays it on its mouth. |
| 4 | Teleoperation | Serial data received | Zowi listens for serial commands from the ZowiApp. |

## How to Play

### Selecting a Mode

1. Press any of Zowi's back buttons to wake/interrupt it.
2. The mode is selected by which button(s) you press:
   - **Button A** → Mode 1 (Adivino Adivinawi — fortune teller)
   - **Button B** → Mode 2 (Dado Dice — dice roller)
   - **Buttons A + B** → Mode 3 (Rock Paper Scissors)
3. Zowi shows the mode number on its LED mouth and plays a confirmation sound.

### Mode 1 — Zowi Adivino Adivinawi (Fortune Teller)

Ask Zowi a yes/no question, then clap or tap near it. Zowi detects the noise (analog reading ≥ 680 on the noise sensor) and responds with:

- **Yes** — A happy dance (`swing` + `S_superHappy` melody) and an `okMouth` expression.
- **No** — An angry shake, `xMouth` expression, and a descending sad tone sequence.

### Mode 2 — Zowi Dado Dice (Dice Roller)

Clap or tap to roll a virtual 6-sided die. Zowi shows the result (1–6) on its LED mouth and plays a sound:
- **1** → sad sound
- **6** → super happy sound
- **2–5** → connection melody

### Mode 3 — Rock Paper Scissors

Clap or tap to play. Zowi randomly picks one of the three and displays it on its LED mouth:

| Choice | LED Pattern | Sound |
|--------|-------------|-------|
| Rock | `0b00000000001100011110011110001100` | `S_fart1` |
| Paper | `0b00011110010010010010010010011110` | `S_OhOoh2` |
| Scissors | `0b00000010010100001000010100000010` | `S_cuddly` |

### Mode 4 — Teleoperation (ZowiPAD)

Activated automatically when serial data is received. Zowi listens for serial commands from the ZowiApp. Buttons are disabled in this mode.

## How to Play

1. **Power on Zowi** — it plays a connection melody and shows a "littleUuh" animation, then smiles.
2. **Press a back button** to select a mode (the mode number is shown on the LED mouth).
3. **Clap or tap** near Zowi to trigger an action in the selected mode.
4. Press buttons again at any time to return to mode selection.

## Serial Protocol

Zowi communicates with the ZowiApp over Serial at 115200 baud. All messages are framed with `&&` ... `%%`.

| Command | Direction | Description |
|---------|-----------|-------------|
| `S` | App → Zowi | Stop current action |
| `R <name>` | App → Zowi | Rename Zowi (stores in EEPROM at address 5) |
| `E` | App → Zowi | Request Zowi's name |
| `B` | App → Zowi | Request battery level |
| `I` | App → Zowi | Request program ID |
| `A` | Zowi → App | Acknowledge |
| `F` | Zowi → App | Final acknowledge |

When serial data is received, Zowi automatically switches to **Mode 4** (teleoperation), disables button interrupts, and processes commands through `ZowiSerialCommand`.

## Magic Animation (`ZowiMagics`)

Before each answer in modes 1–3, Zowi plays a "magic" animation: a 6-frame mouth animation (`adivinawi`) with a rising/falling tone sweep (400 Hz → 1000 Hz → 400 Hz), repeated 1–2 times. The animation frames are:

```
Frame 0:  █   █
Frame 1:  █ █ █ █
Frame 2:   █ █ █ █
Frame 3:      █ █
Frame 4:        ██
Frame 5:  (empty)
```

## Hardware Setup

| Component | Pin |
|-----------|-----|
| YL servo | 2 |
| YR servo | 3 |
| RL servo | 4 |
| RR servo | 5 |
| Button A | 6 |
| Button B | 7 |
| Buzzer | 10 |
| US Trigger | 8 |
| US Echo | 9 |
| Noise sensor | A6 |
| Battery | A7 |

## Serial Commands

| Command | Handler | Description |
|---------|---------|-------------|
| `S` | `receiveStop` | Stops Zowi and returns to home position |
| `R <name>` | `receiveName` | Renames Zowi (stores in EEPROM at address 5) |
| `E` | `requestName` | Sends Zowi's name over serial |
| `B` | `requestBattery` | Sends battery percentage |
| `I` | `requestProgramId` | Sends the program ID string |
| (default) | `receiveStop` | Unknown commands trigger a stop |

All responses are framed as `&&<command> <value>%%`.

## Key Functions

### `ZowiMagics(repetitions)`

Plays the signature "magic" animation: a 6-frame mouth animation (`adivinawi`) with a rising tone sweep (400 Hz → 1000 Hz) and then a falling sweep (1000 Hz → 400 Hz). Interruptible by button press.

### `ZowiSleeping_withInterrupts()`

After 80 seconds of inactivity in Mode 0, Zowi leans forward and plays a 4-frame dream mouth animation with soft rising/falling tones, then shows a `lineMouth` and plays a cuddly sound.

### `ZowiLowBatteryAlarm()`

If battery is below 45%, Zowi flashes a `thunder` mouth and plays an alternating high-pitched alarm tone until a button is pressed.

## Interrupt System

Both back buttons are configured with `RISING` edge interrupts via `EnableInterrupt`:

- **Button A** (pin 6) → sets `buttonAPushed = true`
- **Button B** (pin 7) → sets `buttonBPushed = true`

Any button press sets `buttonPushed = true` and shows `smallSurprise` on the mouth. All animations and sound loops check `buttonPushed` and break early, making the system fully interruptible.

## Serial Protocol

Baud rate: **115200**. All frames use the format `&&<command> <value>%%`.

| Command | Direction | Description |
|---------|-----------|-------------|
| `S` | App → Zowi | Stop and return to home |
| `R <name>` | App → Zowi | Set Zowi's name (stored in EEPROM[5..15]) |
| `E` | App → Zowi | Request name → responds `&&E <name>%%` |
| `B` | App → Zowi | Request battery → responds `&&B <percent>%%` |
| `I` | App → Zowi | Request program ID → responds `&&I ZOWI_Adivinawi_v2%%` |
| `A` | Zowi → App | Acknowledge |
| `F` | Zowi → App | Final acknowledge |

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
