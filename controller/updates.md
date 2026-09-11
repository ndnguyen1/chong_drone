# SpeedyBee F405 V4 — Bare-Metal Firmware Learning Log

## What this document is

This is a **learning log** for a from-scratch, bare-metal firmware project on a
SpeedyBee F405 V4 flight controller. The end goal (eventually) is a custom flight
controller; the **first concrete milestone is reading the gyro**. This file records
both the *technical progress* and the *way we're learning it*, so any future session
can continue in the same style.

## How we're doing this (teaching approach — please preserve this)

This is a **guided learning exercise, not a code-delivery task.** The working
agreement is:

- **Do NOT hand over finished code or answers.** Claude acts as a *teacher/mentor*,
  not a code generator. The learner writes every line themselves.
- **Teach from first principles**, assuming no prior knowledge — build intuition
  *before* syntax. Use analogies (e.g. flash = printed textbook, RAM = whiteboard).
- **Go one concept at a time**, in digestible "lessons," and **pause to check
  understanding** with small questions before moving on.
- **Socratic style:** ask the learner to reason, predict, and answer in their own
  words. Correct misconceptions gently and specifically.
- When pointing at a file, teach the learner *how to find/derive* the answer
  themselves (e.g. how to read a startup file to discover the linker contract) rather
  than just stating it.
- The learner is an **electrical engineer** — leverage that (clocks, buses, signals,
  registers are familiar territory; register-level programming = "applied EE").

If picking this up fresh: keep this tone. The learner explicitly asked for it.

## Hardware facts established

- **MCU:** STM32F405RGT6 — ARM Cortex-M4, 1 MB flash, 128 KB main SRAM (+64 KB CCM,
  left alone for now).
- **Flash:** origin `0x08000000`, length `1024K` (confirmed from the DFU device
  descriptor: `04*016Kg,01*064Kg,07*128Kg` = 1 MB).
- **RAM:** origin `0x20000000`, length `128K`.
- **Gyro:** likely ICM-42688-P (some units shipped BMI270). *To be confirmed at
  Rung 6 by reading the WHO_AM_I register* — don't assume.
- **Gyro connection:** SPI (find exact bus + pins from the schematic / Betaflight
  `SPEEDYBEEF405V4` target config at Rung 3).

## Tooling / environment

- **Host:** macOS. Compiler: `arm-none-eabi-gcc` (bare-metal ARM toolchain).
- **Flashing:** USB-only (no ST-Link yet) → **DFU via `dfu-util`**.
  - Enter bootloader: hold BOOT while plugging in USB.
  - Verify: `dfu-util -l` shows `[0483:df11]`, `alt=0` = `@Internal Flash /0x08000000`.
  - Flash: `dfu-util -a 0 -s 0x08000000:leave -D firmware.bin`
  - Note: this overwrites the stock Betaflight (reversible via Betaflight Configurator).
- **No SWD debugger** → no gdb halt/inspect. At Rung 1, "seeing values" will need a
  **USB-TTL serial adapter** (~$5) for UART output. Worth ordering now. (Not needed
  for blinky — the LED is the output.)
- **Abstraction level chosen: CMSIS** (register-name headers + ST startup file),
  NOT HAL/LL. Learner writes all peripheral logic directly against registers.

## The roadmap (the "rung ladder")

Each rung is independently verifiable; don't advance until the current one works.

- **Rung 0 — Toolchain + blinky** ← *currently here*
- Rung 1 — A way to see values (UART → USB-TTL serial)
- Rung 2 — Clock setup (PLL, know the crystal freq)
- Rung 3 — Read the hardware on paper (gyro chip, SPI bus + pins, datasheet)
- Rung 4 — GPIO + SPI peripheral init (CPOL/CPHA, clock speed, CS as GPIO)
- Rung 5 — Send/receive one SPI byte (prove the plumbing)
- Rung 6 — **Read WHO_AM_I** 🎯 (proves the whole chain; confirms the chip)
- Rung 7 — Wake + configure the gyro (power mgmt, full-scale range)
- Rung 8 — Read the X/Y/Z data registers (burst read, signed 16-bit)
- Rung 9 — Convert raw → °/s and sanity-check (still ≈ 0, rotate → spikes)

## Rung 0 progress

Sub-steps: 0a toolchain → 0b anatomy/CMSIS → 0c find LED pin → 0d blink logic →
0e build → 0f flash → 0g verify.

**Done:**
- DFU flashing path verified (`dfu-util -l` sees the board in bootloader mode).
- CMSIS files gathered into `controller/CMSIS/Include/` (core + device headers).
- Missing files copied in from the full STM32CubeF4 package
  (`/Users/miffy/chong_drone/STM32CubeF4/`): `system_stm32f4xx.h`,
  `system_stm32f4xx.c`, `startup_stm32f405xx.s`.

**In progress:** writing the **linker script** (`linker.ld`) — see lessons below.

