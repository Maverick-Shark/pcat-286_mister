# Chameleon96 port

Build target for the Arrow/NovTech Chameleon96 (Cyclone V SE 5CSEBA6U19I7,
512 MiB DDR3L on the HPS, no SDRAM connector). The DE10-nano project is
unchanged: every board difference is behind `MISTER_CHAMELEON96`.

Status: sources prepared and checked with a SystemVerilog front end only.
**Not yet compiled in Quartus and not yet run on a board.**

## Building

Open `PCAT-286_ch96.qpf` in Quartus Prime 17.1 (Lite is enough) and compile.
The bitstream is `output_files/PCAT-286_ch96.rbf`.

The project is `PCAT-286.qsf` with:

- `MISTER_CHAMELEON96` and `MISTER_DISABLE_ALSA=1` defined;
- device, pins and HPS locations from `sys/pins_ch96.tcl` and
  `sys/hps_ch96.tcl` instead of `sys/sys.tcl` and `sys/sys_analog.tcl`;
- no SDRAM pin assignments.

`PCAT-286_ch96.qsf` keeps the 326 register placement constraints of the
DE10-nano project. Both devices are the same 5CSEBA6 die in different
packages; if the fitter rejects a location, remove the `FF_X*` lines.

## What changes on this board

| Area | DE10-nano | Chameleon96 |
|---|---|---|
| 50 MHz clocks | three oscillator pins | one net from HPS `h2f_user1_clk` |
| Video | ADV7513, RGB888 | TDA19988, RGB565 (`LCD_R/G/B`) |
| HDMI I2C | HPS I2C through FPGA pins | same atom, pins U7/U10 |
| LEDs | `LED[7:0]` pins | user LEDs through HPS LoanIO, RF LEDs on Y19/Y20 |
| SDRAM, VGA, analog audio, SDIO, ADC, user port, I/O board | pins | internal nets, tied off |
| ALSA audio from Linux | yes | no |

The user port lines read as idle (all ones), so COM2 and the hardware MIDI
output have no pins. MT32-pi and the 15 kHz CRT outputs need hardware the
board does not have.

## DDR3 map

Linux owns `0x00000000-0x0FFFFFFF`. The board device tree reserves
`0x10000000-0x1FFFFFFF` (`no-map`) for the FPGA.

| Use | DE10-nano | Chameleon96 |
|---|---|---|
| CRT conversion framebuffer (3 x 2 MiB) | `0x30000000` | `0x10000000` |
| AT RAM (16 MiB + EMS) | `0x31000000` | `0x11000000` |
| MiSTer framebuffer | `0x22000000` | `0x1D000000` |
| Scaler buffers (`RAMBASE`) | `0x20000000` | `0x1E800000` |

The first two are set in `PCAT-286.sv` (`AT_DDR_FB_BASE`, `AT_DDR_RAM_BASE`,
64-bit word addresses); the scaler base is set in `sys/sys_top.v`.

## Timing constraints

`sys/sys_top.sdc` detects the board macro. The single HPS-fed 50 MHz clock
is named `FPGA_CLK2_50`, so `SYSTEM.sdc` applies unchanged apart from the
SDRAM and SDIO pin constraints, which are skipped.

## Software side

The core needs a Main_MiSTer built for the Chameleon96 that also knows the
`PCAT-286` core name (`is_pcat286()`), that is, teiram's `Main_MiSTer_ch96`
merged with a MiSTer-devel `Main_MiSTer` from November 2026 or later.
