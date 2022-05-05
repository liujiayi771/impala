# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.


cmake_minimum_required(VERSION 2.6)

# generate CTest input files
enable_testing()

# Setting this enables compiling for assembly output.  To compile to assembly:
#  1. cd into the directory containing the source file
#  2. 'make help' will list the assembly file targets (i.e. <srcfile.s>
#  3. 'make <srcfile>.s' to build the assembly for that file.  The file is built
#      to CMakeFiles/<currentdir>.dir/<srcfile>.s
PROJECT(ASSEMBLER)

# compiler flags that are common across debug/release builds
#  -Wall: Enable all warnings.
#  -Wno-sign-compare: suppress warnings for comparison between signed and unsigned
#    integers
#   -fno-strict-aliasing: disable optimizations that assume strict aliasing. This
#       is unsafe to do if the code uses casts (which we obviously do).
#  -Wno-unknown-pragmas: suppress warnings for unknown (compiler specific) pragmas
#  -fsigned-char: on aarch64 platform, type of char default is unsigned char, here
#        set it to signed-char to be compatible with x86-64
#  -Wno-deprecated: gutil contains deprecated headers
#  -Wno-vla: we use C99-style variable-length arrays
#  -pthread: enable multithreaded malloc
#  -DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG: enable nanosecond precision for boost
#  -fno-omit-frame-pointers: Keep frame pointer for functions in register
if (CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
  SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -march=armv8-a+crc")
endif()
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -Wall -Wno-sign-compare -Wno-unknown-pragmas -pthread")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -fno-strict-aliasing -fno-omit-frame-pointer")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -fsigned-char")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -std=c++14")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -Wno-deprecated -Wno-vla")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -DBOOST_SYSTEM_NO_DEPRECATED")
SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -B $ENV{IMPALA_TOOLCHAIN_PACKAGES_HOME}/binutils-$ENV{IMPALA_BINUTILS_VERSION}/bin/")
IF($ENV{USE_GOLD_LINKER} STREQUAL "true")
  SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -fuse-ld=gold")
ENDIF()

# On Apple we build with clang and need libstdc++ instead of libc++
if (APPLE)
  SET(CXX_COMMON_FLAGS "${CXX_COMMON_FLAGS} -stdlib=libstdc++")
endif()

SET(CXX_COVERAGE_FLAGS "-fprofile-arcs -ftest-coverage -DCODE_COVERAGE_ENABLED")

# For any clang builds (currently only ASAN):
#   -Qunused-arguments: quiet warnings about unused arguments to clang because ccache
#        makes extra calls to clang which may have extra includes (-I) that are unused.
#   -fcolor-diagnostics: ensure clang generates colorized output, which is necessary
#        when using ccache as clang thinks it is not called from a terminal.
#   -Wno-zero-as-null-pointer-constant: We are slowly moving towards the use of nullptr,
#        but till we switch to it completely, we will ignore the warnings due to use of
#        NULL as a null pointer constant.
#   -Wno-c++17-extensions: ignore warnings caused due to the use of [[nodiscard]]
#        attribute which our current compiler does not support but is used in conjunction
#        with WARN_UNUSED_RESULT with our current toolchain to be effective.
#   -Wno-inconsistent-missing-destructor-override: ignore warnings to mark virtual
#        destructors with 'override' which is enforced by clang by not recommended by c++
#        core guidelines (read C.128).
SET(CXX_CLANG_FLAGS "-Qunused-arguments -fcolor-diagnostics -Wno-unused-local-typedef")
SET(CXX_CLANG_FLAGS "${CXX_CLANG_FLAGS} -fsigned-char")
if (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
  SET(CXX_CLANG_FLAGS "${CXX_CLANG_FLAGS} -march=armv8-a+crc")
endif()
SET(CXX_CLANG_FLAGS "${CXX_CLANG_FLAGS} -Wno-zero-as-null-pointer-constant")
SET(CXX_CLANG_FLAGS "${CXX_CLANG_FLAGS} -Wno-c++17-extensions")
SET(CXX_CLANG_FLAGS "${CXX_CLANG_FLAGS} -Wno-inconsistent-missing-destructor-override")
SET(CXX_CLANG_FLAGS "${CXX_CLANG_FLAGS} -Wno-return-type-c-linkage")
# For any gcc builds:
#   -g: Enable symbols for profiler tools
#   -Wno-unused-local-typedefs: Do not warn for local typedefs that are unused.
SET(CXX_GCC_FLAGS "-g -Wno-unused-local-typedefs")
if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 7.0)
  # We need to add additional arguments for GCC 7+. We go down this branch if building
  # with a non-GCC compiler of version 7+, but in that case CXX_GCC_FLAGS is not used,
  # so it is inconsequential. TODO: IMPALA-5490: make this non-conditional when we
  # upgrade GCC.
  #  -faligned-new: new will automatically align types. Otherwise "new Counter()" in the
  #       Kudu util code produces a warning (see KUDU-2094).
  #   TODO: -faligned-new is part of C++17, remove flag when we bump language version.
  SET(CXX_GCC_FLAGS "${CXX_GCC_FLAGS} -faligned-new")
