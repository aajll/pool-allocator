# Design

`pool-allocator` hands out fixed-size blocks of memory from storage the caller provides, with no heap and a known worst-case time. This document describes the slot model, the two lookup strategies, release and reuse, the concurrency contract, storage on 16-bit-MAU targets, and how the library is verified. The API is in [api.md](api.md).

## Goals

- All memory is reserved at build time, in one structure the caller places, and its size is known at compile time.
- Every operation finishes in time bounded by `POOL_MAX_SLOTS`, with no recursion and no data-dependent loop bound beyond it.
- Misuse that a pool can detect, such as a double release, a free or out-of-range ID, or a NULL argument, returns a status and leaves the pool unchanged.
- Portable C11, including targets where `CHAR_BIT` is 16 and the toolchain has no `<stdatomic.h>`.

Variable-size allocation, growing a pool, per-slot ownership tracking and payload synchronisation are out of scope.

## The pool

`struct pool_t` holds three members:

- `storage`, a union of `POOL_MAX_SLOTS * POOL_ITEM_SIZE` storage units with one `pool_align_t`, which aligns the block as strictly as that type.
- `slot_status`, one flag per slot, either free or allocated.
- `next_index`, the cursor the hash strategy starts from.

Slot `i` occupies units `i * POOL_ITEM_SIZE` to `(i + 1) * POOL_ITEM_SIZE - 1` of the storage. Because `POOL_ITEM_SIZE` must be a multiple of `_Alignof(pool_align_t)`, every slot starts on that alignment. The ID of a slot is its index, a `uint16_t`, which is why `POOL_MAX_SLOTS` is limited to `UINT16_MAX`.

The flags are the only record of which slots are allocated. The pool does not record who holds a slot, and its payload is opaque to it.

## Lookup strategies

Both strategies visit each slot at most once per acquisition and stop at the first free one, so an acquisition costs at most `POOL_MAX_SLOTS` flag reads. They differ only in where the search starts.

The linear strategy starts at slot 0. The same sequence of acquisitions and releases always produces the same IDs, and the lowest free slot is always taken, which keeps allocated slots packed at the start of the storage.

The hash strategy starts at `next_index` and wraps around. Every acquisition sets `next_index` to the slot after the one it returned, so allocation rotates through the array. This spreads writes evenly when the storage is backed by memory with limited write endurance, and a freed slot is not reused until the cursor comes round to it again, which delays the reuse of a stale ID. A cursor at or beyond `POOL_MAX_SLOTS`, which only a damaged structure can hold, restarts the search at 0.

The cursor is updated under both strategies; only the hash strategy reads it.

## Release and reuse

`pool_release` checks that the slot is allocated, writes zero to every unit of its storage, and then marks it free. The zeroing goes through a `volatile` pointer, so the compiler cannot drop it as a dead store. Clearing before the flag changes means that a slot is never free while it still holds the previous owner's data, and a later acquisition always starts from zero.

A release of a free slot returns `POOL_ERR_INVALID_ID`, so a double release is detected and harmless. A release also rejects any flag value other than "allocated", so a pool that was never initialised, whose flags hold arbitrary bytes, rejects releases rather than freeing slots it never handed out.

An ID carries no generation count. Once a slot is released and acquired again, an old copy of its ID names the new allocation, and the pool cannot tell the two apart. Clear or invalidate IDs when the slot they name is released. The hash strategy makes immediate reuse less likely, but does not prevent it.

## Concurrency

The contract is one writer and many readers per pool:

- One context, which may be a task, a control loop or an interrupt handler, calls `pool_init`, `pool_acquire` and `pool_release`.
- Any number of other contexts call `pool_get_pointer` and `pool_get_pointer_checked` at the same time.

An acquisition reads flags until it finds a free slot and then writes that flag. Two writers could both read the same flag as free and both claim the slot, so writers that share a pool need the caller's own exclusion, such as a mutex or a masked interrupt.

A reader reads one flag and returns a pointer. The flags are `_Atomic uint_least8_t` in the C11 mode, so every flag access is sequentially consistent and free of data races, and ThreadSanitizer reports none in the concurrency test. In the volatile mode the flags are `volatile uint_least8_t`. That is enough on a single-core target, where a naturally aligned store of a byte or word cannot be split and the compiler cannot cache the flag in a register, but not between cores.

The library synchronises the flags only, not the payload. A reader that sees a slot allocated can see it before the writer has filled it, and a pointer it receives can be released and zeroed while it is still in use. Readers that touch the payload need a protocol of their own, such as publishing a slot's ID only after it is filled and releasing it only after every reader has finished.

## Storage on 16-bit-MAU targets

On a target where `CHAR_BIT` is 16, `uint8_t` does not exist and `sizeof` counts 16-bit words. `pool_platform.h` detects this from `CHAR_BIT` or `UCHAR_MAX` and makes the storage unit, `pool_byte_t`, a `uint16_t` instead of a `uint8_t`. Every size in the configuration and the footprint is then in words: `POOL_ITEM_SIZE` of 64 is 128 octets, and `instance_size_octets` doubles `instance_size_units`. The flags stay `uint_least8_t`, which every conforming target provides, and occupy a word each.

