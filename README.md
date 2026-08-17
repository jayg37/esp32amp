# ESP32 Outdoor Amplifier

A network-connected outdoor stereo amplifier built around an **ESP32-S3**, **PCM5102A DAC**, and **TPA3116D2 Class-D amplifier**, designed to integrate with **Home Assistant** and **Music Assistant** for whole-home synchronized music.

The goal is a relatively inexpensive, modular network amplifier that can drive a pair of passive outdoor speakers while behaving like another room/endpoint in the home's multi-room audio system.

---

## Project Goals

The amplifier is intended to provide:

* Stereo amplification for a pair of passive outdoor speakers
* Network audio playback
* Integration with Home Assistant
* Integration with Music Assistant
* Synchronized playback with other network audio endpoints
* Wireless control rather than a dedicated physical source
* OTA firmware updates
* Software volume control
* Physical rotary encoder support for local volume/control
* A dedicated 24 V power supply
* A regulated 5 V supply for the ESP32 and DAC
* Retention of the original audio files/stream sources in the central music system rather than storing music locally

The long-term architecture is to make this amplifier another endpoint in the home's distributed audio system rather than treating it as a conventional Bluetooth or analog-input amplifier.

---

# System Architecture

```text
                    Home Network
                         │
                         │ Wi-Fi
                         ▼
                ┌─────────────────┐
                │ Music Assistant │
                │  Home Assistant │
                └────────┬────────┘
                         │
                    Network Audio
                         │
                         ▼
                ┌─────────────────┐
                │    ESP32-S3     │
                │                 │
                │ Network Audio   │
                │   Endpoint      │
                └────────┬────────┘
                         │ I²S
                         ▼
                ┌─────────────────┐
                │    PCM5102A     │
                │   Stereo DAC    │
                └────────┬────────┘
                         │ Analog L/R
                         ▼
                ┌─────────────────┐
                │    TPA3116D2    │
                │ Class-D Amplifier│
                └───────┬─┬───────┘
                        │ │
                     L  │ │  R
                        │ │
                 ┌──────┘ └──────┐
                 ▼               ▼
             Left Speaker    Right Speaker
```

The amplifier itself does not need to contain a music library. Music is supplied by the network audio system.

---

# Major Hardware

## Controller

### ESP32-S3 DevKitC-1

The ESP32-S3 provides:

* Wi-Fi connectivity
* Network audio processing
* Home Assistant connectivity
* Firmware updates
* Local control logic
* Rotary encoder interface
* I²S audio output

The ESP32-S3 is the central controller for the project.

---

## DAC

### PCM5102A

The PCM5102A converts the digital I²S audio stream from the ESP32 into analog stereo audio.

Connections:

* I²S BCLK
* I²S LRCK / word clock
* I²S DATA
* Ground
* 5 V power
* Left analog output
* Right analog output

The PCM5102A is used because the ESP32's digital audio output should not be connected directly to the analog amplifier input.

---

## Amplifier

### TPA3116D2

The TPA3116D2 is the Class-D power amplifier.

It receives the analog stereo signal from the PCM5102A and drives the passive speakers.

The amplifier provides substantially more output power than the ESP32/DAC section and is therefore powered directly from the 24 V supply.

---

## MA12070P + ESP32 Backplane

The project also uses the MA12070P + ESP32 backplane as the physical/control platform associated with the amplifier project.

**Important:** the backplane does not replace the external amplifier module used in this build. The actual power amplification is provided by the TPA3116D2 amplifier.

---

# Power System

The project uses a **24 V DC primary supply**.

```text
24 V DC Power Supply
        │
        ├──────────────► TPA3116D2 amplifier
        │
        │
        ▼
   24 V → 5 V Buck
     Converter
        │
        ├──────────────► ESP32-S3 5 V / VIN
        │
        └──────────────► PCM5102A 5 V
```

## Why two voltage levels?

The TPA3116D2 amplifier benefits from the higher 24 V supply.

