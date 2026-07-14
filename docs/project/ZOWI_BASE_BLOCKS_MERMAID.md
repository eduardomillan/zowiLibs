# ZOWI_BASE_v2 — Mermaid Block Diagrams

Este archivo describe el firmware ZOWI_BASE_v2 mediante diagramas Mermaid y tablas.
Cada sección es un diagrama renderizable independiente.

---

## 1. Mode State Machine

Transiciones entre los 5 modos de operación. Los eventos son: botón A, botón B,
botón A+B simultáneo, llegada de datos seriales, y cualquier botón (volver a IDLE).

```mermaid
stateDiagram-v2
    direction LR
    state "MODE 0: IDLE" as IDLE
    state "MODE 1: DANCE" as DANCE
    state "MODE 2: OBSTACLE" as OBSTACLE
    state "MODE 3: NOISE" as NOISE
    state "MODE 4: TELEOP" as TELEOP
    
    [*] --> IDLE : power on / reset
    
    IDLE --> DANCE : button A pressed
    IDLE --> OBSTACLE : button B pressed
    IDLE --> NOISE : button A + B pressed
    IDLE --> TELEOP : serial data received
    
    DANCE --> IDLE : any button pressed
    OBSTACLE --> IDLE : any button pressed
    NOISE --> IDLE : any button pressed
    TELEOP --> IDLE : any button pressed
    
    note right of IDLE : Sleep after 80s inactivity
    note right of DANCE : Random walk/swing/jump/turn
    note right of OBSTACLE : Walk forward, avoid < 15cm
    note right of NOISE : React to claps > 650
    note right of TELEOP : App control via serial
```

---

## 2. Setup Flow

Secuencia completa de inicialización al encender o resetear.

```mermaid
flowchart TD
    A["POWER ON / RESET"] --> B["Init Serial @ 115200 baud"]
    B --> C["zowi.init(PIN_YL, PIN_YR, PIN_RL, PIN_RR, true)"]
    C --> D["randomSeed(analogRead(A6))"]
    D --> E["Enable interrupts on button A (pin6) and B (pin7)"]
    E --> F["Register 14 serial command handlers"]
    F --> G["Play S_connection sound"]
    G --> H["home() — servos to 90°, then detach"]
    
    H --> I{"EEPROM[5] == '$' ?<br/>(factory mode)"}
    I -->|"YES"| J["Show mouth 'culito' (29)"]
    J --> K["REPEAT FOREVER — wait for app to write name"]
    K --> K
    
    I -->|"NO"| L["Send initial telemetry:<br/>name, program ID, battery"]
    
    L --> M{"Battery < 45% ?"}
    M -->|"YES"| N["Low battery alarm loop:<br/>thunder mouth + tone sweep 880→2000Hz<br/>until button press"]
    N --> O
    
    M -->|"NO"| O["Show littleUuh animation (2 cycles)"]
    
    O --> P{"Any button<br/>was pressed?"}
    P -->|"NO"| Q["Show mouth happyOpen (11)"]
    Q --> R["Play S_happy sound"]
    R --> S
    
    P -->|"YES"| S
    
    S --> T{"EEPROM[5] == '#' ?<br/>(unbaptized)"}
    
    T -->|"YES"| U["Jump (1 step)"]
    U --> V["Shake leg"]
    V --> W["Swing"]
    W --> X["Show mouth happyOpen (11)"]
    X --> Y
    
    T -->|"NO"| Y["Show mouth bigSmile (10)"]
    
    Y --> Z["previousMillis = millis()"]
    Z --> END["Ready — enter loop()"]
```

---

## 3. Main Loop

Bucle principal con detección de modo y ejecución según el modo activo.