**Still to write (learner's own files):** `linker.ld`, `main.c`, `Makefile`.

## Project file layout

```
controller/
├── CMSIS/Include/          ← ARM core + STM32F4 device headers (incl. system_stm32f4xx.h)
├── startup_stm32f405xx.s   ← vector table / reset handler (from CMSIS, GCC variant)
├── system_stm32f4xx.c      ← SystemInit()
├── main.c                  ← learner writes (blink logic)
├── linker.ld               ← learner writes (memory map) — NEXT
├── Makefile                ← learner writes (compile → link → objcopy)
└── updates.md              ← this file
```

Build gotchas noted for later:
- Must compile with **`-DSTM32F405xx`** or the register headers won't resolve.
- Startup calls `SystemInit()` (from `system_stm32f4xx.c`) and `__libc_init_array`
  (from the C library — link with `--specs=nano.specs --specs=nosys.specs`).

---

# Lessons so far

## Lesson 1 — The memory model (mental model of a bare-metal chip)

- **No OS = no butler.** On a PC the OS loads your program and places everything.
  On the bare F405, nothing does — *you* provide that setup (that's what the linker
  script + startup code are for).
- **The linker script is a floor plan of memory** — it says what goes where.
- **Two kinds of memory, fundamentally different:**
  - **FLASH = printed textbook** — permanent (survives power-off), read-only at
    runtime. Holds the **program + constants**. (1 MB @ `0x08000000`)
  - **RAM = whiteboard** — fast, read/write, but **wiped blank on power-off**.
    Holds **variables + the stack**. (128 KB @ `0x20000000`)
- **The one magic address:** at power-on the CPU mechanically reads the first 8 bytes
  at `0x08000000`: first 4 = **initial stack pointer**, next 4 = **address to start
  executing** (reset handler). This table is the **vector table** (`.isr_vector`), and
  it **must be placed first, at `0x08000000`**, or the chip boots into garbage.

## Lesson 2 — `.data` vs `.bss` (the two-homes puzzle)

The puzzle: `int speed = 100;` must *live* in RAM (it changes), but RAM is blank at
boot. So where does `100` come from?

- **`.data` = initialized variables** (e.g. `speed = 100`, `gain = 0.8`). Their values
  are **stored in flash** (the textbook) and **copied into RAM at boot** (transcribed
  onto the whiteboard). They have **two homes**: a flash storage copy + a live RAM copy.
- **`.bss` = zero-initialized variables** (e.g. `count;`, `buffer[256];`). Start at 0,
  so there's **nothing worth storing in flash** — startup just **blanks that RAM region
  to zero**. Costs zero flash.
- `.data` and `.bss` are **two separate groups of variables**, not a source/destination
  pair. Both live in RAM; the difference is only "was the initial value worth storing in
  flash?"

| Kind | Section | In flash? | In RAM? | Startup does |
|------|---------|-----------|---------|--------------|
| code + constants | `.text` | ✅ | — | nothing (runs from flash) |
| initialized vars | `.data` | stores values | ✅ | **copies** flash → RAM |
| zero-init vars | `.bss` | — | ✅ | **zeros** the region |

**Full boot story:** CPU reads SP + reset vector → reset handler copies `.data`
(flash→RAM) and zeros `.bss` → calls `SystemInit()` → calls `main`.

## Lesson 3 — Reading the startup file to find the "contract"

The startup `.s` and the linker script are two halves of a contract: the startup code
**uses** boundary addresses; the linker script must **define** them.

**Technique to find the contract:** find symbols that are **referenced but never
defined** in the startup file.

Four assembly idioms needed:
- `Name:` (colon) — **defines** a symbol *here* (internal; linker doesn't supply it).
- `ldr rX, =Name` — load the **address** of `Name` into rX. (How boundary symbols are
  referenced.)
- `.word Name` — place `Name`'s address as a literal (vector table, data pool).
- `bl Name` — call function `Name`.

**Procedure:** in `Reset_Handler`, list every `ldr rX, =something`; for each, search the
file for `something:`. Those *without* a colon definition are what the linker script owes.

**Reading assembly gotcha:** code does **not** run top-to-bottom — it jumps via labels.
The `.data`/`.bss` loops use the "jump to the test first, body sits above it" idiom, so
they don't *look* like loops when read straight down. You have to trace the branches.
The startup file even comments the two chores ("Copy the data segment…", "Zero fill the
bss segment.").

## The six contract symbols (what `linker.ld` must define)

Naming convention: `s` = start, `e` = end, `i` = init.

| Symbol | Reads as | Meaning | Lives in |
|--------|----------|---------|----------|
| `_sdata`  | start of data      | start of `.data` in RAM        | RAM |
| `_edata`  | end of data        | end of `.data` in RAM          | RAM |
| `_sidata` | start of init-data | **source** of `.data` values   | **FLASH** ⚠️ |
| `_sbss`   | start of bss       | start of `.bss` in RAM         | RAM |
| `_ebss`   | end of bss         | end of `.bss` in RAM           | RAM |
| `_estack` | end of stack       | **top of RAM**; SP starts here and grows *down* | RAM (top) |

Key nuances:
- **`_sidata` is the odd one out — it's a FLASH address** (the textbook copy of
  `.data`). It gets defined via the special "load address" (LMA) mechanism in the
  linker script. This is the trickiest line in the script.
- **`_estack` = the very top of RAM** (`0x20000000 + 128K`); the stack grows downward
  from there.

---

# Where we are now / next step

**Lesson 4 (in progress): writing `linker.ld`.** The script has two halves:
1. `MEMORY` — declare the physical regions (FLASH, RAM). ← learner is writing this now.
2. `SECTIONS` — place `.isr_vector`/`.text`/`.data`/`.bss` and **define the six
   contract symbols above**. (Harder half; `_sidata` via load-address is the crux.)

`MEMORY` block shape the learner is filling in:
```
MEMORY
{
  <NAME> (<permissions>) : ORIGIN = <address>, LENGTH = <size>
  ...
}
```
Permissions to reason about: `r` read, `w` write, `x` execute — flash vs RAM differ.

**Immediate next action:** learner writes the `MEMORY` block (real names, addresses,
lengths, reasoned permissions) → Claude checks it → then move on to `SECTIONS`.
