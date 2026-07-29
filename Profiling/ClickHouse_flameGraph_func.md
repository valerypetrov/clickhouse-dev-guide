# ClickHouse CPU and Memory Flame Graph Analysis Guide

This document explains how to generate CPU and memory flame graphs using `system.trace_log`, `query_profiler_cpu_time_period_ns`, `memory_profiler_sample_probability`, and the `flameGraph` aggregate function (the client must be started with `--allow_introspection_functions=1`).

---

## Prerequisites

- A compiled or installed `clickhouse` client and server.
- The flame graph script `flamegraph.pl` (for example, placed in `~/Perf/`).
- Optional: Install a debug-symbol package such as `clickhouse-common-static-dbg` to show symbol names instead of addresses in stack traces.

---

## 1. Prepare Test Data

Create a `MergeTree` table and insert sample data:

```sql
CREATE TABLE default.hits_test
(
    `SearchPhrase` String,
    `UserID` UInt64
)
ENGINE = MergeTree
ORDER BY SearchPhrase
SETTINGS index_granularity = 8192;

INSERT INTO hits_test
SELECT
    arrayElement(['clickhouse', 'database', 'flamegraph', 'performance', ''], rand() % 5 + 1) AS SearchPhrase,
    rand() % 100000 AS UserID
FROM numbers(30000000);
```

---

## 2. CPU Sampling and Test Query

Enable periodic CPU sampling (in nanoseconds), run the query, and **note the Query id returned by the server** (`abc123` is used as a placeholder below).

```sql
SET query_profiler_cpu_time_period_ns = 10000000;

SELECT *
FROM hits_test
ORDER BY rand() DESC;
```

Note: For CPU traces, `flameGraph` can use `arrayReverse(trace)` to display stack frames in the conventional flame graph order.

---

## 3. Verify That `trace_log` Contains CPU Records

Replace `'abc123'` with the actual `query_id`:

```sql
SELECT count()
FROM system.trace_log
WHERE query_id = 'abc123'
  AND trace_type = 'CPU'
  AND event_date = today();
```

---

## 4. Generate a CPU Flame Graph (SVG)

```bash
~/ClickHouse/build_release/programs/clickhouse client \
    --allow_introspection_functions=1 \
    -q "SELECT arrayJoin(flameGraph(arrayReverse(trace)))
        FROM system.trace_log
        WHERE trace_type = 'CPU'
          AND query_id = 'abc123'
          AND event_date = today()" \
    | ~/Perf/flamegraph.pl > ~/Perf/output/flame_cpu.svg
```

Open the resulting `flame_cpu.svg` in a browser to zoom and search the graph.

---

## 5. Optional: Memory Flame Graphs

### 5.1 Enable Memory Sampling and Rerun the Query

Increase the sampling probability, lower the untracked-memory threshold, and then **run a similar query again**. Record the **new** `query_id`.

```sql
SET memory_profiler_sample_probability = 1, max_untracked_memory = 1;

SELECT SearchPhrase, COUNT(DISTINCT UserID) AS u
FROM hits_test
WHERE SearchPhrase <> ''
GROUP BY SearchPhrase
ORDER BY u DESC
LIMIT 10;
```

### 5.2 All Allocated Bytes (by `size`)

```bash
~/ClickHouse/build_release/programs/clickhouse client \
    --allow_introspection_functions=1 \
    -q "SELECT arrayJoin(flameGraph(trace, size))
        FROM system.trace_log
        WHERE trace_type = 'MemorySample'
          AND query_id = 'abc123'
          AND event_date = today()" \
    | ~/Perf/flamegraph.pl --countname=bytes --color=mem > ~/Perf/output/flame_mem.svg
```

### 5.3 Allocations Still Outstanding When the Query Finishes (`trace`, `size`, `ptr`)

The three-argument form uses the same `ptr` to match allocations (`size > 0`) with deallocations (`size < 0`); **the flame graph retains allocations that could not be matched**.

Use **`trace_type = 'MemorySample'`**: in `trace_log`, **`ptr` is meaningful only for `MemorySample`** (both allocation and deallocation samples include a pointer). The `Memory` type records a different class of events, such as crossing a profiler threshold, and **usually does not include `ptr`**. If `'Memory'` is used by mistake and every `ptr` is 0, `flameGraph` takes the "no pointer" branch, whose semantics differ from "outstanding after matching by `ptr`."

