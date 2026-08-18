# ESP32 Outdoor Amplifier

A network-connected outdoor stereo amplifier built around an **ESP32-S3**, **PCM5102A DAC**, and **TPA3116D2 Class-D amplifier**, designed to integrate with **Home Assistant** and **Music Assistant** for whole-home synchronized music.

The amplifier is intended to behave as another network audio endpoint rather than as a conventional Bluetooth or analog-input amplifier.

---

## Project Goals

The amplifier is intended to provide:

* Stereo amplification for a pair of passive outdoor speakers
* Network audio playback
* Home Assistant integration
* Music Assistant integration
* Synchronized playback with other network audio endpoints
* Wireless/software control
* OTA firmware updates
* Software volume control
* 24 V primary power for the amplifier
* Regulated 5 V power for the ESP32 and DAC
* OLED status/metadata display
* No local music library; audio remains sourced by the central music system

---

# Current Hardware Design

The current build consists of the following hardware:

* **ESP32-S3 DevKitC-1** — network controller and audio endpoint
* **PCM5102A** — stereo I²S DAC
* **TPA3116D2** — stereo Class-D power amplifier
* **SSD1306 OLED** — status/metadata display
* **24 V DC power supply** — primary amplifier supply
* **24 V → 5 V buck converter** — regulated low-voltage supply for the ESP32 and DAC

There is **no rotary encoder or other dedicated local-control hardware** in the current build.

There is also **no logic-level converter** between the boards. Control and digital signals use the normal ESP32 board logic levels and are connected directly between the appropriate devices.

The OLED display is powered from the ESP32 board's **3.3 V supply** and uses the ESP32's normal 3.3 V logic levels.

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
                └───────┬─┬───────┘
                        │ │
                  I²S   │ │ I²C
                        │ │
                        ▼ ▼
                 ┌──────────┐ ┌──────────┐
                 │ PCM5102A │ │ SSD1306  │
                 │ Stereo   │ │  OLED    │
                 │ DAC      │ │ Display  │
                 └────┬─────┘ └──────────┘
                      │ Analog L/R
                      ▼
                ┌─────────────────┐
                │    TPA3116D2    │
                │ Class-D Amplifier│
                └───────┬─┬───────┘
                        │ │
                     L  │ │  R
                        ▼ ▼
                 Left Speaker  Right Speaker
```

The amplifier itself does not contain a music library. Music is supplied by the network audio system.

---

# Major Hardware

## ESP32-S3 DevKitC-1

The ESP32-S3 is the central controller and network audio endpoint. It provides:

* Wi-Fi connectivity
* Network audio processing
* Home Assistant connectivity
* Firmware/OTA updates
* Software/control logic
* I²S audio output
* I²C display interface

All ESP32 control and digital interfaces operate at the board's normal logic levels. No external logic-level converter is used in the current build.

## PCM5102A

The PCM5102A converts the ESP32's digital I²S audio stream into analog stereo audio.

Connections include:

* I²S BCLK
* I²S LRCK / word clock
* I²S DATA
* Ground
* 5 V power
* Left analog output
* Right analog output

The implementation does not require a separate MCLK/SCK connection.

## TPA3116D2

The TPA3116D2 is the Class-D power amplifier. It receives analog stereo audio from the PCM5102A and drives the passive speakers.

The amplifier is powered directly from the 24 V supply.

## SSD1306 OLED

The SSD1306 provides local status and playback metadata display.

The display is powered from the ESP32 board's **3.3 V supply**. Its I²C communication uses the ESP32's normal **3.3 V logic levels**. No logic-level converter is installed or required between the ESP32 and display.

---

# Power System

The project uses a **24 V DC primary supply**.

```text
24 V DC Power Supply
        │
        ├──────────────► TPA3116D2 amplifier
        │
        ▼
   24 V → 5 V Buck
     Converter
        │
        ├──────────────► ESP32-S3 5 V / VIN
        │
        └──────────────► PCM5102A 5 V

ESP32-S3 3.3 V
        │
        └──────────────► SSD1306 OLED
