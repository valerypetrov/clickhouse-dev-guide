## Installation
Server:
```
# Running as root is recommended
wget http://dl.minio.org.cn/server/minio/release/linux-amd64/minio
chmod +x minio

# Start the server
./minio server /data

# Optionally move it to /usr/bin
mv minio /usr/bin/
```
Client:
```
wget http://dl.minio.org.cn/client/mc/release/linux-amd64/mc
chmod +x mc

# Optionally move it to /usr/bin
mv minio /usr/bin/
```
Check the server version:
```
root@ubantu64:~# minio --version
minio version RELEASE.2025-07-23T15-54-02Z (commit-id=7ced9663e6a791fef9dc6be798ff24cda9c730ac)
Runtime: go1.24.5 linux/amd64
License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
Copyright: 2015-2025 MinIO, Inc.
```
Check the client version:
```
root@ubantu64:~# mc --version
mc version RELEASE.2025-07-21T05-28-08Z (commit-id=ee72571936f15b0e65dc8b4a231a4dd445e5ccb6)
Runtime: go1.24.5 linux/amd64
Copyright (c) 2015-2025 MinIO, Inc.
License GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
```

Note: If it produces a core dump after downloading, download it again.
