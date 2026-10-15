# SPDX-License-Identifier: MIT

test_exe = executable(
  'test_pool',
  'test_pool.c',
  c_args: lookup_define,
  dependencies: [pool_dep],
)

test('test_pool', test_exe)

# Re-run the same source against the sim library so the test suite
# exercises the pool_byte_t == uint16_t (C2000) code path on a host CI
# runner. The cross-compile catches build-time issues; this catches
# runtime issues that only manifest under 16-bit storage.

test_16bit_exe = executable(
  'test_pool_16bit',
  'test_pool.c',
  c_args: lookup_define,
  dependencies: [pool_sim_dep],
)

test('test_pool_16bit_sim', test_16bit_exe)

# Slots wider than 16 bits of storage units, so the footprint must report
# POOL_ITEM_SIZE in full.
test_wide_exe = executable(
  'test_pool_wide_slots',
  ['test_pool.c'] + library_sources,
  c_args: [lookup_define, '-DPOOL_MAX_SLOTS=4', '-DPOOL_ITEM_SIZE=65552'],
  include_directories: public_headers,
)

test('test_pool_wide_slots', test_wide_exe)

# Concurrency stress test - single writer, many readers - exercises the
# documented threading contract. TSAN is opt-in via -Dtsan=true (it is
# incompatible with ASan, so it lives in its own override).

thread_dep = dependency('threads')

concurrency_override = []
if get_option('tsan')
  concurrency_override = ['b_sanitize=thread']
endif

test_concurrency_exe = executable(
  'test_pool_concurrency',
  'test_pool_concurrency.c',
  dependencies: [pool_dep, thread_dep],
  override_options: concurrency_override,
)

test('test_pool_concurrency', test_concurrency_exe, timeout: 60)

# Compiles every C block in README.md, CONTRIBUTING.md and docs/ against the
# public headers, so a documented signature cannot drift from the code. The
# 'docs' suite runs a compiler, not library code, so memory checks skip it.
test(
  'pool docs examples',
  import('python').find_installation(),
  args: [
    files('docs/check_snippets.py'),
    '--root', meson.project_source_root(),
    '--lang', 'c',
    '--prelude', '#include "pool.h"',
    '--',
  ] + meson.get_compiler('c').cmd_array() + [
    '-std=c11',
    '-Wall',
    '-Wextra',
    '-Wpedantic',
    '-Werror',
    '-Wno-unused-function',
    '-Wno-unused-variable',
    '-I' + (meson.project_source_root() / 'include'),
    '-c', '{source}', '-o', '{object}',
  ],
  suite: 'docs',
  timeout: 60,
)
