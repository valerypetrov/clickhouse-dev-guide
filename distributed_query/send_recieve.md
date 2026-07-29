## Data receiver
The node responsible for distributed-query execution receives data from the other shards.

It uses `RemoteSource` to receive data:
```
RemoteSource::tryGenerate()  ->
  RemoteQueryExecutor::readAsync()  ->
    RemoteQueryExecutor::processPacket(Packet packet)
```
In general, a distributed query involves four packet types:

1. `Protocol::Server::Data`: a data packet.

2. `Protocol::Server::Progress`: records read-progress statistics.

3. `Protocol::Server::ProfileEvents`: records performance events from the remote server.

4. `Protocol::Server::EndOfStream`: indicates that the peer has no more data.

The transmitted data is represented as a `Block`.

## Data sender
```
TCPHandler::sendData(QueryState & state, const Block & block)
```
