# Class Diagrams

## Complete Class Hierarchy

```plantuml
@startuml SignalingFakeTimeProvider_ClassDiagram

skinparam classAttributeIconSize 0
skinparam classFontSize 12
skinparam class {
    BackgroundColor White
    BorderColor Black
    ArrowColor Black
}

package "System (.NET BCL)" {
    abstract class TimeProvider {
        + {static} System : TimeProvider
        --
        + {abstract} GetUtcNow() : DateTimeOffset
        + GetLocalNow() : DateTimeOffset
        + {abstract} LocalTimeZone : TimeZoneInfo
        + {abstract} TimestampFrequency : long
        + {abstract} GetTimestamp() : long
        + {abstract} CreateTimer(callback, state, dueTime, period) : ITimer
    }

    interface ITimer {
        + Change(dueTime : TimeSpan, period : TimeSpan) : bool
    }

    interface IDisposable {
        + Dispose() : void
    }

    interface IAsyncDisposable {
        + DisposeAsync() : ValueTask
    }
}

package "System.Threading (.NET BCL)" {
    class TaskCompletionSource {
        + TaskCompletionSource(creationOptions)
        --
        + Task : Task
        + TrySetResult() : bool
        + SetResult() : void
        + TrySetCanceled() : bool
        + TrySetException(exception) : bool
    }

    delegate TimerCallback {
        + Invoke(state : object) : void
    }
}

package "Microsoft.Extensions.Time.Testing" {
    class FakeTimeProvider {
        + FakeTimeProvider()
        + FakeTimeProvider(startDateTime : DateTimeOffset)
        --
        + Start : DateTimeOffset
        + GetUtcNow() : DateTimeOffset
        + Advance(delta : TimeSpan) : void
        + SetUtcNow(value : DateTimeOffset) : void
        + CreateTimer(callback, state, dueTime, period) : ITimer
    }
}

package "NServiceBus.Transport.SQS.Tests" {
    class SignalingFakeTimeProvider {
        - _nextTimerRegistered : TaskCompletionSource <<volatile>>
        --
        + WaitForNextTimer() : Task
        + CreateTimer(callback, state, dueTime, period) : ITimer
    }
}

ITimer --|> IDisposable
ITimer --|> IAsyncDisposable
FakeTimeProvider --|> TimeProvider
SignalingFakeTimeProvider --|> FakeTimeProvider
SignalingFakeTimeProvider --> TaskCompletionSource : signals
TimeProvider ..> ITimer : <<creates>>
TimeProvider ..> TimerCallback : <<uses>>

@enduml
```

## Production Components Class Diagram

```plantuml
@startuml Production_ClassDiagram

skinparam classAttributeIconSize 0
skinparam classFontSize 12

package "NServiceBus.Transport.SQS" {

    class Renewal <<static>> {
        - {static} MaximumRenewBufferDuration : TimeSpan
        - {static} Logger : ILog
        --
        + {static} RenewMessageVisibility(receivedMessage, visibilityExpiresOn,\n    visibilityTimeoutInSeconds, sqsClient, inputQueueUrl,\n    messageVisibilityLostCancellationTokenSource,\n    timeProvider, cancellationToken) : Task<Result>
        + {static} CalculateRenewalTime(visibilityTimeExpiresOn,\n    timeProvider) : TimeSpan
    }

    enum "Renewal.Result" as RenewalResult {
        Failed
        Stopped
    }

    class InputQueuePump {
        - inputQueueUrl : string
        - visibilityTimeoutInSeconds : int
        - maxAutoMessageVisibilityRenewalDuration : TimeSpan
        --
        + Initialize(limitations, onMessage, onError, ct) : Task
        + StartReceive(ct) : Task
        + StopReceive(ct) : Task
        + ChangeConcurrency(limitations, ct) : Task
        ~ ProcessMessageWithVisibilityRenewal(receivedMessage,\n    visibilityExpiresOn, timeProvider, ct) : Task
        ~ ProcessMessage(receivedMessage, ct) : Task
    }

    interface IMessageReceiver {
        + Initialize(limitations, onMessage, onError, ct) : Task
        + StartReceive(ct) : Task
        + StopReceive(ct) : Task
    }
}

package "Amazon.SQS" {
    interface IAmazonSQS {
        + ChangeMessageVisibilityAsync(request, ct) : Task<Response>
        + ReceiveMessageAsync(request, ct) : Task<Response>
        + DeleteMessageAsync(queueUrl, receiptHandle, ct) : Task<Response>
    }
}

package "System" {
    abstract class TimeProvider
}

Renewal ..> RenewalResult : <<returns>>
Renewal ..> IAmazonSQS : <<uses>>
Renewal ..> TimeProvider : <<uses>>
InputQueuePump ..|> IMessageReceiver
InputQueuePump --> Renewal : delegates renewal to
InputQueuePump ..> TimeProvider : <<uses>>
InputQueuePump ..> IAmazonSQS : <<uses>>

@enduml
```