```

The **5 V buck converter powers both the ESP32 and the PCM5102A**. The ESP32's onboard regulation provides its normal 3.3 V rail, which powers the SSD1306 display.

There is no separate logic-level power domain and no level-shifting hardware in the control/data path.

The TPA3116D2 remains on the 24 V supply because it is the high-power portion of the system.

---

# Wiring

## ESP32 → PCM5102A

The ESP32 sends stereo digital audio to the PCM5102A using I²S.

| ESP32 Signal | PCM5102A Signal | Function |
| ------------ | --------------- | -------- |
| BCLK | BCK / BCLK | Bit clock |
| LRCK | LRCK / LCK | Left/right word clock |
| DATA | DIN | Digital audio data |
| GND | GND | Common reference |
| 5 V | VCC | DAC power |

```text
ESP32
 ├── BCLK ─────► PCM5102A BCK
 ├── LRCK ─────► PCM5102A LRCK
 └── DATA ─────► PCM5102A DIN
```

Do not add an SCK/MCLK connection simply because the DAC board exposes a pin for it.

## ESP32 → SSD1306

The OLED is connected directly to the ESP32's I²C interface.

* ESP32 3.3 V → OLED VCC
* ESP32 GND → OLED GND
* ESP32 SDA → OLED SDA
* ESP32 SCL → OLED SCL

All of these control/data signals use the ESP32's normal 3.3 V logic level. **No logic-level converter is used.**

## PCM5102A → TPA3116D2

```text
PCM5102A                    TPA3116D2

LEFT OUT  ─────────────────► INL
RIGHT OUT ─────────────────► INR
GND       ─────────────────► GND
```

Keep analog signal wiring reasonably short and separated from high-current amplifier wiring where practical.

## TPA3116D2 → Speakers

```text
TPA3116D2

L+ / L- ─────────► Left passive speaker
R+ / R- ─────────► Right passive speaker
```

**Do not connect either speaker negative terminal directly to system ground.** The TPA3116D2 uses bridged amplifier outputs.

---

# Physical Wiring Reference

The prototype wiring uses the following wire-color convention.

## ESP32 / amplifier-side wiring

| Signal | Wire Color |
| ------ | ---------- |
| 5 V / VIN | Green |
| GND | Blue |
| LRCK | Purple |
| DIN / DATA | Gray |
| BCK | White |
| SCK / MCLK | Black — not used |

## DAC → amplifier

| Signal | Wire Color |
| ------ | ---------- |
| INL / Left | White |
| GND | Black |
| INR / Right | Red |

These colors document the prototype wiring and are not electrical requirements.

---

# ESP32 Pin Planning

The ESP32-S3 DevKitC-1 has different accessible GPIOs than earlier ESP32 boards used in other projects.

**Do not assume GPIO25/26/27 are available on this hardware.**

The YAML in this repository is the authoritative source for the current GPIO assignment.

---

# Controls and Logic

There is **no physical rotary encoder or other local control input** in the current hardware build.

Control logic is implemented by the ESP32 and the connected software ecosystem:

* Home Assistant provides automation, status, and control.
* Music Assistant provides music selection and network playback.
* The ESP32 handles the endpoint logic, audio processing, display updates, and software-controlled playback/volume functions.
* The SSD1306 provides local visual status and metadata feedback.

All control and digital interfaces remain at the ESP32's standard board logic level. **No logic-level converter is used between the boards.**

---

# Software

## ESPHome

[ESPHome](https://esphome.io/) is used as the firmware/configuration environment.

ESPHome provides:

* ESP32 firmware generation
* Wi-Fi
* OTA updates
* Home Assistant integration
* I²S audio support
* I²C display support
* diagnostic sensors

The project configuration is maintained as YAML.

## Home Assistant

[Home Assistant](https://www.home-assistant.io/) provides the overall home automation environment.

It can provide:

* Volume control
* Playback control
* Automation
* Room grouping
* Status
* Diagnostics
* Remote control

## Music Assistant

[Music Assistant](https://music-assistant.io/) is the primary music-management layer.

The amplifier is intended to become another Music Assistant playback endpoint. The amplifier does not need to know where the music originated.

Supported sources can include services such as Spotify, Tidal, Qobuz, Apple Music, local files, network files, and other Music Assistant providers.

---

# Network Synchronization

Synchronized playback is a primary project requirement.

```text
Music Assistant
       │
       ├────────► Indoor Player
       │
       ├────────► Outdoor Player
       │
       └────────► Other Network Players
