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

- Configure SDK to use no-op storage adapter
- Increase flush frequency to compensate for lack of persistence
- Accept potential data loss on crashes

**Implement Storage Cleanup:**

- Create custom storage adapter with cleanup logic
- Remove old events periodically
- Implement TTL (time-to-live) for stored events

**Adjust Configuration:**

- Reduce `maxBufferSize` to limit storage usage
- Increase flush frequency
- Reduce batch size

#### Prevention

- Monitor storage usage proactively
- Implement storage cleanup policies
- Use appropriate `maxBufferSize` limits
- Flush events more frequently in constrained environments

---

### 2. Network Offline for Extended Periods

#### What Happens

When network is unavailable:

1. Events are queued in memory and persisted to storage
2. SDK respects `maxBufferSize` limit
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

**Graceful Degradation:**

- Reduce event volume during network issues
- Sample non-critical events
- Prioritize important events

#### Prevention

- Set appropriate `maxBufferSize` for your use case
- Implement network monitoring
- Test offline scenarios during development
- Consider event priority systems for critical data

---

### 3. Server Down for Hours

#### What Happens

When the backend server is unavailable:

1. SDK retries with exponential backoff (typically 3 retries)
2. Events are persisted to storage between retries
3. After max retries, events remain in storage
4. Events are restored and retried on next application start

#### Detection

Monitor HTTP adapter responses and SDK error logs for:

- Connection timeouts
- HTTP 5xx errors
- Network unreachable errors

#### Recovery Strategies

**Increase Retry Attempts:** Configure SDK with more retries for extended
outages.

**Implement Fallback Endpoint:** Create custom HTTP adapter that tries multiple
endpoints:

1. Primary endpoint
2. Fallback/backup endpoint
3. Emergency endpoint

**Manual Recovery After Outage:** After server is restored:

1. Initialize SDK (restores persisted events)
2. Manually trigger flush
3. Monitor success rate

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

Send metrics to monitoring systems and set up alerts for anomalies.

### 2. Graceful Degradation

Reduce functionality under stress:

- Switch to no-op storage when quota exceeded
- Increase flush frequency when storage is limited
- Reduce batch size when memory is constrained
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

- **Browser**: Use `beforeunload` event
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
maxRetries: 10
flushInterval: 5000 (flush frequently)
Storage: Persistent adapter
```

### Resource Constrained (Limited storage/memory)

```
maxBufferSize: 200
maxRetries: 3
flushInterval: 5000 (flush frequently)
Storage: No-op adapter
```

### Unreliable Network

```
maxBufferSize: 2000
maxRetries: 5
flushInterval: 15000 (batch more)
Storage: Persistent adapter
```

### High Volume

```
maxBufferSize: 10000
maxRetries: 3
flushInterval: 10000
Storage: Persistent adapter with cleanup
```

---

**Related Documentation:**

- [Performance Tuning Guide](./PERFORMANCE_TUNING.md)
