# XeWe LED - Buggy — animated RGB strips and a glowing logo for a dune buggy

Personal project · built 2019–2020, uploaded 2023-11-18 · Solo: Max Dokukin · Status: Completed

![Buggy front at night with the RGB strips and X logo lit](static/media/resources/IMG_0425.webp)

## Overview

Software for a dune buggy (originally published as **BuggyOS**). The repository holds two Arduino sketches. The
**Front Controller** runs the lights: an Arduino Nano drives four RGB LED strips through a PCA9685 PWM board (4 strips ×
3 colour channels = 12 PWM channels) and a 31-LED addressable strip inside a 3D-printed "X" logo. The strips play a
flicker-on, hold and cross-fade loop in cyan and magenta; the logo plays a staggered two-colour intro and then settles
into a continuous Perlin-noise animation. The **Central Controller** (`BuggyOS 1.04.13`) is the vehicle-side sketch:
gearbox motor control with position sensing, speed, RPM, mileage and engine hours stored in EEPROM, a temperature-driven
choke servo, an 8×8 LED-matrix warning display and a 25-command serial console.

## Highlights

- 31 addressable LEDs plus 4 RGB strips on 12 PCA9685 channels, all animated without `delay()` in one non-blocking loop — `Front_Controller/`
- Per-LED instruction queues: each LED holds a linked list of timed colour fades, so the logo intro is written as data (`LedInstruction`, `AdvancedLed`)
- Procedural "fire" animation from FastLED's `inoise8` noise mapped to HSV, redrawn every 10 ms — `PerlinFade.h`
- Randomised "ignition" flicker: 150–200 ms flashes shrinking by 30% per blink for 2 s before each strip settles — `StripsController.h`
- Central sketch: 25 serial commands, 8 animated warning icons, gear shifting with a 500 ms timeout and rollback error codes — `Central_Controller/`

## How it works

```
Front Controller (Arduino Nano)
  loop() ─┬─ LogoController → AdvancedLEDcontroller (per-LED fade queues) → PerlinFade → Adafruit_NeoPixel (31 LEDs, pin 5)
          └─ if pin 6 HIGH: StripsController → 4 channel state machines → Adafruit_PWMServoDriver → PCA9685 (I2C, 300 Hz) → 4 RGB strips

Central Controller (BuggyOS 1.04.13)
  Hall interrupts (speed, RPM) · DS18B20 temps · gear/brake/oil/fuel/park sensors
      → gear shifting, choke servo, EEPROM counters → 8×8 LED-matrix icon queue + serial console (9600 baud)
```

- **LogoController** (`Front_Controller/LogoController.h`) — stage 0: LEDs fade cyan → magenta in two waves, 500 ms per LED,
  starts staggered by 50 ms; stage 1: the reverse from the middle out; stage 2: all LEDs fade over 1 s into the first noise
  colour; then the noise animation runs forever.
- **PerlinFade** (`PerlinFade.h`) — `inoise8(i * 30, t)` per LED, `t += 5` every 10 ms; noise value → hue 25000–60000 on
  the 16-bit wheel, saturation 255 → 180, brightness 10 → 255.
- **AdvancedLEDcontroller / AdvancedLed / LedInstruction** — each LED owns a queue of `(start time, end time, start
  colour, end colour)` fades interpolated with `getMixCol`; the controller reports "done" when the last queued fade ends.
- **StripsController** (`StripsController.h`) — four channels (top left, lower left, top right, lower right), each a
  state machine: blink → hold 3 s → fade 3 s → hold → fade back → repeat. Top strips start cyan, lower strips magenta.
  Colours are 8-bit per channel scaled to the PCA9685's 0–4095 range.
- **Central Controller** (`Central_Controller/buggyOS_1_04_13.ino`, `serialConsole.h`, `icons.h`) — gear position read as a
  voltage on A2 (forward 5 V, neutral 4.2 V, rear 3.5 V, ±0.3 V); shifting drives the gearbox motor until the sensor
  confirms or 500 ms pass, then rolls back using error codes 251–254; gear buttons work only with the brake pressed and
  the buggy stopped. Wheel speed and RPM come from Hall-sensor interrupts; mileage, trip and engine hours persist in EEPROM;
  the choke servo opens 0–180° as engine temperature rises from 27 to 50 °C.

### Serial console (Central Controller, 9600 baud)

