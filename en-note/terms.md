# Kafka Core Terms and Concepts

## 1. Cluster Architecture

### Cluster
A Kafka cluster is a distributed system composed of multiple Brokers, providing high availability and horizontal scaling capabilities.

### Broker
A Broker is a server node in the Kafka cluster responsible for storing data and handling client requests. Each Broker has a unique ID, and a Kafka cluster consists of multiple Brokers.

## 2. Storage Architecture

### Topic
A Topic is a logical categorization of messages. Each topic can be configured with M partitions, and each partition can be configured with N replicas.

### Partition
A partition is the physical division unit of a topic. Among the N replicas of each partition, only one can be the Leader, which handles all external services (the Kafka team is working on supporting closest replica reads instead of only Leader supporting read/write operations). Other replicas (Followers) synchronize data from the Leader to provide data redundancy.

### Message
A partition contains multiple messages, with each message's offset starting from 0 and incrementing sequentially.

## 3. Replication Mechanism

### Leader/Follower
- **Leader**: The primary replica of each partition, responsible for handling all read and write requests
- **Follower**: Secondary replicas that synchronize data from the Leader, providing data redundancy

### ISR (In-Sync Replicas)
The set of synchronized replicas that contains all replicas keeping in sync with the Leader replica (including the Leader itself).

#### Conditions for Replicas to Join ISR
1. **Time Synchronization Requirement**: Follower replica's data lag time must not exceed `replica.lag.time.max.ms` (default 10 seconds)
2. **Connection Status Requirement**:
   - In ZooKeeper-managed clusters: The Broker hosting the replica must maintain heartbeat connection with ZooKeeper
   - In KRaft mode clusters: The Broker hosting the replica must maintain heartbeat connection with the Controller
3. **Active Synchronization**: Followers must continuously send Fetch requests to the Leader to synchronize data

#### Conditions for Replicas to be Removed from ISR
1. **Excessive Synchronization Lag**: Follower data falls behind beyond the `replica.lag.time.max.ms` time threshold
2. **Node Disconnection**: Broker loses connection with the cluster (heartbeat timeout)
3. **Synchronization Errors**: Replica encounters errors during data synchronization process

#### Leader Election Mechanism
**Standard Situation**: Only replicas in the ISR are eligible to be elected as the new Leader, ensuring the new Leader has the latest committed data and avoiding data loss.

**Special Situation**: When `unclean.leader.election.enable=true`, if all ISR replicas are unavailable, Kafka allows electing a Leader from non-ISR replicas. However, this carries **data loss risks** because non-ISR replicas may be missing some committed messages.

#### ISR Maintenance Mechanism
ISR list maintenance involves multiple components and has undergone significant changes across Kafka versions:

**Monitoring and Judgment**:
- **Leader replica** is responsible for monitoring the synchronization status of Followers
- Leader tracks synchronization status based on Fetch requests sent by Followers

**Status Update and Persistence**:
- **Before Kafka 2.7**: Leader directly updates ZooKeeper state nodes `/brokers/topics/[topic]/partitions/[partitionId]/state`
- **Kafka 2.7 and later**: According to KIP-497, Leader sends `AlterISR` requests to **Controller**, and Controller is responsible for ISR state changes and persistence

**Controller's Role**:
- Receives and processes `AlterISR` requests sent by Leaders
- Validates the legitimacy of ISR changes (such as checking if Brokers are online)
- Persists ISR changes to cluster metadata storage
- Sends `LeaderAndIsr` requests to relevant Brokers to synchronize the latest status

**Real-time Monitoring**: ISR status changes can be monitored in real-time through JMX metrics

## 4. Clients

### Producer
A Producer is a client application that sends messages to Kafka topics. The Producer is responsible for selecting which partition of a topic to send messages to.

### Consumer
A Consumer is a client application that reads messages from Kafka topics. A consumer can be a process or a thread.

### Consumer Group
A consumer group consists of multiple consumer instances forming a group to consume a set of topics. Each partition in these topics can only be consumed by one consumer in this consumer group, not by multiple consumers in the same consumer group. This mechanism improves consumer-side throughput (TPS).

### Group Coordinator
The Broker responsible for managing consumer groups, handling consumer joining, leaving, and rebalancing coordination.

## 5. Offsets and Consumption

### Offset
- **Partition Offset**: The position of a message in a partition; once a message is written, its position remains unchanged
- **Consumer Offset**: The consumer's consumption progress, which changes constantly

### High Watermark (HW)
The high watermark is the core mechanism for Kafka to guarantee data consistency, representing the maximum message offset that has been synchronized by all ISR replicas.

#### Basic Concepts
- **Definition**: The offset of the last message in a partition that has been replicated by all ISR replicas
- **Purpose**: Ensures consumers can only read messages that have been fully replicated, avoiding data inconsistency

#### Relationship between HW and LEO
- **LEO (Log End Offset)**: The log end offset, representing the offset of the last message in each replica
- **Relationship**: HW ≤ min(LEO of all ISR replicas)

#### HW Update Mechanism
**Leader's HW Update**:
1. Leader receives Fetch requests from Followers, learning each Follower's LEO
2. Calculates the minimum LEO value among all ISR replicas
3. Updates HW to this minimum value

**Follower's HW Update**:
1. Follower sends Fetch request to Leader
2. Leader includes current HW value in Fetch response
3. Follower updates its own HW (usually slightly behind the Leader)

#### Data Visibility Rules
- **Consumers**: Can only read messages before HW, ensuring all read data is committed
- **Producers**: Waiting strategy depends on `acks` configuration
  - `acks=1`: Returns after Leader writes, doesn't wait for HW update
  - `acks=all`: Returns after all ISR replicas confirm

#### HW in Failure Recovery
When Leader fails:
1. After new Leader election completes, HW may roll back (due to HW synchronization delays)
2. Followers truncate log portions higher than the new Leader's HW
3. Ensures data consistency across all replicas

#### Configuration Parameters
- `replica.lag.time.max.ms`: Affects ISR membership, indirectly affecting HW calculation
- `min.insync.replicas`: Affects data commit safety requirements

### Commit (Offset Commit)
The operation where consumers save their consumption progress (offset) to Kafka's internal topic `__consumer_offsets`.

## 6. Data Persistence

### Log
Kafka uses message logs to store data. A log is a physical file on disk that only allows append-only message operations. Since it only allows appending, it avoids slow random I/O operations and uses better-performing sequential I/O write operations, which is an important factor in achieving Kafka's high throughput.

### Log Segment
At the underlying Kafka level, a log is further divided into multiple log segments. Messages are appended to the latest log segment. When a log segment is full, Kafka automatically creates a new log segment and seals the old one. At the same time, Kafka periodically checks whether old log segments can be deleted to achieve regular disk space recovery.

## 7. Operations Concepts

### Rebalance
Rebalancing refers to the process where consumer instances within a consumer group automatically redistribute subscribed topic partitions when the following situations occur:
- A consumer instance in the consumer group fails
- A new consumer joins the consumer group
- The number of partitions changes
- The number of topics changes

### Partition Assignment Strategies
- **Range**: Assigns partition ranges by topic individually
- **RoundRobin**: Round-robin assignment of all topic partitions
- **Sticky**: Maintains original assignments as much as possible to reduce rebalancing overhead 