cmake_minimum_required(VERSION 3.30.5)

project("zygisk")

set(CXX_FLAGS "${CXX_FLAGS} -fno-exceptions -fno-rtti -fvisibility=hidden -fvisibility-inlines-hidden")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CXX_FLAGS}")

# the check is not needed for androidndk
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
set(EXTRA_LLVM_FLAGS
"-mllvm --polly"
"-mllvm --polly-run-inliner"
"-mllvm --polly-invariant-load-hoisting"
"-mllvm --polly-run-dce"
"-mllvm --polly-vectorizer=stripmine"
"-mllvm --polly-default-tile-size=16"
"-mllvm --polly-register-tiling"
"-mllvm --enable-loopinterchange"
"-mllvm --enable-partial-inlining"
"-mllvm --disable-mr-partial-inlining"
"-mllvm --enable-loop-flatten"
"-mllvm --enable-jump-table-to-switch"
"-mllvm --enable-dfa-jump-thread"
"-mllvm --enable-knowledge-retention"
"-mllvm --extra-vectorizer-passes"
"-mllvm --hot-cold-split"
#"-mllvm --regalloc-enable-advisor=release"
#"-mllvm --enable-local-reassign=true"
#"-mllvm --regalloc-eviction-max-interference-cutoff=20"
)

foreach(FLAG IN LISTS EXTRA_LLVM_FLAGS)
    add_compile_options("SHELL:${FLAG}")
endforeach()
foreach(FLAG IN LISTS EXTRA_LLVM_FLAGS)
    string(REPLACE " " "," LINKER_FLAG "${FLAG}")
    add_link_options("-Wl,${LINKER_FLAG}")
endforeach()

add_compile_options("-funsafe-math-optimizations")
add_link_options("-funsafe-math-optimizations")
add_compile_options("-fno-math-errno")
add_link_options("-fno-math-errno")

add_link_options("-Wl,-O3,--relax,--lto-O3,-S,--icf=safe")
endif()

if(NOT DEFINED TARGET_ABI)
    if(DEFINED ANDROID_ABI)
        set(TARGET_ABI "${ANDROID_ABI}")
    elseif(DEFINED CMAKE_ANDROID_ARCH_ABI)
        set(TARGET_ABI "${CMAKE_ANDROID_ARCH_ABI}")
    else()
        message(FATAL_ERROR "Could not determine target ABI (ANDROID_ABI or CMAKE_ANDROID_ARCH_ABI not set)")
    endif()
endif()

if (TARGET_ABI STREQUAL "arm64-v8a")
add_compile_options("-march=armv8.2-a+dotprod")
add_compile_options("-mtune=cortex-a710")
add_link_options("-march=armv8.2-a+dotprod")
add_link_options("-mtune=cortex-a710")
endif()
if (TARGET_ABI STREQUAL "armeabi-v7a")
add_compile_options("-mthumb")
add_compile_options("-mtune=cortex-a55")
add_link_options("-mthumb")
add_link_options("-mtune=cortex-a55")
endif()

include(CheckIPOSupported)
check_ipo_supported(RESULT CMAKE_IPO_AVAILABLE)
if (CMAKE_IPO_AVAILABLE)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()

link_libraries(log)

find_package(cxx REQUIRED CONFIG)

link_libraries(cxx::cxx)

add_library(zygisk SHARED zygisk.cpp pif_config.cpp local_cxa_atexit_finalize_impl/atexit.cpp)

add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/Dobby" Dobby)

target_link_libraries(zygisk PRIVATE dobby_static)
