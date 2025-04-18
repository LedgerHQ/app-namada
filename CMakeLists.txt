#*******************************************************************************
#*   (c) 2018 -2024 Zondax AG
#*
#*  Licensed under the Apache License, Version 2.0 (the "License");
#*  you may not use this file except in compliance with the License.
#*  You may obtain a copy of the License at
#*
#*      http://www.apache.org/licenses/LICENSE-2.0
#*
#*  Unless required by applicable law or agreed to in writing, software
#*  distributed under the License is distributed on an "AS IS" BASIS,
#*  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#*  See the License for the specific language governing permissions and
#*  limitations under the License.
#********************************************************************************
cmake_minimum_required(VERSION 3.28)
include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/cpp-pm/hunter/archive/v0.25.5.tar.gz"
    SHA1 "a20151e4c0740ee7d0f9994476856d813cdead29"
    LOCAL
)

if(CMAKE_GENERATOR MATCHES "Ninja")
        message(FATAL_ERROR "This project does not support the Ninja generator. "
                            "Please use Unix Makefiles or another supported generator. "
                            "This error is typical in CLion. In this case, switch to generator Unix Makefiles.")
endif()

########################################################

project(ledger-namada VERSION 0.0.0)

set(CMAKE_CXX_STANDARD 17)
cmake_policy(SET CMP0025 NEW)
cmake_policy(SET CMP0144 NEW)

set(HUNTER_STATUS_DEBUG ON)
set(HUNTER_TLS_VERIFY OFF)

enable_testing()

option(ENABLE_FUZZING "Build with fuzzing instrumentation and build fuzz targets" OFF)
option(ENABLE_COVERAGE "Build with source code coverage instrumentation" OFF)
option(ENABLE_SANITIZERS "Build with ASAN and UBSAN" OFF)

string(APPEND CMAKE_C_FLAGS " -fno-omit-frame-pointer -g")
string(APPEND CMAKE_CXX_FLAGS " -fno-omit-frame-pointer -g")
string(APPEND CMAKE_LINKER_FLAGS " -fno-omit-frame-pointer -g")

hunter_add_package(fmt)
find_package(fmt CONFIG REQUIRED)

hunter_add_package(jsoncpp)
find_package(jsoncpp CONFIG REQUIRED)

hunter_add_package(GTest)
find_package(GTest CONFIG REQUIRED)

if(ENABLE_FUZZING)
    add_definitions(-DFUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION=1)
    SET(ENABLE_SANITIZERS ON CACHE BOOL "Sanitizer automatically enabled" FORCE)
    SET(CMAKE_BUILD_TYPE Debug)

    if (DEFINED ENV{FUZZ_LOGGING})
        add_definitions(-DFUZZING_LOGGING)
        message(FATAL_ERROR "Fuzz logging enabled")
    endif()

    set(CMAKE_CXX_CLANG_TIDY clang-tidy -checks=-*,bugprone-*,cert-*,clang-analyzer-*,-cert-err58-cpp,misc-*)

    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
        # require at least clang 3.2
        if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 10.0)
            message(FATAL_ERROR "Clang version must be at least 10.0!")
        endif()
    else()
        message(FATAL_ERROR
                "Unsupported compiler! Fuzzing requires Clang 10.\n"
                "1. Install clang-10 \n"
                "2. Use -DCMAKE_C_COMPILER=clang-10 -DCMAKE_CXX_COMPILER=clang++-10")
    endif()

    string(APPEND CMAKE_C_FLAGS " -fsanitize=fuzzer-no-link")
    string(APPEND CMAKE_CXX_FLAGS " -fsanitize=fuzzer-no-link")
    string(APPEND CMAKE_LINKER_FLAGS " -fsanitize=fuzzer-no-link")
endif()

if(ENABLE_COVERAGE)
    string(APPEND CMAKE_C_FLAGS " -fprofile-instr-generate -fcoverage-mapping")
    string(APPEND CMAKE_CXX_FLAGS " -fprofile-instr-generate -fcoverage-mapping")
    string(APPEND CMAKE_LINKER_FLAGS " -fprofile-instr-generate -fcoverage-mapping")
endif()

if(ENABLE_SANITIZERS)
    string(APPEND CMAKE_C_FLAGS " -fsanitize=address,undefined -fsanitize-recover=address,undefined")
    string(APPEND CMAKE_CXX_FLAGS " -fsanitize=address,undefined -fsanitize-recover=address,undefined")
    string(APPEND CMAKE_LINKER_FLAGS " -fsanitize=address,undefined -fsanitize-recover=address,undefined")
endif()

set (RETRIEVE_MAJOR_CMD
        "cat ${CMAKE_CURRENT_SOURCE_DIR}/app/Makefile.version | grep APPVERSION_M | cut -b 14- | tr -d '\n'"
)
set (RETRIEVE_MINOR_CMD
        "cat ${CMAKE_CURRENT_SOURCE_DIR}/app/Makefile.version | grep APPVERSION_N | cut -b 14- | tr -d '\n'"
)
execute_process(
        COMMAND bash "-c" ${RETRIEVE_MAJOR_CMD}
        RESULT_VARIABLE MAJOR_RESULT
        OUTPUT_VARIABLE MAJOR_VERSION
)
execute_process(
        COMMAND bash "-c" ${RETRIEVE_MINOR_CMD}
        RESULT_VARIABLE MINOR_RESULT
        OUTPUT_VARIABLE MINOR_VERSION
)

