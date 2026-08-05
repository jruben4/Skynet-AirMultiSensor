# Skynet-AirMultiSensor
# Skynet Multisensor

A ESPHome air-quality and climate monitor built on an ESP32-WROOM board, combining four sensors across two I²C buses: a DHT22 for temperature and humidity (with absolute humidity derived in firmware), a Sensirion SCD41 for true NDIR CO2, a Sensirion SEN55 for PM1.0/2.5/4.0/10 particulates plus VOC and NOx indices, and a MICS-4514 for nitrogen dioxide, carbon monoxide, hydrogen, ethanol, methane, and ammonia — sixteen live values in total, all exposed to Home Assistant over the encrypted ESPHome API. A 240×320 ST7789V SPI display shows the readings in one of three user-selectable modes, with every value color-coded green, amber, or red against configurable thresholds. Display mode, page rotation speed, sensor polling interval, temperature units, and backlight brightness are all adjustable at runtime from Home Assistant without reflashing. The device doubles as an active Bluetooth proxy and BLE tracker.

---

## Case 3D Model
https://makerworld.com/en/models/3134930-skynet-air-multisensor

## Features

- **16 live measurements** across temperature, humidity, CO2, particulates, VOC/NOx, and six trace gases
- **Three display modes**, switchable from Home Assistant:
  - **All 16** — dense single page, every value at once
  - **Rotating** — four pages of four values in large type, auto-advancing, with page indicator dots
  - **Highlight** — static page showing temperature, VOC, CO2, and PM2.5 only
- **Threshold color coding** — green below moderate, amber above moderate, red above high; thresholds drawn from EPA AQI breakpoints, WHO guidance, and OSHA/NIOSH exposure limits
- **Runtime configuration** — no reflashing needed to change display mode, rotation speed, sensor polling interval, °F/°C, or backlight brightness
- **Bluetooth proxy** — active proxy with 3 connections plus continuous BLE tracking, extending Home Assistant's Bluetooth range
- **WiFi fault indicator** — onboard RGB LED blinks red every 5 seconds when disconnected
- **Graceful missing data** — sensors that haven't reported yet show `--` in grey rather than `nan`

---

## Hardware

| Component | Part | Interface | Link |
|---|---|---|---|
| MCU | Freenove ESP32-WROOM (ESP32 rev3.1, 4MB flash, no PSRAM) | — | https://www.amazon.com/dp/B0C9THDPXP |
| Temp / humidity | DHT22 (AM2302) | 1-Wire | https://www.amazon.com/dp/B0CPHQC9SF |
| CO2 | Sensirion SCD41 | I²C bus A | https://www.amazon.com/dp/B0GWQMQVN8 |
| Particulate / VOC / NOx | Sensirion SEN55 | I²C bus B | https://www.seeedstudio.com/Grove-All-in-one-Environmental-Sensor-SEN55-p-5373.html |
| Trace gases | MICS-4514 | I²C bus A | https://www.amazon.com/dp/B09G9PZ4XZ |
| Display | ST7789V 240×320 IPS, SPI | SPI (VSPI) | https://www.amazon.com/dp/B082GFTZQD |
| Grove breakout cable (optional) | — | — | https://www.amazon.com/dp/B074MDM36N |

---

## Pinout

### I²C — two buses

Two separate buses are used because the SEN55 and SCD41 both benefit from short, clean runs, and splitting them keeps each bus lightly loaded.

| Bus | Signal | GPIO | Devices |
|---|---|---|---|
| **Bus A** | SDA | `GPIO26` | SCD41, MICS-4514 |
| **Bus A** | SCL | `GPIO17` | |
| **Bus B** | SDA | `GPIO21` | SEN55 |
| **Bus B** | SCL | `GPIO22` | |

Both buses run at 50 kHz. 

### DHT22

| Signal | GPIO |
|---|---|
| Data | `GPIO25` |

Internal pull-up is enabled in firmware.

