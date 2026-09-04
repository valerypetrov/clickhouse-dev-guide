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

## 2. The AST fuzzer failure is not this PR's

What the bot links: `Not-ready Set is passed as the second argument for function 'A (STID: 0250-4e52)` → issue #117806. That link is by normalized message only (the STID replaces every identifier with `A`, so every `Not-ready Set` error shares it). Issue #117806 was an object-storage `_path GLOBAL IN` query, closed as a duplicate fixed by PR #112968 (merged Sep 3, after this branch's last merge of master on Sep 2). The failure on this PR is a different shape:

```
Logical error: 'Not-ready Set is passed as the second argument for function 'in''.
SELECT count() IGNORE NULLS FROM merge('default', '.+')
PREWHERE exists((SELECT tuple('') WINDOW w_fuzz_2659 AS ()))
WHERE hasAll(['(', '4012', '104\0577', NULL], val) GROUP BY ... LIMIT -378
SETTINGS query_plan_direct_read_from_text_index = 0
```

raised in `FunctionIn::executeImpl` from `MergeTreeRangeReader::executePrewhereActionsAndFilterColumns`: a PREWHERE predicate with a subquery-backed set is executed by the MergeTree reader before that set was built. Evidence that it is a pre-existing master bug (public CI database, `checks` table on play.clickhouse.com):

| When (UTC) | Where | Failing shape |
|---|---|---|
| 2026-09-04 01:49 | PR 117781 (`fix-callback-runner-stuck-on-schedule-failure`, unrelated) | `default.t_neg PREWHERE in(1, (SELECT ... FROM system.one LIMIT 1))` |
| 2026-09-03 21:18 | this PR, head `e1ae54c` | `merge('default', '.+') PREWHERE exists((SELECT ...))` |
| 2026-09-01 10:59 | PR 117320 (`fix-statistics-part-pruning-pending-alter-mutation`) | `merge('.+') PREWHERE NOT (exists((SELECT ...)))` |
| 2026-09-01 19:17 | PR 113941 (`fix-merge-over-gap-with-mutated-part`) | `merge('.+.+.+.+') PREWHERE globalIn(database, (SELECT DISTINCT currentUser()))` |
| 2026-08-16 21:02 | master (`b8f2919eab`), AST fuzzer | same error, `merge(...)` |

The PR touches only `src/Storages/TimeSeries/*`, `src/Storages/StoragePrometheusQuery.cpp`, `src/Storages/StorageTimeSeriesSelector.cpp`, `src/Storages/StorageDistributed.h` (two accessors made public), `src/Server/PrometheusRequestHandler.cpp`, `src/TableFunctions/TableFunctionTimeSeries.cpp` and tests: nothing in PREWHERE, prepared sets, `StorageMerge` or the MergeTree reader. PR 117781's failing commit predates #112968 as well, so that fix has not been shown to cover the PREWHERE shape either way; the shape is on the MergeTree PREWHERE path, which #112968 does not touch.

Candidate minimal repro against master (untested here):

```sql
CREATE TABLE t_neg (A UInt64) ENGINE = MergeTree ORDER BY A;
INSERT INTO t_neg SELECT number FROM numbers(10);
SELECT count() FROM t_neg PREWHERE in(1, (SELECT 1 FROM system.one LIMIT 1));
SELECT count() FROM merge(currentDatabase(), '^t_neg$') PREWHERE exists((SELECT 1));
```

Suggested stand-down comment for the PR:

> The `AST fuzzer (amd_debug)` failure (`Not-ready Set is passed as the second argument for function 'in'`) is a PREWHERE-with-subquery failure over `merge('default', '.+')`, raised in `MergeTreeRangeReader` — code this PR does not touch (it only changes the Prometheus/TimeSeries path and its tests). The same shape fails on unrelated branches and on master: PR #117781 (`PREWHERE in(1, (SELECT ...))` on a MergeTree table, 2026-09-04), PR #117320 and PR #113941 (`merge(...) PREWHERE exists/globalIn(...)`, 2026-09-01), master AST fuzzer run `b8f2919eab` (2026-08-16). The bot's link to #117806 is by normalized message only; that issue was an object-storage `_path GLOBAL IN` shape fixed by #112968, a different code path. I'll merge master again once a fix for the PREWHERE shape lands.

## 3. What is in this directory and how to use it

- `0001-Check-the-local-shard-s-own-grants-before-the-promet.patch`: the fix, on top of `e1ae54c` (`git am` it onto `promql-over-distributed`). Files: `src/Storages/TimeSeries/resolvePrometheusQueryTarget.{cpp,h}`, `docs/concepts/features/interfaces/prometheus.mdx`, `tests/integration/test_prometheus_protocols/test_local_shard_distributed.py`.

```sh
git checkout promql-over-distributed
git am PRs/117170/0001-Check-the-local-shard-s-own-grants-before-the-promet.patch
```

Run the tests that cover the change:

```sh
cd tests/integration
pytest -s test_prometheus_protocols/test_local_shard_distributed.py test_prometheus_protocols/test_access_distributed.py
```

Expected: the four new HTTP cases and the two `prom_no_shard_select` table-function cases fail on `e1ae54c` with the probe's `UNEXPECTED_TABLE_ENGINE` text and pass with the patch; everything else is unchanged.

### What was verified, and what was not

- The in-process path and its order of checks were traced twice independently (my reading and a separate agent's): `getStructureOfRemoteTable.cpp` (first local shard, caller's context, no `prefer_localhost_replica` involvement), `StorageTimeSeriesSelector::getConfiguration` (`resolveStorageID`, `SELECT`, policy, engine cast), `ITableFunction::execute` (`CREATE TEMPORARY TABLE`; `timeSeriesSelector` is not `allow_readonly`, `cluster()` and `view()` are), `Cluster.cpp` (`is_local` needs no `<default_database>`, this server's `tcp_port`, a local interface).
- Every identifier used by the patch was checked against the headers in the checkout: `StorageDistributed::getCluster() const`, `Cluster::getShardsInfo()`, `Cluster::ShardInfo::isLocal() const` (no overload), `Context::resolveStorageID(StorageID, StorageNamespace) const`, `Context::checkAccess(const AccessFlags &, const StorageID &)`, `AccessType::CREATE_TEMPORARY_TABLE`; `std::ranges::any_of` is already used in `src/Storages`. No variable named `cluster` exists in the function before the change.
- The expected messages come from `src/Access/ContextAccess.cpp` (`<user>: Not enough privileges. To execute this query, it's necessary to have the grant <grant>`) and from `checkShardTargets`; the HTTP handler returns `Exception::message()` in the `error` field, which the test helper extracts verbatim.
- The test file is `black`-formatted (26.5.1) and parses; the users are created by SQL in the module fixture, the idiom `test_row_policy_distributed.py` already uses.
- Not done: no compilation or test run was possible here (no contrib submodules, no docker), and the adversarial review agents were cut off by an API rate limit before reporting. Treat CI as the final check.

Note on `resolveStorageID`: the selector resolves the shard-local name the same way (temporary tables first for an undeclared database, then the current database, `UNKNOWN_DATABASE` for a declared database that does not exist here), so the pre-check produces exactly the error the in-process path would have produced for the same caller, only before the probe.
