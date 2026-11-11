# sgtl5000-nv

A driver for NXP's SGTL5000, a low-power stereo codec with a headphone
amplifier, a line input and output, a microphone amplifier, an I2C control
port with 16-bit registers and an I2S or PCM audio port, specified in the
data sheet "SGTL5000: Low Power Stereo Codec with Headphone Amp", document
SGTL5000, Rev. 7, 1/2022. The driver implements the `AudioCodec` and
`Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and configures the
board's `I2s` handle of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) for the
format it sets. The part is the codec of the Teensy 4 audio adapter.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
chip, and no board package has an I2S block yet. The driver has run only
against a double of the part and a double of the I2S handle, in the host
suite. The device program below builds for the nRF52840-DK and the
nRF54LM20-DK against a stand-in handle and has not been flashed.

## Install

```sh
novo pkg add sgtl5000-nv
```

The device program is `examples/headphones.nv`. Make it the `main` of a
package that depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/headphones.nv` brings the codec up and plays a square wave of 1 kHz
at 48 kHz on the headphone output, printing one line per step over the
board's UART:

```
codec found
```

```novo norun:needs-pkg
use embedded_hal.{ I2s, I2sConfig, BoardI2c }
use dev_audio.{ AudioCodec, AudioFormat, AudioFraming, CodecPath, MilliDb }
use dev_bus
use sgtl5000

// The board's I2S block goes here; this stand-in refuses to be configured.
@value
struct NoI2s
    id: Int

impl I2s for NoI2s
    fn configure(self, c: I2sConfig) -> Bool [hw]
        false
    fn sample_rate(self) -> Int [hw]
        0
    fn enable(self) -> Bool [hw]
        false
    fn disable(self) [hw]
        pass
    fn transmit(self, var buf: Vec[i32; 256]) -> Int [hw]
        -1
    fn receive(self, var buf: Vec[i32; 256], n: Int) -> Int [hw]
        -1
    fn xruns(self) -> Int [hw]
        0

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    // The codec answers at 0x0A with CTRL_ADR0_CS low, on the first I2C bus,
    // and the board gives it a SYS_MCLK of 12.288 MHz.
    let part = sgtl5000.attach(BoardI2c { id: 0 }, dev_bus.wiring_i2c(0, sgtl5000.address_of(false)),
                               NoI2s { id: 0 }, sgtl5000.config_default(12288000))
    let f = AudioFormat { sample_rate: 48000, bit_depth: 16, channels: 2,
                          framing: AudioFraming.AudioPhilips, codec_is_master: false }
    // The part answers on I2C only while SYS_MCLK runs, so the board's
    // I2S block is started first.
    let _ = part.start_clock(f)
    if not part.begin()
        hal.uart.write("no answer from the codec at 0x0A\n")
        return
    if part.codec_set_power(true) and part.codec_set_format(f)
        let set = part.codec_set_gain(CodecPath.CodecOutput, MilliDb { mdb: -20000 })
        if set != None
            // After the first buffer is queued the output is opened.
            let _ = part.codec_set_mute(false)
```

The full program, which also fills and sends the buffers, is
`examples/headphones.nv`. A board package that depends on this package builds
it as its `main`. `tests/device_build.sh` does so from a scratch package for
the two boards below and for the emulated board, where it runs the image and
checks that the program reports that the codec does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 15,192 B | 4,356 B | 3,236 B |
| `nrf54lm20-dk` | 16,656 B | 4,788 B | 3,236 B |

The numbers are from the build's memory report of this program, 2026-11-11,
toolchain 0.24.1. Flash also holds the runtime, the compiler's support
library and the program's tables. The build drops the functions the program
does not call. The arena is the runtime's heap and is reserved whether or not
the program uses it.

**The I2S block is absent hardware on both boards; the I2C bus is not.**
embedded-hal-nv has a trait for an I2S interface and no board handle for one,
and neither Nordic board package drives its I2S block: no board implements
`I2s` or `PdmIn` yet, and there is no `hal.i2s` hook. The Nordic DKs drive the
control port's I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26 and SCL
P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04), on the pins
`docs/hal/README.md` names under "The Nordic DKs' buses", so both images carry
the driver's reading path. The program is written against the `I2s` trait and
is given `NoI2s`, so on both images `begin` runs on the bus, finds no codec
and prints `no answer from the codec at 0x0A`. A reply from the codec over a
DK's bus is NOT VERIFIED until the driver rig carries one. The one argument of
`attach` is what changes when a board package gives an I2S handle.

### The effects

