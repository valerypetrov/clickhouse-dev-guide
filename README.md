# ClickHouse Developer Guide

A practical collection of English-language notes for understanding, building, testing, debugging, and profiling ClickHouse. The guides focus on source-code paths, storage internals, query execution, and reproducible operational workflows.

> This is a community-maintained learning resource and is not official ClickHouse documentation. Details may vary by ClickHouse version.

## Contents

### Build, test, and CI

- [Build ClickHouse](Compile/compile.md)
- [Troubleshoot compilation](Compile/compile_issue.md)
- [Run unit tests](Compile/unit-test.md)
- [Run integration tests](Tests/integration_test.md)
- [Read CI results](CI/How-to-read-CI-result.md)
- [PR 117170: acting on clickhouse-gh bot findings (fuzzer triage and an access-ordering fix)](PRs/117170/README.md)

### Core query engine and analyzer (Query Pipeline 2.0)

- [New Analyzer, QueryTreeNode AST, and Processor Execution Graph](Analyzer_analyzer_pipeline.md)

### TimeSeries storage engine and PromQL

- [ENGINE = TimeSeries, PromQL-to-SQL Translation, Welford/Chan Variance & Stale Markers](TimeSeries_timeseries_engine_promql.md)

### MergeTree, Object Storage, and Zero-Copy

- [MergeTree interfaces](merge_tree.md)
- [MergeTree read path](merge_tree_read.md)
- [MergeTree read modes](MergeTree/mergetree_read_types.md)
- [MergeTree Granules, Object Storage (S3/Azure), and Zero-Copy Replication](MergeTree_object_storage_zero_copy.md)
- [Broken parts](MergeTree/broken_parts.md)
- [Merge implementation notes](doc_about_merge.md)
- [Disk abstractions](disks.md)
- [MinIO setup](minio.md)

### Replication, ClickHouse Keeper, and Parallel Replicas

- [ReplicatedMergeTree Replication Log, Keeper Concurrency & Parallel Replicas](Replication_replicated_mergetree_keeper.md)

### SIMD vectorization, memory architecture, and profiling

- [SIMD Vectorization (AVX2/AVX-512/ARM Neon), Memory Infrastructure (PODArray/Arena) & FlameGraphs](Performance_simd_memory_layout.md)
- [CPU and memory flame graphs](Profiling/ClickHouse_flameGraph_func.md)
- [Profiling release builds with perf](Profiling/perf_release.md)

### Data types, expressions, and functions

- [Dynamic and JSON storage models](DataType/data_type_store_model.md)
- [Map column design and storage](DataType/column_Map.md)
- [Expression trees and actions](expressions.md)
- [Function return types](functions.md)
- [Utility classes](utils.md)

### Query execution and indexes

- [Distributed query send/receive path](distributed_query/send_recieve.md)
- [Hash join internals](Join/hash_join.md)
- [Text skip index internals](SkipIndex/text_index.md)
- [`apply_mutations_on_fly` internals](Mutations/apply_mutations_on_fly.md)
- [Python client and concurrent insert example](Annoying-conf/clickhouse_connect.md)

## Using this guide

Start with the topic closest to the subsystem you are investigating. Most pages assume familiarity with Linux, C++, SQL, and the ClickHouse source tree. Commands and source paths should be checked against the ClickHouse version you are using.

## Contributing

Corrections, clearer explanations, and version-specific updates are welcome. Keep examples reproducible, identify the ClickHouse version when behavior is version-dependent, and link to relevant source files where possible.
