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

# where to put generated libraries.
set(LIBRARY_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/service")

# where to put generated binaries.
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/service")

# Mark the protobuf file as generated
set_source_files_properties(${CONTROL_SERVICE_PROTO_SRCS} PROPERTIES GENERATED TRUE)
set_source_files_properties(${DATA_STREAM_SERVICE_PROTO_SRCS} PROPERTIES GENERATED TRUE)

add_library(Service
  ${CONTROL_SERVICE_PROTO_SRCS}
  ${DATA_STREAM_SERVICE_PROTO_SRCS}
  frontend.cc
  fe-support.cc
  hs2-util.cc
  query-options.cc
  query-result-set.cc
)
add_dependencies(Service gen-deps)

# this shared library provides Impala executor functionality to FE test.
add_library(fesupport SHARED
  fe-support.cc
)
add_dependencies(fesupport gen-deps)

if ("${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN"
    OR "${CMAKE_BUILD_TYPE}" STREQUAL "UBSAN_FULL")
  target_link_libraries(fesupport "-Wl,--whole-archive \
    $ENV{IMPALA_TOOLCHAIN_PACKAGES_HOME}/llvm-$ENV{IMPALA_LLVM_DEBUG_VERSION}/lib/clang/$ENV{IMPALA_LLVM_UBSAN_BASE_VERSION}/lib/linux/libclang_rt.ubsan_standalone_cxx-${CMAKE_SYSTEM_PROCESSOR}.a \
    $ENV{IMPALA_TOOLCHAIN_PACKAGES_HOME}/llvm-$ENV{IMPALA_LLVM_DEBUG_VERSION}/lib/clang/$ENV{IMPALA_LLVM_UBSAN_BASE_VERSION}/lib/linux/libclang_rt.ubsan_standalone-${CMAKE_SYSTEM_PROCESSOR}.a \
    -Wl,--no-whole-archive")
endif()

target_link_libraries(fesupport ${IMPALA_LINK_LIBS_DYNAMIC_TARGETS})