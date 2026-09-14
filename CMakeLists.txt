cmake_minimum_required(VERSION 3.20)
project(bg3fgvk LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
# Where VulkanSDK/, streamline/ and Detours/ live. Override with -DBG3SE=... ;
# the default is the original Windows layout.
set(BG3SE "C:/Dev/bg3se-upscaler/External" CACHE PATH "Root holding VulkanSDK, streamline and Detours")
add_library(fgvk SHARED
  src/dllmain.cpp src/log.cpp src/config.cpp src/vkhooks.cpp src/slboot.cpp src/inputs.cpp)
target_include_directories(fgvk PRIVATE
  "${BG3SE}/VulkanSDK/Include" "${BG3SE}/streamline/include" "${BG3SE}/Detours/include" src)
target_link_libraries(fgvk PRIVATE "${BG3SE}/Detours/lib.X64/detours.lib")
target_compile_definitions(fgvk PRIVATE VK_NO_PROTOTYPES WIN32_LEAN_AND_MEAN NOMINMAX)
set_target_properties(fgvk PROPERTIES OUTPUT_NAME "fgvk")
if(MSVC)
  target_link_options(fgvk PRIVATE "/MAP:${CMAKE_BINARY_DIR}/fgvk.map")
else()
  # GCC auto-exports every symbol from a DLL. Left alone, this one would publish
  # Detours' entire public API -- and on a mod whose failure mode is export/dispatch
  # aliasing under Wine, that is not a cosmetic difference. MSVC exports nothing here.
  target_link_options(fgvk PRIVATE
    "-Wl,-Map=${CMAKE_BINARY_DIR}/fgvk.map" "-Wl,--exclude-all-symbols"
    -static -static-libgcc -static-libstdc++)
endif()

# Out-of-process stack dumper spawned by fgvk's watchdog on a present-loop stall.
add_executable(fgvk-stack src/tools/fgvk-stack.cpp)
target_compile_definitions(fgvk-stack PRIVATE WIN32_LEAN_AND_MEAN NOMINMAX)
target_link_libraries(fgvk-stack PRIVATE dbghelp)

# Unit tests (host-only, no Vulkan/Streamline dependency).
add_executable(fgvk-tests src/tests/tests_main.cpp src/tests/mvecscale_test.cpp src/tests/exportroute_test.cpp src/tests/slmodule_test.cpp)
target_include_directories(fgvk-tests PRIVATE src)
enable_testing()
add_test(NAME mvecscale COMMAND fgvk-tests)
