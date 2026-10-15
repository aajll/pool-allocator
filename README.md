# pool-allocator

[![CI](https://github.com/aajll/pool-allocator/actions/workflows/ci.yml/badge.svg)](https://github.com/aajll/pool-allocator/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**A C11 static object pool: a fixed number of fixed-size slots in caller-owned storage, handed out and returned by ID with bounded, deterministic time and no heap.**

## Quick start

Copy `include/pool.h`, `include/pool_conf.h`, `include/pool_platform.h` and `src/pool.c` into your project. For a Meson subproject or an installed `pkg-config` package, see the [integration guide](docs/integration.md).

```c
#include <stdio.h>

#include "pool.h"

struct reading {
        uint32_t sensor;
        float value;
};

_Static_assert(sizeof(struct reading) <= POOL_ITEM_SIZE,
               "a reading must fit in one slot");

static struct pool_t pool;

int
main(void)
{
        pool_id_t a;
        pool_id_t b;

        if (pool_init(&pool) != POOL_OK) {
                return 1;
        }
        if ((pool_acquire(&pool, &a) != POOL_OK)
            || (pool_acquire(&pool, &b) != POOL_OK)) {
                return 1;
        }

        struct reading *r = pool_get_pointer(&pool, b);
        r->sensor = 7u;
        r->value = 21.5f;

        /* Prints "slots 0 and 1, sensor 7 reads 21.5". */
        printf("slots %u and %u, sensor %u reads %.1f\n", (unsigned)a,
               (unsigned)b, (unsigned)r->sensor, (double)r->value);

        /* A second release of the same ID is rejected: prints "-3". */
        (void)pool_release(&pool, a);
        printf("%d\n", (int)pool_release(&pool, a));

        /* The linear strategy reuses the lowest free slot: prints "0". */
        (void)pool_acquire(&pool, &a);
        printf("%u\n", (unsigned)a);
        return 0;
}
```

## Description

The pool is one structure the caller owns: `POOL_MAX_SLOTS` slots of `POOL_ITEM_SIZE` storage units each, a status flag per slot, and a cursor. Both sizes are compile-time constants, so the RAM cost is known at link time and can be checked against a budget at compile time.

- Acquire, release and pointer lookup by a 16-bit slot ID, with every result reported as a status code
- Linear or round-robin slot search, chosen at build time, each bounded by `POOL_MAX_SLOTS` iterations
- Released slots are zeroed, and a double release or a stale ID is rejected rather than corrupting the pool
- One writer and any number of concurrent readers per pool, with `_Atomic` status flags where the toolchain has `<stdatomic.h>` and `volatile` ones where it does not
- Tests under ASan, UBSan, TSan and Valgrind with 100% line and branch coverage of both strategies, and MISRA C:2023 analysis with [`misch`](https://pypi.org/project/misch/)

## Validated toolchains

| Toolchain               | Target                        | Validation                                                   |
| ----------------------- | ----------------------------- | ------------------------------------------------------------ |
| GCC                     | x86_64 Linux                  | All tests, both strategies and both atomicity modes          |
| GCC                     | x86_64 Linux, 16-bit storage  | All tests with the 16-bit-MAU storage layout simulated       |
| DSP vendor C11 compiler | 16-bit MAU (`CHAR_BIT == 16`) | Library compile, both strategies, two object formats         |

Target linking and execution belong to the consuming firmware. See [cross builds](docs/integration.md#cross-builds).

## Documentation

- [Integration guide](docs/integration.md) — copy-in, Meson subproject, installed package, configuration, building, and cross builds
- [API reference](docs/api.md) — initialisation, acquire and release, pointer lookup, the footprint, configuration macros, and recipes
- [Design](docs/design.md) — the slot model, lookup strategies, the concurrency contract, storage on 16-bit-MAU targets, failure behaviour, and verification
- [Changelog](CHANGELOG.md) — release history

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) before you open a pull request. Report security issues as described in [SECURITY.md](.github/SECURITY.md).

## License

MIT. See [LICENSE](LICENSE).