endif()

# compiler flags for different build types (run 'cmake -DCMAKE_BUILD_TYPE=<type> .')
# For CMAKE_BUILD_TYPE=Debug
#   -ggdb: Enable gdb debugging
# For CMAKE_BUILD_TYPE=Release
#   -O3: Enable all compiler optimizations
#   -DNDEBUG: Turn off dchecks/asserts/debug only code.
#
# Debug information is stored as dwarf2 to be as compatible as possible
SET(CXX_FLAGS_DEBUG "${CXX_GCC_FLAGS} -ggdb -gdwarf-2")
# -Werror: compile warnings should be errors when using the toolchain compiler.
#   Enabled for DEBUG, ASAN, TSAN and UBSAN builds which are built pre-commit.
SET(CXX_FLAGS_DEBUG "${CXX_FLAGS_DEBUG} -Werror")
SET(CXX_FLAGS_RELEASE "${CXX_GCC_FLAGS} -O3 -DNDEBUG -gdwarf-2")
SET(CXX_FLAGS_ADDRESS_SANITIZER
        "${CXX_CLANG_FLAGS} -Werror -O1 -g -fsanitize=address -fno-omit-frame-pointer -DADDRESS_SANITIZER")

# Set the flags to the undefined behavior sanitizer, also known as "ubsan"
# Turn on sanitizer and debug symbols to get stack traces:
SET(CXX_FLAGS_UBSAN "${CXX_CLANG_FLAGS} -Werror -ggdb3 -fno-omit-frame-pointer -fsanitize=undefined")
# Set preprocessor macros to facilitate initialization the relevant configuration.
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -DUNDEFINED_SANITIZER")
# Calling getenv() in __ubsan_default_options doesn't work, likely because of
# initialization ordering. We need to double-quote to create a macro that expands
# to a string-literal.
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -DUNDEFINED_SANITIZER_SUPPRESSIONS=\\\"$ENV{IMPALA_HOME}/bin/ubsan-suppressions.txt\\\"")
# Add flags to enable symbol resolution in the stack traces:
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -rtlib=compiler-rt -lgcc_s")
# Ignore a number of noisy errors with too many false positives:
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -fno-sanitize=alignment,function,vptr,float-divide-by-zero,float-cast-overflow")
# Don't enforce wrapped signed integer arithmetic so that the sanitizer actually sees
# undefined wrapping:
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -fno-wrapv")
# To ease debugging, turn off all optimizations:
SET(CXX_FLAGS_UBSAN "${CXX_FLAGS_UBSAN} -O0")

# Set the flags to the thread sanitizer, also known as "tsan"
# Turn on sanitizer and debug symbols to get stack traces:
SET(CXX_FLAGS_TSAN "${CXX_CLANG_FLAGS} -Werror -O1 -ggdb3 -fno-omit-frame-pointer")
SET(CXX_FLAGS_TSAN "${CXX_FLAGS_TSAN} -fsanitize=thread -DTHREAD_SANITIZER -DDYNAMIC_ANNOTATIONS_ENABLED")
SET(CXX_FLAGS_TSAN "${CXX_FLAGS_TSAN} -DTHREAD_SANITIZER_SUPPRESSIONS=\\\"$ENV{IMPALA_HOME}/bin/tsan-suppressions.txt\\\"")

