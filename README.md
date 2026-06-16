# Smart Camper Fridge Controller (Secop / ESPHome)

An advanced, ESPHome-powered smart controller for camper vans and overland vehicles equipped with a **Secop (formerly Danfoss) compressor controller** (e.g., 101N0650). 

Built around the compact **M5Stack AtomS3R** (ESP32-S3) with an integrated color display, this project replaces mechanical thermostats with intelligent, dynamic PWM speed adjustments, live power/voltage tracking, multi-zone temperature monitoring, and a fully customizable bilingual UI (English/German).

---

## Features

* 🧠 **Smart PWM Speed Control:** Dynamically adjusts the compressor speed between 2,500 and 4,400 RPM (scaled linearly from 0% to 100% in the UI) to maximize efficiency and cooling performance.
* 🍦 **Triple-Zone Temperature Monitoring:** Dedicated high-precision DS18B20 sensor buses for the Fridge compartment, Freezer compartment, and the Compressor housing.
* ⚡ **Real-Time Energy Tracking:** Measures live current (via ACS712 Hall-effect sensor) and dynamic battery voltage (via an analog voltage divider). Essential for LiFePO4 batteries where voltage heavily fluctuates during solar or alternator charging. Includes a continuous Watt-hour (Wh) consumption tracker.
* 🇩🇪 🇬🇧 **Bilingual On-The-Fly UI:** Toggle the entire user interface on the physical screen and Home Assistant states between English and German instantly with a single click.
* 🔒 **Compressor Protection Engine:** Includes a protective 30-second "Soft-Start" (holding the compressor at a safe 2,500 RPM) and a configurable anti-short-cycle "Locktime" countdown bar to protect the compressor from pressure-lock damage.
* 🎛️ **GitHub-Ready Calibration Sliders:** Zero hardcoded sensor data. Users can calibrate the voltage divider offsets, current sensor zero-points, and noise-cutoffs directly via Home Assistant sliders.
* 🔌 **Galvanically Isolated & Relay-Free:** Replaced clicking mechanical relays with optocouplers for silent, highly reliable, and automotive-safe communication with the Secop unit.

---

## Hardware Requirements

### Core Components
* **Microcontroller:** M5Stack AtomS3R (ESP32-S3)
* **Current Sensor:** ACS712 Hall-Effect Sensor Module (20A version recommended)
* **Temperature Sensors:** 3x DS18B20 (Waterproof probe style)

### Discretes & Safety
* **Optocouplers:** 2x PC817 (For isolating DIAG and PWM signals)
* **Resistors:** * 1x 47 kΩ + 1x 10 kΩ (For the 14V -> 3.3V analog voltage divider)
  * 1x 10 kΩ (Strong external pull-up for the DIAG optocoupler output to guarantee noise immunity)
  * Resistors matching your PC817 LED inputs (typically 220 Ohm to 470 Ohm)
  * 3x 4.7 kΩ Pull-up resistors for the DS18B20 data lines

---

## Pin Assignment

| Function | Pin (GPIO) | Type | Notes |
| :--- | :--- | :--- | :--- |
| **Fridge Temp** | GPIO5 | OneWire Bus | Temperature inside fridge compartment |
| **Freezer Temp** | GPIO6 | OneWire Bus | Temperature inside freezer compartment |
| **Compressor Temp** | GPIO7 | OneWire Bus | Monitor compressor surface temperatures |
| **Current Meter** | GPIO1 | Analog In (ADC) | Connected to ACS712 `OUT` pin |
| **Voltage Meter** | GPIO2 | Analog In (ADC) | Connected to the 47k/10k Resistor Divider |
| **Secop DIAG** | GPIO38 | Digital In | Reads blink codes from Secop Error Terminal |
| **Secop PWM** | GPIO39 | LEDC Out | Generates the 5kHz Speed Signal for Secop |
| **Built-in Button**| GPIO41 | Binary Sensor | Multi-click interface for display navigation |

---

## Wiring & Schematic

### 1. High-Current DC Path (12V/24V)
* Connect the main battery positive (+) through a fuse to the **IP+** terminal of the ACS712.
* Connect **IP-** of the ACS712 directly to the **[+]** terminal of the Secop controller.
* Connect Battery minus (-) directly to the **[-]** terminal of the Secop controller.

### 2. Voltage Divider (Battery Voltage Sense)
Connect the Battery positive (+) to the 47 kΩ resistor. Bridge the 47 kΩ and 10 kΩ resistors into **GPIO2**. Connect the other side of the 10 kΩ resistor to **GND**.

> ⚠️ **CRITICAL NOTE:** The ground (GND) of the 12V Battery/Power supply and the GND of the AtomS3R **must** be tied together for the analog voltage and current inputs to share a valid electrical reference plane!

### 3. Optocoupler Isolation (Secop Interface)
* **PWM Output:** AtomS3R GPIO39 drives the internal LED of the first PC817 (via a current limiting resistor). The transistor side connects directly to Secop terminals **T** and **C**.
* **DIAG Input:** Secop terminal **D** drives the second PC817 input. The output transistor pulls **GPIO38** down to GND when active. 

---

## Software Installation

1. Copy the contents of `camper-fridge.yaml` into your ESPHome environment.
2. Create a `secrets.yaml` file in the same directory (or use your existing one) containing your network and access parameters based on the following template:

```yaml
wifi_ssid: "Your_Camper_WIFI"
wifi_password: "Your_Secret_Password"
wifi_backup_ssid: "Your_Backup_Home_WIFI"
wifi_backup_password: "Your_Home_Password"
ap_password: "Fallback_Access_Point_Password"
```

3. Compile and flash the firmware to your AtomS3R. If transitioning between compilation environments, performing a **"Clean Build"** prior to flashing is recommended.

---

## UI Calibration Guide

Once the controller boots up and integrates into Home Assistant, you will find four unique calibration entities under your device settings. This allows you to configure your controller perfectly on any custom PCB design without changing the code:

1. **Calibration: Voltage-Multiplier:** Measure your real battery voltage with a multimeter. Adjust this slider until the `Fridge Voltage` entity matches your multimeter exactly.
2. **Calibration: Current Zero Voltage (V):** Turn off the fridge entirely (0 Amperes flowing). Check the raw voltage readout of your current sensor. Adjust this slider until your `Fridge Current` sits solidly at `0.00 A`.
3. **Calibration: Current Gain (A/V):** Defines the sensitivity of your Hall sensor. The standard value of `10.0` is optimized for the **20A version** of the ACS712 (100 mV/A). If you use a 30A version (66 mV/A), adjust this value to `15.1`.
4. **Calibration: Current Cutoff (A):** Hall sensors suffer from minor ambient magnetic noise. Any value below this threshold (Default: `0.30 A`) will be forced straight to `0.00 A` in standby to keep your metrics perfectly clean.

---

## User Interface (On-Device Controls)

The built-in screen button (GPIO41) allows direct navigation if your smartphone or Home Assistant dashboard is offline:
* **Single Short Click:** Rotates through Target Temperature Presets (*Fresh -> Cold -> Eco -> Vacation*).
* **Double Click:** Cyclically alternates Compressor Profiles (*Night -> Normal -> Power*).
* **Long Press (1 Second):** Toggles the Main Switch ON or OFF (shuts down the display backlight and compressor safely).

---

## License & Contributing

Free to fork this project, report issues, or submit pull requests. Let's make mobile off-grid living more efficient together! 

Developed by [ubarni](https://github.com/ubarni). Distributed under the MIT License.
