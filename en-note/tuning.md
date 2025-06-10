# Kafka Performance Tuning

## Overview

The core objectives of Kafka tuning are to **improve throughput** and **reduce latency**, which often involve trade-offs between these two goals.

**Latency Definition**:

- Producer latency: Time interval from message sending to Broker persistence completion
- End-to-end latency: Total time from Producer sending message to Consumer successful consumption

## Operating System Level Tuning

### File System Optimization

```bash
# 1. Disable atime updates to reduce unnecessary write operations
mount -o noatime /dev/sda1 /kafka-logs

# 2. Adjust system parameters
ulimit -n 1000000                    # Increase file descriptor limit
echo 262144 > /proc/sys/vm/max_map_count  # Avoid memory mapping limits
```

### Memory Management

- **Page Cache**: Should contain at least one log segment size (default 1GB)
- **Swappiness**: Set to 1-10 to avoid frequent swapping

```bash
echo 1 > /proc/sys/vm/swappiness
```

### Disk I/O Optimization

- Use SSD for storing log files
- Separate different types of disks: OS, ZooKeeper, Kafka logs

## JVM Tuning

### Heap Memory Configuration

```bash
# Recommended heap size: 6GB ~ 8GB
-Xms6g -Xmx6g

# Calculation formula: Size of surviving objects after Full GC × 1.5~2
# Use the following command to trigger Full GC for observation
jmap -histo:live <pid>
```

### GC Collector Selection

```bash
# Recommend using G1 collector
-XX:+UseG1GC
-XX:MaxGCPauseMillis=20
-XX:+PrintAdaptiveSizePolicy  # Help troubleshoot Full GC issues

# Solve large object problems
-XX:G1HeapRegionSize=32m     # By default, objects larger than N/2 are large objects
```

### Common GC Issues

- **Error Message**: "too many humongous allocations"
- **Solutions**:
  1. Increase heap size
  2. Adjust G1HeapRegionSize
  3. Optimize message body size

## Broker-side Tuning

### Version Compatibility

⚠️ **Important**: Keep client and Broker versions consistent to enjoy Zero Copy optimization

### Key Parameter Configuration

```properties
# Number of network threads (usually equal to CPU cores)
num.network.threads=8

# Number of I/O threads (usually 2x CPU cores)
num.io.threads=16

# Socket buffer size
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576

# Log segment size
log.segment.bytes=1073741824

# Number of replica fetcher threads
num.replica.fetchers=4
```

### Log Segment Size Tuning

`log.segment.bytes` controls the size of each log segment file and needs to be balanced based on actual scenarios.

#### Configuration Principles

**Not smaller is better**, need to consider the following factors:

**Problems with setting too small**:

- **File count explosion**: Too many small files lead to insufficient file descriptors
- **Metadata overhead**: Each file needs to maintain metadata, increasing memory and disk overhead
- **Disk fragmentation**: Small files cause increased disk fragmentation
- **Inefficient log cleanup**: Compression and deletion operations need to handle many small files

**Problems with setting too large**:

- **Slow startup recovery**: Broker restart requires rebuilding indexes for large files
- **Memory pressure**: Index files corresponding to large segment files occupy more memory
- **Log cleanup delays**: Only when entire segments expire can they be deleted, affecting timely data cleanup

#### Recommended Configuration

```properties
# Default setting, suitable for most scenarios
log.segment.bytes=1073741824  # 1GB

# High throughput scenarios
log.segment.bytes=2147483648  # 2GB

# Low throughput or small message scenarios
log.segment.bytes=536870912   # 512MB
```

#### Tuning Considerations

**1. Data Retention Time**

```bash
# If data is retained for 7 days, segment size should allow logs to be reasonably distributed across multiple segments
# Avoid single segments containing too many days of data
```

**2. Message Characteristics**

- **High throughput + Large messages** → Use larger segments (1-2GB)
- **Low throughput + Small messages** → Use smaller segments (512MB-1GB)

**3. Hardware Resources**

- **Sufficient memory** → Can use large segments (indexes occupy more memory)
- **Good disk performance** → Can use large segments (better sequential write performance)

#### Monitoring Commands

```bash
# View number of segment files per partition
ls -la /kafka-logs/topic-partition-0/ | wc -l

# Monitor file descriptor usage
lsof -p <broker-pid> | wc -l

# View segment file size distribution
du -sh /kafka-logs/topic-*/*.log

# Check if file descriptor limit is exceeded
ulimit -n
```

## Application Layer Tuning

### Client Reuse

