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

add_subdirectory(bufferpool)
add_subdirectory(io)

# where to put generated libraries
set(LIBRARY_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/runtime")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/runtime")

# Mark the protobuf file as generated
set_source_files_properties(${ROW_BATCH_PROTO_SRCS} PROPERTIES GENERATED TRUE)

add_library(Runtime
  blocking-row-batch-queue.cc
  buffered-tuple-stream.cc
  client-cache.cc
  collection-value.cc
  coordinator.cc
  coordinator-backend-state.cc
  coordinator-backend-resource-state.cc
  datetime-iso-sql-format-parser.cc
  datetime-iso-sql-format-tokenizer.cc
  datetime-parser-common.cc
  datetime-simple-date-format-parser.cc
  date-parse-util.cc
  date-value.cc
  debug-options.cc
  descriptors.cc
  dml-exec-state.cc
  exec-env.cc
  fragment-state.cc
  fragment-instance-state.cc
  hbase-table.cc
  hbase-table-factory.cc
  hdfs-fs-cache.cc
  initial-reservations.cc
  krpc-data-stream-mgr.cc
  krpc-data-stream-recvr.cc
  krpc-data-stream-sender.cc
  krpc-data-stream-sender-ir.cc
  lib-cache.cc
  mem-tracker.cc
  mem-pool.cc
  query-driver.cc
  query-exec-mgr.cc
  query-exec-params.cc
  query-state.cc
  test-env.cc
  types.cc
  raw-value.cc
  raw-value-ir.cc
  reservation-manager.cc
  row-batch.cc
  ${ROW_BATCH_PROTO_SRCS}
  runtime-filter.cc
  runtime-filter-bank.cc
  runtime-filter-ir.cc
  runtime-state.cc
  scanner-mem-limiter.cc
  sorted-run-merger.cc
  sorter.cc
  sorter-ir.cc
  spillable-row-batch-queue.cc
  string-value.cc
  thread-resource-mgr.cc
  timestamp-parse-util.cc
  timestamp-value.cc
  tuple.cc
  tuple-ir.cc
  tuple-row.cc
  tmp-file-mgr.cc
)
add_dependencies(Runtime gen-deps)