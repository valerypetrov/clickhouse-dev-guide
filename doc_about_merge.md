`Processors/Merges/Algorithms/MergedData.cpp`: stores merged data.
`Processors/Merges/Algorithms/MergingSortedAlgorithm.cpp`: merges different blocks from multiple source parts into `MergedData`; it does not merge rows that have the same sorting key.
`Core/SortCursor.h`: an iterator used to access data in `MergedData`.
`Processors/Merges/Algorithms/IMergingAlgorithm.h`: the base class for merge algorithms.
`Storages/MergeTree/MergeTreeSequentialSource.cpp`: the processor that reads data from a part during a merge.
`Storages/MergeTree/MergeTask.cpp`: represents a merge task and contains its complete execution flow.
`Storages/MergeTree/MergedBlockOutputStream.h`: serializes a part's data and flushes it to disk.
`Storages/MergeTree/MergeTreeDataPartWriterWide.cpp`: writes parts in Wide format.
`Storages/MergeTree/MergeTreeIndexGranularity.cpp`: index granularity, including logic that calculates adaptive granularity from a block's `rows` and `uncompressed_bytes`.
`Storages/MergeTree/MergeTreeIndexGranularityConstant.h`: constant index granularity, which does not change.



<horizatol, new_part> -> blocks_are_granules = false.
<horizatol, old_part> -> blocks_are_granules = false.
<vertical, old_part> -> blocks_are_granules = true.due to new_part, then index_granularity is not ok.
<vertical, new_part> -> blocks_are_granules = true.due to new_part, then index_granularity is ok.