```java
// ✅ Correct: Reuse Producer instance
static KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// ❌ Wrong: Create new instance each time
// KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

### Resource Management

- Producer is thread-safe and can be shared among multiple threads
- Consumer is not thread-safe, each thread needs an independent instance
- Close clients promptly to release resources (Socket connections, ByteBuffer, etc.)

### Multi-threaded Consumption Patterns

```java
// Option 1: Multiple Consumer instances
ExecutorService executor = Executors.newFixedThreadPool(10);
for (int i = 0; i < 10; i++) {
    executor.submit(new ConsumerWorker());
}

// Option 2: Single Consumer + multi-threaded processing
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        executor.submit(() -> processRecord(record));
    }
}
```

## Performance Metrics Tuning

### Throughput Optimization Principles

Improve throughput through **batch processing**:

**Example Comparison**:

**Scenario 1: Single Message Sending**

- Send latency per message: 2ms
- Messages that can be sent per second: 1000ms ÷ 2ms = 500 messages
- **TPS = 500/s**

**Scenario 2: Batch Sending**

- Wait time: 8ms (accumulate messages)
- Send latency: 2ms (network transmission)
- Total latency: 8ms + 2ms = 10ms
- Batch size: 1000 messages
- Batches completed per second: 1000ms ÷ 10ms = 100 batches
- Messages sent per second: 100 batches × 1000 messages = 100,000 messages
- **TPS = 100,000/s**

**Key insight**: Although single message latency increases from 2ms to 10ms, overall throughput improves by 200x through batch processing!

### Throughput Tuning Parameters

| Component          | Parameter                | Recommended Value   | Description                                       |
| ------------------ | ------------------------ | ------------------- | ------------------------------------------------- |
| **Broker**   | `num.replica.fetchers` | CPU cores           | Increase replica fetch threads                    |
| **Broker**   | GC parameters            | G1GC                | Avoid frequent Full GC                            |
| **Producer** | `batch.size`           | 512KB~1MB           | Default 16KB, increase appropriately              |
| **Producer** | `linger.ms`            | 10~100ms            | Wait time to accumulate more messages             |
| **Producer** | `compression.type`     | `lz4` or `zstd` | Enable compression to reduce network transfer     |
| **Producer** | `acks`                 | `0` or `1`      | Lower consistency requirements                    |
| **Producer** | `retries`              | `0`               | Disable retries                                   |
| **Producer** | `buffer.memory`        | 128MB+              | Increase when multiple threads share              |
| **Consumer** | Multi-process/thread     | -                   | Parallel consumption improves processing capacity |
| **Consumer** | `fetch.min.bytes`      | 1KB+                | Batch fetch messages                              |

### Latency Tuning Parameters

| Component          | Parameter                | Recommended Value      | Description                                |
| ------------------ | ------------------------ | ---------------------- | ------------------------------------------ |
| **Broker**   | `num.replica.fetchers` | Increase appropriately | Accelerate replica synchronization         |
| **Producer** | `linger.ms`            | `0`                  | Send immediately, don't wait               |
| **Producer** | `compression.type`     | `none`               | Disable compression to reduce CPU overhead |
| **Producer** | `acks`                 | `1`                  | Balance consistency and performance        |
| **Consumer** | `fetch.min.bytes`      | `1`                  | Return available data immediately          |

## Monitoring and Troubleshooting

### Key Monitoring Metrics

```properties
# Producer monitoring
record-send-rate          # Send rate
record-send-total         # Total sends
batch-size-avg            # Average batch size
compression-rate-avg      # Compression rate

# Consumer monitoring
records-consumed-rate     # Consumption rate
fetch-latency-avg         # Fetch latency
consumer-lag             # Consumer lag

# Broker monitoring
MessagesInPerSec         # Message inflow rate
BytesInPerSec           # Byte inflow rate
RequestHandlerAvgIdlePercent  # Request handler idle rate
```

### Common Performance Issue Troubleshooting

#### 1. Low Producer Performance

```bash
# Check batch size and latency settings
bin/kafka-run-class.sh kafka.tools.ProducerPerformance \
  --topic test --num-records 1000000 \
  --record-size 1024 --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092
```

#### 2. Low Consumer Performance

```bash
# Check consumer lag
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-group
```

#### 3. Broker Performance Bottlenecks

```bash
# View JVM memory usage
jstat -gc <broker-pid> 1s

# View disk I/O
iostat -x 1
```

## Best Practices Summary

### 🚀 High Throughput Scenarios

1. Increase `batch.size` and `linger.ms`
2. Enable compression (lz4/zstd)
3. Set `acks=1`
4. Multiple Consumer parallel consumption

### ⚡ Low Latency Scenarios

1. Set `linger.ms=0`
2. Disable compression
3. Optimize network and disk configuration
4. Reduce GC pause time

### 🔄 Production Environment Recommendations

1. Monitor cluster key metrics
2. Conduct regular performance testing
3. Maintain version consistency
4. Properly plan partition count
