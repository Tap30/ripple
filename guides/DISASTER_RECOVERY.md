# Disaster Recovery Guide

## Overview

This guide covers failure scenarios and recovery strategies for Ripple SDK
deployments. Understanding how the SDK behaves under adverse conditions helps
you build resilient event tracking systems.

## Failure Scenarios

### 1. Storage is Full

#### What Happens

When storage is exhausted:

1. SDK logs a storage quota exceeded warning
2. New events are still queued in memory
3. Events are sent to the backend but not persisted locally
4. If the application crashes before flush, in-memory events are lost

#### Detection

Monitor SDK logs for storage quota warnings. The SDK will log when storage
operations fail due to quota limits.

#### Recovery Strategies

**Monitor Storage Quota:**

- Implement storage monitoring in your application
- Alert when storage usage exceeds thresholds (e.g., 80%)
- Track storage trends over time

**Use No-Persistence Mode:**

- Configure SDK without a `storageAdapter`
- Increase flush frequency to compensate for lack of persistence
- Accept potential data loss on crashes

**Implement Storage Cleanup:**

- Create custom storage adapter with cleanup logic
- Remove old events periodically
- Use `eventTtl` to automatically drop stale events at flush time

**Adjust Configuration:**

- Reduce `maxBufferSize` to limit storage usage (default: 50)
- Increase flush frequency via `batchOptions.interval`
- Reduce batch size via `batchOptions.size`

#### Prevention

- Monitor storage usage proactively
- Implement storage cleanup policies
- Use appropriate `maxBufferSize` limits (default: 50)
- Configure `eventTtl` to prevent unbounded accumulation of stale events
- Flush events more frequently in constrained environments

---

### 2. Network Offline for Extended Periods

#### What Happens

When network is unavailable:

1. Events are queued in memory and persisted to storage
2. SDK respects `maxBufferSize` limit (default: 50)
3. Oldest events are dropped (FIFO) when limit is reached
4. Automatic retry with exponential backoff when network returns

#### Detection

**Client-Side (Browser, Mobile):**

- Listen for network connectivity events
- Monitor SDK retry attempts in logs
- Track failed flush operations

**Server-Side:**

- Monitor network interface status
- Check connectivity to backend endpoints
- Track SDK error logs

#### Recovery Strategies

**Increase Buffer Size:** Configure SDK with larger `maxBufferSize` to store
more events during outages.

**Manual Flush on Reconnect:** Implement connectivity monitoring and manually
trigger flush when network is restored.

**Critical Events Priority:** For critical events, flush immediately rather than
batching.

**Use `eventTtl` to Manage Staleness:** When events have been persisted for too
long during outages, `eventTtl` ensures stale events are automatically dropped
at flush time rather than being sent with outdated data. This prevents sending
irrelevant historical events after prolonged offline periods.

**Graceful Degradation:**

- Reduce event volume during network issues
- Sample non-critical events
- Prioritize important events

#### Prevention

- Set appropriate `maxBufferSize` for your use case
- Configure `eventTtl` to limit maximum event age
- Implement network monitoring
- Test offline scenarios during development
- Consider event priority systems for critical data

---

### 3. Server Down for Hours

#### What Happens

When the backend server is unavailable:

1. SDK retries with exponential backoff (default: 3 attempts, backoff factor 2)
2. Events are persisted to storage between retries
3. After `retryOptions.maxAttempts` exhausted, events remain in storage
4. Events are restored and retried on next application start

#### Detection

Monitor HTTP adapter responses and SDK error logs for:

- Connection timeouts
- HTTP 5xx errors
- Network unreachable errors

#### Recovery Strategies

**Increase Retry Attempts:** Configure SDK with higher
`retryOptions.maxAttempts` for extended outages.

**Implement Fallback Endpoint:** Create custom HTTP adapter that tries multiple
endpoints:

1. Primary endpoint
2. Fallback/backup endpoint
3. Emergency endpoint

