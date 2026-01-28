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
| **Concurrency Safety**    | Protect all shared state (queues, metadata, flush operations) using appropriate synchronization primitives (mutexes, locks).                                     |
| **Minimal Dependencies**  | Keep core dependencies restricted to the standard library to minimize bundle size and security risk.                                                             |
| **Initialization Safety** | Prevent data loss by requiring `init()` before `track()`. Throw errors for invalid operation order.                                                              |
| **Idiomatic APIs**        | Adhere to language-specific conventions (e.g., naming, error handling, async patterns) while maintaining behavioral consistency.                                 |

---

## II. Component Architecture

The SDK uses four primary, single-responsibility components:

| Component           | Design Pattern            | Primary Responsibilities                                                                                      |
| :------------------ | :------------------------ | :------------------------------------------------------------------------------------------------------------ |
| **Client**          | **Facade**                | Public API (`track`, `setMetadata`, `flush`, `init`, `dispose`). Coordinates MetadataManager and Dispatcher.  |
| **MetadataManager** | **Repository**            | Stores and retrieves global metadata. Provides thread-safe metadata snapshots for events.                     |
| **Dispatcher**      | **Command Queue**         | Manages event queue, batch processing, **atomic flush operations**, and retry logic with exponential backoff. |
| **Mutex**           | **Mutual Exclusion Lock** | Serializes concurrent operations (e.g., `flush()`) to prevent race conditions.                                |

### Adapter Interfaces

All custom functionality is abstracted via interfaces:

- **`HttpAdapter`**: Handles API communication. Sends POST request with
  `X-API-Key: {apiKey}` (or any other custom name) header. Must return an
  `HttpResponse`.
- **`StorageAdapter`**: Handles event persistence on failure or initialization.
  Methods (`save()`, `load()`, `clear()`) must be **idempotent** and handle
  errors gracefully.
- **`LoggerAdapter`**: Handles SDK internal logging with configurable log levels
  (`DEBUG`, `INFO`, `WARN`, `ERROR`, `NONE`). Built-in implementations include
  `ConsoleLoggerAdapter` and `NoOpLoggerAdapter`.
- **`SessionManager`** (Runtime-Specific): Generates and persists the unique
  session ID (`{timestamp}-{random}`).

---

## III. Type Safety and Generic Constraints

### Generic Type Parameters

The SDK provides compile-time type safety through generic parameters:

| Generic     | Constraint                | Description                                                                         |
| :---------- | :------------------------ | :---------------------------------------------------------------------------------- |
| `TEvents`   | `Record<string, unknown>` | **Optional:** Maps event names to their payload types for type-safe event tracking. |
| `TMetadata` | `Record<string, unknown>` | **Optional:** Defines the structure of metadata attached to events.                 |

#### Usage Examples

```ts
// Define event types mapping
type AppEvents = {
  "user.login": {
    email: string;
    method: "google" | "email";
  };
  "page.view": {
    url: string;
    title: string;
    duration?: number;
  };
  "purchase.completed": {
    orderId: string;
    amount: number;
    currency: string;
  };
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
  email: "<user@example.com>",
  method: "google",
});
```

### Backward Compatibility

- **Single Generic**: `RippleClient<Record<string, unknown>, AppMetadata>`
  (metadata-only typing)
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
| `maxBatchSize?`  | `number`         | **Optional:** Max events per batch. **(Default: 10)**.                                                            |
| `maxRetries?`    | `number`         | **Optional:** Max retry attempts. **(Default: 3)**.                                                               |
| `httpAdapter`    | `HttpAdapter`    | **Required:** Custom HTTP adapter for API requests.                                                               |
| `storageAdapter` | `StorageAdapter` | **Required:** Custom storage adapter for persistence.                                                             |
| `loggerAdapter?` | `LoggerAdapter`  | **Optional:** Custom logger adapter for SDK internal logging. **(Default: ConsoleLoggerAdapter with WARN level)** |

### 2. `Event` (API Payload)

The structure must be JSON-serializable.

| Field       | Type                             | Description                                                 |
| :---------- | :------------------------------- | :---------------------------------------------------------- |
| `name`      | `string`                         | Event identifier.                                           |
| `payload`   | `Map<string, unknown>` or `null` | Event data.                                                 |
| `issuedAt`  | `number`                         | Unix timestamp in milliseconds.                             |
| `sessionId` | `string` or `null`               | Session identifier (browser/native only).                   |
| `metadata`  | `Map<string, unknown>` or `null` | Event/environment-specific metadata (e.g., schema version). |
| `platform`  | `Platform` or `null`             | Platform information (auto-detected by runtime).            |

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