```mermaid
flowchart TD
    START["loop() begins"] --> SERIAL_CHECK{"Serial available<br/>AND mode ≠ 4 ?"}
    
    SERIAL_CHECK -->|"YES"| SERIAL_ENTER["Set MODE = 4 (TELEOP)<br/>Show happyOpen mouth<br/>Disable button interrupts<br/>buttonPushed = FALSE"]
    SERIAL_ENTER --> MAIN
    
    SERIAL_CHECK -->|"NO"| BTN_CHECK{"buttonPushed<br/>== TRUE ?"}
    
    BTN_CHECK -->|"YES"| BTN_HANDLE["home()<br/>Wait 100ms<br/>Play S_buttonPushed<br/>Wait 200ms"]
    
    BTN_HANDLE --> BTN_SELECT{"Which button?"}
    BTN_SELECT -->|"A only"| MODE1_SET["MODE = 1 (DANCE)<br/>Play S_mode1"]
    BTN_SELECT -->|"B only"| MODE2_SET["MODE = 2 (OBSTACLE)<br/>Play S_mode2"]
    BTN_SELECT -->|"A + B"| MODE3_SET["MODE = 3 (NOISE)<br/>Play S_mode3"]
    
    MODE1_SET --> BTN_CLEANUP
    MODE2_SET --> BTN_CLEANUP
    MODE3_SET --> BTN_CLEANUP
    
    BTN_CLEANUP["Show mode number on mouth (2s)<br/>Show happyOpen mouth<br/>Reset all button flags"] --> MAIN
    
    BTN_CHECK -->|"NO"| MAIN
    
    MAIN["SWITCH MODE"] --> MODE0
    MAIN --> MODE1
    MAIN --> MODE2
    MAIN --> MODE3
    MAIN --> MODE4
    
    subgraph MODE0["MODE 0: IDLE"]
        direction TB
        M0_CHECK{"millis() - previousMillis<br/>>= 80000 ?"}
        M0_CHECK -->|"NO"| M0_EXIT
        M0_CHECK -->|"YES"| M0_SLEEP["ZowiSleeping():<br/>1. Lean forward (100,80,60,120)<br/>2. 4 dreamMouth frames + soft tones<br/>3. lineMouth + S_cuddly"]
        M0_SLEEP --> M0_EXIT
        M0_EXIT["exit"]
    end
    
    subgraph MODE1["MODE 1: DANCE"]
        direction TB
        M1_STEPS["randomSteps = random(5..20)<br/>randomReps = random(3..5)"]
        M1_LOOP["REPEAT randomReps times:<br/>Execute movement randomSteps<br/>Wait 500ms<br/>Break if button pressed"]
        M1_WAIT["Show happyOpen<br/>Wait 2000ms"]
        M1_STEPS --> M1_LOOP --> M1_WAIT --> M1_EXIT
        M1_EXIT["exit"]
    end
    
    subgraph MODE2["MODE 2: OBSTACLE"]
        direction TB
        M2_DIST["distance = read ultrasonic sensor"]
        M2_DIST --> M2_CHECK{"distance < 15 cm ?"}
        M2_CHECK -->|"YES"| M2_BACK["Walk backward (1 step, 2000ms)"]
        M2_BACK --> M2_TURN{"random(0,1) ?"}
        M2_TURN -->|"0"| M2_LEFT["Turn left (1 step, 2000ms)"]
        M2_TURN -->|"1"| M2_RIGHT["Turn right (1 step, 2000ms)"]
        M2_LEFT --> M2_EXIT
        M2_RIGHT --> M2_EXIT
        M2_CHECK -->|"NO"| M2_FWD["Walk forward (1 step, 2000ms)"]
        M2_FWD --> M2_EXIT
        M2_EXIT["exit"]
    end
    
    subgraph MODE3["MODE 3: NOISE"]
        direction TB
        M3_NOISE["noise = read noise sensor (A6)"]
        M3_NOISE --> M3_CHECK{"noise > 650 ?"}
        M3_CHECK -->|"NO"| M3_SMILE["Show mouth bigSmile (10)"]
        M3_SMILE --> M3_EXIT
        M3_CHECK -->|"YES"| M3_SELECT{"Which range?"}
        M3_SELECT -->|"noise < 700"| M3_HAPPY["Gesture: HAPPY (0)"]
        M3_SELECT -->|"noise < 800"| M3_SURPRISE["Gesture: SURPRISE (6)"]
        M3_SELECT -->|"noise >= 800"| M3_CONFUSED["Gesture: CONFUSED (5)"]
        M3_HAPPY --> M3_EXIT
        M3_SURPRISE --> M3_EXIT
        M3_CONFUSED --> M3_EXIT
        M3_EXIT["exit"]
    end
    
    subgraph MODE4["MODE 4: TELEOP"]
        direction TB
        M4_READ["SCmd.readSerial()"]
        M4_READ --> M4_MOVE{"Zowi resting?"}
        M4_MOVE -->|"NO"| M4_EXEC["Execute movement<br/>according to global moveId"]
        M4_MOVE -->|"YES"| M4_EXIT
        M4_EXEC --> M4_EXIT
        M4_EXIT["exit"]
    end
    
    MODE0 --> TIMER_RESET{"Movement<br/>executed?"}
    MODE1 --> TIMER_RESET
    MODE2 --> TIMER_RESET
    MODE3 --> TIMER_RESET
    MODE4 --> TIMER_RESET
    
    TIMER_RESET -->|"YES"| RESET_TIMER["previousMillis = millis()"]
    TIMER_RESET -->|"NO"| LOOP_END
    RESET_TIMER --> LOOP_END
    
    LOOP_END["back to loop() top"]
    LOOP_END --> SERIAL_CHECK
```

