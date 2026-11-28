# Chameleon96 port

Build target for the Arrow/NovTech Chameleon96 (Cyclone V SE 5CSEBA6U19I7,
512 MiB DDR3L on the HPS, no SDRAM connector). The DE10-nano project is
unchanged: every board difference is behind `MISTER_CHAMELEON96`, and the
DE10-nano view of every edited source and constraint file is functionally
identical to upstream.

Status: sources checked with a SystemVerilog front end (both boards) and the
SDC files dry-run with mocked TimeQuest commands (both boards).
**Not yet compiled in Quartus and not yet run on a board.**

## Building

Open `PCAT-286_ch96.qpf` in Quartus Prime 17.1 (Lite is enough) and compile.
The bitstream is `output_files/PCAT-286_ch96.rbf`.

The project is `PCAT-286.qsf` with:

- `MISTER_CHAMELEON96` and `MISTER_DISABLE_ALSA=1` defined;
- device, pins and HPS locations from `sys/pins_ch96.tcl` and
  `sys/hps_ch96.tcl` instead of `sys/sys.tcl` and `sys/sys_analog.tcl`;
- no SDRAM pin assignments.

`sys_top` stays the top-level entity (teiram's `Template_MiSTer_ch96` wraps
it in `ch96_top` instead). This core's SDC and its 326 `FF_X*` placement
constraints name instances relative to `sys_top` (`ascal:ascal|...`,
`emu_at:emu|...`), and a wrapper would break all of them.

The `FF_X*` constraints place the `hps_io` `video_calc` registers as in the
DE10-nano fit. Both devices are the same 5CSEBA6 die in different packages;
if the fitter rejects a location or struggles, remove those lines.

## What changes on this board

| Area | DE10-nano | Chameleon96 |
|---|---|---|
| 50 MHz clocks | three oscillator pins | one net from HPS `h2f_user1_clk` |
| Video | ADV7513, RGB888 | TDA19988, RGB565 (`LCD_R/G/B`) |
| HDMI PLL power-up / timing target | 148.5 MHz (1080p) | 74.25 MHz (720p), as the template |
| HDMI OSD tint | `OSD_COLOR` 4 | `OSD_COLOR` 1, as the template |
| HDMI I2C | HPS I2C through FPGA pins | same atom, pins U7/U10 |
| LEDs | `LED[7:0]` pins | user LEDs through HPS LoanIO, RF LEDs on Y19/Y20 |
| Fan | - | ~50 % PWM on HPS GPIO49 (LoanIO), as the template |
| SDRAM, VGA, analog audio, SDIO, ADC, user port, I/O board | pins | internal nets, tied off |
| ALSA audio from Linux | yes | no |
| 350-line CRT conversion (DDR3 capture + 15 kHz raster) | yes | not built |
| Splash logo / credits title | "MiSTer" (`splash_vga_320x200.hex`, `msg.bin`) | "ch96" (`splash_vga_320x200_ch96.hex`, `msg_ch96.bin`) |

Main_MiSTer reprograms the HDMI PLL for the selected video mode, so the 74.25
MHz default only sets the power-up clock and the frequency TimeQuest checks.
The board image runs 720p60; a 1080p build needs the 148.5 MHz default back
and timing closed at that frequency.

The fan PWM needs a preloader that lends HPS GPIO49 to the FPGA (hps-ch96
97e5fe4 or later). With an older preloader the pin is simply not driven.

### OSD on this board

Hidden, because the hardware is missing:

- **2nd SD card** (no secondary SD slot; forced to *Disable*).
- **USER I/O** and the **MT32-pi** page (no user port; MT32-pi bridge and
  COM2-on-USER-I/O are off whatever the stored setting says).
- **350-line CRT** (the conversion is not built).

Still available:

- **COM2** through the MiSTer UART (modem, PPP, console), as on the DE10-nano
  when USER I/O is not set to COM2.
- **MPU-401 MIDI** through the MiSTer UART in MIDI mode (MidiLink). External
  USB MIDI works; the software synths need ALSA and stay silent.
- **CRT H/V offset**: besides the 350-line conversion they also shift the
  native raster (CRTC and mode 13h), so they stay.

## DDR3 map

Cross-checked with the board map (`DDR_MAP.md`, Nov 2026). Linux owns
`0x00000000-0x0FFFFFFF`; the device tree reserves `0x10000000-0x1FFFFFFF`
(`no-map`) for the FPGA. Only one RBF is loaded at a time, so the ao486
window (`0x10000000-0x1B7FFFFF`) is free while this core runs.

Everything that touches DDR3 while PCAT-286 is loaded on the Chameleon96:

| Range | Who | Access |
|---|---|---|
| `0x100CE000-0x100CFFFF` | Main_MiSTer x86 share poll (the ao486 `shr_rgn`; Main runs `x86_poll(0)` for this core too) | reads; writes only after a valid request signature |
| `0x11000000-0x121FFFFF` | AT RAM, 16 MiB + 2 MiB EMS (owner's 25-bit address: 32 MiB worst case, up to `0x12FFFFFF`) | FPGA ram1, read/write |
| `0x1CFFF000-0x1CFFFFFF` | menu LFB palette (`LFB_BASE - 4 KiB`) | FPGA ram2, read |
| `0x1D000000-0x1E7B7FFF` | Main_MiSTer framebuffer (LFB) | FPGA vbuf, read while the LFB is shown |
| `0x1E800000-0x1FFFFFFF` | scaler buffers, 3 x 8 MiB (`RAMBASE`) | FPGA vbuf, read/write |

| Base | DE10-nano | Chameleon96 |
|---|---|---|
| AT RAM (`AT_DDR_RAM_BASE`) | `0x31000000` | `0x11000000` |
| 350-line CRT framebuffer (`AT_DDR_FB_BASE`) | `0x30000000` | not built (`0x10000000` reserved) |
| Scaler (`RAMBASE`) | `0x20000000` | `0x1E800000` |
| Main_MiSTer LFB | `0x22000000` | `0x1D000000` |

The AT RAM keeps the DE10-nano offsets modulo 512 MiB. That keeps the x86
share poll outside the AT RAM, exactly as on the DE10-nano, and the worst-case
32 MiB span far below the C64 GCR shadow (`0x1B800000`) and the LFB. The bases
are 64-bit word addresses in `PCAT-286.sv` (`byte = word << 3`); the scaler
base is in `sys/sys_top.v`.

## Timing constraints

`sys/sys_top.sdc` and `SYSTEM.sdc` detect `MISTER_CHAMELEON96` themselves.
The single HPS-fed 50 MHz clock is named `FPGA_CLK2_50`, so the AT
constraints apply as on the DE10-nano. Skipped on this board: SDRAM and SDIO
pin constraints, the analog-I/O false paths, the MCP23009 false path and the
350-line CRT configuration snapshot.

`SYSTEM.sdc` stops TimeQuest with `AT SDC required collection empty: ...`
when an instance it constrains is missing from the netlist. If a Chameleon96
compile stops there, the named instance was removed by synthesis; guard that
constraint with `$at_ch96` as done for `at_crt350_config_cdc`.

## Software side

The core needs a Main_MiSTer built for the Chameleon96 that also knows the
`PCAT-286` core name (`is_pcat286()`), that is, teiram's `Main_MiSTer_ch96`
merged with a MiSTer-devel `Main_MiSTer` from November 2026 or later.
