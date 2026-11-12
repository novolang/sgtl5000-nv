# Changelog

## 0.2.0 — 2026-11-12

Moves off its stand-ins onto devices-nv 0.4.0 and embedded-hal-nv 0.4.0. The release depends on both.

- `codec_start(f)` replaces the inherent `start_clock` and `begin`: it starts the
  I2S handle so that SYS_MCLK runs, reads the chip ID and leaves the part quiet,
  and answers 0 microseconds to wait or -1. The device program and the example
  call it, and print one refusal line for the two causes.
- `codec_set_gain` takes `CodecHeadphone` (the headphone volume, as
  `CodecOutput`) and `CodecDac` (the DAC's digital volume), which replaces the
  inherent `set_dac_volume` and `dac_volume`. The line out, the speaker and
  `CodecAdc` answer `None`.
- `codec_select_output` powers the headphone or the line-out driver up or down;
  `codec_set_path_mute` and `codec_path_muted` mute the headphone, the line out,
  the input, the DAC or the whole output.
- The framings `AudioRightJustified`, `AudioDspA` and `AudioDspB` replace the
  inherent `set_rj_format` and `set_pcm_format`; they are part of the format and
  written with it. `AudioPdm` is refused.
- `slot_bits` is read: 24-bit samples in a 64-clock frame (`slot_bits` 32) and
  16-bit samples in a 64-clock frame are accepted, and `i2s_config_of` passes
  `slot_bits` to the handle. A 24-bit format with `slot_bits` 0 is still refused.
  The right-justified form needs the 64-clock frame.
- A record literal of `AudioFormat` adds `slot_bits`.

## 0.1.0

The first release: a driver for NXP's SGTL5000 low-power stereo codec on an
I2C control bus with 16-bit registers and a board's I2S handle.

- `Sgtl5000`, the type `attach` builds from the board's I2C bus, the wiring and
  the I2S handle, implementing devices-nv's `AudioCodec` and `Identifiable`,
  with its own methods for the DAC volume and ramp, the line-out level, the
  routing, the ADC input, the analogue bypass, the zero-cross detectors, the
  microphone amplifier and bias, the right-justified and PCM formats, and the
  PLL and short-detector flags.
- The chip ID: PARTID 0xA0 as `who` and the revision byte as `rev`.
- The 16-bit register access through devices-nv's `wiring_read16_into` and
  `wiring_write16_from`; the registers are read back for every
  read-modify-write, and CHIP_SSS_CTRL, whose I2S_SELECT field is write-only,
  is written whole.
- The power-up of the data sheet's "Chip Powerup and Supply Configurations" in
  its order, for a VDDD made by the chip or driven from outside and for
  supplies below, above and across 3.1 V. For VDDIO 3.3 V, VDDA 1.8 V and an
  internal VDDD it is the data sheet's example byte for byte. It keeps the
  PLL bits a format has set, and leaves everything muted.
- `step_quiet`, run by `begin`: everything muted, and a VAG left powered by an
  earlier run switched off and given 500 ms to ramp down.
- The clock: SYS_FS and RATE_MODE for 8 to 96 kHz, the synchronous ratios of
  256, 384 and 512, and the PLL for any clock from 8 to 27 MHz with the
  data sheet's divisors (12 MHz at 48 kHz is 16 and 786); the PLL needs the
  part as the I2S master, and the format step waits up to 50 ms for the lock.
- The serial port in I2S and left-justified formats through `AudioCodec`, and
  the right-justified and PCM A and B formats through `set_rj_format` and
  `set_pcm_format`; 16 and 32 bits. `codec_set_format` configures and enables
  the board's I2S handle first, and `start_clock` does so alone, because the
  part answers I2C only while SYS_MCLK runs.
- The output gain of `AudioCodec` is the headphone volume, +12 to -51.5 dB in
  0.5 dB steps, and the input gain is the ADC volume, -6 to +22.5 dB in 1.5 dB
  steps, each never above the request.
- The mute and the unmute in the order that gives no click: the DAC's soft mute
  and a wait for the volume ramp before the analogue mutes, and the analogue
  unmutes before the ramp up.
- The power-down: the outputs muted, VAG switched off with the headphone and
  line out powered for 500 ms, then the blocks.
- The suite runs over a double of the part and of the I2S handle and covers
  every line of `src/`.
- The device program `examples/headphones.nv`. Built for the nRF52840-DK it is
  15,192 B of flash and for the nRF54LM20-DK 16,656 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an SGTL5000, and no
  board package has an I2S block. The Nordic DKs drive I2C with TWIM0 and
  TWIM22, so the device program's images carry the reading path; a reply from
  the codec over either bus is NOT VERIFIED. The device program is given a
  stand-in handle.
- The data sheet is inconsistent with itself and with the reference drivers on
  the charge pump bit and on the external-VDDD write; the choices are named in
  `src/sgtl5000.nv` and the README. The VDDD level of the chip's own regulator
  is NOT VERIFIED.
- The waits of 400 ms after the power-up and 500 ms for the VAG ramp down are
  the Teensy library's and Linux's; the data sheet gives no time.
- The manifest names devices-nv and embedded-hal-nv by the registry range
  `^0.3.3`, the first releases with `AudioCodec` and `I2s`.
- Not implemented: the signal processor (surround, bass enhance, equalisers,
  automatic volume control, the dual input mixer), the pad drive strengths, the
  test registers, the ADC high-pass filter bits, the 20 and 24 bit data lengths,
  mono formats, the SPI port, and `AudioCodecAsync`.
- A local named `codec` breaks the embedded tier check
  (`a-local-named-like-a-stdlib-module-makes-the-embedded-tier-check-read-its-method-calls-as-module-calls`);
  the device program calls its codec `part`.
- The linter's `char-literal` rule reads register numbers as characters in the
  test suite. It is wrong for a register map, and those sites are left.