## Test Infrastructure Class Diagram

```plantuml
@startuml Test_ClassDiagram

skinparam classAttributeIconSize 0
skinparam classFontSize 12

package "NServiceBus.Transport.SQS.Tests" {

    class RenewalTests <<TestFixture>> {
        + Should_calculate_correctly(renewalTime, expected) : void
        + Should_return_zero_delay_for_remaining_time_smaller_than_minimum_threshold() : void
        + Should_renew_until_cancelled_according_to_renewal_time() : Task
        + Should_not_renew_when_time_passed_below_renewal_time() : Task
        + Should_renew_when_expired_but_below_threshold() : Task
        + Should_renew_when_expired_but_below_buffer() : Task
        + Should_notify_about_visibility_extension_failing() : Task
        + Should_indicate_stopped_when_cancelled_from_beginning() : Task
    }

    class SignalingFakeTimeProvider {
        - _nextTimerRegistered : TaskCompletionSource <<volatile>>
        --
        + WaitForNextTimer() : Task
        + CreateTimer(callback, state, dueTime, period) : ITimer
    }

    class MockSqsClient {
        + ChangeMessageVisibilityRequestsSent : List<ChangeMessageVisibilityRequest>
        + ChangeMessageVisibilityRequestResponse : Func<...>
        --
        + ChangeMessageVisibilityAsync(request, ct) : Task<Response>
    }
}

package "Microsoft.Extensions.Time.Testing" {
    class FakeTimeProvider
}

package "Amazon.SQS" {
    interface IAmazonSQS
}

RenewalTests +-- SignalingFakeTimeProvider : <<nested>>
SignalingFakeTimeProvider --|> FakeTimeProvider
MockSqsClient ..|> IAmazonSQS
RenewalTests --> SignalingFakeTimeProvider : <<creates>>
RenewalTests --> MockSqsClient : <<creates>>
RenewalTests --> FakeTimeProvider : <<creates>>

@enduml
```

## Dependency Injection Flow

```plantuml
@startuml DI_Flow

skinparam classAttributeIconSize 0

package "Production Runtime" {
    class "TimeProvider.System" as SystemTP
}

package "Test Runtime" {
    class FakeTimeProvider
    class SignalingFakeTimeProvider
}

package "Consumer" {
    class Renewal <<static>>
    class InputQueuePump
}

abstract class TimeProvider

SystemTP --|> TimeProvider
FakeTimeProvider --|> TimeProvider
SignalingFakeTimeProvider --|> FakeTimeProvider

InputQueuePump ..> Renewal : calls
InputQueuePump ..> TimeProvider : "timeProvider ?? TimeProvider.System"
Renewal ..> TimeProvider : "timeProvider ?? TimeProvider.System"

note right of SystemTP
  Used in production.
  Real wall-clock time
  and real timers.
end note

note right of SignalingFakeTimeProvider
  Used in tests requiring
  multi-step timer
  synchronization.
end note

note right of FakeTimeProvider
  Used in simpler tests
  that don't need timer
  signaling.
end note

@enduml
```
