## How `apply_mutations_on_fly` works

After an `ALTER ...` mutation is executed, a mutation entry is added to `current_mutations_by_version`.
```
Int64 StorageMergeTree::startMutation(const MutationCommands & commands, ContextPtr query_context)
{
    /// Choose any disk, because when we load mutations we search them at each disk
    /// where storage can be placed. See loadMutations().
    auto disk = getStoragePolicy()->getAnyDisk();
    TransactionID current_tid = Tx::PrehistoricTID;
    String additional_info;
    auto txn = query_context->getCurrentTransaction();
    if (txn)
    {
        current_tid = txn->tid;
        additional_info = fmt::format(" (TID: {}; TIDH: {})", current_tid, current_tid.getHash());
    }

    MergeTreeMutationEntry entry(commands, disk, relative_data_path, insert_increment.get(), current_tid, getContext()->getWriteSettings());
    auto block_holder = allocateBlockNumber(CommittingBlock::Op::Mutation);

    Int64 version = block_holder->block.number;
    entry.commit(version);
    String mutation_id = entry.file_name;
    if (txn)
        txn->addMutation(shared_from_this(), mutation_id);

    {
        std::lock_guard lock(currently_processing_in_background_mutex);

        auto [it, inserted] = current_mutations_by_version.try_emplace(version, std::move(entry));
        if (!inserted)
            throw Exception(ErrorCodes::LOGICAL_ERROR, "Mutation {} already exists, it's a bug", version);

        incrementMutationsCounters(mutation_counters, *it->second.commands);
    }

    LOG_INFO(log, "Added mutation: {}{}", mutation_id, additional_info);
    background_operations_assignee.trigger();
    return version;
}


```
`current_mutations_by_version` is a `map` whose key is an increasing block number and whose value is a mutation operation.
```
    std::map<UInt64, MergeTreeMutationEntry> current_mutations_by_version;

```

```
MergeTreeData::getAlterConversionsForPart  ->
  StorageMergeTree::MutationsSnapshot::getOnFlyMutationCommandsForPart  ->
    
```


```
MutationCommands StorageMergeTree::MutationsSnapshot::getOnFlyMutationCommandsForPart(const DataPartPtr & part) const
{
    MutationCommands result;
    UInt64 part_data_version = part->info.getDataVersion();

    for (const auto & [mutation_version, commands] : mutations_by_version | std::views::reverse)
    {
        if (mutation_version <= part_data_version)
            break;

        addSupportedCommands(*commands, mutation_version, result);
    }

    std::reverse(result.begin(), result.end());
    return result;
}
```

```
        if (read_task_info.alter_conversions->hasMutations())
        {
            auto columns_list = storage_snapshot->getColumnsByNames(options, column_names);
            auto mutation_steps
                = read_task_info.alter_conversions->getMutationSteps(part_info, columns_list, storage_snapshot->metadata, getContext());
            std::move(mutation_steps.begin(), mutation_steps.end(), std::back_inserter(read_task_info.mutation_steps));
        }
```

The main question is whether this function can be reused:
```
std::vector<MutationActions> AlterConversions::getMutationActions(
    const IMergeTreeDataPartInfoForReader & part_info,
    const NamesAndTypesList & read_columns,
    const StorageMetadataPtr & metadata_snapshot,
    const ContextPtr & context) const
{
    if (mutation_commands.empty())
        return {};

    const auto * loaded_part_info = dynamic_cast<const LoadedMergeTreeDataPartInfoForReader *>(&part_info);
    if (!loaded_part_info)
        throw Exception(ErrorCodes::NOT_IMPLEMENTED,
            "Applying mutations on-fly is supported only for loaded data parts");

    Names storage_read_columns;
    NameSet storage_read_columns_set;

    for (const auto & column : read_columns)
    {
        auto name_in_storage = column.getNameInStorage();
        if (storage_read_columns_set.emplace(name_in_storage).second)
            storage_read_columns.emplace_back(name_in_storage);
    }

    addColumnsRequiredForMaterialized(storage_read_columns, storage_read_columns_set, metadata_snapshot, context);
    auto filtered_commands = filterMutationCommands(storage_read_columns, std::move(storage_read_columns_set));

    if (filtered_commands.empty())
        return {};

    ProfileEvents::increment(ProfileEvents::ReadTasksWithAppliedMutationsOnFly);
    ProfileEvents::increment(ProfileEvents::MutationsAppliedOnFlyInAllReadTasks, filtered_commands.size());

    MutationsInterpreter::Settings settings(true);
    settings.return_all_columns = true;
    settings.recalculate_dependencies_of_updated_columns = false;

    const auto & part = loaded_part_info->getDataPart();
    auto alter_conversions = std::make_shared<AlterConversions>();

    MutationsInterpreter interpreter(
        const_cast<MergeTreeData &>(part->storage),
        part,
        alter_conversions,
        metadata_snapshot,
        std::move(filtered_commands),
        std::move(storage_read_columns),
        context,
        settings);

    return interpreter.getMutationActions();
}

```
