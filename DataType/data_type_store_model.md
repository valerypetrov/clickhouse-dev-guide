## Tuple


## Array


## Map


## Variant


## Dynamic
Dynamic is a new semi-structured data column type.
It stores dynamically typed data, allowing values with different underlying types to coexist in a single column. For example:
```
SELECT CAST(1 AS Dynamic), CAST('abc' AS Dynamic), CAST([1,2] AS Dynamic);
```
### 1. Storage Model Overview

Unlike the traditional `Variant` type:

- `Variant` is a statically defined polymorphic type with a fixed set of types.

- `Dynamic` is an extensible polymorphic type whose set of types grows automatically at runtime.

Both rely on `ColumnVariant` internally, but `Dynamic` adds a self-describing dynamic management layer on top.

The source code shows that the `ColumnDynamic` storage structure is effectively a `ColumnVariant` plus a metadata layer.
```
// simplified view:
class ColumnDynamic {
    ...
    WrappedPtr variant_column;      // Actual data storage: ColumnVariant
    ColumnVariant* variant_column_ptr; // Direct pointer retained for performance
    VariantInfo variant_info;       // Type information for all current Dynamic variants
    ...
}
```
### 2. Internal Structure (Memory Layer)

At its core, `Dynamic` is an extensible `Variant`. Each type that actually occurs (`Int32`, `String`, `Array`, `Map`, etc.) is stored in a subcolumn, while a discriminator (type identifier) records the type of each row.
| Layer         | Object                               | Description                                                           |
| ---------- | -------------------------------- | ------------------------------------------------------------ |
| **Top-level column**  | `ColumnDynamic`                  | Exposed as a regular column                                                    |
| **Internal variant column**  | `ColumnVariant`                  | Stores the data for all dynamic types                                                |
| **Variant mapping metadata** | `VariantInfo`                    | Maps each type to its discriminator (numeric ID) and type name                              |
| **Subcolumn collection**   | `ColumnVariant::columns[]`       | One subcolumn per type (`ColumnVector`, `ColumnString`, `ColumnArray`, etc.) |
| **Shared-type column**  | `SharedVariant` (`ColumnString`) | When too many types occur, values of new types are serialized into this column in binary format                                   |

Structure overview:
```
ColumnDynamic
 └── ColumnVariant
      ├── discriminator[]  ← UInt8 vector indicating the type of each row
      ├── offsets[]        ← Index offsets
      ├── variants:
      │    ├── [0]: ColumnUInt32
      │    ├── [1]: ColumnString
      │    ├── [2]: ColumnArray(UInt32)
      │    ├── [3]: ColumnMap(String, UInt32)
      │    └── [255]: ColumnString (SharedVariant)
      └── ...
```
### 3. On-Disk Storage Structure (MergeTree Layer)

In a MergeTree part, the physical layout of a `Dynamic` column is determined by its internal `ColumnVariant`.
Each variant type has a separate data stream stored using a naming convention similar to:
```
column_name.variant_<type>.<stream_suffix>
```
If the `Dynamic` values inserted into a table have three types (`UInt32`, `String`, and `Array(UInt32)`),
the resulting on-disk layout is approximately:
```
t/
 ├── dyn.variant_UInt32.bin
 ├── dyn.variant_UInt32.mrk3
 ├── dyn.variant_String.bin
 ├── dyn.variant_String.mrk3
 ├── dyn.variant_Array_UInt32.bin
 ├── dyn.variant_Array_UInt32.mrk3
 ├── dyn.discriminators.bin      ← Stores the type ID for each row
 ├── dyn.offsets.bin             ← Row offset table
 └── ...
```

### 4. Dynamic Type Expansion

When a value of a new type is inserted:

void ColumnDynamic::insert(const Field & x)


The following operations occur internally:
1. Call `x.getType()` to determine the value's type.
2. Look up the type in `variant_info.variant_name_to_discriminator`.
3. If the type already exists:
    - Write the value directly to the corresponding subcolumn.
4. If it is a new type and the limit has not been reached:
    - Call `addNewVariant()`.
    - Add a new subcolumn to `ColumnVariant`.
    - Update the `variant_info` mapping.
5. If the `max_dynamic_types` limit has been exceeded:
    - Call `insertValueIntoSharedVariant()`.
    - Serialize the value to binary and write it to `SharedVariant` (`ColumnString`).

### 5. The SharedVariant Fallback

