# SoftStarter for Universal Motor

Firmware project based on STM32Cube/HAL for an experimental soft starter for a universal motor. The main code is located in `Core/Src/main.c` and uses timers to perform gradual actuation, measure the signal applied to the motor, and output the results over the serial port.

> **Safety Warning:** This project involves electrical power and inductive load switching. Do not connect or test the circuit on a bench without proper isolation, adequate protection, an appropriate TRIAC/driver, and electrical installation expertise. Verify voltages, currents, thermal dissipation, and grounding before use.

## Overview

The system uses `TIM2` in trigger and toggle mode to generate pulses on `PA1`, alternating the compare timing between pulse array values. `TIM3` executes an actuation ramp by progressively decreasing the configured firing angle and adjusting the repetition period. Simultaneously:

* `TIM5` measures input signal edges on `PA0`;
* `TIM10` periodically sends telemetry data over `USART2`;
* `TIM11` toggles `PB9` at a fixed frequency;
* The onboard LED `PA5` remains available for status indication;
* The blue push-button `PC13` is configured as an interrupt input, although its callback is not yet implemented in the current code.

## Hardware

### Core

* Microcontroller: STM32F446;
* System clock: 84 MHz;
* Framework: STM32 HAL;
* Project configuration: `soft2.ioc`;
* Toolchain configured in CubeIDE: GNU Arm Embedded (`arm-none-eabi-gcc`).

### Pinout & Connections

| Pin | Project Function | Configuration |
| :--- | :--- | :--- |
| `PA0` | `TIM5_CH1` input capture | Signal edge measurement |
| `PA1` | `TIM2_CH2` output | Timer-driven control pulse |
| `PA2` / `USART_TX` | `USART2_TX` | Serial output, 115200 baud |
| `PA3` / `USART_RX` | `USART2_RX` | `USART2` serial input |
| `PA5` | LED `LD2` | Digital output |
| `PB8` | `TIM2_ETR` | External trigger for `TIM2` |
| `PB9` | Digital output | Toggled by `TIM11` |
| `PC13` | `B1` — blue button | External interrupt on falling edge |
| `PA13` / `PA14` | `SWDIO` / `SWCLK` | Debugging |

The `soft2.ioc` file contains the complete peripheral and pinout configuration.

## Firmware Operation

### Initialization

During startup, `main()` executes:

1. HAL and system clock configuration;
2. GPIO and timer initialization;
3. `USART2` initialization at 115200 baud, 8-N-1;
4. Interrupt startup for timers `TIM2`, `TIM3`, `TIM5`, `TIM10`, and `TIM11`;
5. Initial compare value setup for `TIM2`;
6. Entering an idle main loop — all core logic runs inside interrupt callbacks.

The system is configured to use the internal HSI oscillator with PLL. `SYSCLK` runs at 84 MHz, with an APB1 timer clock of 84 MHz for the timers used.

### Actuation Ramp

`TIM3` generates periodic interrupts and runs the ramp inside `HAL_TIM_PeriodElapsedCallback()`:

* `angulo` starts at 179 (180 degrees);
* On each `TIM3` event, the angle is decremented;
* The decrement continues until `angulo == 20`, at which point reduction stops;
* `pulso[0]` is updated with `angulo * 10`;
* `TIM3`'s auto-reload period is updated with `arr`.

At initialization:

```c
arr = ((84000000 * Tempo_UP / 179) / 8399);
```

Both `Tempo_UP` and `Tempo_DOWN` are defined as 20, though only `Tempo_UP` is currently used in the `arr` calculation.

`TIM2` is configured with a prescaler of 387, a period of 1799, and output on `PA1` (Channel 2). Alternately, the output compare callback switches the compare value between `pulso[SOBE]` and `pulso[DESCE]`. Initial values are:

* `pulso[SOBE] = 1000`;
* `pulso[DESCE] = 1789`.

This setup shifts the firing pulse along the ramp. The external power circuit is not documented in this repository and must be verified before connecting a real load.

### Signal Measurement

The `TIM5_CH1` input on `PA0` captures every edge transition, as it is configured with `BOTHEDGE` polarity.

Inside `HAL_TIM_IC_CaptureCallback()`:

* `borda[atual]` stores the captured timer counter;
* `borda[subida]` and `borda[descida]` are updated as references for previous edges;
* The period is computed as the difference between the current edge and the rising edge;
* Frequency is calculated as `1 / period_in_seconds`;
* Duty cycle is estimated via `(time between rising and falling edge / period) * 100`.

`TIM5` uses a prescaler of 83 against the 84 MHz clock, yielding an approximate resolution of 1 $\mu\text{s}$. The 32-bit timer counter allows capturing across a wide range of time intervals.

The current implementation retains only the last three edge timestamps. For noisy or unstable signals, digital filtering and edge-sequence validation may be required.

### Serial Output

`USART2` transmits on `PA2` (`TX`) at 115200 baud. `TIM10` triggers an interrupt approximately every 1 second and transmits the following telemetry data:

```text
Borda de subida: <value> | Borda de descida: <value> | Borda atual <value>
Periodo: <value> us | Frequencia: <value>
Dutycicle: <value>%
```

Expected output example:

```text
Borda de subida: 100.0 | Borda de descida: 600.0 | Borda atual 1100.0
Periodo: 1000.00 us | Frequencia: 1000.00
Dutycicle: 50.00%
```

