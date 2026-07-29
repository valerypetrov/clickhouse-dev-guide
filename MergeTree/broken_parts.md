### Causes of Broken Parts
When the server or a table is restarted, ClickHouse loads its parts and performs safety checks on them. A part is treated as valid only if it passes these checks.
Parts that fail the safety checks are added to `unexpected_data_parts`.

#### How Parts Are Added to `unexpected_data_parts`
The main entry point is `MergeTreeData::loadDataParts`.

The following function loads a part from disk into memory. Note that it loads only metadata:
MergeTreeData::loadDataPart
`loadDataPart` primarily loads two groups of metadata:

- `MergeTreeDataPartBuilder::build()` constructs the basic metadata, such as whether the part uses the Compact or Wide format based on `part_type`.
- `IMergeTreeDataPart::loadColumnsChecksumsIndexes` loads column metadata, index metadata, checksums, and so on.
If an abnormal event occurred earlier, such as an abrupt shutdown, it may have produced broken parts. For write performance, ClickHouse does not use `fsync` for disk writes by default; it can be enabled with [fsync_after_insert](https://clickhouse.com/docs/operations/settings/merge-tree-settings#fsync_after_insert).
When the service subsequently restarts and loads the table's parts, the broken parts are likely to be detected during the second step above.

If an exception occurs during loading, `is_broken` is set to `true` for that load result:
```
    /// Ignore broken parts that can appear as a result of hard server restart.
    auto mark_broken = [&]
    {
        if (!res.part)
        {
            /// Build a fake part and mark it as broken in case of filesystem error.
            /// If the error impacts part directory instead of single files,
            /// an exception will be thrown during detach and silently ignored.
            res.part = getDataPartBuilder(part_name, single_disk_volume, part_name, getReadSettings())
                .withPartStorageType(MergeTreeDataPartStorageType::Full)
                .withPartType(MergeTreeDataPartType::Wide)
                .build();
        }

        res.is_broken = true;
        tryLogCurrentException(log, fmt::format("while loading part {} on path {}", part_name, part_path));

        res.size_of_part = calculatePartSizeSafe(res.part, log.load());
        auto part_size_str = res.size_of_part ? formatReadableSizeWithBinarySuffix(*res.size_of_part) : "failed to calculate size";

        LOG_ERROR(log,
            "Detaching broken part {} (size: {}). "
            "If it happened after update, it is likely because of backward incompatibility. "
            "You need to resolve this manually",
            fs::path(getFullPathOnDisk(part_disk_ptr)) / part_name, part_size_str);
    };
```

For a replicated table:

(1) First, all parts under the `parts` znode are retrieved from the table replica path in Keeper. These entries contain only part metadata, not the data itself. All part names are then added to `expected_parts_on_this_replica`:
```
std::optional<std::unordered_set<std::string>> expected_parts_on_this_replica;
```
Parts are added under the `parts` path in Keeper in the following cases:
- Data is written to the table and the resulting part is committed to Keeper.
- A merged part is committed to Keeper.

(2) `loadDataParts` then examines all physical parts on the current node's disks. If a part is not in `expected_parts_on_this_replica`, it is added to `unexpected_disk_parts`. Expected parts are added to `parts_to_load_by_disk`:
```
    std::vector<PartLoadingTree::PartLoadingInfos> parts_to_load_by_disk(disks.size());
    std::vector<PartLoadingTree::PartLoadingInfos> unexpected_parts_to_load_by_disk(disks.size());

    ThreadPoolCallbackRunnerLocal<void> runner(getActivePartsLoadingThreadPool().get(), "ActiveParts");

    bool all_disks_are_readonly = true;
    for (size_t i = 0; i < disks.size(); ++i)
    {
        const auto & disk_ptr = disks[i];
        if (disk_ptr->isBroken())
            continue;
        if (!disk_ptr->isReadOnly())
            all_disks_are_readonly = false;

        auto & disk_parts = parts_to_load_by_disk[i];
        auto & unexpected_disk_parts = unexpected_parts_to_load_by_disk[i];

        runner([&expected_parts, &unexpected_disk_parts, &disk_parts, this, disk_ptr]()
        {
            for (auto it = disk_ptr->iterateDirectory(relative_data_path); it->isValid(); it->next())
            {
                /// Skip temporary directories, file 'format_version.txt' and directory 'detached'.
                if (startsWith(it->name(), "tmp")
                    || it->name() == MergeTreeData::FORMAT_VERSION_FILE_NAME
                    || it->name() == DETACHED_DIR_NAME)
                    continue;

                if (auto part_info = MergeTreePartInfo::tryParsePartName(it->name(), format_version))
                {
                    if (expected_parts && !expected_parts->contains(it->name()))
                        unexpected_disk_parts.emplace_back(*part_info, it->name(), disk_ptr);
                    else
                        disk_parts.emplace_back(*part_info, it->name(), disk_ptr);
                }
            }
        }, Priority{0});
    }
```
#### Local Parts Present in Keeper's `parts` Path
(3) Parts stored in `parts_to_load_by_disk` must be loaded into memory.

  - The first step is to build a loading tree with `PartLoadingTree::build`.
    Parts may cover one another (for example, one part may have been merged from several smaller parts), so a containment tree is needed to determine the loading order. If a part is broken, this structure allows its child parts—the original data covered by it—to be loaded instead, thereby recovering the data.

    Each part has the following attributes:
    - partition_id
    - min_block, max_block
    - level
    
    Multiple parts in the same partition may have the following relationship:
    ```
           A (covering part)
          / \
         B   C   (smaller parts)
    ```
    `PartLoadingTree` builds this kind of tree:
    - A is the root.
    - B and C are children of A.
    - If A is broken, B and C are loaded in its place.
   
(4) `traverse` adds the top-level parts in the tree to `active_parts`:
```
    PartLoadingTreeNodes active_parts;

    /// Collect only "the most covering" parts from the top level of the tree.
    loading_tree.traverse(/*recursive=*/ false, [&](const auto & node)
    {
        active_parts.emplace_back(node);
    });
```
(5) `loadDataPartsFromDisk` loads `active_parts` from disk. The resulting `loaded_parts` are then traversed. If a loaded part has `is_broken` set to `true`, the broken-part count and byte size are recorded. Parts that are not among those retrieved from Keeper are counted separately:
    ```
        if (res.is_broken)
    {
        broken_parts_to_detach.push_back(res.part);
        bool unexpected = expected_parts != std::nullopt && !expected_parts->contains(res.part->name);
        if (unexpected)
        {
            LOG_DEBUG(log, "loadDataParts: Part {} is broken, but it's not expected to be in parts set, "
                      " will not count it as suspicious broken part", res.part->name);
            ++suspicious_broken_unexpected_parts;
        }
        else
            ++suspicious_broken_parts;

        if (res.size_of_part)
        {
            if (unexpected)
                suspicious_broken_unexpected_parts_bytes += *res.size_of_part;
            else
                suspicious_broken_parts_bytes += *res.size_of_part;
        }
    }
    ```

(6) If the number of broken parts exceeds the MergeTree setting `max_suspicious_broken_parts`, or their total size exceeds `max_suspicious_broken_parts_bytes`, part loading fails and may prevent the server from starting.
```
if (!skip_sanity_checks)
{
    if (suspicious_broken_parts > (*settings)[MergeTreeSetting::max_suspicious_broken_parts])
        throw Exception(
            ErrorCodes::TOO_MANY_UNEXPECTED_DATA_PARTS,
            "Suspiciously many ({} parts, {} in total) broken parts "
            "to remove while maximum allowed broken parts count is {}. You can change the maximum value "
            "with merge tree setting 'max_suspicious_broken_parts' in <merge_tree> configuration section or in table settings in .sql file "
            "(don't forget to return setting back to default value)",
            suspicious_broken_parts,
            formatReadableSizeWithBinarySuffix(suspicious_broken_parts_bytes),
            (*settings)[MergeTreeSetting::max_suspicious_broken_parts].value);

    if (suspicious_broken_parts_bytes > (*settings)[MergeTreeSetting::max_suspicious_broken_parts_bytes])
        throw Exception(
            ErrorCodes::TOO_MANY_UNEXPECTED_DATA_PARTS,
            "Suspiciously big size ({} parts, {} in total) of all broken "
            "parts to remove while maximum allowed broken parts size is {}. "
            "You can change the maximum value with merge tree setting 'max_suspicious_broken_parts_bytes' in <merge_tree> configuration "
            "section or in table settings in .sql file (don't forget to return setting back to default value)",
            suspicious_broken_parts,
            formatReadableSizeWithBinarySuffix(suspicious_broken_parts_bytes),
            formatReadableSizeWithBinarySuffix((*settings)[MergeTreeSetting::max_suspicious_broken_parts_bytes]));
}
```
(7) If the storage containing the table is writable and not write-once, each broken part detected in step (5) is renamed using the `"broken-on-start"` prefix and the partition name, then moved to the `detached` directory.
Note: Steps (3) through (7) describe how local parts that are present under Keeper's `parts` path are handled.
(8) Unexpected parts are handled in essentially the same way as expected parts:
```
for (auto & load_state : unexpected_data_parts)
{
    std::lock_guard lock(unexpected_data_parts_mutex);
    chassert(!load_state.part);
    if (unexpected_data_parts_loading_canceled)
    {
        runner.waitForAllToFinishAndRethrowFirstError();
        return;
    }
    runner([&]()
    {
        loadUnexpectedDataPart(load_state);

        chassert(load_state.part);
        if (load_state.is_broken)
            load_state.part->renameToDetached("broken-on-start", /*ignore_error=*/ replicated); /// detached parts must not have '_' in prefixes
    }, Priority{});
}
```
(9) For parts that have already been covered:
```
loading_tree.traverse(/*recursive=*/ true, [&](const auto & node)
{
    if (!node->is_loaded)
        unloaded_parts.push_back(node);
});

/// By the way, if all disks are readonly, it does not make sense to load outdated parts (we will not own them).
if (!unloaded_parts.empty() && !all_disks_are_readonly)
{
    LOG_DEBUG(log, "Found {} outdated data parts. They will be loaded asynchronously", unloaded_parts.size());

    {
        std::lock_guard lock(outdated_data_parts_mutex);
        outdated_unloaded_data_parts = std::move(unloaded_parts);
        outdated_data_parts_loading_finished = false;
    }

    outdated_data_parts_loading_task = getContext()->getSchedulePool().createTask(
        "MergeTreeData::loadOutdatedDataParts",
        [this] { loadOutdatedDataParts(/*is_async=*/ true); });
}
```
#### Local Parts Not Present in Keeper's `parts` Path
Local parts that are not present under Keeper's `parts` path are handled in `ReplicatedMergeTreeAttachThread::runImpl`:
```
ReplicatedMergeTreeAttachThread::runImpl()  ->
  StorageReplicatedMergeTree::checkParts()  ->
    StorageReplicatedMergeTree::checkPartsImpl()
```