```

### Sendspin

Sendspin is the preferred network-audio direction for the ESP32 implementation where supported.

The project should retain flexibility around the network audio protocol as the ecosystem evolves.

VBAN has been considered as a possible short-term network source for applications such as a future turntable integration, but it is not the primary architecture of this amplifier.

---

# Audio Pipeline

```text
Music Assistant / Network Source
              │
              ▼
          ESP32-S3
              │
          I²S digital
              ▼
          PCM5102A
              │
        Analog L / R
              ▼
          TPA3116D2
              │
       Amplified stereo
              ▼
       Outdoor speakers
```

The ESP32 is the network endpoint and digital-audio processor. The PCM5102A performs digital-to-analog conversion, and the TPA3116D2 provides the speaker power stage.

---

# Why This Is Not Just a Bluetooth Amplifier

A Bluetooth amplifier would make the phone the source and would make multi-room synchronization more difficult.

This project instead treats the amplifier as a **network audio endpoint**. A phone or tablet can act as a controller while the network audio system remains responsible for the actual audio transport.

---

# Future Architecture

The current amplifier should remain focused on network playback through Music Assistant/Sendspin.

Future work may add a network-based source/announcement architecture without changing the fundamental ESP32 → DAC → amplifier audio path.

Possible future sources, such as a network-connected turntable, should feed the network audio architecture rather than adding an unnecessary local analog source path to the amplifier itself.

---

# Bill of Materials

## Required

* ESP32-S3 DevKitC-1
* PCM5102A stereo DAC
* TPA3116D2 stereo Class-D amplifier
* SSD1306 OLED
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

# Initial Assembly Procedure

## 1. Verify the hardware

Before applying power, verify the ESP32-S3, DAC, OLED, amplifier, buck converter, power supply, and speakers. Confirm the voltage requirements of every module.

## 2. Configure the buck converter

Set the buck converter output to the required regulated **5 V** before connecting the ESP32 or DAC. Verify the output with a multimeter.

## 3. Test the ESP32

Power the ESP32 from the regulated 5 V supply and verify boot, Wi-Fi, ESPHome logs, and OTA operation.

## 4. Connect the DAC and OLED

Connect the DAC I²S signals and 5 V supply. Connect the SSD1306 to the ESP32's 3.3 V/I²C interface. No logic-level converter is used.

## 5. Connect the amplifier

With power removed, connect the DAC analog outputs, signal ground, amplifier power, and speakers. Double-check amplifier output wiring before applying power.

## 6. First audio test

Start at low volume and verify left/right channel operation, absence of excessive hum or distortion, and reasonable amplifier temperature.

---

# Troubleshooting

## No audio

Check:

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

## Hum or buzzing

Check common ground, power supply quality, buck converter noise, audio cable routing, amplifier grounding, and separation between high-current power wiring and analog audio wiring.

## Distorted audio

Check amplifier gain, software volume, DAC output level, speaker impedance, supply voltage, amplifier temperature, and clipping.

## ESP32 resets when music starts

Check buck converter current capability, 5 V voltage under load, ESP32 supply stability, common-ground wiring, amplifier power wiring, and electrical noise from the amplifier.

---

# Safety

This project contains high-current DC power and a high-power Class-D amplifier.

Always:

* Disconnect power before changing wiring.
* Verify polarity before applying power.
* Fuse the supply appropriately.
* Use appropriately sized wire.
* Insulate exposed conductors.
* Provide strain relief.
* Keep conductive objects away from powered circuitry.
* Provide adequate amplifier ventilation.
* Use an enclosure suitable for the outdoor environment.
