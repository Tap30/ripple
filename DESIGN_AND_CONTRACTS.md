# Ripple SDK Architectural Specification

## I. Overview and Core Principles

The Ripple SDK is a high-performance, fault-tolerant event tracking system
designed using **object-oriented principles** and **dependency injection**. The
architecture emphasizes type safety, concurrency control, and flexible
extensibility.

### A. Core Architectural Principles

| Principle                 | Rationale                                                                                                                                                        |
| :------------------------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Consistency**           | Uniform core functionality and behavioral contracts across all language implementations.                                                                         |
| **Runtime Separation**    | Independent implementations for different environments (browser, Node.js, mobile) sharing a common core logic to optimize binary size and API compatibility.     |
| **Type Safety**           | Maximize compile-time error detection and developer experience using language-specific typing (generics, type hints, etc.).                                      |
| **Dependency Injection**  | Use **constructor injection** for adapters (e.g., HTTP, Storage) to ensure **testability** and **flexibility**. Pass interfaces/protocols, not concrete classes. |
| **Extensibility**         | Define clear interfaces for all key extension points (e.g., `HttpAdapter`, `StorageAdapter`).                                                                    |
| **Concurrency Safety**    | Protect all shared state (buffers, metadata, flush operations) using appropriate synchronization primitives (mutexes, locks).                                    |
| **Minimal Dependencies**  | Keep core dependencies restricted to the standard library to minimize bundle size and security risk.                                                             |
| **Initialization Safety** | Prevent data loss by queuing operations before `init()`. Track calls are automatically queued and processed after initialization completes.                      |
| **Idiomatic APIs**        | Adhere to language-specific conventions (e.g., naming, error handling, async patterns) while maintaining behavioral consistency.                                 |

---

## II. Component Architecture

The SDK uses four primary, single-responsibility components:

| Component           | Design Pattern            | Primary Responsibilities                                                                                       |
| :------------------ | :------------------------ | :------------------------------------------------------------------------------------------------------------- |
| **Client**          | **Facade**                | Public API (`track`, `setMetadata`, `flush`, `init`, `dispose`). Coordinates MetadataManager and Dispatcher.   |
| **MetadataManager** | **Repository**            | Stores and retrieves global metadata. Provides thread-safe metadata snapshots for events.                      |
| **Dispatcher**      | **Command Queue**         | Manages event buffer, batch processing, **atomic flush operations**, and retry logic with exponential backoff. |
| **Mutex**           | **Mutual Exclusion Lock** | Serializes concurrent operations (e.g., `flush()`) to prevent race conditions.                                 |

### Adapter Interfaces

All custom functionality is abstracted via interfaces:

- **`HttpAdapter`**: Handles API communication. Sends POST request with
  `X-API-Key: {apiKey}` (or any other custom name) header. Must return an
  `HttpResponse`.
- **`StorageAdapter`**: Handles event persistence on failure or initialization.
  Methods (`save()`, `load()`, `clear()`, `close()`) must be **idempotent** and
  handle errors gracefully. **Throws `StorageQuotaExceededError`** when storage
  quota is exceeded and events are dropped (after successfully saving reduced
  data). The `close()` method is called during disposal to release resources
  (e.g., close database connections, file handles).
- **`LoggerAdapter`**: Handles SDK internal logging with configurable log levels
  (`DEBUG`, `INFO`, `WARN`, `ERROR`, `NONE`). Built-in implementations include
  `ConsoleLoggerAdapter` and `NoOpLoggerAdapter`.
- **`SessionManager`** (Runtime-Specific): Generates and persists the unique
  session ID (`{timestamp}-{random}`).

---

## III. Type Safety and Generic Constraints

### Generic Type Parameters

The SDK provides compile-time type safety through generic parameters:

| Generic     | Constraint | Description                                                                         |
| :---------- | :--------- | :---------------------------------------------------------------------------------- |
| `TEvents`   | `Record`   | **Optional:** Maps event names to their payload types for type-safe event tracking. |
| `TMetadata` | `Record`   | **Optional:** Defines the structure of metadata attached to events.                 |

#### Usage Examples

