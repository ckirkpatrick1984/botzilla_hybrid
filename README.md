# botzilla_hybrid

Marlin firmware for **Botzilla** — a LulzBot TAZ 6 frame/motion system,
converted onto a **BigTreeTech SKR 3 EZ** (STM32H723VGT6) mainboard, with a
**Voron Stealthburner** toolhead running an **E3D V6** (knockoff) hotend, and
a 110V bed driven through a solid-state, zero-cross SSR.

This is a new, independent repo/milestone in the same overall project family
as [`lulzbot-marlin-bltouch`](https://github.com/ckirkpatrick1984) and
[`taz6-skr3ez-marlin`](https://github.com/ckirkpatrick1984/taz6-skr3ez-marlin)
— see `PROJECT.md` for the full background. This repo doesn't modify either
of those; both stay independently maintained as read-only references.

## Base source (verified, not guessed)

Plain upstream `MarlinFirmware/Marlin` release `2.1.2.4` — **not** LulzBot's
own fork. `Marlin/Version.h` reports `SHORT_BUILD_VERSION "2.1.2.4"`.

This is a deliberate departure from the LulzBot-fork lineage used by the
sibling repos: once the board (SKR 3 EZ), toolhead (Stealthburner + E3D V6),
and bed heating (SSR) are all non-stock, LulzBot's `LULZBOT_*` config
framework (Universal Toolhead I2C detection, bed-washer probing, per-model
`LULZBOT_Oliveoil_TAZ6` branching, etc.) mostly doesn't apply anymore. This
firmware instead hand-configures plain Marlin directly for Botzilla's actual
hardware.

## Provenance

Copied unmodified from
`/Users/ekirkpatrick/Projects/Botzilla Marlin 2.1.2.4/Marlin-2.1.2.4/` (a
folder renamed from the stale/misleading `Nicked Lulzbot Marlin 2.0.9.0.13`
— that name was left over from an earlier, since-replaced download and no
longer describes its contents). That folder is **confirmed by the user to be
the firmware currently flashed and running on the physical Botzilla
printer**, and stays untouched as the read-only baseline reference for this
repo. It was last known to build cleanly on 2024-12-12
(`.pio/build/STM32H723VG_btt/firmware.bin` present in that source).

`Marlin/Configuration.h` / `Marlin/Configuration_adv.h` already carry
extensive hand-tuning for Botzilla (see inline `//NF` notes throughout),
including:

- `MOTHERBOARD BOARD_BTT_SKR_V3_0_EZ`, `default_envs = STM32H723VG_btt`
- `CUSTOM_MACHINE_NAME "EXLAB Botzilla"`
- All axes (`X`/`Y`/`Z`/`Z2`/`E0`) on `TMC2209` drivers
- `X_BED_SIZE 290`, `Y_BED_SIZE 290`, `Z_MAX_POS 250`
- `TEMP_SENSOR_0 5` (ATC Semitec 104GT-2 — standard E3D V6-type thermistor),
  `TEMP_SENSOR_BED 7`
- Hotend PID already tuned (`Kp 22.20` / `Ki 1.08` / `Kd 114.00`)
- BLTouch already enabled: `BLTOUCH`, `USE_PROBE_FOR_Z_HOMING`,
  `AUTO_BED_LEVELING_BILINEAR`, `Z_SAFE_HOMING`,
  `NOZZLE_TO_PROBE_OFFSET { 0, 15, -3 }`
- `E0_AUTO_FAN_PIN FAN1_PIN` (hotend cooling fan on FAN1)
- `REPRAP_DISCOUNT_FULL_GRAPHIC_SMART_CONTROLLER` LCD

## What this repo adds on top

See `PROJECT.md` for the full task breakdown, but in short: a systematic,
documented comparison against `taz6-skr3ez-marlin` (LulzBot's own official
`bugfix-2.1.x` fork, retargeted to the same `BOARD_BTT_SKR_V3_0_EZ` board),
to identify specific, evaluated optimizations worth pulling in — e.g. TMC
stepper RMS current, UART/driver config, homing/thermal tuning — each
applied as its own reviewable, individually-buildable commit. Nothing is
adopted silently; every candidate is recorded with a recommendation before
being applied.

## PlatformIO environment

- `STM32H723VG_btt` — inherited from this source's `ini/stm32h7.ini`,
  targets the confirmed SKR 3 EZ chip variant (STM32H723VGT6).

## Not yet done

Compile-only work so far in this repo. Nothing has been (re-)flashed from
this repo's own commits yet — the printer is currently running the baseline
this repo started from. See `PROJECT.md`'s task breakdown and bring-up
checklist before flashing anything built here.
