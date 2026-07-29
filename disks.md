## Related files
`Disks/IVolume.h`: the logical-volume abstraction. It contains disks and the `ROUND_ROBIN` and `LEAST_USED` disk-selection algorithms.

`Disks/VolumeJBOD.h`: implements JBOD (Just a Bunch Of Disks):

- Multiple disks form one volume without RAID or data replication.

- When a new data part is written, a disk is selected according to the load-balancing policy.

`Disks/registerDisks.cpp`: registers every disk type.

`Disks/ObjectStorages/RegisterDiskObjectStorage.h`: registers object-storage disk types and contains the creator.

`Disks/DiskLocal.cpp`: registers local disks.

`Disks/ObjectStorages/ObjectStorageFactory.cpp`: factory for object-storage disks.

`Disks/ObjectStorages/S3/S3ObjectStorage.h`: abstraction for S3 object storage.

`Disks/DiskSelector.cpp`: disk selector containing all created disks.

`Disks/ObjectStorages/MetadataStorageFactory.cpp`: factory for metadata storage.

`Disks/DiskFactory.cpp`: disk factory. Object-storage disks require both object storage and metadata storage.


disk0: MetadataStorageFromPlainObjectStorageTransaction

disk1: MetadataStorageFromDiskTransaction

disk0: MetadataStorageFromPlainObjectStorage

disk1: MetadataStorageFromDisk
