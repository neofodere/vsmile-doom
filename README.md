# DOOM for V.Smile

A port of DOOM to the VTech V.Smile (SPG200 / unSP architecture).

## What this is

A from-scratch port of the DOOM engine to the VTech V.Smile educational
console, targeting the Sunplus SPG200 SoC with its unSP 16-bit CPU.

**Hardware specs (confirmed from V.Dream and veesem emulator source):**
- CPU: unSP, 16-bit word-addressed, 27 MHz
- RAM: 20 KB (0x2800 words at 0x0000–0x27FF)
- ROM: cartridge, up to 48 KB usable code+data (0x4000–0xFFF5)
- Video: SPG200 PPU, 320×240, 2 BG layers + 256 sprites, 256-colour palette
- Our viewport: 160×92 pixels (PPU scales to fill the physical display)

## Repository contents

```
src/
  boot.s              unSP assembly entry point (reset vector → doom_main)
  doom/
    d_main.c          Main game loop and subsystem init
    g_game.c/h        Player movement and game state
    r_main.c/h        BSP renderer and column draw
  wad/
    w_wad.c/h         WAD directory reader (streaming from ROM, no heap)
  i_vsmile/
    i_video.c/h       PPU bitmap mode, framebuffer, palette upload
    i_input.c/h       GPIO controller polling
include/
  types.h             Width-exact integer types for bare-metal GCC
  vsmile_memmap.h     RAM/ROM layout constants
tools/
  unsp_isa.py         unSP instruction encoder (all encodings verified)
  unsp_asm.py         Two-pass assembler for cc1 output syntax
  compile.py          Build driver: cc1 + assembler + linker
  mkrom.py            ROM image builder (embeds WAD at build time only)
work/
  unsp-gcc/           GCC 2.95 with unSP backend (compiled from source)
  harness/            Encoder test suite using real V.Dream cpu.c
Makefile
```

## Building

### Prerequisites

- Python 3.8+
- GCC (host, for building the test harness)
- The unsp-gcc cross-compiler (built during project setup; lives in `work/unsp-gcc/`)
- `freedoom1.wad` from the [FreeDoom project](https://freedoom.github.io/)
  — **not included in this repository**

### Full build

```sh
make WAD=/path/to/freedoom1.wad
```

Output: `build/vsmile_doom.bin` — load this as the cartridge ROM in any
V.Smile emulator (V.Dream, veesem, VFrown).

### Encoder test suite (no WAD needed)

```sh
make check
```

Runs 12 round-trip tests: each test encodes a small program with our
Python encoder, executes it through the real V.Dream `cpu.c`, and checks
that registers and RAM contain the expected values.  All 12 must pass
before a build is considered valid.

### Build info

```sh
make info    # show toolchain paths and WAD size
make size    # show code binary size vs ROM budget
```

## Running

```sh
make run-vdream  # requires vdream in PATH
make run-veesem  # requires veesem in PATH
```

Or load `build/vsmile_doom.bin` directly as the cartridge in your preferred
emulator.  The ROM is self-contained: no external BIOS dump is required
because the reset vector at 0xFFF7 points directly into the cartridge code.

## Licences

**Game engine source code** (`src/`, `include/`, `tools/`): GNU General
Public License v2.0 only.  See `LICENSE` for the full text.  This engine
is derived from id Software's DOOM source code, which was released under
the GPL-2.0 in 1997.

**Game assets** (`build/vsmile_doom.bin` when built with FreeDoom): the
FreeDoom project assets embedded in the ROM image at build time are
distributed under the FreeDoom licence.  See `LICENSE-ASSETS` for the full
text.  The assets are **not** present in this source repository.

The separation is intentional: the source tree is GPL-2.0 only; the FreeDoom
licence applies only to release artefacts (the `.bin` file).

## Architecture notes

### Why 160×92 instead of 320×200

The SPG200 has 20 KB of RAM total for everything: stack, game state, and
framebuffer.  A full 320×200×8bpp framebuffer would require 62.5 KB — more
than three times the total RAM.  At 160×92 the framebuffer uses 7.4 KB
(packed two pixels per 16-bit word), leaving 12.6 KB for the BSP engine,
player state, and WAD directory cache.  The PPU's built-in bitmap scaling
fills the physical 320×240 display without any CPU cost.

### WAD streaming from ROM

The original DOOM caches entire lumps into a 4 MB heap.  We have no heap.
Instead, `w_wad.c` keeps a 32-entry directory cache in RAM (512 bytes) and
reads lump data directly through ROM pointers at runtime.  The WAD blob is
embedded into the cartridge ROM image by `tools/mkrom.py` at build time.

### Toolchain

The cross-compiler is GCC 2.95 with an unSP backend (`work/unsp-gcc/`).
`cc1` produces assembly in SunPlus native syntax; `tools/unsp_asm.py`
translates that to machine words and resolves all forward references.  Every
encoding in `tools/unsp_isa.py` was verified against two independent
emulators (V.Dream and veesem) before being used in the project.
