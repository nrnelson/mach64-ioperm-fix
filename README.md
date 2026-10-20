# mach64-ioperm-fix

Fix for `xf86-video-mach64` segfaulting in PreInit on Debian 12 (Xorg 21.1)
with a PCI ATI Rage XL whose block I/O sits above 0x3FF. Host: x2p3ntium
(Intel STL2, dual PIII, Debian 12 i386).

## Cause

Xorg's Linux/x86 `hwEnableIO()` only does `ioperm(0, 1024, 1)`. The driver's
first `inl()` at block I/O 0x1000 (and later `outb()` to 0x46E8) raises #GP,
reported as `SIGSEGV` at address 0. See `ISSUE.md` for the full write-up.

## Contents

| Path | What |
|---|---|
| `patches/0001-mach64-take-iopl-for-block-io.patch` | `iopl(3)` in `ATIMach64ProbeIO()`; applies with `-p1` to 6.9.7 |
| `debs/` | Built `6.9.7-1+ioperm1` i386 packages (+ dbgsym) |
| `config/20-mach64.conf` | `/etc/X11/xorg.conf.d/` config: depth 16, 1600x1200 @ 131 MHz |
| `evidence/` | strace, gdb backtraces, working Xorg log, system info |
| `ISSUE.md` | Draft upstream issue |

## Install

```sh
sudo dpkg -i debs/xserver-xorg-video-mach64_6.9.7-1+ioperm1_i386.deb
sudo apt-mark hold xserver-xorg-video-mach64
sudo cp config/20-mach64.conf /etc/X11/xorg.conf.d/
```

Keep the hold: a stock Debian update brings the crash back.

## Rebuild

```sh
apt-get source xserver-xorg-video-mach64
sudo apt-get build-dep xserver-xorg-video-mach64
cd xserver-xorg-video-mach64-6.9.7
patch -p1 < .../patches/0001-mach64-take-iopl-for-block-io.patch
dch -l +ioperm -D bookworm "Take iopl(3) for block I/O above 0x3FF."
dpkg-buildpackage -us -uc -b
```

## Config notes

- **Depth 16, not 24.** The driver caps the pixel clock by memory bandwidth
  (XCLK 62.8 MHz, 32-bit SGRAM): ~125 MHz at 32 bpp, 230 MHz at 16 bpp.
  1600x1200@60 needs 131-162 MHz.
- **`1600x1200R` (131 MHz)** is CVT reduced blanking with the sync nudged a
  few pixels. X rejects exact CVT-RB timings because the 2007FP's EDID 1.3
  lacks the RB flag, and mach64 ignores `Option "ReducedBlanking"`.
  The EDID `1600x1200` (162 MHz) mode is the fallback.
- Run the monitor's Auto Adjust after any mode change.