---

## 4. Button Interrupts

Manejadores de interrupción por flanco de subida en los botones.

```mermaid
flowchart LR
    subgraph BTN_A["Button A (pin 6 — RISING)"]
        A1["buttonAPushed = TRUE"] --> A2["buttonPushed = TRUE"] --> A3["Show mouth smallSurprise (27)"]
    end
    
    subgraph BTN_B["Button B (pin 7 — RISING)"]
        B1["buttonBPushed = TRUE"] --> B2["buttonPushed = TRUE"] --> B3["Show mouth smallSurprise (27)"]
    end
```

---

## 5. Movement Reference

| ID | Block | Zowi Method | Parameters |
|----|-------|-------------|------------|
| 0 | HOME / STOP | `home()` | — |
| 1 | WALK FORWARD | `walk(1, T, 1)` | T = period (ms) |
| 2 | WALK BACKWARD | `walk(1, T, -1)` | T = period (ms) |
| 3 | TURN LEFT | `turn(1, T, 1)` | T = period (ms) |
| 4 | TURN RIGHT | `turn(1, T, -1)` | T = period (ms) |
| 5 | UPDOWN | `updown(1, T, moveSize)` | T, moveSize |
| 6 | MOONWALKER LEFT | `moonwalker(1, T, moveSize, 1)` | T, moveSize |
| 7 | MOONWALKER RIGHT | `moonwalker(1, T, moveSize, -1)` | T, moveSize |
| 8 | SWING | `swing(1, T, moveSize)` | T, moveSize |
| 9 | CRUSAITO FORWARD | `crusaito(1, T, moveSize, 1)` | T, moveSize |
| 10 | CRUSAITO BACKWARD | `crusaito(1, T, moveSize, -1)` | T, moveSize |
| 11 | JUMP | `jump(1, T)` | T |
| 12 | FLAPPING FORWARD | `flapping(1, T, moveSize, 1)` | T, moveSize |
| 13 | FLAPPING BACKWARD | `flapping(1, T, moveSize, -1)` | T, moveSize |
| 14 | TIPTOE SWING | `tiptoeSwing(1, T, moveSize)` | T, moveSize |
| 15 | BEND LEFT | `bend(1, T, 1)` | T |
| 16 | BEND RIGHT | `bend(1, T, -1)` | T |
| 17 | SHAKE LEG LEFT | `shakeLeg(1, T, 1)` | T |
| 18 | SHAKE LEG RIGHT | `shakeLeg(1, T, -1)` | T |
| 19 | JITTER | `jitter(1, T, moveSize)` | T, moveSize |
| 20 | ASCENDING TURN | `ascendingTurn(1, T, moveSize)` | T, moveSize |
| 30+ | MANUAL SERVO | `_moveServos(200, raw)` | 4 raw positions (0–180) |

