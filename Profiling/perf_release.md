## Background
Some production queries are slow in ways that cannot be diagnosed by reading the code alone. In these cases, use a performance-analysis tool such as `perf`.
However, `perf` needs symbol information, which release binaries usually do not contain.

## Using `perf` with debug symbols
Use the following command to extract the debug section from the compiled binary.
```
objcopy --only-keep-debug yourBinary yourBinary.debug
```
Then use `--add-gnu-debuglink` to associate the debug information with the binary. This does not bloat the binary; it only adds a debug-link section.
```
objcopy --add-gnu-debuglink=yourBinary.debug yourBinary
```
After these steps, `perf` can show the names of hot functions when analyzing a stripped binary instead of reporting them as unknown.