```ts
// Define event types mapping
type AppEvents = {
  "user.login": { email: string; method: "google" | "email" };
  "page.view": { url: string; title: string; duration?: number };
  "purchase.completed": { orderId: string; amount: number; currency: string };
};

// Define metadata structure
type AppMetadata = {
  userId: string;
  sessionId: string;
  schemaVersion: string;
};

// Type-safe client instantiation
const client = new RippleClient<AppEvents, AppMetadata>(config);

// Type-safe event tracking (compile-time validation)
await client.track("user.login", {
  email: "<email>",
  method: "google",
});
```

### Backward Compatibility

- **Single Generic**: `RippleClient<any, AppMetadata>` (metadata-only typing)
- **No Generics**: `RippleClient` (no compile-time type checking)
- **Both Generics**: `RippleClient<AppEvents, AppMetadata>` (full type safety)

---

## IV. Data Structures (Contracts)

### 1. `Config` (Initialization)

| Field            | Type             | Constraint/Default                                                                                                |
| :--------------- | :--------------- | :---------------------------------------------------------------------------------------------------------------- |
| `apiKey`         | `string`         | **Required:** API authentication key.                                                                             |
| `endpoint`       | `string`         | **Required:** Valid HTTPS URL.                                                                                    |
| `apiKeyHeader?`  | `string`         | **Optional:** Header name for API key **(Default: "X-API-Key")**                                                  |
| `flushInterval?` | `number`         | **Optional:** Auto-flush interval in ms. **(Default: 5000)**.                                                     |
| `maxBatchSize?`  | `number`         | **Optional:** Max events per batch. **(Default: 10)**. Triggers immediate flush when reached.                     |
| `maxBufferSize?` | `number`         | **Optional:** Max events in memory/storage. **(Default: unlimited)**. Drops oldest events (FIFO) when exceeded.   |
| `maxRetries?`    | `number`         | **Optional:** Max retry attempts. **(Default: 3)**.                                                               |
| `httpAdapter`    | `HttpAdapter`    | **Required:** Custom HTTP adapter for API requests.                                                               |
| `storageAdapter` | `StorageAdapter` | **Required:** Custom storage adapter for persistence.                                                             |
| `loggerAdapter?` | `LoggerAdapter`  | **Optional:** Custom logger adapter for SDK internal logging. **(Default: ConsoleLoggerAdapter with WARN level)** |

### 2. `Event` (API Payload)

The structure must be JSON-serializable.

| Field       | Type                 | Description                                                 |
| :---------- | :------------------- | :---------------------------------------------------------- |
| `name`      | `string`             | Event identifier.                                           |
| `payload`   | `Map` or `null`      | Event data.                                                 |
| `issuedAt`  | `number`             | Unix timestamp in milliseconds.                             |
| `sessionId` | `string` or `null`   | Session identifier (browser/native only).                   |
| `metadata`  | `Map` or `null`      | Event/environment-specific metadata (e.g., schema version). |
| `platform`  | `Platform` or `null` | Platform information (auto-detected by runtime).            |

#### Platform (Discriminated Union)

**WebPlatform** (`type: "web"`):

- `browser`: `{ name: string, version: string }`
- `device`: `{ name: string, version: string }`
- `os`: `{ name: string, version: string }`

**NativePlatform** (`type: "native"`):

- `device`: `{ name: string, version: string }`
- `os`: `{ name: string, version: string }`

**ServerPlatform** (`type: "server"`):

- No additional fields

### 3. `HttpResponse` (Adapter Output)

| Field    | Type      | Description             |
| :------- | :-------- | :---------------------- |
| `status` | `number`  | HTTP status code.       |
| `data?`  | `unknown` | Optional response body. |

---

## V. Public API Contract