To view the output, connect a USB-UART adapter or use the onboard ST-Link Virtual COM Port configured at 115200 baud, 8 data bits, 1 stop bit, and no parity.

### Auxiliary Timer

`TIM11` toggles the logic level of `PB9` in its period-elapsed callback. `TIM11` is configured with a prescaler of 349 and a period of 999, producing a periodic output with transitions roughly every 2 ms.

LED `LD2` on `PA5` is initialized low and is not toggled by the current firmware.

## Timer Configuration

| Timer | Function | Effective Clock | Prescaler | Period | Effect |
| :--- | :--- | ---: | ---: | ---: | :--- |
| `TIM2` | OC/ETR output pulse | 84 MHz | 387 | 1799 | Toggles `PA1` / `TIM2_CH2` |
| `TIM3` | Soft-start ramp | 84 MHz | 8399 | 9999 (initial) | Periodic interrupt updating firing angle |
| `TIM5` | Input capture | 84 MHz | 83 | $2^{32}-1$ | Measures `PA0` with $\approx 1\ \mu\text{s}$ resolution |
| `TIM10` | Serial telemetry | 84 MHz | 4199 | 9999 | $\approx 1\text{ s}$ periodic interrupt |
| `TIM11` | Auxiliary output on `PB9` | 84 MHz | 349 | 999 | Periodic toggling on `PB9` |

Exact timing intervals may vary depending on clock tree settings, prescalers, and how HAL accounts for zero-indexed timer register values.

## Project Structure

```text
motor-softstarter/
├── Core/
│   ├── Inc/
│   │   ├── main.h
│   │   ├── stm32f4xx_hal_conf.h
│   │   └── stm32f4xx_it.h
│   └── Src/
│       ├── main.c                 # Initialization, peripherals, and callbacks
│       ├── stm32f4xx_hal_msp.c   # HAL GPIO and peripheral MSP initialization
│       ├── stm32f4xx_it.c        # Interrupt handlers
│       └── system_stm32f4xx.c    # CMSIS system initialization
├── Drivers/
│   ├── CMSIS/
│   └── STM32F4xx_HAL_Driver/
├── Debug/                        # Pre-generated local build artifacts
├── STM32F446RETX_FLASH.ld        # Linker script for Flash memory
├── STM32F446RETX_RAM.ld          # Linker script for RAM execution
├── soft2.ioc                     # STM32CubeMX / CubeIDE configuration
└── soft2 Debug.launch            # Debug launch configuration
```

## How to Build

### STM32CubeIDE

1. Open STM32CubeIDE.
2. Go to **File → Import → Existing Projects into Workspace**.
3. Select the root directory of this repository.
4. Confirm the detected project.
5. Build via **Project → Build All** or press `Ctrl+B`.
6. Flash the firmware to the board using the debug configuration provided in `soft2 Debug.launch`.

## How to Run

1. Import and build the project.
2. Flash the firmware onto an STM32F446RE Nucleo or compatible board.
3. Connect `PA2` to the `RX` pin of a USB-UART adapter, sharing a common `GND`.
4. Configure your serial terminal to 115200 baud, 8-N-1.
5. Reset the board.
6. Monitor the period, frequency, and duty cycle messages.
7. For real motor actuation, connect only a previously validated power stage to the control signal on `PA1`/`PB8` and verify isolation and protection ratings.

There is no runtime configuration interface over `USART` in the current firmware; parameters are compiled directly into `main.c`.

## Tunable Parameters

The primary parameters are defined in the `USER CODE PV` section of `Core/Src/main.c`:

| Variable | Initial Value | Purpose |
| :--- | ---: | :--- |
| `angulo` | 179 | Ramp starting angle |
| `Tempo_UP` | 20 | Parameter used in `arr` calculation |
| `Tempo_DOWN` | 20 | Defined, though not yet used by current logic |
| `pulso[0]` / `pulso[SOBE]` | 1000 | Initial compare value for rising phase |
| `pulso[1]` / `pulso[DESCE]` | 1789 | Compare value for falling phase |
| `estado3` | 0 | State toggled by `TIM2` callback |
| `msgSIZE` | 100 | Serial message buffer size |
| `TIMEOUT_SERIAL` | 100 ms | UART transmission timeout |

## Known Areas for Improvement

* Rename `dutycicle` to `dutycycle` in telemetry output;
* Validate frequency calculations on the very first capture and during incomplete transitions;
* Protect or synchronize shared variables accessed between interrupt callbacks and `USART` transmission;
* Use integer arithmetic or digital filters to improve measurement noise immunity;
* Implement the external interrupt callback for the blue user button (`PC13`);
* Implement a safe stop routine and full ramp-down control using `Tempo_DOWN`;
* Add zero-crossing / signal-loss detection, overcurrent protection, and power-stage fault handling;
* Remove unused functions or document the external trigger signal expected on `PB8` / `TIM2_ETR`;
* Document the TRIAC/optotriac driver schematic, galvanic isolation, auxiliary power supply, and load protection circuitry.

## Key Files

* `Core/Src/main.c`: peripheral setup, callbacks, and ramp control logic.
* `Core/Src/stm32f4xx_hal_msp.c`: peripheral MSP initialization.
* `Core/Src/stm32f4xx_it.c`: exception and interrupt service routines.
* `Core/Inc/main.h`: pin definitions and shared prototypes.
* `soft2.ioc`: graphical configuration of clocks, peripherals, and GPIOs.
* `STM32F446RETX_FLASH.ld`: memory layout linker script for Flash execution.