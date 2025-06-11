# Kafka Topics and Partitions Complete Guide

## Topic Fundamentals

**Topic** is a logical categorization of data streams in Kafka, similar to tables in databases. Each Topic consists of one or more partitions, which are the basic units for Kafka's parallel processing.

**Partition**: The physical division of a Topic. Each partition is an ordered, immutable sequence of messages. Partitions enable Kafka to scale horizontally and provide parallel processing capabilities.

**Replica**: Each partition can have multiple replicas distributed across different brokers (replica count ≤ broker count), providing fault tolerance.

## Partition Count Design Considerations

### More Partitions Enable Higher Throughput

**Throughput-based Partition Count Calculation Formula**:

```
Partition Count = max(Target Throughput/Producer Single-Partition Throughput, Target Throughput/Consumer Single-Partition Throughput)
```

Where:

- **Producer Single-Partition Throughput (p)**: Depends on batch size, compression codec, ack type, replication factor, etc. Modern hardware typically achieves tens of MB/s
- **Consumer Single-Partition Throughput (c)**: Primarily depends on the complexity of application consumption logic

### More Partitions Require More File Handles

Each partition corresponds to a directory in the file system, containing two types of files:

- **Index files**: For quick message location
- **Log files**: Store actual message data

#### Kafka Index Mechanism Details

**Index Types**:

1. **Offset Index (.index)**: Offset-based index for quickly locating messages at specific offsets
2. **Timestamp Index (.timeindex)**: Timestamp-based index for finding messages by time
3. **Transaction Index (.txnindex)**: Index for transactional messages (if transactions are enabled)

**Index Working Principles**:

- **Sparse Index**: Not every message has an index entry; entries are created at intervals configured by `index.interval.bytes` (default 4KB)
- **Binary Search**: Uses binary search algorithm to quickly locate target positions in index files
- **Mapping Relationship**: Index files store mappings of `offset → physical position` or `timestamp → offset`

**Search Process Example**:

```
1. Client requests message at offset=1000
2. Binary search in .index file finds closest index entry: offset=950 → position=12345
3. Sequential scan from log file position=12345 until finding offset=1000
```

**Index File Characteristics**:

- **Fixed-size Entries**: Each index entry is fixed at 8 bytes (4 bytes offset + 4 bytes position)
- **Memory Mapping**: Index files are loaded into memory via mmap for faster access
- **Pre-allocated Space**: Index files pre-allocate fixed space (default 10MB) to avoid frequent expansion

**Performance Advantages**:

- Without index: Finding specific offset requires scanning entire segment (O(n))
- With index: Binary search + minimal sequential scan (O(log n))

**Actual File Example**:

```
/kafka-logs/my-topic-0/
├── 00000000000000000000.index      # Offset index
├── 00000000000000000000.timeindex  # Timestamp index  
├── 00000000000000000000.log        # Log data file
├── 00000000000000001000.index      # Next segment's index
├── 00000000000000001000.timeindex  
└── 00000000000000001000.log
```

**Related Configuration Parameters**:

- `segment.bytes`: Segment size, affects number of index files
- `index.interval.bytes`: Index interval, affects index density and search performance
- `segment.index.bytes`: Maximum index file size (default 10MB)

These files compose a segment. Each broker opens index and data files for every segment of each partition, so file handle usage is determined by:

**File Handle Calculation Formula**:

```
File Handle Count = Partition Count × Segments per Partition × Files per Segment
```

**Files per Segment**:

- **Minimum 3 files**:
  - 1 log file (.log)
  - 1 offset index file (.index)
  - 1 timestamp index file (.timeindex)
- **If transactions enabled**: Additional 1 transaction index file (.txnindex)
- **Actual formula**: Usually `Partition Count × Segments per Partition × 3`

**Influencing Factors**:

- **Partition Count**: More partitions = more file handles
- **segment.size Configuration**: Smaller segments = more segments per partition = higher file handle consumption
- **Data Volume**: With same segment.size, more data = more segments

**Examples**:

- If `segment.size=1GB` and a partition has 10GB data, the partition has ~10 segments, requiring 30 file handles (10 × 3)
- If `segment.size=100MB` with same 10GB data, requires ~100 segments, needing 300 file handles (100 × 3)

**Recommendation**: Balance segment.size settings to avoid excessive file handle consumption from too-small segments, while avoiding too-large segments that impact log compaction and cleanup efficiency.

### More Partitions Lead to Higher Unavailability Risk

**Graceful Shutdown Scenario**:

- Controller proactively migrates leaders away from the shutting-down broker
- Single leader migration takes only milliseconds
- Clients experience minimal impact

**Ungraceful Shutdown Scenario (e.g., kill -9)**:

**Leader Election Mechanism**:

- Controller detects broker failure and elects new leaders for partitions that lost their leader
- Election process: Selects first available replica from ISR (In-Sync Replicas) list as new leader
- **Key Point**: Controller processes leader elections **serially**, handling one partition at a time

**Unavailability Time Impact**:

- **Scenario**: A broker stores 2000 partition replicas (replication factor=2)
- **Leader Distribution**: Under normal conditions, leaders are relatively evenly distributed, so this broker is leader for ~1000 partitions
- **Failure Impact**: When this broker fails, these 1000 partitions simultaneously lose their leader and need re-election
- **Election Time**: If single partition leader election takes 5ms, total time is ~5 seconds for all partitions
- **User Experience**: Affected partitions cannot provide read/write services during election
- **Key Conclusion**: Unavailability time is proportional to the number of partitions this broker leads

