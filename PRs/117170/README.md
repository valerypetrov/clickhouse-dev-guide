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

### Round 2: the follow-up review on a7a3a25

The bot's next review accepted the grant pre-check but blocked on consistency: three places must agree on which shard is read in-process on the caller's context, and they did not.

- `SelectStreamFactory` takes the local plan only with `prefer_localhost_replica = 1` and task-based parallel replicas off; the Distributed sink writes a local shard in-process only with `prefer_localhost_replica = 1`.
- The shard probe resolved an undeclared database as the caller's current database for a local replica when `prefer_localhost_replica` was on, ignoring parallel replicas; the grant pre-check fired for any local shard regardless of either setting.
- So a caller bringing `prefer_localhost_replica = 0`, or parallel replicas on a multi-replica shard, had the probe validate one table (`metrics.ts_local`) while the shard read another (`ts_local` in the connection's default database), and was asked for grants the connection path never uses.

The reviewer offered two fixes: share the exact "local plan under the caller's context" predicate, or disable those modes for the generated read. The second is a few lines and removes the disagreement instead of tracking it, so that is what the follow-up commit does:

- Both read executors (the HTTP endpoints and the `prometheusQuery` table functions) pin `prefer_localhost_replica = 1` and `enable_parallel_replicas = 0` with `Context::setSetting` on the context that executes the generated SQL, next to the `enable_materialized_cte` they already force. A first draft put them in the generated `SETTINGS` clause instead; a verifier showed that a nested clause is clamped against the caller's settings constraints and silently dropped by a read-only constraint, whereas `setSetting`, which the write path already uses, bypasses constraints. Context copies carry the pinned values down to the view's inner query, so `SelectStreamFactory` and `ClusterProxy::executeQuery` see them.
- The remote-write context pins `prefer_localhost_replica = 1` next to the delivery settings it already forces, so the sink writes a local shard in-process too.
- The probe treats a local replica as the caller's own context unconditionally (`runs_on_the_caller = address.is_local`), and the `Setting::prefer_localhost_replica` extern goes away.
- Docs: one parenthetical in the sentence that already stated the in-process semantics for a local shard.
- Tests: a second cluster with the same local replica listed twice (a shard parallel replicas could spread over), a wrapper over it, and one test over three explicit cases (each wrapper with `prefer_localhost_replica = 0`, and the two-replica wrapper with `enable_parallel_replicas = 1, max_parallel_replicas = 2`; the one-replica wrapper never fans out under parallel replicas, so that cell would prove nothing) that reads through the table function and the HTTP endpoint, expecting the caller's own database either way (the sample is found; over the connection the shard would hit the MergeTree decoy in `default` and fail), and the grant refusal still first.

The bot's comment on the PR still shows the review of a7a3a25 (it says "measured on commit a7a3a25") because CI had not yet started on 3c20def96 when this was written; the blocker text above is that review, not a new one. A third, comment-only commit (`687057e62`, patch 0003) names the pin at the two places in `resolvePrometheusQueryTarget.cpp` that rely on it (the probe's database resolution and the grant pre-check), so the guarantee is visible where it is assumed rather than only in the executors.

On a7a3a25, CI came back green for everything that matters: `AST fuzzer (amd_debug)` passed (the earlier failure was the unrelated master bug from section 2), `Code Review`, `Style check` and `Fast test` passed; the only red jobs were the two `Docker ... image` builds for arm64, which do not touch this code.

Why pinning is safe: the PR's docs already defined a local shard as "resolves it in the requesting user's current database, exactly as any query here would", which is the in-process behaviour; the pin makes that definition hold whatever the caller's fan-out settings say, exactly as the write path already does for `distributed_foreground_insert`, `async_insert` and `skip_unavailable_shards`. The main table of the generated read is the `view()` storage, not a replicated table, so the stale-replica fallback that `prefer_localhost_replica = 0` is normally used for does not apply.

### Master merged (conflicts resolved)