| Field    | Type      | Description                    |
| :------- | :-------- | :----------------------------- |
| `ok`     | `boolean` | `true` for `2xx` status codes. |
| `status` | `number`  | HTTP status code.              |
| `data?`  | `unknown` | Optional response body.        |

---

## V. Public API Contract

| Method               | Signature                                                                                         | Description/Behavior                                                                                                                                                                |
| :------------------- | :------------------------------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`init()`**         | `() -> void` (or `Future/Promise`)                                                                | Restores persisted events and starts scheduled flush. **Must be called before `track()`**.                                                                                          |
| **`track()`**        | `<K extends keyof TEvents>(name: K, payload?: TEvents[K], metadata?: Partial<TMetadata>) -> void` | Creates an enriched `Event` and enqueues it. **Type-safe** event names and payloads. Triggers **auto-flush** if `maxBatchSize` is reached. **Throws error if `init()` not called**. |
| **`setMetadata()`**  | `<K extends keyof TMetadata>(key: K, value: TMetadata[K]) -> void`                                | Stores **type-safe** metadata for all subsequent events.                                                                                                                            |
| **`getMetadata()`**  | `() -> Partial<TMetadata>`                                                                        | **Returns all stored metadata** as a shallow copy. Returns empty object if no metadata is set.                                                                                      |
| **`getSessionId()`** | `() -> string \| null`                                                                            | **Returns current session ID** or `null` if not set. Session ID is auto-generated in browser environments.                                                                          |
| **`flush()`**        | `() -> void` (or `Future/Promise`)                                                                | **Manually sends all queued events immediately**. **Mutex-protected**.                                                                                                              |
| **`dispose()`**      | `() -> void`                                                                                      | **Cleans up resources and frees memory** (cancels timers, clears queues, releases locks, resets state). **Supports re-initialization** after disposal.                              |

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
  `[...failedEvents, ...currentQueue]`.

### D. Initialization and Lifecycle

- **`init()` must be called before `track()`** to prevent data loss.
- Calling `track()` before initialization throws an error.
- Platform information is automatically detected during event creation.
- Metadata and payload are optional for all events.
- **Logger adapter** defaults to `ConsoleLoggerAdapter` with `WARN` level if not
  provided.
- **Type safety** is enforced at compile-time when using generic type
  parameters.
- **`dispose()` provides complete memory cleanup**: clears event queues, cancels
  scheduled flushes, releases mutex locks, and resets all internal state.
- **Re-initialization after disposal**: After calling `dispose()`, you can call
  `init()` again to restart the client with a clean state.

---

## VII. Quality and Implementation Checklist

### A. Performance

- **Time Complexity**: `enqueue()`, `dequeue()`, `setMetadata()`,
  `getMetadata()` are **O(1)**. `flush()` is **O(n)**, where $n$ is the batch
  size.
- **Space Complexity**: Queue is **O(n)** (number of queued events).

### B. Testing and Standards

- **Testing**: Must include comprehensive **Unit**, **Integration**, and
  **Concurrency Tests**.
- **Versioning**: Follow **Semantic Versioning 2.0.0**.
- **Security**: **HTTPS only**. API keys should not be hardcoded (use
  environment variables). PII should be hashed or anonymized.

### C. Future Improvements

1. **Event Validation**: Add optional schema validation for event payloads and
   metadata.
2. **Middleware System**: Introduce hooks for event transformation, filtering,
   and enrichment.
3. **Offline Queue Limits**: Implement maximum queue size with configurable
   eviction policies (FIFO, LRU).
4. **Batch Compression**: Add optional Gzip compression for large event batches.
5. **Enhanced Error Handling**: Provide detailed error categorization and custom
   error handlers.
6. **Performance Monitoring**: Built-in metrics for queue size, flush frequency,
   and retry rates.
7. **Event Sampling**: Configurable sampling rates for high-volume scenarios.
8. **Custom Serialization**: Support for custom event serialization formats
   (MessagePack, Avro).
9. **Background Sync**: Intelligent background synchronization with network
   state awareness.
10. **Event Deduplication**: Optional duplicate event detection based on
    configurable keys.
11. **Advanced Logger Features**: Structured logging, log rotation, and remote
    log shipping capabilities.
12. **Runtime Type Validation**: Optional runtime validation of event payloads
    against TypeScript types using libraries like Zod or Joi.