**Manual Recovery After Outage:** After server is restored:

1. Initialize SDK (restores persisted events)
2. Manually trigger flush
3. Monitor success rate

**Use `eventTtl` to Drop Stale Events:** If the server was down for an extended
period, events persisted during the outage may be outdated. Configure `eventTtl`
so that events older than the TTL threshold are automatically dropped at flush
time rather than being sent with stale data.

**Health Check Integration:**

- Implement server health checks
- Wait for server health before flushing
- Retry periodically until successful

#### Prevention

- Implement server health checks
- Use load balancers with failover
- Set up monitoring and alerting
- Test extended outage scenarios
- Document recovery procedures

---

## General Best Practices

### 1. Monitoring and Alerting

Implement custom logger to track:

- Events tracked count
- Flush success/failure rate
- Storage errors
- Network errors
- Retry attempts

Use `telemetryOptions` and `TelemetryHooks` (onFlush, onSendSuccess,
onSendFailure, onRetry, onDrop, onEnqueue) to feed metrics into monitoring
systems and set up alerts for anomalies.

### 2. Graceful Degradation

Reduce functionality under stress:

- Switch to no-op storage when quota exceeded
- Decrease `batchOptions.interval` when storage is limited
- Reduce `batchOptions.size` when memory is constrained
- Sample events when volume is too high

### 3. Testing Failure Scenarios

Test your application with:

- Simulated storage failures
- Simulated network failures
- Simulated server errors
- Extended outage scenarios

Create test adapters that simulate failures to verify recovery behavior.

### 4. Data Loss Prevention

**Application Shutdown:**

- Always flush events before shutdown
- Implement proper disposal in cleanup handlers
- Use lifecycle hooks (beforeunload, onPause, etc.)
- Implement graceful shutdown procedures

**Platform-Specific:**

- **Browser**: Use `beforeunload` event (or `appClosed()` convenience method)
- **Mobile**: Use app lifecycle events (onPause, onStop)
- **Server**: Handle SIGTERM/SIGINT signals

---

## Recovery Checklist

When disaster strikes:

- [ ] Check SDK logs for error messages
- [ ] Verify storage availability and quota
- [ ] Test network connectivity
- [ ] Verify backend server health
- [ ] Check persisted events in storage
- [ ] Manually trigger flush if needed
- [ ] Monitor retry attempts and success rate
- [ ] Review and adjust configuration if needed
- [ ] Document incident for future prevention

---

## Configuration Recommendations by Scenario

### High Reliability (Can't lose events)

```
maxBufferSize: 5000
retryOptions: { maxAttempts: 10, minDelay: 1000, maxDelay: 360000, backoffFactor: 2 }
batchOptions: { interval: 5000, size: 10 }
eventTtl: 86400000 (24 hours - keep events relevant for a full day)
# Use a persistent storageAdapter (e.g., IndexedDB, file-based)
```

### Resource Constrained (Limited storage/memory)

```
maxBufferSize: 200
retryOptions: { maxAttempts: 3 }
batchOptions: { interval: 5000, size: 10 }
eventTtl: 3600000 (1 hour - drop stale events quickly)
# Omit storageAdapter for no-persistence mode
```

### Unreliable Network

```
maxBufferSize: 2000
retryOptions: { maxAttempts: 5, minDelay: 2000, backoffFactor: 2 }
batchOptions: { interval: 15000, size: 20 }
eventTtl: 43200000 (12 hours - balance freshness vs. recovery window)
# Use a persistent storageAdapter
```

### High Volume

```
maxBufferSize: 10000
retryOptions: { maxAttempts: 3 }
batchOptions: { interval: 10000, size: 50, maxPayloadSize: 65536 }
eventTtl: 7200000 (2 hours - prevent stale event buildup)
# Use a persistent storageAdapter with cleanup
```

---

**Related Documentation:**

- [Performance Tuning Guide](./PERFORMANCE_TUNING.md)
