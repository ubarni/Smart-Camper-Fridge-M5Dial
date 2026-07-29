# Smart Camper Fridge Controller (M5Dial)

An advanced, fully integrated open-source controller for 12V/24V camper refrigerators using SECOP (Danfoss) compressors, powered by an [**M5Stack Dial 1.1**](https://docs.m5stack.com/en/core/M5Dial) running **ESPHome**.

This project provides a modern, touch-enabled smart UI for your fridge, replacing analog thermostats with precise digital temperature management, dynamic compressor speed control, and built-in battery protection.

<img width="1063" height="858" alt="Bildschirmfoto 2026-07-29 um 21 31 03" src="https://github.com/user-attachments/assets/0ed21991-6cd3-4f77-aba5-3ce099350175" />

## ✨ Features

* **Modern Touch UI & Rotary Encoder:** Intuitive control over temperature and compressor speeds using the M5Dial's rotating bezel and round touch screen.
* **Intelligent Compressor Control:** Variable PWM speed control allows you to run the compressor in `Night` (Quiet/Slow), `Normal`, or `Power` (Fast cooling) modes.
* **Smart Soft-Start:** Gently spins up the compressor to reduce mechanical wear and voltage drops.
* **Low Voltage Disconnect (LVD):** Monitors battery voltage and safely shuts down the fridge before your camper's battery is depleted (Configurable via Home Assistant).
* **Failsafe Mechanisms:** Prevents infinite compressor loops if a temperature sensor fails or gets disconnected.
* **SECOP Error Decoding:** Directly reads and decodes the diagnostic pulses from the SECOP controller (e.g., "E3: Motor failure") and displays the exact cause on the screen.
* **Software Power Meter:** Accurately estimates current power consumption (Watts) and integrates it into total Energy (Wh) used.
* **100% Home Assistant Integration:** All presets, parameters, and logs are seamlessly synced with Home Assistant.

## 🛠️ Hardware Requirements

* **M5Stack M5Dial 1.1** (ESP32-S3 with integrated touch display and rotary encoder)
* **SECOP Compressor Controller** (e.g., 101N0212 for BD35F/BD50F)
* **DS18B20 Temperature Sensors** (Up to 3 sensors supported for Fridge, Freezer, and Compressor ambient)
* **Voltage Divider** (To safely measure the 12V/24V battery line via the M5Dial's 3.3V ADC)
* **Optocouplers:** 2x PC817 (For isolating DIAG and PWM signals)
* **Resistors:** * 1x 47 kΩ + 1x 10 kΩ (For the 14V -> 3.3V analog voltage divider)
  * 1x 10 kΩ (Strong external pull-up for the DIAG optocoupler output to guarantee noise immunity)
  * Resistors matching your PC817 LED inputs (typically 220 Ohm to 470 Ohm)
  * 1x 4.7 kΩ Pull-up resistors for the DS18B20 data line

---

## 🔌 Wiring (Grove Ports)

The code is pre-configured to utilize the easily accessible Grove ports on the back of the M5Dial:

* **Port A (Red - I2C/GPIO):**
  * `GPIO15` (Yellow Wire): PWM Output to SECOP Controller (Speed limit control).
  * `GPIO13` (White Wire): Input for SECOP Error diagnostic pulses.
* **Port B (Black - I/O/ADC):**
  * `GPIO2` (Yellow Wire): Shared OneWire Data Pin for ALL DS18B20 Temperature Sensors.
  * `GPIO1` (White Wire): ADC Input for Battery Voltage Divider.
---

## Wiring & Schematic

### 1. High-Current DC Path (12V/24V)
* Connect the main battery positive (+) and negative (-) through a fuse to the M5Dial terminal. Voltage input range is 6 ~ 36V DC.

### 2. Voltage Divider (Battery Voltage Sense)
Connect the Battery positive (+) to the 47 kΩ resistor. Bridge the 47 kΩ and 10 kΩ resistors into **GPIO1**. Connect the other side of the 10 kΩ resistor to **GND**.

> ⚠️ **CRITICAL NOTE:** The ground (GND) of the 12V Battery/Power supply and the GND of the M5Dial **must** be tied together for the analog voltage and current inputs to share a valid electrical reference plane!

### 3. Optocoupler Isolation (Secop Interface)
* **PWM Output:** M5Dial GPIO39 drives the internal LED of the first PC817 (via a current limiting resistor). The transistor side connects directly to Secop terminals **T** and **C**.
* **DIAG Input:** Secop terminal **D** drives the second PC817 input. The output transistor pulls **GPIO38** down to GND when active. 

## Wiring "Diagram"
<img width="1220" height="689" alt="Smart Camper Fridge M5Stack Dial" src="https://github.com/user-attachments/assets/6f382df6-070e-4aa6-944d-a062819fc44a" />

---

## Software Installation

1. Copy the contents of `smart-camper-fridge-m5dial.yaml` into your ESPHome environment.
2. The power estimation logic (`fridge_power`) is tuned for a standard SECOP BD35F compressor pulling ~57W at minimum speed and ~72W at full throttle. You can modify these values in the YAML file based on your specific compressor and cooling agent.

3. Create a `secrets.yaml` file in the same directory (or use your existing one) containing your network and access parameters based on the following template:

```yaml
wifi_ssid: "Your_Camper_WIFI"
wifi_password: "Your_Secret_Password"
wifi_backup_ssid: "Your_Backup_Home_WIFI"
wifi_backup_password: "Your_Home_Password"
ap_password: "Fallback_Access_Point_Password"
```

3. Compile and flash the firmware to your M5Dial. If transitioning between compilation environments, performing a **"Clean Build"** prior to flashing is recommended.

---
## 🖱️ User Interface (M5Dial Operation)

The M5Dial offers a highly intuitive, tactile interface using its rotary encoder, touch screen, and integrated physical button. The UI is split into three main modes, color-coded for quick recognition: **Temperature** (Blue), **Compressor Speed** (Orange), and **Brightness** (Yellow).

### 🔄 Rotary Ring (Bezel)
Turning the physical outer ring allows you to manually adjust the target value of the currently active mode:
* **In Temperature Mode:** Adjusts the target temperature (1°C to 20°C).
* **In Compressor Mode:** Adjusts the compressor speed limit (0% to 100%).
* **In Brightness Mode:** Adjusts the screen backlight intensity (10% to 100%).
* *Note: Manually turning the dial will instantly switch the active preset to "Manual" (unless the exact value matches a predefined preset).*

### 👆 Touch Screen Gestures
* **Tap the Center Screen:** Cycles through the three main UI modes (Temperature -> Compressor Speed -> Brightness).
* **Tap the Bottom Area:** Toggles the bottom information bar between two pages:
  * *Page 1:* Sensor Temperatures (Fridge, Freezer, Compressor).
  * *Page 2:* Energy & Connectivity (Power Draw in Watts, Battery Voltage, WiFi Signal Strength).

### 🔘 Physical Button (Pressing the Screen)
The entire screen of the M5Dial acts as a physical push button, enabling quick preset changes without looking:
* **Single Click:** Cycles through your configured presets for the currently active mode.
  * *In Temperature Mode:* Cold -> Fresh -> Eco -> Vacation -> ...
  * *In Compressor Mode:* Night -> Normal -> Power -> ...
  * *In Brightness Mode:* 30% -> 70% -> 100% -> ...
* **Double Click:** Cycles through the three main UI modes (serves as a tactile alternative to tapping the center screen).
* **Long Press (Hold for 1s):** Main Power Switch. Turns the entire fridge system (and the display) ON or OFF (Standby).

---

## UI Calibration Guide

Once the controller boots up and integrates into Home Assistant, you will find four unique calibration entities under your device settings. This allows you to configure your controller perfectly on any custom PCB design without changing the code:

## ⚙️ Calibration & Configuration

The system exposes a comprehensive set of configuration entities directly to Home Assistant. This allows you to fine-tune the fridge's behavior on the fly without having to recompile or flash the ESP32.

### General Settings
* **Low Voltage Cutoff (LVD):** Sets the minimum battery voltage (e.g., `11.5V`) before the fridge automatically shuts down to protect your camper's battery.
* **Startup Lock Duration:** Defines the minimum wait time (in seconds, default `30s`) between compressor cycles to equalize system pressure and protect the compressor motor.
* **Hysteresis:** The temperature tolerance (e.g., `1.5°C`). If target is 5°C, it cools until 5°C and waits until it reaches 6.5°C before starting again.

### Temperature Presets
You can define the exact target temperatures for the UI presets:
* **Temp Preset Cold:** (Default `2°C`)
* **Temp Preset Fresh:** (Default `4°C`)
* **Temp Preset Eco:** (Default `6°C`)
* **Temp Preset Vacation:** (Default `10°C`)

### Compressor Speed Presets
You can configure the exact PWM output limits (0-100%) for the different compressor modes:
* **Compressor Preset Night:** Ultra-quiet, slow running (Default `20%`)
* **Compressor Preset Normal:** Standard operation (Default `50%`)
* **Compressor Preset Power:** Fast cooling (Default `80%`)

### Hardware Calibration
* **Voltage-Multiplier:** Adjusts the ADC reading to match your physical voltage divider circuit. Measure your battery with a multimeter and adjust this multiplier until the M5Dial matches the real voltage.
* **Sensor Assignment:** The system uses 3 physical DS18B20 OneWire sensors. Since OneWire addresses can change, you can dynamically assign which index (`Index 0`, `Index 1`, `Index 2`) corresponds to the `Fridge`, `Freezer`, or `Compressor` directly via Home Assistant dropdown menus!

### Power Estimation Tuning
Because the system (no more GPIOs) lacks a physical shunt resistor, power consumption is estimated based on the compressor speed. 
If you want to calibrate this to your specific compressor, measure the consumption at lowest (0%) and highest (100%) speed using a SmartShunt. Then, update the lambda in the `fridge_power` sensor within the YAML file:
```yaml
// Example: ~57W at minimum speed, ~72W at full throttle (15W span)
return 57.0f + ((speed / 100.0f) * 15.0f);
```
<img width="423" height="563" alt="Bildschirmfoto 2026-07-29 um 21 32 17" src="https://github.com/user-attachments/assets/8bd5f361-3a0b-4965-b364-a2c43cdb34aa" />
<img width="426" height="482" alt="Bildschirmfoto 2026-07-29 um 21 32 30" src="https://github.com/user-attachments/assets/a654291a-b1b7-460f-bf8a-5c9eb69df57d" />
<img width="428" height="935" alt="Bildschirmfoto 2026-07-29 um 21 32 51" src="https://github.com/user-attachments/assets/8c6d1983-3c4b-4779-85a3-ea04c35f614c" />
<img width="428" height="675" alt="Bildschirmfoto 2026-07-29 um 21 33 02" src="https://github.com/user-attachments/assets/dd6eaded-26b0-4079-b5ca-841311bf066d" />

## License & Contributing

Free to fork this project, report issues, or submit pull requests. Let's make mobile off-grid living more efficient together! 
Developed by [ubarni](https://github.com/ubarni). Distributed under the MIT License.
