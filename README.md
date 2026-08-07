# Skynet-AirMultiSensor

<img src="Images/IMG_0815.jpeg" alt="alt text" width="500">

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
- **Runtime configuration via ESPHOME** — no reflashing needed to change display mode, rotation speed, sensor polling interval, °F/°C, or backlight brightness
- **Bluetooth proxy** — active proxy with 3 connections plus continuous BLE tracking, extending Home Assistant's Bluetooth range
- **WiFi fault indicator** — onboard RGB LED blinks red every 5 seconds when disconnected
- **CO2 calibration to atmostpheric pressure**
- **Temperature Accuracy** - provides a fused temperature reading based on the published performance of the three temperature sensors.  Available calibration routine for adjusting for device-related heating.  Thermal considerations and isolation were a big part of the case design.
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
| M3x4 and M2x4 screws | - | - | - |
| USB-C power cable | - | - | - |

---

## Safety notice

**This is not a safety device.** The MICS-4514 is an uncalibrated metal-oxide sensor whose readings drift substantially with temperature and humidity; its ppm figures are useful as relative trends, not absolute concentrations. Do not rely on the carbon monoxide reading for life safety — use a certified CO alarm. The same applies to the methane channel and gas leak detection.

---

## Wiring

You will be making two different I2C buses.  The SCD41 and 4515 chips are on one, and the SEN55 is by itself on the other.

There should be cables that came with the SCD41 grove chip, as well as the SEN55 Gravity sensor.  Below are the colors that I had:

Grove cable (for the Sen55) - white = SDA, Yellow = SCL, Red = 5V, Black = Ground

Gravity cable (for the SCD41) - green = SDA, Blue = SCL, Red = 3.3V, Black = Ground

1. Bus 1: Splice together and I2C "bus" for the SCD41 and Gravity 4515 chips (connect power, ground, SDA, SCL of the two sensors, and also connect them to jumpers that end in breadboard connectors to plug into the ESP32 pins).  SCL for this bus goes to RX (GPIO17), and SDA - GPIO26
2. Bus 2: Make a second I2C "bus" for the gravity SEN55 sensor - Can use the Gravity cable that comes with the chip and cut off the non-gravity side and splice onto breadboard connectors, or purchase the premade cable (link above). SCL for this bus goes to GPIO22, SDA - GPIO21
3. Power Bus#1 goes to 3.3V pin, Bus#2 goes to 5V, both buses ground goes to ground.

4. Make sure to solder the pins to the SCD41 board to the backside (opposite the sensor)
5. DHT22: 3.3V, GND, signal to GPIO25

6. Display hookup (these are the colors of the cable that came with my display, YMMV):
DC - blue - GPIO4
CS - yellow - GPIO5
RST - brown - GPIO27
CLK - orange - GPIO18
DIN (MOSI) - green - GPIO23
VCC - purple - 3.3V
GND - white - GND
BL - grey - GPIO32


## Assembly

<img src="Images/IMG_0808.jpeg" alt="alt text" width="300">

1. Attach Display with M2x4 screws.  Plug goes on top.  Use a allen wrench key through the back holes to get the bottom two screws in.
2. Place SCD41, secure with two-sided tape on the front-facing side below the sensor box.
3. Attach DHT22 above it (with jumpers attached already) with M3x4 screws
4. Attach MICS with M2x4 screws
5. Slide the ESP32 in the center section, under the display, usb facing the hole and pins pointing up
6. Attach USB-C power cable into ESP32 through back hole.
7. Make sure the sensor cables that travel from the center section to the left/righ sections are sitting in the channel gaps in the separating walls before you put the lid on to prevent them being pinched.

<img src="Images/IMG_0811.jpeg" alt="alt text" width="300">

## Software/Firmeware Installation