**Additional Complexity with Controller Failure**:

- If the failed broker happens to be the controller, impact is more severe
- **Controller Failover Process**:
  1. ZooKeeper detects controller offline (via session timeout)
  2. Other brokers compete to become new controller
  3. New controller needs to rebuild cluster state
- **Metadata Reconstruction**: New controller must read metadata for all partitions from ZooKeeper for initialization
- **Time Estimation**: With 10,000 partitions in cluster, if each partition initialization takes 2ms, initialization alone requires additional 20 seconds
- **Overall Impact**: Controller failover may prevent the entire cluster from performing leader elections temporarily

**Recommendation**: If availability is critical, limit partition count per broker to 2000-4000, and total cluster partitions to tens of thousands.

### More Partitions Increase End-to-End Latency

**End-to-End Latency Definition**: Time from when producer publishes a message to when consumer reads it.

**Latency Causes**:

- Kafka only exposes messages to consumers after they're replicated to all in-sync replicas
- By default, brokers use single thread to replicate data between two brokers for all partitions they share
- Experiments show: Replicating 1000 partitions adds ~20ms latency

**Mitigation Approaches**:

- This issue is alleviated in larger clusters
- Example: With 1000 partition leaders distributed across 10 brokers, each broker only needs to fetch ~100 partitions on average, reducing latency to few milliseconds

**Latency Optimization Formula**:

```
Partition Count Limit per Broker = 100 × Broker Count × Replication Factor
```

### More Partitions Require More Client Memory

**Producer Memory Requirements**:

**Memory Usage Mechanism**:

- Producers maintain independent message buffers for each partition
- Messages accumulate in buffers until batching conditions are met (batch.size or linger.ms)
- When partition count increases, need to maintain buffers for more partitions simultaneously

**Impact of Increased Partitions**:

- **Linear Memory Growth**: Each additional partition requires extra buffer space
- **More Accumulated Messages**: More partitions = more messages accumulated in memory simultaneously
- **Memory Pressure**: Total memory usage = Partition Count × Buffer Size per Partition

**Memory Limit Consequences**:

- **buffer.memory Configuration**: Producer total memory limit (default 32MB)
- **Behavior when limit exceeded**:
  - **Blocking Mode**: If `max.block.ms` > 0, producer blocks waiting for memory release
  - **Exception Mode**: If wait times out, throws `TimeoutException`
  - **Message Loss**: May lead to message loss under certain configurations

**Memory Calculation Example**:

```
Assumptions: batch.size=16KB, 100 partitions
Worst-case memory requirement = 100 × 16KB = 1.6MB (batch buffering only)
Actual requirements include compression, network buffering, etc., typically need 2-3x overhead
```

**Configuration Recommendations**:

- **Basic Configuration**: Allocate at least tens of KB memory per producing partition
- **Memory Planning**: `buffer.memory` ≥ Partition Count × batch.size × 2
- **Monitoring Metrics**: Watch `buffer-available-bytes` and `buffer-exhausted-rate`

**Consumer Memory Requirements**:

- Consumers fetch message batches per partition
- More consumed partitions = more memory needed
- Primarily affects non-real-time consumers

## Common Issues and Solutions

### 1. Topic Deletion Failure

**Common Causes**:

- Brokers hosting replicas are down
- Some partitions of the topic being deleted are undergoing migration

**Solutions**:

- **Broker Down**: Restart the corresponding broker
- **Migration Conflict**: Two operations interfere with each other, more complex to handle

**Universal Solution**:

1. Manually delete ZooKeeper node `/admin/delete_topics` with znode named after the topic to delete
2. Manually delete the topic's partition directories on disk
3. Execute `rmr /controller` in ZooKeeper to trigger controller re-election and refresh controller cache

   > **Note**: Step 3 may cause widespread partition leader re-elections. Actually, executing only the first two steps works fine - the pending deletion topic information in controller cache doesn't affect normal usage.
   >

### 2. `__consumer_offsets` Consuming Too Much Disk Space

**Diagnosis Method**:

```bash
jstack <kafka-pid> | grep "kafka-log-cleaner-thread"
```

**Common Cause**: kafka-log-cleaner-thread is down, unable to clean this internal topic timely

**Solution**: Restart the corresponding broker

## Best Practice Recommendations

### Partition Count Planning

1. **Throughput-Oriented**: Use formula to calculate base partition count
2. **Availability Consideration**: Limit 2000-4000 partitions per broker
3. **Latency-Sensitive**: Use `100 × Broker Count × Replication Factor` formula
4. **Future Expansion**: Consider business growth and reserve appropriate margin

### Monitoring Metrics

- Partition count per broker
- Leader partition distribution balance
- File handle usage
- Replication latency
- Client memory usage

### Performance Tuning

- Adjust single-partition throughput expectations based on hardware capabilities
- Monitor and adjust producer/consumer memory configurations
- Regularly evaluate partition distribution and perform rebalancing

---

> **Reference**: This document is organized and translated based on [Confluent Official Blog: How to Choose the Number of Topics/Partitions in a Kafka Cluster](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/).
