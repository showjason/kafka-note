# Controller

Kafka's controller is the core component that manages and coordinates the entire Kafka cluster. Depending on the deployment mode, the Controller has two implementation approaches:

- **Traditional ZooKeeper Mode**: Relies on external ZooKeeper for metadata management and election
- **KRaft Mode**: Uses built-in Raft protocol without ZooKeeper (Kafka 2.8+ support, 3.3+ production ready, 4.0 completely abandons ZooKeeper)

## Traditional ZooKeeper Mode

### Controller Election and Management

Kafka's controller manages and coordinates the entire Kafka cluster with the help of ZooKeeper. Any broker in the cluster can serve as the controller role, but only one controller can exist at the same time.

The controller election rule is: when a broker starts up, it will try to create the /controller node in ZooKeeper. The first broker that successfully creates the /controller node will be designated as the controller.

The controller actively broadcasts metadata updates to various brokers and sends metadata update requests.

***If the controller has problems, you can delete the /controller node from ZooKeeper to trigger a controller re-election*** `rmr /controller`

## KRaft Mode (ZooKeeper-less Architecture)

### KRaft Controller Architecture

KRaft (Kafka Raft) mode uses a built-in Raft consensus protocol to replace ZooKeeper for metadata management.

**Core Components:**

- **Controller Quorum**: Consists of 3 or 5 Controller nodes
- **Active Controller**: Raft leader, responsible for handling all metadata updates
- **Standby Controllers**: Raft followers, in hot standby state for fast failover
- **Metadata Log**: Stored in a dedicated `__cluster_metadata` internal topic

### KRaft Controller Working Principles

1. **Raft Election**: Uses Raft algorithm to elect leader controller
2. **Metadata Storage**: All cluster metadata stored in a single partition of `__cluster_metadata` topic
3. **State Synchronization**: Controllers maintain state consistency through Raft protocol
4. **Broker Communication**: Brokers pull incremental metadata updates from Controller

### Election Mechanism Comparison

| Feature              | ZooKeeper Mode       | KRaft Mode           |
| -------------------- | -------------------- | -------------------- |
| Election Dependency  | External ZooKeeper   | Built-in Raft Protocol |
| Election Mechanism   | Ephemeral Node Race  | Raft Leader Election |
| Failover Time        | Minutes              | Seconds (Near-instant) |
| Metadata Storage     | ZooKeeper znodes     | Kafka topic          |

## Controller Functions

**Topic Management**: Create, delete topics and add partitions (kafka-topics.sh).

**Partition Reassignment**: kafka-reassign-partitions.sh.

**Preferred Leader Election**: A solution for changing leaders to avoid excessive broker load.

**Cluster Membership Management**:

- **ZooKeeper Mode**: Automatic detection of new brokers, broker active shutdown, and broker failures through Watch functionality and ZooKeeper ephemeral nodes. ZooKeeper's Watch mechanism notifies the Controller of broker changes.
- **KRaft Mode**: Monitor broker status changes through heartbeat mechanisms and metadata logs.

**Data Service**: The controller maintains cluster metadata, and all other brokers periodically receive metadata update requests from the controller to update their in-memory cached data.

## Data Maintained by Controller

1. All partitions and replicas on a specific broker
2. All partitions and replicas for a group of topics
3. All currently alive replicas
4. List of partitions undergoing reassignment
5. All replicas under a group of partitions
6. List of currently alive brokers
7. List of brokers in the process of shutting down
8. Partitions undergoing preferred leader election
9. Replica list assigned to each partition
10. Topic list
11. Leader and ISR information for each partition
12. All information for removing a specific topic

## KRaft vs ZooKeeper Comparison

### Advantages Comparison

| Aspect               | ZooKeeper Mode       | KRaft Mode          | KRaft Advantages      |
| -------------------- | -------------------- | ------------------- | --------------------- |
| **Architecture Complexity** | Manage two systems | Single system      | ✅ Simplified deployment and operations |
| **Scalability**     | ~200K partition limit | Millions of partitions | ✅ Large-scale cluster support |
| **Failure Recovery** | Minutes             | Seconds             | ✅ Fast failover      |
| **Startup Time**     | Load ZK state       | State already in memory | ✅ Fast startup    |
| **Monitoring Complexity** | Two monitoring systems | Unified monitoring | ✅ Simplified monitoring |
| **Security Model**   | Two security configurations | Unified security model | ✅ Simplified security management |

### Performance Comparison

**ZooKeeper Mode Bottlenecks:**

- Controller needs to maintain both ZooKeeper persistent storage and RPC communication to Brokers
- Metadata changes require Controller to write to ZooKeeper then notify each Broker via RPC
- Extensive watch mechanisms and RPC calls introduce additional overhead

**KRaft Mode Improvements:**

- Direct Controller → Broker communication
- Based on Kafka's high-performance log storage
- Event-driven metadata propagation

### Deployment Recommendations

**ZooKeeper Mode:**

```bash
# ZooKeeper cluster configuration (odd number of nodes)
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181
```

**KRaft Mode:**

```bash
# Controller quorum configuration
process.roles=controller  # Dedicated controller node
# or
process.roles=broker,controller  # Combined mode (development environment only)

controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
```

### Migration Considerations

**When to Choose KRaft:**

- ✅ New Kafka cluster deployments (recommended)
- ✅ Need large-scale partition support
- ✅ Want to simplify operational complexity
- ✅ Require fast failure recovery

**When to Keep ZooKeeper:**

- ⚠️ Existing production environments (migration requires caution)
- ⚠️ Other systems dependent on ZooKeeper
- ⚠️ Need certain features not yet supported by KRaft

### Limitations and Considerations

**KRaft Mode Limitations:**

- Combined mode not recommended for production use
- Some management tools may need adaptation
- Migration process requires careful planning

**Version Requirements:**

- Kafka 2.8+: Tech Preview
- Kafka 3.3+: Production ready
- Kafka 3.5+: ZooKeeper officially deprecated 