| Method               | Signature                                                                | Description/Behavior                                                                                                                                                                            |
| :------------------- | :----------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`init()`**         | `() -> void` (or `Future/Promise`)                                       | Restores persisted events (applying `maxBufferSize` limit) and starts scheduled flush. Events tracked before initialization are automatically queued and processed after `init()` completes.    |
| **`track()`**        | `(name: K, payload?: TEvents[K], metadata?: Partial<TMetadata>) -> void` | Creates an enriched `Event` and enqueues it. **Type-safe** event names and payloads. Triggers **auto-flush** if `maxBatchSize` is reached. **Operations are queued if called before `init()`**. |
| **`setMetadata()`**  | `(key: K, value: TMetadata[K]) -> void`                                  | Stores **type-safe** metadata for all subsequent events.                                                                                                                                        |
| **`getMetadata()`**  | `() -> Partial<TMetadata>`                                               | **Returns all stored metadata** as a shallow copy. Returns empty object if no metadata is set.                                                                                                  |
| **`getSessionId()`** | `() -> string \| null`                                                   | **Returns current session ID** or `null` if not set. Session ID is auto-generated in browser environments.                                                                                      |
| **`flush()`**        | `() -> void` (or `Future/Promise`)                                       | **Manually sends all queued events immediately**. **Mutex-protected**.                                                                                                                          |
| **`dispose()`**      | `() -> void`                                                             | **Cleans up resources and frees memory** (cancels timers, clears buffers, releases locks, resets state). **Supports re-initialization** after disposal.                                         |

---

## VI. Behavioral Guarantees

### A. Flush Guarantees

Flush is triggered by auto-flush, scheduled flush, or manual call.

- **Atomicity**: Only one flush operation runs at a time (mutex-protected).
- **Ordering**: Events are sent in **FIFO** order.
- **Persistence**: Failed events are re-queued and persisted to storage.

### B. Retry Logic (Exponential Backoff)

Retries are implemented with **exponential backoff and jitter**.

| Status Code / Error         | Retry Decision | Behavior After Max Retries   |
| :-------------------------- | :------------- | :--------------------------- |
| **2xx** (Success)           | **No Retry**   | Clear storage.               |
| **4xx** (Client Error)      | **No Retry**   | Drop events, clear storage.  |
| **5xx** (Server Error)      | **Retry**      | Re-queue and persist events. |
| **Network Error** (Timeout) | **Retry**      | Re-queue and persist events. |

The backoff delay calculation is:

$$\text{delay} = (\text{baseDelay} \cdot 2^{\text{attempt}}) + \text{jitter}$$

Where $\text{baseDelay}$ is $1000\text{ms}$ and $\text{jitter}$ is random
($0-1000\text{ms}$).

### C. Concurrency

- All public methods are thread-safe.
- Concurrent `flush()` calls are serialized via the Mutex.
- Re-queueing failed events maintains order:
  `[...failedEvents, ...currentBuffer]`.

### D. Initialization and Lifecycle

- **Events tracked before `init()` are automatically queued** and processed
  after initialization completes.
- **`init()` can be called multiple times** but only initializes once
  (subsequent calls are no-ops).
- Platform information is automatically detected during event creation.
- Metadata and payload are optional for all events.
- **Logger adapter** defaults to `ConsoleLoggerAdapter` with `WARN` level if not
  provided.
- **Type safety** is enforced at compile-time when using generic type
  parameters.
- **`dispose()` provides complete memory cleanup**: clears event buffers,
  cancels scheduled flushes, releases mutex locks, closes storage adapter, and
  resets all internal state.
- **Re-initialization after disposal**: After calling `dispose()`, you can call
  `init()` again to restart the client with a clean state.

---

## VII. Sequence Diagrams

### Initialization Flow

```mermaid
sequenceDiagram
    participant App
    participant Client
    participant Dispatcher
    participant Storage@{ "type" : "database" }
    participant Buffer@{ "type" : "queue" }

    App->>Client: init()
    Client->>Client: Check if already initialized
    alt Already initialized
        Client-->>App: Return (no-op)
    else Not initialized
        Client->>Client: Acquire init lock
        Client->>Dispatcher: restore()
        Dispatcher->>Storage: load()
        Storage-->>Dispatcher: Persisted events (or empty)
        Dispatcher->>Dispatcher: Apply maxBufferSize limit
        Dispatcher->>Buffer: Load events into buffer
        alt Buffer has events
            Dispatcher->>Dispatcher: Schedule flush timer
        end
    end
```

### Track Event Flow

