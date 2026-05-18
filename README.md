# FurPy

A drop-in replacement mainboard for the 1998 Tiger Electronics Furby, powered by a Raspberry Pi Zero 2 W.

![FurPy board](https://github.com/gsswrk/FurPy/assets/26857790/cf27e0b5-9f9a-42a9-9bc4-a1816d970fae)

> Follow progress on the blog: [gsswrk.com/posts/furpy-redux](https://gsswrk.com/posts/furpy-redux/)

---

## What Is FurPy?

The original 1998 Furby runs on a custom Tiger Electronics chip (SPC81A) with a fixed ROM. FurPy replaces that mainboard entirely with a custom PCB that plugs directly into the Furby's existing wiring harness. Swap the board, add a Raspberry Pi Zero 2 W, and your Furby runs Linux.

All original Furby hardware is preserved and fully accessible:

- Single motor cam mechanism (eyes, mouth, ears, body rock)
- Belly button and tongue switches
- Gear position sensor and cam home sensor
- Light sensor (LDR)
- IR port for Furby-to-Furby communication
- Speaker and microphone

On top of that, FurPy adds:

- I2S audio output via MAX98357A amplifier
- I2S MEMS microphone (SPH0645LM4H) for always-on voice input
- LSM6DSOX IMU replacing the original tilt ball switch
- USB-C power input with polyfuse protection
- LiPo battery header (optional, for wireless operation)
- WiFi and Bluetooth via the RPi Zero 2 W

---

## Design Goals

**1. Full original compatibility.** Every sensor and actuator from the original SPC81A port map is wired to a dedicated GPIO. The Furby behaves like a Furby, just smarter.

**2. Open platform.** Drop a Python script into `/home/pi/furpy/behaviors/` and it loads automatically. The hardware abstraction layer (HAL) exposes everything as a clean API so you can write custom behaviors without touching the core firmware.

**3. Furby-to-Furby IR.** The IR port is a first-class feature. Use it to script how your Furby interacts with others, create synchronized behaviors, or build your own IR protocol on top of the driver.

**4. Cost effective.** Target bill of materials under $25 at quantity via JLCPCB PCBA. You supply the Raspberry Pi Zero 2 W and the Furby.

**5. Power efficient.** Runs on USB-C (5V/2A). Optional LiPo battery header on the board for wireless operation.

---

## Hardware

### Components (on the FurPy board)

| Function | Part |
|---|---|
| Motor driver | TB6612FNG (SSOP-24) |
| Audio amplifier | MAX98357A (I2S, drives original 8 ohm speaker) |
| MEMS microphone | SPH0645LM4H-1 (I2S) |
| IMU / motion | LSM6DSOX (replaces tilt ball switch) |
| Gear signal conditioning | 74AHCT1G14GW Schmitt trigger |
| Power input | USB-C + RXEF030 3A polyfuse |
| IR transmit | TSAL6400 IR LED (BCM 12, 38 kHz carrier) |
| IR receive | TSOP38238 demodulator (BCM 27) |
| LiPo (optional) | JST-PH 2-pin header, IP5306 power path IC |

You supply: **Raspberry Pi Zero 2 W**

### GPIO Pinout

| BCM | Function | Direction | Notes |
|---|---|---|---|
| 4 | Light sensor (LDR) | Input | RC timing, 220 pF filter |
| 5 | Motor PWMA | Output | Hardware PWM, 50% duty |
| 6 | Gear LED drive (Motored) | Output | Must stay HIGH while motor runs |
| 12 | IR transmit | Output | Hardware PWM, 38 kHz carrier |
| 13 | Tongue switch | Input | Active LOW |
| 17 | Belly button | Input | Active LOW |
| 18 | I2S BCLK | Output | Audio clock |
| 19 | I2S LRCLK | Output | Audio frame clock |
| 20 | I2S DIN | Input | Mic data in |
| 21 | I2S DOUT | Output | Speaker data out |
| 22 | Cam home sensor | Input | Active LOW, once per revolution |
| 23 | Motor AIN1 | Output | Direction control |
| 24 | Motor AIN2 | Output | Direction control |
| 25 | Motor STBY | Output | Enable, active HIGH |
| 26 | Gear position (GEAR_POS) | Input | ~1000 pulses/rev via Schmitt trigger |
| 27 | IR receive | Input | TSOP38238 demodulated output |

---

## Software

The software stack is still in development. Current test scripts in this repository were written to validate individual subsystems during prototyping.

The full firmware will include:

- `hal.py`: hardware abstraction layer exposing all Furby I/O as a Python API
- `motor.py`: cam position control with GEAR_POS feedback loop
- `ir.py`: IR transmit and receive driver with Furby protocol support
- `behaviors.py`: idle, wake, sleep, and reaction state machine
- `stt.py`: wake word detection and speech-to-text (openWakeWord + Whisper tiny)
- `tts.py`: text-to-speech output synced to mouth motor
- `main.py`: event loop that loads user behavior scripts at startup

### Writing Custom Behaviors

Once the HAL is complete, you will be able to drop a Python file into `/home/pi/furpy/behaviors/` and FurPy will load it automatically. The API is designed to be simple:

```python
import furpy

@furpy.on("belly_press")
def on_belly(event):
    furpy.motor.move_to("surprised")
    furpy.speak("Hey, watch it!")

@furpy.on("ir_receive")
def on_ir(payload):
    furpy.ir.send({"reply": "hello back"})
```

---

## Repository Contents

| File | Description |
|---|---|
| `motor.py` | TB6612FNG driver with gear LED and position tracking |
| `bellybutton_motorsensor.py` | Belly button and gear sensor interrupt test |
| `button_belly_only_test.py` | Belly button standalone test |
| `tongue_btn_test.py` | Tongue switch standalone test |
| `photo_sensor_test.py` | LDR light sensor RC timing test |
| `neopixel_w_accelerometer.py` | Legacy: NeoPixel test (NeoPixels are not part of v2 design) |

---

## Project Status

- [x] Original SPC81A port map decoded from Dave Hampton 1998 source
- [x] All Furby sensors and actuators mapped to RPi GPIO
- [x] PCB schematic complete (KiCad, ERC clean)
- [x] PCB layout routed
- [ ] PCB ordered from JLCPCB
- [ ] Hardware validated on physical board
- [ ] HAL and firmware complete
- [ ] IR driver and protocol implementation
- [ ] Voice pipeline (wake word, STT, TTS)
- [ ] Behavior scripting system
- [ ] Install guide and documentation

---

## Background

The Furby's original SPC81A MCU uses a single motor and cam mechanism to drive all physical movement: eyes open and close, the mouth moves, ears tilt, and the body rocks, all from one motor at different cam positions. FurPy preserves this entirely. The cam home sensor (BCM 22) provides a reference position once per revolution, and the gear position sensor (BCM 26, cleaned by a Schmitt trigger) counts ~1000 pulses per revolution for fine position control.

Original Furby source code analysis by Dave Hampton (1998) was used to derive the correct GPIO assignments, motor constants, and gear sensor requirements.

---

## License

MIT License. See [LICENSE](LICENSE) for details.

---

## Contributing

This project is actively developed. Issues and pull requests are welcome. If you are building your own FurPy or have a 1998 Furby to test with, reach out.
