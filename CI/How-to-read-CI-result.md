### Unit test failures
For example: https://s3.amazonaws.com/clickhouse-test-reports/PRs/89432/5ebbe964d9e1b3ce6db54a131643bf5965b25e71//unit_tests_msan/job.log
First search globally for `Error`, then read upward to find the failure log. Identify the specific CI failure that caused the job to fail, for example:
```
[2025-11-07 01:05:20] [----------] 1 test from T64Test
[2025-11-07 01:05:20] [ RUN      ] T64Test.TranscodeRawInput
[2025-11-07 01:05:21] ==27==WARNING: MemorySanitizer: use-of-uninitialized-value
[2025-11-07 01:05:21] [Detaching after fork from child process 12428]
[2025-11-07 01:05:22]     #0 0x5555a89c9503 in DB::(anonymous namespace)::getValuableBitsNumber(long, long) ci/tmp/build/./src/Compression/CompressionCodecT64.cpp:463:17
[2025-11-07 01:05:22]     #1 0x5555a89c9503 in unsigned int DB::(anonymous namespace)::compressData<short, false>(char const*, unsigned int, char*) ci/tmp/build/./src/Compression/CompressionCodecT64.cpp:523:23
[2025-11-07 01:05:22]     #2 0x5555a89c9503 in unsigned int DB::(anonymous namespace)::compressData<short>(char const*, unsigned int, char*, DB::CompressionCodecT64::Variant) ci/tmp/build/./src/Compression/CompressionCodecT64.cpp:644:12
[2025-11-07 01:05:22]     #3 0x5555a89c9503 in DB::CompressionCodecT64::doCompressData(char const*, unsigned int, char*) const ci/tmp/build/./src/Compression/CompressionCodecT64.cpp:669:24
[2025-11-07 01:05:22]     #4 0x5555a8ac39f4 in DB::ICompressionCodec::compress(char const*, unsigned int, char*) const ci/tmp/build/./src/Compression/ICompressionCodec.cpp:95:39
[2025-11-07 01:05:22]     #5 0x55555f8a0650 in (anonymous namespace)::T64Test_TranscodeRawInput_Test::TestBody() ci/tmp/build/./src/Compression/tests/gtest_compressionCodec.cpp:1398:38
[2025-11-07 01:05:22]     #6 0x55557d8e5538 in void testing::internal::HandleSehExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2653:10
[2025-11-07 01:05:22]     #7 0x55557d8e5538 in void testing::internal::HandleExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2689:14
[2025-11-07 01:05:22]     #8 0x55557d8e5170 in testing::Test::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2728:5
[2025-11-07 01:05:22]     #9 0x55557d8e83bd in testing::TestInfo::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2874:11
[2025-11-07 01:05:22]     #10 0x55557d8eb8dd in testing::TestSuite::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:3052:30
[2025-11-07 01:05:22]     #11 0x55557d92761b in testing::internal::UnitTestImpl::RunAllTests() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:5982:44
[2025-11-07 01:05:22]     #12 0x55557d9264f8 in bool testing::internal::HandleSehExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2653:10
[2025-11-07 01:05:22]     #13 0x55557d9264f8 in bool testing::internal::HandleExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2689:14
[2025-11-07 01:05:22]     #14 0x55557d92612f in testing::UnitTest::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:5561:10
[2025-11-07 01:05:22]     #15 0x55555f54df64 in RUN_ALL_TESTS() ci/tmp/build/./contrib/googletest/googletest/include/gtest/gtest.h:2334:73
[2025-11-07 01:05:22]     #16 0x55555f54df64 in main ci/tmp/build/./src/Common/tests/gtest_main.cpp:17:12
[2025-11-07 01:05:22]     #17 0x7ffff7da8d8f  (/lib/x86_64-linux-gnu/libc.so.6+0x29d8f) (BuildId: 4f7b0c955c3d81d7cac1501a2498b69d1d82bfe7)
[2025-11-07 01:05:22]     #18 0x7ffff7da8e3f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x29e3f) (BuildId: 4f7b0c955c3d81d7cac1501a2498b69d1d82bfe7)
[2025-11-07 01:05:22]     #19 0x55555eaab7ed in _start (/home/ubuntu/actions-runner/_work/ClickHouse/ClickHouse/ci/tmp/unit_tests_dbms+0x95577ed) (BuildId: 75361b4e828a0383eb8bb5dfb9dbd2c795ca5ff9)
[2025-11-07 01:05:22] 
[2025-11-07 01:05:22]   Uninitialized value was created by a heap allocation
[2025-11-07 01:05:22]     #0 0x55555eada632 in malloc (/home/ubuntu/actions-runner/_work/ClickHouse/ClickHouse/ci/tmp/unit_tests_dbms+0x9586632) (BuildId: 75361b4e828a0383eb8bb5dfb9dbd2c795ca5ff9)
[2025-11-07 01:05:22]     #1 0x5555aaefaef8 in void* (anonymous namespace)::allocNoTrack<false, false>(unsigned long, unsigned long) ci/tmp/build/./src/Common/Allocator.cpp:86:19
[2025-11-07 01:05:22]     #2 0x5555aaefaef8 in Allocator<false, false>::alloc(unsigned long, unsigned long) ci/tmp/build/./src/Common/Allocator.cpp:133:18
[2025-11-07 01:05:22]     #3 0x55555f89ff43 in DB::Memory<Allocator<false, false>>::alloc(unsigned long) ci/tmp/build/./src/IO/BufferWithOwnMemory.h:135:49
[2025-11-07 01:05:22]     #4 0x55555f89ff43 in DB::Memory<Allocator<false, false>>::resize(unsigned long, bool) ci/tmp/build/./src/IO/BufferWithOwnMemory.h:84:13
[2025-11-07 01:05:22]     #5 0x55555f89ff43 in (anonymous namespace)::T64Test_TranscodeRawInput_Test::TestBody() ci/tmp/build/./src/Compression/tests/gtest_compressionCodec.cpp:1388:27
[2025-11-07 01:05:22]     #6 0x55557d8e5538 in void testing::internal::HandleSehExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2653:10
[2025-11-07 01:05:22]     #7 0x55557d8e5538 in void testing::internal::HandleExceptionsInMethodIfSupported<testing::Test, void>(testing::Test*, void (testing::Test::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2689:14
[2025-11-07 01:05:22]     #8 0x55557d8e5170 in testing::Test::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2728:5
[2025-11-07 01:05:22]     #9 0x55557d8e83bd in testing::TestInfo::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2874:11
[2025-11-07 01:05:22]     #10 0x55557d8eb8dd in testing::TestSuite::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:3052:30
[2025-11-07 01:05:22]     #11 0x55557d92761b in testing::internal::UnitTestImpl::RunAllTests() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:5982:44
[2025-11-07 01:05:22]     #12 0x55557d9264f8 in bool testing::internal::HandleSehExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2653:10
[2025-11-07 01:05:22]     #13 0x55557d9264f8 in bool testing::internal::HandleExceptionsInMethodIfSupported<testing::internal::UnitTestImpl, bool>(testing::internal::UnitTestImpl*, bool (testing::internal::UnitTestImpl::*)(), char const*) ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:2689:14
[2025-11-07 01:05:22]     #14 0x55557d92612f in testing::UnitTest::Run() ci/tmp/build/./contrib/googletest/googletest/src/gtest.cc:5561:10
[2025-11-07 01:05:22]     #15 0x55555f54df64 in RUN_ALL_TESTS() ci/tmp/build/./contrib/googletest/googletest/include/gtest/gtest.h:2334:73
[2025-11-07 01:05:22]     #16 0x55555f54df64 in main ci/tmp/build/./src/Common/tests/gtest_main.cpp:17:12
[2025-11-07 01:05:22]     #17 0x7ffff7da8d8f  (/lib/x86_64-linux-gnu/libc.so.6+0x29d8f) (BuildId: 4f7b0c955c3d81d7cac1501a2498b69d1d82bfe7)
[2025-11-07 01:05:22] 
[2025-11-07 01:05:23] SUMMARY: MemorySanitizer: use-of-uninitialized-value ci/tmp/build/./src/Compression/CompressionCodecT64.cpp:463:17 in DB::(anonymous namespace)::getValuableBitsNumber(long, long)
[2025-11-07 01:05:23] Exiting
[2025-11-07 01:05:23] [Inferior 1 (process 27) exited with code 01]
[2025-11-07 01:05:23] No stack.
[2025-11-07 01:05:23] ERROR: No test result file [/home/ubuntu/actions-runner/_work/ClickHouse/ClickHouse/ci/tmp/gtest.json]
[2025-11-07 01:05:23] ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
[2025-11-07 01:05:23] error [Unit tests (msan)]
[2025-11-07 01:05:23]   | No test result file [/home/ubuntu/actions-runner/_work/ClickHouse/ClickHouse/ci/tmp/gtest.json]
[2025-11-07 01:05:23]   | To run locally: python -m ci.praktika run "Unit tests (msan)" --test TEST_NAME_1..TEST_NAME_N
[2025-11-07 01:05:23] ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
[2025-11-07 01:05:23] 
[2025-11-07 01:05:23] ERROR: Run failed with exit code [1]
```
