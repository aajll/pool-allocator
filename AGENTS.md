# AGENTS.md

---

## 1) Project-specific instructions

**Project:** `pool-allocator`
**Primary goal:** A C11 static object pool: `POOL_MAX_SLOTS` slots of `POOL_ITEM_SIZE` storage units in one caller-owned structure, acquired and released by 16-bit ID in time bounded by the slot count, with one writer and any number of concurrent readers. No allocation, no system calls, no global state.

### 1.1 Essential commands

#### Configure and build (library only)

```sh
meson setup build --wipe --buildtype=release -Dbuild_tests=false
meson compile -C build
```

#### Configure, build, and run unit tests

```sh
meson setup build --wipe --buildtype=debug -Dbuild_tests=true -Db_sanitize=address,undefined
meson compile -C build
meson test -C build --verbose
```

`test_pool_16bit_sim` in this suite runs the unit tests against the 16-bit storage layout.

#### Volatile atomicity mode and the hash strategy

```sh
meson setup build-volatile --wipe --buildtype=debug -Dbuild_tests=true -Datomicity_mode=volatile -Dlookup_strategy=hash
meson compile -C build-volatile
meson test -C build-volatile --verbose
```

#### ThreadSanitizer (concurrency test, C11 mode only)

```sh
meson setup build-tsan --wipe --buildtype=debug -Dbuild_tests=true -Dtsan=true -Datomicity_mode=c11
meson compile -C build-tsan
meson test -C build-tsan test_pool_concurrency --verbose
```

#### MISRA C:2023, once per atomicity mode (reads `build/compile_commands.json`)

```sh
env -u MISRA_RULE_TEXTS misch run --baseline
env -u MISRA_RULE_TEXTS misch run --baseline --profile volatile
misch deviations
```

#### Formatting (pinned clang-format through `uvx`)

```sh
scripts/format.sh          # check, as CI does
scripts/format.sh --fix    # rewrite in place
```

Valgrind, coverage and cppcheck commands are in `CONTRIBUTING.md`.

### 1.2 Notes

- `include/pool.h` defines the version macros, and `meson setup` fails when they differ from `project(version:)`, so a release updates both. `include/pool_version.h` is a deprecated header that includes `pool.h`.
- The build produces only a static library, installed as `libpool.a` with headers under `pool/` and `pool.pc`. Meson options: `pool_max_slots`, `pool_item_size`, `lookup_strategy`, `atomicity_mode`, `build_tests`, `tsan`.
- `POOL_MAX_SLOTS`, `POOL_ITEM_SIZE`, the atomicity mode, `POOL_NO_MAX_ALIGN_T` and `POOL_SIMULATE_16BIT_MAU` shape `struct pool_t`. The Meson options set them for the library build only, so callers need the same `-D` definitions.

---

## 2) Non-negotiable design invariants

- **Bounded.** Every loop is bounded by `POOL_MAX_SLOTS` or `POOL_ITEM_SIZE`. No recursion.
- **Clear on release.** A slot's storage is zeroed through a `volatile` access before its flag becomes free.
- **Reject, never corrupt.** Release and lookup accept only a flag equal to "allocated", so a double release, a free ID or an uninitialised pool returns `POOL_ERR_INVALID_ID`. A full pool returns `POOL_ERR_FULL` without writing `*p_id`.
- **One writer.** Only `pool_init`, `pool_acquire` and `pool_release` change the pool, and they are not safe against each other. Readers never write. The library synchronises the flags, not the payload.
- **Matching layout.** The library and every caller must be built with the same layout macros, because the caller allocates the structure the library indexes.
- No heap, no system calls, no blocking, no global state.

---

## 3) CI / source of truth

- CI definitions live in `.github/workflows/ci.yml`.
- Prefer running the same commands locally as CI runs (see §1.1 above).
- Install Python tools (Meson, Ninja, gcovr, misch) with `uv tool install`, never `pip`; CI does the same through `astral-sh/setup-uv`. CI installs `cppcheck` for the MISRA job from Homebrew because Ubuntu's 2.13 reports different findings.
- `misch run --baseline` must pass against `analysis/baseline/` for both profiles. Never regenerate a baseline to admit a new finding. Run it with `MISRA_RULE_TEXTS` unset, as CI does.
- The CI jobs are: tests under ASan/UBSan for each lookup strategy and atomicity mode, a release build matrix, cppcheck, ThreadSanitizer, Valgrind in both modes, the 16-bit storage simulation, a 100 percent line and branch coverage gate across both strategies, a TI C2000 library build (CGT 22.6.2 and 25.11.1, COFF and EABI, both strategies, warnings as errors), `clang-format`, and `misch` for both modes.