Master moved on since the branch's last merge of Sep 2, and two files conflicted, both because the user's own PR #117816 (merged Sep 3) made the two PromQL executors run the generated SQL on a fresh `query_context` copy with `empty_result_for_aggregation_by_empty_set = 0`, exactly where the round-2 pin sits:

- `src/Storages/TimeSeries/PrometheusHTTPProtocolAPI.cpp`: kept master's `query_context` copy and moved the pin onto it (it had been on `getContext()`), so `executeQuery` runs on the pinned copy.
- `src/Storages/StoragePrometheusQuery.cpp`: kept master's copy and the new setting, then the pin block, unchanged.

The merge commit is `f49502b2411` ("Merge remote-tracking branch 'origin/master' into promql-over-distributed", the author's own convention on this branch). Master's other changes to the Prometheus files (remote-write timeout clamping and 503 handling, new PromQL functions, instant-vector timestamp formatting) auto-merged; the test helpers the new tests call kept their signatures.

### Simplification pass (behavior unchanged)

A review of the PR's whole diff for dead code, duplication and over-long comments, with three reviewers (resolver, executors, tests) proposing and one adversarial reviewer verifying. What changed (commit `3000468a377`, patch 0004):

- `resolvePrometheusQueryTarget`: the wrapper's declared shard-skip settings are now two fields of the resolved target instead of a separate accessor and two `std::tie` sites; the probe iterates shards zipped with their addresses (every cluster reachable here is built in lockstep, so the index loops and their bounds guard were dead); the local-shard pre-check uses `getLocalShardCount()`; an empty-block check the remote executor can never satisfy is gone; `hasInstantSelector` is one expression.
- Executors: the `is_distributed_target` flag is replaced by the `cluster_name` test the sibling path already used; the HTTP handler gets one `getTimeSeriesTable(required_access)` helper for its three "grant before existence, then the table" sites.
- Docs: the `prometheusQuery`/`prometheusQueryRange` argument docs now say a Distributed wrapper is accepted; the docs parenthetical no longer claims the write pins parallel replicas off.
- Tests: `error_code`, `keyed_result` and the instant/range endpoint switch live once in `prometheus_test_utils.py`; the write tests build every batch through one `write()` helper and count with a `flush` switch; all seven modules are black-formatted (two were not); every comment and docstring the branch added is at most two lines (five module docstrings, one XML comment and the stateless test's tag block were longer).

What was deliberately kept: the per-class grant re-checks (HTTP API constructor, metadata endpoints, remote read and write constructors, the selector's read, the shared read check's own wrapper `SELECT`). They are reachable code, added on purpose across the earlier review rounds as each class's own access contract; removing them would keep today's behavior only as long as every construction site keeps checking first, which is a security-posture change rather than a simplification. Also kept: test cells a reviewer judged redundant (they cover code paths the PR does not own, cheaply) and the retry-based assertions (dropping them trades stability for lines).

Verification: the repo's C++ style script and `ruff` pass, `black` passes on the seven PR modules, a clang-format dry run over the changed lines proposes only the pre-existing include order of the handler, the comment audit finds no added block over two lines, and the adversarial reviewer confirmed identical behavior (including that a zero-row block cannot reach the probe loop and that no assertion depends on the sample values the write helper changes).

### Master merged again (TimeSeries versioning)

Master (5cbca3e, Sep 5 20:45 UTC) added schema versioning to the TimeSeries engine with guards at the PromQL and remote-write entry points; four files conflicted because those guards take the TimeSeries table itself while the PR's entry points hold a storage that may be a Distributed wrapper. Resolution (merge `ad5ec290192`): the initiator runs `checkTimeSeriesVersionSupportedByPromQL` / `checkTimeSeriesVersionIsWritable` only when the target is a TimeSeries table (right after `resolvePrometheusQueryTarget` says so); over a wrapper the shards apply them, in the selector each shard runs and in `StorageTimeSeries::write`, which master also guards. One auto-merged call in the table function's read path would have cast the wrapper and thrown, so it is gated on the target's cluster name. The two files the PR had switched to the generic storage header include `StorageTimeSeries.h` again for the cast idiom master uses.

### Round 3: the check-then-insert race in remote write

The bot's next review (on `ad5ec290192`) blocked on `PrometheusRemoteWriteProtocol::write`: `checkPrometheusQueryDistributedWrite` is a preflight on the request thread, and once it returns another session can `EXCHANGE` a same-schema non-TimeSeries table under a shard-local name before `insertBlock` runs; the ordinary Distributed sink then accepts the batch into that table and the write answers `204`. It asked for the engine/type validation to move into the delivery path ("the same place that chooses the shard destination"), or for the validated identity to be synchronized with shard-local DDL before the INSERT is dispatched, plus a test that pauses the write between the two, swaps a target, and asserts the write is rejected rather than acknowledged.

Two commits answer it; the second supersedes the first, and both are on the PR.

**First attempt, `e5fe445b401` (patch 0006): a post-delivery witness.** The probe returned one identity per replica (answered, engine, `time_series` type, table UUID); after the INSERT the write probed again and compared, withholding the acknowledgement with `UNKNOWN_STATUS_OF_INSERT` (a 500) on any difference. A `DDLGuard` on the local shard's name was tried and dropped because it is an exclusive per-name mutex that would serialize every concurrent write on a shard node. Two things were wrong with the witness as an answer to the review: the rows had already landed in the swapped-in table when the write was refused, and a table swapped in and back within one request was not seen. The user asked for the fix "as it's said in requested changes", i.e. in the delivery path.

**The fix, `4dbe1d113ec` (patch 0007): the shard's own INSERT checks the table it resolves.** A new query setting, `insert_expected_table_engine` (String, default empty; `src/Core/Settings.cpp`, recorded under 26.9 in `SettingsChangesHistory.cpp`):

- `InterpreterInsertQuery::execute` checks it right after the INSERT grant and `checkInsertIsAllowed`: the table the INSERT names must have that engine (`IStorage::getName()`), else `UNEXPECTED_TABLE_ENGINE` ("Table X has engine MergeTree while the INSERT expects TimeSeries"). A `Distributed` table is exempt: it forwards the setting to its shards as any query setting, over the connection (`RemoteInserter` sends the changed settings) and on the context copy the sink writes a local shard with in-process. Once a table passes, the interpreter clears the setting on its own context, so the writes that table makes on its own, the TimeSeries inner-table inserts of `TimeSeriesSink`, materialized views, a nested Distributed hop, are not checked. Placing the check after the access check means a caller without INSERT is not told the engine.
- `DistributedSink::writeToLocal` (the background path's in-process write) gives the interpreter a context copy, as the foreground job already does, so the consumption cannot empty the setting on the sink's shared context before the queue headers for the remote shards are written.
- `PrometheusRemoteWriteProtocol` pins `insert_expected_table_engine = 'TimeSeries'` next to its other pins when the target is Distributed. The preflight probe stays as it was (engine and type on every replica, unavailable replicas refused), so the messages and the earlier tests are unchanged; the witness code and the UUID column are gone. The pausable failpoint `prometheus_remote_write_before_insert` between the check and the INSERT stays for the tests.
- Exactness: the check runs on the storage object the INSERT writes, so there is no window at all for the engine, on remote and local shards alike, and nothing lands in a swapped-in table; the refusal is a 500, so Prometheus resends and the next preflight refuses too. Residual, stated in the docs and the commit: a shard-local TimeSeries table of another `time_series` type swapped in that late is not refused by the shard (the samples land in a TimeSeries table, rounded), and a shard whose user profile pins the setting cannot check.
- Tests: `05099_insert_expected_table_engine.sql` covers the named table (`Memory`, robust under the SharedMergeTree CI run), a view's Log target reached directly and through a Distributed table (must not be checked), and a Distributed table over a local shard written in-process and, with `prefer_localhost_replica = 0`, over the connection (both refused). The two integration tests keep the pause / EXCHANGE / NOTIFY choreography and now assert `UNEXPECTED_TABLE_ENGINE` with `engine MergeTree` and `expects TimeSeries` in the body, that the swapped-in MergeTree table holds nothing, and that a retry after swapping back lands. `h3` routes to shard 0 (a helper agent re-derived CityHash64's short-string path to confirm).

The scoping rule went through two designs: a first draft exempted `isRemote()` tables and checked only "the query context itself or a raised distributed depth"; the adversarial reviewer showed that `isRemote()` also covers views and Buffer tables onto remote targets (a swapped-in view onto a Distributed table would have been skipped) and that the setting leaked into nested Distributed hops (a TimeSeries table with a Distributed inner table, or a view onto a Distributed table, would have had the far shard refuse a legitimate write). Consuming the setting once checked, and exempting only `Distributed`, replaced the heuristic; the reviewer then traced the initiator's INSERT, both sink paths, the shard's top-level INSERT, `TimeSeriesSink`, the views builder, the async insert queue (checked at flush), Buffer flushes and the stateless test row by row, and confirmed the final tree clean.

Verification: the repo's C++ style script passes; a clang-format dry run proposes only a pre-existing line in restored code; the comment audit finds no added block over two lines; `black` and `ruff` pass on the two test modules; CI's `Fast test` and `Style check` were green on `e5fe445b401`, which shares the failpoint and the test scaffolding. No build here: the stateless test's reference (`1 3` for the Memory table, `1 1 3 3` for the Log table) was traced by hand and by the reviewer, not run. Master at `319bfd73ff3` merged cleanly; nothing to resolve.

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

All seven patches below were pushed to the PR branch `valerypetrov/ClickHouse:promql-over-distributed` (`a7a3a2501`, the round-2 follow-up `3c20def968ed6690c862b811f2b276e32fb6643e`, the comment-only `687057e62`, after the master merge `f49502b` the simplification `3000468a377` plus its assertion-message follow-up, and after the second merge `ad5ec290192` the round-3 witness `e5fe445b401` and the fix that supersedes it, `4dbe1d113ec`), so PR 117170 already carries them; the files are kept here as the record. The stand-down comment in section 2 still has to be posted on the PR by hand.

- `0001-Check-the-local-shard-s-own-grants-before-the-promet.patch`: the grant pre-check, on top of `e1ae54c`. Files: `src/Storages/TimeSeries/resolvePrometheusQueryTarget.{cpp,h}`, `docs/concepts/features/interfaces/prometheus.mdx`, `tests/integration/test_prometheus_protocols/test_local_shard_distributed.py`.
- `0003-Name-the-pin-where-the-shard-probe-and-the-grant-pre.patch`: comment-only; names the pin at the two assumption points in `resolvePrometheusQueryTarget.cpp`.
- `0004-Simplify-the-distributed-prometheus-code-and-tests-w.patch`: the behavior-preserving simplification pass (on top of the master merge `f49502b`).
- `0005-Keep-the-response-body-in-the-remote-write-success-a.patch`: keeps the server's response text in the two remote-write success assertions (a diagnostic the shared helper had dropped).
- `0006-Acknowledge-a-distributed-remote-write-only-once-eve.patch`: the round-3 witness (on top of the second master merge `ad5ec290192`), superseded by 0007 but part of the PR's history. Files: `src/Common/FailPoint.cpp`, `src/Storages/TimeSeries/PrometheusRemoteWriteProtocol.cpp`, `src/Storages/TimeSeries/resolvePrometheusQueryTarget.{cpp,h}`, the docs, `tests/integration/test_prometheus_protocols/test_write_distributed.py` and `test_local_shard_distributed.py`.
- `0007-Refuse-a-distributed-remote-write-on-the-shard-whose.patch`: the round-3 fix, the `insert_expected_table_engine` setting checked in the delivery path. Files: `src/Core/Settings.cpp`, `src/Core/SettingsChangesHistory.cpp`, `src/Interpreters/InterpreterInsertQuery.cpp`, `src/Storages/Distributed/DistributedSink.cpp`, `src/Storages/TimeSeries/PrometheusRemoteWriteProtocol.cpp`, `src/Storages/TimeSeries/resolvePrometheusQueryTarget.{cpp,h}` (restored to their pre-witness state), the docs, `tests/queries/0_stateless/05099_insert_expected_table_engine.{sql,reference}` and the two integration modules.
- `0002-Pin-a-shard-that-is-this-server-itself-to-the-in-pro.patch`: the round-2 pin, on top of the first. Files: `src/Storages/TimeSeries/PrometheusHTTPProtocolAPI.cpp`, `src/Storages/StoragePrometheusQuery.cpp`, `src/Storages/TimeSeries/PrometheusRemoteWriteProtocol.cpp`, `src/Storages/TimeSeries/resolvePrometheusQueryTarget.{cpp,h}`, the docs, `tests/integration/test_prometheus_protocols/configs/config.d/local_shard_dist.xml` and the test module.

```sh
git checkout promql-over-distributed
git am PRs/117170/000*.patch
```

Run the tests that cover the change:

```sh
cd tests/integration
pytest -s test_prometheus_protocols/test_local_shard_distributed.py test_prometheus_protocols/test_access_distributed.py test_prometheus_protocols/test_write_distributed.py
```

And the stateless test of the setting:

```sh
tests/clickhouse-test 05099_insert_expected_table_engine
```

Expected: the four new HTTP cases and the two `prom_no_shard_select` table-function cases fail on `e1ae54c` with the probe's `UNEXPECTED_TABLE_ENGINE` text and pass with the patch; everything else is unchanged.

### What was verified, and what was not

- The in-process path and its order of checks were traced twice independently (my reading and a separate agent's): `getStructureOfRemoteTable.cpp` (first local shard, caller's context, no `prefer_localhost_replica` involvement), `StorageTimeSeriesSelector::getConfiguration` (`resolveStorageID`, `SELECT`, policy, engine cast), `ITableFunction::execute` (`CREATE TEMPORARY TABLE`; `timeSeriesSelector` is not `allow_readonly`, `cluster()` and `view()` are), `Cluster.cpp` (`is_local` needs no `<default_database>`, this server's `tcp_port`, a local interface).
- Every identifier used by the patch was checked against the headers in the checkout: `StorageDistributed::getCluster() const`, `Cluster::getShardsInfo()`, `Cluster::ShardInfo::isLocal() const` (no overload), `Context::resolveStorageID(StorageID, StorageNamespace) const`, `Context::checkAccess(const AccessFlags &, const StorageID &)`, `AccessType::CREATE_TEMPORARY_TABLE`; `std::ranges::any_of` is already used in `src/Storages`. No variable named `cluster` exists in the function before the change.
- The expected messages come from `src/Access/ContextAccess.cpp` (`<user>: Not enough privileges. To execute this query, it's necessary to have the grant <grant>`) and from `checkShardTargets`; the HTTP handler returns `Exception::message()` in the `error` field, which the test helper extracts verbatim.
- The test file is `black`-formatted (26.5.1) and parses; the users are created by SQL in the module fixture, the idiom `test_row_policy_distributed.py` already uses.
- Not done: no compilation or test run was possible here (no contrib submodules, no docker), and the adversarial review agents were cut off by an API rate limit before reporting. Treat CI as the final check.

Note on `resolveStorageID`: the selector resolves the shard-local name the same way (temporary tables first for an undeclared database, then the current database, `UNKNOWN_DATABASE` for a declared database that does not exist here), so the pre-check produces exactly the error the in-process path would have produced for the same caller, only before the probe.
