# Building RC2040-ROMWBW

How to build RC2040-ROMWBW for the Pico (RP2040) and Pico 2 (RP2350), based on `CMakeLists.txt`, `makeboth.sh` and the `#define` switches in `RomWBW.c`.

> These steps have not been run with a Pico toolchain. Check them on your own machine.

## Prerequisites

```
sudo apt install cmake gcc-arm-none-eabi libnewlib-arm-none-eabi build-essential libstdc++-arm-none-eabi-newlib
git clone -b master https://github.com/raspberrypi/pico-sdk.git
cd pico-sdk && git submodule update --init
export PICO_SDK_PATH=/path/to/pico-sdk
```

`pico_sdk_import.cmake` is already in the project.

## Build

```
cd RC2040-ROMWBW
export PICO_SDK_PATH=/path/to/pico-sdk     # makeboth.sh hard-codes /home/pico-sdk, so edit it
./makeboth.sh      # makes Build2040 (rp2040) and Build2350 (rp2350), make -j7
```

Or by hand:

```
mkdir Build2040 && cd Build2040
cmake -DPICO_PLATFORM=rp2040 .. && make -j7      # Pico
cd .. && mkdir Build2350 && cd Build2350
cmake -DPICO_PLATFORM=rp2350 .. && make -j7      # Pico 2
```

- **Output:** `RomWBW.uf2`, plus `.bin`, `.hex` and `.elf`.

## CMake options

| Option | Values | Notes |
|---|---|---|
| `-DPICO_PLATFORM` | `rp2040` (Pico) or `rp2350` (Pico 2) | The only option `makeboth.sh` sets. An existing build cache shows `rp2040` as the default. |
| `-DCMAKE_BUILD_TYPE` | `Release`, `Debug`, `RelWithDebInfo`, `MinSizeRel` | Unset gives `Release` with the SDK, which is what you want for speed. |
| `-DPICO_SDK_PATH` | path | Alternative to the environment variable. |
| `-DPICO_SDK_FETCH_FROM_GIT=ON` | on/off | Downloads the SDK if no path is set. `PICO_SDK_FETCH_FROM_GIT_PATH` sets where it goes. |
| `-DPICO_BOARD` | e.g. `pico`, `pico2` | Optional. An existing build cache shows `pico`. |

Use a fresh build directory when you change `PICO_PLATFORM`, because CMake remembers the old value. Any shipped `build/` folder holds a cache pointing at a different SDK path, so delete it rather than reusing it.

## Fixed in `CMakeLists.txt`

- Language standards are C11 and C++17.
- `-Wl,--print-memory-usage` is set, so the build prints RAM and flash use.
- USB stdio is enabled. UART stdio is commented out, so output goes over USB only.
- `PICO_STDIO_USB_ENABLE_RESET_VIA_BAUD_RATE=1` is set, so a 1200 baud connection resets the board into the bootloader.
- `PICO_STDIO_USB_ENABLE_RESET_VIA_VENDOR_INTERFACE=0` is set.
- Libraries linked: `pico_multicore`, `pico_stdlib`, `hardware_gpio`, `hardware_pwm`, `hardware_pio`, `hardware_dma`, `hardware_irq`, `hardware_flash`, `hardware_spi`, `pico_fatfs`, `libz80` and `display`.
- The `display` library is always built and linked, even without `WithDisplay`. It pulls in `hardware_i2c`.
- `display/CMakeLists.txt` sets `PICO_BOARD pico_w`, but that only applies inside that subfolder. An existing cache shows the real board as `pico`, so nothing changes.

## Source switches

These are edited at the top of `RomWBW.c`, not passed to CMake.

| Switch | Default | Effect |
|---|---|---|
| `RCROMWBW` | defined | Uses the RomWBW PCB kit pin map. Comment it out for the RC2040 PCB kit. |
| `WithDisplay` | commented out | Adds the display and moves the USB/UART switch to GPIO12. |
| `FFS_ENABLE` | defined automatically on RP2350 only | Serial file transfer from the AUX button. It needs the extra RAM of the Pico 2. |
| `DevBoard` | not defined | Sound pins become GPIO20/21 instead of GPIO15/14. |
| `SD_PIN_MISO`, `SD_PIN_CS`, `SD_PIN_SCK`, `SD_PIN_MOSI` | 16, 17, 18, 19 | SD card pins on SPI0. |

## Chip-dependent settings (automatic)

| | Pico (RP2040) | Pico 2 (RP2350) |
|---|---|---|
| RAM pages (16 KB each) | 10 | 20 |
| ROM pages (16 KB each) | 32 (512 KB) | 32 (512 KB) |

The ROM is held in flash at a 1 MB offset (`FLASH_TARGET_OFFSET`).

## Local changes to be aware of

- **SD card:** the pins are set with `pico_fatfs_set_config()` just before `f_mount()`.
- **8-bit port:** with `RCROMWBW` defined, `PIOAp[]` is `{26,22,21,20,5,4,3,2}`, so port bits 4-7 are on GPIO5-2 because GPIO16-19 are used by the SD card. Without `RCROMWBW` it is `{2,3,4,5,20,21,26,27}`.
- **Speed:** `libz80` is compiled with `-Os`, and its hot code and the memory callbacks are placed in RAM. If the Pico 1 build fails with "RAM overflowed", trim what goes into RAM.

## At run time

The SD card needs the files from the project's `sd-card` folder, including `rc2040.ini`. The ROM image on the SD card is copied into flash at startup.