`SharedVariant` is a fallback type:
- When there are too many types, or a type cannot be determined, new values are serialized into `SharedVariant`.
- The serialization format is `<type_name><binary_value>`.
- This ensures that a `Dynamic` column can accept values of any type.

```
insert 1 → UInt32
insert 'abc' → String
insert map('k',1) → new type Map(String,UInt32)
insert tuple(1,2) → limit exceeded → stored in SharedVariant as "Tuple(UInt32,UInt32)|<bin>"
```
### 6. Example
1. Create a table
```
CREATE TABLE default.dynamic_merge_tree_test
(
    `id` UInt64,
    `name` String,
    `dyn` Dynamic,
    `ts` DateTime DEFAULT now()
)
ENGINE = MergeTree
ORDER BY (id, ts)
SETTINGS index_granularity = 8192
```
2. Insert data
```
INSERT INTO default.dynamic_merge_tree_test (id, name, dyn)
SELECT
    number AS id,
    concat('user_', toString(number)) AS name,
    multiIf(
        (rand() % 3) = 0, CAST(toUInt32(rand() % 1000) AS Dynamic),
        (rand() % 3) = 1, CAST(concat('str_', toString(rand() % 1000)) AS Dynamic),
        CAST(arrayMap(x -> toUInt32(x % 1000), range((rand() % 5) + 1)) AS Dynamic)
    ) AS dyn
FROM numbers(1000000);
```

Inspect the files of the wide part:
```
root@ubantu64:~/work/ClickHouse/build/programs/store/a76/a7609f34-18c3-419c-a364-9f7d44d52e29/all_2_2_0# ll
total 12292
drwxr-x--- 2 root root    4096 Oct 11 18:13  ./
drwxr-x--- 5 root root    4096 Oct 11 18:13  ../
-rw-r----- 1 root root    1136 Oct 11 18:13  checksums.txt
-rw-r----- 1 root root      91 Oct 11 18:13  columns.txt
-rw-r----- 1 root root     307 Oct 11 18:13  columns_substreams.txt
-rw-r----- 1 root root       7 Oct 11 18:13  count.txt
-rw-r----- 1 root root      10 Oct 11 18:13  default_compression_codec.txt
-rw-r----- 1 root root  924507 Oct 11 18:13 'dyn.Array(UInt32).bin'
-rw-r----- 1 root root     466 Oct 11 18:13 'dyn.Array(UInt32).cmrk2'
-rw-r----- 1 root root  811132 Oct 11 18:13 'dyn.Array(UInt32).size0.bin'
-rw-r----- 1 root root     464 Oct 11 18:13 'dyn.Array(UInt32).size0.cmrk2'
-rw-r----- 1 root root       0 Oct 11 18:13  dyn.SharedVariant.bin
-rw-r----- 1 root root      56 Oct 11 18:13  dyn.SharedVariant.cmrk2
-rw-r----- 1 root root 1127869 Oct 11 18:13  dyn.String.bin
-rw-r----- 1 root root     426 Oct 11 18:13  dyn.String.cmrk2
-rw-r----- 1 root root  841164 Oct 11 18:13  dyn.UInt32.bin
-rw-r----- 1 root root     408 Oct 11 18:13  dyn.UInt32.cmrk2
-rw-r----- 1 root root      69 Oct 11 18:13  dyn.dynamic_structure.bin
-rw-r----- 1 root root      63 Oct 11 18:13  dyn.dynamic_structure.cmrk2
-rw-r----- 1 root root  565564 Oct 11 18:13  dyn.variant_discr.bin
-rw-r----- 1 root root     268 Oct 11 18:13  dyn.variant_discr.cmrk2
-rw-r----- 1 root root 4004915 Oct 11 18:13  id.bin
-rw-r----- 1 root root     415 Oct 11 18:13  id.cmrk2
-rw-r----- 1 root root       1 Oct 11 18:13  metadata_version.txt
-rw-r----- 1 root root 4188196 Oct 11 18:13  name.bin
-rw-r----- 1 root root     415 Oct 11 18:13  name.cmrk2
-rw-r----- 1 root root     177 Oct 11 18:13  primary.cidx
-rw-r----- 1 root root     229 Oct 11 18:13  serialization.json
-rw-r----- 1 root root   18042 Oct 11 18:13  ts.bin
-rw-r----- 1 root root     375 Oct 11 18:13  ts.cmrk2

```
## Json



## Nullable
