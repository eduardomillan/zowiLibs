# Factory Zowi

A factory calibration and testing sketch for Zowi. Developed by BQ (Anita de Prado & Jose Alberca, September 2015). Released under GPL.

## Purpose

This sketch is used during manufacturing to verify that Zowi's hardware is working correctly before shipping. It tests the LED matrix, ultrasonic sensor, servos, and buttons in a sequential procedure.

## Procedure

The sketch runs entirely in `setup()` — the `loop()` is empty. The operator follows a visual sequence on Zowi's LED mouth and interacts via the back buttons and ultrasonic sensor.

### Step 1 — Factory Name Initialization

```cpp
char n = EEPROM.read(fac_eeAddress);
if (n != '$') {
    EEPROM.write(fac_eeAddress, '$');
}
```

If the EEPROM at address 5 does not already contain `$`, it writes the factory name marker. This ensures Zowi ships with a clean name.

### Step 2 — Button Release Check

```cpp
while ((digitalRead(PIN_SecondButton) == 1) || (digitalRead(PIN_ThirdButton) == 1))
```

Zowi waits for both back buttons to be released. While any button is held, it shows an **X mouth** (`xMouth_code`) on the LED matrix. This prevents the test from starting with a button already pressed.

### Step 3 — Ultrasonic Sensor Test

```cpp
while ((digitalRead(PIN_SecondButton) == 0) || (digitalRead(PIN_ThirdButton) == 0))
```

With both buttons pressed, Zowi reads the ultrasonic sensor:
- If an object is within **15 cm**, it shows a **heart** on the LED mouth.
- Otherwise, it lights up the **entire matrix**.

This verifies the HC-SR04 sensor is working. The operator places an object close to verify the heart appears, then removes it to confirm the full matrix lights up.

### Step 4 — Servo Check

```cpp
zowi.moveServos(500, checkPosition);  // {70, 110, 60, 120}
```

Zowi moves its servos to a spread position (70, 110, 60, 120) to verify all four servos are attached and moving correctly. The operator visually confirms the legs move to the expected positions.

### Step 5 — Button B Test

```cpp
while ((digitalRead(PIN_ThirdButton) == 0)) {
    delay(100);
}
```

Zowi waits for button B (pin 7) to be pressed. This verifies the button is functional.

### Step 6 — Final OK

```cpp
ledmatrix.writeFull(okMouth_code);
zowi.moveServos(500, homePosition);
```

Zowi shows an OK mouth and returns all servos to home position (90°). The test is complete.

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
| LED data (SER) | 11 |
| LED latch (RCK) | 12 |
| LED clock (CLK) | 13 |

## Functions

### `TP_init(trigger_pin, echo_pin)`

Sends a 10 µs trigger pulse and returns the echo pulse width in microseconds (timeout: 100 ms).

### `Distance(trigger_pin, echo_pin)`

Returns the distance in cm using `TP_init`. Formula: `microseconds / 29 / 2`. Returns 999 if no echo.