### Display — ST7789V over SPI

The ST7789V is write-only, so there is no MISO connection.

| Display pin | GPIO | Notes |
|---|---|---|
| GND | GND | |
| VCC | 3.3 V | **not 5 V** — bare breakouts have no regulator |
| SCL / CLK / SCK | `GPIO18` | SPI clock |
| SDA / DIN / MOSI | `GPIO23` | SPI data — this is MOSI, *not* I²C |
| RES | `GPIO27` | Reset |
| DC | `GPIO4` | Data/command |
| CS | `GPIO5` | Chip select |
| BLK | `GPIO32` | Backlight, LEDC PWM, dimmable |


### Status LED

| Signal | GPIO | Notes |
|---|---|---|
| WS2812 data | `GPIO16` | Onboard on the Freenove board, silkscreened "16" |

This is an addressable RGB LED, not a plain indicator. It must be driven by the `esp32_rmt_led_strip` component — toggling the pin high or low does nothing, and any unrelated signal on GPIO16 (I²C, UART) will be interpreted as color data and light it at full brightness.

### Full GPIO map

| GPIO | Function |
|---|---|
| 4 | Display DC |
| 5 | Display CS |
| 16 | WS2812 status LED |
| 17 | I²C bus A — SCL |
| 18 | SPI CLK |
| 21 | I²C bus B — SDA |
| 22 | I²C bus B — SCL |
| 23 | SPI MOSI |
| 25 | DHT22 data |
| 26 | I²C bus A — SDA |
| 27 | Display RESET |
| 32 | Display backlight (PWM) |

**Free:** 13, 14, 19, 33
**Avoid:** 0, 2, 12, 15 (strapping pins), 6–11 (flash), 34–39 (input-only)

GPIO5 is technically a strapping pin, but it's the ESP32's default VSPI CS and idles high, so it's safe here. ESPHome will warn about it regardless.

---

## Home Assistant entities

### Sensors

| Entity | Unit |
|---|---|
| DHT22 Temperature | °C |
| DHT22 Temperature F | °F |
| DHT22 Humidity | % |
| DHT22 Absolute Humidity | g/m³ |
| CO2 | ppm |
| SD41 Temperature / Humidity | °C / % |
| PM <1µm, <2.5µm, <4µm, <10µm | µg/m³ |
| Sen55 Temperature / Humidity | °C / % |
| VOC / NOx | index |
| Nitrogen Dioxide, Carbon Monoxide, Hydrogen, Ethanol, Methane, Ammonia | ppm |
| WiFi Signal dB / Percent | dBm / % |

### Controls

| Entity | Type | Range / options |
|---|---|---|
| Display Mode | select | All 16 · Rotating · Highlight |
| Display Rotate Seconds | number | 2–60 s |
| Sensor Update Seconds | number | 10–600 s |
| Display Fahrenheit | switch | on / off |
| Display Backlight | light | dimmable |
| Status LED | light | RGB |
| Restart | switch | — |

---

## Thresholds

Set in `substitutions:` at the top of the YAML. Each value has a *moderate* and *high* cutoff.

| Measurement | Moderate | High | Basis |
|---|---|---|---|
| Temperature | 80 °F / 26 °C | 90 °F / 32 °C | comfort |
| Humidity | 60 % | 70 % | comfort / mould risk |
| Absolute humidity | 12 g/m³ | 17 g/m³ | comfort |
| CO2 | 1000 ppm | 1500 ppm | common IAQ guidance |
| PM1.0 / PM2.5 | 12 µg/m³ | 35 µg/m³ | EPA AQI breakpoints |
| PM4.0 | 25 µg/m³ | 50 µg/m³ | interpolated |
| PM10 | 54 µg/m³ | 154 µg/m³ | EPA AQI breakpoints |
| VOC index | 150 | 250 | Sensirion (100 = baseline) |
| NOx index | 20 | 150 | Sensirion (1 = clean) |
| NO2 | 0.05 ppm | 0.10 ppm | WHO |
| CO | 9 ppm | 35 ppm | EPA 8 h / 1 h standards |
| H2 | 50 ppm | 100 ppm | relative |
| Ethanol | 50 ppm | 100 ppm | relative |
| Methane | 1000 ppm | 5000 ppm | ~2 % / ~10 % of LEL |
| Ammonia | 25 ppm | 50 ppm | NIOSH REL / OSHA PEL |

