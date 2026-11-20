cmake_minimum_required(VERSION 3.20)
project(ens_ts CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(MSVC)
  add_compile_options(/W4)
else()
  add_compile_options(-Wall -Wextra)
endif()

set(ENS_M68K_TESTS_DIR "${CMAKE_SOURCE_DIR}/testdata/680x0/68000/m68t" CACHE PATH
  "Converted Tom Harte 68000 tests (.m68t, see docs/m68k.md); m68k_suite skips when absent")
set(ENS_ROM_DIR "${CMAKE_SOURCE_DIR}/roms" CACHE PATH "TS ROM directory (roms/, see roms/README.txt)")
set(ENS_DISK_DIR "" CACHE PATH "Folder of user disk images; image_user_files skips when empty")
option(ENS_BUILD_APP "Build the JUCE app (Windows, Linux, or macOS x64)" OFF)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})  # build/ens_tests
if(ENS_BUILD_APP)
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)  # the static cores link into the Linux VST3 .so
endif()

enable_testing()

# Header-only helpers shared by every chip core: #include "common/bits.h"
add_library(ens_common INTERFACE)
target_include_directories(ens_common INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/src)

add_library(ens_esp STATIC src/esp/esp_alu.cpp src/esp/esp_mac.cpp src/esp/esp_agen.cpp
  src/esp/esp_isa.cpp src/esp/esp.cpp)
target_link_libraries(ens_esp PUBLIC ens_common)

# Motorola 68000 core (copied from KS-32 Reborn).
add_library(ens_m68k STATIC src/m68k/m68k_decode.cpp src/m68k/m68k_cpu.cpp src/m68k/m68k_ea.cpp
  src/m68k/m68k_ops_data.cpp src/m68k/m68k_alu.cpp src/m68k/m68k_ops_arith.cpp
  src/m68k/m68k_ops_logic.cpp src/m68k/m68k_ops_flow.cpp src/m68k/m68k_dis.cpp)
target_link_libraries(ens_m68k PUBLIC ens_common)

# ES5506 OTTO (Plan 2): the TS's wave chip.
add_library(ens_otto STATIC src/otto/otto.cpp)
target_link_libraries(ens_otto PUBLIC ens_common)

# MC68681 DUART: the TS's serial and timer chip.
add_library(ens_duart STATIC src/duart/duart.cpp)
target_link_libraries(ens_duart PUBLIC ens_common)

# TS machine (Plan 3): ROM set, wave memory; the machine itself joins in later tasks.
add_library(ens_ts STATIC src/ts/roms.cpp src/ts/board.cpp src/ts/machine.cpp src/ts/panel.cpp src/ts/disk.cpp src/ts/disk_image.cpp src/ts/fdc.cpp src/ts/cd_image.cpp src/ts/cd_target.cpp src/ts/wd33c93.cpp)
target_link_libraries(ens_ts PUBLIC ens_otto ens_esp ens_m68k ens_duart)

# The JUCE-free engine between the machine and a host's audio callback (Plan 4): plain C++17, no file I/O.
add_library(ens_app STATIC src/app/engine.cpp src/app/resampler.cpp src/app/panel_layout.cpp
  src/app/media_record.cpp)
target_link_libraries(ens_app PUBLIC ens_ts)

# Tom Harte's per-instruction tests (.m68t reader and runner); the data itself is not in the repo.
add_library(ens_m68k_harte STATIC src/m68k/harte_format.cpp)
target_link_libraries(ens_m68k_harte PUBLIC ens_m68k)

# File reading and writing, the MIDI file reader and the WAV writer, for the tools and the tests.
add_library(ens_tools_common STATIC tools/common/files.cpp tools/common/smf.cpp tools/common/wav.cpp tools/common/cd_file.cpp)
target_include_directories(ens_tools_common PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/tools/common)
target_link_libraries(ens_tools_common PUBLIC ens_ts)
target_compile_definitions(ens_tools_common PRIVATE _CRT_SECURE_NO_WARNINGS)

add_subdirectory(tests)

# Command-line tools (file I/O allowed here, not in src/).
add_executable(esp_dis tools/esp_dis/main.cpp)
target_link_libraries(esp_dis PRIVATE ens_esp ens_common)
target_compile_definitions(esp_dis PRIVATE _CRT_SECURE_NO_WARNINGS)  # fopen on MSVC (C4996)

# 68000 per-instruction suite runner (tools/m68k_harte/convert_harte.py makes the .m68t files).
add_executable(m68k_harte tools/m68k_harte/main.cpp)
target_link_libraries(m68k_harte PRIVATE ens_m68k_harte)
target_compile_definitions(m68k_harte PRIVATE _CRT_SECURE_NO_WARNINGS)
add_test(NAME m68k_suite COMMAND m68k_harte ${ENS_M68K_TESTS_DIR})

# TS machine runner: boots the OS ROM in emulation and reports what it did, renders a MIDI file to WAV.
# tests/ts_render.h is the render loop it shares with the tests.
add_executable(tsemu tools/tsemu/main.cpp)
target_link_libraries(tsemu PRIVATE ens_ts ens_tools_common)
target_include_directories(tsemu PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/tests)
target_compile_definitions(tsemu PRIVATE _CRT_SECURE_NO_WARNINGS)

# The JUCE app (Plan 4): TS Reborn, Standalone and VST3. MSVC x64 on Windows, GCC x64 on Linux.
if(ENS_BUILD_APP)
  if(NOT CMAKE_SIZEOF_VOID_P EQUAL 8 OR NOT (MSVC OR CMAKE_SYSTEM_NAME STREQUAL "Linux" OR APPLE))
    message(FATAL_ERROR "ENS_BUILD_APP needs a 64-bit Windows, Linux, or macOS toolchain")
  endif()
  add_subdirectory(app)
endif()
