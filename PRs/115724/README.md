# PR 115724: do not overwrite files on an S3Queue move destination collision

[PR 115724](https://github.com/ClickHouse/ClickHouse/pull/115724) stops silent data loss in `S3Queue`/`AzureQueue` with
`after_processing = 'move'`: with the default `preserve_path = 0`, objects sharing a basename under different prefixes
mapped to the same destination key and overwrote each other. It de-duplicates within a batch, guards across batches with
an `If-None-Match` precondition, leaves a colliding object at the source, and counts it in the new
`ObjectStorageQueueMoveCollisions` profile event.

## 1. The review's remaining ask

> `test_move_retry_recognizes_committed_copy` still only proves that the destination exists and the source disappears
> after the failpoint-triggered retry. It never snapshots `move_collisions(node)` or checks that the retry kept
> `ObjectStorageQueueMoveCollisions` unchanged, so the test is still weaker than the contract claimed in the thread.

The point is sharp. The failpoint `object_storage_queue_fail_after_move_copy` interrupts a move *after* its guarded copy
has committed, so the retry meets a destination that already exists. The contract is that it recognizes that object as
its own committed copy and finishes the move. The end state the test asserted, destination present and source gone, is
also what a run would show if the retry had instead mistaken its own copy for another object's, counted a collision and
left the source in place until some later attempt removed it. The counter is the only thing that separates the two.

The test now snapshots the event before the interrupted move and asserts it is unchanged once the retry has removed the
source (commit `901217b8cf0`, patch 0001). It is parametrized over `S3Queue` and `AzureQueue` and over an external
destination, so all four cases carry the assertion, which is what the review asked for both engines.

Snapshot-and-compare rather than an absolute value, because the event is a node-wide counter that earlier tests in the
module may already have moved. The assertion sits after the `finally` that disables the failpoint, once the object
counts have settled, so nothing else can still be in flight.

## 2. Master merged (conflicts resolved)

The branch was 3537 commits behind, with five conflicting files (merge `93fb2bbc841`). Two changes in master caused all
of them:

- `copyAzureBlobStorageFile` lost its source-offset parameter. The native copy (`CopyFromUri` / `StartCopyFromUri`)
  carries no byte range and always transfers the whole blob, so a range argument could only ever be honored by the
  read-write fallback. This branch's calls passed `0` for it and now simply do not pass it; the `dest_if_none_match`
  parameter the branch adds sits after the ones master kept, and the declaration itself auto-merged.
- S3 objects are now addressed by a bucket split out of the remote path (`splitBucketAndKey`, which falls back to the
  storage's own bucket), rather than by `uri.bucket` plus the path. The guarded copy's own additions, the source's
  headers for the precondition and its tags read explicitly, now use that split bucket and key.

The header conflict was only over a doc comment; master's, which explains why there is no range argument, is kept
alongside the branch's `isAzureDestinationAlreadyExistsError` declaration.

Verified: the repo's C++ style script passes on the merged tree, `ruff` passes on the test module, and no call site of
`copyAzureBlobStorageFile` still passes an offset. Not verified: no build or test run was possible here, so CI is the
check. Note that `black` in this environment reformats this test module wholesale, so the test edit was applied without
it; the added lines follow the file's existing formatting.

## 3. What is in this directory

- `0001-Prove-the-interrupted-move-s-retry-reports-no-collis.patch`: the counter assertion, on top of the master merge.

Both the merge and the patch are pushed to `valerypetrov/ClickHouse:fix/s3queue-move-collision-114847`, so the PR
already carries them.
