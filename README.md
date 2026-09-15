# AM32 High-Current BLDC ESC

A custom three-phase **Electronic Speed Controller** for sensorless brushless (BLDC) motors, designed from the ground up to run the open-source [AM32 firmware](https://github.com/am32-firmware/AM32). Built for robotics, UAVs and underwater vehicles (AUV/ROV thrusters) where you need a compact controller that can push serious current.

<p align="center">
  <img width="778" alt="ESC PCB render" src="https://github.com/user-attachments/assets/6ca646cd-5e5d-49a4-a805-81319fa64ca6" />
</p>

<p align="center">
  <img width="49%" alt="ESC PCB render" src="https://github.com/user-attachments/assets/91f54e1d-260a-4a36-ae26-6609ca7e2d04" />
  <img width="49%" alt="ESC layout" src="https://github.com/user-attachments/assets/3dd8e8e2-2dda-4d0c-ae80-309fd9c67f49" />
</p>

---

## Highlights

- **Sensorless six-step commutation** via Back-EMF zero-crossing detection (AM32)
- **6 × TI CSD19531Q5A** N-channel MOSFETs, ~3.1 mΩ R<sub>DS(on)</sub> each
- **FD6288Q** three-phase gate driver with bootstrap high-side supplies and UVLO
- **INA199A1** current-sense amplifier for over-current and stall protection
- **Battery voltage monitoring** for low-voltage cutoff and telemetry
- **On-board power:** JW5026 buck → 5 V, AP2210K LDO → 3.3 V
- **SMF8.0A TVS** protection, SWD programming interface and UART for debugging
- Layout partitioned into power / control / supply zones with thermal vias under the bridge

## Bill of Materials (key parts)

| Function | Part | Notes |
|---|---|---|
| MCU | STM32G0 series (Cortex-M0+) | Runs AM32; timers for complementary PWM, fast ADC |
| Power stage | TI CSD19531Q5A × 6 | 100 V N-channel, SON 5×6, three half-bridges |
| Gate driver | Fortior FD6288Q | 3 high-side + 3 low-side channels, UVLO |
| Current sense | TI INA199A1DCKR | Amplifies shunt voltage for the MCU ADC |
| Buck regulator | JW5026 + 6.8 µH | Battery → 5 V for gate driver and peripherals |
| LDO | AP2210K-3.3TRG1 | 5 V → clean 3.3 V for MCU and analog front end |
| Protection | SMF8.0A | TVS diode |
| Connectors | JST-XH × 2 | Board-to-board / signal connections |

## How It Works

```
 Throttle ─► STM32G0 (AM32) ──PWM──► FD6288Q ──gate──► 6× MOSFET bridge ──► Motor
                 ▲                                          │
                 ├──── Back-EMF dividers (3 phases) ◄───────┤
                 ├──── INA199 current sense ◄───────────────┤
                 └──── Battery voltage divider ◄────────────┘
```

1. The MCU reads the three phase voltages through resistor dividers and detects Back-EMF zero crossings to estimate rotor position.
2. AM32 computes the next commutation step and generates complementary PWM.
3. The FD6288Q translates the logic-level PWM into gate drive for the six MOSFETs; bootstrap diode/capacitor pairs create the floating supply the high-side gates need.
4. Current and battery voltage are sampled continuously for protection (over-current, stall, low-voltage cutoff).

This loop runs thousands of times per second.

<p align="center">
  <img width="49%" alt="Schematic view" src="https://github.com/user-attachments/assets/678cdfe2-9f87-47b8-acf7-7fe2b8eec684" />
  <img width="49%" alt="PCB view" src="https://github.com/user-attachments/assets/da0763be-b5a8-4ff7-8f00-4824cc4099bc" />
</p>

## Design Details

<details>
<summary><b>Power stage & MOSFET selection</b></summary>

Six CSD19531Q5A NexFET devices form a standard three-phase inverter (one high-side and one low-side switch per phase). They were chosen for their low R<sub>DS(on)</sub> (conduction loss) and low gate charge (switching loss at high PWM frequencies). The datasheet rates them above 100 A continuous under ideal thermal conditions; the real rating of this ESC is set by copper weight, thermal vias, heatsinking, airflow and switching frequency, not the MOSFET alone.
</details>

<details>
<summary><b>Gate driver & bootstrap supply</b></summary>

MCU GPIOs can't supply the voltage or current to switch power MOSFETs quickly, so the FD6288Q sits in between. Its UVLO keeps the MOSFETs off when the supply is too low to fully enhance them, preventing linear-region heating, and its drive timing helps avoid shoot-through.

Because both high- and low-side switches are N-channel, the high-side gate must be driven above the phase node. Each phase has a bootstrap diode and capacitor: the capacitor charges while the low-side switch conducts, then acts as a floating supply when the high side turns on.
</details>

<details>
<summary><b>Sensing: Back-EMF, current, battery voltage</b></summary>

- **Back-EMF:** each phase is scaled into the ADC range by a precision divider. Commutation timing depends directly on these readings, so the dividers are tuned for low noise across the full input voltage range.
- **Current:** a low-value shunt feeds the INA199, which rejects the high common-mode voltage and outputs an ADC-ready signal used for over-current protection, stall detection and load monitoring.
- **Battery:** a high-impedance divider lets the firmware implement low-voltage cutoff and compensate for sag as the pack discharges.
</details>

<details>
<summary><b>Power supply architecture</b></summary>

The JW5026 synchronous buck steps the battery down to 5 V efficiently over a wide input range and powers the gate driver. The AP2210K LDO then produces 3.3 V for the MCU; putting an LDO after the switcher strips out ripple that would otherwise corrupt ADC readings. Decoupling is placed at every IC.
</details>

<details>
<summary><b>PCB layout</b></summary>

- **Zoning:** MOSFETs, gate driver and bootstrap parts are grouped into a tight switching block; the MCU and analog sensing sit away from the switch nodes; the buck/LDO get their own section.
- **Power routing:** wide copper pours on battery, bridge and phase paths keep resistance and inductance low. High di/dt loop areas are minimised to reduce overshoot, ringing and EMI.
- **Signal integrity:** Back-EMF, current-sense and battery-sense traces are routed clear of the power stage.
- **Thermal:** large pours around the bridge and via arrays under each MOSFET pad move heat to the other layer.
</details>

## Repository Contents

| File | Description |
|---|---|
| `AM32_ESC.PrjPcb` | Altium Designer project (open this) |
| `AM32_ESC.SchDoc` | Schematic |
| `AM32_ESCRev1.PcbDoc` | PCB layout, revision 1 |
| `AM32_ESC.SchLib` / `AM32_ESC.PcbLib` | Schematic symbols and footprints |

**Tools:** Altium Designer

## Getting Started

1. Open `AM32_ESC.PrjPcb` in Altium Designer.
2. Generate Gerbers, drill files and BOM from the PCB document for fabrication.
3. After assembly, flash an AM32 build for the STM32G0 target over SWD (e.g. ST-Link + STM32CubeProgrammer), then configure motor settings with the AM32 configurator.

> ⚠️ Bring up new boards with a current-limited bench supply before connecting a battery.

## Roadmap

- Bench validation: thermal and continuous-current testing
- DShot / telemetry verification with AM32
- Integration as the power stage for FOC-based thruster control

## Team

| Name | Role |
|---|---|
| **Afraaz Khan** | Hardware design |
| **Ansh Wadhera** | Hardware design |
