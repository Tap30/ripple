# 🎯 Ripple SDK Architectural Specification

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
| **Concurrency Safety**    | Protect all shared state (queues, context, flush operations) using appropriate synchronization primitives (mutexes, locks).                                      |
| **Minimal Dependencies**  | Keep core dependencies restricted to the standard library to minimize bundle size and security risk.                                                             |
| **Initialization Safety** | Prevent data loss by requiring `init()` before `track()`. Throw errors for invalid operation order.                                                              |
| **Idiomatic APIs**        | Adhere to language-specific conventions (e.g., naming, error handling, async patterns) while maintaining behavioral consistency.                                 |

---

## II. Component Architecture

The SDK uses four primary, single-responsibility components:

| Component          | Design Pattern            | Primary Responsibilities                                                                                      |
| :----------------- | :------------------------ | :------------------------------------------------------------------------------------------------------------ |
| **Client**         | **Facade**                | Public API (`track`, `setContext`, `flush`, `init`, `dispose`). Coordinates ContextManager and Dispatcher.    |
| **ContextManager** | **Repository**            | Stores and retrieves global context. Provides thread-safe context snapshots for events.                       |
| **Dispatcher**     | **Command Queue**         | Manages event queue, batch processing, **atomic flush operations**, and retry logic with exponential backoff. |
| **Mutex**          | **Mutual Exclusion Lock** | Serializes concurrent operations (e.g., `flush()`) to prevent race conditions.                                |

### Adapter Interfaces

All custom functionality is abstracted via interfaces:

- **`HttpAdapter`**: Handles API communication. Sends POST request with
  `Authorization: Bearer {apiKey}` header. Must return an `HttpResponse`.
- **`StorageAdapter`**: Handles event persistence on failure or initialization.
  Methods (`save()`, `load()`, `clear()`) must be **idempotent** and handle
  errors gracefully.
- **`SessionManager`** (Runtime-Specific): Generates and persists the unique
  session ID (`{timestamp}-{random}`).

---

## III. Data Structures (Contracts)

### 1. `Config` (Initialization)

| Field            | Type     | Constraint/Default                            |
| :--------------- | :------- | :-------------------------------------------- |
| `apiKey`         | `string` | **Required**.                                 |
| `endpoint`       | `string` | **Required**. Valid HTTPS URL.                |
| `flushInterval?` | `number` | Auto-flush interval in ms. **Default: 5000**. |
| `maxBatchSize?`  | `number` | Max events per batch. **Default: 10**.        |
| `maxRetries?`    | `number` | Max retry attempts. **Default: 3**.           |

### 2. `Event` (API Payload)

The structure must be JSON-serializable.

| Field       | Type                             | Description                                                |
| :---------- | :------------------------------- | :--------------------------------------------------------- |
| `name`      | `string`                         | Event identifier.                                          |
| `payload`   | `Map<string, unknown>` or `null` | (optional) Event data.                                     |
| `issuedAt`  | `number`                         | Unix timestamp in milliseconds.                            |
| `context?`  | `Map<string, unknown>`           | Snapshot of global context.                                |
| `sessionId` | `string` or `null`               | Session identifier (browser only).                         |
| `metadata`  | `EventMetadata` or `null`        | (optional) Event-specific metadata (e.g., schema version). |
| `platform`  | `Platform` or `null`             | Platform information (auto-detected by runtime).           |

#### EventMetadata

| Field            | Type     | Description                  |
| :--------------- | :------- | :--------------------------- |
| `schemaVersion?` | `string` | Schema version for the event |

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

## IV. Public API Contract

| Method             | Signature                                                                          | Description/Behavior                                                                                                                        |
| :----------------- | :--------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| **`init()`**       | `() -> void` (or `Future/Promise`)                                                 | Restores persisted events and starts scheduled flush. **Must be called before `track()`**.                                                  |
| **`track()`**      | `(name: string, payload?: Map<string, unknown>, metadata?: EventMetadata) -> void` | Creates an enriched `Event` and enqueues it. Triggers **auto-flush** if `maxBatchSize` is reached. **Throws error if `init()` not called**. |
| **`setContext()`** | `(key: string, value: unknown) -> void`                                            | Stores context for all subsequent events.                                                                                                   |
| **`flush()`**      | `() -> void` (or `Future/Promise`)                                                 | **Manually sends all queued events immediately**. **Mutex-protected**.                                                                      |
| **`dispose()`**    | `() -> void`                                                                       | **Cleans up resources** (cancels timers, detaches listeners). Call `flush()` first if pending events must be sent.                          |

---

## V. Behavioral Guarantees

### A. Flush Guarantees

Flush is triggered by auto-flush, scheduled flush, or manual call.

- **Atomicity**: Only one flush operation runs at a time (mutex-protected).
- **Ordering**: Events are sent in **FIFO** order.
- **Persistence**: Failed events are re-queued and persisted to storage.

### B. Retry Logic (Exponential Backoff)

Retries are implemented with **exponential backoff and jitter**.

| Status Code / Error         | Retry Decision         | Behavior After Max Retries   |
| :-------------------------- | :--------------------- | :--------------------------- |
| **2xx**                     | **No Retry** (Success) | Clear storage.               |
| **4xx** (Client Error)      | **No Retry**           | Re-queue and persist events. |
| **5xx** (Server Error)      | **Retry**              | Re-queue and persist events. |
| **Network Error** (Timeout) | **Retry**              | Re-queue and persist events. |

The backoff delay calculation is:
$$\text{delay} = (\text{baseDelay} \cdot 2^{\text{attempt}}) + \text{jitter}$$
Where $\text{baseDelay}$ is $1000\text{ms}$ and $\text{jitter}$ is random
($0-1000\text{ms}$).

### C. Concurrency

- All public methods are thread-safe.
- Concurrent `flush()` calls are serialized via the Mutex.
- Re-queueing failed events maintains order:
  `[...failedEvents, ...currentQueue]`.

### D. Initialization

- **`init()` must be called before `track()`** to prevent data loss.
- Calling `track()` before initialization throws an error.
- Platform information is automatically detected during event creation.
- Metadata and payload are optional for all events.

---

## VI. Quality and Implementation Checklist

### A. Performance

- **Time Complexity**: `enqueue()`, `dequeue()`, `setContext()`, `getContext()`
  are **O(1)**. `flush()` is **O(n)**, where $n$ is the batch size.
- **Space Complexity**: Queue is **O(n)** (number of queued events).

### B. Testing and Standards

- **Testing**: Must include comprehensive **Unit**, **Integration**, and
  **Concurrency Tests**.
- **Versioning**: Follow **Semantic Versioning 2.0.0**.
- **Security**: **HTTPS only**. API keys should not be hardcoded (use
  environment variables). PII should be hashed or anonymized.

### C. Future Improvements

1. **Event Validation**: Add optional schema validation.
2. **Middleware System**: Introduce hooks for event transformation/filtering.
3. **Offline Queue Limits**: Implement a maximum queue size with eviction
   policies.
4. **Batch Compression**: Add optional Gzip compression.
