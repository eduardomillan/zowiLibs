# Zowi Library — Complete API Reference

Full reference for the `Zowi` class, including every public method, its parameters, default values, return types, and behavior.

## Contents

- [Initialization](#initialization)
- [Attach & Detach](#attach--detach)
- [Calibration Trims](#calibration-trims)
- [Low-Level Motion](#low-level-motion)
- [Home & Rest State](#home--rest-state)
- [High-Level Movements](#high-level-movements)
- [Sensors](#sensors)
- [Battery](#battery)
- [LED Mouth](#led-mouth)
- [Sounds](#sounds)
- [Gestures](#gestures)
- [Constants](#constants)
- [Mouth Indices](#mouth-indices)
- [Song IDs](#song-ids)
- [Gesture IDs](#gesture-ids)
- [Animation IDs](#animation-ids)

---

## Initialization

### `init()`

```cpp
void init(
    int YL,
    int YR,
    int RL,
    int RR,
    bool load_calibration = true,
    int NoiseSensor      = PIN_NoiseSensor,
    int Buzzer           = PIN_Buzzer,
    int USTrigger        = PIN_Trigger,
    int USEcho           = PIN_Echo
);
```

Main initialization. Configures pins, attaches servos, optionally loads trim calibration from EEPROM, initializes the ultrasonic sensor, buzzer, and noise sensor.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `YL` | `int` | — | Servo pin: left hip |
| `YR` | `int` | — | Servo pin: right hip |
| `RL` | `int` | — | Servo pin: left foot |
| `RR` | `int` | — | Servo pin: right foot |
| `load_calibration` | `bool` | `true` | If `true`, reads trim values from EEPROM[0..3] and applies them |
| `NoiseSensor` | `int` | `A6` | Analog pin for the noise sensor |
| `Buzzer` | `int` | `10` | Digital pin for the buzzer |
| `USTrigger` | `int` | `8` | Digital pin: ultrasonic sensor trigger |
| `USEcho` | `int` | `9` | Digital pin: ultrasonic sensor echo |

---

## Attach & Detach

### `attachServos()`

```cpp
void attachServos();
```

Attaches all four servos to their configured pins. Called automatically by `init()` and `_moveServos()`.

### `detachServos()`

```cpp
void detachServos();
```

Detaches all four servos, freeing them for PWM use. Called automatically by `home()`.

---

## Calibration Trims

### `setTrims()`

```cpp
void setTrims(int YL, int YR, int RL, int RR);
```

Sets the oscillator trim offset for each servo. Used to compensate for mechanical misalignment.

| Parameter | Type | Description |
|-----------|------|-------------|
| `YL` | `int` | Trim offset for left hip servo |
| `YR` | `int` | Trim offset for right hip servo |
| `RL` | `int` | Trim offset for left foot servo |
| `RR` | `int` | Trim offset for right foot servo |

### `saveTrimsOnEEPROM()`

```cpp
void saveTrimsOnEEPROM();
```

Saves the current trim values to EEPROM addresses 0–3. These values are loaded automatically on the next boot if `load_calibration=true`.

---

## Low-Level Motion

### `_moveServos()`

```cpp
void _moveServos(int time, int servo_target[]);
```

Moves the 4 servos linearly from their current positions to the target positions over `time` milliseconds. If `time <= 10`, the servos jump instantly (no interpolation).

| Parameter | Type | Description |
|-----------|------|-------------|
| `time` | `int` | Movement duration in milliseconds |
| `servo_target` | `int[4]` | Target positions: `{YL, YR, RL, RR}` |

### `oscillateServos()`

```cpp
void oscillateServos(int A[4], int O[4], int T, double phase_diff[4], float cycle = 1);
```

Executes sinusoidal oscillation on all 4 servos for `cycle` number of periods. This is the core engine for all high-level movements.

| Parameter | Type | Description |
|-----------|------|-------------|
| `A` | `int[4]` | Amplitude for each servo (degrees) |
| `O` | `int[4]` | Offset (center position) for each servo |
| `T` | `int` | Period in milliseconds |
| `phase_diff` | `double[4]` | Initial phase for each servo (radians) |
| `cycle` | `float` | Number of oscillation periods (default: `1`) |

---

## Home & Rest State

### `home()`

```cpp
void home();
```

Moves all servos to 90° (rest position) over 500 ms, then detaches them. Sets the rest state flag to `true`. No-op if already resting.

### `getRestState()`

```cpp
bool getRestState();
```

Returns `true` if Zowi is currently in the rest (home) position.

### `setRestState()`

```cpp
void setRestState(bool state);
```

Manually sets the rest state flag. Use this if you need to override the internal state tracking.

| Parameter | Type | Description |
|-----------|------|-------------|
| `state` | `bool` | `true` = resting, `false` = moving |

---

## High-Level Movements

### `jump()`

```cpp
void jump(float steps = 1, int T = 2000);
```

Jump: moves feet up then back to home.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of jumps |
| `T` | `int` | `2000` | Period in ms |

### `walk()`

```cpp
void walk(float steps = 4, int T = 1000, int dir = FORWARD);
```

Walk forward or backward. Hip servos are in phase, feet servos are 90° out of phase (direction-dependent).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `4` | Number of steps |
| `T` | `int` | `1000` | Period in ms |
| `dir` | `int` | `FORWARD` | `FORWARD` (1) or `BACKWARD` (-1) |

### `turn()`

```cpp
void turn(float steps = 4, int T = 2000, int dir = LEFT);
```

Turn left or right. Uses asymmetric hip amplitudes to create an arc.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `4` | Number of steps |
| `T` | `int` | `2000` | Period in ms |
| `dir` | `int` | `LEFT` | `LEFT` (1) or `RIGHT` (-1) |

### `bend()`

```cpp
void bend(int steps = 1, int T = 1400, int dir = LEFT);
```

Lateral bend left or right. The internal T2 is fixed at 800 ms to prevent falls.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `int` | `1` | Number of bends |
| `T` | `int` | `1400` | Period in ms |
| `dir` | `int` | `LEFT` | `LEFT` (1) or `RIGHT` (-1) |

### `shakeLeg()`

```cpp
void shakeLeg(int steps = 1, int T = 2000, int dir = RIGHT);
```

Shake a leg (right or left). Bends first, then shakes twice. Internal T2 is 1000 ms.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `int` | `1` | Number of shake cycles |
| `T` | `int` | `2000` | Period in ms |
| `dir` | `int` | `RIGHT` | `RIGHT` (-1) or `LEFT` (1) |

### `updown()`

```cpp
void updown(float steps = 1, int T = 1000, int h = 20);
```

Bouncing up-and-down. Both feet are 180° out of phase.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of bounces |
| `T` | `int` | `1000` | Period in ms |
| `h` | `int` | `20` | Amplitude (height). Use `SMALL` (5), `MEDIUM` (15), `BIG` (30), or any int 0–90 |

### `swing()`

```cpp
void swing(float steps = 1, int T = 1000, int h = 20);
```

Side-to-side swinging. Both feet are in phase with an offset of h/2.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of swings |
| `T` | `int` | `1000` | Period in ms |
| `h` | `int` | `20` | Swing amplitude (0–50 approx) |

### `tiptoeSwing()`

```cpp
void tiptoeSwing(float steps = 1, int T = 900, int h = 20);
```

Swinging on tiptoes without touching the floor with the heel.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of swings |
| `T` | `int` | `900` | Period in ms |
| `h` | `int` | `20` | Amplitude (0–50 approx) |

### `jitter()`

```cpp
void jitter(float steps = 1, int T = 500, int h = 20);
```

Fast jitter motion using hip servos. Height is clamped to max 25.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of jitters |
| `T` | `int` | `500` | Period in ms |
| `h` | `int` | `20` | Height (5–25, clamped) |

### `ascendingTurn()`

```cpp
void ascendingTurn(float steps = 1, int T = 900, int h = 20);
```

Jitter while ascending. All four servos are 180° out of phase in pairs. Height is clamped to max 13.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of turns |
| `T` | `int` | `900` | Period in ms |
| `h` | `int` | `20` | Height (5–15, clamped) |

### `moonwalker()`

```cpp
void moonwalker(float steps = 1, int T = 900, int h = 20, int dir = LEFT);
```

Moonwalk — a travelling wave from one side to the other, similar to caterpillar locomotion. Uses a 60° phase shift between feet.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of steps |
| `T` | `int` | `900` | Period in ms |
| `h` | `int` | `20` | Height (15–40 typical) |
| `dir` | `int` | `LEFT` | `LEFT` (1) or `RIGHT` (-1) |

### `crusaito()`

```cpp
void crusaito(float steps = 1, int T = 900, int h = 20, int dir = FORWARD);
```

A mixture between moonwalker and walk. Uses 25° hip amplitude.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of steps |
| `T` | `int` | `900` | Period in ms |
| `h` | `int` | `20` | Height (20–50) |
| `dir` | `int` | `FORWARD` | `FORWARD` (1) or `BACKWARD` (-1) |

### `flapping()`

```cpp
void flapping(float steps = 1, int T = 1000, int h = 20, int dir = FORWARD);
```

Flapping motion. Hip servos are 180° out of phase; feet are 90° out of phase (direction-dependent).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `steps` | `float` | `1` | Number of flaps |
| `T` | `int` | `1000` | Period in ms |
| `h` | `int` | `20` | Height (10–30) |
| `dir` | `int` | `FORWARD` | `FORWARD` (1) or `BACKWARD` (-1) |

---

## Sensors

### `getDistance()`

```cpp
float getDistance();
```

Returns the distance in centimeters from the ultrasonic sensor (HC-SR04). Returns `999` if no echo is received.

### `getNoise()`

```cpp
int getNoise();
```

Returns an averaged noise level from the analog noise sensor (default pin A6). Takes 2 readings with a 4 ms delay between them.

---

## Battery

### `getBatteryLevel()`

```cpp
double getBatteryLevel();
```

Returns battery percentage (0–100%). Averages 10 readings. The first read is discarded (often inaccurate).

### `getBatteryVoltage()`

```cpp
double getBatteryVoltage();
```

Returns battery voltage in volts. Averages 10 readings. Range: 3.25V (0%) to 4.2V (100%).

---

## LED Mouth

### `putMouth()`

```cpp
void putMouth(unsigned long int mouth, bool predefined = true);
```

Displays a mouth shape on the 5×6 LED matrix.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `mouth` | `unsigned long int` | — | If `predefined=true`: index of a predefined mouth (0–30). If `predefined=false`: raw 30-bit pattern |
| `predefined` | `bool` | `true` | Whether to look up the mouth by index or use the raw value |

### `putAnimationMouth()`

```cpp
void putAnimationMouth(unsigned long int anim, int index);
```

Displays a single frame of a mouth animation on the LED matrix.

| Parameter | Type | Description |
|-----------|------|-------------|
| `anim` | `unsigned long int` | Animation ID: `littleUuh` (0), `dreamMouth` (1), `adivinawi` (2), `wave` (3) |
| `index` | `int` | Frame index (0-based). `littleUuh` has 8 frames, `dreamMouth` has 4, `adivinawi` has 6, `wave` has 10 |

### `clearMouth()`

```cpp
void clearMouth();
```

Clears all LEDs on the mouth matrix.

---

## Sounds

### `_tone()`

```cpp
void _tone(float noteFrequency, long noteDuration, int silentDuration);
```

Plays a single tone on the buzzer, followed by a silent pause.

| Parameter | Type | Description |
|-----------|------|-------------|
| `noteFrequency` | `float` | Frequency in Hz |
| `noteDuration` | `long` | Duration of the note in ms |
| `silentDuration` | `int` | Duration of silence after the note in ms. If 0, treated as 1 |

### `bendTones()`

```cpp
void bendTones(float initFrequency, float finalFrequency, float prop, long noteDuration, int silentDuration);
```

Plays a frequency sweep from `initFrequency` to `finalFrequency` (or vice versa). Each step multiplies (or divides) the frequency by `prop`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `initFrequency` | `float` | Starting frequency in Hz |
| `finalFrequency` | `float` | Ending frequency in Hz |
| `prop` | `float` | Frequency multiplier per step (e.g., `1.02` = 2% increase). Typical values: `1.01–1.05` |
| `noteDuration` | `long` | Duration of each tone step in ms |
| `silentDuration` | `int` | Silent pause between steps in ms. If 0, treated as 1 |

### `sing()`

```cpp
void sing(int songName);
```

Plays a predefined song. Uses `_tone()` and `bendTones()` internally.

| Parameter | Type | Description |
|-----------|------|-------------|
| `songName` | `int` | Song ID (0–18). See [Song IDs](#song-ids) |

---

## Gestures

### `playGesture()`

```cpp
void playGesture(int gesture);
```

Executes a complex gesture that combines body movement, sounds, and mouth animations. Each gesture is a self-contained sequence that returns Zowi to the home position with `happyOpen` mouth when finished.

| Parameter | Type | Description |
|-----------|------|-------------|
| `gesture` | `int` | Gesture ID (0–12). See [Gesture IDs](#gesture-ids) |

---

## Constants

Defined in `Zowi.h`:

| Constant | Value | Description |
|----------|-------|-------------|
| `FORWARD` | `1` | Forward direction |
| `BACKWARD` | `-1` | Backward direction |
| `LEFT` | `1` | Left direction |
| `RIGHT` | `-1` | Right direction |
| `SMALL` | `5` | Small amplitude (for `updown`) |
| `MEDIUM` | `15` | Medium amplitude (for `updown`) |
| `BIG` | `30` | Large amplitude (for `updown`) |
| `PIN_Buzzer` | `10` | Default buzzer pin |
| `PIN_Trigger` | `8` | Default ultrasonic trigger pin |
| `PIN_Echo` | `9` | Default ultrasonic echo pin |
| `PIN_NoiseSensor` | `A6` | Default noise sensor pin |

---

## Mouth Indices

Used with `putMouth()`. Defined in `Zowi_mouths.h`.

| Index | Name | Description |
|-------|------|-------------|
| 0 | `zero` | Digit 0 |
| 1 | `one` | Digit 1 |
| 2 | `two` | Digit 2 |
| 3 | `three` | Digit 3 |
| 4 | `four` | Digit 4 |
| 5 | `five` | Digit 5 |
| 6 | `six` | Digit 6 |
| 7 | `seven` | Digit 7 |
| 8 | `eight` | Digit 8 |
| 9 | `nine` | Digit 9 |
| 10 | `smile` | Smile |
| 11 | `happyOpen` | Happy (mouth open) |
| 12 | `happyClosed` | Happy (mouth closed) |
| 13 | `heart` | Heart shape |
| 14 | `bigSurprise` | Big surprise (O mouth) |
| 15 | `smallSurprise` | Small surprise |
| 16 | `tongueOut` | Tongue out |
| 17 | `vamp1` | Vampire teeth 1 |
| 18 | `vamp2` | Vampire teeth 2 |
| 19 | `lineMouth` | Straight line mouth |
| 20 | `confused` | Confused expression |
| 21 | `diagonal` | Diagonal line |
| 22 | `sad` | Sad face |
| 23 | `sadOpen` | Sad (mouth open) |
| 24 | `sadClosed` | Sad (mouth closed) |
| 25 | `okMouth` | OK mouth |
| 26 | `xMouth` | X mouth (error/no) |
| 27 | `interrogation` | Question mark |
| 28 | `thunder` | Thunder/lightning |
| 29 | `culito` | Little butt (factory marker) |
| 30 | `angry` | Angry face |

---

## Song IDs

Used with `sing()`. Defined in `Zowi_sounds.h`.

| ID | Name | Description |
|----|------|-------------|
| 0 | `S_connection` | Connection melody (3 notes up) |
| 1 | `S_disconnection` | Disconnection melody (3 notes down) |
| 2 | `S_buttonPushed` | Button pressed (2 quick bends) |
| 3 | `S_mode1` | Mode 1 selected (bend up) |
| 4 | `S_mode2` | Mode 2 selected (bend up higher) |
| 5 | `S_mode3` | Mode 3 selected (3 quick notes) |
| 6 | `S_surprise` | Surprise (quick sweep up and down) |
| 7 | `S_OhOoh` | Oh-oh (sweep with tremolo) |
| 8 | `S_OhOoh2` | Oh-oh 2 (higher sweep with tremolo) |
| 9 | `S_cuddly` | Cuddly (bend up then down) |
| 10 | `S_sleeping` | Sleeping (low sweep up and down) |
| 11 | `S_happy` | Happy (sweep up and down) |
| 12 | `S_superHappy` | Super happy (fast high sweep) |
| 13 | `S_happy_short` | Happy short (two quick sweeps) |
| 14 | `S_sad` | Sad (slow descending bend) |
| 15 | `S_confused` | Confused (up-down-up sweep) |
| 16 | `S_fart1` | Fart 1 (quick upward sweep) |
| 17 | `S_fart2` | Fart 2 (faster higher sweep) |
| 18 | `S_fart3` | Fart 3 (up then down sweep) |

---

## Gesture IDs

Used with `playGesture()`. Defined in `Zowi_gestures.h`.

| ID | Name | Description |
|----|------|-------------|
| 0 | `ZowiHappy` | Happy dance: swing + happy sounds |
| 1 | `ZowiSuperHappy` | Super happy: tiptoe swing + high-pitched sounds |
| 2 | `ZowiSad` | Sad slump: leans forward, descending tones, sad mouths |
| 3 | `ZowiSleeping` | Sleep: leans forward, dream mouth animation, snoring sounds |
| 4 | `ZowiFart` | Fart: bends in 3 positions, tongue out, fart sounds |
| 5 | `ZowiConfused` | Confused: leans left, confused mouth, confused sound |
| 6 | `ZowiLove` | Love: heart mouth, crusaito, cuddly sound |
| 7 | `ZowiAngry` | Angry: angry mouth, descending tones, head left/right |
| 8 | `ZowiFretful` | Fretful: angry mouth, tones, trembles 4 times |
| 9 | `ZowiMagic` | Magic: 4× adivinawi animation, rising/falling tones |
| 10 | `ZowiWave` | Wave: 2× wave animation, rising/falling tones |
| 11 | `ZowiVictory` | Victory: raises feet, tiptoe swing, super happy sounds |
| 12 | `ZowiFail` | Fail: bends down progressively, sad sounds, detach servos |

---

## Animation IDs

Used with `putAnimationMouth()`. Defined in `Zowi_gestures.h`.

| ID | Name | Frames | Description |
|----|------|--------|-------------|
| 0 | `littleUuh` | 8 | Mouth opening/closing animation (surprise) |
| 1 | `dreamMouth` | 4 | Dream/mouth opening animation (sleeping) |
| 2 | `adivinawi` | 6 | Magic fortune-teller animation (dots expanding/contracting) |
| 3 | `wave` | 10 | Hand-wave animation (wave pattern moving across LED matrix) |
