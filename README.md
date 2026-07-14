# Zowi

![Zowi](https://it-bqcom15-media.s3.amazonaws.com/prod/images/200_200/1/a/9/a/1a9ae51edfd369f598cab125e75906bb2813e4a2.jpg)
![Zowi banner](http://zowi.bq.com/wp-content/themes/bq-zowi-wp/assets/images/01_inicio_RESPONSIVE_v2_1600x520.jpg)

Zowi was born as a toy, but after playing with it, it becomes an educational robotic platform with a long didactic journey.

Zowi starts by charming and entertaining, but at the same time it sparks curiosity about how robots and technological products work. Through its projects and games, it touches on concepts like hardware, software, and 3D design — the fundamental pillars of any technological product.

To summarize its technical qualities: Zowi is a biped robot capable of dancing, measuring distances, detecting noise, emitting sounds, and displaying mouths on an LED panel. Zowi has a reprogrammable brain board powered by a wireless battery. Finally, Zowi can communicate and be controlled via Bluetooth from an Android application (called ZowiApp).

<https://youtu.be/YlPRHd19NHI>

## Basic Program

Zowi's basic program is written in **Arduino code**.

The program has the identifier *ZOWI_BASE_vXX* (programID) where XX is the version.

### EEPROM Variables

- Servo calibration:
    ```
    EEPROM[0] = TRIM_YL
    EEPROM[1] = TRIM_YR
    EEPROM[2] = TRIM_RL
    EEPROM[3] = TRIM_RR
    ```

- Zowi's name:
    Starting at EEPROM address 5
    ```
    EEPROM[5] = '#'
    ```

## Installation and Libraries

*ZOWI_BASE* depends on the following Arduino libraries:

- **Zowi.h**
    Library with all movement, gesture, sound and mouth functions that Zowi can execute.

- **Oscillator.h**
    Generates sinusoidal oscillations for the servos. Created by Obijuan (https://github.com/Obijuan)

- **ZowiSerialCommand.h**
    Reads and interprets commands received via serial port. A modification of the original "SerialCommand.h" by Steven Cogswell (http://awtfy.com)

- **BatReader.h**
    Reads the battery level.

- **US.h**
    Reads the ultrasonic sensor.

- **LedMatrix.h**
    Programs the LED matrix of Zowi's mouth.

- **EnableInterrupt.h**
    Enables interrupts for Zowi's buttons.

To use them, add them to your program or install them. One way to manually install libraries is to copy each library folder into the Arduino libraries folder, i.e. into ***Arduino\libraries***.

On Windows, it may be called *My Documents\Arduino\libraries*. For Mac users, it is likely *Documents/Arduino/libraries*. On Linux, it will be a folder like *libraries* inside the '*sketchbook*'.

## Requirements

- ``Zowi robot``
- ``Arduino IDE`` <http://arduino.cc/en/Main/Software>
- ``Zowi App``: Free Android app to control Zowi

[![Download Zowi App](http://zowi.bq.com/wp-content/themes/bq-zowi-wp/assets/images/03_DescargalaApp_RESPONSIVE.png)](https://play.google.com/store/apps/details?id=com.bq.zowi&hl=es)

<https://play.google.com/store/apps/details?id=com.bq.zowi&hl=es>

![Zowi App](http://zowi.bq.com/wp-content/themes/bq-zowi-wp/assets/images/app-web-app-big.png)

## Communication Protocol with ZowiApp

*ZOWI_BASE* uses its own language (a series of commands) to communicate with *ZowiApp*. These commands allow the Android app to monitor Zowi's status and send instructions.

### Arduino - Android Communication Setup

### Zowi Commands or Instructions

## Standalone Mode (No App)

ZOWI_BASE has 3 mini-games that work without the App so you can use Zowi and see how it moves its robo-skeleton before even installing the App.

The mini-games are executed by pressing Zowi's back buttons:
![botones_zowi](http://zowi.bq.com/wp-content/uploads/2015/11/todosBotones-300x117.png)
- Button **A** = Random dance.
- Button **B** = Reacts and dodges obstacles.
- Button **A + B** = Reacts to taps or claps.

## License

Zowi is a free robot, meaning both its physical design and programming are available for anyone to see, study, and modify to improve it.

Zowi is distributed under the terms of the GPL license. Visit http://www.gnu.org/licenses/ for more details.
</details>
