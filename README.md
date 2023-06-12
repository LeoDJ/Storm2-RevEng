# Shargeek Storm² Liquid Powerbank Reverse Engineering

I own both a first-gen Storm2 directly from the first Kickstarter campaign and a second-gen Storm2.  
This repo contains my **WIP** reverse engineering efforts concerning the first-gen.  
I don't know the extent to how they differ yet, except that the second-gen uses a smaller "Master Controller" package (48 instead of 64 pins).

I will probably keep updating [this thread](https://chaos.social/@LeoDJ/110164612794922315), if you want to keep up-to-date.

### Motivation
I have a few small gripes with the stock firmware I'd like to improve, if possible:
1. The button long press time is way too long, which makes the 1-button interface even more tedious to use
1. Pass-through mode gets disabled if the SoC falls under 50%
1. DC in/out is limited to 3A, but USB-C1 output is capable of 5A
    - Both the DC jack (`PJ-051AH`) and the DC-DC converter are capable of 5A, I don't understand this limit at all

## General Interesting Points
- If your Storm2 ever acts up, it can probably be fixed by pressing the reset button on the BMS PCB (underneath the right cover, only 4 screws to remove).
- The "USB Controller" is actually capable of the full 20V 5A PD, but it's buck-only, so it's limited to ~15V output (and probably a bit less when the battery gets more empty). But they limited it to 3A, maybe because of thermal reasons.

## Teardown
- Remove the 4 screws of the right cover and remove it
- Use a spudger to pry out the button and display cover. You need to use sketchy amount of force.
- Slide out the transparent case
- Disconnect the BMS data cable (yellow wires)
- Disconnect the BMS from the mainboard by pushing against it from the back side (besides the XT30 connector)
- Remove the 4 mainboard screws
- The mainboard is held in place with a thick thermal pad, but you should now be able to remove it from the case
- There's also [this great teardown + review](https://www.youtube.com/watch?v=MWEFiAiD0Wg) by ChargerLAB

## ICs
(Silkscreen) Name             | Chip                                                                                        | Notes
------------------------------|---------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------
Master Controller             | Marking: `Naxim NXMCB6 AOC18220F 19181010C 2020 A` <br>Actually: `STM32F103RBT6` (or clone) | - Does display & button handling, talks to BMS, "USB Controller" and "PD Controller"
USB Controller                | `SW3517S`                                                                                   | - Handles USB-C2 and USB-A port <br>- Complete stand-alone DC-DC buck solution <br>- Connected to the MC via I²C
PD Controller                 | `CS32G020` (`CSA36FX30`)                                                                    | - Handles USB-C1 and DC port <br>- Another MCU, Cortex M0 <br>- _Could_ be programmed via USB-C1 CC pin, but no documentation seems to exist
DC Buck-Boost Controller      | `SC8812A`                                                                                   | - Controlled by PD Controller
Flash                         | `BY25D80`                                                                                   | - 1MB <br>- Probably only stores display graphics
Coulombmeter Controller (BMS) | `Atmel SAMD10U 906B`                                                                        | - Connected to the MC via I²C

## Pinouts
### Debug Pads
There are a few big debug pads in a cluster on the underside. 3.3V is on the top (and marked in silkscreen).
| MC SWD       | MC Debug UART | PDCtrl      |
|--------------|---------------|-------------|
| 3V3          | 3V3           | 3V3         |
| PA14 (SWCLK) | PA9 (TX1)     | GND         |
| PA13 (SWDIO) | PA10 (RX1)    | PA1 (SWDIO) |
| GND          | GND           | PA2 (SWCLK) |
|              |               | RST         |

### Master Controller
Periph/Pin | Pins               | Connected to
-----------|--------------------|-----------------------------------------------------------------
USART1     | TX/RX: PA9/PA10    | Debug pads (see above)
USART2     | TX/RX: PA2/PA3     | PDCtrl PA8 (UART RX) <br>(RX/TX connected together via 20kOhm)
I2C1       | SCL/SDA: PB8/PB9   | BMS
I2C2       | SCL/SDA: PB10/PB11 | USBCtrl
PA4        |                    | Prob. SPI Flash CS
PB12       |                    | Enters debug state when high?
PC8        |                    | Mayb. display backlight?
PC15       |                    | Button (active low)

## Unsorted RevEng Notes
- The UART protocol always starts with 4 preamble bytes (`0x55AA8181` for debug UART and `0x55AA0181` for comms with PD controller)
- `0x0801F000` - `FFF` (4K) is some kind of programmable flash data page