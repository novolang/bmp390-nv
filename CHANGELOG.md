# Changelog

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
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, whose
  board packages answer for I2C and SPI as absent hardware: on those images
  `begin` answers `false` until the boards drive the buses.
- The FIFO and the sensor time are not implemented: the traits name none.