SET(CXX_FLAGS_TIDY "${CXX_CLANG_FLAGS}")
# Catching unused variables requires an optimization level greater than 0
SET(CXX_FLAGS_TIDY "${CXX_FLAGS_TIDY} -O1")
# Ignore assert() and DCHECK() to avoid dead code errors on "DCHECK(false); return
# nullptr" in impossible default switch/case statements.
SET(CXX_FLAGS_TIDY "${CXX_FLAGS_TIDY} -DNDEBUG")
# Turn all warnings back on. Some will be ignored via .clang-tidy's "Checks" value, but
# this allows different "Checks" settings to be used in different clang-tidy runs without
# recompiling.
SET(CXX_FLAGS_TIDY "${CXX_FLAGS_TIDY} -Wall -W -Weverything")
# The Tidy build is so verbose (becasue of -Weverything) that it is unlikely to be viewed
# in a terminal and most likely will be redirecto to less, a log file, or /dev/null. In
# those places color codes just make the output harder to read.
SET(CXX_FLAGS_TIDY "${CXX_FLAGS_TIDY} -fno-color-diagnostics")

# Set compile flags based on the build type.
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_DEBUG})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")
  SET(CMAKE_CXX_FLAGS ${CXX_FLAGS_RELEASE})
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_ADDRESS_SANITIZER}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "TIDY")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_TIDY}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_UBSAN}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN_FULL")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_UBSAN}")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DUNDEFINED_SANITIZER_FULL")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "TSAN")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_TSAN}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "TSAN_FULL")
  SET(CMAKE_CXX_FLAGS "${CXX_FLAGS_TSAN}")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DTHREAD_SANITIZER_FULL")
else()
  message(FATAL_ERROR "Unknown build type: ${CMAKE_BUILD_TYPE}")
endif()

if (ENABLE_CODE_COVERAGE)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CXX_COVERAGE_FLAGS}")
endif()

# Add flags that are common across build types
#  - fverbose-asm creates better annotated assembly.  This doesn't seem to affect
#    when building the binary.
# LLMV_CFLAGS - Adding llvm compile flags
SET(CMAKE_CXX_FLAGS "${CXX_COMMON_FLAGS} ${CMAKE_CXX_FLAGS}")
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fverbose-asm")
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${LLVM_CFLAGS}")

# Use ccache when found and not explicitly disabled by setting the DISABLE_CCACHE envvar.
find_program(CCACHE ccache)
set(RULE_LAUNCH_PREFIX)
if (CCACHE AND NOT DEFINED ENV{DISABLE_CCACHE})
  set(RULE_LAUNCH_PREFIX ccache)
  if ("${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER"
          OR "${CMAKE_BUILD_TYPE}" STREQUAL "TIDY"
          OR "${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN"
          OR "${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN_FULL"
          OR "${CMAKE_BUILD_TYPE}" STREQUAL "TSAN"
          OR "${CMAKE_BUILD_TYPE}" STREQUAL "TSAN_FULL")
    # Need to set CCACHE_CPP so that ccache calls clang with the original source file for
    # both preprocessing and compilation. Otherwise, ccache will use clang to preprocess
    # the file and then call clang with the preprocessed output if not cached. However,
    # the preprocessed output from clang may contain code (e.g. from macro expansions)
    # that generates compilation warnings that would not be reported if compiling the
    # original source directly with clang.
    SET(ENV{CCACHE_CPP} YES)
  endif()
endif()

# There can be RULE_LAUNCH_COMPILE / RULE_LAUNCH_LINK settings already at the parent
# level. The parent layer should wrap any launcher used here.
get_property(PARENT_RULE_LAUNCH_COMPILE GLOBAL PROPERTY RULE_LAUNCH_COMPILE)
get_property(PARENT_RULE_LAUNCH_LINK GLOBAL PROPERTY RULE_LAUNCH_LINK)

set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE
        "${PARENT_RULE_LAUNCH_COMPILE} ${RULE_LAUNCH_PREFIX}")
set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK
        "${PARENT_RULE_LAUNCH_LINK} ${RULE_LAUNCH_PREFIX}")

# Thrift requires these definitions for some types that we use
add_definitions(-DHAVE_INTTYPES_H -DHAVE_NETINET_IN_H -DHAVE_NETDB_H)

