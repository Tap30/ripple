# Performance Tuning Guide

## Overview

This guide helps you optimize Ripple SDK performance for your specific use case.
Proper configuration balances event delivery latency, network efficiency, and
resource usage.

## Key Configuration Parameters

### 1. `flushInterval` vs `maxBatchSize`

These two parameters work together to control when events are sent to the
backend.

#### `flushInterval`

Time in milliseconds to wait before automatically flushing events.

**When to increase:**

- Low event volume applications
- Network requests are expensive
- Batch efficiency is more important than latency

**When to decrease:**

- High event volume applications
- Real-time analytics requirements
- Critical events need immediate delivery

#### `maxBatchSize`

Maximum number of events in a single batch before auto-flush.

**When to increase:**

- High event volume
- Backend can handle larger payloads
- Network latency is high
- Want to minimize HTTP requests

**When to decrease:**

- Backend has payload size limits
- Memory constrained environments
- Want more frequent updates
- Network bandwidth is limited

#### Tradeoffs

| Configuration              | Latency | Network Efficiency | Memory Usage | Backend Load |
| -------------------------- | ------- | ------------------ | ------------ | ------------ |
| High interval, Large batch | High    | Excellent          | Higher       | Lower        |
| Low interval, Small batch  | Low     | Poor               | Lower        | Higher       |
| High interval, Small batch | Medium  | Good               | Lower        | Medium       |
| Low interval, Large batch  | Low     | Good               | Higher       | Medium       |

#### Recommended Configurations

**Real-time Analytics:**

```
flushInterval: 5000ms
maxBatchSize: 30
```

**High Volume Production:**

```
flushInterval: 10000ms
maxBatchSize: 100
```

**Low Volume / Background:**

```
flushInterval: 30000ms
maxBatchSize: 50
```

**Mobile / Constrained:**

```
flushInterval: 15000ms
maxBatchSize: 20
```

---

### 2. Storage Adapter Selection

Choose the right storage adapter based on your environment and requirements.

#### Storage Types

**Persistent Storage**

- **Capacity:** Platform-dependent (5MB-50MB+ for browsers, unlimited for
  servers)
- **Performance:** Varies by implementation
- **Persistence:** Survives app restarts
- **Use case:** Most production applications

**No-Op Storage**

- **Capacity:** N/A (no persistence)
- **Performance:** Instant (no-op)
- **Persistence:** None
- **Use case:** Testing, stateless servers, storage-constrained environments

#### Selection Criteria

| Requirement           | Client-Side                 | Server-Side         |
| --------------------- | --------------------------- | ------------------- |
| High volume           | Persistent (large capacity) | Persistent or No-op |
| Low latency           | Persistent (fast)           | No-op               |
| Persistence needed    | Persistent                  | Persistent          |
| No persistence needed | No-op                       | No-op               |
| Storage constrained   | No-op                       | No-op               |
| Stateless deployment  | No-op                       | No-op               |

#### Platform-Specific Considerations

**Browser:**

- LocalStorage: ~5-10MB, synchronous, fast
- IndexedDB: ~50MB+, asynchronous, more capacity
- No-op: No persistence, use with frequent flushing

**Mobile:**

- Platform storage APIs (SharedPreferences, UserDefaults, etc.)
- SQLite for large volumes
- No-op for memory-constrained devices

**Server:**

- File-based storage for persistent servers
- Database storage for distributed systems
- No-op for stateless/containerized deployments

---

### 3. `maxBufferSize` Sizing

Maximum number of events to keep in memory and storage.

**Typical default: 1000 events**

#### Calculation Formula

```
maxBufferSize = (expected_events_per_hour / flushes_per_hour) * safety_factor
```

**Example:**

- 10,000 events/hour
- Flush every 10 seconds = 360 flushes/hour
- Events per flush = 10,000 / 360 ≈ 28
- With 3x safety factor = 28 \* 3 = 84
- Recommended: `maxBufferSize: 100`

#### Sizing Guidelines

**Low Volume (<1000 events/hour):**

```
maxBufferSize: 500
```

**Medium Volume (1000-10000 events/hour):**

```
maxBufferSize: 1000
```

**High Volume (>10000 events/hour):**

```
maxBufferSize: 5000
```

**Memory Constrained:**

```
maxBufferSize: 200
maxBatchSize: 20
flushInterval: 5000 (flush more frequently)
```

#### Memory Impact

Estimate memory usage:

