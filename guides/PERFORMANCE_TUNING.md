# Performance Tuning Guide

## Overview

This guide helps you optimize Ripple SDK performance for your specific use case.
Proper configuration balances event delivery latency, network efficiency, and
resource usage.

## Key Configuration Parameters

### 1. `batchOptions.interval`, `batchOptions.size`, and `batchOptions.maxPayloadSize`

These parameters work together to control when events are sent to the backend.

#### `batchOptions.interval`

Time in milliseconds to wait before automatically flushing events.

**Default:** 10000ms

**When to increase:**

- Low event volume applications
- Network requests are expensive
- Batch efficiency is more important than latency

**When to decrease:**

- High event volume applications
- Real-time analytics requirements
- Critical events need immediate delivery

#### `batchOptions.size`

Maximum number of events in a single batch before auto-flush.

**Default:** 10

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

#### `batchOptions.maxPayloadSize`

Maximum accumulated payload size in bytes before auto-flush is triggered.

**Default:** 65536 (64KB)

**When to increase:**

- Events are large and backend supports bigger payloads
- Want fewer HTTP requests at the cost of larger individual requests
- Network bandwidth is not a concern

**When to decrease:**

- Backend has strict payload size limits
- Network bandwidth is limited
- Mobile or constrained environments

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
batchOptions: { interval: 5000, size: 30, maxPayloadSize: 65536 }
retryOptions: { maxAttempts: 3, minDelay: 1000, maxDelay: 360000, backoffFactor: 2 }
```

**High Volume Production:**

```
batchOptions: { interval: 10000, size: 100, maxPayloadSize: 131072 }
retryOptions: { maxAttempts: 5, minDelay: 1000, maxDelay: 360000, backoffFactor: 2 }
```

**Low Volume / Background:**

```
batchOptions: { interval: 30000, size: 50, maxPayloadSize: 65536 }
retryOptions: { maxAttempts: 3, minDelay: 2000, maxDelay: 360000, backoffFactor: 2 }
```

**Mobile / Constrained:**

```
batchOptions: { interval: 15000, size: 20, maxPayloadSize: 32768 }
retryOptions: { maxAttempts: 3, minDelay: 1000, maxDelay: 360000, backoffFactor: 2 }
```

---

### 2. `eventTtl` (Event Time-to-Live)

Time in milliseconds after which events are considered stale and dropped at
flush time. The age is calculated from the event's `issuedAt` timestamp.

**Default:** undefined (no TTL, events never expire)

**When to use:**

- Application may go offline for extended periods
- Stale events have no analytical value
- Want to prevent sending outdated data after reconnection
- Storage is limited and old events should be discarded

**Recommended values:**

- Real-time analytics: `eventTtl: 300000` (5 minutes)
- Standard applications: `eventTtl: 3600000` (1 hour)
- Offline-tolerant: `eventTtl: 86400000` (24 hours)

---

### 3. Storage Adapter Selection

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

### 4. `maxBufferSize` Sizing

Maximum number of events to keep in memory and storage.

**Default: 50 events**

> **Important:** All event rate estimates in this section refer to the rate at a
> **single SDK instance** level — i.e., one user session in one browser tab, one
> mobile app instance, or one Node.js process. Do **not** use your aggregate
> platform traffic (total events across all users) to size `maxBufferSize`. Each
> SDK instance manages its own independent buffer.

#### Typical Event Rates Per SDK Instance

| Environment                       | Typical rate (events/min) | Description                                      |
| :-------------------------------- | :------------------------ | :----------------------------------------------- |
| Browser (normal browsing)         | 1–10                      | Page views, clicks, occasional purchases         |
| Browser (active interaction)      | 10–50                     | E-commerce power user, frequent searches/filters |
| Browser (high-frequency tracking) | 50–500                    | Scroll/gesture tracking, real-time dashboards    |
| Mobile app                        | 1–30                      | Screen views, taps, lifecycle events             |
| Server (per-process)              | 100–10,000+               | API gateway tracking on behalf of many users     |
| Server (event aggregator)         | 10,000–100,000+           | Centralized tracking service, batch ingestion    |

#### Calculation Formula

```
maxBufferSize = (expected_events_per_hour / flushes_per_hour) * safety_factor
```

Where `flushes_per_hour = 3600000 / batchOptions.interval` and all rates are
**per SDK instance**.

**Example (Browser — active user):**

- 30 events/min per session → 1,800 events/hour
- `batchOptions.interval: 10000` → 360 flushes/hour
- Events per flush = 1,800 / 360 = 5
- With 3x safety factor = 5 \* 3 = 15
- Recommended: `maxBufferSize: 50` (default is sufficient)

**Example (Server — high-throughput process):**

- 10,000 events/min per process → 600,000 events/hour
- `batchOptions.interval: 5000` → 720 flushes/hour
- Events per flush = 600,000 / 720 ≈ 833
- With 3x safety factor = 833 \* 3 ≈ 2,500
- Recommended: `maxBufferSize: 3000`

#### Memory Impact

Estimate memory usage:

$$\text{memoryUsage} = \text{maxBufferSize} * \text{averageEventSize}$$

**Example:**

- maxBufferSize: 1000
- Average event: 500 bytes
- Memory: 1000 \* 500 = 500KB

#### Validation

The SDK validates that `maxBufferSize >= batchOptions.size` to ensure proper
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

## Troubleshooting Performance Issues

### High Memory Usage

**Solutions:**

- Reduce `maxBufferSize`
- Reduce `batchOptions.size`
- Decrease `batchOptions.interval` (flush more often)
- Use no-op storage
- Implement event sampling

### High Network Usage

**Solutions:**

- Increase `batchOptions.interval` (batch more)
- Increase `batchOptions.size`
- Increase `batchOptions.maxPayloadSize`
- Reduce event volume
- Optimize payload size
- Implement sampling

### High Latency

**Solutions:**

- Decrease `batchOptions.interval`
- Decrease `batchOptions.size`
- Use faster storage adapter
- Optimize network path
- Use CDN or edge locations

### Events Being Dropped

**Solutions:**

- Increase `maxBufferSize`
- Decrease `batchOptions.interval`
- Check storage availability
- Monitor network connectivity
- Review `eventTtl` setting (events may be expiring)
- Check `retryOptions.maxAttempts` is sufficient

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

- [Disaster Recovery Guide](./DISASTER_RECOVERY.md)
