# API reference

`pool.h` declares the whole interface and includes `pool_conf.h` and `pool_platform.h`. The examples in this document compile together as one file after `#include "pool.h"`.

## Conventions

- The pool is a `struct pool_t` that the caller owns. It may be static, on the stack, or embedded in another structure. No function allocates or blocks.
- Every function except `pool_footprint` takes a `pool_handle_t`, which is a `struct pool_t *`. Treat the members as private: their layout follows the configuration and may change between releases.
- Every pointer argument is checked. A NULL argument never dereferences.
- One context may call `pool_init`, `pool_acquire` and `pool_release` on a pool; any number of others may call the two pointer functions at the same time. [design.md](design.md#concurrency) gives the full contract.

## Status codes

<!-- snippet: skip -->
```c
typedef enum {
        POOL_OK = 0,
        POOL_ERR_NULL_PTR = -1,
        POOL_ERR_FULL = -2,
        POOL_ERR_INVALID_ID = -3
} pool_status_t;
```

`POOL_ERR_NULL_PTR` means a required pointer was NULL, `POOL_ERR_FULL` that no slot is free, and `POOL_ERR_INVALID_ID` that an ID is out of range or names a slot that is not allocated. The values are part of the API.

## Initialisation

```c
pool_status_t pool_init(pool_handle_t p_pool);
```

`pool_init` zeroes the storage, marks every slot free and resets the search cursor. It returns `POOL_ERR_NULL_PTR` for a NULL pool and `POOL_OK` otherwise. Call it before any other function on the pool. Calling it again frees every slot at once, and IDs held from before then refer to free slots.

## Acquire and release

```c
pool_status_t pool_acquire(pool_handle_t p_pool, pool_id_t *const p_id);
pool_status_t pool_release(pool_handle_t p_pool, const pool_id_t id);
```

`pool_acquire` finds a free slot, marks it allocated and writes its ID, a `uint16_t` from 0 to `POOL_MAX_SLOTS - 1`, to `*p_id`. It returns `POOL_ERR_NULL_PTR` if either pointer is NULL and `POOL_ERR_FULL` if every slot is allocated; in both cases `*p_id` is not written. The slot's storage is all zero, because `pool_init` and `pool_release` both clear it.

Which slot is chosen depends on `POOL_LOOKUP_STRATEGY`. The linear strategy always takes the lowest free index. The hash strategy starts searching just after the slot it handed out last and wraps around, so successive acquisitions move through the array even when earlier slots have been freed.

`pool_release` returns a slot. It returns `POOL_ERR_NULL_PTR` for a NULL pool, and `POOL_ERR_INVALID_ID` if `id` is not below `POOL_MAX_SLOTS` or the slot is not allocated, which includes a second release of the same ID. On success it zeroes the slot's storage before marking it free.

<!-- snippet: body -->
```c
static struct pool_t frames;
pool_id_t id;

(void)pool_init(&frames);
if (pool_acquire(&frames, &id) == POOL_OK) {
        /* use the slot */
        (void)pool_release(&frames, id);
}
```

## Pointer lookup

```c
void *pool_get_pointer(pool_handle_t p_pool, const pool_id_t id);
pool_status_t pool_get_pointer_checked(pool_handle_t p_pool, const pool_id_t id,
                                       void **const p_ptr);
```

`pool_get_pointer` returns the start of an allocated slot's storage, or NULL if the pool is NULL, the ID is out of range, or the slot is free. The pointer is aligned to `_Alignof(pool_align_t)`, which suits any object type when `pool_align_t` is `max_align_t`, and `POOL_ITEM_SIZE` storage units from it belong to the slot.

`pool_get_pointer_checked` performs the same checks and reports which one failed. It returns `POOL_ERR_NULL_PTR` if the pool or `p_ptr` is NULL, `POOL_ERR_INVALID_ID` for a bad or free ID, and `POOL_OK` otherwise. Whenever the pool and `p_ptr` are both non-NULL it writes `*p_ptr`, with NULL on failure.

Neither function changes the pool. A pointer stays valid until the slot is released or the pool is initialised again; after that it points into storage the pool may hand to another caller.

## Footprint

<!-- snippet: skip -->
```c
typedef struct {
        size_t instance_size_units;
        size_t instance_size_octets;
        size_t payload_units;
        size_t overhead_units;
        pool_id_t capacity;
        size_t item_size;
        uint_least8_t addr_unit_bits;
} pool_footprint_t;
```

```c
pool_footprint_t pool_footprint(void);
```

`pool_footprint` describes the configured pool type as the library was built; it reads no pool and is safe from any context. Sizes in `*_units` are in the units `sizeof` uses on the target, and `instance_size_octets` converts the instance size to 8-bit octets for comparison across targets.

- `instance_size_units` is `sizeof(struct pool_t)`, including padding.
- `payload_units` is `POOL_MAX_SLOTS * POOL_ITEM_SIZE`, also available as the constant `POOL_PAYLOAD_UNITS`.
- `overhead_units` is the difference: status flags, cursor and padding.
- `capacity` is `POOL_MAX_SLOTS`, `item_size` is `POOL_ITEM_SIZE`, and `addr_unit_bits` is 8 or 16.

With the default configuration on x86-64 the instance is 1056 bytes: 1024 of payload and 32 of overhead, because the structure is padded to the 16-byte alignment of `max_align_t`.

## Configuration macros

Each macro lives in `pool_conf.h` or `pool_platform.h` behind an `#ifndef` guard and is overridden with a `-D` flag. The ones that change `struct pool_t` must match between the library and its callers; [integration.md](integration.md#configuration) explains why and how.

| Macro | Default | Effect |
| --- | --- | --- |
| `POOL_MAX_SLOTS` | `16U` | Number of slots, from 1 to `UINT16_MAX`. Layout. |
| `POOL_ITEM_SIZE` | `64U` | Size of each slot in storage units: bytes, or 16-bit words where `CHAR_BIT` is 16. A positive multiple of `_Alignof(pool_align_t)`. Layout. |
| `POOL_LOOKUP_STRATEGY` | `POOL_LOOKUP_LINEAR` | `POOL_LOOKUP_LINEAR` or `POOL_LOOKUP_HASH`. |
| `POOL_USE_C11_ATOMIC`, `POOL_USE_VOLATILE_ATOMIC` | neither | Force the atomicity mode. Without either, the C11 mode is used unless `__STDC_NO_ATOMICS__` is defined or `CHAR_BIT` exceeds 8. Layout. |
| `POOL_NO_MAX_ALIGN_T` | undefined | Use `uint64_t` instead of `max_align_t` as `pool_align_t`. Layout. |
| `POOL_RAM_BUDGET_OCTETS` | undefined | Fails the build when `sizeof(struct pool_t)` in octets exceeds the value. |
| `POOL_SIMULATE_16BIT_MAU` | undefined | Uses 16-bit storage units on a byte-addressed host, for tests. Layout. |

The configuration is checked when `pool.h` is compiled. A zero size, a slot count above `UINT16_MAX`, a payload that overflows `size_t`, an unknown strategy, and a slot size that is not a multiple of the alignment each stop the build with a `_Static_assert`. On x86-64 `pool_align_t` is `max_align_t`, aligned to 16 bytes, so `POOL_ITEM_SIZE` must be a multiple of 16; with `POOL_NO_MAX_ALIGN_T` it must be a multiple of 8.

## Recipes

### A typed pool

Wrap the pool in functions that know the stored type, and check at compile time that it fits:

```c
struct can_frame {
        uint32_t can_id;
        uint8_t len;
        uint8_t data[8];
};

_Static_assert(sizeof(struct can_frame) <= POOL_ITEM_SIZE,
               "struct can_frame does not fit in a pool slot");

static struct pool_t frame_pool;

struct can_frame *
frame_alloc(pool_id_t *id)
{
        if (pool_acquire(&frame_pool, id) != POOL_OK) {
                return NULL;
        }
        return pool_get_pointer(&frame_pool, *id);
}
```

The slot pointer is aligned to `max_align_t` on a hosted target, so the conversion from `void *` is valid. Pass the ID, not the pointer, to other contexts: a stale ID is rejected by the pool, and a stale pointer is not.

### Checking the layout at start-up

A configuration mismatch between the library and its caller cannot be seen at compile time, because the library is already built. Compare the two views once:

```c
bool
pool_layout_matches(void)
{
        const pool_footprint_t fp = pool_footprint();

        return (fp.instance_size_units == sizeof(struct pool_t))
               && (fp.capacity == (pool_id_t)POOL_MAX_SLOTS);
}
```

### A RAM budget

Define `POOL_RAM_BUDGET_OCTETS` for the build to turn an oversized pool into a compile error:

```sh
cc -DPOOL_MAX_SLOTS=64U -DPOOL_RAM_BUDGET_OCTETS=4096 -Iinclude -c src/pool.c
```

On x86-64, 64 slots of 64 bytes take 4176 octets, so this build stops with "pool instance exceeds POOL_RAM_BUDGET_OCTETS". The check runs in every translation unit that includes `pool.h`.

## Version

`pool.h` defines the release it belongs to, so a copied-in header still identifies itself:

<!-- snippet: skip -->
```c
#define POOL_VERSION_MAJOR 2
#define POOL_VERSION_MINOR 1
#define POOL_VERSION_PATCH 0
#define POOL_VERSION "2.1.0"
```

`POOL_VERSION_NUMBER` is `MAJOR * 10000 + MINOR * 100 + PATCH` as a `long`, so it holds any version where `int` is 16 bits. Compare it in the preprocessor to require a minimum release:

```c
#if POOL_VERSION_NUMBER < 20100L
#error "pool 2.1.0 or later is required"
#endif
```

`POOL_VERSION_STRING` is the same string as `POOL_VERSION`.
