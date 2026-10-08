# STM32 Register-Level LED and Button Patterns

A learning project using the STM32F407G-DISC1 development board.
The application controls onboard LEDs and detects button gestures
through direct register access, without HAL.

## Hardware and Tools

- STM32F407G-DISC1 development board
- USB data cable connected to the ST-LINK port
- STM32CubeIDE 2.2.0
- GNU Arm toolchain supplied with STM32CubeIDE

No external wiring is required.

## Pin Mapping

| Component | MCU pin | Behavior |
|---|---|---|
| Green LED, LD4 | PD12 | High turns the LED on |
| Blue LED, LD6 | PD15 | High turns the LED on |
| Blue USER button, B1 | PA0 | High when pressed |

## Features

- Basic LED blink checkpoint
- Button-controlled LED checkpoint
- SysTick timing through direct register access
- Software button debouncing
- Single-click, double-click and long-press detection
- LED patterns that run while button input is monitored
- Standalone operation after disconnecting the debugger

## Final Behavior

| Action | Result |
|---|---|
| Reset or power on | Green and blue user LEDs start off |
| Single short click | Green LED blinks slowly |
| Double short click | Green LED blinks quickly |
| Hold for at least one second | Green LED stays on |
| Single click after long press | Green returns to slow blinking |
| Hold USER button | Blue LED indicates the debounced pressed state |

The single-click action waits approximately 350 ms after release
to determine whether a second press follows.

A double click requires the second debounced press to occur within
350 ms of the first debounced release. Releasing the second short
press selects fast blinking.

## Timing

The project retains the STM32 reset clock configuration:
the internal 16 MHz HSI clock supplies the processor.

SysTick reload is set to 15999, producing a nominal 1 ms tick.
The program polls COUNTFLAG and does not enable SysTick interrupts.

| Setting | Value |
|---|---|
| Debounce interval | 20 ms |
| Double-click window | 350 ms |
| Long-press threshold | 1000 ms |
| Slow blink | 500 ms on, 500 ms off |
| Fast blink | 100 ms on, 100 ms off |

Timing accuracy depends on the internal oscillator.

## Registers Used

| Register | Address | Purpose |
|---|---|---|
| RCC_AHB1ENR | 0x40023830 | Enable GPIOA and GPIOD clocks |
| GPIOA_MODER | 0x40020000 | Configure PA0 as an input |
| GPIOA_PUPDR | 0x4002000C | Configure PA0 pull-down |
| GPIOA_IDR | 0x40020010 | Read the USER button |
| GPIOD_MODER | 0x40020C00 | Configure PD12 and PD15 as outputs |
| GPIOD_OTYPER | 0x40020C04 | Select push-pull outputs |
| GPIOD_OSPEEDR | 0x40020C08 | Select low output speed |
| GPIOD_PUPDR | 0x40020C0C | Disable LED pin pull resistors |
| GPIOD_BSRR | 0x40020C18 | Set or reset LED outputs |
| SysTick CTRL | 0xE000E010 | Enable timer and read COUNTFLAG |
| SysTick LOAD | 0xE000E014 | Set timer reload value |
| SysTick VAL | 0xE000E018 | Clear current counter value |

Register access uses volatile uint32_t pointers.

BSRR bits 0–15 set output pins.
BSRR bits 16–31 reset output pins.

## Program Structure

- main(): configures GPIO and SysTick, then processes button input.
- green_write(): switches the green LED on or off.
- set_mode(): selects a pattern and resets its timing.
- pattern_tick(): advances the active blinking pattern.

Debouncing accepts an input change only after it remains stable
for 20 ms. Gesture detection uses the debounced press and release
events.

## Build and Run

1. Open the project in STM32CubeIDE.
2. Select Build Project.
3. Connect the board through its ST-LINK USB port.
4. Select Debug As → STM32 C/C++ Application.
5. Use ST-LINK with the SWD interface.
6. Resume execution with F8 when paused at main().
7. Test the blue USER button.

After flashing, the program also starts when the board is powered
without an active debugging session.

## Manual Test Results

| Test | Result |
|---|---|
| Initial green LED blink | Passed |
| LED follows button press and release | Passed |
| Timer blink runs alongside button input | Passed |
| Single click selects slow blink | Passed |
| Double click selects fast blink | Passed |
| Long press selects steady on | Passed |
| Single click restores slow blink | Passed |
| Gestures work after USB power cycle | Passed |

These results come from manual testing on the development board.
Debouncing is implemented, but electrical bounce waveforms were
not measured.

## Limitations

- SysTick is polled. If the loop takes longer than one millisecond,
  multiple elapsed ticks can collapse into one COUNTFLAG event.
- Clock changes require updating the SysTick reload value.
- Gesture thresholds are fixed in the source code.
- No automated tests have been performed.
- Only the green and blue user LEDs are used.

## Learning Outcomes

- GPIO clock enabling and pin configuration
- Bit masking and direct hardware register access
- Input reading through IDR
- Output control through BSRR
- SysTick configuration
- Button debouncing and gesture detection
- Nonblocking LED pattern logic
- Building, flashing and debugging embedded C

## References

- ST UM1472: Discovery kit with STM32F407VG MCU user manual
- ST RM0090: STM32F4 reference manual
- ST PM0214: STM32 Cortex-M4 programming manual