Every function of the driver that reaches the bus or the I2S handle declares
`[hw]`, and nothing else. The driver performs no input or output of its own
and keeps no state: the registers can be read, and `CHIP_SSS_CTRL`, whose
I2S_SELECT field cannot, is written whole. `attach` allocates one cell of the
arena, because a struct with a generic field cannot be a `@value`; the
sequences build only the four-byte frames of devices-nv's 16-bit register
functions, in the caller's frame. The module checks at every tier, `embedded`
included.

## What it is

The SGTL5000 has a stereo ADC behind a line input or a microphone amplifier, a
stereo DAC, a headphone amplifier (capless or not), a line output, a signal
processor for the sound effects, an internal PLL and an audio switch that
routes I2S, ADC, DAC and the processor to each other. It runs from 8 to 96 kHz.
The I2C address is 0x0A with CTRL_ADR0_CS low and 0x2A with it high; the 20-pin
package has no such pin and answers at 0x0A. Every register has a 16-bit
address and 16 bits of data, high byte first. The chip ID is PARTID 0xA0 in the
high byte of register 0 and the revision in the low byte; `device_id` answers
0xA0 as `who` and the revision as `rev`.

**The part has no reset and answers on I2C only while SYS_MCLK runs.** The
internal reset ends 8 SYS_MCLK cycles after the supplies are up. So the board
starts its I2S block first, with `start_clock(f)`, and calls `begin` after.

### The power-up

`codec_set_power(true)` follows the data sheet's "Chip Powerup and Supply
Configurations" in its order. The steps depend on the supplies, which the
`Sgtl5000Config` names (`config_default(mclk_hz)` is VDDA and VDDIO at 3.3 V
with VDDD made by the chip):

| Case | Writes |
|---|---|
| VDDD made by the chip | LINREG_CTRL 0x0008, ANA_POWER 0x7260 |
| VDDD external | ANA_POWER 0x4060 |
| VDDA and VDDIO both below 3.1 V | CLK_TOP_CTRL ENABLE_INT_OSC, ANA_POWER with the charge pump bit |
| VDDA and VDDIO both at 3.1 V or more | the charge pump source set to VDDIO in LINREG_CTRL |
| always, then | REF_CTRL (VAG = VDDA/2, bias), LINE_OUT_CTRL, REF_CTRL slow ramp, SHORT_CTRL 0x1106, ANA_CTRL 0x0133, ANA_POWER (the blocks), DIG_POWER 0x0073, a wait of 400 ms, LINE_OUT_VOL, SSS_CTRL, ADCDAC_CTRL muted with the ramp, DAC_VOL at 0 dB, ANA_HP_CTRL at its lowest |

For VDDIO 3.3 V, VDDA 1.8 V and an internal VDDD the writes are the data
sheet's example, byte for byte: 0x0008, 0x7260, 0x004E, 0x0322, 0x004F,
0x1106, 0x0133, 0x6AFF, 0x0073 and 0x0505. `begin` runs a first step,
`step_quiet`: it mutes everything and, if VAG was left powered by an earlier
run, switches it off and waits 500 ms for the ramp down, because writing
ANA_POWER with it on pops.

### The clock

SYS_MCLK is 256, 384 or 512 times SYS_FS (256 only at 96 kHz), where SYS_FS is
32, 44.1, 48 or 96 kHz. RATE_MODE divides it by 2 or 4 for the other rates,
and the ratio stays relative to SYS_FS, so 16 kHz is SYS_FS 32 kHz halved and
its synchronous clock is 8.192 MHz, not 4.096 MHz. Any other clock from 8 to
27 MHz goes through the PLL, which needs the part as I2S master
(`codec_is_master: true`):

| SYS_MCLK | Rate | CLK_CTRL | PLL INT_DIVISOR | FRAC_DIVISOR |
|---|---|---|---|---|
| 12.288 MHz | 48 kHz | 0x0008 | | |
| 11.2896 MHz | 44.1 kHz | 0x0004 | | |
| 8.192 MHz | 16 kHz | 0x0010 | | |
| 12 MHz | 48 kHz | 0x000B | 16 | 786 |
| 12 MHz | 44.1 kHz | 0x0007 | 15 | 108 |
| 26 MHz (halved) | 48 kHz | 0x000B | 15 | 253 |

`Sgtl5000Config.force_pll` takes the PLL for a synchronous clock too. The
format step writes the divisors, powers the PLL, waits up to 50 ms for the
lock, and only then selects the PLL in CLK_CTRL.

## What the package contains