---

## 4) Docs / commit conventions

### Documentation layout

- `README.md` is a short landing page (under 100 lines). Put detail in `docs/`, never in the README; link to the owning file instead.
- `docs/integration.md` owns requirements, install methods, configuration, building, and cross builds.
- `docs/api.md` owns the API reference, the configuration macros and recipes. Change it together with the headers; keep signatures in fenced `c` blocks and describe them in prose, not function tables. Check every value an example states against a build.
- `docs/design.md` is the single authoritative design record: the slot model, lookup strategies, release and reuse, the concurrency contract, 16-bit-MAU storage, failure behaviour, MISRA position, and verification.
- `CONTRIBUTING.md` documents the contributor workflow; `CHANGELOG.md` holds release history (Keep a Changelog); `.github/SECURITY.md` covers vulnerability reporting.
- Code blocks in `README.md`, `CONTRIBUTING.md`, and `docs/` are compiled by the `pool docs examples` test (`tests/docs/check_snippets.py`). Each document is compiled as one source file: a `c` fence is file-level code, `<!-- snippet: body -->` wraps statements, and `<!-- snippet: skip -->` excludes pseudo-code. Do not use `skip` to silence a broken example.
- Prose uses British spelling. Do not hard-wrap Markdown: write one line per paragraph. The 80-column limit applies to C source, not Markdown.

### Commits

- Use **Conventional Commits** format when asked to commit.
- Keep commits focused; explain *why* in the message body.

---

## 5) C style expectations

### Build & configuration

- Use the Meson build system. Do not introduce CMake, Make, or other systems.
- Update `meson.build` and `tests/meson.build` when adding or removing files.
- `pool_conf.h` and `pool_platform.h` are public and installed beside `pool.h`. Every option stays behind an `#ifndef` guard and is checked by a `_Static_assert` in `pool.h`.

### Formatting

- `.clang-format` is present and **mandatory**. Run `scripts/format.sh --fix` before committing. It runs the clang-format release pinned in the script through `uvx`. Do not use a system `clang-format`: another release can format the same code differently.
- Do not reformat unrelated code.
- Key settings: 8-space indent, `BreakBeforeBraces: Linux`, column limit 80.

### Style & correctness

- Match conventions in the existing files (indentation, braces, naming).
- Every public identifier uses the `pool_` or `POOL_` prefix. Keep public headers minimal; do not expose internal helpers.
- Validate pointer arguments at every public API boundary; no `errno`.
- Use fixed-width types from `<stdint.h>`; never plain `int` for a fixed-width quantity. Storage that touches slot memory uses `pool_byte_t`, and flags use `POOL_ATOMIC(uint_least8_t)`, so the code builds where `CHAR_BIT` is 16 and `uint8_t` does not exist.
- New code builds and passes in both atomicity modes and both lookup strategies.
- `pool_status_t` values, and which slot each strategy returns for a given history, are part of the API. Changing either is a breaking change. A change to the layout of `struct pool_t` needs a `CHANGELOG.md` entry.
- Public symbols carry Doxygen with the full contract: preconditions and return semantics.

### MISRA deviations

- Prefer compliant code; never restructure only to hide a finding. Deviate at the point of use with `/* cppcheck-suppress misra-c2012-<rule> ; @deviation <rationale> */`, or project-wide in `analysis/deviations/misra-deviations.txt` for a house-style rule.

### Comment placement (Doxygen)

- Inline trailing annotations (`/**< ... */`) on `enum`/`struct` members are allowed only when the resulting line fits the 80-column limit.
- If any member's annotation would overrun, move **all** of that aggregate's member docs into a single structured Doxygen block above the type, as an `@details` list of `- ::SYMBOL  description` entries.
- Never mix inline and block forms within one aggregate, and never leave a trailing comment that clang-format would wrap onto a second line.
- After editing, run `scripts/format.sh` and an 80-column scan.

### Testing

- Run `meson test -C build` after every change.
- Unit tests live in `tests/test_pool.c` with its `RUN_TEST` macro, and the one-writer, many-readers stress test in `tests/test_pool_concurrency.c`. Do not add Unity or another external framework.
- Test each guard with the input that reaches it: NULL arguments, IDs at `POOL_MAX_SLOTS`, free slots, a full pool.
- Add a test case for each bug fix and each new feature.
- Keep `src/` and `include/` at the coverage gate (100 percent line and branch, across both strategies).