message(STATUS "LEDGER_MAJOR_VERSION [${MAJOR_RESULT}]: ${MAJOR_VERSION}" )
message(STATUS "LEDGER_MINOR_VERSION [${MINOR_RESULT}]: ${MINOR_VERSION}" )

add_definitions(
    -DLEDGER_MAJOR_VERSION=${MAJOR_VERSION}
    -DLEDGER_MINOR_VERSION=${MINOR_VERSION}
)
##############################################################
##{TODO}
##############################################################
#  static libs
file(GLOB_RECURSE LIB_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/app_mode.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/base58.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/bech32.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/bignum.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/hexutils.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/zxmacros.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/zxformat.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/segwit_addr.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/src/timeutils.c
        ####
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser_print_common.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser_print_txn.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser_impl.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser_impl_common.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser_impl_txn.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser_impl_masp.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/bech32_encoding.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/parser_address.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/crypto_helper.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/leb128.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/txn_validator.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/txn_delegation.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/c_api/rust.c
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/blake2/ref/blake2b-ref.c
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/blake2s/blake2s-ref.c
        )

add_library(app_lib STATIC ${LIB_SRC})

target_include_directories(app_lib PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/ledger-zxlib/include
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/lib
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/common
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/picohash/
       ###
        ${CMAKE_CURRENT_SOURCE_DIR}/app/rust/include
        ###
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/blake2/ref
        )

##############################################################
##############################################################
## Rust library for CPP tests
set(RUST_LIB_DIR "${CMAKE_CURRENT_SOURCE_DIR}/app/rust")

# Determine the Rust target triple based on the host system
if(CMAKE_HOST_SYSTEM_NAME STREQUAL "Linux")
    if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "aarch64")
        set(RUST_TARGET_TRIPLE "aarch64-unknown-linux-gnu")
    elseif(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "x86_64")
        set(RUST_TARGET_TRIPLE "x86_64-unknown-linux-gnu")
    else()
        message(FATAL_ERROR "Unsupported processor: ${CMAKE_HOST_SYSTEM_PROCESSOR}")
    endif()
elseif(CMAKE_HOST_SYSTEM_NAME STREQUAL "Darwin")
    if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "arm64")
        set(RUST_TARGET_TRIPLE "aarch64-apple-darwin")
    elseif(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "x86_64")
        set(RUST_TARGET_TRIPLE "x86_64-apple-darwin")
    else()
        message(FATAL_ERROR "Unsupported processor: ${CMAKE_HOST_SYSTEM_PROCESSOR}")
    endif()
else()
    message(FATAL_ERROR "Unsupported host system: ${CMAKE_HOST_SYSTEM_NAME}")
endif()

# Use debug mode for debugging tests
set(RUST_TARGET_DIR "${RUST_LIB_DIR}/target/${RUST_TARGET_TRIPLE}/release")

# Custom target for the Rust library
add_custom_target(RustLibClean
    COMMAND cargo clean
    WORKING_DIRECTORY ${RUST_LIB_DIR}
)
add_custom_target(RustLibBuild
    COMMAND cargo build --release --target ${RUST_TARGET_TRIPLE} --features cpp_tests
    WORKING_DIRECTORY ${RUST_LIB_DIR}
    DEPENDS RustLibClean
)

# Assuming the Rust library outputs a file named librslib.a
set(RUST_LIB "${RUST_TARGET_DIR}/librslib.a")

# Ensure the Rust library is built before the C++ project
add_library(rslib STATIC IMPORTED)
set_property(TARGET rslib PROPERTY IMPORTED_LOCATION ${RUST_LIB})
add_dependencies(rslib RustLibBuild)

# Ensure your C++ targets depend on the Rust library being built first
# For example, for your app_lib static library:
add_dependencies(app_lib rslib)

##############################################################
#  Fuzz Targets
if(ENABLE_FUZZING)
    set(FUZZ_TARGETS
        parser_parse
        )

    foreach(target ${FUZZ_TARGETS})
        add_executable(fuzz-${target} ${CMAKE_CURRENT_SOURCE_DIR}/fuzz/${target}.cpp)
        target_include_directories(fuzz-${target} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/deps/picohash)
        target_link_libraries(fuzz-${target} PRIVATE app_lib rslib)
        target_link_options(fuzz-${target} PRIVATE "-fsanitize=fuzzer")
    endforeach()
else()
##############################################################
#  Tests
file(GLOB_RECURSE TESTS_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/tests/*.cpp)

add_executable(unittests ${TESTS_SRC})
target_include_directories(unittests PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src
        ${CMAKE_CURRENT_SOURCE_DIR}/app/src/lib
        ###
        ${CMAKE_CURRENT_SOURCE_DIR}/deps/blake2/ref
        )

target_link_libraries(unittests PRIVATE
        GTest::gtest_main
        app_lib
        rslib
        fmt::fmt
        JsonCpp::JsonCpp)

add_compile_definitions(TESTVECTORS_DIR="${CMAKE_CURRENT_SOURCE_DIR}/tests/")
add_compile_definitions(COMPILE_MASP=1)
add_test(NAME unittests COMMAND unittests)
set_tests_properties(unittests PROPERTIES WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/tests)

##############################################################
endif()