The ESP32-S3 and PCM5102A require a regulated lower voltage supply.

A buck converter is therefore used rather than attempting to power the ESP32 directly from 24 V.

### Power requirements

The buck converter must be capable of supplying the ESP32, DAC, and any associated peripherals with adequate current margin.

The amplifier's 24 V supply must be sized for the expected speaker load and desired output power.

---

# Wiring

## ESP32 → PCM5102A

The ESP32 sends stereo digital audio to the PCM5102A using I²S.

The three required audio signals are:

| ESP32 Signal | PCM5102A Signal | Function              |
| ------------ | --------------- | --------------------- |
| BCLK         | BCK / BCLK      | Bit clock             |
| LRCK         | LRCK / LCK      | Left/right word clock |
| DATA         | DIN             | Digital audio data    |
| GND          | GND             | Common reference      |
| 5 V          | VCC             | DAC power             |

### Important

The PCM5102A does **not** require a separate MCLK/SCK connection for this implementation.

The three-wire I²S interface is therefore:

```text
ESP32
 ├── BCLK ─────► PCM5102A BCK
 ├── LRCK ─────► PCM5102A LRCK
 └── DATA ─────► PCM5102A DIN
```

Do not add an SCK/MCLK connection simply because the DAC board exposes a pin for it.

---

# PCM5102A → TPA3116D2

The DAC produces analog stereo audio.

```text
PCM5102A                    TPA3116D2

LEFT OUT  ─────────────────► INL
RIGHT OUT ─────────────────► INR
GND       ─────────────────► GND
```

The analog ground must be common between the DAC and amplifier.

Keep the analog signal wiring reasonably short and separated from the high-current amplifier/power wiring where practical.

---

# TPA3116D2 → Speakers

The TPA3116D2 provides the amplified stereo output.

```text
TPA3116D2

L+ / L- ─────────► Left passive speaker

R+ / R- ─────────► Right passive speaker
```

**Do not connect either speaker negative terminal directly to system ground.**

The TPA3116D2 uses bridged amplifier outputs. Speaker wiring must follow the amplifier board's output terminals.

Use speaker wire appropriate for the amplifier power and cable length.

---

# ESP32 Power Wiring

The established power wiring uses the regulated 5 V output from the buck converter.

```text
24 V supply
    │
    ▼
24 V → 5 V buck converter
    │
    ├── 5 V ─────► ESP32 5 V/VIN
    │
    └── 5 V ─────► PCM5102A VCC
       
Ground
    │
    ├── ESP32 GND
    ├── PCM5102A GND
    └── amplifier signal ground
```

The amplifier's power ground and the low-voltage audio/control ground ultimately need an appropriate common reference.

---

# Physical Wiring Reference

The original wiring notes for the prototype use the following wire-color convention.

## ESP32 / amplifier-side wiring

| Signal     | Wire Color       |
| ---------- | ---------------- |
| 5 V / VIN  | Green            |
| GND        | Blue             |
| LRCK       | Purple           |
| DIN / DATA | Gray             |
| BCK        | White            |
| SCK / MCLK | Black — not used |

## DAC → amplifier

| Signal      | Wire Color |
| ----------- | ---------- |
| INL / Left  | White      |
| GND         | Black      |
| INR / Right | Red        |

These colors are documentation for the prototype wiring and are not electrical requirements.

---

# ESP32 Pin Planning

The ESP32-S3 DevKitC-1 has different accessible GPIOs than some of the earlier ESP32 boards used in other projects.

**Do not assume GPIO25/26/27 are available on this particular hardware.**

The final GPIO assignment must therefore be based on the actual ESP32-S3 DevKitC-1 pinout and the pins physically available on the assembled board.

The YAML in this repository should be considered the authoritative source for the final GPIO assignment once the hardware configuration is finalized.

---

# Rotary Encoder

A rotary encoder is planned for local control.

The intended functions are:

* Rotate clockwise → volume up
* Rotate counter-clockwise → volume down
* Press → play/pause or another configurable control function

