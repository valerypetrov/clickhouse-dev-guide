## CMake build options
`-DENABLE_RUST` controls whether Rust-related modules, such as the `chdig` component, are built.

`-DENABLE_JEMALLOC` controls whether the jemalloc memory allocator is used.

`-DSANITIZE` selects which sanitizer is enabled.

The following example disables Rust modules and jemalloc, and builds with MemorySanitizer:
```
cmake -DENABLE_JEMALLOC=0 -DSANITIZE=memory -DENABLE_RUST=0 ..
```

## Ninja commands
ClickHouse uses Ninja by default. The CMake `-G` option selects the build-system generator.

Build only the unit-test target:
```
ninja unit_tests_dbms
```

## Limiting the maximum number of concurrent build jobs
ClickHouse does not necessarily use every build job requested on the command line:
```
ninja -j 32
```
For example, even if you specify `-j32` to request 32 concurrent jobs, the actual concurrency is determined by the following parameters:
- PARALLEL_COMPILE_JOBS
- PARALLEL_LINK_JOBS
- MAX_COMPILER_MEMORY
- MAX_LINKER_MEMORY

The top-level `CMakeLists.txt` contains the logic that configures `MAX_COMPILER_MEMORY` and `MAX_LINKER_MEMORY`.
1. First, `cmake_host_system_information` checks the system's available memory and assigns the result to `AVAILABLE_PHYSICAL_MEMORY`:
```
cmake_host_system_information(RESULT AVAILABLE_PHYSICAL_MEMORY QUERY AVAILABLE_PHYSICAL_MEMORY) # Not available under freebsd
```
2. When the system has more than 8 GB of available memory, or when the amount cannot be detected, the compiler's `-pipe` option is enabled to reduce disk I/O (and the use of temporary files) at the cost of increased memory usage.
```
if(NOT AVAILABLE_PHYSICAL_MEMORY OR AVAILABLE_PHYSICAL_MEMORY GREATER 8000)
    # Less `/tmp` usage, more RAM usage.
    option(COMPILER_PIPE "-pipe compiler option" ON)
endif()
```
3. The maximum compiler and linker memory limits depend on whether the `-pipe` compiler option is enabled. The compiler limit is 2.5 GB when enabled and 1.5 GB otherwise; the linker limit is always 5 GB.
```
if(COMPILER_PIPE)
    set(MAX_COMPILER_MEMORY 2500)
else()
    set(MAX_COMPILER_MEMORY 1500)
endif()
set(MAX_LINKER_MEMORY 5000)
include(cmake/limit_jobs.cmake)
```

The logic in `cmake/limit_jobs.cmake` is as follows:
```
# Limit compiler/linker job concurrency to avoid OOMs on subtrees where compilation/linking is memory-intensive.
#
# Usage from CMake:
#    set (MAX_COMPILER_MEMORY 2000 CACHE INTERNAL "") # megabyte
#    set (MAX_LINKER_MEMORY 5000 CACHE INTERNAL "") # megabyte
#    include (cmake/limit_jobs.cmake)
#
# (bigger values mean fewer jobs)

cmake_host_system_information(RESULT TOTAL_PHYSICAL_MEMORY QUERY TOTAL_PHYSICAL_MEMORY)
cmake_host_system_information(RESULT NUMBER_OF_LOGICAL_CORES QUERY NUMBER_OF_LOGICAL_CORES)

# Set to disable the automatic job-limiting
option(PARALLEL_COMPILE_JOBS "Maximum number of concurrent compilation jobs" OFF)
option(PARALLEL_LINK_JOBS "Maximum number of concurrent link jobs" OFF)

if (NOT PARALLEL_COMPILE_JOBS AND MAX_COMPILER_MEMORY)
    math(EXPR PARALLEL_COMPILE_JOBS ${TOTAL_PHYSICAL_MEMORY}/${MAX_COMPILER_MEMORY})

    if (NOT PARALLEL_COMPILE_JOBS)
        set (PARALLEL_COMPILE_JOBS 1)
    endif ()
    if (PARALLEL_COMPILE_JOBS LESS NUMBER_OF_LOGICAL_CORES)
        message("The auto-calculated compile jobs limit (${PARALLEL_COMPILE_JOBS}) underutilizes CPU cores (${NUMBER_OF_LOGICAL_CORES}). Set PARALLEL_COMPILE_JOBS to override.")
    endif()
endif ()

if (NOT PARALLEL_LINK_JOBS AND MAX_LINKER_MEMORY)
    math(EXPR PARALLEL_LINK_JOBS ${TOTAL_PHYSICAL_MEMORY}/${MAX_LINKER_MEMORY})

    if (NOT PARALLEL_LINK_JOBS)
        set (PARALLEL_LINK_JOBS 1)
    endif ()
    if (PARALLEL_LINK_JOBS LESS NUMBER_OF_LOGICAL_CORES)
        message("The auto-calculated link jobs limit (${PARALLEL_LINK_JOBS}) underutilizes CPU cores (${NUMBER_OF_LOGICAL_CORES}). Set PARALLEL_LINK_JOBS to override.")
    endif()
endif ()
```
1. Calculation:
```
math(EXPR PARALLEL_COMPILE_JOBS ${TOTAL_PHYSICAL_MEMORY}/${MAX_COMPILER_MEMORY})
Number of compile jobs = total memory / MAX_COMPILER_MEMORY (2.5 GB or 1.5 GB by default)

Number of link jobs = total memory / MAX_LINKER_MEMORY (5 GB by default)
```
2. Precedence rules:
Explicit user settings take precedence: if `PARALLEL_COMPILE_JOBS` or `PARALLEL_LINK_JOBS` is set, its user-provided value is used.

Otherwise, the value is calculated automatically from the memory limits.

At least one job is guaranteed: if the calculated result is zero, it is set to one.

In summary, ClickHouse calculates a safe number of concurrent jobs by dividing total memory by the per-job memory limit. Avoiding out-of-memory errors takes priority over maximizing CPU utilization. For local builds, you can increase the following parameters according to the available CPU cores and memory:
```
PARALLEL_COMPILE_JOBS
PARALLEL_LINK_JOBS
```
