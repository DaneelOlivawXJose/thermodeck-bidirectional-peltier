# ThermoDeck: Bidirectional Peltier PID Controller

![Status: Active](https://img.shields.io/badge/Status-Active-brightgreen) ![Control: PID](https://img.shields.io/badge/Control-PID-blue) ![Hardware: Power Electronics](https://img.shields.io/badge/Hardware-120W_Power-red) ![Math: MATLAB](https://img.shields.io/badge/SysID-MATLAB-orange)

ThermoDeck is a solid-state thermal management platform designed to actively heat or cool a liquid payload. Unlike standard ON/OFF relay-based coolers, this system implements a continuous proportional-integral-derivative (PID) control loop over a 120W power stage, achieving precise temperature tracking without thermal shock.

![ThermoDeck Hardware](./docs/thermodeck_prototype.jpg)

## 🔧 Hardware Architecture

Driving a Peltier cell with high-frequency PWM directly from a microcontroller is a known recipe for destroying the internal bismuth telluride ceramics. This project solves that at the hardware level:

* **Power Stage:** 12V 120W PSU -> BTS7960 H-Bridge (43A) -> **10A Toroidal Inductor** -> TEC1-12706 Peltier. The inductor acts as an LC filter (along with the cell's parasitic capacitance) to smooth the PWM into a nearly flat DC voltage, preserving the Peltier's lifespan and efficiency.
* **Primary Sensing (Process Variable):** An MLX90614-BCC (35º FOV) I2C Infrared sensor. Pointed directly at the fluid at a 45-degree angle, it avoids the thermal mass lag of the cup and completely eliminates condensation/waterproofing issues.
* **Safety Limits:** A 10k NTC thermistor is mounted via a 3.3V voltage divider directly to the 60x60x39mm extruded aluminum heatsink. If the heatsink exceeds 65ºC (due to fan failure or high ambient temps), the ESP32 triggers a hardware safety shutdown.
* **Aerodynamics:** The 6028 Blower fan is coupled to the heatsink via a custom Onshape-designed divergent duct to maximize static pressure through the cooling fins.

## 📈 System Identification & Dual PID Tuning

Thermoelectric coolers suffer from inherent physical asymmetry: **Joule Heating ($I^2R$)**. 
When cooling, the Peltier must pump the payload's heat *and* evacuate its own electrical waste heat. When heating, that same waste heat *assists* the pumping action. This makes the system extremely non-linear if treated as a single plant.

To solve this, I performed an Open-Loop Step Response and modeled two separate First-Order Plus Dead Time (FOPDT) transfer functions in MATLAB:

$$G(s) = \frac{K}{\tau s + 1} e^{-L s}$$

* **Cooling PID:** Slower response (larger $\tau$), requires aggressive integral action.
* **Heating PID:** Extremely fast response (smaller $\tau$, high $K$), requires heavy derivative dampening to prevent overshoot.

## 🚀 How to Execute & Replicate

### Phase 1: Plant Characterization (Data Gathering)
Before running the PID, you need to model your specific thermal mass (your cup + water).
1. Flash the ESP32 with the `open_loop_step.cpp` (found in the `/firmware/sysid` folder).
2. Place a room-temperature cup of water on the cold plate.
3. The ESP32 will apply a constant 50% PWM step and print `Time, PWM, IR_Temp, Heatsink_Temp` to the serial port.
4. Let it run for 15 minutes. Save the Serial Monitor output as `step_data.csv`.

### Phase 2: MATLAB Tuning
1. Open MATLAB and run `sysid_fopdt_tuner.m` located in the `/matlab` directory.
2. The script will import `step_data.csv`, plot the S-curve, calculate the tangent (Ziegler-Nichols method), and output the optimal $K_p$, $K_i$, and $K_d$ gains for both heating and cooling regimes.

### Phase 3: Closed-Loop Execution
1. Open `/firmware/src/main.cpp`.
2. Input your MATLAB-calculated gains into the `cooling_pid_config` and `heating_pid_config` structs.
3. Build and upload to the ESP32:
\`\`\`bash
cd firmware
pio run -t upload
\`\`\`
4. Open the Serial Monitor. Send a target temperature (e.g., `set_temp 15.0`). The dual-PID algorithm will automatically select the H-Bridge polarity and begin modulating the PWM to reach the setpoint.

---
*Developed by [Tu Nombre/Usuario]*
