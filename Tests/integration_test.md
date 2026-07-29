
### Create a Virtual Environment
Create a virtual environment in `/root/work/ClickHouse/tests/integration`:
```
python3 -m venv integration-venv
```
Inspect the created environment:
```
find . -type d -name "venv" -o -name ".venv" 2>/dev/null
```
Activate the virtual environment:
```
source venv/bin/activate
```
### Prerequisites

- Ubuntu 20.04 (Focal) or later

- Docker (API version 1.25 or later; check with `docker version`)

- Install the latest version of Docker by following the official documentation. Do not use the Docker package from the system repository.
- Install pip and libpq-dev:
```
sudo apt-get install python3-pip libpq-dev zlib1g-dev libcrypto++-dev libssl-dev libkrb5-dev python3-dev openjdk-17-jdk requests urllib3
```
- Install the pytest testing framework:
```
sudo -H pip install pytest
```
- Install Docker Compose and the other Python libraries:
```
sudo -H pip install \
    PyMySQL avro cassandra-driver confluent-kafka dicttoxml docker grpcio grpcio-tools kafka-python kazoo minio \
    lz4 protobuf psycopg2-binary pymongo pytz pytest pytest-timeout redis tzlocal==2.1 urllib3 requests-kerberos dict2xml \
    hypothesis pika nats-py pandas numpy jinja2 pytest-xdist==2.4.0 pyspark azure-storage-blob delta paramiko psycopg pyarrow boto3 deltalake snappy pyiceberg python-snappy thrift
```
You can use the Tsinghua University mirror:
```
# Use the Tsinghua University mirror
sudo pip install \
    PyMySQL avro cassandra-driver confluent-kafka dicttoxml docker grpcio grpcio-tools kafka-python kazoo minio lz4 protobuf psycopg2-binary pymongo pytz pytest pytest-timeout redis tzlocal==2.1 urllib3 requests-kerberos dict2xml hypothesis pika nats-py pandas numpy jinja2 pytest-xdist==2.4.0 pyspark azure-storage-blob delta paramiko psycopg pyarrow boto3 deltalake snappy pyiceberg python-snappy thrift \
    -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn --timeout=300
```
You may also need to install the following package:
```
# Install the krb5 development package
sudo apt-get install libkrb5-dev krb5-config -y
```
Use `pip list` to view the packages installed in the current virtual environment:
```
pip list
```
- For Spark tests: install Spark and add its `bin` directory to the `PATH` environment variable. See `ci/docker/integration/runner/Dockerfile` for details. Set `JAVA_PATH` to the path of the Java binary.

- To run tests as a non-privileged user, add the user to the `docker` group:
```
sudo usermod -aG docker $USER
# Log in again or restart the computer
```

### Run the Tests
Run tests with pytest. The following example shows commonly used options:
```
pytest <test-or-path> \
  [-k <expression>] \
  [-n <process-count> --dist=loadfile] \
  [--count <number-of-runs> --repeat-scope=function]
```
Option descriptions:

`-n <process-count>`: The number of parallel processes (CI uses 4 for tests that can run in parallel and 1 for other tests). When running tests in parallel, use `--dist=loadfile` to keep tests from the same file in the same worker process. See pytest-xdist for details.

`--count <number-of-runs>`: Run each test multiple times; use it with `--repeat-scope=function`. See pytest-repeat for details.

The tests look for the server binary in the repository root and configuration files in `./programs/server/`. Override these paths with the following environment variables:
- CLICKHOUSE_TESTS_SERVER_BIN_PATH

- CLICKHOUSE_TESTS_ODBC_BRIDGE_BIN_PATH (path to the clickhouse-odbc-bridge binary)

- CLICKHOUSE_TESTS_CLIENT_BIN_PATH

- CLICKHOUSE_TESTS_BASE_CONFIG_DIR (path to config.xml and users.xml)

Build notes:
If you use a partial build (`ENABLE_CLICKHOUSE_ALL=OFF`), build all required components (for example, with `ENABLE_CLICKHOUSE_KEEPER=ON`). Using `ENABLE_CLICKHOUSE_ALL=ON` is simpler.

### What If a Slow Network Prevents Docker from Pulling Images?
Download all Docker images required by the tests in advance.

### Build a Docker Image to Run Integration Tests Locally
```
root@ubantu64:~/work/ClickHouse/docker/server# pwd
/root/work/ClickHouse/docker/server
root@ubantu64:~/work/ClickHouse/docker/server# docker build -t clickhouse/integration-test:latest -f Dockerfile.ubuntu .
```
```
root@ubantu64:~/work/ClickHouse/docker/server# docker images|grep inte
clickhouse/integration-test                                 latest    7153119c0a63   11 minutes ago   828MB
```
You may need `ubuntu:22.04`:
```
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/ubuntu:22.04
```
Tag it with the name required by `DockerFile`:
```
docker tag swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/ubuntu:22.04 ubuntu:22.04
```

1️⃣ Clean Up Leftover Containers
```
export PYTEST_CLEANUP_CONTAINERS=1
pytest --cleanup-containers
```
