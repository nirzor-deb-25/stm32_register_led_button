# STM32 Register-Level LED and Button Patterns

A project using the **STM32F407G-DISC1** development board to control LEDs and detect button gestures through direct register access, without HAL.

## Demo

[Watch the project demo](demo_compressed.mp4)

The video demonstrates slow blinking, fast blinking, and a steady LED controlled by button gestures.

## Features

- Direct GPIO register configuration
- Button debouncing
- Single-click, double-click, and long-press detection
- SysTick-based timing
- Nonblocking LED control
- Standalone operation after programming

## Hardware and Tools

- STM32F407G-DISC1 development board
- USB connection to the onboard ST-LINK
- STM32CubeIDE
- GNU Arm toolchain supplied with STM32CubeIDE
- Git and GitHub

No external components or wiring are required.

## Pin Mapping

| Component | MCU pin | Function |
|---|---|---|
| Green LED — LD4 | PD12 | Displays the selected LED pattern |
| Blue LED — LD6 | PD15 | Indicates a debounced button press |
| USER button — B1 | PA0 | Selects the LED pattern |

The LEDs turn on when their GPIO outputs are high. The USER button reads high when pressed.

## Button Controls

| Action | Green LED behavior |
|---|---|
| Single click | Slow blinking |
| Double click | Fast blinking |
| Hold for at least one second | Stays on after release |
| Single click after steady-on mode | Returns to slow blinking |

The blue LED follows the debounced button state.

## Timing

| Parameter | Value |
|---|---|
| SysTick interval | Approximately 1 ms |
| Debounce time | 20 ms |
| Double-click window | 350 ms after the first release |
| Long-press threshold | 1000 ms |
| Slow blink | 500 ms on, 500 ms off |
| Fast blink | 100 ms on, 100 ms off |

A single click is confirmed after the double-click window expires, so its response includes a short delay.

## How It Works

### GPIO configuration

The program enables the GPIOA and GPIOD peripheral clocks through the RCC registers.

PD12 and PD15 are configured as outputs for the onboard LEDs. PA0 is configured as an input for the USER button.

LED outputs are controlled through the GPIO bit set/reset register, `BSRR`.

### SysTick timing

The program uses the default 16 MHz HSI clock.

SysTick is configured with a reload value of `15999`, producing an approximately 1 ms interval. The main loop polls the SysTick `COUNTFLAG` to maintain a software millisecond counter.

This implementation does not use a SysTick interrupt.

### Button debouncing

A change in the raw button input must remain stable for 20 ms before it becomes an accepted button-state change.

This reduces false events caused by mechanical contact bounce.

### Gesture detection

The program tracks button presses and releases to distinguish:

- A single short click
- Two short clicks within the double-click window
- A press held for at least one second

The detected gesture selects the green LED mode.

### Nonblocking LED patterns

The program checks elapsed time to decide when to change the LED output.

The final application uses no blocking delay loops, allowing button processing to continue while the LED blinks.

Unsigned time differences are used to handle counter wraparound.

## Main Registers

| Register | Address | Purpose |
|---|---|---|
| RCC_AHB1ENR | `0x40023830` | Enables GPIO peripheral clocks |
| GPIOA_MODER | `0x40020000` | Configures button pin mode |
| GPIOA_PUPDR | `0x4002000C` | Configures input pull resistors |
| GPIOA_IDR | `0x40020010` | Reads the USER button |
| GPIOD_MODER | `0x40020C00` | Configures LED pin modes |
| GPIOD_OTYPER | `0x40020C04` | Configures output type |
| GPIOD_OSPEEDR | `0x40020C08` | Configures output speed |
| GPIOD_PUPDR | `0x40020C0C` | Configures output pull resistors |
| GPIOD_BSRR | `0x40020C18` | Sets and resets LED outputs |
| SysTick_CTRL | `0xE000E010` | Controls SysTick and reads COUNTFLAG |
| SysTick_LOAD | `0xE000E014` | Sets the reload value |
| SysTick_VAL | `0xE000E018` | Clears the current counter value |

Registers are accessed using volatile 32-bit pointers.

## Project Files

| File or folder | Description |
|---|---|
| `Src/main.c` | Final LED patterns and button gesture application |
| `Src/stage1_blink.txt` | Saved basic LED blink implementation |
| `Src/stage2_button.txt` | Saved button-controlled LED implementation |
| `Src/stage3_patterns.txt` | Saved LED patterns implementation |
| `Src/syscalls.c` | Toolchain system-call support |
| `Src/sysmem.c` | Toolchain memory support |
| `Startup/` | MCU startup code |
| `STM32F407VGTX_FLASH.ld` | Flash linker script |
| `STM32F407VGTX_RAM.ld` | RAM linker script |
| `.project` and `.cproject` | STM32CubeIDE project configuration |
| `demo_compressed.mp4` | Hardware demonstration video |

The `.txt` files are learning checkpoints and are not compiled into the application.

## Build and Run

1. Clone this repository:

   ```bash
   git clone https://github.com/nirzor-deb-25/stm32_register_led_button.git
   ```

2. Open STM32CubeIDE.

3. Select **File → Import → General → Existing Projects into Workspace**.

4. Select the cloned repository folder and import the project.

5. Connect the board through its onboard ST-LINK USB connector.

6. Build the project using **Project → Build Project**.

7. Start a Debug session using the onboard ST-LINK.

8. If execution pauses at `main()`, press **F8** to resume.

9. Test the gestures using the blue **USER button**, not the reset button.

After programming, the application can run without an active debugger. Disconnect and reconnect USB power to test standalone operation.

## Hardware Test Results

The following tests were performed manually on the board:

| Test | Result |
|---|---|
| Basic LED blinking | Passed |
| Button-controlled LED | Passed |
| SysTick timing test | Passed |
| Single click selects slow blinking | Passed |
| Double click selects fast blinking | Passed |
| Long press selects steady-on mode | Passed |
| Single click restores slow blinking | Passed |
| Operation after USB power reconnection | Passed |

The project built with **0 errors and 0 warnings**.

## Limitations

- Timing depends on the accuracy of the internal HSI oscillator.
- The SysTick configuration assumes a 16 MHz processor clock.
- Polling COUNTFLAG can miss elapsed ticks if the main loop is delayed for longer than one SysTick interval.
- Validation was performed manually; no automated tests or oscilloscope timing measurements were performed.

## What I Learned

- Configuring GPIO through peripheral registers
- Enabling peripheral clocks through RCC
- Reading digital inputs and controlling digital outputs
- Using SysTick for time-based application logic
- Debouncing a mechanical button
- Implementing button gestures with state-based logic
- Building and debugging an embedded C application
- Documenting and publishing a hardware project with Git

## References

- [STM32F4 Discovery user manual — UM1472](https://www.st.com/resource/en/user_manual/um1472-discovery-kit-with-stm32f407vg-mcu-stmicroelectronics.pdf)
- [STM32F4 reference manual — RM0090](https://www.st.com/resource/en/reference_manual/rm0090-stm32f405415-stm32f407417-stm32f427437-and-stm32f429439-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [Cortex-M4 programming manual — PM0214](https://www.st.com/resource/en/programming_manual/pm0214-stm32-cortexm4-mcus-and-mpus-programming-manual-stmicroelectronics.pdf)