```mermaid
sequenceDiagram
    participant App
    participant Client
    participant MetadataManager
    participant Dispatcher
    participant Buffer@{ "type" : "queue" }
    participant Storage@{ "type" : "database" }

    App->>Client: track(name, payload, metadata)
    alt Client is disposed
        Client-->>App: Silently drop event
    else Client is active
        Client->>Client: init() (auto-initialize if needed)
        Client->>MetadataManager: getAll()
        MetadataManager-->>Client: Shared metadata
        Client->>Client: Merge shared + event metadata
        Client->>Client: Create Event
        Client->>Dispatcher: enqueue(event)
        Dispatcher->>Buffer: Add event to buffer
        Dispatcher->>Dispatcher: Apply maxBufferSize limit (FIFO eviction)
        Dispatcher->>Storage: save(events)
        alt Buffer size >= maxBatchSize
            Dispatcher->>Dispatcher: flush()
        else Buffer size < maxBatchSize
            Dispatcher->>Dispatcher: Schedule one-shot flush timer
        end
    end
```

### Flush Flow

A flush operation can be triggered:

- Automatically at defined `flushInterval` schedules
- Whenever the `maxBatchSize` is exceeded
- Manually via `client.flush()`

```mermaid
sequenceDiagram
    participant Dispatcher
    participant Buffer@{ "type" : "queue" }
    participant HTTP
    participant Storage@{ "type" : "database" }

    Dispatcher->>Dispatcher: Acquire flush lock
    Dispatcher->>Dispatcher: Cancel scheduled timer
    alt Buffer is empty
        Dispatcher-->>Dispatcher: Return (nothing to flush)
    else Buffer has events
        Dispatcher->>Buffer: Get all events and clear buffer
        Buffer-->>Dispatcher: Returns events
        loop For each batch (up to maxBatchSize)
            Dispatcher->>HTTP: Send batch with headers
            alt 2xx Success
                Dispatcher->>Storage: clear()
            else 4xx Client Error
                Dispatcher->>Storage: clear()
                Note over Dispatcher: Events dropped (no retry)
            else 5xx Server Error
                Dispatcher->>Dispatcher: Retry with backoff (see Retry Flow)
            else Network Error
                Dispatcher->>Dispatcher: Retry with backoff (see Retry Flow)
            end
        end
    end
    Dispatcher->>Dispatcher: Release flush lock
```

### Retry Flow

```mermaid
sequenceDiagram
    participant Dispatcher
    participant HTTP
    participant Buffer@{ "type" : "queue" }
    participant Storage@{ "type" : "database" }

    Dispatcher->>Dispatcher: Wait for calculated backoff delay (or cancel if disposed)
    alt Cancelled (disposed)
        Dispatcher-->>Dispatcher: Abort retry
    else Delay completed
        Dispatcher->>HTTP: Retry send batch
        alt Success (2xx)
            Dispatcher->>Storage: clear()
        else Still failing & attempts remaining
            Dispatcher->>Dispatcher: Retry again (increment attempt)
        else Max retries reached
            Dispatcher->>Buffer: Re-queue failed events at front
            Dispatcher->>Dispatcher: Apply maxBufferSize limit
            Dispatcher->>Storage: save(events)
            Note over Dispatcher: Events preserved for next flush
        end
    end
```

### Dispose Flow

```mermaid
sequenceDiagram
    participant App
    participant Client
    participant MetadataManager
    participant Dispatcher
    participant Buffer@{ "type" : "queue" }
    participant Storage@{ "type" : "database" }

    App->>Client: dispose()
    Client->>Dispatcher: dispose()
    Dispatcher->>Dispatcher: Cancel in-flight retries (context cancellation)
    Dispatcher->>Dispatcher: Stop scheduled flush timer
    Dispatcher->>Buffer: Clear all events
    Dispatcher->>Storage: close()
    Client->>MetadataManager: clear()
```

### Scheduled Flush (One-Shot Timer) Flow

```mermaid
sequenceDiagram
    participant Dispatcher
    participant Timer

    Dispatcher->>Dispatcher: Check if timer already scheduled
    alt Timer exists or disposed
        Dispatcher-->>Dispatcher: Skip (no duplicate timers)
    else No active timer
        Dispatcher->>Timer: Schedule one-shot timer (flushInterval)
        Note over Timer: Timer fires once after interval
        Timer->>Timer: Clear timer reference
        Timer->>Dispatcher: flush()
        Note over Dispatcher: Next enqueue will schedule a new timer
    end
```

### Buffer Limit (FIFO Eviction) Flow

