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

add_subdirectory(cache)

set(SQUEASEL_SRC_DIR "${CMAKE_SOURCE_DIR}/be/src/thirdparty/squeasel")
set(MUSTACHE_SRC_DIR "${CMAKE_SOURCE_DIR}/be/src/thirdparty/mustache")
set(MPFIT_SRC_DIR "${CMAKE_SOURCE_DIR}/be/src/thirdparty/mpfit")

# where to put generated libraries
set(LIBRARY_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/util")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/util")

add_library(Util
  auth-util.cc
  avro-util.cc
  backend-gflag-util.cc
  benchmark.cc
  bitmap.cc
  bit-packing.cc
  bit-util.cc
  bloom-filter.cc
  bloom-filter-ir.cc
  cgroup-util.cc
  coding-util.cc
  codec.cc
  collection-metrics.cc
  common-metrics.cc
  compression-util.cc
  compress.cc
  cpu-info.cc
  cyclic-barrier.cc
  dynamic-util.cc
  debug-util.cc
  decompress.cc
  default-path-handlers.cc
  disk-info.cc
  error-util.cc
  event-metrics.cc
  filesystem-util.cc
  flat_buffer.cc
  hdfs-util.cc
  hdfs-bulk-ops.cc
  hdr-histogram.cc
  histogram-metric.cc
  impalad-metrics.cc
  jni-util.cc
  json-util.cc
  ldap-util.cc
  ldap-search-bind.cc
  ldap-simple-bind.cc
  logging-support.cc
  mem-info.cc
  memory-metrics.cc
  memusage-path-handlers.cc
  metrics.cc
  min-max-filter.cc
  min-max-filter-ir.cc
  minidump.cc
  mpfit-util.cc
  network-util.cc
  openssl-util.cc
  os-info.cc
  os-util.cc
  parse-util.cc
  path-builder.cc
  periodic-counter-updater
  pprof-path-handlers.cc
  progress-updater.cc
  process-state-info.cc
  redactor.cc
  runtime-profile.cc
  sharded-query-map-util.cc
  simple-logger.cc
  string-parser.cc
  string-util.cc
  symbols-util.cc
  system-state-info.cc
  static-asserts.cc
  summary-util.cc
  table-printer.cc
  test-info.cc
  thread.cc
  time.cc
  tuple-row-compare.cc
  uid-util.cc
  url-parser.cc
  ${SQUEASEL_SRC_DIR}/squeasel.c
  webserver.cc
  zip-util.cc
  ${MUSTACHE_SRC_DIR}/mustache.cc
  ${MPFIT_SRC_DIR}/mpfit.c)
add_dependencies(Util gen-deps gen_ir_descriptions)

add_library(UtilTests STATIC
  benchmark-test.cc
  bitmap-test.cc
  bit-packing-test.cc
  bit-stream-utils-test.cc
  bit-util-test.cc
  blocking-queue-test.cc
  bloom-filter-test.cc
  coding-util-test.cc
  cyclic-barrier-test.cc
  debug-util-test.cc
  dict-test.cc
  error-util-test.cc
  filesystem-util-test.cc
  fixed-size-hash-table-test.cc
  hdfs-util-test.cc
  hdr-histogram-test.cc
  logging-support-test.cc
  metrics-test.cc
  min-max-filter-test.cc
  openssl-util-test.cc
  os-info-test.cc
  os-util-test.cc
  parse-util-test.cc
  pretty-printer-test.cc
  priority-queue-test.cc
  proc-info-test.cc
  redactor-config-parser-test.cc
  redactor-test.cc
  redactor-test-utils.cc
  redactor-unconfigured-test.cc
  rle-test.cc
  runtime-profile-test.cc
  simple-logger-test.cc
  string-parser-test.cc
  string-util-test.cc
  symbols-util-test.cc
  sys-info-test.cc
  system-state-info-test.cc
  thread-pool-test.cc
  time-test.cc
  tuple-row-compare-test.cc
  uid-util-test.cc
  zip-util-test.cc
)
add_dependencies(UtilTests gen-deps gen_ir_descriptions)

# Squeasel requires C99 compatibility to build.
SET_SOURCE_FILES_PROPERTIES(${SQUEASEL_SRC_DIR}/squeasel.c
  PROPERTIES COMPILE_FLAGS -std=c99)

# shared library which provides native logging support to JVMs over JNI.
add_library(loggingsupport SHARED
  logging-support.cc
)

add_executable(parquet-reader parquet-reader.cc)
add_executable(impala-profile-tool impala-profile-tool.cc)

target_link_libraries(parquet-reader ${IMPALA_LINK_LIBS})
target_link_libraries(impala-profile-tool ${IMPALA_LINK_LIBS})

target_link_libraries(loggingsupport ${IMPALA_LINK_LIBS_DYNAMIC_TARGETS})