# Timers

The Game Boy Advance has four 16-bit hardware timers (**TM**0 - **TM**3 or generally **TM**[x]).

Each timer is made up of a 16-bit counter, a 16-bit reload value and a control register.

## IO registers

### Overview

| Address    | Name      | Description                                            |
|------------|-----------|--------------------------------------------------------|
| 0x04000100 |TM0D       | Timer #0 counter (on read) and reload (on write) value |
| 0x04000102 |TM0CNT     | Timer #0 control register                              |
| 0x04000104 |TM1D       | Timer #1 counter (on read) and reload (on write) value |
| 0x04000106 |TM1CNT     | Timer #1 control register                              |
| 0x04000108 |TM2D       | Timer #2 counter (on read) and reload (on write) value |
| 0x0400010A |TM2CNT     | Timer #2 control register                              |
| 0x0400010C |TM3D       | Timer #3 counter (on read) and reload (on write) value |
| 0x0400010E |TM3CNT     | Timer #3 control register                              |

### TM[x]D

Reads return the current 16-bit counter value. Writes set the 16-bit reload value.

### TM[x]CNT

| Bit(s) | R/W | Name                 | Description                                                               |
|:------:|:---:|----------------------|---------------------------------------------------------------------------|
| 0 - 1  | RW  | Clock Divider        | Select a clock divider frequency (see [Clock Divider](#clock-divider))    |
| 2      | RW  | Clock Select [^1]    | 0 = clock divider output, 1 = on **TM**[x-1] overflow                     |
| 3 - 5  | 0   | Unused               |                                                                           |
| 6      | RW  | IRQ enable           | 0 = IRQ disabled, 1 = IRQ enabled                                         |
| 7      | RW  | Enable               | 0 = disable, 1 = enabled                                                  |
| 8 - 15 | 0   | Unused               |                                                                           |

#### Clock Divider

The 16 MiHz (16.777.216 Hz) system clock is divided into four frequencies using a single clock divider.

When a timer uses the system clock (meaning `Clock Select` is zero), it runs at one of those frequencies:

| Value | Frequency | Divisor |
|:-----:|-----------|:-------:|
| 0     | 16 MiHz   | 1       |
| 1     | 256 KiHz  | 64      |
| 2     | 64 KiHz   | 256     |
| 3     | 16 KiHz   | 1024    |

The relation between frequency and divisor is: `Frequency = 16 MiHz / Divisor`

## Functionality

When the enable bit (**TM**[x]**CNT**.bit7) changes from `0` to `1` the reload value is loaded into the counter.

While the enable bit is set, the counter is incremented every time its input clock pulses:
- when `Clock Select = 0`: at the frequency selected via `Clock Divider`.
- when `Clock Select = 1`: when **TM**[x-1] ticks and overflows (the overflow flag of **TM**[x-1] is connected as input clock).

When the 16-bit counter overflows:
1. the reload value is loaded into the counter
2. if IRQ enable (**TM**[x]**CNT**.bit6) is set: an IRQ is requested in **IE** bit `3 + x`

## Timing notes

1. Changes to **TM**[x]**D** and **TM**[x]**CNT** take one cycle to apply[^2].
2. The clock divider appears to generate pulses every `n` cycles (where `n` is the divisor) relative to system startup (meaning a timer can tick in less than `n` cycles after it was enabled). This however hasn't been thoroughly researched yet.

[^1]: for **TM**0 this bit always reads zero (the clock divider output is forcibly used)

[^2]: See [timer/reload](https://github.com/nba-emu/hw-test/tree/master/timer/reload) and [timer/start-stop](https://github.com/nba-emu/hw-test/tree/master/timer/start-stop) 