```mermaid
sequenceDiagram
    participant Dispatcher
    participant Buffer@{ "type" : "queue" }
    participant Storage@{ "type" : "database" }

    Dispatcher->>Dispatcher: Check maxBufferSize
    alt maxBufferSize isn't set
        Dispatcher-->>Dispatcher: No eviction needed
    else Events exceed maxBufferSize
        Dispatcher->>Dispatcher: Keep newest N events (drop oldest)
        Dispatcher->>Buffer: Replace buffer with limited events
        Dispatcher->>Storage: save(limited events)
        Note over Dispatcher: Oldest events dropped (FIFO)
    end
```

### HTTP Response Handling Decision Tree

```mermaid
flowchart TD
    Start([HTTP Request Sent]) --> Response{Response<br/>Received?}

    Response -->|Network Error| NetworkErr[Network Error]
    Response -->|Success| StatusCheck{Status<br/>Code?}

    StatusCheck -->|2xx| Success[Clear Storage]
    StatusCheck -->|4xx| ClientErr[Client Error]
    StatusCheck -->|5xx| ServerErr[Server Error]
    StatusCheck -->|Other| UnexpectedErr[Unexpected Status]

    Success --> End([Complete])

    ClientErr --> Drop[Drop Events<br/>Clear Storage]
    Drop --> End

    UnexpectedErr --> DropUnexpected[Drop Events<br/>Clear Storage]
    DropUnexpected --> End

    ServerErr --> RetryCheck{Attempts <<br/>MaxRetries?}
    NetworkErr --> RetryCheck

    RetryCheck -->|Yes| Backoff[Calculate Backoff<br/>2^attempt + jitter]
    RetryCheck -->|No| MaxRetry[Max Retries Reached]

    Backoff --> Wait[Wait for Backoff]
    Wait --> Cancelled{Disposed?}

    Cancelled -->|Yes| Abort([Abort Retry])
    Cancelled -->|No| Retry[Retry Request]
    Retry --> Response

    MaxRetry --> Requeue[Re-queue Events<br/>Apply Buffer Limit<br/>Persist to Storage]
    Requeue --> End
```

### Concurrent Flush Handling

```mermaid
sequenceDiagram
    participant Thread1
    participant Thread2
    participant FlushMutex
    participant Dispatcher

    Thread1->>Dispatcher: flush()
    Thread1->>FlushMutex: Try acquire lock
    FlushMutex-->>Thread1: Lock acquired

    par Thread 1 flushing
        Thread1->>Dispatcher: Process flush
        Note over Dispatcher: Sending events...
    and Thread 2 attempts flush
        Thread2->>Dispatcher: flush()
        Thread2->>FlushMutex: Try acquire lock
        Note over FlushMutex: Blocked (Thread1 holds lock)
    end

    Thread1->>Dispatcher: Flush complete
    Thread1->>FlushMutex: Release lock

    FlushMutex-->>Thread2: Lock acquired
    Thread2->>Dispatcher: Process flush
    Note over Dispatcher: Queue may be empty<br/>if Thread1 flushed all events
    Thread2->>FlushMutex: Release lock
```

---

## VIII. Configuration Best Practices

### A. Buffer and Batch Size Relationship

**Critical Rule**: `maxBufferSize` should always be **greater than or equal to**
`maxBatchSize`.

**Runtime Validation**: The SDK returns an error at initialization when
`maxBufferSize < maxBatchSize` to help catch misconfigurations early.

| Configuration                                | Behavior                                                                                  | Recommendation  |
| :------------------------------------------- | :---------------------------------------------------------------------------------------- | :-------------- |
| `maxBatchSize: 10, maxBufferSize: 100`       | ✅ Events flush every 10, buffer can hold 100 during offline periods                      | **Recommended** |
| `maxBatchSize: 20, maxBufferSize: 1000`      | ✅ Large buffer for extended offline periods, efficient batching                          | **Recommended** |
| `maxBatchSize: 100, maxBufferSize: 50`       | ❌ Batch size never reached, events dropped unnecessarily, only time-based flush triggers | **Avoid**       |
| `maxBatchSize: 50, maxBufferSize: 50`        | ⚠️ Works but no room for accumulation during brief network issues                         | **Suboptimal**  |
| `maxBatchSize: 10, maxBufferSize: unlimited` | ✅ Unlimited buffer, events never dropped (may cause memory issues in extreme cases)      | **Default**     |

