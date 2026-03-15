# Component Design

## Inheritance Hierarchy

```
System.TimeProvider                          (abstract, .NET BCL)
    └── FakeTimeProvider                     (Microsoft.Extensions.Time.Testing)
            └── SignalingFakeTimeProvider     (NServiceBus.Transport.SQS.Tests)
```

---

## 1. `System.TimeProvider` (Base Class)

**Namespace:** `System`
**Assembly:** `System.Runtime` (.NET 8+)

An abstract base class introduced in .NET 8 that provides an abstraction for time-related operations, enabling testability of time-dependent code.

### Key Members

| Member | Signature | Description |
|--------|-----------|-------------|
| `System` | `static TimeProvider System { get; }` | Returns the default system time provider using `DateTimeOffset.UtcNow` and real timers. |
| `GetUtcNow()` | `virtual DateTimeOffset GetUtcNow()` | Returns the current UTC time. Overridden by `FakeTimeProvider` to return controlled fake time. |
| `GetLocalNow()` | `DateTimeOffset GetLocalNow()` | Returns the current local time based on `GetUtcNow()` and `LocalTimeZone`. |
| `LocalTimeZone` | `virtual TimeZoneInfo LocalTimeZone { get; }` | The time zone for local time conversions. |
| `TimestampFrequency` | `virtual long TimestampFrequency { get; }` | Frequency of the high-performance timestamp counter. |
| `GetTimestamp()` | `virtual long GetTimestamp()` | Returns a high-performance timestamp. |
| `CreateTimer()` | `virtual ITimer CreateTimer(TimerCallback callback, object? state, TimeSpan dueTime, TimeSpan period)` | Creates a new `ITimer` instance. This is the key virtual method overridden by `SignalingFakeTimeProvider`. |

### Role in This Design

`TimeProvider` is the injection point. Production code defaults to `TimeProvider.System`, while tests inject `FakeTimeProvider` or `SignalingFakeTimeProvider`. The `Renewal` class and `InputQueuePump` accept it as an optional parameter:

```csharp
public static async Task<Result> RenewMessageVisibility(
    ..., TimeProvider timeProvider = null, ...)
{
    timeProvider ??= TimeProvider.System;
    ...
}
```

---

## 2. `FakeTimeProvider` (Intermediate Class)

**Namespace:** `Microsoft.Extensions.Time.Testing`
**Package:** `Microsoft.Extensions.TimeProvider.Testing` v10.4.0

A concrete implementation of `TimeProvider` designed for unit testing. It provides full control over the passage of time.

### Key Members

| Member | Signature | Description |
|--------|-----------|-------------|
| `FakeTimeProvider()` | Constructor | Initializes with a default start time. |
| `FakeTimeProvider(DateTimeOffset startDateTime)` | Constructor | Initializes with a specific start time. |
| `Start` | `DateTimeOffset Start { get; }` | The initial time value set at construction. |
| `GetUtcNow()` | `override DateTimeOffset GetUtcNow()` | Returns the current fake UTC time. |
| `Advance(TimeSpan delta)` | `void Advance(TimeSpan delta)` | Advances fake time by the specified duration, triggering any timers that become due. |
| `SetUtcNow(DateTimeOffset value)` | `void SetUtcNow(DateTimeOffset value)` | Sets the fake time to an exact value. |
| `CreateTimer()` | `override ITimer CreateTimer(TimerCallback callback, object? state, TimeSpan dueTime, TimeSpan period)` | Creates a fake timer that fires when time is advanced past its due time. |

### How Timer Advancement Works

When `Advance()` is called:
1. The fake time is incremented by the specified duration.
2. All registered timers are checked against the new time.
3. Timers whose due time has been reached have their callbacks invoked.
4. For periodic timers, the next due time is calculated.

This deterministic behavior replaces real wall-clock delays with controlled time progression.

---

## 3. `SignalingFakeTimeProvider` (Test Double)

**Namespace:** `NServiceBus.Transport.SQS.Tests`
**File:** `src/NServiceBus.Transport.SQS.Tests/Receiving/RenewalTests.cs` (lines 87-107)
**Scope:** Nested class within `RenewalTests`

Extends `FakeTimeProvider` with timer-registration signaling to synchronize test execution with asynchronous renewal loops.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `_nextTimerRegistered` | `volatile TaskCompletionSource` | A `TaskCompletionSource` that is signaled (`TrySetResult()`) each time `CreateTimer` is called. Marked `volatile` for cross-thread visibility. Initialized with `TaskCreationOptions.RunContinuationsAsynchronously`. |

### Methods

#### `WaitForNextTimer()`

```csharp
public Task WaitForNextTimer()
{
    _nextTimerRegistered = new TaskCompletionSource(TaskCreationOptions.RunContinuationsAsynchronously);
    return _nextTimerRegistered.Task;
}
```

- Creates a fresh `TaskCompletionSource` (replacing any previous one).
- Returns the associated `Task` which completes when `CreateTimer` is next called.
- **Must be called before** the action that triggers timer creation (e.g., before `Advance()`).

