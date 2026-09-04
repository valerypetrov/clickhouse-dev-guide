# PR 117170: what the clickhouse-gh bot reported, and the fix

PR: https://github.com/ClickHouse/ClickHouse/pull/117170 ("Support PromQL over a Distributed table of TimeSeries shards"), head commit `e1ae54c7ddb0e6bde21bc8b5f22f694eed44cb8d`.

The bot's workflow summary on that head reports two things:

| Item | Bot verdict | What it is | Action |
|---|---|---|---|
| `AST fuzzer (amd_debug)` | FAIL: `Logical error: Not-ready Set is passed as the second argument for function 'A (STID: 0250-4e52)`, linked to issue #117806 | A master-side fuzzer bug in PREWHERE with an `IN`/`EXISTS` subquery over `merge()` / MergeTree. Not this PR's. | No code change. Post the stand-down note in section 3. |
| AI Review | Changes requested, one blocker at `src/Storages/TimeSeries/PrometheusHTTPProtocolAPI.cpp:253` | An access-control ordering hole: the shard probe describes the shard-local table before the grants an in-process local shard enforces later. | Fixed by the patch in this directory. |

All inline review threads by the bot on the PR files page are marked resolved or outdated; the workflow summary above is the current state. All other CI jobs on the head commit are green (`Code Review`, `Style check`, `Docs examples`, builds, integration and stateless suites); the earlier `Docs examples` and `Style check` failures were on older commits and are already fixed on the branch.

## 1. The blocker

### What the bot said

> `executePromQLQuery` calls `checkPrometheusQueryDistributedRead` before the generated `cluster(view(timeSeriesSelector(...)))` read has built any local-shard plan. On a genuinely local shard (`prefer_localhost_replica` path, e.g. `local_shard_dist`), `SelectStreamFactory` executes the `view(...)` table function locally; `TableFunctionView` then analyzes the nested query under the caller's context, and only then does the nested `timeSeriesSelector` enforce its own grants (`CREATE TEMPORARY TABLE` and `SELECT` on the shard-local source table). A caller with wrapper `SELECT` (and `READ ON REMOTE`) but lacking one of those downstream grants can still receive `UNEXPECTED_TABLE_ENGINE` / `TYPE_MISMATCH` / missing-table specifics from `checkShardTargets` instead of a plain access error.

### Where the leak actually is

The bot's description is right, and the in-process path starts even earlier than the local plan. The generated read is

```sql
SELECT timeSeriesTagsToGroup(tags) AS group, timestamp, value
FROM view(SELECT tags, timestamp, value
          FROM cluster('<cluster>', view(SELECT timeSeriesIdToTags(id) AS tags, timestamp, value
                                        FROM timeSeriesSelector([<db>,] '<table>', '<selector>', t0, t1))
               SETTINGS skip_unavailable_shards = ..., skip_unavailable_shards_mode = ...)
     SETTINGS serialize_query_plan = 0)
```

When the analyzer resolves the `cluster()` call on the initiator it needs the structure of its `view(...)` argument. `getStructureOfRemoteTable` takes the first shard with `ShardInfo::isLocal()` and, for a table function, resolves it in-process on the caller's context (`src/Storages/getStructureOfRemoteTable.cpp`). That analyzes the nested `SELECT ... FROM timeSeriesSelector(...)`, and the selector then enforces, in this order:

1. `StorageTimeSeriesSelector::getConfiguration`: `resolveStorageID` (an undeclared database becomes the caller's current database), `checkAccess(SELECT, <shard-local table>)`, the row-policy/filter refusal, the engine cast.
2. `ITableFunction::execute`: `checkAccess(CREATE_TEMPORARY_TABLE)`, because `timeSeriesSelector` is not registered `allow_readonly` (`cluster()` and `view()` are, which is why the HTTP endpoints need the grant only when a shard is this server itself).

This happens whenever the cluster has a local shard, regardless of `prefer_localhost_replica`; that setting only decides the later, execution-time local plan in `SelectStreamFactory`. A replica is local when it declares no `<default_database>`, uses this server's own `tcp_port`, and resolves to a local interface (`src/Interpreters/Cluster.cpp`).

`checkPrometheusQueryDistributedRead` ran the shard probe (on the server's own context) before any of that, so a caller holding `SELECT` on the wrapper and `READ ON REMOTE` but lacking `SELECT` on the shard-local table or `CREATE TEMPORARY TABLE` was told the shard-local engine or `time_series` type instead of being denied.

### The fix

Mirror those two grants before the probe, in the shared check so both the HTTP endpoints and the `prometheusQuery`/`prometheusQueryRange` table functions get it:

```cpp
    /// A shard that is this server itself is read in-process, on the caller's context: its selector's grants too.
    const auto cluster = typeid_cast<const StorageDistributed &>(storage).getCluster();
    if (std::ranges::any_of(cluster->getShardsInfo(), &Cluster::ShardInfo::isLocal))
    {
        context->checkAccess(AccessType::SELECT, context->resolveStorageID(target->remote_time_series_storage_id));
        context->checkAccess(AccessType::CREATE_TEMPORARY_TABLE);
    }
```

Why this shape:

- The condition is "any shard is local", the exact trigger of the in-process structure fetch. Gating on `prefer_localhost_replica` would leave the leak open when that setting is off; the condition never over-refuses, because the analysis demands the same two grants anyway.
- `resolveStorageID` is the selector's own resolution, so the `SELECT` check names the same table it would (`default.ts_local` for a caller whose current database is `default` and a wrapper declaring no database).
- `CREATE TEMPORARY TABLE` stays conditional on a local shard: a fully remote cluster never asks for it (commit db39212's rationale). The table-function path keeps its own unconditional check; the shared one is redundant there and harmless.
- The order `SELECT`, then `CREATE TEMPORARY TABLE`, is the in-process order.

The tests run on the `local_shard_dist` cluster of `test_local_shard_distributed.py`, where the wrapper `metrics.prom_local` declares no database and `default.ts_local` is a MergeTree table of the same outer schema. For a caller whose current database is `default`, the probe would say `1 shard-local target(s) named ts_local are not TimeSeries tables`; the tests prove that a fully granted caller is told exactly that (the leak is present to be leaked), and that a caller missing one grant is told only `Not enough privileges ... CREATE TEMPORARY TABLE ON *.*` or `... SELECT ON default.ts_local`, with none of the probe's wording, on `/query`, `/query_range`, `prometheusQuery` and `prometheusQueryRange`.

### Not changed: the write path

The remote-write probe (`checkPrometheusQueryDistributedWrite`) has a smaller analogue: with `prefer_localhost_replica` on, `DistributedSink` inserts into a local shard through `InterpreterInsertQuery` on the caller's context, which enforces `INSERT` on the shard-local table (with the column list) after the probe has spoken. The bot did not flag it, it is gated by a setting, and it needs the sink's column list to mirror column-level grants, so it is left for a follow-up rather than widening this change.

