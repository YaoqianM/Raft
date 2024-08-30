# raft-java
A Raft consensus algorithm implementation library for Java.

Based on: The Raft paper ([https://github.com/maemual/raft-zh_cn]) and the Raft author's open-source implementation, LogCabin ([https://github.com/logcabin/logcabin]).

# Supported Features:
* Leader election
* Log replication
* Snapshots
* Dynamic membership changes
* Quick Start

## Quick Start
To deploy a 3-node Raft cluster locally, execute the following script:
cd raft-java-example && sh deploy.sh
Use code with caution.

This script will deploy three instances (example1, example2, example3) in the raft-java-example/env directory and create a client directory for testing.

After successful deployment, test a write operation with the following script:

cd env/client
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world

To test a read operation:

./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello

# Explain
Here's how to use the Raft-Java library to implement a distributed storage system.

## Configure dependencies (not yet published to the central maven repository, need to manually install locally)
XML
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>   


## Define the data write and read interfaces
```protobuf
message SetRequest {
    string key = 1; string value = 2; string
    string value = 2; 
}
message SetResponse {
    bool success = 1; 
}
message GetRequest {
    string key = 1; 
}
message GetResponse {
    string value = 1; 
}
```
```java
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
```

## Server-side usage
1. Implement the StateMachine interface implementation class.
```java
// The three methods of this interface are mainly for internal Raft calls.
public interface StateMachine {
    /**
     * Snapshot the data in the state machine, called locally on each node at regular intervals.
     * @param snapshotDir snapshot data output directory
     */
    void writeSnapshot(String snapshotDir);
    /**
     * Read snapshot to state machine, called when node startup.
     * @param snapshotDir snapshot data directory
     */
    void readSnapshot(String snapshotDir);
    /**
     * Apply data to state machine.
     * @param dataBytes dataBinary
     */
    void apply(byte[] dataBytes);
}
```

2. Implementing the data write and read interfaces
```
// The ExampleService implementation class needs to contain the following members

private RaftNode raftNode; 
private ExampleStateMachine stateMachine; 
```
``` 
// The main logic for writing the data
byte[] data = request.toByteArray(); 
// Synchronize data writing to the raft cluster.
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA); 
// Synchronize the data write to the raft cluster.
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
```
```
// The main logic for reading the data is implemented by the application-specific state machine.
Example.GetResponse response = stateMachine.get(request);
```

3. server-side startup logic
```
// Initialize the RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort()); 
// Apply the state machine
ExampleStateMachine stateMachine = new ExampleStateMachine(); 
// Set the Raft options, e.g:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize the RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine); 
// Initialize the RaftNode with a new RaftNode.Register the services that Raft nodes call each other with

RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode); 
// register the service to be invoked between Raft nodes; 
server.registerService(raftConsensusService); 
// Register the Raft service for the Client to call.
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode); server.registerService(raftConsensusService); 
// Register the Raft service for the Client to call.
server.registerService(raftClientService); 
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine); server.registerService(exampleService); 
// Register the service provided by the application.
server.registerService(exampleService); 
// Start the RPCServer and initialize the Raft node.
server.start();
raftNode.init();
``
