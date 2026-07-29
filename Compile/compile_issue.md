### Symptoms

```
[291/13133] Running cpp protocol buffer compiler on prompb/types.proto
FAILED: contrib/prometheus-protobufs-cmake/prompb/types.pb.h contrib/prometheus-protobufs-cmake/prompb/types.pb.cc /data/code/build/contrib/prometheus-protobufs-cmake/prompb/types.pb.h /data/code/build/contrib/prometheus-protobufs-cmake/prompb/types.pb.cc 
cd /data/code/build/contrib/prometheus-protobufs-cmake && /data/code/build/contrib/google-protobuf-cmake/protoc --cpp_out /data/code/build/contrib/prometheus-protobufs-cmake -I /data/code/contrib/prometheus-protobufs-cmake -I /data/code/contrib/google-protobuf/src -I /data/code/contrib/prometheus-protobufs -I /data/code/contrib/prometheus-protobufs-gogo /data/code/contrib/prometheus-protobufs/prompb/types.proto
Illegal instruction (core dumped)
[292/13133] Running cpp protocol buffer compiler on gogoproto/gogo.proto
FAILED: contrib/prometheus-protobufs-cmake/gogoproto/gogo.pb.h contrib/prometheus-protobufs-cmake/gogoproto/gogo.pb.cc /data/code/build/contrib/prometheus-protobufs-cmake/gogoproto/gogo.pb.h /data/code/build/contrib/prometheus-protobufs-cmake/gogoproto/gogo.pb.cc 
cd /data/code/build/contrib/prometheus-protobufs-cmake && /data/code/build/contrib/google-protobuf-cmake/protoc --cpp_out /data/code/build/contrib/prometheus-protobufs-cmake -I /data/code/contrib/prometheus-protobufs-cmake -I /data/code/contrib/google-protobuf/src -I /data/code/contrib/prometheus-protobufs -I /data/code/contrib/prometheus-protobufs-gogo /data/code/contrib/prometheus-protobufs-gogo/gogoproto/gogo.proto
Illegal instruction (core dumped)
[293/13133] Running cpp protocol buffer compiler on prompb/remote.proto
FAILED: contrib/prometheus-protobufs-cmake/prompb/remote.pb.h contrib/prometheus-protobufs-cmake/prompb/remote.pb.cc /data/code/build/contrib/prometheus-protobufs-cmake/prompb/remote.pb.h /data/code/build/contrib/prometheus-protobufs-cmake/prompb/remote.pb.cc 
cd /data/code/build/contrib/prometheus-protobufs-cmake && /data/code/build/contrib/google-protobuf-cmake/protoc --cpp_out /data/code/build/contrib/prometheus-protobufs-cmake -I /data/code/contrib/prometheus-protobufs-cmake -I /data/code/contrib/google-protobuf/src -I /data/code/contrib/prometheus-protobufs -I /data/code/contrib/prometheus-protobufs-gogo /data/code/contrib/prometheus-protobufs/prompb/remote.proto
Illegal instruction (core dumped)
```

The build repeatedly fails with `Illegal instruction (core dumped)`.

The error shows that the failure occurs when `protoc` generates the Protocol Buffers files:

```
/data/code/build/contrib/google-protobuf-cmake/protoc --cpp_out 
```

### Investigation

Debugging the compiled `protoc` binary with GDB shows that it crashes in `__libc_csu_init`:

```
GNU gdb (GDB) KylinOS 9.2-7.p02.ky10
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "aarch64-kylin-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from protoc...
(gdb) r
Starting program: /data1/clickhouse_build/ck_25_8/clickhouse/build/contrib/google-protobuf-cmake/protoc 
warning: File "/usr/lib64/libthread_db-1.0.so" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
        add-auto-load-safe-path /usr/lib64/libthread_db-1.0.so
line to your configuration file "/root/.gdbinit".
To completely disable this security protection add
        set auto-load safe-path /
line to your configuration file "/root/.gdbinit".
For more information about this security protection see the
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
        info "(gdb)Auto-loading safe path"
warning: Unable to find libthread_db matching inferior's thread library, thread debugging will not be available.

Program received signal SIGILL, Illegal instruction.
0x0000aaaaaaf52934 in global constructors keyed to 000100 ()
(gdb) bt
#0  0x0000aaaaaaf52934 in global constructors keyed to 000100 ()
#1  0x0000aaaaaafc09a0 in __libc_csu_init (argc=1, argv=0xfffffffff298, envp=0xfffffffff2a8) at elf-init.c:88
#2  0x0000fffff7dd3f28 in __libc_start_main () from /usr/lib64/libc.so.6
#3  0x0000aaaaaac27c34 in _start () at ../sysdeps/aarch64/start.S:92
Backtrace stopped: previous frame identical to this frame (corrupt stack?)
```

