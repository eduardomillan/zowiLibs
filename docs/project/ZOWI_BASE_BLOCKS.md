# ZOWI_BASE_v2 — Block Diagram (Blockly/Scratch Style)

## ON START (setup)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ON START                                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  INIT SERIAL at 115200 baud                                           │  │
│  │  INIT Zowi (servos on pins 2,3,4,5)                                   │  │
│  │  RANDOM SEED from pin A6                                              │  │
│  │  ENABLE INTERRUPTS on buttons A (pin6) and B (pin7)                   │  │
│  │  REGISTER serial commands (S,L,T,M,H,K,C,G,R,E,D,N,B,I)               │  │
│  │  PLAY sound S_connection (connection jingle)                           │  │
│  │  WAIT 500ms                                                           │  │
│  │  HOME (servos to 90°, then DETACH servos)                             │  │
│  │  IF EEPROM[5] == '$' (factory mode) THEN                              │  │
│  │    ┌─────────────────────────────────────────────────────────┐        │  │
│  │    │  SHOW mouth "culito" (29)                                │        │  │
│  │    │  REPEAT FOREVER: WAIT (infinite loop)                    │        │  │
│  │    │  (Waits for app to write name in EEPROM)                 │        │  │
│  │    └─────────────────────────────────────────────────────────┘        │  │
│  │  SEND initial telemetry (name, program, battery)                       │  │
│  │  IF battery < 45% THEN                                                │  │
│  │    ┌─────────────────────────────────────────────────────────┐        │  │
│  │    │  REPEAT:                                                   │        │  │
│  │    │    SHOW mouth "thunder" (28)                              │        │  │
│  │    │    PLAY tone sweep 880Hz → 2000Hz                        │        │  │
│  │    │  UNTIL button pressed                                     │        │  │
│  │    └─────────────────────────────────────────────────────────┘        │  │
│  │  SHOW mouth animation "littleUuh" (2 cycles)                          │  │
│  │  IF no button was pressed THEN                                        │  │
│  │    ┌─────────────────────────────────────────────────────────┐        │  │
│  │    │  SHOW mouth "happyOpen" (11)                             │        │  │
│  │    │  PLAY sound S_happy                                       │        │  │
│  │    └─────────────────────────────────────────────────────────┘        │  │
│  │  IF EEPROM[5] == '#' (unbaptized) THEN                                │  │
│  │    ┌─────────────────────────────────────────────────────────┐        │  │
│  │    │  JUMP (1 step)                                           │        │  │
│  │    │  SHAKE LEG                                               │        │  │
│  │    │  SWING                                                   │        │  │
│  │    │  SHOW mouth "happyOpen" (11)                             │        │  │
│  │    └─────────────────────────────────────────────────────────┘        │  │
│  │  SHOW mouth "bigSmile" (10)                                           │  │
│  │  SET previousMillis = millis()                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## REPEAT FOREVER (loop)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  REPEAT FOREVER                                                               │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  IF serial data available AND mode ≠ 4 THEN                             │  │
│  │    ┌─────────────────────────────────────────────────────────────┐       │  │
│  │    │  SET mode to 4 (teleoperation)                              │       │  │
│  │    │  SHOW mouth "happyOpen"                                     │       │  │
│  │    │  DISABLE button interrupts                                   │       │  │
│  │    │  buttonPushed = FALSE                                       │       │  │
│  │    └─────────────────────────────────────────────────────────────┘       │  │
│  │                                                                          │  │
│  │  IF buttonPushed == TRUE THEN                                           │  │
│  │    ┌─────────────────────────────────────────────────────────────┐       │  │
│  │    │  HOME (rest)                                                │       │  │
│  │    │  WAIT 100ms                                                 │       │  │
│  │    │  PLAY sound S_buttonPushed                                  │       │  │
│  │    │  WAIT 200ms                                                 │       │  │
│  │    │  │                                                          │       │  │
│  │    │  ├─ IF buttonA AND NOT buttonB: MODE = 1 (dance)            │       │  │
│  │    │  │  PLAY sound S_mode1                                      │       │  │
│  │    │  ├─ IF buttonB AND NOT buttonA: MODE = 2 (obstacle)         │       │  │
│  │    │  │  PLAY sound S_mode2                                      │       │  │
│  │    │  └─ IF buttonA AND buttonB: MODE = 3 (noise)                │       │  │
│  │    │     PLAY sound S_mode3                                      │       │  │
│  │    │                                                              │       │  │
│  │    │  SHOW mode number on mouth (2 seconds)                       │       │  │
│  │    │  SHOW mouth "happyOpen"                                     │       │  │
│  │    │  buttonPushed = FALSE, buttonAPushed = FALSE,                │       │  │
│  │    │    buttonBPushed = FALSE                                     │       │  │
│  │    └─────────────────────────────────────────────────────────────┘       │  │
│  │                                                                          │  │
│  │  ELSE (no button pressed)                                                │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │  │
│  │  │  SWITCH MODE (0, 1, 2, 3, 4):                                       │ │  │
│  │  │                                                                     │ │  │
│  │  │  ┌─ MODE 0 (IDLE) ─────────────────────────────────────────────┐    │ │  │
│  │  │  │  IF millis() - previousMillis >= 80000 (80s) THEN           │    │ │  │
│  │  │  │    ┌───────────────────────────────────────────────────┐     │    │ │  │
│  │  │  │    │  SLEEP_with_interrupts():                         │     │    │ │  │
│  │  │  │    │    ┌────────────────────────────────────────┐     │     │    │ │  │
│  │  │  │    │    │  MOVE servos: YL=100, YR=80, RL=60,    │     │     │    │ │  │
│  │  │  │    │    │    RR=120 (lean forward)                │     │     │    │ │  │
│  │  │  │    │    │  REPEAT 4 times:                       │     │     │    │ │  │
│  │  │  │    │    │    SHOW dreamMouth frame[i]             │     │     │    │ │  │
│  │  │  │    │    │    PLAY soft tone                      │     │     │    │ │  │
│  │  │  │    │    │    WAIT                                │     │     │    │ │  │
│  │  │  │    │    │  SHOW mouth "lineMouth"                │     │     │    │ │  │
│  │  │  │    │    │  PLAY sound S_cuddly                   │     │     │    │ │  │
│  │  │  │    │    └────────────────────────────────────────┘     │     │    │ │  │
│  │  │  │    └───────────────────────────────────────────────────┘     │    │ │  │
│  │  │  └──────────────────────────────────────────────────────────────┘    │ │  │
│  │  │                                                                     │ │  │
│  │  │  ┌─ MODE 1 (DANCE) ──────────────────────────────────────────┐     │ │  │
│  │  │  │  randomSteps = random 5 to 20                              │     │ │  │
│  │  │  │  randomReps = random 3 to 5                               │     │ │  │
│  │  │  │  REPEAT randomReps times:                                 │     │ │  │
│  │  │  │    EXECUTE movement randomSteps (period 1000ms)            │     │ │  │
│  │  │  │    WAIT 500ms                                             │     │ │  │
│  │  │  │    IF buttonPushed == TRUE: BREAK                         │     │ │  │
│  │  │  │  SHOW mouth "happyOpen"                                   │     │ │  │
│  │  │  │  WAIT 2000ms                                              │     │ │  │
│  │  │  └───────────────────────────────────────────────────────────┘     │ │  │
│  │  │                                                                     │ │  │
│  │  │  ┌─ MODE 2 (OBSTACLE AVOIDANCE) ─────────────────────────────┐     │ │  │
│  │  │  │  distance = READ ultrasonic distance                       │     │ │  │
│  │  │  │  IF distance < 15cm THEN                                  │     │ │  │
│  │  │  │    ┌─────────────────────────────────────────────────┐     │     │ │  │
│  │  │  │    │  WALK BACKWARD (1 step, 2000ms)                  │     │     │ │  │
│  │  │  │    │  IF random(0,1) == 0 THEN:                      │     │     │ │  │
│  │  │  │    │    TURN LEFT (1 step, 2000ms)                    │     │     │ │  │
│  │  │  │    │  ELSE:                                          │     │     │ │  │
│  │  │  │    │    TURN RIGHT (1 step, 2000ms)                   │     │     │ │  │
│  │  │  │    └─────────────────────────────────────────────────┘     │     │ │  │
│  │  │  │  ELSE:                                                    │     │ │  │
│  │  │  │    WALK FORWARD (1 step, 2000ms)                          │     │ │  │
│  │  │  └───────────────────────────────────────────────────────────┘     │ │  │
│  │  │                                                                     │ │  │
│  │  │  ┌─ MODE 3 (NOISE DETECTION) ───────────────────────────────┐     │ │  │
│  │  │  │  noise = READ noise sensor (pin A6)                      │     │ │  │
│  │  │  │  IF noise > 650 THEN                                     │     │ │  │
│  │  │  │    ┌─────────────────────────────────────────────────┐     │     │ │  │
│  │  │  │    │  GESTURE based on noise value:                   │     │     │ │  │
│  │  │  │    │  ┌─ noise < 700: GESTURE "happy" (0)            │     │     │ │  │
│  │  │  │    │  ├─ noise < 800: GESTURE "surprise" (6)         │     │     │ │  │
│  │  │  │    │  └─ noise >= 800: GESTURE "confused" (5)        │     │     │ │  │
│  │  │  │    └─────────────────────────────────────────────────┘     │     │ │  │
│  │  │  │  ELSE:                                                    │     │ │  │
│  │  │  │    SHOW mouth "bigSmile" (10)                             │     │ │  │
│  │  │  └───────────────────────────────────────────────────────────┘     │ │  │
│  │  │                                                                     │ │  │
│  │  │  ┌─ MODE 4 (TELEOPERATION) ──────────────────────────────────┐     │ │  │
│  │  │  │  READ serial command (SCmd.readSerial())                   │     │ │  │
│  │  │  │  IF Zowi is NOT resting THEN                              │     │ │  │
│  │  │  │    EXECUTE movement according to global moveId             │     │ │  │
│  │  │  │    (see MOVE block)                                       │     │ │  │
│  │  │  └───────────────────────────────────────────────────────────┘     │ │  │
│  │  │                                                                     │ │  │
│  │  │  IF any movement was executed:                                      │ │  │
│  │  │    previousMillis = millis() (reset sleep timer)                    │ │  │
│  │  └─────────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## ON BUTTON PRESSED (interrupt)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ON PRESS button A (pin 6, rising edge)                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  buttonAPushed = TRUE                                                  │  │
│  │  buttonPushed = TRUE                                                   │  │
│  │  SHOW mouth "smallSurprise" (27)                                       │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│  ON PRESS button B (pin 7, rising edge)                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  buttonBPushed = TRUE                                                  │  │
│  │  buttonPushed = TRUE                                                   │  │
│  │  SHOW mouth "smallSurprise" (27)                                       │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## MOVE Zowi

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  MOVE Zowi (movement ID, period T, size moveSize)                            │
│                                                                              │
│  SWITCH ID:                                                                  │
│  ┌──────┬──────────────────────────────────────┬──────────────────────────┐  │
│  │ ID   │  BLOCK                               │  Zowi METHOD             │  │
│  ├──────┼──────────────────────────────────────┼──────────────────────────┤  │
│  │  0   │  HOME / STOP                         │  home()                  │  │
│  │  1   │  WALK FORWARD 1 step                 │  walk(1, T, 1)          │  │
│  │  2   │  WALK BACKWARD 1 step                │  walk(1, T, -1)         │  │
│  │  3   │  TURN LEFT 1 step                    │  turn(1, T, 1)          │  │
│  │  4   │  TURN RIGHT 1 step                   │  turn(1, T, -1)         │  │
│  │  5   │  UPDOWN                              │  updown(1, T, moveSize) │  │
│  │  6   │  MOONWALKER LEFT                     │  moonwalker(1,T,ms,1)   │  │
│  │  7   │  MOONWALKER RIGHT                    │  moonwalker(1,T,ms,-1)  │  │
│  │  8   │  SWING                               │  swing(1, T, moveSize)  │  │
│  │  9   │  CRUSAITO FORWARD                    │  crusaito(1,T,ms,1)     │  │
│  │ 10   │  CRUSAITO BACKWARD                   │  crusaito(1,T,ms,-1)    │  │
│  │ 11   │  JUMP                                │  jump(1, T)             │  │
│  │ 12   │  FLAPPING FORWARD                    │  flapping(1,T,ms,1)     │  │
│  │ 13   │  FLAPPING BACKWARD                   │  flapping(1,T,ms,-1)    │  │
│  │ 14   │  TIPTOE SWING                        │  tiptoeSwing(1,T,ms)    │  │
│  │ 15   │  BEND LEFT                           │  bend(1, T, 1)          │  │
│  │ 16   │  BEND RIGHT                          │  bend(1, T, -1)         │  │
│  │ 17   │  SHAKE LEG LEFT                      │  shakeLeg(1, T, 1)      │  │
│  │ 18   │  SHAKE LEG RIGHT                     │  shakeLeg(1, T, -1)     │  │
│  │ 19   │  JITTER                              │  jitter(1, T, moveSize) │  │
│  │ 20   │  ASCENDING TURN                      │  ascendingTurn(1,T,ms)  │  │
│  │ 30+  │  MANUAL SERVO (raw positions)        │  _moveServos(200, raw)  │  │
│  └──────┴──────────────────────────────────────┴──────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## GESTURE

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  GESTURE (gesture ID)                                                         │
│                                                                              │
│  SWITCH ID:                                                                  │
│  ┌──────┬─────────────────────────────────────────────────────────────────┐  │
│  │ ID   │  SEQUENCE                                                        │  │
│  ├──────┼─────────────────────────────────────────────────────────────────┤  │
│  │  0   │  HAPPY: sound S_happy → swing → mouth "bigSmile"                │  │
│  │  1   │  SUPER HAPPY: sound S_happy + S_superHappy → tiptoe swing       │  │
│  │  2   │  SAD: sad position → descending tones → sad mouth               │  │
│  │  3   │  SLEEPING: bed position → dream mouth animation → S_sleeping     │  │
│  │  4   │  FART: 3 positions → 3 fart sounds → tongueOut mouth            │  │
│  │  5   │  CONFUSED: confused position → S_confused → confused mouth       │  │
│  │  6   │  LOVE: heart mouth → S_cuddly → crusaito                        │  │
│  │  7   │  ANGRY: position → angry face → S_confused → jitter             │  │
│  │  8   │  FRETFUL: angry face → fretful oscillation                      │  │
│  │  9   │  MAGIC: 4 adivinawi cycles + ascending/descending tones         │  │
│  │ 10   │  WAVE: 2 wave cycles + tone sweep                               │  │
│  │ 11   │  VICTORY: legs up + tones → tiptoe swing → S_happy             │  │
│  │ 12   │  FAIL: progressive bend + descending tones →                     │  │
│  │      │    detach servos → long low tone                                │  │
│  └──────┴─────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## PLAY SOUND / SONG

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PLAY song (ID)                                                              │
│                                                                              │
│  ┌────────┬──────────────────────────────────────────┐                      │
│  │ ID     │  SONG                                    │                      │
│  ├────────┼──────────────────────────────────────────┤                      │
│  │  0     │  Connection                              │                      │
│  │  1     │  Disconnection                           │                      │
│  │  2     │  Button pushed                           │                      │
│  │  3     │  Mode 1 (dance)                          │                      │
│  │  4     │  Mode 2 (obstacle)                       │                      │
│  │  5     │  Mode 3 (noise)                          │                      │
│  │  6     │  Surprise (OhOoh)                        │                      │
│  │  7     │  OhOoh (alternate)                       │                      │
│  │  8     │  Cuddly                                  │                      │
│  │  9     │  Sleeping                                │                      │
│  │ 10     │  Happy                                   │                      │
│  │ 11     │  Super happy                             │                      │
│  │ 12     │  Happy (short)                           │                      │
│  │ 13     │  Sad                                     │                      │
│  │ 14     │  Confused                                │                      │
│  │ 15     │  Fart 1                                 │                      │
│  │ 16     │  Fart 2                                 │                      │
│  │ 17     │  Fart 3                                 │                      │
│  │ 18     │  Farts (long)                            │                      │
│  └────────┴──────────────────────────────────────────┘                      │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────────────┐     │
│   │  TONE (freq_Hz, duration_ms)  →  tone(pinBuzzer, freq, dur)       │     │
│   │  SWEEP (start_Hz, end_Hz, step, duration_ms)                      │     │
│   │    (multiplies/divides frequency by step each iteration)           │     │
│   └────────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## SHOW MOUTH (LED Matrix 5×6)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SHOW mouth (shape)                                                          │
│                                                                              │
│  Predefined (ID → 5×6 matrix pattern, 30 bits):                             │
│  ┌────────┬─────────────────────┬────────┬─────────────────────────┐        │
│  │ ID     │  SHAPE              │ ID     │  SHAPE                  │        │
│  ├────────┼─────────────────────┼────────┼─────────────────────────┤        │
│  │  0—9   │  Digits 0—9        │  10    │  bigSmile               │        │
│  │ 11     │  happyOpen          │  12    │  bigHappyOpen           │        │
│  │ 13     │  heartMouth         │  14    │  bigHeart               │        │
│  │ 15     │  littleUuh          │  16    │  dogMouth               │        │
│  │ 17     │  tigerMouth         │  18    │  catMouth               │        │
│  │ 19     │  tongueOut          │  20    │  sceptic                │        │
│  │ 21     │  sadMouth           │  22    │  confMouth              │        │
│  │ 23     │  lineMouth          │  24    │  smallMouth             │        │
│  │ 25     │  bigSurprise        │  26    │  medSurprise            │        │
│  │ 27     │  smallSurprise      │  28    │  thunder                │        │
│  │ 29     │  culito             │  30    │  openMouth              │        │
│  └────────┴─────────────────────┴────────┴─────────────────────────┘        │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐   │
│   │  ANIMATE mouth (animation, frame_index)                               │   │
│   │  ┌──────────────┬──────────────────────────────────────────────┐     │   │
│   │  │  Animation   │  Frames                                        │     │   │
│   │  ├──────────────┼──────────────────────────────────────────────┤     │   │
│   │  │  littleUuh   │  8 frames (small blinking mouth)             │     │   │
│   │  │  dreamMouth  │  4 frames (dreamy mouth, closed eyes)        │     │   │
│   │  │  adivinawi   │  6 frames (guessing mouth)                   │     │   │
│   │  │  wave        │  10 frames (wave)                            │     │   │
│   │  └──────────────┴──────────────────────────────────────────────┘     │   │
│   └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐   │
│   │  RAW MOUTH (30-bit binary) → writes pattern directly                 │   │
│   └──────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## READ SENSORS

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  READ distance (cm)                                                         │
│    triggerPin = 8, echoPin = 9                                              │
│    trigger 10µs → measure echo pulse width → cm = microseconds / 29 / 2    │
│  └────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  READ noise (value 0—1023)                                                  │
│    pin = A6, average of 2 readings                                         │
│  └────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  READ battery (% , 0—100)                                                   │
│    pin = A7, average of 10 readings                                         │
│    linear map: 3.25V → 0%, 4.2V → 100%                                      │
│  └────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  READ battery voltage (V)                                                   │
│    pin = A7, average of 10 readings → scale to volts                       │
│  └────────────────────────────────────────────────────────────────────────  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## SERIAL COMMANDS (app ↔ Zowi protocol)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ON RECEIVE serial command (only in MODE 4 — teleoperation)                  │
│                                                                              │
│  Format: &&<command_letter> [arg1] [arg2]...%%                              │
│  Baud rate: 115200                                                            │
│                                                                              │
│  ┌──────┬────────────────────────────────┬────────────────────────────────┐  │
│  │ CMD  │  BLOCK                        │  RESPONSE                      │  │
│  ├──────┼────────────────────────────────┼────────────────────────────────┤  │
│  │  S   │  STOP Zowi                    │  → &&A%%  (home)  → &&F%%      │  │
│  │  L   │  MOUTH <30bits>               │  → &&A%%  → &&F%%              │  │
│  │  T   │  TONE <freq_Hz> <dur_ms>      │  → &&A%%  → &&F%%              │  │
│  │  M   │  MOVE <ID> <T> <size>         │  → &&A%%  → (move) → &&F%%    │  │
│  │  H   │  GESTURE <ID> (0-12)          │  → &&A%%  → &&F%%              │  │
│  │  K   │  SONG <ID> (0-18)             │  → &&A%%  → &&F%%              │  │
│  │  C   │  CALIBRATE <YL> <YR> <RL> <RR>│  → &&A%%  → (EEPROM) → &&F%%  │  │
│  │  G   │  SERVO <YL> <YR> <RL> <RR>    │  → &&A%%  → (move) → &&F%%    │  │
│  │  R   │  NAME <text>                  │  → &&A%%  → (EEPROM) → &&F%%  │  │
│  │  E   │  GET name                     │  → &&E <name>%%                │  │
│  │  D   │  GET distance                 │  → &&D <cm>%%                  │  │
│  │  N   │  GET noise                    │  → &&N <value>%%               │  │
│  │  B   │  GET battery                  │  → &&B <%>%%                   │  │
│  │  I   │  GET program ID               │  → &&I ZOWI_BASE_v2%%          │  │
│  └──────┴────────────────────────────────┴────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## SERVO CONFIGURATION

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ASSIGN servo pins                                                           │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  PIN_YL (left leg)   =  2   →  Servo YL                                │  │
│  │  PIN_YR (right leg)  =  3   →  Servo YR                                │  │
│  │  PIN_RL (left foot)  =  4   →  Servo RL                                │  │
│  │  PIN_RR (right foot) =  5   →  Servo RR                                │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  FUNCTIONS:                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  ATTACH servos()     → attach() each servo to its pin                  │  │
│  │  DETACH servos()     → detach() each servo                             │  │
│  │  HOME()              → move all to 90° (500ms), then detach            │  │
│  │  CALIBRATE(YL,YR,RL,RR) → update trims and save to EEPROM[0..3]       │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## EEPROM (persistent memory)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  EEPROM MEMORY MAP                                                           │
│  ┌──────────┬──────────┬──────────────────────────────────────────────────┐  │
│  │  Bytes   │  Usage   │  Values                                          │  │
│  ├──────────┼──────────┼──────────────────────────────────────────────────┤  │
│  │  0—3     │  Trims   │  YL, YR, RL, RR (signed byte, calibration)       │  │
│  │  4       │  (free)  │                                                   │  │
│  │  5       │  State   │  '$' = factory (infinite loop on boot)           │  │
│  │          │          │  '#' = unbaptized (extended greeting)             │  │
│  │          │          │  other = active name                              │  │
│  │  5—15    │  Name    │  Up to 10 chars + '\0'                           │  │
│  └──────────┴──────────┴──────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```
