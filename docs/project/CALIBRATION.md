# Zowi Calibration

## Theoretical Operation

Calibration compensates for the mechanical misalignment of the 4 servos (servo horns not perfectly centered). Each servo receives a signed integer **trim offset** (in degrees) that is added to every commanded angle.

The workflow is:
1. Use the **`G`** command to position servos at known positions (e.g. 90°) and observe where they physically rest.
2. Compute the difference between the commanded and actual angle → that difference is the trim for that joint.
3. Send the **`C`** command with all 4 trims → they are applied immediately and saved to EEPROM.
4. On every subsequent boot, the trims are reloaded automatically.

---

## Architecture by Layer

| Layer | Files | Role |
|-------|-------|------|
| **Hardware** | bq Zum (ATmega328P), servos on pins 2,3,4,5 | 4 servos: YL(2), YR(3), RL(4), RR(5) |
| **Persistence** | EEPROM[0..3] | Stores trims as signed bytes (2's complement) |
| **Motion Engine** | `Oscillator.cpp` / `Oscillator.h` | Applies `_trim` in `SetPosition()` and `refresh()` |
| **Robot API** | `Zowi.cpp` / `Zowi.h` | `init()`, `setTrims()`, `saveTrimsOnEEPROM()` |
| **Serial Protocol** | `ZowiSerialCommand.cpp/.h` | Parses `&&CMD args%%` frames |
| **Application** | `ZOWI_BASE_v2.ino`, `factoryZowi.ino` | Registers `C`/`G` commands, orchestrates calibration |

---

## Key Files

**Core implementation:**
- `arduinolibs/Zowi/Zowi.h` / `.cpp` — Declares/implements `init()`, `setTrims()`, `saveTrimsOnEEPROM()`
- `arduinolibs/Oscillator/Oscillator.h` / `.cpp` — Where the trim is **applied**: `_servo.write(position + _trim)`
- `arduinolibs/ZowiSerialCommand/ZowiSerialCommand.cpp` — Serial command parser

**Firmware:**
- `code/base/ZOWI_BASE_v2.ino` — Main firmware, registers `C` and `G` commands
- `code/factoryZowi/factoryZowi.ino` — Factory calibration (trims to 0, verification sweep)

**Documentation:**
- `docs/project/ZOWI_API.md` (§Calibration Trims)
- `docs/project/ZOWI_BASE.md` (serial command table)
- `docs/project/FACTORY_ZOWI.md` (factory procedure)

---

## Serial Commands

| Command | Function | Example |
|---------|----------|---------|
| `C <YL> <YR> <RL> <RR>` | Sets and saves the 4 trims | `C 20 0 -8 3` |
| `G <YL> <YR> <RL> <RR>` | Positions raw servos (for observation) | `G 90 85 96 78` |

Protocol: `&&CMD args%%`, baud rate 115200, terminated by `\r`.

---

## Leg vs Foot Example

There are no separate commands — the distinction is by argument position:

| Servo | Physical role | Trim arg | EEPROM | Trim example |
|-------|---------------|----------|--------|--------------|
| **YL** | Left leg (hip) | 1st | [0] | `C 20 ...` → +20° |
| **YR** | Right leg (hip) | 2nd | [1] | `C ... 0 ...` → no adjustment |
| **RL** | Left foot | 3rd | [2] | `C ... -8 ...` → -8° |
| **RR** | Right foot | 4th | [3] | `C ... 3` → +3° |

**Leg example:** If the left leg rests tilted 20° to the left in the neutral position, send `C 20 ...`. The trim is added in `Oscillator.cpp:97` (`_servo.write(position + _trim)`).

**Foot example:** If the right foot is not flat, adjust the 4th argument: `C ... 3`. The trim applies both to absolute movements and to sinusoidal oscillation (`Oscillator.cpp:117`).

---

## EEPROM Storage

```
EEPROM[0] = TRIM_YL (left leg)     — signed byte
EEPROM[1] = TRIM_YR (right leg)    — signed byte
EEPROM[2] = TRIM_RL (left foot)    — signed byte
EEPROM[3] = TRIM_RR (right foot)   — signed byte
EEPROM[4] = free
EEPROM[5-15] = name / factory marker
```

Read in `init()` with decode: `if (val > 128) val -= 256` (range ±127°).

---

## Note

There are no Python scripts or external configuration files — it is all Arduino C/C++. The only calibration tools are the `C`/`G` serial commands.