Inspect the instruction at the point of the crash:

```
(gdb) x/i $pc
=> 0xaaaaaaf52934 <_GLOBAL__I_000100+20>:       ldaprb  w8, [x8]
```

`objdump` confirms that instructions from the `rcpc` family were generated:

```
#objdump -d protoc |grep ldaprb | head -10
  1db4f4:       38bfc129        ldaprb  w9, [x9]
  1dc8cc:       38bfc108        ldaprb  w8, [x8]
  1dcd74:       38bfc108        ldaprb  w8, [x8]
  1dcd94:       38bfc108        ldaprb  w8, [x8]
  1de208:       38bfc108        ldaprb  w8, [x8]
  1df21c:       38bfc108        ldaprb  w8, [x8]
  1e1004:       38bfc108        ldaprb  w8, [x8]
  1e2528:       38bfc108        ldaprb  w8, [x8]
  238620:       38bfc129        ldaprb  w9, [x9]
  267074:       38bfc108        ldaprb  w8, [x8]
```

The ARM manual identifies `ldaprb` as an instruction in the `rcpc` family. This family includes:

[LDAPP](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPP--Load-acquire-RCpc-pair-of-registers-?lang=en): Load-acquire RCpc pair of registers.

[LDAPR](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPR--Load-acquire-RCpc-register-?lang=en): Load-acquire RCpc register.

[LDAPRB](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPRB--Load-acquire-RCpc-register-byte-?lang=en): Load-acquire RCpc register byte.

[LDAPRH](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPRH--Load-acquire-RCpc-register-halfword-?lang=en): Load-acquire RCpc register halfword.

[LDAPUR](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPUR--Load-acquire-RCpc-register--unscaled--?lang=en): Load-acquire RCpc register (unscaled).

[LDAPURB](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPURB--Load-acquire-RCpc-register-byte--unscaled--?lang=en): Load-acquire RCpc register byte (unscaled).

[LDAPURH](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPURH--Load-acquire-RCpc-register-halfword--unscaled--?lang=en): Load-acquire RCpc register halfword (unscaled).

[LDAPURSB](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPURSB--Load-acquire-RCpc-register-signed-byte--unscaled--?lang=en): Load-acquire RCpc register signed byte (unscaled).

[LDAPURSH](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPURSH--Load-acquire-RCpc-register-signed-halfword--unscaled--?lang=en): Load-acquire RCpc register signed halfword (unscaled).

[LDAPURSW](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDAPURSW--Load-acquire-RCpc-register-signed-word--unscaled--?lang=en): Load-acquire RCpc register signed word (unscaled).

