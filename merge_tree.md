## Interfaces
`Storages/MergeTree/IDataPartStorage.h`: the abstract interface for the data-part storage layer. It hides differences between underlying storage systems such as local disks, local JBOD, multi-volume policies, and S3/MinIO object storage.

`Storages/MergeTree/DataPartStorageOnDiskFull.h`: `DataPartStorageOnDiskFull` is a storage implementation for MergeTree data parts. It operates directly on files, treating a part much like a filesystem tree.

`Storages/MergeTree/MergeTreeDataPartBuilder.h`:

### Selecting a disk for writes
```
StackTrace::StackTrace() @ 0x00000000103f2de4
1. DB::IVolume::IVolume(String, std::vector<std::shared_ptr<DB::IDisk>, std::allocator<std::shared_ptr<DB::IDisk>>>, unsigned long, bool, DB::VolumeLoadBalancing) @ 0x000000001580bc16
2. DB::SingleDiskVolume::SingleDiskVolume(String const&, SingleDiskVolumestd::shared_ptr<DB::IDisk>, unsigned long) @ 0x000000001580b84b
3. std::shared_ptr<DB::SingleDiskVolume> std::allocate_shared[abi:ne190107]<DB::SingleDiskVolume, std::allocator<DB::SingleDiskVolume>, String const&, std::shared_ptr<DB::IDisk>, unsigned long&, 0>(std::allocator<DB::SingleDiskVolume> const&, String const&, std::shared_ptr<DB::IDisk>&&, unsigned long&) @ 0x0000000015855e14
4. DB::createVolumeFromReservation(std::unique_ptr<DB::IReservation, std::default_delete<DB::IReservation>> const&, std::shared_ptr<DB::IVolume>) @ 0x0000000015854d74
5. DB::MergeTreeDataWriter::writeTempPartImpl(DB::BlockWithPartition&, std::shared_ptr<DB::StorageInMemoryMetadata const>, String, DB::SourcePartsSetForPatch, std::shared_ptr<DB::Context const>, unsigned long) @ 0x0000000019af8620
6. DB::MergeTreeDataWriter::writeTempPart(DB::BlockWithPartition&, std::shared_ptr<DB::StorageInMemoryMetadata const>, std::shared_ptr<DB::Context const>) @ 0x0000000019af6d4f
7. DB::MergeTreeSink::consume(DB::Chunk&) @ 0x0000000019c4c2a3
8. DB::SinkToStorage::onConsume(DB::Chunk) @ 0x000000001a560f13
9. void std::__function::__policy_invoker<void ()>::__call_impl[abi:ne190107]<std::__function::__default_alloc_func<DB::ExceptionKeepingTransform::work()::$_1, void ()>>(std::__function::__policy_storage const*) @ 0x000000001a46ad76
10. DB::runStep(std::function<void ()>, std::shared_ptr<DB::ThreadGroup>&) @ 0x000000001a46ab85
11. DB::ExceptionKeepingTransform::work() @ 0x000000001a46a67e
12. DB::ExecutionThreadContext::executeTask() @ 0x000000001a1efd99
13. DB::PipelineExecutor::executeStepImpl(unsigned long, DB::IAcquiredSlot*, std::atomic<bool>*) @ 0x000000001a1e3db8
14. DB::PipelineExecutor::executeStep(std::atomic<bool>*) @ 0x000000001a1e33f0
15. DB::TCPHandler::processInsertQuery(DB::QueryState&) @ 0x000000001a112883
16. DB::TCPHandler::runImpl() @ 0x000000001a105ef1
17. DB::TCPHandler::run() @ 0x000000001a122696
18. Poco::Net::TCPServerConnection::start() @ 0x00000000208303c7
19. Poco::Net::TCPServerDispatcher::run() @ 0x000000002083097e
20. Poco::PooledThread::run() @ 0x00000000207cfbbf
21. Poco::ThreadImpl::runnableEntry(void*) @ 0x00000000207cd1cf

```

```
reserveSpacePreferringTTLRules  ->
  tryReserveSpacePreferringTTLRules  ->
    Usually follows this path:
    if (!reservation)
    {
        LOG_TRACE(log, "Trying to reserve {} using storage policy from min volume index {}", ReadableSize(expected_size), min_volume_index);
        reservation = getStoragePolicy()->reserve(expected_size, min_volume_index).value_or(nullptr);
    }

    StoragePolicy::reserve  ->
      VolumeJBOD::reserve  -> # or another volume
        ReservationPtr VolumeJBOD::reserve(UInt64 bytes)
        {
            /// This volume can not store data which size is greater than `max_data_part_size`
            /// to ensure that parts of size greater than that go to another volume(s).
            if (max_data_part_size != 0 && bytes > max_data_part_size)
                return {};
        
            switch (load_balancing)
            {
                case VolumeLoadBalancing::ROUND_ROBIN:
                {
                    size_t disks_num = disks.size();
                    for (size_t i = 0; i < disks_num; ++i)
                    {
                        size_t start_from = last_used.fetch_add(1u, std::memory_order_acq_rel);
                        size_t index = start_from % disks_num;
        
                        if (disks[index]->isReadOnly())
                            continue;
        
                        auto reservation = disks[index]->reserve(bytes);
        
                        if (reservation)
                            return reservation;
                    }
                    return {};
                }
                case VolumeLoadBalancing::LEAST_USED:
                {
                    std::lock_guard lock(mutex);
        
                    ReservationPtr reservation;
                    for (size_t i = 0; i < disks.size() && !reservation; ++i)
                    {
                        if (i == 0 && (!least_used_ttl_ms || least_used_update_watch.elapsedMilliseconds() >= least_used_ttl_ms))
                        {
                            disks_by_size = LeastUsedDisksQueue(disks.begin(), disks.end());
                            least_used_update_watch.restart();
        
                            DiskWithSize disk = disks_by_size.top();
                            if (!disk.disk->isReadOnly())
                                reservation = disk.reserve(bytes);
                        }
                        else
                        {
                            DiskWithSize disk = disks_by_size.top();
                            disks_by_size.pop();
        
                            if (!disk.disk->isReadOnly())
                                reservation = disk.reserve(bytes);
                            disks_by_size.push(disk);
                        }
                    }
        
                    return reservation;
                }
            }
        }
        
```
