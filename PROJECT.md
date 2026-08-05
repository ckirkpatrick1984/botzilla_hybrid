# Botzilla Marlin

## Overview

Botzilla is a LulzBot TAZ 6 frame/motion system converted onto a BigTreeTech
SKR 3 EZ (STM32H723VGT6) mainboard, with a Voron Stealthburner toolhead
running an E3D V6 (knockoff) hotend, and a 110V bed driven through a
solid-state, zero-cross SSR. This project mines a known-good, already
validated TAZ6-on-SKR3EZ conversion (`taz6-skr3ez-marlin`) for optimizations
worth pulling into Botzilla's actual, currently-running firmware — without
ever modifying either source.

## Goals

- A new, independent repo (`botzilla_hybrid`) seeded from Botzilla's actual
  currently-flashed firmware, so there's a clean, version-controlled record
  of the working baseline.
- A systematic, documented comparison against `taz6-skr3ez-marlin`'s
  TAZ6-on-SKR3EZ config, to find concrete optimization candidates (stepper
  current, driver config, homing/thermal tuning, etc.).
- Every adopted change applied as its own reviewable, individually-buildable
  commit — nothing adopted silently.
- Compile-verified at every step. Actual flashing/hardware bring-up stays a
  manual, staged process performed by the user.

## Non-goals

- Modifying `Nicked Lulzbot Marlin 2.0.9.0.13`/`Botzilla Marlin 2.1.2.4` (the
  baseline source) or `taz6-skr3ez-marlin` (the optimization source) — both
  stay independently maintained, read-only references.
- Re-deriving Botzilla's already-working, field-tuned values (hotend PID,
  E-steps, BLTouch probe offset, bed size) from scratch — they're treated as
  correct until there's a specific reason to revisit them.
- Redesigning the toolhead, bed, or SSR wiring — out of scope, same as the
  earlier BLTouch/SKR3EZ projects.

## Background

Two prior, independent projects lead into this one:

- [`lulzbot-marlin-bltouch`](https://github.com/ckirkpatrick1984) — added
  BLTouch support to the stock LulzBot Mini 1 and TAZ 6 firmware on their
  original (RAMBo/Mini-RAMBo) boards. Explicitly deferred an SKR 3 EZ port
  as future work.
- [`taz6-skr3ez-marlin`](https://github.com/ckirkpatrick1984/taz6-skr3ez-marlin)
  — took that deferred work on: retargeted stock TAZ 6 firmware (LulzBot's
  own `bugfix-2.1.x` fork, tag `v2.1.3.0.21`) onto `BOARD_BTT_SKR_V3_0_EZ`,
  with BLTouch added via a `LULZBOT_SKR3EZ_BLTOUCH` flag. Compile-verified
  (`TAZ6_SKR3EZ_BLTouch` env) but **not yet flash-tested on real hardware**.

Independently of both, the user's physical printer ("Botzilla") was already
converted in hardware — SKR 3 EZ board, Voron Stealthburner toolhead
(E3D V6 knockoff hotend), SSR-driven 110V bed — and its firmware was
hand-built directly against plain upstream Marlin `2.1.2.4` rather than
LulzBot's fork, since most of LulzBot's `LULZBOT_*` framework (toolhead
auto-detection, bed-washer probing, per-model branching) doesn't apply once
the board and toolhead are both non-stock. That firmware lived in a folder
misleadingly named `Nicked Lulzbot Marlin 2.0.9.0.13` (stale from an earlier,
since-replaced download) — renamed to `Botzilla Marlin 2.1.2.4` as part of
this project's first step, to stop misdescribing its contents.

**Key finding**: Botzilla's firmware and `taz6-skr3ez-marlin` independently
converged on several of the same decisions — both disable `SENSORLESS_HOMING`
(keeping physical mechanical endstops) and both rely on the SKR 3 EZ's
default `Z_MIN_PROBE_PIN`/`SERVO0_PIN` for BLTouch without any pin-repurposing
hack (confirmed: the two sources' `pins_BTT_SKR_V3_0_common.h` are
functionally identical, only trivial comment drift). This gives good
confidence the remaining differences are genuine, evaluable optimization
candidates rather than conflicting approaches.

**Concrete example already found**: Botzilla's `Configuration_adv.h` leaves
`X_CURRENT`/`Y_CURRENT`/`Z_CURRENT` at Marlin's stock default (`1000` mA),
while `taz6-skr3ez-marlin` carries forward LulzBot's own measured TAZ6 value
(`975` mA) from the original RAMBo config. User-confirmed Botzilla uses the
same X/Y/Z motors as stock TAZ6, so this was adopted (see below).

**Critical feature — independent dual-Z auto-leveling (do not lose this)**:
Botzilla's Z2 motor is deliberately wired through the SKR 3 EZ board's E1
driver slot (`Z2_DRIVER_TYPE TMC2209`, per the existing `//NF` note) so that
Z and Z2 can be driven **independently**, not as a mirrored pair. This is
what makes `Z_STEPPER_AUTO_ALIGN` (`Configuration_adv.h`) work — it adds the
`G34` command, which uses the BLTouch probe to auto-correct gantry tilt by
moving the two Z motors different amounts. This is a load-bearing feature
for Botzilla and must be preserved in any future config changes; it was
verified byte-identical to the untouched baseline after the TMC current
change below (current and independent step control are orthogonal — G34
works by commanding different step counts per driver, not different
current).

## Task breakdown / milestones

- [x] Renamed `Nicked Lulzbot Marlin 2.0.9.0.13` → `Botzilla Marlin 2.1.2.4`
      (stale name, wrong version, no longer LulzBot's fork).
- [x] New repo `botzilla_hybrid` created, git initialized, Botzilla's actual
      currently-flashed firmware copied in unmodified as the initial commit.
- [x] `README.md` / `PROJECT.md` written, documenting lineage and both
      read-only source references.
- [x] Build-verified the untouched baseline: `pio run -e STM32H723VG_btt`
      succeeds (Flash 19.9% / RAM 3.5%).
- [x] Systematic comparison pass vs. `taz6-skr3ez-marlin`'s
      `TAZ6_SKR3EZ_BLTouch` env — see the comparison table below.
- [x] Confirmed Botzilla-specific items that neither source fully covers
      (extruder E-steps, SSR bed heater — see below).
- [x] Applied TMC `RSENSE` correction (0.11→0.12) as its own commit;
      rebuilt, size unchanged (208812 bytes flash).
- [x] Applied TMC `X/Y/Z_CURRENT` (1000mA→975mA) as its own commit, after
      user confirmed Botzilla's X/Y/Z motors match stock TAZ6, and after
      verifying this doesn't affect the independent dual-Z auto-align
      feature (see Background). Rebuilt (208820 bytes flash).
- [x] Final build verification done; bring-up/flash checklist below.
- [ ] Flash-test on real hardware (user-performed — see checklist).

## Comparison findings (Botzilla vs. `taz6-skr3ez-marlin`)

| Setting | Botzilla (before) | `taz6-skr3ez-marlin` | Recommendation | Status |
|---|---|---|---|---|
| `X/Y/Z/E0_RSENSE` (`Configuration_adv.h`) | `0.11` (Marlin stock default) | `LULZBOT_RSENSE` = `0.12`, LulzBot's measured value for BTT EZ2209 modules | **Adopt** — same board/driver-module hardware, low risk, only affects current-register accuracy, not behavior | ✅ Applied (commit `0da1fb6`) |
| `X/Y/Z_CURRENT` (`Configuration_adv.h`) | `1000` mA (stock default) | `975` mA, LulzBot's measured TAZ6 value | **Adopt** — user confirmed Botzilla's X/Y/Z motors match stock TAZ6. Verified independent of the Z2/E1-driver-slot dual-Z auto-align setup (current and per-motor step control are orthogonal). | ✅ Applied (commit `0db885f`) |
| `HOMING_BUMP_DIVISOR` (`Configuration_adv.h`) | `{2, 2, 4}` | `{1, 2, 4}` | Minor (X-axis re-bump speed only) — low priority, skip unless homing repeatability becomes an issue | Skipped |
| `HOMING_FEEDRATE_MM_M` (`Configuration.h`) | `{75*60, 75*60, 10*60}` (75 mm/s X/Y) | `{50*60, 50*60, ...}` (50 mm/s X/Y, stock TAZ6 value) | Botzilla already runs homing faster than LulzBot's validated stock value, and it currently works — **not** a candidate to copy backward. Noted for awareness only, in case homing reliability is ever investigated. | Skipped (informational only) |
| `THERMAL_PROTECTION_BED_PERIOD` / `_HYSTERESIS` (`Configuration_adv.h`) | `20`s / `2°C` | `20`s / `2°C` | Identical — no action | No action |
| `Z_SAFE_HOMING_X/Y_POINT` (`Configuration.h`) | `X_CENTER` / `Y_CENTER` (auto-derives from actual bed size) | Hardcoded TAZ6-280×280-derived values | Botzilla's approach is already better — auto-adapts to its real 290×290 bed. **No action needed.** | No action |
| BLTouch pins (`Z_MIN_PROBE_PIN`/`SERVO0_PIN`) | Board defaults, no override | Board defaults, no override | Both rely on the SKR 3 EZ's own dedicated pins — confirmed identical `pins_BTT_SKR_V3_0_common.h` between sources (trivial comment-only drift). No action. | No action |
| `E0_AUTO_FAN_PIN` (`Configuration_adv.h`) | `FAN1_PIN` (explicit) | Board default | Consistent, not contradicted — no action | No action |
| `SENSORLESS_HOMING` | Disabled | Disabled | Both independently keep physical endstops — consistent, no action | No action |

## Botzilla-specific gaps confirmed

- **Stealthburner extruder E-steps** (`DEFAULT_AXIS_STEPS_PER_UNIT`, 4th
  value `725`): already present in the baseline. Since Botzilla prints
  successfully today, this is treated as field-calibrated — not touched.
- **SSR bed heater**: `PIDTEMPBED` is already enabled (standard PWM PID),
  which matches the user-confirmed solid-state, zero-cross SSR. No firmware
  change needed — `SLOW_PWM_HEATERS` (for mechanical/non-zero-cross relays)
  does not apply here.

## Open questions log

- ~~TMC current for the Z axis: needs physical confirmation Botzilla's Z
  motors match stock TAZ6~~ — resolved: user confirmed same motors, 975mA
  adopted for X/Y/Z (commit `0db885f`).
- Whether any further Stealthburner-specific tuning (part-cooling duct fan
  behavior, ADXL345 input shaping if present) is wanted — not currently
  configured in either source; flag if raised later.
- `Z_STEPPER_ALIGN_XY` / `Z_STEPPER_ALIGN_STEPPER_XY` (`Configuration_adv.h`,
  both commented out) — Botzilla currently relies on Marlin's computed
  defaults for the G34 probe/stepper positions rather than explicit tuned
  values. Not touched; flag if G34 alignment accuracy ever needs tuning.

## Hardware bring-up / flash checklist

Only relevant once/if this repo's build is intentionally reflashed onto the
physical printer (it currently differs from the running firmware by the
`RSENSE` and `X/Y/Z_CURRENT` corrections above — both low risk, but still
firmware changes):

- [ ] Re-review this repo's diff against `Botzilla Marlin 2.1.2.4`
      (`diff -r`) immediately before flashing, to confirm only the intended,
      documented changes are present.
- [ ] After flashing, specifically re-test `G34` (dual-Z auto-align) —
      this is the highest-value feature to protect. Confirm it still
      probes and aligns Z/Z2 correctly before trusting any print.
- [ ] Keep the currently-flashed baseline's `.bin` available as a rollback
      (already preserved, untouched, in `Botzilla Marlin 2.1.2.4/Marlin-2.1.2.4/.pio/build/STM32H723VG_btt/firmware.bin`).
- [ ] Flash `botzilla_hybrid`'s build; confirm boot, LCD comes up, and
      `M115`/machine name reports as expected.
- [ ] Verify each subsystem individually before printing: stepper
      directions, endstop triggers, thermistor readings (hotend + bed),
      heater outputs, BLTouch deploy/stow and a probe test
      (`M48` repeatability test).
- [ ] Re-run a bed mesh (`G29`) and confirm leveling looks sane before the
      first print.
- [ ] First test print, compare against a known-good print from the
      current baseline.

## Repo status

- `botzilla_hybrid` — new repo. Baseline commit + docs + two adopted
  optimizations (`RSENSE` correction, `X/Y/Z_CURRENT` correction), all
  build-verified (`STM32H723VG_btt`: Flash 208820 bytes / RAM 20032 bytes).
  Dual-Z auto-align (`Z_STEPPER_AUTO_ALIGN`/G34, Z2-via-E1-driver-slot)
  confirmed intact and unaffected. Not yet flashed on real hardware.
- `Botzilla Marlin 2.1.2.4` (renamed from `Nicked Lulzbot Marlin 2.0.9.0.13`)
  — untouched, read-only baseline reference. This is what's actually
  flashed and running on the printer today.
- `taz6-skr3ez-marlin` — untouched, read-only optimization reference. See
  its own `PROJECT.md` for its independent status.