Optional: To retain more "unmatched" allocations after the query finishes (for example, in a scenario that uses the uncompressed cache), enable the following cache-related settings:

```sql
SET memory_profiler_sample_probability = 1, max_untracked_memory = 1,
    use_uncompressed_cache = 1,
    merge_tree_max_rows_to_use_cache = 100000000000,
    merge_tree_max_bytes_to_use_cache = 1000000000000;
```

```bash
~/ClickHouse/build_release/programs/clickhouse client \
    --allow_introspection_functions=1 \
    -q "SELECT arrayJoin(flameGraph(trace, size, ptr))
        FROM system.trace_log
        WHERE trace_type = 'MemorySample'
          AND query_id = 'abc123'
          AND event_date = today()" \
    | ~/Perf/flamegraph.pl --countname=bytes --color=mem > ~/Perf/output/flame_mem_unfreed.svg
```

---

## 6. Complete Shell Examples (Fixed Paths and `query_id`)

The following commands assume this local directory layout: the `clickhouse` client is in `~/ClickHouse/build_release/programs/`, the flame graph script is in `~/Perf/`, and output is written to `~/Perf/output/`. Replace the `query_id` with your own query ID.

**CPU flame graph -> `flame_cpu.svg`**

```bash
~/ClickHouse/build_release/programs/clickhouse client \
    --allow_introspection_functions=1 \
    -q "SELECT arrayJoin(flameGraph(arrayReverse(trace)))
        FROM system.trace_log
        WHERE trace_type = 'CPU'
          AND query_id = 'cd9c0cd3-4ead-467a-bbb8-a9cdbc470518'
          AND event_date = today()" \
    | ~/Perf/flamegraph.pl > ~/Perf/output/flame_cpu.svg
```
<img width="1243" height="721" alt="image" src="https://github.com/user-attachments/assets/f4f17907-f03d-4e51-bbf1-e82fe43822a7" />

**Memory samples (allocation stacks aggregated by byte count) -> `flame_mem.svg`**

```bash
~/ClickHouse/build_release/programs/clickhouse client \
    --allow_introspection_functions=1 \
    -q "SELECT arrayJoin(flameGraph(trace, size))
        FROM system.trace_log
        WHERE trace_type = 'MemorySample'
          AND query_id = 'cd9c0cd3-4ead-467a-bbb8-a9cdbc470518'
          AND event_date = today()" \
    | ~/Perf/flamegraph.pl --countname=bytes --color=mem > ~/Perf/output/flame_mem.svg
```
<img width="1210" height="524" alt="image" src="https://github.com/user-attachments/assets/b4b94b80-ac19-4277-9793-2d292e7b8805" />

**Outstanding allocations (`trace`, `size`, `ptr`) -> `flame_mem_unfreed.svg`**

(The filter uses `trace_type = 'MemorySample'`, as in section 5.3.)

```bash
~/ClickHouse/build_release/programs/clickhouse client \
    --allow_introspection_functions=1 \
    -q "SELECT arrayJoin(flameGraph(trace, size, ptr))
        FROM system.trace_log
        WHERE trace_type = 'MemorySample'
          AND query_id = 'cd9c0cd3-4ead-467a-bbb8-a9cdbc470518'
          AND event_date = today()" \
    | ~/Perf/flamegraph.pl --countname=bytes --color=mem > ~/Perf/output/flame_mem_unfreed.svg
```

---

## Key Points

| Setting or expression | Purpose |
|------|------|
| `query_profiler_cpu_time_period_ns` | Controls the CPU stack sampling interval |
| `trace_type = 'CPU'` + `flameGraph(arrayReverse(trace))` | Generates the CPU flame graph |
| `memory_profiler_sample_probability` / `max_untracked_memory` | Triggers memory sampling |
| `trace_type = 'MemorySample'` + `flameGraph(trace, size)` | Aggregates sampled allocation stacks by byte count |
| `trace_type = 'MemorySample'` + `flameGraph(trace, size, ptr)` | Shows allocations still outstanding after matching by `ptr` |
| `--allow_introspection_functions=1` | Enables introspection functions such as `flameGraph` |

If the flame graph shows mostly addresses rather than function names, Linux will generally attempt to resolve instruction addresses to symbols using ELF debug information. Install a debug-symbol package that matches the server binary, or build locally with debug information, to improve symbol resolution.