| Command | Action |
|---|---|
| `?` | list commands |
| `all` | print every reading below |
| `mileage` / `setmileage XXXX` | mileage in km / set mileage in metres (EEPROM) |
| `trip [reset]` | trip in km / reset trip |
| `hours` / `sethours XXXX` | engine hours / set engine hours in minutes (EEPROM) |
| `speed`, `rpm` | wheel speed, engine RPM |
| `gear [0/1/2]` | current gear or gearbox error / shift to drive, neutral, rear |
| `gearsensor [debug]` | active gear sensor / toggle gearbox debug output |
| `choke [auto/manual/angle]` | choke servo angle / auto or manual mode / set angle 0–180° |
| `brake`, `park`, `oil`, `fuel`, `brakeliquid` | brake pedal, parking brake, oil pressure, fuel level, brake-fluid level |
| `temp`, `oiltemp` | engine and oil temperature |
| `voltage`, `time` | supply voltage on A0, minutes since power-on |
| `matrixque` | warning-icon queue as a bit mask |
| `eeprom [save]` | read / write all counters in EEPROM |
| `reset` | save counters to EEPROM and restart |
| `debug` | toggle debug output |

## Results

| Measure | Value | Note |
|---|---|---|
| Addressable LEDs (logo) | 31 | `NUM_LEDS`, data pin 5, GRB 800 kHz |
| RGB strips / PWM channels | 4 / 12 | PCA9685 at 300 Hz, 12-bit duty |
| Animation frame period (noise) | 10 ms | `PerlinFade::frame()` |
| Strip timings | 2 s flicker, 3 s hold, 3 s fade | `BLINK_DELAY`, `HOLD_DELAY`, `FADE_DELAY` |
| Serial commands (central) | 25 | `serialConsole.h` |
| Warning icons on the 8×8 matrix | 8 in rotation, 3 s each | `MAX_ICON_QUE`, `ICON_DISPLAY_TIME` |
| Code size | 733 lines (front) + 1,297 lines (central) | `wc -l`, 11 source files |

This is a hardware build, so the results are its specifications; no measurements (current draw, brightness) were recorded.

## Getting started

Arduino IDE with these libraries:

- Front Controller: **Adafruit NeoPixel**, **FastLED** (for `inoise8`), **Adafruit PWM Servo Driver Library** (PCA9685), `Wire`
- Central Controller: **OneWire**, **DallasTemperature**, **Servo**, `EEPROM`, and **LedControlMS** (included by the sketch
  but not in this repository — install it separately)

```text
1. Open Front_Controller/Front_Controller.ino, select Arduino Nano, upload.
   Wiring: addressable strip data → D5; strip-enable input → D6 (HIGH = animate); PCA9685 on I2C (A4/A5);
   strip n uses PCA9685 channels 3n, 3n+1, 3n+2 (R, G, B).
2. Open Central_Controller/buggyOS_1_04_13.ino. It uses pins up to 36, so it needs a board with that many pins
   (e.g. an Arduino Mega 2560). Upload, then open the Serial Monitor at 9600 baud and type ? for the command list.
```

The central sketch still has debug switches marked for removal: the power-loss EEPROM save is commented out and the
engine-running pin is forced HIGH in `setup()`.

## Documents

- Demo video: [static/media/resources/IMG_0765_optimized.mp4](static/media/resources/IMG_0765_optimized.mp4)
- Photos: [static/media/resources/](static/media/resources/) — logo casing ([IMG_0349](static/media/resources/IMG_0349.webp)),
  finished logo ([IMG_0407](static/media/resources/IMG_0407.webp)), Nano + PCA9685 board ([IMG_0490](static/media/resources/IMG_0490.webp)),
  front controller installed ([IMG_0627](static/media/resources/IMG_0627.webp)), buggy at night ([IMG_0630](static/media/resources/IMG_0630.webp))
- Original BuggyOS README photos: [Buggy (IMG_4729)](https://github.com/xeweva/BuggyOS/assets/54597813/4ab92900-4517-49e6-9cb9-bc44df458f10),
  [IMG_0630](https://github.com/xeweva/BuggyOS/assets/54597813/f6bd9bf7-23e4-45ed-8612-cf353dcd85c2),
  [Front Controller (IMG_0627)](https://github.com/xeweva/BuggyOS/assets/54597813/5d102cc4-dc25-48d6-9aa4-f696aa8b9cda)
- Project page: [maxdokukin.com/projects/xewe-led-buggy](https://maxdokukin.com/projects/xewe-led-buggy)
- Related: XeWe LED OS — [project page](https://maxdokukin.com/projects/xewe-led-os) ·
  [github.com/xewe-labs/xewe-led-os](https://github.com/xewe-labs/xewe-led-os)
