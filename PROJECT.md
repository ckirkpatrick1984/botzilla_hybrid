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

- A new, independent repo (`botzilla-marlin`) seeded from Botzilla's actual
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
(`975` mA) from the original RAMBo config. Worth evaluating, with the caveat
that Botzilla's Z-axis setup may not exactly match stock TAZ6 (an existing
`//NF` note says Z2 routes through the board's E1 driver slot on this SKR 3
EZ wiring) — so this isn't a blind copy.

## Task breakdown / milestones

- [x] Renamed `Nicked Lulzbot Marlin 2.0.9.0.13` → `Botzilla Marlin 2.1.2.4`
      (stale name, wrong version, no longer LulzBot's fork).
- [x] New repo `botzilla-marlin` created, git initialized, Botzilla's actual
      currently-flashed firmware copied in unmodified as the initial commit.
- [x] `README.md` / `PROJECT.md` written, documenting lineage and both
      read-only source references.
- [ ] Build-verify the untouched baseline (`pio run -e STM32H723VG_btt`).
- [ ] Systematic comparison pass vs. `taz6-skr3ez-marlin`'s
      `TAZ6_SKR3EZ_BLTouch` env — documented per item below, each as
      adopt / skip / needs-verification:
  - [ ] TMC stepper RMS current (1000 mA stock default vs. LulzBot's
        measured 975 mA) — verify Z-axis wiring first given the Z2/E1
        driver-slot note.
  - [ ] TMC UART addresses / microstepping / driver config.
  - [ ] Endstop bump distance / homing feedrates / thermal protection
        windows.
  - [ ] `Z_SAFE_HOMING` XY point — needs Botzilla-specific recomputation for
        its actual 290×290 bed (not copyable from TAZ6's 280×280-derived
        value).
  - [ ] Fan pin/logic conventions cross-check (`E0_AUTO_FAN_PIN FAN1_PIN`
        already set on Botzilla).
- [ ] Confirm Botzilla-specific items that neither source fully covers:
  - [ ] Stealthburner extruder E-steps (`725` already present — treat as
        field-calibrated since the printer already prints; verify, don't
        casually change).
  - [ ] SSR bed heater stays on standard `PIDTEMPBED` PWM (matches the
        confirmed solid-state, zero-cross SSR — no change expected).
- [ ] Apply each adopted optimization as its own commit, rebuilding after
      each.
- [ ] Final build verification + staged hardware bring-up/flash checklist
      written into this file.
- [ ] Flash-test on real hardware (user-performed).

## Open questions log

- TMC current for Z axis specifically: needs physical confirmation of how
  Botzilla's Z motors are wired (given the Z2→E1-driver-slot note) before
  adopting LulzBot's 975 mA TAZ6 value wholesale.
- Whether any further Stealthburner-specific tuning (part-cooling duct fan
  behavior, ADXL345 input shaping if present) is wanted — not currently
  configured in either source; flag if raised later.

## Repo status

- `botzilla-marlin` — new repo, initial baseline commit only so far. Not yet
  build-verified in this repo (baseline was last known-good as of
  2024-12-12 in the source it was copied from). No optimizations from
  `taz6-skr3ez-marlin` applied yet.
- `Botzilla Marlin 2.1.2.4` (renamed from `Nicked Lulzbot Marlin 2.0.9.0.13`)
  — untouched, read-only baseline reference. This is what's actually
  flashed and running on the printer today.
- `taz6-skr3ez-marlin` — untouched, read-only optimization reference. See
  its own `PROJECT.md` for its independent status.