```mermaid
flowchart LR
    subgraph MOVEMENTS["Movement library"]
        M0["0: HOME"] --- M1["1: WALK FW"]
        M1 --- M2["2: WALK BW"]
        M2 --- M3["3: TURN L"]
        M3 --- M4["4: TURN R"]
        M4 --- M5["5: UPDOWN"]
        M5 --- M6["6: MOONWALK L"]
        M6 --- M7["7: MOONWALK R"]
        M7 --- M8["8: SWING"]
        M8 --- M9["9: CRUSAITO FW"]
        M9 --- M10["10: CRUSAITO BW"]
        M10 --- M11["11: JUMP"]
        M11 --- M12["12: FLAP FW"]
        M12 --- M13["13: FLAP BW"]
        M13 --- M14["14: TIPTOE SWING"]
        M14 --- M15["15: BEND L"]
        M15 --- M16["16: BEND R"]
        M16 --- M17["17: SHAKE LEG L"]
        M17 --- M18["18: SHAKE LEG R"]
        M18 --- M19["19: JITTER"]
        M19 --- M20["20: ASCEND TURN"]
    end
```

---

## 6. Gesture Reference

| ID | Name | Sequence |
|----|------|----------|
| 0 | HAPPY | S_happy → swing → mouth bigSmile |
| 1 | SUPER HAPPY | S_happy + S_superHappy → tiptoe swing |
| 2 | SAD | sad position → descending tones → sad mouth |
| 3 | SLEEPING | bed position → dream mouth animation → S_sleeping |
| 4 | FART | 3 fart positions → 3 fart sounds → tongueOut mouth |
| 5 | CONFUSED | confused position → S_confused → confused mouth |
| 6 | LOVE | heart mouth → S_cuddly → crusaito |
| 7 | ANGRY | angry position/face → S_confused → jitter |
| 8 | FRETFUL | angry face → fretful oscillation |
| 9 | MAGIC | 4 adivinawi cycles + ascending/descending tones |
| 10 | WAVE | 2 wave cycles + tone sweep |
| 11 | VICTORY | legs up + tones → tiptoe swing → S_happy |
| 12 | FAIL | progressive bend + descending tones → detach servos → long low tone |

---

## 7. Sound Reference

| ID | Song Name | Notes |
|----|-----------|-------|
| 0 | S_connection | Power-on jingle |
| 1 | S_disconnection | Power-off jingle |
| 2 | S_buttonPushed | UI feedback |
| 3 | S_mode1 | Dance mode activation |
| 4 | S_mode2 | Obstacle mode activation |
| 5 | S_mode3 | Noise mode activation |
| 6 | S_surprise | OhOoh surprise |
| 7 | S_OhOoh | Alternate OhOoh |
| 8 | S_cuddly | Affectionate melody |
| 9 | S_sleeping | Lullaby |
| 10 | S_happy | Happy melody |
| 11 | S_superHappy | Super happy melody |
| 12 | S_happy_short | Short happy |
| 13 | S_sad | Sad melody |
| 14 | S_confused | Confused melody |
| 15 | S_fart1 | Fart sound 1 |
| 16 | S_fart2 | Fart sound 2 |
| 17 | S_fart3 | Fart sound 3 |
| 18 | S_fart3 | Long fart |

### Low-level sound blocks

```mermaid
flowchart LR
    TONE["TONE(freq_Hz, duration_ms)<br/>→ tone(buzzerPin, freq, dur)"] 
    SWEEP["SWEEP(start_Hz, end_Hz, step, duration_ms)<br/>→ multiply/divide freq by step each iteration"]
```

---

## 8. Mouth Reference (LED Matrix 5×6)

30 predefined mouth shapes driven by a 74HC595 shift register (pins 11=SER, 12=RCK, 13=CLK).

```mermaid
flowchart LR
    subgraph MOUTHS["Predefined mouth shapes"]
        direction LR
        M10["10: bigSmile"] --- M11["11: happyOpen"]
        M11 --- M12["12: bigHappyOpen"]
        M12 --- M13["13: heartMouth"]
        M13 --- M14["14: bigHeart"]
        M14 --- M15["15: littleUuh"]
        M15 --- M16["16: dogMouth"]
        M16 --- M17["17: tigerMouth"]
        M17 --- M18["18: catMouth"]
        M18 --- M19["19: tongueOut"]
        M19 --- M20["20: sceptic"]
        M20 --- M21["21: sadMouth"]
        M21 --- M22["22: confMouth"]
        M22 --- M23["23: lineMouth"]
        M23 --- M24["24: smallMouth"]
        M24 --- M25["25: bigSurprise"]
        M25 --- M26["26: medSurprise"]
        M26 --- M27["27: smallSurprise"]
        M27 --- M28["28: thunder"]
        M28 --- M29["29: culito"]
        M29 --- M30["30: openMouth"]
    end
```

