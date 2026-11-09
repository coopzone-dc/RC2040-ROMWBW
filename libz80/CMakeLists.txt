
add_library(libz80 z80.c)

# Hot code is placed in RAM (see Z80_FAST in z80.c), so optimise for size
# to keep the RAM footprint down.
target_compile_options(libz80 PRIVATE -Os)