Thresholds are one-sided — they only flag values that are too *high*. A cold room shows green.

---

## Safety notice

**This is not a safety device.** The MICS-4514 is an uncalibrated metal-oxide sensor whose readings drift substantially with temperature and humidity; its ppm figures are useful as relative trends, not absolute concentrations. Do not rely on the carbon monoxide reading for life safety — use a certified CO alarm. The same applies to the methane channel and gas leak detection.

---

## Wiring
Grove cable - white = SDA, Yellow = SCL
Gravity cable - green = SDA, Blue = SCL

1. Splice together I2C bus for the SCD41 and Gravity 4515 chips (power, ground, SDA, SCL).  Can use the Gravity cable that comes with the chip and cut off the non-gravity side.
2. Can either get a grove breakout cable, or cut the end off the grove cable and splice into breadboard pin cable.
3. Make sure to solder the pins to the SCD41 board to the backside (opposite the sensor)

Sen55/Gravity SCL - GPIO22, SDA - GPIO21
Grove/SCD41 SCL - RX (GPIO17), SDA - GPIO26
DHT22: 3.3V, GND, signal to GPIO25

Display hookup (these are the colors of the cable that came with my display, YMMV):
DC - blue - GPIO4
CS - yellow - GPIO5
RST - brown - GPIO27
CLK - orange - GPIO18
DIN (MOSI) - green - GPIO23
VCC - purple - 3.3V
GND - white - GND
BL - grey - GPIO32


## Assembly
1. Attach Display with M2x4 screws.  Plug goes on top.  Use a allen wrench key through the back holes to get the bottom two screws in.
2. Place SCD41, secure with two-sided tape on the front-facing side below the sensor box.
3. Attach DHT22 above it (with jumpers attached already) with M3x4 screws
4. Attach MICS with M2x4 screws
5. Slide the ESP32 in the center section, under the display, usb facing the hole and pins pointing up
6. Attach USB into ESP32 through back hole.

## Installation

1. Install [ESPHome](https://esphome.io/) (2026.7.3 or later — the config uses `mipi_spi`).
2. Add `wifi_ssid` and `wifi_password` to your `secrets.yaml`.
3. Replace the API encryption key, OTA password, and `use_address` with your own.
4. Flash over USB the first time. **USB flashing is required at least once** — `sram1_as_iram: true` needs a bootloader from ESP-IDF 5.1 or later, and bootloaders do not update over OTA.

### Notes on first boot

- The MICS-4514 needs several minutes of heater warm-up before it reports; expect `--` on those values initially.
- The SEN55 VOC and NOx algorithms need hours to establish a baseline. Early index values are not meaningful.
- If the display renders as a photo negative, flip `invert_colors` in the display block. Roughly half of ST7789V panels want it one way, half the other.

---

## Configuration notes

**Runtime handlers must never touch the display.** Any `on_turn_on` / `on_turn_off` / `on_value` handler on a component with `restore_mode` or `restore_value` fires during `setup()`, before the display's framebuffer is allocated. Calling `my_display.update()` from one of those handlers causes a `StoreProhibited` crash. The config sets a `force_redraw` global instead, which a 1 s interval acts on — intervals only run after setup completes.

The same pattern applies to the sensor polling interval, which is applied from a separate 2 s interval rather than the number's `on_value`.

---

## License

[PolyForm Noncommercial License 1.0.0](LICENSE) — free for personal, hobby, educational, research, and nonprofit use. Commercial use is not permitted.
