# SPDX-License-Identifier: MIT

project(
  'pool_allocator',
  'c',
  version: '2.1.0',
  meson_version: '>=0.57.0',
  default_options: [
    'c_std=c11',
    'warning_level=3',
  ],
  license: 'MIT',
)

project_description = 'Static object pool allocator for safety-critical systems.'

# === Version ==================================================================
# include/pool.h defines the version macros, so a copy-in build has them.
# Configuration fails when they differ from the project version.

fs = import('fs')
version_parts = meson.project_version().split('.')
version_macros = {
  'POOL_VERSION_MAJOR': version_parts[0],
  'POOL_VERSION_MINOR': version_parts[1],
  'POOL_VERSION_PATCH': version_parts[2],
  'POOL_VERSION': '"' + meson.project_version() + '"',
}
header_lines = []
foreach line : fs.read('include' / 'pool.h').split('\n')
  header_lines += ' '.join(line.split())
endforeach
foreach name, value : version_macros
  if '#define @0@ @1@'.format(name, value) not in header_lines
    error('include/pool.h must define @0@ as @1@'.format(name, value))
  endif
endforeach

# === Build-time configuration =================================================

lookup_strategy = get_option('lookup_strategy')
pool_max_slots = get_option('pool_max_slots')
pool_item_size = get_option('pool_item_size')
atomicity = get_option('atomicity_mode')

lookup_define = {
  'linear': '-DPOOL_LOOKUP_STRATEGY=POOL_LOOKUP_LINEAR',
  'hash': '-DPOOL_LOOKUP_STRATEGY=POOL_LOOKUP_HASH',
}[lookup_strategy]

add_project_arguments(
  lookup_define,
  '-DPOOL_MAX_SLOTS=@0@'.format(pool_max_slots),
  '-DPOOL_ITEM_SIZE=@0@'.format(pool_item_size),
  language: 'c',
)

# 'auto' leaves pool_conf.h to its own detection (C11 on hosted, volatile
# where <stdatomic.h> is unavailable). The explicit modes pin the choice.
if atomicity == 'c11'
  add_project_arguments('-DPOOL_USE_C11_ATOMIC', language: 'c')
elif atomicity == 'volatile'
  add_project_arguments('-DPOOL_USE_VOLATILE_ATOMIC', language: 'c')
endif

# === Include directories ======================================================

public_headers = include_directories('include')

# === Library ==================================================================

library_sources = ['src/pool.c']

pool_lib = static_library(
  'pool',
  library_sources,
  include_directories: public_headers,
  install: true,
  install_dir: get_option('libdir'),
)

install_headers(
  [
    'include/pool.h',
    'include/pool_conf.h',
    'include/pool_platform.h',
    'include/pool_version.h',
  ],
  subdir: 'pool',
)

pool_dep = declare_dependency(
  include_directories: public_headers,
  link_with: pool_lib,
)

meson.override_dependency('pool', pool_dep)

# === Sim library: 16-bit MAU code path on a byte-addressable host ============
# Second build of the library with -DPOOL_SIMULATE_16BIT_MAU so a host test
# binary linked against it exercises the pool_byte_t == uint16_t code path
# end-to-end (the same path the TI C2000 cross-build runs but cannot test).

if get_option('build_tests')
  pool_sim_lib = static_library(
    'pool_sim',
    library_sources,
    include_directories: public_headers,
    c_args: ['-DPOOL_SIMULATE_16BIT_MAU'],
    install: false,
  )

  pool_sim_dep = declare_dependency(
    include_directories: public_headers,
    link_with: pool_sim_lib,
    compile_args: ['-DPOOL_SIMULATE_16BIT_MAU'],
    )
endif

pkg = import('pkgconfig')
pkg.generate(
  name: 'pool',
  description: project_description,
  filebase: 'pool',
  version: meson.project_version(),
  libraries: pool_lib,
)

# === Tests ====================================================================

if get_option('build_tests')
  subdir('tests')
endif