**Understanding the Parameters**:

- **`maxBatchSize`**: Controls **when** events are sent (triggers immediate
  flush)
  - Determines network request frequency
  - Affects latency and throughput
  - Smaller values = more frequent requests, lower latency

- **`maxBufferSize`**: Controls **how many** events are kept in memory/storage
  - Prevents unbounded memory growth
  - Drops oldest events (FIFO) when exceeded
  - Applied before saving to storage AND when restoring from storage
  - Larger values = better offline resilience

**Scenarios**:

1. **Normal Operation** (`maxBatchSize: 10, maxBufferSize: 100`):
   - Events flush every 10 events
   - Buffer rarely fills (events sent frequently)
   - Optimal for real-time tracking

2. **Offline Mode** (`maxBatchSize: 10, maxBufferSize: 100`):
   - Buffer accumulates up to 100 events
   - Oldest events dropped when limit exceeded
   - Events sent when connection restored

3. **Misconfigured** (`maxBatchSize: 100, maxBufferSize: 50`):
   - Buffer hits 50 → drops to 50 → never reaches 100
   - Batch size never triggers flush
   - Events only sent via time-based flush (`flushInterval`)
   - Data loss without warning

### B. Storage Quota Handling

Storage adapters should automatically handle quota exceeded errors:

1. **Detection**: Catch platform-specific quota errors (e.g.,
   `QuotaExceededError` in browsers, disk full errors in native/server)
2. **Graceful Degradation**: Drop oldest 50% of events and retry save
3. **Transparency**: Throw/return custom error with details about saved/dropped
   counts
4. **Logging**: Dispatcher logs warning with dropped event count

**Implementation Guidelines**:

- Set reasonable `maxBufferSize` to prevent quota issues
- Use TTL to automatically expire old events
- Monitor storage quota warnings in production
- Choose storage mechanism appropriate for expected data volume

### C. Storage Mechanism Selection Guidelines

| Characteristic       | Small Volume (<100 events)      | Medium Volume (100-1000 events) | Large Volume (>1000 events) |
| :------------------- | :------------------------------ | :------------------------------ | :-------------------------- |
| **Browser**          | LocalStorage                    | LocalStorage                    | IndexedDB                   |
| **Native (Mobile)**  | SharedPreferences, UserDefaults | SQLite, Realm                   | SQLite, Realm               |
| **Server (Node.js)** | In-Memory, File System          | File System, SQLite             | Database, File System       |

**Capacity Guidelines**:

- **LocalStorage**: ~5-10MB (browser-dependent)
- **IndexedDB**: ~50MB+ (browser-dependent, can request more)
- **Native Storage**: Device-dependent (typically 100MB+ available)
- **File System**: Disk-dependent (typically unlimited for practical purposes)

## IX. Quality and Implementation Checklist

### A. Performance

- **Time Complexity**: `enqueue()`, `dequeue()`, `setMetadata()`,
  `getMetadata()` are **O(1)**. `flush()` is **O(n)**, where $n$ is the batch
  size.
- **Space Complexity**: Buffer is **O(n)** (number of queued events).

### B. Testing and Standards

- **Testing**: Must include comprehensive **Unit**, **Integration**, and
  **Concurrency Tests**.
- **Versioning**: Follow **Semantic Versioning 2.0.0**.
- **Security**: **HTTPS only**. API keys should not be hardcoded (use
  environment variables). PII should be hashed or anonymized.

### C. Future Improvements

1. **Middleware System**: Introduce hooks for event transformation, filtering,
   and enrichment.
2. **Batch Compression**: Add optional Gzip compression for large event batches.
3. **Event Sampling**: Configurable sampling rates for high-volume scenarios.
4. **429 Rate Limit Handling**: Treat HTTP 429 (Too Many Requests) as a
   retryable error instead of a client error. Implement retry with exponential
   backoff, respect `Retry-After` header when present, and re-queue events.
5. **Byte-based Buffer Limit**: The current `maxBufferSize` is event-count
   based, but storage quotas are byte-based. A byte-size limit option would more
   accurately prevent `QuotaExceededError`.