[LDIAPP](https://developer.arm.com/documentation/ddi0602/2025-09/Base-Instructions/LDIAPP--Load-Acquire-RCpc-ordered-pair-of-registers-?lang=en): Load-Acquire RCpc ordered pair of registers.

Check the instruction-set features supported by the current CPU:

```
#lscpu |grep 'Flags'
Flags:                           fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma dcpop asimddp asimdfhm ssbs
```

This CPU does not support `rcpc` instructions.

Now return to ClickHouse's `cmake/cpu_features` configuration:

```
if (NO_ARMV81_OR_HIGHER)
        # crc32 is optional in v8.0 and mandatory in v8.1. Enable it as __crc32()* is used in lot's of places and even very old ARM CPUs
        # support it.
        set (COMPILER_FLAGS "${COMPILER_FLAGS} -march=armv8+crc")
        list(APPEND RUSTFLAGS_CPU "-C" "target_feature=+crc,-neon")
    else ()
        # ARMv8.2 is quite ancient but the lowest common denominator supported by both Graviton 2 and 3 processors [1, 10]. In particular, it
        # includes LSE (made mandatory with ARMv8.1) which provides nice speedups without having to fall back to compat flag
        # "-moutline-atomics" for v8.0 [2, 3, 4] that requires a recent glibc with runtime dispatch helper, limiting our ability to run on
        # old OSs.
        #
        # simd:    NEON, introduced as optional in v8.0, A few extensions were added with v8.1 but it's still not mandatory. Enables the
        #          compiler to auto-vectorize.
        # sve:     Scalable Vector Extensions, introduced as optional in v8.2. Available in Graviton 3 but not in Graviton 2, and most likely
        #          also not in CI machines. Compiler support for autovectorization is rudimentary at the time of writing, see [5]. Can be
        #          enabled one-fine-day (TM) but not now.
        # ssbs:    "Speculative Store Bypass Safe". Optional in v8.0, mandatory in v8.5. Meltdown/spectre countermeasure.
        # crypto:  SHA1, SHA256, AES. Optional in v8.0. In v8.4, further algorithms were added but it's still optional, see [6].
        # dotprod: Scalar vector product (SDOT and UDOT instructions). Probably the most obscure extra flag with doubtful performance benefits
        #          but it has been activated since always, so why not enable it. It's not 100% clear in which revision this flag was
        #          introduced as optional, either in v8.2 [7] or in v8.4 [8].
        # rcpc:    Load-Acquire RCpc Register. Better support of release/acquire of atomics. Good for allocators and high contention code.
        #          Optional in v8.2, mandatory in v8.3 [9]. Supported in Graviton >=2, Azure and GCP instances.
        # bf16:    Bfloat16, a half-precision floating point format developed by Google Brain. Optional in v8.2, mandatory in v8.6.
        #
        # [1]  https://github.com/aws/aws-graviton-getting-started/blob/main/c-c%2B%2B.md
        # [2]  https://community.arm.com/arm-community-blogs/b/tools-software-ides-blog/posts/making-the-most-of-the-arm-architecture-in-gcc-10
        # [3]  https://mysqlonarm.github.io/ARM-LSE-and-MySQL/
        # [4]  https://dev.to/aws-builders/large-system-extensions-for-aws-graviton-processors-3eci
        # [5]  https://developer.arm.com/tools-and-software/open-source-software/developer-tools/llvm-toolchain/sve-support
        # [6]  https://developer.arm.com/documentation/100067/0612/armclang-Command-line-Options/-mcpu?lang=en
        # [7]  https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html
        # [8]  https://developer.arm.com/documentation/102651/a/What-are-dot-product-intructions-
        # [9]  https://developer.arm.com/documentation/dui0801/g/A64-Data-Transfer-Instructions/LDAPR?lang=en
        # [10] https://github.com/aws/aws-graviton-getting-started/blob/main/README.md

        set (COMPILER_FLAGS "${COMPILER_FLAGS} -march=armv8.2-a+simd+crypto+dotprod+ssbs+rcpc+bf16")

        # Not adding `+v8.2a,+crypto` to rust because it complains about them being unstable
        list(APPEND RUSTFLAGS_CPU "-C" "target_feature=+dotprod,+ssbs,+rcpc,+bf16")
    endif ()
```

For ARMv8.2, it adds `-march=armv8.2-a+simd+crypto+dotprod+ssbs+rcpc+bf16`, allowing the compiler to use these instruction-set extensions when building the binary:

```
set (COMPILER_FLAGS "${COMPILER_FLAGS} -march=armv8.2-a+simd+crypto+dotprod+ssbs+rcpc+bf16")
```

This includes the `rcpc` instruction family.

### Summary

The issue occurs because the target CPU does not support the `rcpc` instruction-set extension introduced in ARMv8.2. To resolve it, update the build configuration to disable `rcpc` support.

Disabling `rcpc` may prevent the corresponding optimizations and reduce performance, but it does not affect the other enabled features.