| Module | What is in it |
|---|---|
| `sgtl5000` | `Sgtl5000Config`, `Sgtl5000Port`, the 16-bit I2C framing, the register access, the supply and clock plans, the named steps, the gain functions, the type `Sgtl5000` and its traits |

## How to choose an entry point

A program on a board builds an `Sgtl5000Config` with `config_default(mclk_hz)`,
passes it with the board's I2C bus, the wiring and the I2S handle to `attach`,
and uses the traits on the result.

| Method | What it does |
|---|---|
| `attach(bus, w, i2s, cfg)` | the codec at `w.addr` and the handle that carries its samples; nothing is sent |
| `start_clock(f)` | configures and enables the I2S handle so that SYS_MCLK runs; before `begin` |
| `begin()` | the bus ready, the chip ID read, the part muted and VAG switched off if it was on |
| `codec_set_power(on)` | the data sheet's power-up with everything muted, or the power-down, and disables the handle |
| `codec_set_format(f)` | configures and enables the I2S handle, then the clock registers, the PLL if needed, and the serial port |
| `codec_set_mute(on)`, `codec_is_muted()` | the DAC's soft mute and the headphone and line-out mutes, in the order that gives no click |
| `codec_set_gain(path, g)`, `codec_gain(path)` | the headphone volume, or the ADC volume, to the largest step at or below `g` |
| `device_id()` | `Identifiable`: PARTID 0xA0 and the revision |
| `set_dac_volume(g)`, `set_ramp(on, expo)`, `set_lineout_level(code)` | the DAC volume, its ramp, the line-out level |
| `set_route(r)`, `set_mic_input(mic)`, `set_hp_bypass(line)`, `set_zero_cross(hp, adc)` | the audio switch, the ADC input, the analogue bypass, the zero-cross detectors |
| `set_mic_gain(g)`, `set_mic_bias(ohms, mv)` | the microphone amplifier and bias |
| `set_rj_format()`, `set_pcm_format(b)` | the right-justified and PCM formats |
| `locked()`, `shorted()` | the PLL lock and the headphone short flags |
| `reg(r)`, `set_reg(r, v)`, `i2s_stop()` | one register, a write, and the handle's clocks stopped |

The steps are functions over `Sgtl5000Port`, a trait a type holding an I2C bus
implements, so a test runs them over a double of the part: `step_quiet`,
`step_power_up`, `step_clock`, `step_i2s` and `step_power_down`.

## The gains

The output gain of `AudioCodec` is the headphone volume of both channels,
+12 dB at code 0 to -51.5 dB at code 127 in 0.5 dB steps; 0 dB is code 24. The
input gain is the ADC's analogue volume of both channels, 0 to +22.5 dB in
1.5 dB steps, and -6 to +16.5 dB with the range shifted by ADC_VOL_M6DB, which
the driver uses for a request below 0 dB. The gain a call sets is never above
the request, and a request below the range sets nothing. The power-up leaves
the headphone at -51.5 dB and the DAC at 0 dB, so a program raises the
headphone volume before it unmutes. The DAC's volume is `set_dac_volume`, 0 to
-90 dB. The line out has a level relative to its reference and not a gain,
`set_lineout_level`, and the power-up sets it from the supplies by the data
sheet's formula, 40 log10(VAG / reference) + 15, rounded up. The microphone
amplifier is `set_mic_gain`: 0, 20, 30 or 40 dB.

## The rules a user needs

1. Start the board's I2S block (`start_clock`) before `begin`. The part does
   not answer I2C without SYS_MCLK.
2. Call `codec_set_power(true)` and `codec_set_format` in either order. The
   power-up keeps the PLL bits a format has set.
3. `codec_set_power(true)` leaves everything muted. Open the output with
   `codec_set_mute(false)` once the first buffer is queued. The unmute sets the
   analogue unmutes first and then ramps the DAC up; the mute ramps the DAC
   down, waits for it to end and then sets the analogue mutes.
4. A format is 16 or 32 bits and two channels: 16 bits are 32 bit clocks per
   frame and 32 bits are 64. The part also has 20 and 24 bits in a 64-clock
   frame, which `I2sConfig` cannot ask the handle for because it has one bit
   depth and no slot width, so they are refused.
5. The PLL needs the part as the I2S master, which means the board handle is
   the slave.
6. `AudioFraming` names the I2S and left-justified formats only.
   `set_rj_format` and `set_pcm_format` cover the others.

## What is not included

- The signal processor (DAP): the surround, bass enhance, the parametric and
  graphic equalisers, the automatic volume control and the dual input mixer.
  The registers from 0x0100 are not written, and DIG_POWER powers the block as
  the data sheet's example does.
