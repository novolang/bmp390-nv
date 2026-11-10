# bmp390-nv

A driver for Bosch's BMP390, a digital pressure sensor with an I2C and
an SPI interface, specified in Bosch Sensortec's "BMP390 Datasheet",
document number BST-BMP390-DS002-07, revision 1.7, March 2021.  The
driver implements the `Barometer`, `Thermometer` and `Identifiable`
traits of [devices-nv](https://novo-lang.org/packages/devices-nv).  It
reads the trimming coefficients from the part and compensates the
readings with the formulas of the data sheet's appendix.

## Install

```sh
novo pkg add bmp390-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go
to the board's I2C pins and its SDO pin to ground, which puts the part
at 0x76.  The program brings the part up, reads the trimming
coefficients once, and once a second starts a forced measurement, waits
the time the part gives and prints one line over RTT.  At sea level the
pressure is about 101 325 000 millipascals.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_baro.{ Barometer }
use bmp390
use bmp390.{ Bmp390 }

// The part's address with SDO low.
const ADDR = 0x76

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let part = Bmp390.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    if not part.begin()
        hal.uart.write("bmp390: no answer\n")
        return
    let Some(cal) = part.calibration() else return
    loop
        let us = part.baro_start()
        if us >= 0
            hal.timer.delay_us(us)
            match part.sample_with(cal)
                Some(s) =>
                    hal.uart.write("bmp390")
                    field(" mC=", s.milli_c)
                    field(" mPa=", s.milli_pa)
                    hal.uart.write("\n")
                None =>
                    hal.uart.write("bmp390: no result\n")
        hal.timer.delay_ms(1000)
```

The program is `tests/device_read.nv`.  Build it for a board with:

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 12,532 bytes | 1,524 bytes | 2,660 bytes |
| nRF54LM20 DK | 13,308 bytes | 1,482 bytes | 2,644 bytes |
| FRDM-MCXN947 | 25,368 bytes | 9,614 bytes | 2,748 bytes |

The two Nordic board packages answer for I2C and SPI as absent hardware,
so the build removes the code that follows the first bus read, and their
rows are the image without the driver's reading path.  The feature
`features/embedded-devex/nordic-twim-i2c-driver.md` in the novo-lang
repository tracks the missing Nordic I2C driver.  The
FRDM-MCXN947 board package drives I2C and SPI, and its row is the image
with the driver's code in it.  The compensation is binary64 arithmetic,
which these cores compute in software; the compiler's support library
holds it, 7,926 bytes of the FRDM-MCXN947 image.  The rest of each image
is the runtime and tables.

**Hardware: NOT VERIFIED.**  No board carries a BMP390 yet.  The program
builds, and the driver's functions run over a double of the part that
follows the data sheet's register map and bus protocols.  A reading from
a part on a board has not been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The compensation functions are pure.  The driver performs
no input or output of its own and allocates nothing of its own.  The bus
functions of devices-nv build one short frame per transaction inside a
`system` block and release it when the call returns.  The module checks
at every tier, `embedded` included.  The program above declares `[hw]`
for the same reason.

## What it is

A barometer measures air pressure.  Air pressure at sea level is about
101 325 pascals (Pa), and falls by about 12 Pa for each metre of height
near the ground.  The package gives no altitude: that needs the pressure
at sea level where the board is, which changes with the weather.

The BMP390 measures pressure and temperature with a piezo-resistive
element and a converter.  Each result is a 24-bit count, which the data
sheet's formulas turn into pascals and degrees Celsius with 14
coefficients that Bosch writes into the part at production (Table 24).
The driver reads them from the 21 bytes at 0x31.

| Coefficient | Bytes | Signed | Divisor |
|---|---|---|---|
| T1 | 2 | no | 2^-8 |
| T2 | 2 | no | 2^30 |
| T3 | 1 | yes | 2^48 |
| P1 | 2 | yes | 2^20, after subtracting 2^14 |
| P2 | 2 | yes | 2^29, after subtracting 2^14 |
| P3 | 1 | yes | 2^32 |
| P4 | 1 | yes | 2^37 |
| P5 | 2 | no | 2^-3 |
| P6 | 2 | no | 2^6 |
| P7 | 1 | yes | 2^8 |
| P8 | 1 | yes | 2^15 |
| P9 | 2 | yes | 2^48 |
| P10 | 1 | yes | 2^48 |
| P11 | 1 | yes | 2^65 |

A reading is converted in `Float`, binary64, with the operations in the
order of the data sheet's C code (sections 8.5 and 8.6).  It is rounded
to a millidegree and a millipascal.  The traits round the pressure to a
pascal.

The part has three power modes (section 3.3).

| Mode | What it does |
|---|---|
| Sleep | no measurement; the registers and the coefficients can be read |
| Forced | one measurement of pressure and temperature, then sleep |
| Normal | repeated measurements at the output data rate |

A forced measurement takes 234 us + 392 us + 2^osr_p * 2020 us + 163 us
+ 2^osr_t * 2020 us with both quantities on (section 3.9.2).  The
oversampling, osr_p and osr_t, is 0 to 5 for ×1 to ×32.  The driver
answers that time plus 20 %, which covers the maximum column of Table 23
at every oversampling.

In normal mode the output data rate is 200 Hz divided by 2^odr_sel, for
odr_sel 0 to 17 (Table 45).  A period shorter than the conversion sets
conf_err in ERR_REG.  `odr_min_sel` answers the smallest safe odr_sel for
an oversampling, and `normal_start` raises a smaller one to it.

The IIR filter has a coefficient code of 0 to 7 for a filter of 0, 1, 3,
7, 15, 31, 63 and 127 (Table 46).  It is off after `begin`.

The part has an INT pin.  `begin` makes it a data-ready line: active
high, push-pull and latched, so a slow reader does not lose the pulse.
Reading INT_STATUS lowers it.

The part has two bus interfaces on the same pins.

| | I2C | SPI |
|---|---|---|
| Address | 0x76, or 0x77 with SDO high | chip select low |
| Read | register address, repeated start, data | control byte with bit 7 set, one dummy byte, then data |
| Write | register address, then the data | control byte with bit 7 clear, then the data |
| Register address | advances after each byte | advances after each byte |
| Clock | 400 kHz | 4 MHz, mode 3 (mode 0 or 3, up to 10 MHz) |

An SPI read clocks one dummy byte in after the control byte (section
5.3.2) that an I2C read does not.  The driver drops it.

## What the package contains

| Module | What is in it |
|---|---|
| `bmp390` | The types `Bmp390` (I2C) and `Bmp390Spi` (SPI), `Bmp390Cal` and `Bmp390Sample`, the trait `Bmp390Port`, the bus framing, the register access, the compensation, the settings and the bring-up |

## How to choose an entry point

A program on a board uses `Bmp390` for a part on I2C and `Bmp390Spi` for
a part on SPI.  Each is built by `attach` with the board's bus, its GPIO
and the wiring, as a board's device table builds it.  Both implement
`Barometer`, `Thermometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | resets and brings the part up, see below |
| `baro_start()`, `temp_start()` | start one forced measurement and answer the time to wait in microseconds; in normal mode nothing is written and the answer is 0; -1 when the part did not answer |
| `baro_read()`, `temp_read()` | the latest pressure in pascals, and the temperature in millidegrees; each reads the coefficients first |
| `calibration()`, `sample_with(cal)` | the coefficients once, and a reading that uses them: one transaction fewer for each reading |
| `sample()` | the latest result as `Bmp390Sample`: millidegrees and millipascals |
| `device_id()` | `Identifiable`: CHIP_ID, 0x60, with REV_ID as `rev` |
| `set_oversampling(osr_p, osr_t)`, `set_iir(code)` | the oversampling and the IIR filter |
| `run_normal(sel)`, `sleep()` | normal mode at odr_sel `sel`, and sleep |
| `ready()`, `clear_irq()`, `errors()` | the data-ready flags, INT_STATUS, and ERR_REG, which both clear on read |
| `irq_level()` | the level of the INT pin, -1 for a wiring with none |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

`begin` checks CHIP_ID, resets the part with the soft-reset command and
waits for por_detected, checks ERR_REG, then sets sleep with both
quantities on, ×8 pressure and ×1 temperature oversampling, the IIR
filter off, odr_sel 3 and the data-ready on INT, and reads the
coefficients once.  It answers `false` for a part that does not answer
0x60 and for a register that does not hold what was written.  The BMP388
answers 0x50 and is not accepted.

A program that holds its own bus, or a test with a double of the part,
implements `Bmp390Port` over its bus and calls the module's functions,
which take the port: `bmp390.sample_read(port)`, `bmp390.start(port)`.
The framing of each bus is `i2c_read_into`, `i2c_write`, `spi_read_into`
and `spi_write`, generic over `I2cBus` and `SpiBus`.  The compensation
is `cal_of`, `temperature_c` and `pressure_pa`, which need no bus.

## The rules a user needs

1. `begin` has to run before a reading, and a forced measurement has to
   be started and waited for.  The data registers read 0x800000 before
   the first measurement, and `baro_read` then answers `None`.
2. The time `baro_start` answers is the time to wait before the read.  A
   read that comes sooner answers the previous result.
3. One measurement gives both results.  `baro_start` and `temp_start` are
   the same write.
4. Read the coefficients once with `calibration()` and use
   `sample_with` in a loop.  `baro_read` and `temp_read` read them
   again each time.
5. Both counts are read in one burst of the six data registers, so they
   come from one measurement (section 3.10).
6. A signed coefficient read as unsigned, or the reverse, gives a large
   error.  The table above is Table 24's.
7. The data sheet recommends normal mode when the IIR filter is used
   (section 3.3.3).  In normal mode the period must be at least the
   conversion time.
8. A change of mode during a measurement waits for the measurement to
   end, and further mode commands are ignored until then (section
   3.3.4).
9. The data sheet's Table 23 prints 18.69 ms for ×8 pressure and ×1
   temperature, where the formula of section 3.9.2 gives 18.97 ms.  The
   driver uses the formula.  Table 45 prints a prescaler of 127 for
   odr_sel 7, and the period it prints, 640 ms, is 128 times 5 ms.

## What is not included

- The FIFO: neither `Barometer` nor `Thermometer` names one.  `reg` and
  `set_reg` reach it.
- The sensor time and the FIFO sensor-time frame.
- The SPI three-wire mode, the I2C watchdog and the INT pin's
  open-drain and level options.  `set_reg` reaches IF_CONF and INT_CTRL.
- A self-test: the data sheet defines none.
- An altitude, which needs the pressure at sea level (see above).
- The BMP388, whose CHIP_ID is 0x50.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  traits this driver implements, the wiring record and the register
  helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus` and `GpioOut` traits and the board handles.
- lis3dh-nv and lis3mdl-nv drive an accelerometer and a magnetometer
  with the same bus conventions, a port per bus and a device program.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus and on an SPI bus, with the clear-on-read registers, the forced
measurement that returns to sleep, the configuration error and the dummy
byte of an SPI read.  It checks the framing of every entry point, the
bring-up and its refusals, the settings, the conversion times, and the
compensation.  The suite covers every line of `src/`.

The compensation is checked against vectors computed by a separate
transcription of the data sheet's appendix 8.4 to 8.6 in binary64.  The
data sheet gives the formulas and no numeric example, so the vectors are
the formulas' results for two sets of coefficients: one of the magnitude
a part has, which gives 24.93 °C and 101.32 kPa at the raw counts
8 453 000 and 6 700 000, and one that exercises the unsigned values
above 32767 and the negative signed ones.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are
the data sheet above and Zephyr's `drivers/sensor/bosch/bmp388`.  Each
fix kept is named in `src/bmp390.nv` where it is kept.

## Licence

Apache-2.0.