1. Install [ESPHome](https://esphome.io/) (2026.7.3 or later — the config uses `mipi_spi`).
2. Add `wifi_ssid` and `wifi_password` to your `secrets.yaml`.
3. Replace the API encryption key, OTA password, and `use_address` with your own.
4. Flash over USB the first time. **USB flashing is required at least once** — `sram1_as_iram: true` needs a bootloader from ESP-IDF 5.1 or later, and bootloaders do not update over OTA.

### Notes on first boot

- The MICS-4514 needs several minutes of heater warm-up before it reports; expect `--` on those values initially.
- The SEN55 VOC and NOx algorithms need hours to establish a baseline. Early index values are not meaningful.
- If the display renders as a photo negative, flip `invert_colors` in the display block. Roughly half of ST7789V panels want it one way, half the other.

---

## Home Assistant Integration


<img src="Images/controls.png" alt="alt text" width="300">

1) Diplay backlight - turning this off will blank display
2) Display mode:
    - All 16 - all 16 sensors on a single screen, but small
    - Rotating - 4 sensors on each screen (bigger), rotating through 4 pages (speed adjustable)
    - Highlights - 4 sensors on single screen (bigger) - temp, VOC, PM2.5, and CO2.

<img src="Images/IMG_0815.jpeg" alt="alt text" width="300">
<img src="Images/IMG_0820.jpeg" alt="alt text" width="200">


3) Status LED - there is an on-chip light that is controlable, but probably not visable from outside the case.

---

<img src="Images/configuration.png" alt="alt text" width="300">

1) Display Fahrenheit - If on, the display will show temp in F.  If not, C.  (both are transmitted to home assistant)
2) Display Rotation Speed - if on "rotating" diplay mode, this will adjust how fast the pages will cycle
3) Restart - will reboot ESP
4) Sensor update - adjust the update speed of all the sensors.  10s is the minimum.

---

<img src="Images/sensors.png" alt="alt text" width="300">

1) Sensor data exposed to home assistant.
2) Wifi data also available "Diagnostic" panel - not pictured.

---

## Calibrations

1) Set Ambient Pressure: The SCD41 relies on accurate atmospheric pressures to report CO2 - not setting this can cause variations up to 20%.  Use a home assistant automation to pull current atmospheric pressure from some other sensor (or web API) in mbar and push it to the sensor.  Set an automation with a time trigger ~ 1 hr with the action:

```yaml
action: esphome.skynet_multisensor_set_ambient_pressure
data:
  pressure_mbar: "{{ states('sensor.air_pressure') | int(0) }}"
```

2) Temperature bias - there are three sensors that report temps (DHT22, SCD41, SEN55).  Care was taken to try to thermally isolate, but they still may experience local heating as the whole sensor package warms up over time.  Let the device become room temperature (off for an hour) and then start it up.  Under the Diagnostic window in home assistant you'll find Boot temps and Warm temps (after 30 min) from the sensors, and a Bias estimate which is how much the temp rose (assuming the room stayed the same over those 30 minutes).  Put these values in the esphome YAML file, at the top under substiutions you'll find "bias_dht", "bias_scd", and "bias_sen".  Bias_SCD has a manufacture default of 4C.  If you measure something different, replace this (don't add/subtract from it).

3) If you REALLY want you can calibrate the relative humidity with a salt calibration, these values are also in the YAML "rhbias_dht", etc.

4) SCD41 CO2 Automatic self-calibration assumes the sensor sees ~400 ppm background weekly. If this lives in a room that never fully vents, ASC will drag your baseline wrong and you won't know. There is a forced-calibration button (target 420 ppm, after 3+ min outdoors) so you can correct it without reflashing.

5) The Sen55 will autoclean weekly.  There is a button exposed to home assistant if you would like to manually trigger a clean (the fan spins up to ~10s at full speed to blow it clear.)
---

## References

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

## License

[PolyForm Noncommercial License 1.0.0](LICENSE) — free for personal, hobby, educational, research, and nonprofit use. Commercial use is not permitted.
