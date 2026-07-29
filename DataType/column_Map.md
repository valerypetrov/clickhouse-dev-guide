<!-- TOC -->

## 1. Overall Design
In ClickHouse, `Map(K, V)` is not an independent low-level storage structure.
It is implemented by composing more fundamental column types:
```
Map(K, V) = Array(Tuple(key, value))
```
Specifically:
- Each row's map is represented as an `Array`.
- Each array element is a `(key, value)` tuple.
- The complete column is physically stored as:
  - A `ColumnArray`, which records the number of array elements in each row.
  - A nested `ColumnTuple` containing:
    - `ColumnVector<KeyType>`: all keys stored sequentially.
    - `ColumnVector<ValueType>`: all values stored sequentially.

Here, `nested` is `ColumnArray(ColumnTuple(keys, values))`.
```
/** Column, that stores a nested Array(Tuple(key, value)) column.
 */
class ColumnMap final : public COWHelper<IColumnHelper<ColumnMap>, ColumnMap>
{
private:
    friend class COWHelper<IColumnHelper<ColumnMap>, ColumnMap>;

    WrappedPtr nested;
...
```
For example, the in-memory layout looks like this:
```
Row1:  [(k1,v1), (k2,v2)]
Row2:  [(k3,v3)]
Row3:  [(k4,v4), (k5,v5), (k6,v6)]
```
It is split into:
| Data structure        | Example contents                     |
| ----------- | ------------------------ |
| `keys` column    | [k1, k2, k3, k4, k5, k6] |
| `values` column  | [v1, v2, v3, v4, v5, v6] |
| `offsets` column | [2, 1, 3]  ← Number of elements in each row |

On disk, the `offsets` column is generally stored with the name `size0`.
## 2. Testing the Wide Storage Model for Map
```
1. CREATE TABLE default.test
(
    `c1` Int32,
    `c2` Map(String, Int32)
)
ENGINE = MergeTree
ORDER BY c1
SETTINGS index_granularity = 8192
```
2. Insert one million random rows with keys in the range `[k0, k9]`
```
INSERT INTO test
SELECT
    number AS c1,
    map(
        concat('k', toString(rand() % 10)),
        rand() % 1000
    ) AS c2
FROM numbers(100000);
```
3. Inspect the data storage path
```
select table,data_paths from system.tables where table = 'test'
```
4. On-disk part layout
```
root@ubantu64:~/work/ClickHouse/build/programs/store/9d8/9d8c8586-ca90-4ae1-9e6a-cb1188d33c3a/all_2_2_0# cat count.txt
1000000
root@ubantu64:~/work/ClickHouse/build/programs/store/9d8/9d8c8586-ca90-4ae1-9e6a-cb1188d33c3a/all_2_2_0# ll
total 8228
drwxr-x--- 2 root root    4096 Oct 14 14:50 ./
drwxr-x--- 5 root root    4096 Oct 14 14:50 ../
-rw-r----- 1 root root 4017256 Oct 14 14:50 c1.bin
-rw-r----- 1 root root     439 Oct 14 14:50 c1.cmrk2
-rw-r----- 1 root root 1432310 Oct 14 14:50 c2%2Ekeys.bin
-rw-r----- 1 root root     342 Oct 14 14:50 c2%2Ekeys.cmrk2
-rw-r----- 1 root root 2875686 Oct 14 14:50 c2%2Evalues.bin
-rw-r----- 1 root root     424 Oct 14 14:50 c2%2Evalues.cmrk2
-rw-r----- 1 root root   36169 Oct 14 14:50 c2.size0.bin
-rw-r----- 1 root root     323 Oct 14 14:50 c2.size0.cmrk2
-rw-r----- 1 root root     559 Oct 14 14:50 checksums.txt
-rw-r----- 1 root root      72 Oct 14 14:50 columns.txt
-rw-r----- 1 root root     139 Oct 14 14:50 columns_substreams.txt
-rw-r----- 1 root root       7 Oct 14 14:50 count.txt
-rw-r----- 1 root root      10 Oct 14 14:50 default_compression_codec.txt
-rw-r----- 1 root root       1 Oct 14 14:50 metadata_version.txt
-rw-r----- 1 root root     267 Oct 14 14:50 primary.cidx
-rw-r----- 1 root root      93 Oct 14 14:50 serialization.json
```
- The `columns_substreams.txt` file records information about each column and its subcolumns.
```
root@ubantu64:~/work/ClickHouse/build/programs/store/9d8/9d8c8586-ca90-4ae1-9e6a-cb1188d33c3a/all_2_2_0# cat columns_substreams.txt
columns substreams version: 1
2 columns:
1 substreams for column `c1`:
        c1
3 substreams for column `c2`:
        c2.size0
        c2%2Ekeys
        c2%2Evalues
```
## Frequently Asked Questions
1. Duplicate keys within a row are all stored.

