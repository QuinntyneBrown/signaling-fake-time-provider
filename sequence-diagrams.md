# Sequence Diagrams

## 1. Production Visibility Renewal Loop

Shows the runtime behavior of the renewal loop when processing a message in production with `TimeProvider.System`.

```plantuml
@startuml Production_Renewal_Loop

participant "InputQueuePump" as Pump
participant "Renewal" as R
participant "TimeProvider.System" as TP
participant "IAmazonSQS" as SQS
participant "Task.Delay" as Delay

Pump -> R : RenewMessageVisibility(\n  message, visibilityExpiresOn,\n  visibilityTimeoutInSeconds,\n  sqsClient, inputQueueUrl,\n  tokenSource, timeProvider)

activate R

loop while !cancellationToken.IsCancellationRequested

    R -> R : CalculateRenewalTime(\n  visibilityExpiresOn, timeProvider)
    R -> TP : GetUtcNow()
    TP --> R : currentTime

    R -> R : remainingTime = expiresOn - currentTime\nbuffer = min(remaining/2, 10s)\nrenewAfter = remaining - buffer

    R -> Delay : Task.Delay(renewAfter, timeProvider, ct)

    note right of Delay
        Internally calls
        timeProvider.CreateTimer()
        which creates a real OS timer
    end note

    Delay -> TP : CreateTimer(callback, state,\n  dueTime, period)
    TP --> Delay : ITimer

    ... renewAfter elapses ...

    Delay --> R : delay completed

    R -> TP : GetUtcNow()
    TP --> R : currentTime

    R -> R : calculatedVisibilityTimeout =\n  max(|remaining| + visibilityTimeout,\n  visibilityTimeout)

    R -> R : visibilityExpiresOn =\n  currentTime + calculatedVisibilityTimeout

    R -> SQS : ChangeMessageVisibilityAsync(\n  queueUrl, receiptHandle,\n  calculatedVisibilityTimeout)
    SQS --> R : success

end

R --> Pump : Result.Stopped

deactivate R

@enduml
```

## 2. Test with SignalingFakeTimeProvider (Multi-Renewal)

Shows the test scenario `Should_renew_until_cancelled_according_to_renewal_time` with timer synchronization.