Toolchains for such targets typically ship no `<stdatomic.h>`, so the atomicity mode defaults to volatile whenever `CHAR_BIT` is above 8. In C11 mode `pool_align_t` is `max_align_t`; `POOL_NO_MAX_ALIGN_T` replaces it with `uint64_t` for a toolchain whose `<stddef.h>` lacks it.

`POOL_SIMULATE_16BIT_MAU` selects the `uint16_t` storage unit on a byte-addressed host, so the host tests run the same indexing and clearing code. `sizeof` still counts bytes there, so the footprint's unit-based fields mix the two units in that build, and only the test suite uses it.

## Configuration

Every option is a compile-time constant. The checks in `pool.h` reject a zero size, more than `UINT16_MAX` slots, a payload larger than `SIZE_MAX`, an unknown strategy, and a slot size that breaks the alignment, so an invalid configuration never builds. `POOL_RAM_BUDGET_OCTETS` adds an upper bound on the instance size in octets, which is comparable across targets.

The slot count, slot size, atomicity mode and alignment type shape `struct pool_t`. The caller allocates the structure and the library indexes into it, so both must be compiled with the same values. [integration.md](integration.md#configuration) shows how to keep them aligned in each kind of build, and [api.md](api.md#checking-the-layout-at-start-up) a start-up check.

## Failure behaviour

| Condition | Result |
| --- | --- |
| NULL pool to any function | `POOL_ERR_NULL_PTR`, or NULL from `pool_get_pointer`; nothing is written. |
| NULL `p_id` or `p_ptr` | `POOL_ERR_NULL_PTR`; nothing is written. |
| Every slot allocated | `POOL_ERR_FULL` from `pool_acquire`; `*p_id` is unchanged. |
| ID at or above `POOL_MAX_SLOTS` | `POOL_ERR_INVALID_ID`, or NULL from `pool_get_pointer`. |
| Release or lookup of a free slot | `POOL_ERR_INVALID_ID`, or NULL from `pool_get_pointer`. |
| Stale ID whose slot was reacquired | Accepted; it names the new allocation. |
| Pool used before `pool_init` | Undefined flag contents; a release of a slot whose flag is not exactly "allocated" is rejected. |
| Two concurrent writers | Undefined; the caller must serialise them. |
| Library and caller built with different layouts | Undefined; the library reads and writes outside the caller's structure. |
| Invalid configuration | Compile error from a `_Static_assert`. |

`pool_footprint` reports `item_size` as a `uint16_t`, so a slot size above 65535 units is reduced modulo 65536 in that field; the pool itself uses the full value.

## MISRA C:2023

The library is analysed with [`misch`](https://pypi.org/project/misch/), a cppcheck-based MISRA C:2023 checker, over `include/` and `src/`, once with C11 atomics and once with `POOL_USE_VOLATILE_ATOMIC`. Each run has its own baseline of accepted findings in `analysis/baseline/`, and CI fails on any finding outside it. Both hold the same 33 findings:

| Rule | Findings | Cause |
| --- | --- | --- |
| 15.5 | 13 | Early returns, which reject invalid arguments before any work. |
| 10.4 | 7 | Unsigned operands compared with `POOL_MAX_SLOTS` and `POOL_ITEM_SIZE`, which the Meson build defines without a `U` suffix, and flags compared with the enumerators of the internal slot status type. |
| 8.7 | 5 | Public functions with no caller outside `src/pool.c` in the analysed code. |
| 19.2 | 2 | The union that aligns the storage block. |
| 20.7 | 2 | The parameter of `POOL_ATOMIC(type)`, which expands after a qualifier and cannot be parenthesised. |
| 1.4 | 1 | `_Alignof` in the alignment check. |
| 12.1 | 1 | The `||` of two comparisons in the addressable-unit check, without parentheses. |
| 2.3, 2.4 | 1 each | The internal slot status type and its tag are never used as a type, because the flags are stored as `uint_least8_t`. |

The baseline records the findings as they stand. Formal compliance needs a certified analyser and a deviation record from the consuming project.

## Verification

The tests run on the host:

| Test | What it checks |
| --- | --- |
| `test_pool` | Initialisation, acquisition to full, release, double release, every NULL and out-of-range argument, both pointer accessors, the hash cursor recovering from an out-of-range value, rejection of uninitialised flags, slot alignment, clearing on release, that slots do not overlap, and the footprint fields. |
| `test_pool_16bit_sim` | The same tests against a library built with `POOL_SIMULATE_16BIT_MAU`. |
| `test_pool_concurrency` | One writer cycling acquisitions and releases against four readers calling both accessors, checking that every status matches its pointer. |
| `pool docs examples` | Every C block in the README, `CONTRIBUTING.md` and `docs/` compiles against the public headers. |

CI runs every test for both lookup strategies and both atomicity modes under AddressSanitizer and UndefinedBehaviorSanitizer, runs the concurrency test under ThreadSanitizer in the C11 mode, and runs the suite under Valgrind in both modes. It gates line and branch coverage of `src/` and `include/` at 100 percent across both strategies, builds the library in release mode for each combination, runs cppcheck, checks formatting, and runs the MISRA analysis for each mode. It also compiles the library with warnings as errors under TI C2000 CGT 22.6.2 and 25.11.1, in both the COFF and EABI formats and for both lookup strategies.
