# Contributing to pool-allocator

`pool-allocator` is a small C11 static object pool. A slot handed out twice, or freed while still in use, corrupts whatever the caller stored in it, so read [design.md](docs/design.md) before you change acquisition, release or the concurrency contract.

## Getting started

Install a C11 compiler and install the Python tools with [uv](https://docs.astral.sh/uv/), as CI does. The MISRA analysis also needs a current `cppcheck`; CI installs it from Homebrew because the 2.13 release in Ubuntu 24.04 reports different findings. The same commands CI runs, locally:

```sh
# Python tools
uv tool install meson
uv tool install ninja
uv tool install gcovr
uv tool install misch

# Configure with tests and sanitisers (CI default)
meson setup build --buildtype=debug -Dbuild_tests=true -Db_sanitize=address,undefined
meson compile -C build
meson test -C build --verbose

# The other lookup strategy and atomicity mode
meson setup build-hash --buildtype=debug -Dbuild_tests=true -Db_sanitize=address,undefined -Dlookup_strategy=hash -Datomicity_mode=volatile
meson compile -C build-hash && meson test -C build-hash

# Release build (the configuration a downstream consumer uses)
meson setup build-rel --buildtype=release -Dbuild_tests=false
meson compile -C build-rel

# ThreadSanitizer on the concurrency test (C11 mode only)
meson setup build-tsan --buildtype=debug -Dbuild_tests=true -Dtsan=true -Datomicity_mode=c11
meson compile -C build-tsan && meson test -C build-tsan test_pool_concurrency

# Valgrind (no sanitisers; ASan and Valgrind are incompatible)
meson setup build-mem --buildtype=debug -Dbuild_tests=true
meson compile -C build-mem
meson test -C build-mem --wrapper='valgrind --error-exitcode=1 --leak-check=full --errors-for-leak-kinds=all'

# Coverage across both lookup strategies (CI gate is 100% line and branch)
for strategy in linear hash; do
  meson setup "build-cov-$strategy" --buildtype=debug -Dbuild_tests=true -Db_coverage=true -Dc_args=-fprofile-update=atomic -Dlookup_strategy=$strategy
  meson compile -C "build-cov-$strategy" && meson test -C "build-cov-$strategy"
done
gcovr --root . --filter 'src/' --filter 'include/' --fail-under-line 100 --fail-under-branch 100 --print-summary build-cov-linear build-cov-hash

# cppcheck (the CI static-analysis gate)
cppcheck --enable=warning,style,performance,portability --error-exitcode=1 --inline-suppr --std=c11 --suppress=missingIncludeSystem --suppress=unusedFunction -I include src/ include/

# MISRA C:2023, once per atomicity mode (reads build/compile_commands.json)
env -u MISRA_RULE_TEXTS misch run --baseline
env -u MISRA_RULE_TEXTS misch run --baseline --profile volatile
misch deviations

# Formatting (pinned clang-format through uvx)
scripts/format.sh --fix
```

## Source style

- Formatting is mandatory. Run `scripts/format.sh --fix` before submitting, or `scripts/format.sh` to check without writing, as CI does. The script pins the clang-format release and runs it through `uvx`, so an editor formatter gives the same result only when it runs the same release.
- 8-space indent, Linux brace style, 80-column limit. Match the existing conventions and do not reformat unrelated code.
- The Meson build system is the single source of truth. Update `meson.build` and `tests/meson.build` when you add or remove files. No CMake, no Make, and no other build systems.

## C language rules

- C11 only.
- Use fixed-width types from `<stdint.h>` and `<stdbool.h>`, and never plain `int` for a fixed-width quantity. Slot flags stay `uint_least8_t`, which exists where `CHAR_BIT` is 16.
- No heap allocation (`malloc`, `free`, VLAs), no recursion, no system calls, and no global state. Every loop is bounded by `POOL_MAX_SLOTS` or `POOL_ITEM_SIZE`.
- Validate every pointer argument at the public API boundary and return a status rather than fault.
- Every public identifier uses the `pool_` or `POOL_` prefix.
- New code builds and passes in both atomicity modes and both lookup strategies, and compiles where `CHAR_BIT` is 16.

## Pool rules

- Clear a slot's storage before its flag becomes free, and keep the clearing behind a `volatile` access so it cannot be removed as a dead store.
- Reject every flag value other than "allocated" on release and lookup, so a double release or an uninitialised pool cannot free a slot.
- Keep mutation to one writer. A change that needs two writers, or that lets a reader change the pool, changes the concurrency contract and needs an issue first.
- A change to `struct pool_t` changes the layout callers allocate. Keep it compatible within a release line, and keep every layout macro in [the integration guide](docs/integration.md#configuration) list.

## MISRA C:2023

The library is analysed with `misch` (cppcheck-backed MISRA C:2023 analysis), configured by `misra.toml`, over `include/` and `src/`, once with C11 atomics and once with the `volatile` profile. CI fails on any finding that is not in that mode's baseline in `analysis/baseline/`. Run it from a configured `build/` directory, with `MISRA_RULE_TEXTS` unset, as CI does. If your change introduces a finding:

1. Prefer restructuring the code so the rule is satisfied. Deviate only when compliance would make the code genuinely worse, and never restructure purely to hide a finding from the checker; an honest deviation beats evasion.
2. Suppress at the point of use with `/* cppcheck-suppress misra-c2012-<rule> ; @deviation <rationale> */`, or add a justified project-wide entry to `analysis/deviations/misra-deviations.txt` for a house-style rule.
3. Verify with `misch run --baseline` for both modes and `misch deviations`, which requires every suppression to carry a rationale, and justify the deviation in the pull request description.

Do not regenerate a baseline to admit a new finding. It changes only after a deliberate review, with `misch baseline`.

## Tests and coverage

- Add a test for every bug fix and every new feature.
- Unit tests live in `tests/test_pool.c`, which also runs as `test_pool_16bit_sim` against the 16-bit storage build. The one-writer, many-readers stress test is `tests/test_pool_concurrency.c`. Each new file is listed in `tests/meson.build`.
- Test each guard with the input that reaches it, including NULL arguments, IDs at `POOL_MAX_SLOTS`, free slots and a full pool.
- CI enforces 100 percent line and branch coverage of `src/` and `include/` across both lookup strategies. Every new branch needs a test that fails when the branch is broken.
- The `pool docs examples` test compiles every C code block in `README.md`, `CONTRIBUTING.md`, and `docs/` against the public headers, through `tests/docs/check_snippets.py`. A block compiles at file scope by default. On the line directly above the fence, put `<!-- snippet: body -->` for a block of statements, or `<!-- snippet: skip -->` for pseudo-code or a copy of a header definition.

## Documentation

| File | Purpose |
| --- | --- |
| `README.md` | Landing page: quick start, description, validated toolchains, and links. Keep it short. |
| `docs/integration.md` | Requirements, installation, configuration, building, and cross builds. |
| `docs/api.md` | Reference for initialisation, acquisition and release, pointer lookup, the footprint, the configuration macros, and recipes. |
| `docs/design.md` | The slot model, lookup strategies, release and reuse, the concurrency contract, 16-bit-MAU storage, failure behaviour, MISRA position, and verification. |

Update `docs/api.md` and the header Doxygen together when an interface changes, and check every value an example states against a build. Update `docs/design.md` when acquisition, release, the concurrency contract or the verification changes. Keep it standalone: describe how the library works, not how it was built or which other projects use it. Write prose in British English, one line per paragraph.

## API stability

- The function signatures in `pool.h` and the configuration macros in `pool_conf.h` are stable within a major release line.
- The values of `pool_status_t` are part of the contract, because callers store and transmit them.
- Which slot an acquisition returns under each strategy is observable behaviour. Changing it, a signature, or a status value is a breaking change and needs a major release.
- The members of `struct pool_t` are private, but its size is not: callers allocate it, so a layout change needs a `CHANGELOG.md` entry.

## Commits

Use Conventional Commits:

- `feat: ...` new feature
- `fix: ...` bug fix
- `docs: ...` documentation only
- `test: ...` test-only changes
- `chore: ...` build, CI, release work
- `refactor: ...` a change that neither fixes a bug nor adds a feature

Keep the subject under about 70 characters. Use the body to explain why the change is needed.

## Pull requests

- Open an issue first for non-trivial changes, and always for a change to acquisition order, release or the concurrency contract, so the design can be agreed before implementation.
- Keep pull requests focused: one feature or one fix.
- All CI checks must pass.
- Add a `CHANGELOG.md` entry under `## [Unreleased]` for a user-visible change.
- Flag any new MISRA deviation, and any change to the layout of `struct pool_t`, in the description.

## When in doubt

Open an issue. The library is small, but every firmware that stores objects in it trusts that a slot belongs to one owner at a time.
