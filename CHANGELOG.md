# Changelog

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Bmp390Port` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Bmp390Port` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Bmp390Port` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Bmp390Port for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `i2c_setup`, `spi_setup`, `irq_level_of`, `reg_read`, `reg_write`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Bmp390Port` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 18,368 bytes before and 18,304 after on the read-out (nrf52-qemu: 8,192 and 8,192).

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.2), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed; the suite passes without an edit.

The device program builds for the boards its README names, flash nrf52840-dk 18,316 B before and 18,316 B after; nrf54lm20-dk 19,412 B before and 19,412 B after.

## 0.1.0

The first release: a driver for Bosch's BMP390 digital pressure sensor on
an I2C or an SPI bus.

- `Bmp390` (I2C) and `Bmp390Spi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Barometer`, `Thermometer` and
  `Identifiable`, with their own methods for the calibration, a reading in
  millidegrees and millipascals, the oversampling, the IIR filter, normal
  mode, the data-ready line and register access.
- The register access as functions over `Bmp390Port`, a trait that a type
  holding a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.  An SPI read drops the dummy byte.
- The compensation of the data sheet's appendix in binary64: the 14
  coefficients from the 21 bytes at 0x31, the temperature, and the
  pressure.
- The conversion time of section 3.9.2 plus 20 %, and the smallest output
  data rate that keeps the period at or above it.
- A reset with the soft-reset command, finished when por_detected reads 1.
- The suite runs over a double of the part on both buses and covers every
  line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a BMP390.
- The data sheet gives no numeric example of the compensation.  The
  suite's vectors come from a separate transcription of its C code.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK.  Their
  board packages answered for I2C and SPI as absent hardware when the
  package was written (images of 12,532 and 13,308 bytes of flash, without
  the reading path); the TWIM and SPIM drivers, tracked by
  `features/embedded-devex/nordic-twim-i2c-driver.md`, landed on 2026-11-10,
  and the images are now 22,980 and 23,988 bytes with the reading path.  A
  reply from a BMP390 over those buses is NOT VERIFIED until the driver rig.
- The FIFO and the sensor time are not implemented: the traits name none.
