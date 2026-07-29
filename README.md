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

### MergeTree and storage

- [MergeTree interfaces](merge_tree.md)
- [MergeTree read path](merge_tree_read.md)
- [MergeTree read modes](MergeTree/mergetree_read_types.md)
- [Broken parts](MergeTree/broken_parts.md)
- [Merge implementation notes](doc_about_merge.md)
- [Disk abstractions](disks.md)
- [MinIO setup](minio.md)

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

### Profiling and client examples

- [CPU and memory flame graphs](Profiling/ClickHouse_flameGraph_func.md)
- [Profiling release builds with perf](Profiling/perf_release.md)
- [Python client and concurrent insert example](Annoying-conf/clickhouse_connect.md)

## Using this guide

Start with the topic closest to the subsystem you are investigating. Most pages assume familiarity with Linux, C++, SQL, and the ClickHouse source tree. Commands and source paths should be checked against the ClickHouse version you are using.

## Contributing

Corrections, clearer explanations, and version-specific updates are welcome. Keep examples reproducible, identify the ClickHouse version when behavior is version-dependent, and link to relevant source files where possible.