1️⃣ Create a test table
```
DROP TABLE IF EXISTS test_map_dup;

CREATE TABLE test_map_dup
(
    id UInt32,
    m Map(String, Int32)
)
ENGINE = MergeTree()
ORDER BY id;
```
2️⃣ Insert data containing duplicate keys
```
INSERT INTO test_map_dup VALUES
(1, map('a', 10, 'b', 20, 'a', 30)),
(2, map('x', 5, 'y', 6, 'x', 7, 'x', 8)),
(3, map('p', 100)),
(4, map('q', 1, 'r', 2, 'q', 3));
```
3️⃣ Query all keys in each row
```
SELECT id, mapKeys(m) AS keys
FROM test_map_dup
ORDER BY id;
```
The result is:
```
ubantu64 :) SELECT id, mapKeys(m) AS keys
FROM test_map_dup
ORDER BY id;

SELECT
    id,
    mapKeys(m) AS keys
FROM test_map_dup
ORDER BY id ASC

Query id: 13d94ac8-845c-4238-8d7d-3473f5a2d6e4

   ┌─id─┬─keys──────────────┐
1. │  1 │ ['a','b','a']     │
2. │  2 │ ['x','y','x','x'] │
3. │  3 │ ['p']             │
4. │  4 │ ['q','r','q']     │
   └────┴───────────────────┘

4 rows in set. Elapsed: 0.002 sec.
```
2. Query the value for a key (the first match is returned by default)
```
ubantu64 :) SELECT
    id,
    m['a'] AS value_a,
    m['x'] AS value_x,
    m['q'] AS value_q
FROM test_map_dup
ORDER BY id;


SELECT
    id,
    m['a'] AS value_a,
    m['x'] AS value_x,
    m['q'] AS value_q
FROM test_map_dup
ORDER BY id ASC

Query id: 823d516d-ba8f-438d-a587-34d75f4805fd

   ┌─id─┬─value_a─┬─value_x─┬─value_q─┐
1. │  1 │      10 │       0 │       0 │
2. │  2 │       0 │       5 │       0 │
3. │  3 │       0 │       0 │       0 │
4. │  4 │       0 │       0 │       1 │
   └────┴─────────┴─────────┴─────────┘

4 rows in set. Elapsed: 0.002 sec.
```
3. Use `column_name.keys` and `column_name.values` to inspect all key and value entries directly.

```
ubantu64 :) SELECT
    c1,
    c2.keys AS keys_col,
    c2.values AS values_col
FROM test
ORDER BY c1 limit 10;

SELECT
    c1,
    c2.keys AS keys_col,
    c2.values AS values_col
FROM test
ORDER BY c1 ASC
LIMIT 10

Query id: 15f2a030-a27c-4661-a89e-95a4895fdbe4

    ┌─c1─┬─keys_col─┬─values_col─┐
 1. │  0 │ ['k3']   │ [93]       │
 2. │  0 │ ['k9']   │ [29]       │
 3. │  1 │ ['k5']   │ [805]      │
 4. │  1 │ ['k1']   │ [251]      │
 5. │  2 │ ['k4']   │ [384]      │
 6. │  2 │ ['k5']   │ [265]      │
 7. │  3 │ ['k5']   │ [705]      │
 8. │  3 │ ['k0']   │ [830]      │
 9. │  4 │ ['k9']   │ [449]      │
10. │  4 │ ['k8']   │ [618]      │
    └────┴──────────┴────────────┘

10 rows in set. Elapsed: 0.002 sec. Processed 16.38 thousand rows, 557.06 KB (6.90 million rows/s., 234.61 MB/s.)
Peak memory usage: 1.49 MiB.
```
## Summary
A ClickHouse `Map` column is not a native hash-table structure. Instead, it implements a logical mapping using `Array(Tuple(key, value))`.
`ColumnMap` cannot provide O(1) access internally. ClickHouse does, however, implement logical mapping functions such as `map[key]`; internally, this operation performs a scan.
