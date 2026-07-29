## ReadTask
`MergeTreeReadTask` represents a read task.

### estimateNumRows
It first estimates the row count from the byte quota and per-column size, ensures that the estimate is no smaller than the index granularity, and then adjusts it using the number of unread rows and the index granularity.

This ensures that the read respects byte and column-size limits without reading too little data to remain efficient.

If no predictor is available, it falls back to the maximum block-row limit.
It then takes the smaller of `max_block_size_rows` and `recommended_rows`. The default value of `max_block_size_rows` is `65409`.
```
  UInt64 recommended_rows = estimateNumRows();
  UInt64 rows_to_read = std::max(static_cast<UInt64>(1), std::min(block_size_params.max_block_size_rows, recommended_rows));
```

## MergeTreeReadPool

`fillPerThreadInfo` creates the read task for each thread, distributing the total marks from all parts that remain after partition and min-max pruning evenly across the threads.

## MergeTreeInOrderSelectAlgorithm
Reads marks in forward order.

## MergeTreeInReverseOrderSelectAlgorithm
Reads marks in reverse order.