# Kudu flags. 1. Enable full support for all backing types of kudu::Slices.
# 2. Don't include stubs.h
add_definitions(-DKUDU_HEADERS_USE_RICH_SLICE -DKUDU_HEADERS_NO_STUBS)

# Set clang flags for cross-compiling to IR.
# IR_COMPILE is #defined for the cross compile to remove code that bloats the IR.
# We enable basic optimizations (-O1) to reduce the IR size and speed up runtime JIT.
# Empirically, the runtime JIT produces slightly better code when starting with IR that
# was optimized at -O1. Higher optimization levels tend to bloat the code.
#  -Wno-deprecated: gutil contains deprecated headers
#  -Wno-return-type-c-linkage: UDFs return C++ classes but use C linkage to prevent
#       mangling
#  -DBOOST_NO_EXCEPTIONS: call a custom error handler for exceptions in codegen'd code.
set(CLANG_IR_CXX_FLAGS "-emit-llvm" "-c" "-std=c++14" "-DIR_COMPILE" "-DHAVE_INTTYPES_H"
        "-DHAVE_NETINET_IN_H" "-DBOOST_DATE_TIME_POSIX_TIME_STD_CONFIG" "-DBOOST_NO_EXCEPTIONS"
        "-DKUDU_HEADERS_NO_STUBS" "-fcolor-diagnostics" "-Wno-deprecated"
        "-Wno-return-type-c-linkage" "-fsigned-char" "-O1")

if (CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
  set(CLANG_IR_CXX_FLAGS "${CLANG_IR_CXX_FLAGS}" "-march=armv8-a+crc"
          "-DCACHELINESIZE_AARCH64=${CACHELINESIZE_AARCH64}")
endif()

# -Werror: compile warnings should be errors when using the toolchain compiler.
set(CLANG_IR_CXX_FLAGS "${CLANG_IR_CXX_FLAGS}" "-Werror")

if ("${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER")
  SET(CLANG_IR_CXX_FLAGS "${CLANG_IR_CXX_FLAGS}" "-DADDRESS_SANITIZER")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RELEASE")
  SET(CLANG_IR_CXX_FLAGS "${CLANG_IR_CXX_FLAGS}" "-DNDEBUG")
endif()

if ("${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN_FULL")
  set(CLANG_IR_CXX_FLAGS "${CLANG_IR_CXX_FLAGS}" "-DUNDEFINED_SANITIZER"
          "-fno-omit-frame-pointer" "-fsanitize=undefined" "-fno-wrapv" "-ggdb3"
          "-fno-sanitize=alignment,function,vptr,float-divide-by-zero,float-cast-overflow"
          "-DUNDEFINED_SANITIZER_SUPPRESSIONS=\\\"$ENV{IMPALA_HOME}/bin/ubsan-suppressions.txt\\\"")
endif()

IF($ENV{ENABLE_IMPALA_IR_DEBUG_INFO} STREQUAL "true")
  # -g: emit debug symbols in IR. These increase IR size and memory overhead of LLVM, but
  #     are useful for debugging codegened code and interpreting codegen disassembly
  #     dumps.
  SET(CLANG_IR_CXX_FLAGS "${CLANG_IR_CXX_FLAGS}" "-g")
endif()

# Flags to pass to LLVM's opt to further optimize cross-compiled IR.
#  -inline: inline with low threshold to get rid of trivial accessor functions.
set(LLVM_OPT_IR_FLAGS "-inline" "-inlinehint-threshold=10" "-inline-threshold=10")

# Additional compile flags that will hide symbols by default, e.g. for building
# UDFs. We have both a concatenated string version and a list version for convenience,
# depending on what is needed in the context.
set(HIDE_SYMBOLS "-fvisibility=hidden -fvisibility-inlines-hidden")
set(HIDE_SYMBOLS_ARGS "${HIDE_SYMBOLS_STRING}")
separate_arguments(HIDE_SYMBOLS_ARGS)

# setup doc generation with Doxygen
find_package(Doxygen)
if (DOXYGEN_FOUND)
  set(DOXYGEN_OUTPUT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/build/docs)
  # Possible to not input the subdirs one by one?
  set(CMAKE_DOXYGEN_INPUT
          ${CMAKE_SOURCE_DIR}/be/src
          ${CMAKE_SOURCE_DIR}/be/src/common/
          ${CMAKE_SOURCE_DIR}/be/src/exec/
          ${CMAKE_SOURCE_DIR}/be/src/exprs/
          ${CMAKE_SOURCE_DIR}/be/src/runtime/
          ${CMAKE_SOURCE_DIR}/be/src/service/
          ${CMAKE_SOURCE_DIR}/be/src/thrift/
          ${CMAKE_SOURCE_DIR}/be/src/util/
          ${CMAKE_SOURCE_DIR}/be/src/transport/
          )
  # CMake appends using ';'. doxygen wants spaces
  string(REPLACE ";" " " DOXYGEN_INPUT "${CMAKE_DOXYGEN_INPUT}")
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/.impala.doxy
          ${CMAKE_CURRENT_SOURCE_DIR}/build/config/.impala.doxy)
  file(MAKE_DIRECTORY ${DOXYGEN_OUTPUT_DIR})
  add_custom_target(docs
          COMMAND ${CMAKE_COMMAND} -E echo_append "Building Docs..."
          COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/build/config/.impala.doxy
          )