### Mouth animations

| Animation | Frames | Description |
|-----------|--------|-------------|
| littleUuh | 8 | Small blinking mouth |
| dreamMouth | 4 | Dreamy mouth with closed eyes |
| adivinawi | 6 | Guessing / magic mouth |
| wave | 10 | Wave pattern |

### Raw mouth

```
RAW MOUTH(30-bit binary)  →  writeFull(pattern)  →  shift register output
```

---

## 9. Sensor Reference

```mermaid
flowchart TD
    subgraph SENSORS["Sensor readings"]
        DIST["DISTANCE (cm)<br/>triggerPin=8, echoPin=9<br/>trigger 10µs → pulseIn<br/>cm = microseconds / 29 / 2"]
        
        NOISE["NOISE (0–1023)<br/>pin=A6<br/>average of 2 readings"]
        
        BATT_PCT["BATTERY (% , 0–100)<br/>pin=A7<br/>average of 10 readings<br/>linear map: 3.25V→0%, 4.2V→100%"]
        
        BATT_VOLT["BATTERY VOLTAGE (V)<br/>pin=A7<br/>average of 10 readings<br/>scale to volts"]
    end
```

---

## 10. Serial Command Protocol

```mermaid
flowchart LR
    subgraph PROTOCOL["Protocol frame"]
        FMT["Format: &&&lt;CMD&gt; [arg1] [arg2]...%%"]
        BAUD["Baud rate: 115200"]
        RESP["Response: &&A%% (ACK) then &&F%% (Final ACK)<br/>Read-only: &&&lt;CMD&gt; &lt;value&gt;%%"]
    end
```

| CMD | Block | Args | Response |
|-----|-------|------|----------|
| `S` | STOP Zowi | — | `&&A%%` → home → `&&F%%` |
| `L` | MOUTH | 30-bit binary | `&&A%%` → `&&F%%` |
| `T` | TONE | freq_Hz duration_ms | `&&A%%` → `&&F%%` |
| `M` | MOVE | ID T moveSize | `&&A%%` → move → `&&F%%` |
| `H` | GESTURE | ID (0–12) | `&&A%%` → `&&F%%` |
| `K` | SONG | ID (0–18) | `&&A%%` → `&&F%%` |
| `C` | CALIBRATE | YL YR RL RR | `&&A%%` → save EEPROM → `&&F%%` |
| `G` | SERVO RAW | YL YR RL RR | `&&A%%` → move → `&&F%%` |
| `R` | NAME | text string | `&&A%%` → save EEPROM → `&&F%%` |
| `E` | GET NAME | — | `&&E <name>%%` |
| `D` | GET DISTANCE | — | `&&D <cm>%%` |
| `N` | GET NOISE | — | `&&N <value>%%` |
| `B` | GET BATTERY | — | `&&B <%>%%` |
| `I` | GET PROGRAM ID | — | `&&I ZOWI_BASE_v2%%` |

---

## 11. Servo Configuration

| Pin | Identifier | Servo | Role |
|-----|------------|-------|------|
| 2 | `PIN_YL` | YL | Left leg |
| 3 | `PIN_YR` | YR | Right leg |
| 4 | `PIN_RL` | RL | Left foot |
| 5 | `PIN_RR` | RR | Right foot |

### Servo functions

```mermaid
flowchart LR
    ATTACH["ATTACH()<br/>servo.attach(pin) each"]
    DETACH["DETACH()<br/>servo.detach() each"]
    HOME["HOME()<br/>all → 90° (500ms)<br/>then detach"]
    CALIB["CALIBRATE(YL, YR, RL, RR)<br/>update trim values<br/>save to EEPROM[0..3]"]
```

---

## 12. EEPROM Memory Map

| Bytes | Usage | Values |
|-------|-------|--------|
| 0–3 | Trims | YL, YR, RL, RR (signed byte, calibration offsets) |
| 4 | (free) | — |
| 5 | State marker | `'$'` = factory (infinite loop on boot), `'#'` = unbaptized (extended greeting), other = active name |
| 5–15 | Name | Up to 10 characters + null terminator (`\0`) |

---

*Generated from `code/base/ZOWI_BASE_v2.ino` and supporting libraries.*
