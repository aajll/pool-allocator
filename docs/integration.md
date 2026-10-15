# Integration guide

This guide shows how to add `pool-allocator` to a project, configure it, build it, and run its tests. The API is described in [api.md](api.md), and the slot model and its guarantees in [design.md](design.md).

## Requirements

- A C11 compiler with `_Static_assert` and `_Alignof`, and a target where `CHAR_BIT` is 8 or 16.
- `<stdint.h>`, `<stddef.h>`, `<limits.h>` and `<string.h>`. `<stdatomic.h>` is needed only for the C11 atomicity mode; a toolchain without it uses the volatile mode.
- For the tests: Meson, Ninja, Python 3, and POSIX threads.

## Copy-in

Copy four files into your source tree:

```
include/pool.h
include/pool_conf.h
include/pool_platform.h
src/pool.c
```

Keep the three headers in the same directory, because `pool.h` includes the other two. Then include the public header:

```c
#include "pool.h"
```

`pool.h` defines the version macros described in [api.md](api.md#version).

## Meson subproject

Put the repository in `subprojects/pool_allocator`, as a checkout or through a wrap file, then declare the dependency:

```meson
pool_dep = dependency('pool', fallback: ['pool_allocator', 'pool_dep'])
```

The build calls `meson.override_dependency('pool', ...)`, so other subprojects in the same build find `pool` by name. `pool_dep` links the static library and adds the include directory.

| Option | Default | Effect |
| --- | --- | --- |
| `pool_max_slots` | `16` | Sets `POOL_MAX_SLOTS` for the library and its consumers. |
| `pool_item_size` | `64` | Sets `POOL_ITEM_SIZE` for the library and its consumers. |
| `lookup_strategy` | `linear` | `linear` or `hash`; sets `POOL_LOOKUP_STRATEGY`. |
| `atomicity_mode` | `auto` | `auto`, `c11` or `volatile`. `auto` leaves the choice to `pool_conf.h`; the others define `POOL_USE_C11_ATOMIC` or `POOL_USE_VOLATILE_ATOMIC` for the library and its consumers. |
| `build_tests` | `false` | Builds the unit, 16-bit storage and concurrency tests and the documentation example test. |
| `tsan` | `false` | Builds the concurrency test under ThreadSanitizer. |

`pool_dep` passes `POOL_MAX_SLOTS`, `POOL_ITEM_SIZE` and the atomicity define to every target that uses it, so the consumer's `struct pool_t` has the library's layout. `POOL_LOOKUP_STRATEGY` stays private to the library.

## Installed package

```sh
meson setup build --buildtype=release -Dbuild_tests=false --prefix=/usr/local
meson compile -C build
meson install -C build
cc app.c $(pkg-config --cflags --libs pool) -o app
```

The install contains the static archive `libpool.a`, the headers in a `pool/` subdirectory, and `pool.pc`. The `pkg-config` flags add the parent include directory, so include the namespaced path:

<!-- snippet: skip -->
```c
#include <pool/pool.h>
```

`pool.pc` records the slot count, slot size and atomicity define the library was built with, so `pkg-config --cflags pool` gives an application the same layout.

## Configuration

`pool_conf.h` holds the options, each behind an `#ifndef` guard, so a `-D` definition replaces the shipped default. They are described in [api.md](api.md#configuration-macros).

Most of them shape `struct pool_t`, which the caller allocates and the library accesses: `POOL_MAX_SLOTS`, `POOL_ITEM_SIZE`, the atomicity mode, `POOL_NO_MAX_ALIGN_T` and `POOL_SIMULATE_16BIT_MAU`. Each must have the same value in the library build and in every translation unit that includes `pool.h`. A caller compiled with 16 slots against a library built with 64 allocates a quarter of the storage the library writes to. Pass them as `-D` flags to the whole build rather than defining them in a source file before the include.

`POOL_LOOKUP_STRATEGY` is read only by `src/pool.c`, and `POOL_RAM_BUDGET_OCTETS` only adds a compile-time check, so a mismatch in either is harmless.

In a Meson build, set the subproject option, and `pool_dep` carries the value to the consumer:

```sh
meson setup build -Dpool_allocator:pool_max_slots=32
```

`POOL_NO_MAX_ALIGN_T` and `POOL_SIMULATE_16BIT_MAU` have no option, so define them in the top-level project's `c_args`, which apply to the consumer and to subprojects. Do not repeat an exported define there with a different value, because the two definitions conflict.

`pool_footprint()` reports the library's view of the layout, so a start-up check that its `instance_size_units` equals `sizeof(struct pool_t)` in the caller catches a mismatch that slipped through. [api.md](api.md#footprint) shows it.

## Build and test

```sh
# Library only
meson setup build --buildtype=release -Dbuild_tests=false
meson compile -C build

# All tests under ASan and UBSan
meson setup build --buildtype=debug -Dbuild_tests=true -Db_sanitize=address,undefined
meson compile -C build
meson test -C build --verbose

# The same tests in the volatile atomicity mode
meson setup build-volatile --buildtype=debug -Dbuild_tests=true -Datomicity_mode=volatile
meson compile -C build-volatile
meson test -C build-volatile --verbose
```

ThreadSanitizer, Valgrind, coverage, formatting and MISRA commands are in [CONTRIBUTING.md](../CONTRIBUTING.md).

## Cross builds

The consuming firmware owns its cross file, linker command file, and memory placement. No cross file ships with this project.

CI compiles the library with warnings as errors using two releases of a vendor C11 compiler for a 16-bit-MAU DSP target, under both of its object formats and both lookup strategies. This covers the source dialect and type assumptions only; linking and execution remain checks for the consumer.

On such a target `CHAR_BIT` is 16 and `uint8_t` does not exist. The pool stores its slots as `uint16_t` words, `POOL_ITEM_SIZE` counts words, and `sizeof` reports words, so a slot of 64 holds 128 octets. The compiler ships no `<stdatomic.h>`, so the volatile atomicity mode is selected without configuration. [design.md](design.md#storage-on-16-bit-mau-targets) describes the layout, and `POOL_SIMULATE_16BIT_MAU` lets the host tests run the same storage path.