else (DOXYGEN_FOUND)
  MESSAGE(STATUS "WARNING: Doxygen not found - Docs will not be created")
endif(DOXYGEN_FOUND)

# resolve "#include "<subdir>/<name>.h"
include_directories(BEFORE ${CMAKE_CURRENT_SOURCE_DIR}/src)

# resolve includes of generated code
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/generated-sources)

set(CLANG_INCLUDE_FLAGS)

# Ensure that clang uses the gcc toolchain headers.
set(CLANG_BASE_FLAGS --gcc-toolchain=${GCC_ROOT})
set(CLANG_INCLUDE_FLAGS ${CLANG_BASE_FLAGS})

set(CLANG_INCLUDE_FLAGS
        ${CLANG_INCLUDE_FLAGS}
        "-I${CMAKE_CURRENT_SOURCE_DIR}/src"
        "-I${CMAKE_CURRENT_SOURCE_DIR}/generated-sources"
        "-I${THRIFT_INCLUDE_DIR}"
        "-I${SQUEASEL_INCLUDE_DIR}"
        "-I${GLOG_INCLUDE_DIR}"
        "-I${GFLAGS_INCLUDE_DIR}"
        "-I${RAPIDJSON_INCLUDE_DIR}"
        "-I${AVRO_INCLUDE_DIR}"
        "-I${ORC_INCLUDE_DIR}"
        # Include Boost as a system directory to suppress warnings from headers.
        "-isystem${BOOST_INCLUDEDIR}"
        # Required so that jni.h can be found during Clang compilation
        "-I${JAVA_INCLUDE_PATH}"
        "-I${JAVA_INCLUDE_PATH2}"
        "-I${RE2_INCLUDE_DIR}"
        "-I${SASL_INCLUDE_DIR}"
        "-I${BZIP2_INCLUDE_DIR}"
        "-I${ZLIB_INCLUDE_DIR}"
        "-I${OPENSSL_INCLUDE_DIR}"
        "-I${PROTOBUF_INCLUDE_DIR}"
        "-I${CCTZ_INCLUDE_DIR}"
        )

# allow linking of static libs into dynamic lib
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# set compile output directory
if ("${CMAKE_BUILD_TYPE}" STREQUAL "DEBUG" OR
        "${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER" OR
        "${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN" OR
        "${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN_FULL" OR
        "${CMAKE_BUILD_TYPE}" STREQUAL "TSAN" OR
        "${CMAKE_BUILD_TYPE}" STREQUAL "TSAN_FULL")
  set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build/debug/")