$$\text{memoryUsage} = \text{maxBufferSize} * \text{averageEventSize}$$

**Example:**

- maxBufferSize: 1000
- Average event: 500 bytes
- Memory: 1000 \* 500 = 500KB

#### Validation

The SDK validates that `maxBufferSize >= maxBatchSize` to ensure proper
operation.

---

## Performance Optimization Strategies

### 1. Event Sampling

For high-volume, non-critical events, implement sampling:

- Sample percentage of events (e.g., 10%)
- Always track critical events (purchases, errors, etc.)
- Use consistent sampling for user sessions

### 2. Event Batching in Application

Batch related events before tracking:

- Combine multiple similar events into one
- Use array payloads for bulk operations
- Reduce total event count

### 3. Lazy Initialization

Initialize SDK only when needed:

- Defer initialization until first event
- Reduce startup time
- Save resources if tracking isn't used

### 4. Debounce High-Frequency Events

For events that fire rapidly:

- Debounce scroll events
- Throttle mouse movement tracking
- Aggregate rapid user actions

### 5. Optimize Payload Size

- Send only necessary data
- Use IDs instead of full objects
- Compress large payloads if supported
- Remove redundant information

---

## Monitoring Performance

### Track SDK Metrics

Implement custom logger to monitor:

- Events tracked per minute
- Flush frequency and duration
- Success/failure rates
- Storage operations
- Network latency

### Measure Flush Performance

Track:

- Time to flush events
- Batch sizes
- Network request duration
- Retry attempts

---

## Environment-Specific Recommendations

### Browser (SPA)

```
flushInterval: 10000ms
maxBatchSize: 50
maxBufferSize: 500
Storage: Persistent (LocalStorage/IndexedDB)
```

### Browser (High Traffic)

```
flushInterval: 5000ms
maxBatchSize: 100
maxBufferSize: 2000
Storage: Persistent (IndexedDB for capacity)
```

### Mobile (iOS/Android)

```
flushInterval: 15000ms
maxBatchSize: 30
maxBufferSize: 500
Storage: Platform persistent storage
```

### Server (API/Web Server)

```
flushInterval: 10000ms
maxBatchSize: 100
maxBufferSize: 1000
Storage: No-op (stateless) or File-based
```

### Server (Background Worker)

```
flushInterval: 30000ms
maxBatchSize: 200
maxBufferSize: 5000
Storage: File-based or Database
```

### Serverless / Lambda

```
flushInterval: 1000ms (flush quickly before function ends)
maxBatchSize: 20
maxBufferSize: 100
Storage: No-op
Always flush before function ends
```

---

## Troubleshooting Performance Issues

### High Memory Usage

**Solutions:**

- Reduce `maxBufferSize`
- Reduce `maxBatchSize`
- Increase `flushInterval` (flush more often)
- Use no-op storage
- Implement event sampling

### High Network Usage

**Solutions:**

- Increase `flushInterval` (batch more)
- Increase `maxBatchSize`
- Reduce event volume
- Optimize payload size
- Implement sampling

### High Latency

**Solutions:**

- Decrease `flushInterval`
- Decrease `maxBatchSize`
- Use faster storage adapter
- Optimize network path
- Use CDN or edge locations

### Events Being Dropped

**Solutions:**

- Increase `maxBufferSize`
- Decrease `flushInterval`
- Check storage availability
- Monitor network connectivity
- Implement retry logic

---

## Performance Testing

### Load Testing

Test SDK under various conditions:

- High event volume (1000+ events/second)
- Network latency simulation
- Storage quota limits
- Memory constraints
- Extended offline periods

### Benchmarking

Measure:

- Event tracking latency
- Flush operation duration
- Storage operation speed
- Memory footprint
- Network bandwidth usage

### Profiling

Profile your application to identify:

- SDK overhead
- Bottlenecks in event tracking
- Storage performance issues
- Network request patterns

---

**Related Documentation:**

### Load Testing

Test SDK under various conditions:

- High event volume (1000+ events/second)
- Network latency simulation
- Storage quota limits
- Memory constraints
- Extended offline periods

### Benchmarking

Measure:

- Event tracking latency
- Flush operation duration
- Storage operation speed
- Memory footprint
- Network bandwidth usage

### Profiling

Profile your application to identify:

- SDK overhead
- Bottlenecks in event tracking
- Storage performance issues
- Network request patterns

---

**Related Documentation:**

- [Disaster Recovery Guide](./DISASTER_RECOVERY.md)
