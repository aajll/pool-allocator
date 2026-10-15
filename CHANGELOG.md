# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- CI cross-builds the library with TI C2000 CGT 22.6.2 and 25.11.1 under the COFF and EABI object formats and both lookup strategies, with warnings as errors.

- Tests for `pool_release` with a NULL handle, `pool_get_pointer_checked` with a NULL handle, NULL output pointer or out-of-range ID, and the hash strategy's recovery from an out-of-range cursor.

### Fixed

- `pool_footprint_t::addr_unit_bits` is `uint_least8_t` instead of `uint8_t`, so `pool.h` compiles where `CHAR_BIT == 16`. Byte-addressed targets see the same type.

### Changed

- CI gates line and branch coverage at 100 % across both lookup strategies, up from 80 % line and 70 % branch on the linear strategy alone.
- `pool_release` and `pool_get_pointer` no longer repeat the slot bounds check after the ID range check, and `pool_get_pointer_checked` relies on `pool_get_pointer` for the ID range check. Results are unchanged.

- CI runs a MISRA C:2023 analysis with `misch` for both atomicity modes against baselines in `analysis/baseline/` that record the existing findings, and checks formatting with the clang-format release pinned in `scripts/format.sh`.
- CI installs Meson, Ninja and gcovr with `uv` instead of `pip`, and the Meson files and tests carry SPDX licence headers.

- The README is a short landing page, with the integration guide, API reference and design record in `docs/`. `CONTRIBUTING.md` and `AGENTS.md` follow the same layout, and a `pool docs examples` test compiles every C example in the documentation.
- Added security policy, SPDX headers, cleaned ignore rules, README badge correction, and license copyright normalisation.

## [2.1.0] - 2026-05-29

### Added

- Added `pool_footprint()`, `pool_footprint_t`, payload sizing helpers, octet conversion helpers, optional RAM budget checks, and footprint self-consistency tests.

### Changed

- Adopted the shared primitive-family source style across public headers and implementation files.

## [2.0.0] - 2026-05-28

### Added

- Added platform abstraction for 8-bit and 16-bit minimum addressable units, configurable atomicity, version-header installation, pkg-config metadata, subproject dependency override, expanded CI, ThreadSanitizer tests, 16-bit MAU simulation, and MISRA deviation documentation.

### Changed

- Changed slot status storage to `POOL_ATOMIC(uint_least8_t)`, changed backing storage to `pool_byte_t[]`, pinned the build to C11, and added `atomicity_mode`.

### Fixed

- Fixed TI C2000 builds where `uint8_t` is unavailable because `CHAR_BIT == 16`.

## [1.1.3] - 2026-03-14

### Added

- Added Meson options for maximum pool slots and item size, plus hash and linear lookup strategy support.

### Fixed

- Improved ISR-safe release ordering, free-slot validation, hash cursor wrap behaviour, and slot-size alignment checks.

## [1.0.3] - 2026-02-28

### Fixed

- Enhanced ISR safety and compliance readiness by reordering release logic and adding validation tests.