- The pad drive strengths (CHIP_PAD_STRENGTH), the test registers, the ADC
  high-pass filter bits, the left-right swaps of CHIP_SSS_CTRL, and the DAC
  mono mode.
- The 20 and 24 bit data lengths, mono formats, and the SPI port.
- The settling wait of `AudioCodecAsync.codec_power_up`: the power-up's wait of
  400 ms is taken in the synchronous step, and no timer-parking body is written.
- The VDDD level of the chip's own regulator: the data sheet asks for
  LINREG_SIMPLE_POWERUP and STARTUP_POWERUP to be cleared for the
  D_PROGRAMMING level to take effect, and the power-up leaves the first set.
  The VDDD level is NOT VERIFIED.

## Where the traits fall short

- `AudioCodec` has one output gain and one input gain and no input mute. The
  output gain here is the headphone; the DAC volume, the line-out level and
  the microphone amplifier are the driver's own functions. The part has no
  input mute apart from the ADC's analogue mute, which the driver does not
  offer separately.
- `AudioFraming` has two framings; the part also has right-justified and PCM.
- `I2sConfig` has one `bit_depth`, which is both the slot width and the sample
  width, so the 24-bit data of the part's 64-clock frame cannot be asked for.
- A trait method gets a plain `self`, so the driver keeps no state between
  calls; the part's registers are readable, so it needs none.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `AudioCodec` and `Identifiable` traits this driver implements and the wiring
  record and register helpers it uses, among them the 16-bit register access.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `I2cBus` and `I2s` traits and the board handles.
- [wm8960-nv](https://novo-lang.org/packages/wm8960-nv) and
  [es8388-nv](https://novo-lang.org/packages/es8388-nv) drive stereo codecs
  with the same trait; [es8311-nv](https://novo-lang.org/packages/es8311-nv) a
  mono one.

## Tests

`novo test` runs the module over a double of the part, a file of 16-bit
registers that starts from the data sheet's reset values and whose log keeps
every transaction, and a double of the I2S handle: the framing of reads and
writes, the chip ID, the supply codes and the line-out level against the data
sheet's Table 32, the clock plan at the synchronous ratios, at the rate modes
and at the PLL (12 MHz at 48 kHz is 16 and 786), the packing of CLK_CTRL and
I2S_CTRL, the power-up as an exact register log for the data sheet's example
and for the supplies of 3.3 V and 1.8 V with VDDD internal and external, the
power-down, the format with and without the PLL, the mute and unmute orders
with the volume ramp busy and never ending, the gains at their steps and
limits, the routes, the microphone, and the type over absent hardware. Every
refusal of a write at each position of the power-up, the power-down, the format
and the mute is checked. The suite covers every line of `src/`;
`bash tests/coverage.sh` prints the number.

**Hardware: NOT VERIFIED.** See the device program section.

The sources read for this driver, for their facts and not ported, are NXP's
data sheet above, the Linux kernel's `sound/soc/codecs/sgtl5000.c` and the
Teensy Audio Library's `control_sgtl5000.cpp`. The fixes and differences kept
are named in `src/sgtl5000.nv` where they are kept:

- A VAG left powered by an earlier run is switched off, with the outputs
  muted, and given 500 ms to ramp down before ANA_POWER is written (Linux),
  where the data sheet's sequence assumes a part just reset.
- The output rises from silence: the headphone volume is lowest at power-up,
  the DAC is muted with its ramp on, and the unmute sets the analogue side
  first (the Teensy library also starts with the headphone volume at its
  lowest).
- The power-up waits 400 ms for the slow VAG ramp before the volumes are set,
  as the Teensy library does; the data sheet gives no time.
- The charge pump bit is on unless both supplies are at 3.1 V or more. The data
  sheet's own example sets it for VDDIO 3.3 V and VDDA 1.8 V (0x6AFF), and
  Linux and the Teensy library leave it off when either supply is at 3.1 V.
- The external-VDDD write is 0x4060, as Linux and the Teensy library have it,
  and not the data sheet's 0x4260, which keeps the internal regulator's bit
  set.
- The line-out level uses the ratio of the voltages. Linux divides the register
  codes, which is a different number.
- CHIP_SSS_CTRL is written whole, as its I2S_SELECT field is write-only; the
  Teensy library keeps a copy of CHIP_ANA_CTRL, which this driver reads back.
- The supply bias is -50% only below 2.7 V of VDDA, where the data sheet's
  example uses it at 1.8 V, because of its warning against -50% in the 3.0 V
  mode.

## Licence

Apache-2.0.