```plantuml
@startuml Test_SignalingFakeTimeProvider

actor "Test" as Test
participant "SignalingFakeTimeProvider" as SFTP
participant "Renewal\nLoop" as R
participant "Task.Delay\n(internals)" as Delay
participant "MockSqsClient" as Mock

== Setup ==

Test -> SFTP ** : new SignalingFakeTimeProvider()
Test -> Mock ** : new MockSqsClient()

Test -> R : RenewMessageVisibility(\n  message, expiresOn=start+10s,\n  visTimeout=10, sqsClient,\n  queueUrl, tokenSource,\n  timeProvider=fakeTP, ct)

activate R

R -> R : CalculateRenewalTime()\nremaining=10s, buffer=5s\nrenewAfter=5s

R -> Delay : Task.Delay(5s, fakeTP, ct)

Delay -> SFTP : CreateTimer(callback,\n  state, 5s, Infinite)

note right of SFTP
  First timer registered
  synchronously before
  RenewMessageVisibility returns.
  No WaitForNextTimer needed.
end note

SFTP -> SFTP : base.CreateTimer()
SFTP -> SFTP : _nextTimerRegistered\n.TrySetResult()
SFTP --> Delay : ITimer (fake)
Delay --> R : awaiting...

== First Renewal Cycle ==

Test -> SFTP : WaitForNextTimer()

note right of Test
  Creates new TaskCompletionSource.
  Must call BEFORE Advance()
  to catch the next timer.
end note

SFTP --> Test : Task (waiting)

Test -> SFTP : Advance(5 seconds)

note right of SFTP
  Time advances to start+5s.
  Timer (due at 5s) fires.
  Callback resumes Task.Delay.
end note

SFTP --> R : delay completed (callback fired)

R -> SFTP : GetUtcNow()
SFTP --> R : start + 5s

R -> R : remaining = 5s\ncalcVisibility = 5 + 10 = 15

R -> Mock : ChangeMessageVisibilityAsync(\n  visibilityTimeout: 15)
Mock --> R : success

R -> R : visibilityExpiresOn =\n  (start+5s) + 15s = start+20s

R -> R : CalculateRenewalTime()\nremaining=15s, buffer=7.5s\nrenewAfter=7.5s

R -> Delay : Task.Delay(7.5s, fakeTP, ct)

Delay -> SFTP : CreateTimer(callback,\n  state, 7.5s, Infinite)
SFTP -> SFTP : base.CreateTimer()
SFTP -> SFTP : _nextTimerRegistered\n.TrySetResult()

note left of SFTP #LightGreen
  Signal! The test's
  WaitForNextTimer() Task
  now completes.
end note

SFTP --> Delay : ITimer (fake)

Test <-- SFTP : await waitForSecondTimer\ncompletes

== Second Renewal Cycle ==

Test -> SFTP : WaitForNextTimer()
SFTP --> Test : Task (waiting)

Test -> SFTP : Advance(11 seconds)

note right of SFTP
  Time advances to start+16s.
  Timer (due at 7.5s from start+5s = start+12.5s) fires.
end note

SFTP --> R : delay completed

R -> SFTP : GetUtcNow()
SFTP --> R : start + 16s

R -> R : remaining = 4s\ncalcVisibility = 4 + 10 = 14

R -> Mock : ChangeMessageVisibilityAsync(\n  visibilityTimeout: 14)
Mock --> R : success

R -> R : visibilityExpiresOn updated
R -> R : CalculateRenewalTime()

R -> Delay : Task.Delay(renewAfter, fakeTP, ct)
Delay -> SFTP : CreateTimer(...)
SFTP -> SFTP : base.CreateTimer()
SFTP -> SFTP : _nextTimerRegistered\n.TrySetResult()

Test <-- SFTP : await waitForThirdTimer\ncompletes

== Teardown ==

Test -> Test : tokenSource.CancelAsync()
R --> Test : Result.Stopped

deactivate R

Test -> Test : Assert: 2 renewals\nAssert: visibility[0] == 15\nAssert: visibility[1] == 14\nAssert: result == Stopped

@enduml
```

## 3. Renewal Calculation Logic

Shows the decision flow inside `CalculateRenewalTime`.

```plantuml
@startuml CalculateRenewalTime_Flow

participant "Caller" as C
participant "Renewal" as R
participant "TimeProvider" as TP

C -> R : CalculateRenewalTime(\n  visibilityTimeExpiresOn,\n  timeProvider)

activate R

R -> TP : GetUtcNow()
TP --> R : utcNow

R -> R : remainingTime =\n  visibilityTimeExpiresOn - utcNow

alt remainingTime < 400ms
    R --> C : TimeSpan.Zero\n(immediate renewal)
else remainingTime >= 400ms
    R -> R : buffer = min(\n  remainingTime / 2,\n  MaximumRenewBufferDuration(10s))
    R -> R : renewAfter =\n  remainingTime - buffer
    R --> C : renewAfter
end

deactivate R

@enduml
```

## 4. InputQueuePump Message Processing with Renewal

Shows how `InputQueuePump` orchestrates concurrent message processing and visibility renewal.