#### `CreateTimer()` (override)

```csharp
public override ITimer CreateTimer(TimerCallback callback, object state, TimeSpan dueTime, TimeSpan period)
{
    var timer = base.CreateTimer(callback, state, dueTime, period);
    _nextTimerRegistered.TrySetResult();
    return timer;
}
```

- Delegates to `base.CreateTimer()` to create the fake timer.
- Signals the `TaskCompletionSource` via `TrySetResult()`.
- Returns the timer to the caller (the `Task.Delay` internals).

---

## 4. `Renewal` (Static Class - Consumer)

**Namespace:** `NServiceBus.Transport.SQS`
**File:** `src/NServiceBus.Transport.SQS/Receiving/Renewal.cs`

Static class containing the message visibility renewal logic. This is the primary consumer of `TimeProvider` in the renewal context.

### Methods

#### `RenewMessageVisibility()`

```csharp
public static async Task<Result> RenewMessageVisibility(
    Message receivedMessage,
    DateTimeOffset visibilityExpiresOn,
    int visibilityTimeoutInSeconds,
    IAmazonSQS sqsClient,
    string inputQueueUrl,
    CancellationTokenSource messageVisibilityLostCancellationTokenSource,
    TimeProvider timeProvider = null,
    CancellationToken cancellationToken = default)
```

Runs a `while` loop that:
1. Calls `CalculateRenewalTime()` to determine when to next renew.
2. Calls `Task.Delay(renewAfter, timeProvider, cancellationToken)` - this internally calls `timeProvider.CreateTimer()`.
3. After the delay, calls `sqsClient.ChangeMessageVisibilityAsync()` to extend visibility.
4. Updates `visibilityExpiresOn` and loops.

Exits with `Result.Stopped` on cancellation or `Result.Failed` on SQS errors.

#### `CalculateRenewalTime()`

```csharp
public static TimeSpan CalculateRenewalTime(
    DateTimeOffset visibilityTimeExpiresOn,
    TimeProvider timeProvider = null)
```

Calculates the delay before the next renewal:
- Returns `TimeSpan.Zero` if remaining time is less than 400ms (immediate renewal).
- Otherwise returns `remainingTime - buffer`, where buffer is `min(remainingTime / 2, 10 seconds)`.

### Enum: `Renewal.Result`

```csharp
public enum Result
{
    Failed,   // Renewal failed due to SQS errors
    Stopped   // Renewal was cancelled gracefully
}
```

---

## 5. `InputQueuePump` (Orchestrator)

**Namespace:** `NServiceBus.Transport.SQS`
**File:** `src/NServiceBus.Transport.SQS/Receiving/InputQueuePump.cs`

Orchestrates message receiving and processing with visibility renewal.

### Relevant Method

#### `ProcessMessageWithVisibilityRenewal()`

```csharp
internal async Task ProcessMessageWithVisibilityRenewal(
    Message receivedMessage,
    DateTimeOffset visibilityExpiresOn,
    TimeProvider timeProvider = null,
    CancellationToken cancellationToken = default)
```

- Creates a linked `CancellationTokenSource` with `maxAutoMessageVisibilityRenewalDuration` timeout.
- Starts `Renewal.RenewMessageVisibility()` as a background task.
- Processes the message concurrently.
- On completion, cancels the renewal task and awaits it.
- If processing failed and renewal did not fail, returns the message to the queue.

---

## 6. `ITimer` (Interface)

**Namespace:** `System.Threading`
**Assembly:** `System.Runtime` (.NET 8+)

```csharp
public interface ITimer : IDisposable, IAsyncDisposable
{
    bool Change(TimeSpan dueTime, TimeSpan period);
}
```

Returned by `TimeProvider.CreateTimer()`. Represents a timer that can be modified or disposed. In the fake implementation, the timer's callback is invoked when `FakeTimeProvider.Advance()` moves past the due time.

---

## 7. `MockSqsClient` (Test Double)

Used in `RenewalTests` to mock the `IAmazonSQS` interface. Captures `ChangeMessageVisibilityRequest` calls for assertion.

### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `ChangeMessageVisibilityRequestsSent` | Collection of `ChangeMessageVisibilityRequest` | Stores all visibility change requests made during the test. |
| `ChangeMessageVisibilityRequestResponse` | `Func<ChangeMessageVisibilityRequest, CancellationToken, ChangeMessageVisibilityResponse>` | Optional delegate to customize the response or throw exceptions. |

---

## 8. `TaskCompletionSource` (Signaling Mechanism)

**Namespace:** `System.Threading.Tasks`

Used within `SignalingFakeTimeProvider` as the signaling mechanism:

- **`TaskCreationOptions.RunContinuationsAsynchronously`**: Ensures that any code awaiting the `Task` property does not run synchronously on the thread that calls `TrySetResult()`. This prevents potential deadlocks when the timer callback and the test code share the same synchronization context.
- **`TrySetResult()`**: Non-throwing variant that returns `false` if the `TaskCompletionSource` has already been completed. This is important because `CreateTimer` may be called multiple times between `WaitForNextTimer()` calls.