else()
  set(BUILD_OUTPUT_ROOT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/build/release/")
endif()

# Create a latest link so that scripts can pick up the correct build automatically
FILE(MAKE_DIRECTORY ${BUILD_OUTPUT_ROOT_DIRECTORY})
if (NOT APPLE)
  set(MORE_ARGS "-T")
endif()
EXECUTE_PROCESS(COMMAND ln ${MORE_ARGS} -sf ${BUILD_OUTPUT_ROOT_DIRECTORY}
        ${CMAKE_CURRENT_SOURCE_DIR}/build/latest)

# Determine what functions are available on the current platform.
INCLUDE(CheckFunctionExists)
CHECK_FUNCTION_EXISTS(sched_getcpu HAVE_SCHED_GETCPU)
CHECK_FUNCTION_EXISTS(pipe2 HAVE_PIPE2)
CHECK_FUNCTION_EXISTS(sync_file_range HAVE_SYNC_FILE_RANGE)

# linux/fs.h defines HAVE_FALLOCATE whether or not the function is available,
# which is why we use IMPALA_HAVE_FALLOCATE here.
CHECK_FUNCTION_EXISTS(fallocate IMPALA_HAVE_FALLOCATE)
CHECK_FUNCTION_EXISTS(preadv HAVE_PREADV)
INCLUDE(CheckIncludeFiles)
CHECK_INCLUDE_FILES(linux/magic.h HAVE_MAGIC_H)

# This is a list of impala library dependencies. Individual libraries
# must not specify library dependencies in their own CMakeLists.txt file.
# Enclose the impala libraries in -Wl,--start-group and -Wl,--end-group
# to resolve cyclic dependencies. As long as those flags are given,
# the order in which impala libraries are listed below does not matter.
# Note: The ld documentation discourages auto-resolving cyclic dependencies
# for performance reasons.
if (NOT APPLE)
  # When compiling on Mac with clang using these linker flags are undefined and Clang on
  # Mac will abort on unknown compiler or linker flags. In the long-term we should
  # move away from using these flags to have a coherent build on OS X and Linux.
  set(WL_START_GROUP "-Wl,--start-group")
  set(WL_END_GROUP "-Wl,--end-group")
endif()
set (IMPALA_LIBS
  BufferPool
  CodeGen
  Common
  Parquet
  Exec
  Exprs
  GlobalFlags
  histogram_proto
  ImpalaThrift
  Io
  kudu_util
  Rpc
  pb_util_proto
  Runtime
  Service
  Udf
  Util
  UtilCache
  )

set (IMPALA_LINK_LIBS
  ${WL_START_GROUP}
  ${IMPALA_LIBS}
  ${WL_END_GROUP}
)

# If using dynamic linking, -Wl does not have any effect (it's only for .a files). So we
# need to add these redundant dependencies to resolve the circular references in our
# libraries when dynamic linking is enabled.
if (BUILD_SHARED_LIBS)
  set (IMPALA_LINK_LIBS ${IMPALA_LINK_LIBS}
    BufferPool
    Io
    Runtime
    Parquet
    Exec
    CodeGen
    Exprs
    Rpc
    Service
    security
    ImpalaThrift
    GlobalFlags
    Common
    Udf
    )
endif ()

set (IMPALA_DEPENDENCIES
  snappy
  lz4
  zstd
  re2
  ${Boost_LIBRARIES}
  ${LLVM_MODULE_LIBS}
  thrift
  cyrus_sasl
  openssl_ssl
  openssl_crypto
  crcutil
  gutil
  glog
  gflags
  libev
  libunwind
  pprof
  breakpad_client
  hdfs
  zlib
  bzip2
  avro
  orc
  java_jvm
  cctz)

# Add all external dependencies. They should come after the impala libs.
set (IMPALA_LINK_LIBS ${IMPALA_LINK_LIBS}
  ${IMPALA_DEPENDENCIES}
  -lrt
  -ldl # Needed for LLVM
)

# The above link list does not include tcmalloc. This is because the Impala JVM support
# libraries (libfesupport, libloggingsupport) cannot use tcmalloc in all cases. When they
# are started up by the FE (for tests) the jvm has already made allocations before
# tcmalloc can be loaded. In all other binaries, we can use tcmalloc except the address
# sanitizer build. Address sanitizer is incompatible with tcmalloc (they both intercept
# malloc/free)
set (IMPALA_LINK_LIBS_NO_TCMALLOC ${IMPALA_LINK_LIBS})
if (NOT "${CMAKE_BUILD_TYPE}" STREQUAL "ADDRESS_SANITIZER" AND
    NOT "${CMAKE_BUILD_TYPE}" STREQUAL "TSAN" AND
    NOT "${CMAKE_BUILD_TYPE}" STREQUAL "TSAN_FULL")
  set (IMPALA_LINK_LIBS ${IMPALA_LINK_LIBS} tcmallocstatic)
  set (UNIFIED_TEST_LINK_LIBS ${UNIFIED_TEST_LINK_LIBS} tcmallocstatic)
endif()

# When we link statically, we need to replace the static libhdfs.a with the dynamic
# version otherwise the dynamic support libraries will pickup the static libhdfs.a
# library. The result will not compile as libhdfs.a is not compiled with -fpic. The same
# is true for other system dependencies that we don't have control over.
set(IMPALA_LINK_LIBS_DYNAMIC_TARGETS ${IMPALA_LINK_LIBS_NO_TCMALLOC})
list(REMOVE_ITEM IMPALA_LINK_LIBS_DYNAMIC_TARGETS hdfs)
set(IMPALA_LINK_LIBS_DYNAMIC_TARGETS ${IMPALA_LINK_LIBS_DYNAMIC_TARGETS}
  ${HDFS_SHARED_LIB})

set (IMPALA_TEST_LINK_LIBS ${IMPALA_LINK_LIBS} gtest)
set (UNIFIED_TEST_LINK_LIBS ${UNIFIED_TEST_LINK_LIBS} gtest)

MESSAGE(STATUS "Compiler Flags: ${CMAKE_CXX_FLAGS}")

if (CMAKE_DEBUG)
  MESSAGE(STATUS "Linker Libs: ${IMPALA_LINK_LIBS}")
endif()

set(LLVM_IR_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/llvm-ir")
file(MAKE_DIRECTORY ${LLVM_IR_OUTPUT_DIRECTORY})

# Variable to use to aggregate all of the filter patterns, joined by ":"
set_property(GLOBAL PROPERTY AGG_UNIFIED_FILTER_PATTERN)

# Function to generate rule to cross compile a source file to an IR module.
# This should be called with the .cc src file and it will generate a
# src-file-ir target that can be built.
# e.g. COMPILE_TO_IR(test.cc) generates the "test-ir" make target.
# Note: this is duplicated in udf_samples/CMakeLists.txt
function(COMPILE_TO_IR SRC_FILE)
  get_filename_component(BASE_NAME ${SRC_FILE} NAME_WE)
  set(OUTPUT_FILE "${LIBRARY_OUTPUT_PATH}/${BASE_NAME}.ll")
  add_custom_command(
    OUTPUT ${OUTPUT_FILE}
    COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} ${HIDE_SYMBOLS_ARGS}
            ${CLANG_INCLUDE_FLAGS} ${SRC_FILE} -o ${OUTPUT_FILE}
    DEPENDS ${SRC_FILE})
  add_custom_target(${BASE_NAME}-ir ALL DEPENDS ${OUTPUT_FILE})
endfunction(COMPILE_TO_IR)

# Gutil is a little bit special
add_subdirectory(src/gutil)

# compile these subdirs using their own CMakeLists.txt
add_subdirectory(src/codegen)
add_subdirectory(src/common)
add_subdirectory(src/exec)
add_subdirectory(src/exprs)
add_subdirectory(src/kudu/util)
add_subdirectory(src/runtime)
add_subdirectory(src/service)
add_subdirectory(src/rpc)
add_subdirectory(src/udf)
add_subdirectory(src/udf_samples)
add_subdirectory(src/util)

# Thrift generated files have unused variables.  Ignore those compiler
# warnings by adding this flag.  Note: impala subdirectories should be
# added *before* this so we can fix our issues.
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-variable")
add_subdirectory(generated-sources/gen-cpp)

link_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}/build/common
  ${CMAKE_CURRENT_SOURCE_DIR}/build/exec
  ${CMAKE_CURRENT_SOURCE_DIR}/build/exprs
  ${CMAKE_CURRENT_SOURCE_DIR}/build/rpc
  ${CMAKE_CURRENT_SOURCE_DIR}/build/runtime
  ${CMAKE_CURRENT_SOURCE_DIR}/build/service
  ${CMAKE_CURRENT_SOURCE_DIR}/build/util
  ${CMAKE_CURRENT_SOURCE_DIR}/build/transport
)

get_property(TOTAL_UNIFIED_FILTER_PATTERN GLOBAL PROPERTY AGG_UNIFIED_FILTER_PATTERN)

# only generate statically linked libs and executables
set(BUILD_SHARED_LIBS OFF)

# where to put generated libraries
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")
set(ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}")