```plantuml
@startuml InputQueuePump_Processing

participant "PumpLoop" as PL
participant "InputQueuePump" as IQP
participant "Renewal" as R
participant "TimeProvider" as TP
participant "IAmazonSQS" as SQS
participant "MessagePipeline" as MP

PL -> SQS : ReceiveMessageAsync()
SQS --> PL : messages[]

PL -> PL : visibilityExpiresOn =\n  UtcNow + visibilityTimeout

loop for each message

    PL -> IQP : ProcessMessageWithVisibilityRenewal(\n  message, visibilityExpiresOn,\n  timeProvider, ct)

    activate IQP

    IQP -> IQP : renewalCTS =\n  LinkedTokenSource(ct)
    IQP -> IQP : renewalCTS.CancelAfter(\n  maxAutoRenewalDuration)
    IQP -> IQP : visibilityLostCTS =\n  LinkedTokenSource(ct)

    IQP -> R : RenewMessageVisibility(\n  message, expiresOn,\n  visibilityTimeout, sqsClient,\n  queueUrl, visibilityLostCTS,\n  timeProvider, renewalCTS.Token)
    activate R

    note over R
        Renewal loop runs
        concurrently with
        message processing
    end note

    IQP -> MP : ProcessMessage(\n  message,\n  visibilityLostCTS.Token)
    activate MP

    ... message processing ...

    MP --> IQP : processing complete
    deactivate MP

    IQP -> IQP : renewalCTS.CancelAsync()

    note right
        Cancelling stops
        the renewal loop
    end note

    R --> IQP : Result.Stopped
    deactivate R

    alt processing failed AND renewal did not fail
        IQP -> SQS : ChangeMessageVisibility(\n  visibilityTimeout: 0)
        note right
            Return message
            to queue
        end note
    end

    deactivate IQP

end

@enduml
```

## 5. Race Condition Without SignalingFakeTimeProvider

Demonstrates the race condition that `SignalingFakeTimeProvider` prevents.

```plantuml
@startuml Race_Condition

actor "Test" as Test
participant "FakeTimeProvider\n(without signaling)" as FTP
participant "Renewal Loop" as R
participant "Task.Delay\n(internals)" as Delay

== The Problem ==

Test -> R : RenewMessageVisibility(...)
activate R

R -> Delay : Task.Delay(5s, fakeTP, ct)
Delay -> FTP : CreateTimer(5s)
FTP --> Delay : ITimer

Test -> FTP : Advance(5 seconds)
FTP --> R : timer fires, delay completes

R -> R : ChangeMessageVisibilityAsync()\n(awaiting SQS call)

note right of Test #FFAAAA
  **RACE CONDITION**
  Test advances time again
  BEFORE the renewal loop
  has registered the next timer!
end note

Test -> FTP : Advance(5 seconds)

note right of FTP #FFAAAA
  No timer is registered yet!
  This advance does nothing
  to the renewal loop.
end note

R -> R : CalculateRenewalTime()
R -> Delay : Task.Delay(renewAfter, fakeTP, ct)
Delay -> FTP : CreateTimer(renewAfter)

note right of FTP #FFAAAA
  Timer registered AFTER
  the second Advance().
  It will never fire because
  the test already advanced
  past its due time.
end note

... test hangs or produces incorrect results ...

deactivate R

== The Solution: SignalingFakeTimeProvider ==

note over Test, Delay #AAFFAA
  With SignalingFakeTimeProvider, the test calls
  WaitForNextTimer() before Advance(), then awaits
  the returned Task. This guarantees the next timer
  is registered before the test proceeds.
end note

@enduml
```

## 6. Error Handling in Renewal

Shows how different error scenarios are handled in the renewal loop.

```plantuml
@startuml Renewal_Error_Handling

participant "Renewal Loop" as R
participant "IAmazonSQS" as SQS
participant "visibilityLostCTS" as VLCTS

R -> SQS : ChangeMessageVisibilityAsync()

alt Success
    SQS --> R : OK
    R -> R : Update visibilityExpiresOn\nContinue loop

else ReceiptHandleIsInvalidException
    SQS --> R : ReceiptHandleIsInvalidException
    R -> VLCTS : CancelAsync()
    note right
        Signals that message visibility
        is lost. Processing should stop.
    end note
    R --> R : return Result.Failed

else AmazonSQSException (visibility expired)
    SQS --> R : AmazonSQSException\n(IsCausedByMessageVisibilityExpiry)
    R -> VLCTS : CancelAsync()
    R --> R : return Result.Failed

else OperationCanceledException
    note right of R
        Cancellation was requested
        (message processing completed)
    end note
    R --> R : return Result.Stopped

else Other Exception
    SQS --> R : unexpected exception
    R --> R : return Result.Failed
end

@enduml
```