The encoder provides local control without requiring a phone or Home Assistant interface.

The encoder should be connected to GPIOs that are actually exposed and suitable for input on the selected ESP32-S3 board.

---

# Software

## ESPHome

[ESPHome](https://esphome.io/) is used as the firmware/configuration environment.

ESPHome provides:

* ESP32 firmware generation
* Wi-Fi
* OTA updates
* Home Assistant integration
* hardware abstraction
* I²S audio support
* local controls
* diagnostic sensors

The project configuration is maintained as YAML.

---

## Home Assistant

[Home Assistant](https://www.home-assistant.io/) provides the overall home automation environment.

The amplifier is intended to appear as a controllable audio endpoint within Home Assistant.

Home Assistant can provide:

* Volume control
* Playback control
* Automation
* Room grouping
* Status
* Diagnostics
* Remote control

---

## Music Assistant

[Music Assistant](https://music-assistant.io/) is the primary music-management layer.

Music Assistant provides access to the household's supported music sources and distributes audio to compatible players.

The project is specifically intended to become another Music Assistant playback endpoint.

Sources available through the Music Assistant installation may include services such as:

* Spotify
* Tidal
* Qobuz
* Apple Music
* Local filesystem
* Network filesystem
* Other supported providers

The amplifier itself does not need to know where the music originated.

---

## Network Synchronization

Synchronized playback is an important project requirement.

The desired behavior is:

```text
Music Assistant
       │
       ├────────► Indoor Player
       │
       ├────────► Outdoor Player
       │
       └────────► Other Network Players
```

All endpoints should remain synchronized closely enough that music can move between rooms without obvious echo or timing differences.

### Sendspin

Sendspin is being kept as the preferred network-audio direction for the ESP32 implementation where supported.

The project should retain flexibility around the network audio protocol because the ecosystem is still evolving.

The earlier short-term alternative considered for sources such as a turntable was VBAN, but that is a separate network-audio problem and is not the primary architecture of this amplifier.

---

# Why This Is Not Just a Bluetooth Amplifier

A Bluetooth amplifier would make the phone the source and would generally make multi-room synchronization much more difficult.

This project instead treats the amplifier as a **network audio endpoint**.

That means:

```text
Phone / Tablet / Home Assistant
              │
              ▼
       Music Assistant
              │
              ▼
        Network audio
              │
              ▼
          ESP32
              │
              ▼
          Amplifier
              │
              ▼
          Speakers
```

The phone is therefore a controller rather than the audio transport.

---

# Materials / Bill of Materials

## Required

* ESP32-S3 DevKitC-1
* PCM5102A stereo DAC
* TPA3116D2 stereo Class-D amplifier
* 24 V DC power supply
* 24 V → 5 V buck converter
* Pair of passive outdoor speakers
* Speaker wire
* Hookup wire
* Appropriate connectors/terminal blocks
* Enclosure
* Heat management/ventilation appropriate for the amplifier
* USB cable for initial ESP32 programming
* Network/Wi-Fi access

## Control

* Rotary encoder
* Encoder knob
* Optional encoder push-button circuitry if the selected encoder requires it

## Recommended

* Inline fuse or appropriately protected DC input
* Power switch
* Terminal blocks or locking connectors
* Strain reliefs
* Ferrules for screw terminals
* Heat-shrink tubing
* Cable management
* Standoffs for PCBs
* Shielded or twisted audio wiring where practical

---

# Tools Required

At minimum:

* Soldering iron
* Solder
* Wire cutters
* Wire strippers
* Small screwdrivers
* Multimeter
* USB cable
* Computer for ESPHome configuration

Strongly recommended:

* Bench power supply or current-limited power source for initial testing
* Oscilloscope for advanced audio troubleshooting
* Logic analyzer for I²S troubleshooting
* Crimping tools
* Heat gun
* Continuity tester

---

# Skills Required

This project is not a beginner-level plug-and-play electronics project.

A builder should be comfortable with:

## Basic electronics

* DC voltage and polarity
* Ground/common connections
* Current requirements
* Power distribution
* Basic resistor/capacitor concepts
* Reading connector labels

## Soldering

The project requires soldering:

* ESP32 connections
* DAC connections
* amplifier connections
* power wiring
* control wiring

Good soldering practices are important because audio equipment can be sensitive to poor grounds and intermittent connections.

## Digital audio

A basic understanding of:

* I²S
* BCLK
* LRCK
* DATA
* digital-to-analog conversion
* sample rate
* bit depth

is useful when troubleshooting the audio path.

## Networking

The builder should understand:

* Wi-Fi
* IP addresses
* Home Assistant
* network-discovered devices
* basic network troubleshooting

## ESPHome

The builder should be comfortable:

* editing YAML
* compiling firmware
* flashing an ESP32
* using OTA updates
* reading ESPHome logs
* troubleshooting GPIO configuration

## Home Assistant / Music Assistant

Basic familiarity with:

* Home Assistant entities
* media players
* Music Assistant
* audio providers
* player grouping

will make configuration substantially easier.

---

# Initial Assembly Procedure

## 1. Verify the hardware

Before applying power, verify:

* ESP32-S3 board
* DAC
* amplifier
* buck converter
* power supply
* speakers

and confirm the voltage requirements of every module.

---

## 2. Configure the buck converter

Set the buck converter output to the required regulated voltage **before connecting the ESP32 or DAC**.

Verify the output with a multimeter.

Do not assume that a buck converter is correctly adjusted from the factory.

---

## 3. Test the ESP32

Initially power only the ESP32 from the regulated 5 V supply.

Verify:

* ESP32 boots
* Wi-Fi connects
* ESPHome logs are available
* OTA works

---

## 4. Test the DAC

Connect:

* 5 V
* GND
* BCLK
* LRCK
* DATA

Verify that the ESP32 recognizes/initializes the audio hardware.

---

## 5. Connect the amplifier

With power removed:

* connect DAC left/right analog output
* connect DAC ground
* connect amplifier power
* connect speakers

Double-check polarity and amplifier output wiring before applying power.

---

## 6. First audio test

Start with low volume.

Verify:

* left channel plays from left speaker
* right channel plays from right speaker
* no excessive hum
* no distortion
* no unexpected noise
* amplifier remains at a reasonable temperature

---

# Troubleshooting

## No audio

Check in this order:

1. ESP32 Wi-Fi connection
2. Music Assistant player availability
3. Network audio connection
4. ESPHome logs
5. I²S BCLK
6. I²S LRCK
7. I²S DATA
8. PCM5102A power
9. PCM5102A ground
10. Analog L/R wiring
11. Amplifier power
12. Speaker wiring

---

## Hum or buzzing

Check:

* common ground
* power supply quality
* buck converter noise
* audio cable routing
* amplifier grounding
* separation between high-current power wiring and analog audio wiring

Do not immediately assume the DAC is defective.

---

## Distorted audio

Check:

* amplifier gain
* software volume
* DAC output level
* speaker impedance
* power supply voltage
* amplifier temperature
* clipping

Reduce software volume before increasing amplifier gain.

---

## Left/right reversed

Swap the PCM5102A analog connections or correct the channel assignment in software.

---

## ESP32 resets when music starts

This commonly indicates a power problem.

Check:

* buck converter current capability
* 5 V voltage under load
* ESP32 supply stability
* common-ground wiring
* amplifier power wiring
* electrical noise from the amplifier

The amplifier's high-current switching activity should not be allowed to destabilize the ESP32 supply.

---

# Safety

This project contains both **high-current DC power** and a high-power Class-D amplifier.

Although the primary supply is low-voltage DC, improper wiring can still cause:

* short circuits
* burns
* damaged electronics
* melted wiring
* fire

Always:

* disconnect power before changing wiring
* verify polarity before applying power
* fuse the supply appropriately
* use appropriately sized wire
* insulate exposed conductors
* provide strain relief
* keep conductive objects away from powered circuitry
* provide adequate amplifier ventilation
* use an enclosure suitable for the outdoor environment

The 24 V supply should be treated as a power source capable of delivering substantial current, not as harmless electronics power.

---

# Project Development Philosophy

The project is intentionally modular.

```text
Network / Music
       │
       ▼
    ESP32-S3
       │
       ▼
    PCM5102A
       │
       ▼
   TPA3116D2
       │
       ▼
    Speakers
```

Each section can therefore be tested independently.

This is preferable to building the complete system at once.

The recommended development sequence is:

1. ESP32
2. Wi-Fi
3. ESPHome
4. I²S
5. PCM5102A
6. Analog audio
7. TPA3116D2
8. Speakers
9. Network audio
10. Music Assistant
11. Synchronization
12. Rotary encoder
13. Final enclosure

---

# Repository Structure

The intended repository structure is:

```text
esp32amp/
├── README.md
├── patio-speakers.yaml
├── secrets.yaml.example
└── docs/
    ├── wiring.md
    ├── troubleshooting.md
    └── hardware.md
```

Only non-sensitive configuration belongs in the public repository.

Wi-Fi passwords, API encryption keys, OTA passwords, and other credentials must never be committed.

---

# Firmware Configuration

The ESPHome YAML in this repository is the authoritative firmware configuration.

Before flashing a new revision:

1. Verify the ESP32 board definition.
2. Verify GPIO assignments against the actual hardware.
3. Verify the I²S configuration.
4. Verify the DAC wiring.
5. Verify the amplifier wiring.
6. Compile before flashing.
7. Test at low volume after installation.

Hardware changes should be reflected in both the YAML comments and this README.

---

# Current Hardware Design

### Controller

ESP32-S3 DevKitC-1

### DAC

PCM5102A

### Amplifier

TPA3116D2

### Primary power

24 V DC

### Logic/DAC power

5 V regulated through buck converter

### Audio interface

I²S

### Network

Wi-Fi

### Home automation

Home Assistant

### Music management

Music Assistant

### Network synchronization

Sendspin-oriented architecture

### Local control

Rotary encoder

### Output

Stereo passive outdoor speakers

---

# Future Development

Potential future additions include:

* Local volume knob
* Play/pause button
* OLED display
* Amplifier temperature monitoring
* Power monitoring
* Automatic amplifier standby
* Enclosure-mounted status LED
* Improved startup/shutdown behavior
* Additional network audio protocols if required
* Turntable integration through a network audio input
* Multi-amplifier synchronization
* Improved outdoor weather protection
* Automatic power management

The turntable/network-AUX requirement is intentionally treated as a **future input/source project**, rather than complicating the basic outdoor amplifier design.

---

# Design Principle

The most important design principle for this project is:

> **Keep the amplifier simple and make the network/audio system responsible for the intelligence.**

The ESP32 should primarily be an audio endpoint and controller.

Music libraries, streaming services, multi-room grouping, source selection, and synchronization should remain centralized in Music Assistant/Home Assistant wherever practical.

This makes additional amplifiers straightforward to build:

```text
                    Music Assistant
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Indoor          Patio          Future
       Player          ESP32           ESP32
                         │
                       DAC
                         │
                     Amplifier
                         │
                     Speakers
```

The result is a scalable whole-home audio architecture rather than a collection of independent Bluetooth speakers.

---

## Project Status

**Development / prototype**

The hardware architecture has been selected and the project is being developed incrementally.

The firmware configuration should not be considered production-ready until the complete hardware wiring has been physically verified and the ESPHome configuration has been compiled and tested on the actual ESP32-S3 hardware.

---

## License

This project is primarily a personal hardware/software build.

Individual third-party components, libraries, firmware, and software retain their respective licenses.

See the documentation for each dependency before redistributing modified firmware or hardware designs.
